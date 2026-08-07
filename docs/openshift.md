# Deploy OpenShell Dashboard on OpenShift

This guide installs one isolated OpenShell Gateway and one Dashboard for a test
cluster. Repeat it for teammates only with a unique installation ID, namespace,
Gateway name, OIDC client, and Route. OpenShell's Kubernetes support is
experimental, so pin versions and validate upgrades before broader use.

## Architecture

```text
Browser
  -> OpenShift Route (HTTPS and WebSocket)
  -> Dashboard (React UI and Go BFF)
  -> cluster-internal OpenShell Gateway (gRPC)
  -> Agent Sandbox controller
  -> Sandbox pods and workspace PVCs
```

The Dashboard does not require Kubernetes API access. Its pod does not mount a
service-account token. The Gateway manages Sandbox resources through its own
RBAC, and only the sandbox service account receives the `privileged` SCC.

## Prerequisites

- OpenShift cluster-admin access for initial installation
- `oc` and Helm 3
- a default `ReadWriteOnce` StorageClass
- an OIDC provider reachable by browsers, the Dashboard, and the Gateway
- a Dashboard container image, preferably pinned by digest
- a stable random secret for encrypting Dashboard login sessions

Choose a lowercase, DNS-safe owner or installation ID. The commands below use
`alice` as an example and derive collision-free names from it. OpenShell chart
`0.0.99` and Agent Sandbox `v0.5.4` were validated together; review current
release compatibility before changing either version.

```bash
export OPENSHELL_OWNER=alice
export OPENSHELL_NAMESPACE="openshell-${OPENSHELL_OWNER}"
export OPENSHELL_NAME="openshell-${OPENSHELL_OWNER}"
export OPENSHELL_VERSION=0.0.99
export AGENT_SANDBOX_VERSION=v0.5.4
export OIDC_CLIENT_ID="openshell-dashboard-${OPENSHELL_OWNER}"
export OIDC_ISSUER=https://idp.example.com/realms/openshell
export APPS_DOMAIN=$(oc get ingresses.config.openshift.io cluster \
  -o jsonpath='{.spec.domain}')
export ROUTE_HOST="openshell-dashboard-${OPENSHELL_NAMESPACE}.${APPS_DOMAIN}"
export GATEWAY_URL="${OPENSHELL_NAME}.${OPENSHELL_NAMESPACE}.svc.cluster.local:8080"

oc get storageclass
oc auth can-i create namespaces
oc auth can-i use securitycontextconstraints/privileged
```

## Install Agent Sandbox

Agent Sandbox is cluster-wide, not per OpenShell installation. Check it first:

```bash
oc get deployment/agent-sandbox-controller -n agent-sandbox-system
oc get crd sandboxes.agents.x-k8s.io
```

If the validated version is already running, do not reinstall or upgrade it.
Otherwise, a cluster administrator installs the versioned core controller
manifest once. Avoid a floating `latest` URL and optional extensions unless
they are required separately.

```bash
oc apply -f \
  "https://github.com/kubernetes-sigs/agent-sandbox/releases/download/${AGENT_SANDBOX_VERSION}/sandbox.yaml"

oc rollout status deployment/agent-sandbox-controller \
  -n agent-sandbox-system \
  --timeout=180s

oc get crd sandboxes.agents.x-k8s.io
```

Upgrading an existing controller is cluster-wide. Inspect the versioned
manifest and verify existing Sandbox objects through every served API version
before proceeding.

## Configure OIDC

Create two realm or tenant roles:

| Role | Purpose |
| --- | --- |
| `openshell-admin` | Platform administration and workspace distribution |
| `openshell-user` | Regular authenticated workspace use |

The identity-provider administrator creates the teammate's account, assigns
both roles above, and creates a public client named `${OIDC_CLIENT_ID}` with:

- Authorization Code flow enabled
- client authentication, direct grants, and service accounts disabled
- PKCE method `S256`
- redirect URI `https://<dashboard-route>/auth/callback`
- web origin `https://<dashboard-route>`

Use the exact `${ROUTE_HOST}` derived above for the redirect URI and web
origin. Prefer a separate client per installation; do not use wildcard
redirect URIs across teammate namespaces.

The ID token must contain:

- the stable user identifier in `sub`;
- the configured roles in a multivalued `roles` claim; and
- `openshell-cli` in `aud`.

The Dashboard forwards the ID token to the Gateway, so claims required for
authorization must be present in the ID token, not only the access token.
Use only OIDC scopes supported by the provider.

## Install the Gateway

Create the namespace and make a private copy of the example values:

```bash
oc create namespace "$OPENSHELL_NAMESPACE"

cp deploy/openshift/openshell-values.example.yaml \
  /tmp/openshell-values.yaml
```

Edit `/tmp/openshell-values.yaml` and set `server.oidc.issuer`. The command
below overrides both installation-specific names so the Service,
service-account, ClusterRole, and ClusterRoleBinding do not collide with
another teammate's installation.

Install the pinned chart:

