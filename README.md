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

Edit `k8s/base/secrets.env` and fill in your values (all fields are optional unless S3 is enabled):

```env
APPLE_DEV_TOKEN=          # Apple MusicKit developer JWT (required for Apple Music provider)
DISCOGS_TOKEN=            # Discogs API token (optional — improves metadata quality)
DECK_STORAGE_ACCESS_KEY_ID=     # S3 access key (required when DECK_STORAGE_PROVIDER=s3)
DECK_STORAGE_SECRET_ACCESS_KEY= # S3 secret key (required when DECK_STORAGE_PROVIDER=s3)
```

**2. Edit `k8s/base/kustomization.yaml`**

Set the variables relevant to your deployment:

```yaml
# Frontend
- API_URL=https://your-domain.example.com        # backend URL as seen from browser
- SITE_URL=https://your-domain.example.com       # public frontend URL (canonical, OG tags)
- SPOTIFY_CLIENT_ID=your-spotify-client-id       # optional — hides Spotify if empty
- GOOGLE_SITE_VERIFICATION=your-token            # optional — Google Search Console
- BING_SITE_VERIFICATION=your-token              # optional — Bing Webmaster Tools

# Backend
- FRONTEND_URL=https://your-domain.example.com   # used in deck share_url response
- DECK_STORAGE_PROVIDER=s3                       # local (default) or s3
- DECK_STORAGE_ENDPOINT=https://...              # required for s3 (R2, AWS, MinIO, etc.)
- DECK_STORAGE_BUCKET=your-bucket
- DECK_STORAGE_REGION=auto                       # auto for Cloudflare R2, region name for AWS
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
├── secrets.env.example      # Template for secrets.env (gitignored)
└── secrets.env              # Your secrets — never commit this file
```

## Configuration reference

### Frontend (`musicguessr-frontend-config` ConfigMap)

| Variable | Description |
|---|---|
| `API_URL` | Backend URL as seen from the user's browser. Used by Angular at runtime. |
| `SITE_URL` | Public frontend URL — canonical tags, OG meta, sitemap. |
| `SPOTIFY_CLIENT_ID` | Spotify app client ID. If empty, Spotify option is hidden. |
| `GOOGLE_SITE_VERIFICATION` | Google Search Console verification token (injected into `<meta>`). |
| `BING_SITE_VERIFICATION` | Bing Webmaster Tools verification token (injected into `<meta>`). |

### Frontend secrets (`musicguessr-secrets` Secret)

| Variable | Description |
|---|---|
| `APPLE_DEV_TOKEN` | Apple MusicKit developer JWT (signed with Apple private key, valid 6 months). If empty, Apple Music option is hidden. |

### Backend (`musicguessr-backend-config` ConfigMap)

| Variable | Default | Description |
|---|---|---|
| `PORT` | `8080` | TCP port the HTTP server listens on. |
| `LOG_LEVEL` | `info` | Log verbosity: `info`, `debug`, `warn`. |
| `INVIDIOUS_INSTANCES` | see file | Comma-separated Invidious instance URLs for YouTube lookups. |
| `METADATA_CACHE_TTL_SECONDS` | `86400` | In-memory metadata cache TTL in seconds (24h). |
| `THEAUDIODB_KEY` | `1` | TheAudioDB API key. `1` is the public key. |
| `FRONTEND_URL` | `https://musicguessr.app` | Public frontend URL — used to build `share_url` in deck API responses. |
| `DECK_STORAGE_PROVIDER` | `local` | Storage backend: `local` (filesystem) or `s3` (S3-compatible). |
| `DECK_STORAGE_PATH` | `./data/decks` | Local filesystem path for decks (only used when `DECK_STORAGE_PROVIDER=local`). |
| `DECK_STORAGE_ENDPOINT` | *(empty)* | S3 endpoint URL. **Required** when provider is `s3`. Compatible with Cloudflare R2, AWS S3, MinIO, OVH Object Storage, etc. |
| `DECK_STORAGE_BUCKET` | *(empty)* | S3 bucket name. Required when provider is `s3`. |
| `DECK_STORAGE_REGION` | `auto` | S3 region. Use `auto` for Cloudflare R2, or a region name (e.g. `eu-central-1`) for AWS. |

### Backend secrets (`musicguessr-secrets` Secret)

| Variable | Description |
|---|---|
| `DISCOGS_TOKEN` | Discogs API personal access token. Optional — enables Discogs as a metadata source. |
| `DECK_STORAGE_ACCESS_KEY_ID` | S3 access key ID. Required when `DECK_STORAGE_PROVIDER=s3`. |
| `DECK_STORAGE_SECRET_ACCESS_KEY` | S3 secret access key. Required when `DECK_STORAGE_PROVIDER=s3`. |
