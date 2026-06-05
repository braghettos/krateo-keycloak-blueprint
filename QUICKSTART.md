# Quickstart — Keycloak for Krateo + OpenStack Horizon SSO

This walks the full path end-to-end:

1. deploy a **Keycloak server** with the lifecycle blueprint
   ([krateo-keycloak-blueprint](https://github.com/braghettos/krateo-keycloak-blueprint),
   CloudNativePG-backed), then
2. manage its configuration as **Kubernetes custom resources** with the KOG
   ([krateo-keycloak-operator-kog](https://github.com/braghettos/krateo-keycloak-operator-kog)).

Every command here was validated on a local `kind` cluster (Keycloak Operator
26.6.3, CloudNativePG 1.29.1, oasgen-provider 0.10).

## The use case

Stand up a **`krateo` realm** that is the single identity provider for *both*:

- the **Krateo portal** (its `authn` service uses the `krateo-authn` OIDC client), and
- **OpenStack Horizon** (Keystone OIDC-federates to the `keystone` client; a
  shared Keycloak session means logging into Krateo silently logs the user into
  Horizon).

End state, all expressed declaratively as CRs:

| Resource | Purpose |
| --- | --- |
| `KeycloakRealm krateo` | the realm / SSO session boundary |
| `KeycloakClient keystone` | Horizon ↔ Keystone OIDC federation RP |
| `KeycloakClient krateo-authn` | Krateo portal login RP |
| `KeycloakProtocolMapper groups` | emits the `groups` claim both apps consume |
| `KeycloakGroup os-project-demo` | drives Keystone project/role mapping |

---

## 0. Prerequisites

A Kubernetes cluster + `kubectl` + `helm`. For a throwaway local one:

```bash
kind create cluster --name keycloak-demo
kubectl create namespace keycloak-system
kubectl create namespace krateo-system
```

---

## 1. Install the operators (one-time, cluster-wide)

```bash
# --- Keycloak Operator 26.6.3 ---
VER=26.6.3
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/$VER/kubernetes/keycloaks.k8s.keycloak.org-v1.yml
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/$VER/kubernetes/keycloakrealmimports.k8s.keycloak.org-v1.yml
kubectl apply -n keycloak-system -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/$VER/kubernetes/kubernetes.yml

# --- CloudNativePG 1.29.1 (the default db.provider) ---
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.29/releases/cnpg-1.29.1.yaml

# --- KOG runtime (oasgen-provider + rest-dynamic-controller) ---
helm repo add krateo https://charts.krateo.io && helm repo update krateo
helm install oasgen-provider krateo/oasgen-provider -n krateo-system

# wait for all three
kubectl -n keycloak-system rollout status deploy/keycloak-operator
kubectl -n cnpg-system     rollout status deploy/cnpg-controller-manager
kubectl -n krateo-system   rollout status deploy/oasgen-provider
```

> On memory-constrained nodes the Keycloak operator can fail its startup probe.
> If it CrashLoopBackOffs: `kubectl -n keycloak-system patch deploy keycloak-operator
> --type=json -p='[{"op":"replace","path":"/spec/template/spec/containers/0/startupProbe/failureThreshold","value":30}]'`

---

## 2. Deploy the Keycloak server (blueprint)

```bash
git clone https://github.com/braghettos/krateo-keycloak-blueprint
helm install keycloak ./krateo-keycloak-blueprint/chart -n keycloak-system \
  --set keycloak.hostname=keycloak.demo.local \
  --set keycloak.http.httpEnabled=true \
  --set keycloak.ingress.enabled=false \
  --set-json 'keycloak.additionalOptions=[{"name":"hostname-strict","value":"false"}]'
```

`db.provider` defaults to `cloudnativepg`, so this also renders a CNPG `Cluster`
(`keycloak-db`) that provisions Postgres and the `keycloak-db-app` credentials.
`hostname-strict=false` lets the KOG reach the server by its in-cluster Service
name. Wait for it:

```bash
kubectl -n keycloak-system wait --for=condition=Ready keycloak/keycloak --timeout=5m
```

> Production: deploy via the Krateo `CompositionDefinition` instead of raw helm,
> set `keycloak.ingress.enabled=true` + a real TLS `hostname`, and
> `db.cloudnativepg.instances: 3` for an HA database.

---

## 3. Give the KOG an admin bearer token

`rest-dynamic-controller` calls the Keycloak Admin API with a bearer token read
from a Secret. Two paths:

### 3a. Quickstart (manual token)

Mint a token from the bootstrap admin **in-cluster** (so its issuer matches the
Service the KOG calls) and store it:

```bash
KC=http://keycloak-service.keycloak-system.svc.cluster.local:8080
# (optional) lengthen the token so it survives a few reconciles
TOKEN=$(kubectl -n krateo-system run kc-mint --rm -i --restart=Never --image=curlimages/curl:8.10.1 -- \
  -s -d grant_type=password -d client_id=admin-cli -d username=admin -d password=change-me \
  $KC/realms/master/protocol/openid-connect/token 2>/dev/null | sed 's/.*"access_token":"//;s/".*//')
kubectl -n krateo-system create secret generic keycloak-admin-token --from-literal=token="$TOKEN"
```

### 3b. Production (ESO-rotated token)

Create a confidential **service-account client** `krateo-kog` (with the
`realm-management` roles you need — e.g. `manage-clients`, `manage-realm`) and
put its secret in a `keycloak-kog-client` Secret. The KOG chart's
`auth.externalSecret` then has External Secrets Operator mint and rotate a fresh
token into `keycloak-admin-token` automatically. See the KOG chart's
`templates/externalsecret.yaml`.

---

## 4. Install the config KOG

```bash
git clone https://github.com/braghettos/krateo-keycloak-operator-kog
helm install keycloak-config-kog ./krateo-keycloak-operator-kog/chart -n krateo-system \
  --set keycloak.baseUrl=http://keycloak-service.keycloak-system.svc.cluster.local:8080 \
  --set auth.externalSecret.enabled=false   # using the manual token from step 3a
```

This emits one `RestDefinition` per resource; oasgen-provider generates a CRD +
controller for each. Wait for them:

```bash
kubectl -n krateo-system wait --for=condition=Ready restdefinition --all --timeout=5m
kubectl get crd | grep keycloak.ogen.krateo.io
```

---

## 5. Provision the SSO config as CRs

```bash
kubectl apply -f - <<'YAML'
# Auth config every resource points at (one per Kind; all share the token Secret)
apiVersion: keycloak.ogen.krateo.io/v1alpha1
kind: KeycloakRealmConfiguration
metadata: { name: keycloak-admin, namespace: krateo-system }
spec: { authentication: { bearer: { tokenRef: { name: keycloak-admin-token, namespace: krateo-system, key: token } } } }
---
apiVersion: keycloak.ogen.krateo.io/v1alpha1
kind: KeycloakClientConfiguration
metadata: { name: keycloak-admin, namespace: krateo-system }
spec: { authentication: { bearer: { tokenRef: { name: keycloak-admin-token, namespace: krateo-system, key: token } } } }
---
apiVersion: keycloak.ogen.krateo.io/v1alpha1
kind: KeycloakProtocolMapperConfiguration
metadata: { name: keycloak-admin, namespace: krateo-system }
spec: { authentication: { bearer: { tokenRef: { name: keycloak-admin-token, namespace: krateo-system, key: token } } } }
---
apiVersion: keycloak.ogen.krateo.io/v1alpha1
kind: KeycloakGroupConfiguration
metadata: { name: keycloak-admin, namespace: krateo-system }
spec: { authentication: { bearer: { tokenRef: { name: keycloak-admin-token, namespace: krateo-system, key: token } } } }
---
# The realm
apiVersion: keycloak.ogen.krateo.io/v1alpha1
kind: KeycloakRealm
metadata: { name: krateo, namespace: krateo-system }
spec:
  configurationRef: { name: keycloak-admin, namespace: krateo-system }
  realm: krateo
  enabled: true
  displayName: "Krateo PlatformOps"
  loginWithEmailAllowed: true
  ssoSessionIdleTimeout: 1800       # the shared-SSO window both apps ride
  ssoSessionMaxLifespan: 36000
---
# Keystone (Horizon WebSSO) federation client
apiVersion: keycloak.ogen.krateo.io/v1alpha1
kind: KeycloakClient
metadata: { name: keystone, namespace: krateo-system }
spec:
  configurationRef: { name: keycloak-admin, namespace: krateo-system }
  realm: krateo
  clientId: keystone
  name: "OpenStack Keystone (Horizon WebSSO)"
  enabled: true
  protocol: openid-connect
  publicClient: false
  standardFlowEnabled: true
  clientAuthenticatorType: client-secret
  redirectUris:
    - "https://<KEYSTONE_HOST>/v3/auth/OS-FEDERATION/identity_providers/keycloak/protocols/openid/websso"
    - "https://<KEYSTONE_HOST>/redirect_uri"
---
# Krateo portal login client
apiVersion: keycloak.ogen.krateo.io/v1alpha1
kind: KeycloakClient
metadata: { name: krateo-authn, namespace: krateo-system }
spec:
  configurationRef: { name: keycloak-admin, namespace: krateo-system }
  realm: krateo
  clientId: krateo-authn
  name: "Krateo AuthN"
  enabled: true
  protocol: openid-connect
  publicClient: false
  standardFlowEnabled: true
  clientAuthenticatorType: client-secret
  redirectUris: [ "https://<KRATEO_HOST>/auth/oidc" ]
---
# groups membership mapper on the keystone client — the claim BOTH apps consume.
# clientUuid is the keystone client's server-generated UUID: read it from the
# KeycloakClient/keystone status.id once that client reconciles, then set it here.
apiVersion: keycloak.ogen.krateo.io/v1alpha1
kind: KeycloakProtocolMapper
metadata: { name: keystone-groups, namespace: krateo-system }
spec:
  configurationRef: { name: keycloak-admin, namespace: krateo-system }
  realm: krateo
  clientUuid: "<KEYSTONE_CLIENT_UUID>"
  name: groups
  protocol: openid-connect
  protocolMapper: oidc-group-membership-mapper
  config:
    "claim.name": "groups"
    "full.path": "false"
    "id.token.claim": "true"
    "access.token.claim": "true"
    "userinfo.token.claim": "true"
---
# An OpenStack project group; the Keystone mapping turns membership into a
# project/role grant.
apiVersion: keycloak.ogen.krateo.io/v1alpha1
kind: KeycloakGroup
metadata: { name: os-project-demo, namespace: krateo-system }
spec:
  configurationRef: { name: keycloak-admin, namespace: krateo-system }
  realm: krateo
  name: os-project-demo
  attributes:
    openstack-project: ["demo"]
    openstack-role: ["member"]
YAML
```

---

## 6. Verify

```bash
# CR side
kubectl -n krateo-system get keycloakrealms,keycloakclients,keycloakgroups

# Keycloak side — the clients now exist in the krateo realm
TOKEN=$(kubectl -n krateo-system get secret keycloak-admin-token -o jsonpath='{.data.token}' | base64 -d)
kubectl -n krateo-system run verify --rm -i --restart=Never --image=curlimages/curl:8.10.1 -- \
  -s -H "Authorization: Bearer $TOKEN" \
  'http://keycloak-service.keycloak-system.svc.cluster.local:8080/admin/realms/krateo/clients?clientId=keystone'
```

You should see the `keystone` client returned. The `krateo` realm is now the
single IdP; the remaining work is on the consumers:

- **Krateo**: point the `authn` service's `OIDCConfig` at
  `…/realms/krateo/.well-known/openid-configuration` with `additionalScopes: groups`.
- **OpenStack**: configure Keystone OIDC federation (`mod_auth_openidc`) against
  the `keystone` client and enable Horizon WebSSO — both already templated in the
  `openstack-as-a-service` Keystone/Horizon charts.

---

## Cleanup

```bash
helm -n krateo-system uninstall keycloak-config-kog oasgen-provider
helm -n keycloak-system uninstall keycloak     # also removes the CNPG Cluster
kind delete cluster --name keycloak-demo
```

## Notes / gotchas (learned the hard way)

- Mint the KOG token via the **same host** the controller calls
  (`keycloak-service:8080`) — a token minted elsewhere fails with 401 (issuer
  mismatch) when `hostname-strict=false`.
- `rest-dynamic-controller` resyncs every **180s**; `status.id` populates on the
  next observe after create, not instantly.
- The blueprint provisions the DB (CNPG) but **not** the operators — those are
  cluster prerequisites (step 1).
