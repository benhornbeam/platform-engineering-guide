# Wymagania wstępne

Zanim zaczniesz, musisz mieć dostęp do kilku zewnętrznych serwisów i lokalnie zainstalowane narzędzia. Ten rozdział jest krótki — wróć do niego gdy napotkasz błąd "command not found".

---

## Konta zewnętrzne

| Serwis | Wymagane | Po co |
|--------|---------|-------|
| Google Cloud | Konto z billing account | Projekt GCP, wszystkie zasoby |
| GitHub | Konto + org lub user | Repo, Actions, Secrets |
| Cloudflare | Konto z domeną | CDN, SSL, DNS dla frontendu |

!!! warning "Billing account"
    Musisz mieć aktywne konto rozliczeniowe GCP. Free tier nie wystarczy — Cloud Run, VPC Connector i API Gateway wymagają włączonego billing (nawet jeśli koszty są $0). Uruchom:
    ```bash
    gcloud beta billing accounts list
    ```
    Jeśli puste — dodaj kartę w [console.cloud.google.com/billing](https://console.cloud.google.com/billing).

---

## Narzędzia lokalne

```bash
# Sprawdź co już masz
terraform version    # wymaga >= 1.5.0
gcloud version       # Google Cloud SDK
gh version           # GitHub CLI
docker version       # Docker Engine
git version
```

### Instalacja (macOS)

```bash
brew install terraform google-cloud-sdk gh docker git
```

### Instalacja (Linux/Debian)

=== "Terraform"
    ```bash
    wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | \
      sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
    echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
      https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
      sudo tee /etc/apt/sources.list.d/hashicorp.list
    sudo apt update && sudo apt install terraform
    ```

=== "gcloud CLI"
    ```bash
    curl https://sdk.cloud.google.com | bash
    exec -l $SHELL
    gcloud init
    ```

=== "GitHub CLI"
    ```bash
    sudo apt install gh
    gh auth login
    ```

---

## Konfiguracja lokalna

### 1. Uwierzytelnienie gcloud

```bash
gcloud auth login
gcloud auth application-default login  # (1)
```

1. Application Default Credentials (ADC) — używane przez Terraform lokalnie. Bez tego `terraform plan` zwróci błąd autentykacji.

### 2. GitHub CLI

```bash
gh auth login
# Wybierz: GitHub.com → HTTPS → Authenticate with browser
```

### 3. Weryfikacja uprawnień GCP

Potrzebujesz uprawnień na poziomie organizacji (do tworzenia projektów) lub przynajmniej prawa do tworzenia projektu w folderze.

```bash
gcloud organizations list
# Powinna pojawić się twoja org z ID
```

Jeśli pusta — skontaktuj się z adminem GCP. Projekt możesz też tworzyć bez organizacji, ale stracisz hierarchię uprawnień.

---

## Struktura repo

Po wykonaniu bootstrapu repo będzie wyglądać tak:

```
.
├── scripts/
│   ├── run.sh                 # entry point — generuje project ID i odpala bootstrap
│   ├── bob_budowniczy.sh      # 7-etapowy idempotentny bootstrap
│   └── cleanup.sh             # usuwa wszystko (nieodwracalne)
├── tf/
│   ├── bootstrap/             # WIF, Service Accounts — tylko lokalnie
│   ├── infra/                 # VPC, subnet, firewall
│   ├── backend/               # Cloud Run, Artifact Registry, VPC Connector
│   ├── auth/                  # Identity Platform, Google SSO
│   ├── api-gateway/           # API Gateway, OpenAPI spec
│   ├── database/              # Firestore
│   ├── frontend/              # GCS buckets
│   └── monitoring/            # Cloud Monitoring dashboard + alerty
├── app/
│   ├── main.py                # FastAPI backend
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── index.html
│   └── app.js
├── .github/
│   └── workflows/
│       ├── deploy.yml          # ręczny deploy TF (wszystkie warstwy)
│       ├── deploy-backend.yml  # auto-deploy po push do app/
│       └── deploy-frontend.yml # auto-deploy po push do frontend/
└── docs/
    ├── adr/                    # Architecture Decision Records
    └── guide/                  # ten przewodnik
```

!!! info "Warstwy Terraform"
    Każda warstwa (`tf/<nazwa>/`) ma własny stan Terraform w osobnym prefiksie GCS. Możesz deployować je niezależnie — zmiana w `tf/frontend/` nie dotyka stanu `tf/backend/`. To kluczowa decyzja architektoniczna (separacja odpowiedzialności + bezpieczeństwo stanu).
