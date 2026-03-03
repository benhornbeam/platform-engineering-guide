# CI/CD — GitHub Actions

Trzy workflowy, zero kluczy serwisowych. Każdy push do odpowiedniego brancha i ścieżki wyzwala automatyczny deploy.

---

## Mapa workflowów

```
.github/workflows/
├── deploy.yml           ← ręczny, wszystkie warstwy TF (plan/apply)
├── deploy-backend.yml   ← auto, push do app/ → pytest → Trivy → Cloud Run
├── deploy-frontend.yml  ← auto, push do frontend/ → GCS
├── security.yml         ← auto, push/PR → pip-audit + Trivy fs + Checkov
├── docs.yml             ← auto, push do docs/guide/ → GitHub Pages
├── sync-guide.yml       ← auto, push do docs/guide/ → publiczne repo
└── release-please.yml   ← auto, push do master → CHANGELOG + GitHub Release
```

| Workflow | Wyzwalacz | Co robi |
|---------|-----------|---------|
| `deploy.yml` | `workflow_dispatch` (ręczny) | terraform plan/apply dla wybranej warstwy |
| `deploy-backend.yml` | push `app/**` na master/develop | **pytest** → docker build → **Trivy** → gcloud run deploy |
| `deploy-frontend.yml` | push `frontend/**` na master/develop | gcloud storage cp → GCS bucket |
| `security.yml` | push `tf/**`, `app/**`, każdy PR | **pip-audit** + **Trivy fs** + **Checkov** |
| `docs.yml` | push `docs/guide/**` lub `mkdocs.yml` | mkdocs gh-deploy → GitHub Pages |
| `release-please.yml` | push na master | semantic versioning, auto CHANGELOG |

---

## Uwierzytelnianie bez kluczy — WIF

Każdy workflow zaczyna się tak samo:

```yaml
permissions:
  id-token: write   # (1) WYMAGANE — bez tego GitHub nie udostępni OIDC token
  contents: read

steps:
  - uses: actions/checkout@v4

  - name: Authenticate to Google Cloud
    uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: ${{ secrets.GCP_WIF_PROVIDER }}
      service_account: ${{ secrets.GCP_SA_EMAIL }}

  - name: Set up Cloud SDK
    uses: google-github-actions/setup-gcloud@v2
```

1. `id-token: write` — kluczowa linia na poziomie `permissions`. Bez niej GitHub odrzuci próbę pobrania OIDC token z komunikatem "Credentials could not be loaded".

Po tych krokach wszystkie narzędzia (`gcloud`, `terraform`, Docker) mają dostęp do GCP przez ADC.

---

## deploy.yml — ręczny deploy Terraform

```yaml
on:
  workflow_dispatch:
    inputs:
      action:
        type: choice
        options: [plan, apply]
        default: plan
      layer:
        type: choice
        options: [infra, backend, auth, api-gateway, database, frontend, frontend-lb, monitoring]

jobs:
  terraform:
    env:
      TF_VAR_project_id:                  ${{ secrets.GCP_PROJECT_ID }}
      TF_VAR_region:                      europe-central2
      TF_VAR_google_oauth_client_id:      ${{ secrets.GOOGLE_OAUTH_CLIENT_ID }}
      TF_VAR_google_oauth_client_secret:  ${{ secrets.GOOGLE_OAUTH_CLIENT_SECRET }}
      TF_VAR_alert_email:                 ${{ secrets.ALERT_EMAIL }}

    steps:
      # ... auth steps ...

      - name: Terraform Init
        working-directory: ./tf/${{ github.event.inputs.layer }}
        run: |
          terraform init \
            -backend-config="bucket=tf-state-${{ secrets.GCP_PROJECT_ID }}" \
            -backend-config="prefix=terraform/${{ github.event.inputs.layer }}/state"

      - name: Terraform Plan
        run: terraform plan -var="project_id=..." -out=tfplan

      # Specjalne kroki dla backend apply: AR → Docker → full apply
      - name: Bootstrap Artifact Registry
        if: inputs.action == 'apply' && inputs.layer == 'backend'
        run: terraform apply -auto-approve -target=google_artifact_registry_repository.backend_repo ...

      - name: Build and push Docker image
        if: inputs.action == 'apply' && inputs.layer == 'backend'
        run: |
          docker build -t "${IMAGE}:${{ github.sha }}" ./app
          docker push "${IMAGE}:${{ github.sha }}"

      # Auth: import Identity Platform config przed apply
      - name: Import Identity Platform config
        if: inputs.action == 'apply' && inputs.layer == 'auth'
        run: |
          terraform import ... google_identity_platform_config.default \
            projects/${{ secrets.GCP_PROJECT_ID }}/config || true

      - name: Terraform Apply
        if: inputs.action == 'apply'
        run: |
          if [ "${{ inputs.layer }}" = "backend" ] || [ "${{ inputs.layer }}" = "auth" ]; then
            terraform apply -auto-approve -var="project_id=..."  # (1)
          else
            terraform apply -auto-approve tfplan                  # (2)
          fi
```

