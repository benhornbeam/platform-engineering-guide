# (Opcjonalna) Google LB + Cloud CDN + Cloud Armor

Warstwa edukacyjna do nauki Google Global Load Balancer, Cloud CDN i Cloud Armor — bez zastępowania głównej architektury Cloudflare. Tworzy niezależny endpoint `app-lb.kamilos.xyz`.

---

## Po co ta warstwa?

Cloudflare Free ($0/mies) jest lepszym wyborem ekonomicznym dla frontendu (ADR-005). Ale Google LB to wiedza wymagana na certyfikatach GCP i na rozmowach o architekturze. Ta warstwa pozwala wdrożyć i przebadać Google LB bez ryzyka dla produkcji — i usunąć ją jedną komendą.

```
app.kamilos.xyz     ← prod (Cloudflare CDN → GCS) — zawsze działa
app-lb.kamilos.xyz  ← edukacja (Google LB → GCS) — opcjonalny, $23-28/mies
```

---

## Architektura warstwy

```
Browser
  │ HTTPS (DNS A → 34.50.151.193)
  ▼
Cloud Armor Edge (geoblocking, IP filtering)
  │
  ▼
Google Global HTTPS Load Balancer
  │ EXTERNAL_MANAGED
  ▼
Cloud CDN (USE_ORIGIN_HEADERS)
  │
  ▼
GCS bucket: app-lb.kamilos.xyz
(te same pliki co app.kamilos.xyz)
```

**HTTP→HTTPS redirect** jako osobna reguła (url_map + http_proxy + forwarding_rule).

---

## Terraform — zasoby w kolejności zależności

```hcl
# tf/frontend-lb/main.tf

# 1. Statyczny globalny IP
resource "google_compute_global_address" "frontend_lb" {
  name    = "frontend-lb-ip"
  project = var.project_id
}

# 2. Cloud Armor Edge policy
resource "google_compute_security_policy" "frontend_lb" {
  name    = "frontend-lb-armor"
  project = var.project_id
  type    = "CLOUD_ARMOR_EDGE"   # (1)

  rule {
    action   = "allow"
    priority = 1000
    match {
      versioned_expr = "SRC_IPS_V1"
      config { src_ip_ranges = ["*"] }
    }
  }
  rule {
    action   = "deny(403)"
    priority = 2147483647
    match {
      versioned_expr = "SRC_IPS_V1"
      config { src_ip_ranges = ["*"] }
    }
  }
}

# 3. GCS bucket jako origin
resource "google_storage_bucket" "frontend_lb" {
  name     = "app-lb.kamilos.xyz"
  location = "EU"

  website {
    main_page_suffix = "index.html"
    not_found_page   = "index.html"
  }
}

resource "google_storage_bucket_iam_member" "public_read" {
  bucket = google_storage_bucket.frontend_lb.name
  role   = "roles/storage.objectViewer"
  member = "allUsers"
}

# 4. Backend bucket (łączy GCS + CDN + Cloud Armor)
resource "google_compute_backend_bucket" "frontend_lb" {
  name        = "frontend-lb-backend"
  bucket_name = google_storage_bucket.frontend_lb.name
  enable_cdn  = true

  cdn_policy {
    cache_mode = "USE_ORIGIN_HEADERS"   # (2)
  }

  edge_security_policy = google_compute_security_policy.frontend_lb.self_link
}

# 5. URL map (routing → backend bucket)
resource "google_compute_url_map" "frontend_lb" {
  name            = "frontend-lb-url-map"
  default_service = google_compute_backend_bucket.frontend_lb.self_link
}

# 6. Managed SSL certificate
resource "google_compute_managed_ssl_certificate" "frontend_lb" {
  name    = "frontend-lb-ssl"
  managed { domains = ["app-lb.kamilos.xyz"] }   # (3)
}

# 7. HTTPS proxy
resource "google_compute_target_https_proxy" "frontend_lb" {
  name             = "frontend-lb-https-proxy"
  url_map          = google_compute_url_map.frontend_lb.self_link
  ssl_certificates = [google_compute_managed_ssl_certificate.frontend_lb.self_link]
}

# 8. Global forwarding rule (public IP:443 → HTTPS proxy)
resource "google_compute_global_forwarding_rule" "frontend_lb_https" {
  name       = "frontend-lb-https"
  target     = google_compute_target_https_proxy.frontend_lb.self_link
  port_range = "443"
  ip_address = google_compute_global_address.frontend_lb.address
}

# 9. HTTP→HTTPS redirect (osobny url_map)
resource "google_compute_url_map" "http_redirect" {
  name = "frontend-lb-http-redirect"
  default_url_redirect {
    https_redirect         = true
    redirect_response_code = "MOVED_PERMANENTLY_DEFAULT"
    strip_query            = false
  }
}
# + google_compute_target_http_proxy + google_compute_global_forwarding_rule (port 80)
```

1. `CLOUD_ARMOR_EDGE` — jedyny typ obsługiwany przez `backend_bucket`. Obsługuje geoblocking i IP filtering. **Nie obsługuje** rate limiting, WAF (XSS/SQLi), reCAPTCHA — te funkcje wymagają `CLOUD_ARMOR` + `backend_service`.
2. `USE_ORIGIN_HEADERS` — CDN respektuje `Cache-Control` z GCS. `index.html: no-cache`, `app.js: max-age=3600`.
3. Google automatycznie provisionuje certyfikat TLS po DNS propagacji (10-30 min).

