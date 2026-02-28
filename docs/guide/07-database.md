# Baza danych — Firestore

Firestore to dokument NoSQL — bez schematu, bez migracji, bez konfiguracji sieci. Jeden zasób Terraform, zerowy koszt baseline.

---

## Dlaczego Firestore, nie Cloud SQL

Pełne porównanie w [ADR-002](https://github.com/benhornbeam/gcp-prototype-1-20260224/blob/master/docs/adr/ADR-002.md). Skrót:

| Kryterium | Cloud SQL | Firestore |
|-----------|-----------|-----------|
| Koszt przy 0 req | ~$7/mies (instancja działa zawsze) | $0 |
| Koszt przy małym ruchu | ~$7/mies | $0 |
| Zasoby Terraform | ~5 (instance, db, user, VPC peering, proxy) | 1 (`google_firestore_database`) |
| Migracje schematu | Wymagane (Alembic/Flyway) | Brak schematu |
| Połączenie z Cloud Run | Cloud SQL Auth Proxy lub Private IP | Bezpośrednio przez HTTP API |
| Backup | Point-in-time recovery | Geo-replikacja managed |

Wybraliśmy Firestore bo model danych aplikacji (lista logowań per user) jest naturalnie dokumentowy, a koszt jest kluczowy dla prototypu.

---

## Terraform — warstwa `tf/database/`

```hcl
# tf/database/main.tf

# Włącz Firestore API
resource "google_project_service" "apis" {
  service            = "firestore.googleapis.com"
  disable_on_destroy = false
}

# Baza danych Firestore (jedna per projekt — "default")
resource "google_firestore_database" "db" {
  project     = var.project_id
  name        = "(default)"            # (1)
  location_id = var.region             # europe-central2
  type        = "FIRESTORE_NATIVE"     # (2)

  depends_on = [google_project_service.apis]
}

# IAM — Cloud Run SA może czytać/pisać
resource "google_project_iam_member" "run_sa_firestore" {
  project = var.project_id
  role    = "roles/datastore.user"     # (3)
  member  = "serviceAccount:cloud-run-backend-sa@${var.project_id}.iam.gserviceaccount.com"
}
```

1. `name = "(default)"` — Firestore obsługuje jedną bazę `(default)` per projekt na Free tier. Kolejne bazy wymagają Blaze plan.
2. `FIRESTORE_NATIVE` vs `DATASTORE_MODE` — Native obsługuje realtime listeners, richer query model, better SDK. Datastore Mode to tryb kompatybilności z Cloud Datastore. Zawsze wybieraj Native dla nowych projektów.
3. `roles/datastore.user` — może czytać, pisać, usuwać dokumenty. Nie może zarządzać bazą (tworzyć indeksów programatycznie, usuwać bazy). Zasada least privilege.

---

## Model danych

Kolekcje Firestore nie wymagają wcześniejszego tworzenia — powstają przy pierwszym zapisie.

```
Firestore "(default)"
│
├─ logins/                              ← kolekcja (prod)
│  └─ {uid}/                            ← dokument = user ID (z JWT sub)
│     └─ events/                        ← subkolekcja
│        ├─ {auto_id}/                  ← dokument = jedno logowanie
│        │  ├─ timestamp: Timestamp
│        │  ├─ email: "user@gmail.com"
│        │  └─ name: "Jan Kowalski"
│        └─ ...
│
└─ staging_logins/                      ← kolekcja (staging)
   └─ {uid}/
      └─ events/
         └─ ...
```

Izolacja prod/staging przez prefix kolekcji (`staging_`) — ten sam projekt GCP, ta sama baza Firestore, ale oddzielne dane.

---

## Klient Python — `google-cloud-firestore`

```python
from google.cloud import firestore

# Client używa Application Default Credentials
# W Cloud Run: automatycznie używa SA cloud-run-backend-sa
db = firestore.Client()

# Zapis
db.collection("logins") \
  .document(uid) \
  .collection("events") \
  .add({
      "timestamp": datetime.now(timezone.utc),
      "email": email,
  })

# Odczyt — ostatnie 5, posortowane malejąco
events = (
    db.collection("logins")
      .document(uid)
      .collection("events")
      .order_by("timestamp", direction=firestore.Query.DESCENDING)
      .limit(5)
      .stream()
)
```

`firestore.Client()` bez argumentów używa ADC. W Cloud Run — SA `cloud-run-backend-sa` z rolą `roles/datastore.user`. Lokalnie — Twoje konto (przez `gcloud auth application-default login`).

---

## Free tier Firestore

| Metryka | Free tier limit/dzień |
|---------|----------------------|
| Odczyty dokumentów | 50,000 |
| Zapisy dokumentów | 20,000 |
| Usunięcia dokumentów | 20,000 |
| Storage | 1 GB łącznie |
| Egress | 10 GB/mies |

Dla prototypu z kilkoma użytkownikami — Free tier wystarczy na wiele miesięcy.

---

## Pułapki

!!! danger "Firestore database jest immutable po stworzeniu"
    Po `terraform apply` nie możesz zmienić `location_id` ani `type`. Zmiana = `destroy + create` = utrata wszystkich danych. Wybierz region raz i trzymaj się go.

!!! warning "roles/datastore.admin nie działa na poziomie projektu"
    Google nie obsługuje `roles/datastore.admin` jako project-level IAM binding (zwraca Error 400). Dostępna alternatywa: `roles/datastore.owner` — szersza rola (może zarządzać samą bazą). Dla uprawnień CI/CD SA (`gh-infra-worker`) używamy `roles/datastore.owner`; dla Cloud Run runtime SA (`cloud-run-backend-sa`) — `roles/datastore.user`.

!!! tip "Indeksy kompozytowe"
    `order_by("timestamp").limit(5)` nie wymaga dodatkowego indeksu — Firestore ma automatyczne indeksy dla pojedynczych pól. Złożone zapytania (order_by po wielu polach, where + order_by po różnych polach) wymagają manualnego tworzenia indeksów w konsoli lub przez `gcloud firestore indexes create`.
