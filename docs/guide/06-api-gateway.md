# API Gateway — walidacja i routing

API Gateway to jedyna brama do backendu. Każde żądanie przechodzi przez nią — bez ważnego JWT, Cloud Run nie dostanie requestu. To tzw. "zero trust at the perimeter".

---

## Czym jest Google Cloud API Gateway

API Gateway to managed proxy opisany specyfikacją OpenAPI (Swagger 2.0). Każde żądanie:

1. Sprawdza JWT (podpis, issuer, audience, expiry)
2. Jeśli OK — przekazuje request do Cloud Run z OIDC tokenem SA
3. Jeśli nie — zwraca 401 bez dotykania backendu

```
[Klient]
    │ HTTPS + Authorization: Bearer <JWT użytkownika>
    ▼
[API Gateway — europe-west1]
    ├─ walidacja JWT (Identity Platform issuer)
    │  ✓ / ✗
    │   ↓ jeśli OK
    ├─ dodaje nagłówek Authorization: Bearer <OIDC token api-gateway-sa>
    ▼
[Cloud Run — europe-central2]
    ├─ sprawdza IAM: czy api-gateway-sa ma roles/run.invoker? ✓
    ▼
[FastAPI: decode JWT, logika biznesowa]
```

---

## OpenAPI spec — serce konfiguracji

```yaml
# tf/api-gateway/openapi.yaml.tpl
# Plik jest szablonem Terraform — ${variable} zastępowane przy terraform apply

swagger: "2.0"
info:
  title: "gpc-api"
  version: "1.0.0"

x-google-backend:
  address: "${cloud_run_url}"          # (1)
  jwt_audience: "${cloud_run_url}"     # (2)
  path_translation: APPEND_PATH_TO_ADDRESS

securityDefinitions:
  firebase:
    type: "oauth2"
    flow: "implicit"
    authorizationUrl: ""
    x-google-issuer:   "https://securetoken.google.com/${project_id}"
    x-google-jwks_uri: "https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com"
    x-google-audiences: "${project_id}"   # (3)

paths:
  /health:
    get:
      operationId: "health_check"
      security:
        - firebase: []                  # (4) JWT wymagany
      responses:
        "200":
          description: "OK"

  /api:
    options:
      operationId: "api_cors_preflight"
      security: []                      # (5) preflight BEZ JWT
      responses:
        "204":
          description: "CORS preflight"
    get:
      operationId: "api_list"
      security:
        - firebase: []
      responses:
        "200":
          description: "OK"
    post:
      operationId: "api_create"
      security:
        - firebase: []
      responses:
        "200":
          description: "OK"
```

1. `address` — URL Cloud Run (pobierany przez `data.google_cloud_run_v2_service.backend.uri`)
2. `jwt_audience` — audience dla OIDC tokenu do Cloud Run. Ten sam URL co adres backendu
3. `x-google-audiences` — audience dla JWT użytkownika (Firebase ID Token ma `aud = PROJECT_ID`)
4. `security: - firebase: []` — ta ścieżka wymaga JWT
5. `security: []` — CORS preflight (`OPTIONS`) nie ma nagłówka `Authorization`. Musi być bez security, inaczej API Gateway zwróci 401 na preflight i przeglądarka zablokuje cały request

---

## Terraform — warstwa `tf/api-gateway/`

### Trzy zasoby na każdy gateway (prod i staging)

```hcl
# API — kontener logiczny
resource "google_api_gateway_api" "api" {
  provider     = google-beta          # (1)
  api_id       = "gpc-api"
  display_name = "GPC API"
}

# Config — konkretna wersja OpenAPI spec
resource "google_api_gateway_api_config" "config" {
  provider      = google-beta
  api           = google_api_gateway_api.api.api_id
  api_config_id = "cfg-${substr(md5(local.openapi_spec), 0, 8)}"  # (2)

  openapi_documents {
    document {
      path     = "spec.yaml"
      contents = base64encode(local.openapi_spec)  # (3)
    }
  }

  gateway_config {
    backend_config {
      google_service_account = google_service_account.api_gw_sa.email
    }
  }

  lifecycle {
    create_before_destroy = true  # (4)
  }
}

# Gateway — publiczny endpoint
resource "google_api_gateway_gateway" "gateway" {
  provider   = google-beta
  api_config = google_api_gateway_api_config.config.id
  gateway_id = "gpc-gateway"
  region     = "europe-west1"    # (5)
}
```

1. `google-beta` — `google_api_gateway_*` zasoby są w beta providerze. Wymaga osobnej konfiguracji `provider "google-beta"` w `provider.tf`
2. `md5(local.openapi_spec)` — config ID zawiera hash specyfikacji. Gdy spec się zmienia → nowy config ID → nowy zasób → automatyczne rolowanie. Bez tego Terraform chciałby zmodyfikować immutable field i wywalał error
3. `base64encode` — API Gateway wymaga specyfikacji jako base64
4. `create_before_destroy` — stary config jest usuwany dopiero po stworzeniu nowego. Bez tego byłby downtime podczas aktualizacji spec
5. `europe-west1` — API Gateway nie obsługuje `europe-central2`. Stały region dla wszystkich gatewayów

### Templatefile — dynamiczny spec

```hcl
locals {
  openapi_spec = templatefile("${path.module}/openapi.yaml.tpl", {
    cloud_run_url = data.google_cloud_run_v2_service.backend.uri
    project_id    = var.project_id
  })
}
```

`data.google_cloud_run_v2_service.backend.uri` pobiera URL Cloud Run z istniejącego stanu — dlatego warstwa `backend` musi być wdrożona przed `api-gateway`.

### IAM: API Gateway → Cloud Run

```hcl
resource "google_service_account" "api_gw_sa" {
  account_id = "api-gateway-sa"
}

# SA może wywoływać Cloud Run
resource "google_cloud_run_v2_service_iam_member" "api_gw_invoker" {
  name   = data.google_cloud_run_v2_service.backend.name
  role   = "roles/run.invoker"
  member = "serviceAccount:${google_service_account.api_gw_sa.email}"
}
```

---

## Pułapki

!!! danger "CORS preflight musi być bez security"
    Browser przed każdym cross-origin requestem wysyła `OPTIONS` preflight bez nagłówków autoryzacji. Jeśli `OPTIONS /api` ma `security: - firebase: []`, API Gateway zwróci 401 → przeglądarka zablokuje właściwy request. Zawsze: `security: []` dla OPTIONS.

!!! danger "api_config_id jest immutable"
    Nie możesz zmodyfikować istniejącego API Config — musisz stworzyć nowy. Dlatego ID zawiera hash spec: `cfg-${substr(md5(spec), 0, 8)}`. Terraform automatycznie tworzy nowy config gdy spec się zmienia.

!!! warning "Zależność: backend musi być wdrożony przed api-gateway"
    `data.google_cloud_run_v2_service.backend` odpytuje istniejący Cloud Run. Jeśli backend nie istnieje → Terraform plan zwróci błąd. Kolejność deploy: backend → api-gateway.

!!! tip "google-beta provider"
    Dodaj osobną konfigurację providera w `provider.tf`:
    ```hcl
    provider "google-beta" {
      project = var.project_id
      region  = var.region
    }
    ```
    Terraform wymaga explicytnej konfiguracji `google-beta` — nie dziedziczy automatycznie z `google`.
