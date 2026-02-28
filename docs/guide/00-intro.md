# Czego się nauczysz

Ten przewodnik pokazuje, jak od zera zbudować produkcyjną platformę cloud na Google Cloud Platform — bez klikania w konsoli, bez kluczy serwisowych w kodzie, bez ręcznych kroków przy każdym deployu.

---

## Co zbudujemy

Kompletna platforma webowa z backendem, frontendem, bazą danych, uwierzytelnianiem i monitoringiem — w pełni zautomatyzowana, z oddzielnym środowiskiem staging.

```
[Użytkownik] → HTTPS → [Cloudflare CDN]
                              │
                              ▼
                    [GCS Bucket — SPA]
                              │ fetch() + JWT
                              ▼
                    [API Gateway] ← walidacja JWT (Identity Platform)
                              │ OIDC token
                              ▼
                    [Cloud Run — FastAPI] ← ingress tylko od API GW (IAM)
                              │ przez VPC
                              ▼
                    [Firestore] ← zerowy koszt przy braku ruchu

[GitHub Actions] → WIF (bez kluczy) → deploy całej infrastruktury
[Cloud Monitoring] → dashboard + alerty → email
```

---

## Stos technologiczny

| Warstwa | Technologia | Dlaczego |
|---------|-------------|----------|
| Infrastruktura jako kod | Terraform v1.5+ | Powtarzalny, wersjonowany, idempotentny |
| Compute | Google Cloud Run | Skaluje do 0 — zerowy koszt przy braku ruchu |
| Auth | Identity Platform + Firebase Auth | Google SSO out-of-the-box, JWT managed |
| API | Google Cloud API Gateway | Walidacja JWT przed backendem, OpenAPI spec |
| Baza danych | Firestore | Zerowy koszt baseline, brak konfiguracji sieci |
| Frontend | GCS bucket + Cloudflare CDN | ~$0/mies, custom domena, DDoS protection |
| CI/CD | GitHub Actions + WIF | Keyless auth — zero sekretów z kluczami SA |
| Monitoring | Cloud Monitoring | Natywna integracja, alerty jako kod TF |
| Sieć | VPC + Serverless Connector | Izolacja backendu od publicznego internetu |

---

## Architektura decyzji (ADR-driven)

Każda nieoczywista decyzja jest udokumentowana jako ADR (Architecture Decision Record) — z kontekstem, alternatywami i uzasadnieniem. Znajdziesz je w `docs/adr/`. To nie jest dokumentacja "co zrobiliśmy", tylko "dlaczego tak, a nie inaczej".

Przykłady decyzji, które mogłyby pójść inaczej:

| Decyzja | Wybraliśmy | Odrzuciliśmy | Powód |
|---------|-----------|--------------|-------|
| Compute | Cloud Run | GKE Autopilot | GKE: $70/mies baseline, overkill dla prototypu |
| CDN | Cloudflare Free | Google Global LB | Google LB: $23/mies, Cloudflare: $0 |
| Auth CI/CD | Workload Identity Federation | Service Account keys | Klucze SA to wektory ataku — WIF eliminuje problem |
| Baza | Firestore | Cloud SQL | SQL: $7/mies zawsze; Firestore: $0 przy braku ruchu |
| API security | IAM + JWT | Network ingress rules | IAM daje lepszą izolację przy tym stosie |

---

## Jak czytać ten przewodnik

Rozdziały są ułożone w kolejności implementacji. Każdy rozdział zawiera:

- **Kontekst** — co budujemy i dlaczego w tej kolejności
- **Kod** — pliki Terraform i konfiguracje z komentarzami
- **Pułapki** — rzeczywiste błędy napotkane podczas budowania (nie hipotetyczne)
- **Alternatywy** — co odrzuciliśmy i dlaczego

!!! tip "Senior thinking"
    Pułapki i odrzucone alternatywy to najcenniejsza część. Każdy może skopiować działający kod — zrozumienie *dlaczego* tak, a nie inaczej, to jest to, co odróżnia architekta od copy-paste engineera.

---

## Szacowany koszt platformy

| Komponent | Koszt/mies | Uwagi |
|-----------|-----------|-------|
| Cloud Run | ~$0 | Skaluje do 0; przy ruchu: $0.00002400/vCPU-sec |
| API Gateway | ~$0 | Free tier: 2M calls/mies |
| Firestore | ~$0 | Free tier: 1 GB storage, 50k reads/dzień |
| GCS bucket | ~$0.02 | 0.026 USD/GB/mies |
| Cloudflare CDN | $0 | Free tier bez limitu bandwidth |
| VPC Connector | ~$7 | Stały koszt nawet przy 0 req — największy koszt |
| Cloud Monitoring | $0 | Free tier: 150 MB metryk/mies |
| **Łącznie** | **~$7–10/mies** | Praktycznie tylko VPC Connector |

VPC Connector to świadoma decyzja — izoluje backend sieciowo. Bez niego koszt = ~$0.

