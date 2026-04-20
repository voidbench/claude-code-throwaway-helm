# claude-code-throwaway

Minimal Helm chart for running an isolated, throwaway Claude Code pod in Kubernetes.

## Goals

- isolated pod with non-root security context
- no service account token mounted by default
- ephemeral home and workspace by default
- default egress locked down with NetworkPolicy
- optional secret-backed env vars
- easy to reach with `kubectl exec`

## What this is

This chart gives you a disposable container to test Claude Code or similar agent tooling away from your personal workstation.

By default it starts a pod that just sleeps forever:

```sh
sleep infinity
```

Then you connect into it with:

```sh
kubectl exec -it deploy/<release>-claude-code-throwaway -- sh
```

Adjust image/command as needed.

## Quick start

Create a secret with your API key, either outside Helm:

```sh
kubectl create ns claude-sandbox
kubectl -n claude-sandbox create secret generic claude-code-env \
  --from-literal=ANTHROPIC_API_KEY=... \
  --from-literal=HOME=/home/claude
```

Install the chart:

```sh
helm upgrade --install claude ./ \
  --namespace claude-sandbox \
  --create-namespace \
  --set secretEnv.name=claude-code-env \
  --set image.repository=ghcr.io/anthropics/claude-code \
  --set image.tag=latest
```

Exec into it:

```sh
kubectl -n claude-sandbox exec -it deploy/claude-claude-code-throwaway -- sh
```

## Safer network policy setup

Default chart behavior allows DNS plus any explicitly configured CIDRs.
You should set `networkPolicy.egressCidrs` to the minimal destinations you need.

Example:

```yaml
networkPolicy:
  enabled: true
  allowDns: true
  egressCidrs:
    - 1.1.1.1/32
    - 8.8.8.8/32
```

That said, for SaaS endpoints like Anthropic or GitHub, IP pinning is annoying and brittle. In practice you may prefer one of these patterns:

- dedicated egress gateway / firewall policy outside Kubernetes
- Cilium FQDN policies, if your cluster supports them
- a very short-lived sandbox namespace with broader egress

## Suggested usage model

- one temporary namespace per experiment
- one release per task
- no persistent volume unless you really want to keep state
- delete the namespace when done

## Values of interest

- `secretEnv.name`: existing Secret to mount as env
- `secretEnv.create`: create Secret from Helm values
- `networkPolicy.*`: lock egress/ingress down
- `persistence.enabled`: keep workspace across restarts
- `service.enabled`: expose a service when needed
- `ingress.enabled`: optional ingress

## Publishing to GHCR as OCI chart

This repo includes a GitHub Actions workflow that packages the chart and pushes it to GHCR as an OCI artifact on every push to `main`.

Published location:

```sh
oci://ghcr.io/<owner>/charts/claude-code-throwaway
```

Pull or install it like this:

```sh
helm pull oci://ghcr.io/<owner>/charts/claude-code-throwaway --version 0.1.0
helm install claude oci://ghcr.io/<owner>/charts/claude-code-throwaway --version 0.1.0
```

Notes:

- the workflow uses the built-in `GITHUB_TOKEN`
- repository Actions permissions must allow package write access
- chart version comes from `Chart.yaml`

## Caveats

- This does not magically make Claude Code safe. It only gives you better isolation.
- A strict standard Kubernetes NetworkPolicy cannot express hostname allowlists. For that you likely want Cilium FQDN policy or external egress filtering.
- The chart defaults to `sleep infinity` because different Claude Code workflows expose different ports and entrypoints.