---

## Konfiguracja DNS

!!! danger "Grey cloud w Cloudflare — obowiązkowo"
    `app-lb.kamilos.xyz` musi wskazywać bezpośrednio na IP LB (DNS A record, proxy OFF).
    Z Cloudflare proxy (orange cloud): dwa SSL handshaki, Google managed cert nigdy nie zostanie wydany.

```
Type   Name      Content          Proxy status
A      app-lb    34.50.151.193    DNS only (grey cloud) ✓
```

Po dodaniu A rekordu poczekaj na SSL certificate provisioning:

```bash
gcloud compute ssl-certificates describe frontend-lb-ssl \
  --project=gcp-prototype-1-20260224 \
  --format="value(managed.status,managed.domainStatus)"
# PROVISIONING → ACTIVE (~10-30 min po propagacji DNS)
```

---

## Deploy i zarządzanie

### Wdrożenie

```bash
# Przez GitHub Actions
gh workflow run deploy.yml \
  -f action=apply \
  -f layer=frontend-lb \
  --repo benhornbeam/gcp-prototype-1-20260224

# Po apply: dodaj DNS A record, poczekaj na SSL
# Deploy plików frontendu (oddzielnie)
gcloud storage cp frontend/index.html gs://app-lb.kamilos.xyz/ \
  --cache-control="no-cache, no-store, must-revalidate"
gcloud storage cp frontend/app.js gs://app-lb.kamilos.xyz/ \
  --cache-control="public, max-age=3600"
```

### Auto-deploy przez CI/CD

`deploy-frontend.yml` ma kroki dla obu bucketów. LB bucket z `continue-on-error: true`:

```yaml
- name: Deploy to prod — LB bucket (opcjonalny)
  if: github.ref_name == 'master'
  continue-on-error: true   # nie fail jeśli warstwa nie istnieje
  run: |
    gcloud storage cp frontend/index.html gs://app-lb.kamilos.xyz/ \
      --cache-control="no-cache, no-store, must-revalidate"
    gcloud storage cp frontend/app.js gs://app-lb.kamilos.xyz/ \
      --cache-control="public, max-age=3600"
```

### Usunięcie

```bash
gh workflow run deploy.yml \
  -f action=apply \
  -f layer=frontend-lb \
  --repo benhornbeam/gcp-prototype-1-20260224
# lub lokalnie:
cd tf/frontend-lb && terraform destroy -var project_id=gcp-prototype-1-20260224
```

Po `destroy`: usuń A record z Cloudflare DNS.

---

## Checkov — świadome wyjątki

```yaml
# .checkov.yaml
- id: "CKV2_GCP_12"
  comment: "LB access logs — pominięte dla prototypu; koszt vs wartość"
- id: "CKV_GCP_78"
  comment: "Cloud Armor Edge nie obsługuje WAF — tylko CLOUD_ARMOR_EDGE dostępny dla backend_bucket"
```

---

## Cloud Armor Edge vs Standard

| | Cloud Armor Edge | Cloud Armor Standard |
|--|-----------------|---------------------|
| Typ policy | `CLOUD_ARMOR_EDGE` | `CLOUD_ARMOR` |
| Dołączany do | `backend_bucket` | `backend_service` |
| Geoblocking | ✅ | ✅ |
| IP filtering | ✅ | ✅ |
| Rate limiting | ❌ | ✅ |
| WAF (XSS, SQLi) | ❌ | ✅ |
| reCAPTCHA | ❌ | ✅ |
| Koszt | Brak dodatkowego kosztu | $5/mies minimum |
| Użycie | Statyczny hosting (GCS) | Cloud Run, GCE, GKE |

**Wniosek:** jeśli chcesz chronić backend API (Cloud Run) przez Google LB — użyj `backend_service` + `CLOUD_ARMOR` Standard. Edge nie da WAF ani rate limiting.

---

## Koszt warstwy

| Komponent | Koszt/mies |
|-----------|-----------|
| Global Forwarding Rules (2×) | ~$18 |
| Cloud CDN (minimal traffic) | ~$2 |
| Cloud Armor Edge | $0 (wbudowany w LB) |
| GCS bucket | ~$0.05 |
| **Łącznie** | **~$20-28/mies** |

Pokrywane przez $300 GCP Free Trial (90 dni). Po `terraform destroy` — zero kosztów.

---

## Pułapki

!!! danger "Grey cloud — inaczej SSL nie zadziała"
    Google managed certificate wymaga bezpośredniego połączenia do LB IP. Cloudflare proxy przerywa ten handshake.

!!! warning "Cloud Armor Edge nie zastąpi WAF"
    `CLOUD_ARMOR_EDGE` na `backend_bucket` to tylko geoblocking/IP. Dla pełnego WAF (XSS, SQLi) — `backend_service` + Cloud Run za LB + `CLOUD_ARMOR` Standard.

!!! tip "Weryfikacja CDN"
    ```bash
    curl -sI https://app-lb.kamilos.xyz/ | grep -i "cache\|x-goog\|age"
    # x-cache: HIT — CDN działa
    ```
