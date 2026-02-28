# Bootstrap — keyless CI/CD

Bootstrap to najtrudniejsza część projektu koncepcyjnie — musisz zbudować maszynę, która zbuduje wszystko inne, zanim ta maszyna istnieje. Robimy to raz, lokalnie, i nigdy więcej do tego nie wracamy.

---

## Problem: jak CI/CD uwierzytelnia się do GCP?

Tradycyjne podejście: wygeneruj klucz Service Account (JSON), wrzuć do GitHub Secrets, użyj w Actions.

**Dlaczego to złe:**

- Klucz ma ważność do ręcznego usunięcia — jeśli wycieknie (GitHub breach, logi, commit), atakujący ma dostęp przez lata
- Rotacja kluczy jest ręczna i często pomijana
- JSON key to wektor ataku #1 w GCP według Google Security

**Nasze podejście: Workload Identity Federation (WIF)**

```
GitHub Actions Runner
        │
        │ 1. prosi GitHub o OIDC token (JWT)
        ▼
GitHub OIDC Provider
        │ 2. wystawia token z claims: repo, branch, sha
        ▼
Google Cloud STS
        │ 3. weryfikuje token (Google ufa GitHub OIDC)
        │ 4. sprawdza attribute_condition: repository == 'benhornbeam/repo'
        ▼
Service Account Token (krótkoterminowy, 1h)
        │ 5. tylko gh-infra-worker SA może być impersonowany
        ▼
Zasoby GCP (Terraform plan/apply)
```

Żaden klucz nigdzie nie istnieje. Token ważny 1 godzinę. Tylko ten repo może impersonować ten SA.

---

## Co robi bootstrap

Dwa etapy: skrypt bash + warstwa Terraform.

### Etap 1: `bob_budowniczy.sh` (bash, lokalnie)

7-etapowy idempotentny skrypt. "Idempotentny" = możesz uruchomić wielokrotnie bez skutków ubocznych — każdy krok sprawdza czy zasób już istnieje.

```bash
# Co robi skrypt w kolejności:
# 1. Tworzy prywatne repo na GitHubie
# 2. Tworzy projekt GCP w organizacji
# 3. Podpina billing account
# 4. Aktywuje ~10 wymaganych API (IAM, Cloud Run, Artifact Registry, ...)
# 5. Tworzy Bootstrap SA (tf-bootstrap-admin) z minimalnymi uprawnieniami
# 6. Tworzy GCS bucket na stan Terraform
# 7. Uruchamia `terraform plan` dla warstwy bootstrap
```

```bash
# Uruchomienie:
cd scripts && ./run.sh
# run.sh auto-generuje project ID w formacie gcp-prototype-1-YYYYMMDD
# i przekazuje do bob_budowniczy.sh
```

!!! warning "Pułapka: IAM propagation delay"
    Po nadaniu roli SA, GCP potrzebuje ~30 sekund na propagację. Skrypt ma wbudowane `sleep 30` po kluczowych operacjach IAM. Jeśli Terraform plan rzuca `Permission denied` chwilę po nadaniu roli — poczekaj minutę i spróbuj ponownie.

### Etap 2: `tf/bootstrap/` (Terraform, lokalnie)

Tworzy wszystko, co CI/CD potrzebuje do działania.

```hcl
# tf/bootstrap/main.tf — uproszczony

# Service Account dla GitHub Actions
resource "google_service_account" "github_sa" {
  account_id   = "gh-infra-worker"
  display_name = "GitHub Actions Infra Worker"
  project      = var.project_id
}

# Role — principle of least privilege
resource "google_project_iam_member" "worker_roles" {
  for_each = toset([
    "roles/compute.networkAdmin",
    "roles/storage.admin",
    "roles/run.admin",
    "roles/artifactregistry.admin",
    "roles/iam.securityAdmin",        # (1)
    "roles/secretmanager.admin",
    "roles/apigateway.admin",
    "roles/monitoring.admin",
    # ... (pełna lista w tf/bootstrap/main.tf)
  ])
  project = var.project_id
  role    = each.key
  member  = "serviceAccount:${google_service_account.github_sa.email}"
}
```

1. `iam.securityAdmin` zamiast `projectIamAdmin` — ma `setIamPolicy`, ale węższy zakres. Świadoma decyzja: używamy węższej roli gdy tylko możliwe.

```hcl
# WIF Pool — rejestruje GitHub jako zaufany issuer OIDC
resource "google_iam_workload_identity_pool" "github_pool" {
  workload_identity_pool_id = "github-actions-pool-v3"
}

resource "google_iam_workload_identity_pool_provider" "github_provider" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github_pool.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-provider"

  attribute_mapping = {
    "google.subject"       = "assertion.sub"        # (1)
    "attribute.repository" = "assertion.repository" # (2)
  }

  # Tylko ten repo może impersonować SA — bezpieczeństwo przez warunek
  attribute_condition = "assertion.repository == '${var.github_repo}'"

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }
}
```

