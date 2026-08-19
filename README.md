# element-web

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Element](https://img.shields.io/badge/client-Element_Web-0dbd8b.svg)](https://element.io)
[![URL](https://img.shields.io/badge/url-element.kudofools.dev-purple.svg)](https://element.kudofools.dev)
[![Image](https://img.shields.io/badge/image-v1.12.26-2ea44f.svg)](https://hub.docker.com/r/vectorim/element-web)

Flux-managed self-hosted [Element Web](https://element.io) Matrix web client on the kudofools k3s cluster.

## Features

- Served at `element.kudofools.dev`
- Points at the Conduit homeserver (`conduit.kudofools.dev`)
- Custom homeserver URLs disabled (`disable_custom_urls: true`) — only your homeserver
- Runs with read-only root filesystem; nginx writable paths on emptyDirs

## Structure

```
clusters/default/
├── namespace.yaml       # element-web namespace
├── config.yaml          # config.json -> points at conduit.kudofools.dev
├── externalsecret.yaml  # webhook token from OpenBao
├── deployment.yaml      # vectorim/element-web:v1.12.26 (nginx, port 80)
├── service.yaml
└── ingress.yaml         # element.kudofools.dev -> element:80
```

## Configuration

| Setting | Value |
|---|---|
| Image | `vectorim/element-web:v1.12.26` |
| Homeserver `base_url` | `https://conduit.kudofools.dev` |
| Homeserver `server_name` | `conduit.kudofools.dev` |
| `disable_custom_urls` | `true` |
| `disable_guests` | `true` |

Config lives in `clusters/default/config.yaml` (mounted at `/app/config.json`).

## Prerequisites

- kudofools cluster with Flux, OpenBao, ESO, and Cloudflare tunnel
- A running Conduit homeserver (see [matrix-conduit](https://forgejo.kudofools.dev/izayoilv/matrix-conduit)) and a registered account

## Setup

The cluster-side plumbing (DNS, tunnel route, Flux resources) lives in kudofools-infra. On the RPi5:

```bash
# 1. Write the webhook token to OpenBao
ROOT_TOKEN=$(jq -r '.root_token' ~/.bao-keys.json)
ELEMENT_TOKEN=$(openssl rand -base64 32)
kubectl exec -n openbao openbao-0 -- env BAO_TOKEN=$ROOT_TOKEN bao kv put kv/element-web/webhook-token token=$ELEMENT_TOKEN

# 2. Force ESO to sync it into the cluster
kubectl annotate externalsecret -n flux-system element-web-webhook force-sync=$(date +%s) --overwrite

# 3. Get the webhook path
echo "element-web internal: http://webhook-receiver.flux-system.svc.cluster.local:80$(kubectl get receiver -n flux-system element-web-webhook -o jsonpath='{.status.webhookPath}')"
echo "element-web public:   https://flux-webhook.kudofools.dev$(kubectl get receiver -n flux-system element-web-webhook -o jsonpath='{.status.webhookPath}')"

# 4. Verify it's live
kubectl get pods -n element-web
curl -s https://element.kudofools.dev/config.json
```

Then configure the Forgejo webhook in the **element-web repo → Settings → Webhooks → Add Webhook** (select **Forgejo**) using the token above and the receiver path. Flux validates via `X-Hub-Signature` HMAC.

Open https://element.kudofools.dev and log in with an account on `conduit.kudofools.dev`.

## Operations

| Task | Command |
|---|---|
| Update Element version | bump `image:` in `clusters/default/deployment.yaml`, push |
| Check health | `kubectl get pods -n element-web` |
| View logs | `kubectl logs -n element-web deploy/element` |

## License

[MIT](LICENSE) © 2026 IzayoiLv