1. Backend i auth: `apply` bez saved plan — stan zmienił się przez `-target apply` (backend) lub `import` (auth), więc saved tfplan byłby nieaktualny
2. Pozostałe warstwy: `apply tfplan` — deterministyczne, dokładnie to co zostało zaplanowane

---

## deploy-backend.yml — pytest gate + auto-deploy Cloud Run

```yaml
on:
  push:
    branches: [master, develop]
    paths:
      - 'app/**'           # (1) tylko gdy zmienił się kod aplikacji

jobs:
  test:
    name: pytest
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r app/requirements.txt -r app/requirements-dev.txt
      - run: pytest app/test_main.py -v   # (2) 15 testów — bramka deployowa

  deploy:
    needs: test             # (3) deploy czeka na testy
    env:
      IMAGE:        europe-central2-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/backend/app
      SERVICE_NAME: ${{ github.ref_name == 'develop' && 'backend-api-staging' || 'backend-api' }}  # (2)

    steps:
      # ... auth ...

      - name: Build and push Docker image
        run: |
          docker build \
            -t "${{ env.IMAGE }}:latest" \
            -t "${{ env.IMAGE }}:${{ github.sha }}" \  # (4)
            ./app
          docker push "${{ env.IMAGE }}:${{ github.sha }}"
          docker push "${{ env.IMAGE }}:latest"

      - name: Trivy vulnerability scan  # (5)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "${{ env.IMAGE }}:${{ github.sha }}"
          exit-code: "1"
          severity: "CRITICAL"

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy ${{ env.SERVICE_NAME }} \
            --image="${{ env.IMAGE }}:${{ github.sha }}" \  # (4)
            --region=europe-central2 \
            --project=${{ secrets.GCP_PROJECT_ID }} \
            --quiet
```

1. `paths: ['app/**']` — workflow nie uruchamia się przy zmianie docs, TF, frontend. Tylko kod aplikacji.
2. pytest 15 testów — pokrywa: endpointy, JWT decode z X-Apigateway-Api-Userinfo, email whitelist, CORS. `deploy` job nie ruszy jeśli jakikolwiek test failuje.
3. `needs: test` — deklaracja zależności między jobami. GitHub Actions nie uruchomi `deploy` przed sukcesem `test`.
4. SHA tag — każdy commit dostaje unikalny tag (40-znakowy git SHA). Pozwala zidentyfikować dokładną wersję obrazu w production.
5. Trivy container scan — skanuje zbudowany obraz Docker pod kątem CVE CRITICAL. `exit-code: "1"` blokuje deploy jeśli znajdzie podatność krytyczną.
6. `--image=...:SHA` zamiast `:latest` — wymusza nową rewizję Cloud Run przy każdym deploy. Terraform z `image = "...:latest"` nie wykryłby zmiany; `gcloud run deploy` zawsze deployuje podany obraz.

---

## deploy-frontend.yml — auto-deploy GCS

```yaml
on:
  push:
    branches: [master, develop]
    paths:
      - 'frontend/**'

jobs:
  deploy:
    steps:
      # ... auth ...

      - name: Deploy to prod — Cloudflare bucket (master)
        if: github.ref_name == 'master'
        run: |
          gcloud storage cp frontend/index.html gs://app.kamilos.xyz/ \
            --cache-control="no-cache, no-store, must-revalidate"
          gcloud storage cp frontend/app.js gs://app.kamilos.xyz/ \
            --cache-control="public, max-age=3600"

      - name: Deploy to prod — LB bucket (master, opcjonalny)  # (*)
        if: github.ref_name == 'master'
        continue-on-error: true
        run: |
          gcloud storage cp frontend/index.html gs://app-lb.kamilos.xyz/ \
            --cache-control="no-cache, no-store, must-revalidate"
          gcloud storage cp frontend/app.js gs://app-lb.kamilos.xyz/ \
            --cache-control="public, max-age=3600"

      - name: Deploy to staging (develop)
        if: github.ref_name == 'develop'
        run: |
          # Pobierz URL staging API Gateway dynamicznie
          STAGING_HOSTNAME=$(gcloud api-gateway gateways describe gpc-gateway-staging \
            --location=europe-west1 \
            --project=${{ secrets.GCP_PROJECT_ID }} \
            --format="value(defaultHostname)")

          # Podmień API_URL w app.js na URL staging (bez modyfikacji repo)
          sed "s|const API_URL = '.*'|const API_URL = 'https://${STAGING_HOSTNAME}'|" \
            frontend/app.js > /tmp/app.staging.js

          gcloud storage cp /tmp/app.staging.js gs://staging.kamilos.xyz/app.js \
            --cache-control="public, max-age=3600"
```

