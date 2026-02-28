# Security Review — checklist

Security review to systematyczne przejście przez każdą warstwę architektury pod kątem bezpieczeństwa. Robimy to po każdej zmianie Terraform, przed każdym apply.

---

## Dlaczego security review jako kod?

Większość projektów ma security review jako slajdy PowerPoint, które nikt nie czyta. My mamy checklist jako `/sec-review` — uruchamiasz przed deployem, dostajesz raport. Zasady nie istnieją dopóki nie są testowalne.

---

## Checklist — warstwa po warstwie

### IAM

```
✅ SA nie mają roles/owner ani roles/editor
✅ gh-infra-worker: roles/iam.securityAdmin (nie projectIamAdmin)
✅ cloud-run-backend-sa: tylko logging.logWriter + datastore.user
✅ api-gateway-sa: tylko roles/run.invoker (na konkretnym service, nie na projekcie)
✅ Brak tworzenia SA keys — wszystko przez WIF lub ADC
```

**Zasada:** każde SA ma minimalny zestaw ról potrzebnych do działania. Nic więcej.

Dlaczego `iam.securityAdmin` zamiast `resourcemanager.projectIamAdmin`?

```
projectIamAdmin:
  - może nadawać WSZYSTKIE role na projekcie
  - jeśli gh-infra-worker zostanie skompromitowany → attacker ma god mode

iam.securityAdmin:
  - może nadawać role, ale tylko te które sam posiada
  - węższy blast radius przy kompromitacji
```

### Cloud Run

```
✅ ingress = INGRESS_TRAFFIC_ALL (wymagane — API Gateway nie jest LB)
✅ IAM isolacja: tylko api-gateway-sa ma roles/run.invoker
✅ VPC Connector — egress przez prywatną sieć
✅ SA: cloud-run-backend-sa (nie default Compute SA)
⚠️  min_instance_count = 0 (cold start, ale nie security issue)
```

Cloud Run z `ingress=ALL` jest **technicznie** dostępny z internetu. Ochrona:
1. Brak `allUsers` w IAM → 403 bez tokenu
2. Brak `allAuthenticatedUsers` → tylko `api-gateway-sa` może wywołać

```bash
# Test: wywołanie Cloud Run bez tokenu
curl -si https://backend-api-mngq3uouha-lm.a.run.app/api
# Oczekiwane: 403 Forbidden
```

### Sieć

```
✅ VPC: auto_create_subnetworks = false
✅ Firewall: deny-all ingress (priority 65534) — eksplicytna dokumentacja intent
✅ private_ip_google_access = true — bez publicznych IP dla zasobów VPC
✅ VPC Connector: oddzielna podsieć 10.8.0.0/28
```

### Secrets Management

```
✅ OAuth credentials w Secret Manager (nie w plaintext TF zmiennych)
✅ TF zmienne sensitive=true dla credentiali
✅ GitHub Secrets — nie repozytoria kluczy SA
✅ Żadnych sekretów w kodzie (app.js apiKey jest publiczny z założenia)
✅ State bucket: public_access_prevention = enforced
```

```hcl
# DOBRZE: Secret Manager
resource "google_secret_manager_secret_version" "oauth_secret" {
  secret_data = var.google_oauth_client_secret  # sensitive var
}

# ŹLE: plaintext w output
output "oauth_secret" {
  value = var.google_oauth_client_secret  # NIGDY NIE RÓB TEGO
}
```

### API Gateway

```
✅ Wszystkie ścieżki mają security: - firebase: []
✅ OPTIONS (CORS preflight) ma security: [] (wymagane — preflight nie ma JWT)
✅ JWT walidowany: issuer, audience, podpis, expiry
✅ Backend SA: api-gateway-sa (nie default SA)
```

### Storage

```
✅ State bucket: public_access_prevention = enforced (wymagane)
⚠️  Frontend buckets: public_access_prevention = unspecified + allUsers objectViewer
    → celowe — hosting publicznej SPA. Tylko czytanie obiektów, nie zarządzanie.
```

Różnica między state bucket a frontend bucket:

```hcl
# State bucket — musi być prywatny
resource "google_storage_bucket" "tf_state" {
  public_access_prevention = "enforced"  # ✅
}

# Frontend bucket — publiczny SPA hosting
resource "google_storage_bucket" "frontend" {
  public_access_prevention = "unspecified"  # (1)
}
resource "google_storage_bucket_iam_member" "public_read" {
  role   = "roles/storage.objectViewer"  # tylko czytanie plików
  member = "allUsers"
}
```

1. `unspecified` bo `enforced` blokowałoby `allUsers` IAM binding. Nie używamy `inherited` (dziedziczyłoby z org policy).

### Service Account Keys

```
✅ Brak tworzenia SA keys w całym TF (google_service_account_key)
✅ CI/CD: WIF (keyless) — tymczasowe tokeny, ważne 1h
✅ Lokalne: gcloud ADC (user credentials) lub impersonacja SA
```

Jeśli znajdziesz `google_service_account_key` w jakimkolwiek TF pliku — usuń natychmiast.

---

## Świadome kompromisy bezpieczeństwa

Nie wszystko jest idealne — to są świadome decyzje z udokumentowanym uzasadnieniem:

| Kompromis | Powód | Ścieżka upgrade |
|-----------|-------|-----------------|
| `ingress=ALL` na Cloud Run | API Gateway nie jest LB | Private Service Connect (kompleks) |
| `datastore.owner` dla gh-infra-worker | `datastore.admin` niedostępny na poziomie projektu | Brak prostszej alternatywy w GCP |
| JWT decode bez weryfikacji w FastAPI | API Gateway już weryfikuje; backend chroniony IAM | Dodać `python-jose` jeśli defence-in-depth wymagane |
| Jeden SA dla Cloud Run prod+staging | Uproszczenie | Osobne SA per env |

---

## Pułapki

!!! danger "State Terraform zawiera wrażliwe dane"
    `terraform.tfstate` może zawierać: endpointy Cloud Run, SA emails, czasem wartości zmiennych. State bucket **musi** mieć `public_access_prevention = enforced` i dostęp tylko dla `gh-infra-worker` i własnych adminów.

!!! danger "Terraform output sensitive=false"
    Nigdy nie dodawaj do `outputs.tf` wartości takich jak `client_secret`, klucze, tokeny bez `sensitive = true`. Outputs są widoczne w logach GitHub Actions.

!!! tip "Security review przed każdym PR"
    Uruchom `/sec-review` przed merge PR zawierającego zmiany w `tf/`. W tym projekcie jest to skill Claude Code — wykonuje automatyczną weryfikację wszystkich powyższych punktów na podstawie `git diff`.