```bash
helm install openshell \
  oci://ghcr.io/nvidia/openshell/helm-chart \
  --version "$OPENSHELL_VERSION" \
  --namespace "$OPENSHELL_NAMESPACE" \
  --values /tmp/openshell-values.yaml \
  --set-string fullnameOverride="$OPENSHELL_NAME" \
  --set-string server.sandboxNamespace="$OPENSHELL_NAMESPACE" \
  --wait \
  --timeout 5m
```

Grant the privileged SCC only to the installation-specific sandbox service
account created by the chart:

```bash
oc adm policy add-scc-to-user privileged \
  -z "${OPENSHELL_NAME}-sandbox" \
  -n "$OPENSHELL_NAMESPACE"

oc auth can-i use securitycontextconstraints/privileged \
  --as="system:serviceaccount:${OPENSHELL_NAMESPACE}:${OPENSHELL_NAME}-sandbox" \
  -n "$OPENSHELL_NAMESPACE"
```

Do not grant the privileged SCC to the Dashboard or Gateway service account.
The Gateway should remain internal; do not create a public Route for it when
the Dashboard is the user entry point.

## Install the Dashboard

Set deployment-specific values. `ROUTE_HOST` must match the OIDC client's
redirect URI and web origin. `DASHBOARD_IMAGE` should be immutable in a shared
installation.

```bash
export DASHBOARD_IMAGE=quay.io/gkrumbach07/openshell-dashboard@sha256:edba1b449fceeb3a75713597d79e809dfd19955e305e1f13558623fb93ec8248

oc create secret generic openshell-dashboard-session \
  -n "$OPENSHELL_NAMESPACE" \
  --from-literal=session-secret="$(openssl rand -hex 32)"

oc process -f deploy/openshift/dashboard-template.yaml \
  -p NAMESPACE="$OPENSHELL_NAMESPACE" \
  -p DASHBOARD_IMAGE="$DASHBOARD_IMAGE" \
  -p GATEWAY_URL="$GATEWAY_URL" \
  -p ROUTE_HOST="$ROUTE_HOST" \
  -p OIDC_ISSUER="$OIDC_ISSUER" \
  -p OIDC_CLIENT_ID="$OIDC_CLIENT_ID" \
  | oc apply -f -

oc rollout status deployment/openshell-dashboard \
  -n "$OPENSHELL_NAMESPACE" \
  --timeout=180s
```

Create the session Secret once and keep it stable across upgrades and
replicas. Rotating it invalidates every active Dashboard login. To use a
different Secret name, pass `-p SESSION_SECRET_NAME=<name>` when processing
the template; its `session-secret` key must contain a high-entropy value.

The template uses edge TLS, redirects HTTP to HTTPS, and gives WebSocket
connections a one-hour router timeout. It applies the `restricted-v2`
compatible security settings validated for the Dashboard image.

If the provider does not define `profile` or `email`, override the default:

```bash
oc process -f deploy/openshift/dashboard-template.yaml \
  -p NAMESPACE="$OPENSHELL_NAMESPACE" \
  -p DASHBOARD_IMAGE="$DASHBOARD_IMAGE" \
  -p GATEWAY_URL="$GATEWAY_URL" \
  -p ROUTE_HOST="$ROUTE_HOST" \
  -p OIDC_ISSUER="$OIDC_ISSUER" \
  -p OIDC_CLIENT_ID="$OIDC_CLIENT_ID" \
  -p OIDC_SCOPES=openid \
  | oc apply -f -
```

## Verify the installation

```bash
curl -fsS "https://${ROUTE_HOST}/api/v1/healthz"
curl -fsS "https://${ROUTE_HOST}/api/v1/readyz"

# An unauthenticated Gateway request must return 401.
curl -sS -o /dev/null -w '%{http_code}\n' \
  "https://${ROUTE_HOST}/api/v1/gateway"

oc get statefulset,deployment,pod,service,route,pvc \
  -n "$OPENSHELL_NAMESPACE"
```

Sign in as an administrator and create a workspace and a Sandbox using the
`base` community image with the locked-down policy. Verify:

```bash
oc get sandboxes.v1beta1.agents.x-k8s.io,pod,pvc \
  -n "$OPENSHELL_NAMESPACE"
```

The Sandbox should become Ready, its pod should run, and its workspace PVC
should bind. Open the Terminal tab and run a simple command. A 401 on the
terminal endpoint while other APIs succeed indicates that the Dashboard image
does not include authenticated WebSocket support.

## Distribute workspaces to regular users

Assign `openshell-user` in the OIDC provider, then add the user to a workspace
by their exact OIDC `sub` value. A username or email is not interchangeable
with `sub` unless the provider explicitly uses it as the subject.

Validate with a user who has `openshell-user` but not `openshell-admin`:

1. The user can sign in.
2. Only assigned workspaces are visible.
3. Allowed Sandbox operations succeed within those workspaces.
4. Gateway-wide administration is unavailable.

Using an administrator account for this test does not prove regular-user
isolation because the admin role may bypass workspace restrictions.

## Upgrade and rollback

- Pin the Helm chart, controller manifest, and Dashboard image.
- Review Agent Sandbox CRD conversion and storage versions before upgrades.
- Test existing Sandbox objects before and after controller changes.
- Keep the prior Dashboard image digest for `oc rollout undo` or `oc set image`.
- Replace example identity-provider values; never commit client secrets.
