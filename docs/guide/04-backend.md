# Backend — Cloud Run + FastAPI

Cloud Run to serce platformy. Bezserwerowy kontener — płacisz za czas procesowania, nie za czas działania. Przy 0 requestach: $0 kosztów.

---

## Architektura backendu

```
[GitHub: push do app/]
         │
         ▼
[GitHub Actions: deploy-backend.yml]
  docker build → tag :SHA
  docker push → Artifact Registry (europe-central2)
         │
         ▼
  gcloud run deploy backend-api --image=...:SHA
         │
         ▼
[Cloud Run v2 Service: backend-api]
  ingress = ALL (API Gateway wymaga, izolacja przez IAM)
  IAM: tylko api-gateway-sa ma roles/run.invoker
  SA: cloud-run-backend-sa
      → roles/logging.logWriter
      → roles/datastore.user
  scaling: min=0, max=3
         │
         ▼ bezpośrednio przez googleapis.com
  [Firestore "(default)"]
```

---

## Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

Port `8080` — wymagany przez Cloud Run (domyślny `containerPort`). `python:3.12-slim` zamiast `python:3.12` — mniejszy obraz (~150MB vs ~1GB), krótszy pull przy cold start.

---

## FastAPI — `app/main.py`

### Konfiguracja per środowisko

```python
import os

# Env var ustawiana przez Cloud Run — różna dla prod i staging
ENV = os.getenv("ENV", "prod")
COLLECTION_PREFIX = "staging_" if ENV == "staging" else ""  # (1)
ALLOWED_ORIGINS = (
    ["https://staging.kamilos.xyz"] if ENV == "staging"
    else ["https://app.kamilos.xyz"]
)
```

1. Ten sam obraz Docker dla prod i staging. Różnica: env var `ENV`. Nie duplikujemy kodu, nie tworzymy osobnych obrazów.

### CORS Middleware

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,    # (1)
    allow_methods=["GET", "POST"],
    allow_headers=["Authorization", "Content-Type"],
)
```

1. `allow_origins` jest listą konkretnych domen, nie `["*"]`. Prod: `["https://app.kamilos.xyz", "https://app-lb.kamilos.xyz"]` (opcjonalna warstwa LB). Staging: `["https://staging.kamilos.xyz"]`. Zbyt liberalne CORS to częsty błąd bezpieczeństwa.

### Pobranie tożsamości użytkownika — `get_user_claims`

API Gateway zastępuje nagłówek `Authorization` własnym OIDC tokenem (`api-gateway-sa`) przy wywołaniu Cloud Run przez IAM. Dekodując `Authorization`, backend widzi SA, nie użytkownika. Poprawne źródło: nagłówek `X-Apigateway-Api-Userinfo` — API Gateway dodaje go automatycznie po walidacji JWT, z base64-encoded JSON claims użytkownika.

```python
def get_user_claims(
    authorization: str | None,
    x_apigateway_api_userinfo: str | None,
) -> dict:
    """Pobiera claims użytkownika.

    Kolejność:
    1. X-Apigateway-Api-Userinfo — claims z Firebase JWT, dodane przez API GW (1)
    2. Authorization — fallback dla lokalnych testów bez API GW
    """
    if x_apigateway_api_userinfo:                                              # (1)
        try:
            padded = x_apigateway_api_userinfo + "=" * (4 - len(x_apigateway_api_userinfo) % 4)
            return json.loads(base64.b64decode(padded))
        except Exception:
            pass

    if not authorization:
        raise HTTPException(status_code=401, detail="Missing authorization")

    try:                                                                        # (2)
        token = authorization.split(" ")[1]
        payload = token.split(".")[1]
        payload += "=" * (4 - len(payload) % 4)
        return json.loads(base64.b64decode(payload))
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")
```

1. Gdy request przychodzi przez API Gateway — `X-Apigateway-Api-Userinfo` zawiera claims Firebase użytkownika. Dekodujemy base64url → JSON dict.
2. Fallback: bezpośrednie wywołanie Cloud Run (np. local `curl`) — dekodujemy `Authorization` JWT. Bez API GW brak IAM check, więc ten fallback przydatny tylko do debugowania.

### Kontrola dostępu — email whitelist

```python
_CACHE_TTL = 300  # 5 minut
_allowed_cache: dict = {"emails": frozenset(), "expires": 0.0}

def get_allowed_emails() -> frozenset:
    """Czyta listę dozwolonych emaili z Firestore config/allowed_emails. Cache 5 min."""
    now = time.monotonic()
    if now < _allowed_cache["expires"]:
        return _allowed_cache["emails"]
    doc = db.collection("config").document("allowed_emails").get()
    emails = frozenset(doc.to_dict().get("emails", [])) if doc.exists else frozenset()
    _allowed_cache["emails"] = emails
    _allowed_cache["expires"] = now + _CACHE_TTL
    return emails

def check_access(email: str) -> None:
    """Rzuca 403 jeśli email nie jest na liście. Pusta lista = brak ograniczeń."""
    allowed = get_allowed_emails()
    if allowed and email not in allowed:
        raise HTTPException(status_code=403, detail="Access denied")
