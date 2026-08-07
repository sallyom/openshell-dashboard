# Agent prompt for an isolated OpenShell installation

Give the prompt below to an agent after:

- logging in to OpenShift with cluster-admin access;
- cloning this repository; and
- receiving an enabled user from the configured OIDC provider with both
  `openshell-admin` and `openshell-user` roles.

The identity-provider administrator must also create the
installation-specific OIDC client described by the agent. That client cannot
be shared safely through a wildcard redirect URI.

## Prompt

```text
Help me install one isolated OpenShell Gateway and Dashboard on this shared
OpenShift cluster.

Context:
- I am already logged in to OpenShift with cluster-admin access.
- The current repository is `openshell-dashboard`.
- My application login is my own user from the configured OIDC provider.
- My OIDC user already has the `openshell-admin` and `openshell-user` roles.
- Ask me for my exact OIDC username, issuer URL, supported scopes, and a
  lowercase, DNS-safe owner ID. Do not infer or normalize any value silently.

Read `AGENTS.md`, `docs/openshift.md`,
`deploy/openshift/openshell-values.example.yaml`, and
`deploy/openshift/dashboard-template.yaml` completely before acting. Treat
`docs/openshift.md` as the installation procedure, with these validated
defaults:

- OpenShell chart version validated on this cluster: `0.0.99`
- Agent Sandbox version already installed cluster-wide: `v0.5.4`
- Dashboard image:
  `quay.io/gkrumbach07/openshell-dashboard@sha256:edba1b449fceeb3a75713597d79e809dfd19955e305e1f13558623fb93ec8248`

Derive these installation-specific names only after I confirm the owner ID:

- namespace and Helm fullname: `openshell-<owner>`
- OIDC client ID: `openshell-dashboard-<owner>`
- Dashboard Route:
  `openshell-dashboard-openshell-<owner>.<cluster apps domain>`
- Gateway URL:
  `openshell-<owner>.openshell-<owner>.svc.cluster.local:8080`
- sandbox service account: `openshell-<owner>-sandbox`

Success means:

1. The unique namespace exists without changing another user's namespace or
   Helm release.
2. The pinned OpenShell Gateway is Ready and remains cluster-internal.
3. Only this installation's sandbox service account receives the privileged
   SCC.
4. The Dashboard uses the digest-pinned upstream image and a stable session
   Secret stored only in Kubernetes.
5. Public health and readiness endpoints return success; unauthenticated
   Gateway and session requests return 401.
6. I can sign in with my own OIDC account, create a workspace and Sandbox,
   and connect to its terminal.

Work in approval-gated phases:

Phase 1 — read-only preflight:
- Confirm the git branch and working-tree state.
- Confirm `oc whoami`, cluster-admin permission, storage, the apps domain, and
  that the Agent Sandbox controller and CRD already exist.
- Do not reinstall or upgrade Agent Sandbox. It is shared cluster-wide.
- Check all namespaces, Helm releases, ClusterRoles, and ClusterRoleBindings
  for collisions with the derived names. Never delete or adopt an existing
  resource.
- Check the currently available OpenShell chart version. Use `0.0.99`, which
  is validated here, unless a newer version exists; if one exists, report the
  compatibility tradeoff and wait for my approval before changing versions.
- Show the exact derived values and planned mutations, then pause for my
  approval.

OIDC gate:
- Before installing the Dashboard, show me the exact OIDC client ID,
  redirect URI (`https://<route>/auth/callback`), and web origin
  (`https://<route>`).
- The client must be public, use Authorization Code flow with S256 PKCE, and
  disable direct grants and service accounts.
- Its ID token must include realm roles as a multivalued `roles` claim and
  `openshell-cli` as an audience.
- Wait for me to confirm that the identity-provider administrator created this
  client. Do not read identity-provider administrator credentials or modify
  the provider yourself.

Phase 2 — namespace:
- After approval, create only the derived namespace. Pause and report the
  result.

Phase 3 — Gateway:
- After approval, install the pinned Helm chart using a private copy of the
  example values, the confirmed OIDC issuer, the unique `fullnameOverride`,
  and the same unique sandbox namespace.
- Grant `privileged` only to the derived sandbox service account.
- Wait for and verify the Gateway. Pause and report the result.

Phase 4 — Dashboard:
- After approval and OIDC-client confirmation, generate the Dashboard session
  Secret without printing it.
- Process and apply the OpenShift Dashboard template with the unique namespace,
  internal Gateway URL, Route, OIDC client ID, confirmed OIDC scopes, and
  pinned image digest.
- Wait for the rollout. Pause and report the result.

Phase 5 — verification:
- Perform the documented API, pod, Route, image-digest, authentication-boundary,
  and Sandbox prerequisite checks.
- Ask me to sign in as my own OIDC user and complete the workspace,
  Sandbox, and terminal UI test. Do not ask for or handle my password.
- Give me a concise final inventory, exact versions and digests, rollback
  command, and any remaining risks.

Safety and documentation rules:
- Never change `openshell-system`, another teammate's namespace, the shared
  Agent Sandbox installation, or existing cluster-scoped resources.
- Do not expose the Gateway with a public Route.
- Do not set `AUTH_DISABLED=true`.
- Do not print, store in git, or include secrets in documentation.
- Preserve unrelated working-tree changes. Do not commit or push unless I
  explicitly ask.
- Record the approved commands and non-secret verification results in a short
  installation runbook for me.
```
