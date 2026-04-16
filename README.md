# musicguessr-kustomize

Kubernetes manifests for [musicguessr](https://musicguessr.app), managed with [Kustomize](https://kustomize.io/).

## Prerequisites

- Kubernetes cluster with [Traefik](https://traefik.io/) as ingress controller
- [cert-manager](https://cert-manager.io/) with a `letsencrypt-prod` ClusterIssuer
- `kubectl` with Kustomize support (`kubectl version` >= 1.14)

## Setup

**1. Create your secrets file**

```sh
cp k8s/base/secrets.env.example k8s/base/secrets.env
```

Edit `k8s/base/secrets.env` and fill in your values:

```
APPLE_DEV_TOKEN=your-apple-music-developer-token
```

**2. Set your Spotify Client ID**

Edit `k8s/base/kustomization.yaml` and update:

```yaml
- SPOTIFY_CLIENT_ID=your-spotify-client-id
```

**3. (Optional) Pin image tags**

By default both images use `latest`. To deploy a specific version, edit the `images` section in `k8s/base/kustomization.yaml`:

```yaml
images:
  - name: ghcr.io/musicguessr/musicguessr-backend
    newTag: "1.2.3"
  - name: ghcr.io/musicguessr/musicguessr-frontend
    newTag: "1.2.3"
```

Or override at deploy time without editing any file (e.g. in CI):

```sh
cd k8s/base
kustomize edit set image \
  ghcr.io/musicguessr/musicguessr-backend:${BACKEND_TAG} \
  ghcr.io/musicguessr/musicguessr-frontend:${FRONTEND_TAG}
kubectl apply -k .
```

**4. Deploy**

```sh
kubectl apply -k k8s/base/
```

## Structure

```
k8s/base/
├── kustomization.yaml       # Kustomize entrypoint, ConfigMap/Secret generators
├── namespace.yaml           # musicguessr namespace
├── backend-deployment.yaml  # Backend (Go) deployment
├── backend-service.yaml     # Backend ClusterIP service
├── frontend-deployment.yaml # Frontend (nginx) deployment
├── frontend-service.yaml    # Frontend ClusterIP service
├── ingress.yaml             # Traefik ingress with TLS via cert-manager
└── secrets.env.example      # Template for secrets.env (gitignored)
```

## Configuration

### Frontend

| Variable | Location | Description |
|---|---|---|
| `SPOTIFY_CLIENT_ID` | `kustomization.yaml` | (optional) Spotify app client ID. If empty, Spotify option is hidden. |
| `APPLE_DEV_TOKEN` | `secrets.env` | (optional) Apple MusicKit developer JWT. If empty, Apple Music option is hidden. |

### Backend

| Variable | Location | Description |
|---|---|---|
| `PORT` | `kustomization.yaml` | TCP port the HTTP server listens on. Default: `8080` |
| `LOG_LEVEL` | `kustomization.yaml` | Log level (`info`, `debug`, `warn`). Default: `info` |
| `INVIDIOUS_INSTANCES` | `kustomization.yaml` | Comma-separated Invidious instance URLs for YouTube lookups |
| `METADATA_CACHE_TTL_SECONDS` | `kustomization.yaml` | TTL for in-memory metadata cache in seconds. Default: `86400` (24h) |
| `THEAUDIODB_KEY` | `kustomization.yaml` | TheAudioDB API key. Default: `1` (public key) |
| `DISCOGS_TOKEN` | `secrets.env` | (optional) Discogs API token. If set, enables the Discogs metadata provider. |
