# Frontend — GCS + Cloudflare CDN

Hosting SPA (Single Page Application) za $0/mies. GCS bucket jako origin, Cloudflare jako CDN, SSL i DDoS protection — bez Google Load Balancera.

---

## Ewolucja architektury frontendu

```
Etap 1 (ADR-003): GCS bucket bezpośrednio
  URL: https://storage.googleapis.com/frontend-bucket/index.html
  ✓ $0 koszt
  ✗ brak custom domeny, brak HTTPS na własnej domenie

Etap 2 (ADR-005): Cloudflare CDN jako proxy przed GCS
  URL: https://app.kamilos.xyz
  ✓ custom domena z HTTPS
  ✓ CDN (PoP Warszawa/Frankfurt)
  ✓ DDoS protection L3/L4/L7
  ✓ WAF (managed rules)
  ✓ $0/mies (Cloudflare Free tier)
  ✗ Bucket musi się nazywać dokładnie jak domena (CNAME trick)
```

---

## Cloudflare CNAME trick — dlaczego bucket musi mieć nazwę domeny

GCS obsługuje custom domeny przez CNAME do `c.storage.googleapis.com`. Gdy request przychodzi na `app.kamilos.xyz`, GCS szuka bucketa o nazwie `app.kamilos.xyz`.

```
Browser → app.kamilos.xyz
    │ DNS
    ▼
Cloudflare Edge (proxy ON — orange cloud)
    │ origin request to storage.googleapis.com
    │ Host: app.kamilos.xyz
    ▼
GCS → szuka bucketa "app.kamilos.xyz"
    → serwuje pliki z tego bucketa
```

Bez Cloudflare: CNAME `app → c.storage.googleapis.com` — HTTP tylko (GCS nie terminuje SSL na custom domenach). Z Cloudflare proxy: SSL terminowany na edge Cloudflare, GCS dostaje HTTP od Cloudflare.

---

## Terraform — warstwa `tf/frontend/`

```hcl
# tf/frontend/main.tf

resource "google_storage_bucket" "frontend" {
  name                        = "app.kamilos.xyz"       # (1)
  project                     = var.project_id
  location                    = "EU"                     # (2)
  uniform_bucket_level_access = true
  public_access_prevention    = "unspecified"            # (3)
  force_destroy               = true

  website {
    main_page_suffix = "index.html"
    not_found_page   = "index.html"  # (4)
  }
}

resource "google_storage_bucket_iam_member" "public_read" {
  bucket = google_storage_bucket.frontend.name
  role   = "roles/storage.objectViewer"
  member = "allUsers"                                    # (5)
}
```

1. `name = "app.kamilos.xyz"` — nazwa bucketa = domena. Wymagane przez CNAME trick
2. `location = "EU"` — multi-region EU, tańszy niż single-region `europe-central2`, szybszy dla użytkowników w Europie
3. `public_access_prevention = "unspecified"` — musi być `unspecified` (nie `enforced`) żeby `allUsers objectViewer` zadziałało. Dla bucketów backendowych (stan TF): `enforced`
4. `not_found_page = "index.html"` — SPA routing. `GET /app/profile` → plik nie istnieje → GCS zwraca `index.html` z HTTP 404. JS router przejmuje → wyświetla właściwy widok. Uwaga: HTTP 404 zamiast 200 dla deep links — Cloudflare może cachować 404. Rozwiązanie: Page Rule lub Transform Rule w Cloudflare (redirect 404 → 200)
5. `allUsers objectViewer` — publiczny dostęp do **czytania obiektów** (pobieranie plików). Nie daje uprawnień do zarządzania bucketem

---

## Konfiguracja Cloudflare DNS

W panelu Cloudflare dla domeny `kamilos.xyz`:

```
Type   Name      Content                        Proxy status
CNAME  app       c.storage.googleapis.com       Proxied (orange cloud) ✓
CNAME  staging   c.storage.googleapis.com       Proxied (orange cloud) ✓
```