1. `sub` = unikalny identyfikator workflow w GitHubie
2. Mapujemy `repository` z tokenu GitHub → atrybut Google, żeby użyć go w `attribute_condition`

```hcl
# Binding: GitHub Actions Runner może impersonować gh-infra-worker
resource "google_service_account_iam_member" "wif_binding" {
  service_account_id = google_service_account.github_sa.name
  role               = "roles/iam.workloadIdentityUser"
  member = "principalSet://iam.googleapis.com/${
    google_iam_workload_identity_pool.github_pool.name
  }/attribute.repository/${var.github_repo}"
}
```

---

## Jak działa w GitHub Actions

```yaml
# .github/workflows/deploy.yml — fragment

permissions:
  id-token: write   # (1) wymagane — bez tego Actions nie może pobrać OIDC token
  contents: read

steps:
  - name: Authenticate to Google Cloud
    uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: ${{ secrets.GCP_WIF_PROVIDER }}  # (2)
      service_account: ${{ secrets.GCP_SA_EMAIL }}                 # (3)
```

1. `id-token: write` — kluczowa linia. Bez niej GitHub nie udostępni OIDC tokenu jobowi.
2. Format: `projects/NUMBER/locations/global/workloadIdentityPools/POOL/providers/PROVIDER`
3. Format: `gh-infra-worker@PROJECT_ID.iam.gserviceaccount.com`

Akcja `google-github-actions/auth` robi całą wymianę tokenu automatycznie. Po tym kroku `gcloud` i Terraform mają dostęp przez ADC (Application Default Credentials).

---

## Stan Terraform: separacja warstw

Jeden bucket GCS, osobny prefiks per warstwa:

```
tf-state-gcp-prototype-1-20260224/
├── terraform/bootstrap/state/
├── terraform/infra/state/
├── terraform/backend/state/
├── terraform/auth/state/
├── terraform/api-gateway/state/
├── terraform/database/state/
├── terraform/frontend/state/
└── terraform/monitoring/state/
```

```hcl
# tf/backend/provider.tf — przykład konfiguracji backendu
terraform {
  backend "gcs" {
    # wartości przekazywane przez -backend-config (nie hardkodujemy project ID)
    # bucket = "tf-state-gcp-prototype-1-20260224"
    # prefix = "terraform/backend/state"
  }
}
```

```bash
# Init z dynamicznym konfiguracją
terraform init \
  -backend-config="bucket=tf-state-${PROJECT_ID}" \
  -backend-config="prefix=terraform/backend/state"
```

!!! tip "Dlaczego osobne prefiksy, nie osobne buckety?"
    Jeden bucket = jeden zasób IAM do zarządzania, jeden zasób TF w bootstrapie, jedna polityka lifecycle. Osobne prefiksy = izolacja stanów (Terraform nie widzi stanu innych warstw). Best of both worlds.

---

## Pułapki

!!! danger "tf/bootstrap/ NIE deployuj przez CI/CD"
    Warstwa bootstrap tworzy SA i WIF — czyli infrastrukturę, którą CI/CD używa do uwierzytelnienia. Jeśli deployujesz bootstrap przez CI/CD, które zależy od bootstrapu → błędne koło. Bootstrap zawsze **lokalnie**, przez `tf-bootstrap-admin` SA z impersonacją lub bezpośrednio przez ADC jako owner projektu.

!!! danger "Nowa rola dla gh-infra-worker → najpierw apply bootstrap"
    Jeśli nowa warstwa TF potrzebuje roli, której `gh-infra-worker` jeszcze nie ma (np. `monitoring.admin` dla warstwy monitoring):
    1. Dodaj rolę do `tf/bootstrap/main.tf`
    2. `terraform apply` lokalnie dla bootstrapu
    3. Dopiero potem deployuj nową warstwę przez CI

    Jeśli tego nie zrobisz, workflow wywali się na `Permission denied` bez czytelnego komunikatu.

!!! warning "State bucket: public_access_prevention = enforced"
    Bucket ze stanem TF musi mieć `public_access_prevention = "enforced"`. Stan Terraform może zawierać wrażliwe informacje (endpointy, SA emails). Domyślnie GCS bucket jest prywatny, ale warto to explicite ustawić i sprawdzić w security review.

---

## Weryfikacja

Po wykonaniu bootstrapu:

```bash
# Sprawdź czy WIF pool istnieje
gcloud iam workload-identity-pools list \
  --location=global \
  --project=gcp-prototype-1-20260224

# Sprawdź role gh-infra-worker
gcloud projects get-iam-policy gcp-prototype-1-20260224 \
  --flatten="bindings[].members" \
  --filter="bindings.members:gh-infra-worker" \
  --format="table(bindings.role)"

# Sprawdź GitHub Secrets (powinny być ustawione)
gh secret list --repo benhornbeam/gcp-prototype-1-20260224
# Oczekiwane: GCP_PROJECT_ID, GCP_WIF_PROVIDER, GCP_SA_EMAIL
```

Jeśli wszystko OK — masz CI/CD bez kluczy, gotowy do deployowania kolejnych warstw.