```

Lista dozwolonych emaili przechowywana w Firestore: kolekcja `config`, dokument `allowed_emails`, pole `emails: [string]`. Cache 5-minutowy — nie odpytujemy Firestore przy każdym requeście. Pusta lista = brak ograniczeń (wszyscy zalogowani użytkownicy mają dostęp).

### Endpointy

```python
@app.post("/api")
def api_post(
    authorization: str = Header(None),
    x_apigateway_api_userinfo: str = Header(None),
):
    claims = get_user_claims(authorization, x_apigateway_api_userinfo)
    check_access(claims.get("email", ""))

    # Hierarchia kolekcji: logins/{uid}/events/{auto_id}
    db.collection(f"{COLLECTION_PREFIX}logins") \
      .document(claims["sub"]) \
      .collection("events") \
      .add({
          "timestamp": datetime.now(timezone.utc),
          "email":     claims.get("email", ""),
          "name":      claims.get("name", ""),
      })
    return {"status": "ok"}


@app.get("/api")
def api_get(
    authorization: str = Header(None),
    x_apigateway_api_userinfo: str = Header(None),
):
    claims = get_user_claims(authorization, x_apigateway_api_userinfo)
    check_access(claims.get("email", ""))

    # Ostatnie 5 logowań, posortowane malejąco
    events = (
        db.collection(f"{COLLECTION_PREFIX}logins")
          .document(claims["sub"])
          .collection("events")
          .order_by("timestamp", direction=firestore.Query.DESCENDING)
          .limit(5)
          .stream()
    )
    return {
        "user": claims.get("email", claims["sub"]),
        "last_logins": [
            {"timestamp": e.to_dict()["timestamp"].isoformat(),
             "email":     e.to_dict().get("email", "")}
            for e in events
        ],
    }
```

---

## Terraform — warstwa `tf/backend/`

### Artifact Registry

```hcl
resource "google_artifact_registry_repository" "backend_repo" {
  repository_id = "backend"
  format        = "DOCKER"
  location      = var.region  # europe-central2
}
```

AR zamiast Docker Hub: obrazy w tym samym regionie co Cloud Run → pull szybszy i bezpłatny (brak egress między regionami). AR integruje się z `gcloud auth` — nie potrzebujesz osobnych credentials dla Docker.

### Service Account dla Cloud Run runtime

```hcl
resource "google_service_account" "cloud_run_sa" {
  account_id = "cloud-run-backend-sa"
}

# Minimalny zestaw ról — tylko co potrzebne
resource "google_project_iam_member" "run_sa_logging" {
  role   = "roles/logging.logWriter"  # logi do Cloud Logging
  member = "serviceAccount:${google_service_account.cloud_run_sa.email}"
}

# roles/datastore.user przyznawany w tf/database/main.tf
# — warstwa database zarządza swoimi uprawnieniami IAM
```

### Cloud Run Service

```hcl
resource "google_cloud_run_v2_service" "backend" {
  name     = "backend-api"
  location = var.region
  ingress  = "INGRESS_TRAFFIC_ALL"  # (1)

  template {
    service_account = google_service_account.cloud_run_sa.email

    containers {
      image = var.image         # (2)
      ports { container_port = 8080 }
    }

    scaling {
      min_instance_count = 0   # (3)
      max_instance_count = 3
    }
  }
}
```

1. `INGRESS_TRAFFIC_ALL` — API Gateway nie jest LB. Izolacja przez IAM: tylko `api-gateway-sa` z `roles/run.invoker` może wywołać Cloud Run. Brak VPC Connectora — ADR-008: Firestore dostępny przez publiczne Google APIs, connector nie był potrzebny i generował stały koszt ~$7/mies.
2. `var.image` — przy pierwszym Terraform apply: `latest`. Kolejne deploye używają SHA tagu przez `gcloud run deploy` (obchodzi Terraform).
3. `min_instance_count = 0` — skaluje do zera. Cold start ~200–500ms po idle. Produkcja z SLA: ustaw `min_instance_count = 1` (~$7/mies za always-on instancję).

---

## Sekwencja bootstrapu — chicken-and-egg

Problem: `terraform apply` dla Cloud Run wymaga obrazu w AR. AR musi istnieć zanim `docker push`.

```bash
# Krok 1: tylko AR (bez Cloud Run)
terraform apply \
  -target=google_project_service.apis \
  -target=google_artifact_registry_repository.backend_repo

# Krok 2: zbuduj i wypchnij obraz
gcloud auth configure-docker europe-central2-docker.pkg.dev
docker build -t europe-central2-docker.pkg.dev/PROJECT/backend/app:latest ./app
docker push europe-central2-docker.pkg.dev/PROJECT/backend/app:latest

# Krok 3: pełny apply (teraz Cloud Run znajdzie obraz)
terraform apply
```

W `deploy.yml` (GitHub Actions) ta sekwencja jest zautomatyzowana jako osobne kroki z `if: github.event.inputs.layer == 'backend'`.

---

## Pułapki

!!! danger "Terraform nie wykrywa zmiany obrazu :latest"
    Gdy aktualizujesz kod i pushasz nowy `:latest` do AR, Terraform widzi `image = "...app:latest"` — ta wartość się nie zmieniła, więc **Terraform nie deployuje nowej wersji Cloud Run**. Dlatego auto-deploy używa `gcloud run deploy` z SHA tagiem bezpośrednio, z pominięciem Terraform state.

!!! warning "Cold start przy min=0"
    Pierwsze żądanie po ~15 minutach idle wymaga uruchomienia kontenera: ~200–500ms dla tego obrazu. Jeśli Twój use case wymaga SLA < 500ms — ustaw `min_instance_count = 1`.

!!! tip "Cloud Run v2 vs v1"
    Używamy `google_cloud_run_v2_service` (API v2). Starsza `google_cloud_run_service` (v1) ma inne pola (`template.0.metadata` zamiast `template`). Przy kopiowaniu przykładów z internetu zawsze sprawdź wersję resource.