!!! warning "Proxy ON (orange) — obowiązkowo dla HTTPS"
    Przy proxy OFF (grey cloud): CNAME wskazuje bezpośrednio na GCS, który nie ma certyfikatu dla `app.kamilos.xyz` → przeglądarka pokazuje "not secure". Z proxy ON: Cloudflare terminuje SSL, GCS dostaje HTTP od Cloudflare edge.

---

## Deploy plików

### Ręczny deploy

```bash
# Prod
gcloud storage cp -r ./dist/* gs://app.kamilos.xyz/

# Ze strategią cache-control
gcloud storage cp frontend/index.html gs://app.kamilos.xyz/ \
  --cache-control="no-cache, no-store, must-revalidate"
gcloud storage cp frontend/app.js gs://app.kamilos.xyz/ \
  --cache-control="public, max-age=3600"
```

### Auto-deploy przez GitHub Actions

```yaml
# deploy-frontend.yml — fragment (prod)
- name: Deploy to prod (master)
  if: github.ref_name == 'master'
  run: |
    gcloud storage cp frontend/app.js \
      gs://app.kamilos.xyz/app.js \
      --cache-control="public, max-age=3600"
    gcloud storage cp frontend/index.html \
      gs://app.kamilos.xyz/index.html \
      --cache-control="no-cache, no-store, must-revalidate"
```

`index.html` z `no-cache` — przeglądarka zawsze pobiera świeżą wersję, która zawiera `<script src="app.js?v=XXX">`. JS i CSS z `max-age=3600` — 1h cache na edge i w przeglądarce.

---

## Aplikacja SPA — `frontend/app.js`

```javascript
// Konfiguracja Firebase (apiKey jest publiczny)
const FIREBASE_CONFIG = {
  apiKey:     'AIzaSyDB9HH...',
  authDomain: 'gcp-prototype-1-20260224.firebaseapp.com',
  projectId:  'gcp-prototype-1-20260224',
};

// URL API Gateway — hardkodowany dla prod
// Dla staging: nadpisywany przez `sed` w deploy-frontend.yml
const API_URL = 'https://gpc-gateway-31x6ik0l.ew.gateway.dev';

// Po zalogowaniu: zapisz logowanie + pobierz historię
async function recordAndShowLogins() {
  const token = await auth.currentUser.getIdToken();
  const headers = { Authorization: `Bearer ${token}` };

  await fetch(`${API_URL}/api`, { method: 'POST', headers });
  const res = await fetch(`${API_URL}/api`, { headers });
  const data = await res.json();
  renderLogins(data.last_logins ?? []);
}
```

---

## Pułapki

!!! danger "Cloudflare cache — po deployu wymagany hard refresh"
    `app.js` jest cachowany przez 1h na edge Cloudflare. Po deployowaniu nowej wersji: `Cmd+Shift+R` (hard refresh) lub poczekaj 1h. Dla automatycznego inwalidowania: Cloudflare Cache Purge przez API lub Page Rule z `Cache-Control: no-store` dla `index.html` (który ładuje `app.js` z nowym hash/version).

!!! warning "GCS nie serwuje HTTPS na custom domenach bezpośrednio"
    GCS ma certyfikat dla `*.storage.googleapis.com`, nie dla `app.kamilos.xyz`. Cloudflare proxy jest wymagany dla HTTPS na custom domenie. Alternatywa bez Cloudflare: Google Load Balancer z SSL certificate resource (~$23/mies).

!!! tip "SPA routing a HTTP 404"
    `not_found_page = "index.html"` zwraca status 404 z treścią `index.html`. Większość SPA routerów działa z tym poprawnie. Jeśli Cloudflare Page Cache cachuje 404 błędnie — dodaj Cloudflare Page Rule: `app.kamilos.xyz/*` → Cache Level: Bypass.
