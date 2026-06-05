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
VER=26.6.3   # any keycloak-k8s-resources tag
# CRDs
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/$VER/kubernetes/keycloaks.k8s.keycloak.org-v1.yml
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/$VER/kubernetes/keycloakrealmimports.k8s.keycloak.org-v1.yml
# Operator Deployment + RBAC (into the install namespace)
kubectl apply -n keycloak-system -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/$VER/kubernetes/kubernetes.yml
```

See https://www.keycloak.org/operator/installation for exact per-release URLs.
Verified end-to-end on kind with operator **26.6.3** (these exact URLs).

## Known constraint: operator startup probe on constrained nodes

On memory-constrained nodes the operator's Quarkus pod can fail its default
`startupProbe` (`/q/health/started`) before the JVM finishes booting, and
CrashLoopBackOff. If `keycloak-operator` won't reach Ready, relax the probe:

```bash
kubectl -n keycloak-system patch deploy keycloak-operator --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/startupProbe/failureThreshold","value":30}]'
```

The Keycloak **server** pod also needs a real DB (Postgres) reachable at
`db.host`; with `db.secret.create=true` the chart's generated credentials must
match the DB. Set `hostname-strict=false` (via `keycloak.additionalOptions`) when
clients call the server by its in-cluster Service name rather than the public
hostname.

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
