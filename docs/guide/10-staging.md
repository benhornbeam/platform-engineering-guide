# Staging — multi-env w jednym projekcie

Staging to oddzielne środowisko testowe — takie same zasoby co prod, ale z sufiksem `-staging`. Jeden projekt GCP, jeden Firestore, różne dane.

---

## Dlaczego jeden projekt, nie dwa?

Szczegóły w [ADR-007](https://github.com/benhornbeam/gcp-prototype-1-20260224/blob/master/docs/adr/ADR-007.md). Skrót:

| Kryterium | Dwa projekty | Jeden projekt (nasze) |
|-----------|-------------|----------------------|
| Izolacja danych | Pełna | Przez prefix kolekcji Firestore |
| Koszt | 2× VPC Connector (~$14/mies) | 1× VPC Connector (~$7/mies) |
| IAM/billing | Oddzielny dla każdego env | Wspólny |
| Złożoność TF | Oddzielne state bucket, providery | Jeden state bucket, prefiksy |
| Wystarczające dla | Compliance-heavy produkcji | Prototyp, wczesna faza produktu |

---

## Wzorzec suffix `-staging`

Każdy zasób staging to kopia prod z sufiksem:

| Zasób | Prod | Staging |
|-------|------|---------|
| Cloud Run | `backend-api` | `backend-api-staging` |
| API Gateway | `gpc-gateway` | `gpc-gateway-staging` |
| API Gateway SA | `api-gateway-sa` | `api-gateway-staging-sa` |
| GCS bucket | `app.kamilos.xyz` | `staging.kamilos.xyz` |
| Firestore kolekcja | `logins/{uid}/events` | `staging_logins/{uid}/events` |
| URL | `https://app.kamilos.xyz` | `https://staging.kamilos.xyz` |

---

## Izolacja przez zmienną środowiskową

```hcl
# tf/backend/main.tf — Cloud Run staging
resource "google_cloud_run_v2_service" "backend_staging" {
  name = "backend-api-staging"
  labels = { env = "staging" }

  template {
    containers {
      image = var.image
      env {
        name  = "ENV"
        value = "staging"   # (1)
      }
    }
  }
}
```

1. `ENV=staging` ustawiane przez Terraform przy tworzeniu Cloud Run. Ten sam obraz Docker — zachowanie zmienia się przez env var.

```python
# app/main.py
ENV = os.getenv("ENV", "prod")
COLLECTION_PREFIX = "staging_" if ENV == "staging" else ""

# Prod:    kolekcja "logins/{uid}/events"
# Staging: kolekcja "staging_logins/{uid}/events"
```

---

## Mapowanie branch → środowisko

```
master  → prod
develop → staging

push do app/ na master  → deploy do backend-api
push do app/ na develop → deploy do backend-api-staging

push do frontend/ na master  → GCS app.kamilos.xyz
push do frontend/ na develop → GCS staging.kamilos.xyz (z podmianą API_URL)
```

```yaml
# deploy-backend.yml — branch-aware deploy
env:
  SERVICE_NAME: ${{ github.ref_name == 'develop' && 'backend-api-staging' || 'backend-api' }}
```

```yaml
# deploy-frontend.yml — osobne steps per branch
- name: Deploy to prod (master)
  if: github.ref_name == 'master'
  run: gcloud storage cp ... gs://app.kamilos.xyz/

- name: Deploy to staging (develop)
  if: github.ref_name == 'develop'
  run: |
    # Pobierz URL staging API Gateway
    STAGING_HOSTNAME=$(gcloud api-gateway gateways describe gpc-gateway-staging ...)

    # Podmień URL w app.js (sed, bez commitowania)
    sed "s|const API_URL = '.*'|const API_URL = 'https://${STAGING_HOSTNAME}'|" \
      frontend/app.js > /tmp/app.staging.js

    gcloud storage cp /tmp/app.staging.js gs://staging.kamilos.xyz/app.js
```

---

## Workflow testowania

```
1. Stwórz feature branch z develop
   git checkout -b feature/my-feature develop

2. Zaimplementuj zmianę (app/ lub frontend/)

3. Otwórz PR do develop
   gh pr create --base develop

4. Po merge do develop — auto-deploy staging
   → https://staging.kamilos.xyz

5. Przetestuj na staging

6. Otwórz PR develop → master

7. Po merge do master — auto-deploy prod
   → https://app.kamilos.xyz
```

---

## Pułapki

!!! danger "Staging API Gateway URL — chicken-and-egg"
    `deploy-frontend.yml` dla staging pobiera URL staging API Gateway przez `gcloud api-gateway gateways describe`. Jeśli gateway staging nie istnieje jeszcze (nie był deployowany przez `deploy.yml`), workflow wywali się z `NOT_FOUND`. Kolejność: najpierw deploy `api-gateway` layer przez `deploy.yml`, potem push do develop.

!!! warning "Wspólny Cloud Run SA dla prod i staging"
    `cloud-run-backend-sa` jest używany przez oba Cloud Run services. Jeśli musisz dać staging SA mniej uprawnień — stwórz osobny SA. Dla prototypu: jedna SA dla obu środowisk jest OK.

!!! warning "Cloudflare — staging.kamilos.xyz wymaga orange cloud"
    Tak samo jak app.kamilos.xyz — Cloudflare proxy musi być ON (orange cloud) dla staging, żeby HTTPS działał. Grey cloud (DNS only) = GCS bez SSL na custom domenie = "not secure" w przeglądarce.

!!! tip "Labels dla zasobów staging"
    Wszystkie zasoby staging mają `labels = { env = "staging" }`. Pozwala to filtrować koszty i zasoby:
    ```bash
    # Wszystkie Cloud Run services staging
    gcloud run services list --filter="metadata.labels.env=staging"
    ```
