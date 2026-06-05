# keycloak-operator-blueprint

Krateo blueprint that deploys and manages a **Keycloak server** through the
**official Keycloak Operator** (`k8s.keycloak.org/v2alpha1`). This is the
**lifecycle** half of the Keycloak↔Krateo↔OpenStack SSO work; the
**configuration** half (realms, clients, mappers as CRs) is the sibling
`keycloak-config-kog`.

## Why the Operator (and not Bitnami)

Bitnami's Keycloak chart/images moved behind a paid subscription in Aug 2025 and
now ship unmaintained `bitnamilegacy` images. The official Operator is the
upstream-maintained, CRD-driven path and cleanly separates *lifecycle* (this
blueprint) from *config* (the KOG).

## What this blueprint renders

- a `Keycloak` CR (instances, hostname, DB, http/TLS, ingress, proxy headers,
  bootstrap admin)
- a DB credentials `Secret` (dev convenience; reference an external/ESO-synced
  Secret in production via `db.secret.create=false`)
- a bootstrap-admin `Secret`

It deliberately does **not** bundle the Operator itself.

## Prerequisite: install the Operator once

The Operator + its CRDs are cluster-scoped and installed once, independent of
any single server instance. Per the release you target (`operator.version`):

```bash
# CRDs
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak/<ver>/operator/src/main/resources/META-INF/fabric8/keycloaks.k8s.keycloak.org-v1.yml
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak/<ver>/operator/src/main/resources/META-INF/fabric8/keycloakrealmimports.k8s.keycloak.org-v1.yml
# Operator Deployment + RBAC
kubectl apply -n keycloak-system -f https://raw.githubusercontent.com/keycloak/keycloak/<ver>/operator/src/main/resources/META-INF/fabric8/kubernetes.yml
```

See https://www.keycloak.org/operator/installation for exact per-release URLs.

## Hand-off to keycloak-config-kog

Once the server is Ready:
1. Create a confidential **service-account client** `krateo-kog` with the
   `realm-management` roles the config KOG needs (manage-clients, manage-realm,
   manage-users, …) and `serviceAccountsEnabled=true`.
2. Put its client secret in the `keycloak-kog-client` Secret that
   `keycloak-config-kog`'s ESO `Webhook` generator reads.
3. Point `keycloak-config-kog`'s `keycloak.baseUrl` at this server's hostname.

> Bootstrapping note: that first `krateo-kog` client can be created by the
> bootstrap admin manually, or — nicely circular — by a one-off
> `KeycloakClient` CR once the config KOG is up against the master realm.

## Federation prerequisites for the OpenStack side

For the Horizon shared-session SSO, this Keycloak's realm must expose a `groups`
claim and the `keystone` OIDC client — both provisioned declaratively by
`keycloak-config-kog` (see its `samples/10-sso-realm.yaml`).