`sed` nadpisuje URL API w locie, bez commitowania zmiany do repo. Staging dostaje swój URL API Gateway, prod — hardkodowany URL z pliku.

`(*)` Krok LB bucket ma `continue-on-error: true` — warstwa `tf/frontend-lb/` jest opcjonalna. Jeśli bucket `app-lb.kamilos.xyz` nie istnieje, ten krok failuje cicho, a deploy do `app.kamilos.xyz` (Cloudflare) kontynuuje.

---

## security.yml — security scanning pipeline

Osobny workflow uruchamiany przy każdym pushu lub PR który dotyka `tf/**` lub `app/**`.

```yaml
on:
  push:
    branches: [master, develop]
    paths: ['tf/**', 'app/**']
  pull_request:
    branches: [master, develop]

jobs:
  pip-audit:      # SCA — CVE w zależnościach Pythona
  trivy-fs:       # CVE scan plików projektu (filesystem)
  checkov:        # IaC misconfigurations w tf/**
```

| Tool | Co skanuje | Blokuje gdy |
|------|-----------|-------------|
| `pip-audit` | `app/requirements.txt` vs PyPI Advisory DB | CVE w zależnościach Pythona |
| `trivy fs` | Pliki projektu (nie kontener) | CVE CRITICAL lub HIGH |
| `checkov` | `tf/**/*.tf` — konfiguracja IaC | Findings poza `.checkov.yaml` |

**`.checkov.yaml` — świadome wyjątki:**
```yaml
skip_checks:
  - id: "CKV_GCP_114"
    comment: "Cloud Run ingress=ALL — API GW nie jest LB; bezpieczeństwo przez IAM (ADR-001)"
  - id: "CKV_GCP_26"
    comment: "VPC Flow Logs — pominięte dla kosztu; prototyp bez wymagań compliance"
  # ... 10 więcej z uzasadnieniami
```

12 świadomie pominięte findings z pełnym uzasadnieniem i referencjami do ADRów. Każdy nowy finding wymaga albo naprawy albo wpisu do `.checkov.yaml` z wyjaśnieniem.

---

## GitHub Secrets

| Secret | Ustawiany przez | Wartość |
|--------|----------------|---------|
| `GCP_PROJECT_ID` | `bob_budowniczy.sh` | `gcp-prototype-1-20260224` |
| `GCP_WIF_PROVIDER` | `bob_budowniczy.sh` | `projects/NUMBER/locations/global/workloadIdentityPools/...` |
| `GCP_SA_EMAIL` | `bob_budowniczy.sh` | `gh-infra-worker@PROJECT.iam.gserviceaccount.com` |
| `GOOGLE_OAUTH_CLIENT_ID` | ręcznie | OAuth 2.0 client ID z GCP Console |
| `GOOGLE_OAUTH_CLIENT_SECRET` | ręcznie | OAuth 2.0 client secret |
| `ALERT_EMAIL` | ręcznie | email na alerty monitoringu |
| `GCP_BILLING_ACCOUNT_ID` | ręcznie (opcjonalny) | ID billing account (budget alert) |

---

## Pułapki

!!! danger "Terraform apply bez aktualnego planu"
    Deploy backend i auth używa `apply` bez saved tfplan — jeśli coś zmieniło się w stanie między `plan` a `apply`, Terraform może zaskoczyć. W praktyce: accept it dla tych warstw, bo `-target` apply i `import` zawsze zmieniają stan.

!!! warning "SHA tag: docker push obu tagów"
    Push `":SHA"` i `":latest"`. Gdybyś pushował tylko SHA, następny `terraform plan` dla backend widzi `image=...:latest` i twierdzi że nie ma zmian (bo obraz latest to stary obraz). Push `":latest"` zapewnia że manual deploy przez `deploy.yml` też dostanie nowy obraz.

!!! tip "Monitoring deployów"
    ```bash
    # Lista ostatnich run dla konkretnego workflow
    gh run list --workflow=deploy-backend.yml --limit=5

    # Podgląd logów konkretnego run
    gh run view RUN_ID --log

    # Watch w czasie rzeczywistym
    gh run watch RUN_ID
    ```
