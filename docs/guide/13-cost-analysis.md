# Analiza kosztów

Koszty to pierwszorzędna decyzja architektoniczna — każda warstwa była wybierana z uwzględnieniem kosztów przy niskim ruchu.

---

## Rzeczywisty koszt tej architektury

| Komponent | Koszt/mies | Model rozliczenia | Uwagi |
|-----------|-----------|-------------------|-------|
| **VPC Connector** | ~$7.00 | Stały (reserved capacity) | Największy koszt. min_throughput=200Mbps |
| Cloud Run | $0–0.50 | Per request + per vCPU-sec | $0.00002400/vCPU-s, $0.00000250/GB-s |
| API Gateway | $0–0.03 | Per million calls | Free: 2M calls/mies |
| Firestore | $0 | Per read/write/storage | Free: 50k reads/dzień, 20k writes/dzień, 1GB |
| GCS buckets | ~$0.05 | Per GB storage + egress | 0.026 USD/GB/mies EU multi-region |
| Artifact Registry | ~$0.10 | Per GB storage | 0.10 USD/GB/mies powyżej 0.5GB free |
| Cloud Monitoring | $0 | Per MB ingested | Free: 150 MB/mies system metrics |
| Cloudflare CDN | $0 | Free tier | Bez limitu bandwidth |
| Secret Manager | ~$0.06 | Per secret version + access | $0.06/version/mies, $0.03/10k access |
| **Łącznie** | **~$7–8/mies** | | Praktycznie tylko VPC Connector |

---

## Gdzie jest Free tier

### Cloud Run — prawie zawsze $0

```
Miesięczny free tier Cloud Run:
  - 2,000,000 requestów
  - 360,000 GB-sekund pamięci RAM
  - 180,000 vCPU-sekund

Przy 1000 requestów/dzień × 30 dni = 30,000 req/mies
  → daleko poniżej 2M free tier
  → $0
```

### API Gateway — prawie zawsze $0

```
Free tier: 2,000,000 calls/mies (pierwsze 12 miesięcy)
Po free tier: $3.50 per million calls

Przy 1000 req/dzień = 30,000/mies → $0 (free tier)
Przy 100k req/dzień = 3M/mies → $3.50 (po free tier)
```

### Firestore — $0 przy niskim ruchu

```
Free tier (Spark plan, bez limitu czasu):
  - 50,000 document reads/dzień
  - 20,000 document writes/dzień
  - 20,000 document deletes/dzień
  - 1 GB storage

Przy 100 userów × 2 logins/dzień:
  - 200 reads + 200 writes → 0.4% free tier
  → $0
```

---

## Co kosztuje $7/mies (VPC Connector)

VPC Connector to jedyny stały koszt — niezależny od ruchu.

```
Cena: $0.010/h (200 Mbps minimum)
Przy pełnym miesiącu: $0.010 × 24h × 30 dni = $7.20/mies

Uwaga: throughput 200-300 Mbps to RESERVED capacity,
nie faktyczne użycie. Płacisz za gotowość,
nawet przy 0 requestach.
```

**Alternatywa bez VPC Connectora:** Firestore przez publiczne API, bez VPC. Koszt = ~$0/mies. Kompromis: brak sieci prywatnej, Cloud Run komunikuje się z Firestore przez internet (szyfrowany, ale publiczny).

```hcl
# Bez VPC Connector — usunąć z Cloud Run template:
# vpc_access { ... }
# I usunąć VPC Connector resource z backend/main.tf
```

---

## Koszt przy skalowaniu

| Ruch | Req/mies | Cloud Run | API Gateway | Firestore | Łącznie |
|------|----------|-----------|-------------|-----------|---------|
| Prototyp | 30K | $0 | $0 | $0 | $7 (VPC) |
| Mały startup | 500K | ~$0.50 | $0 | ~$0.20 | $8 |
| Rosnący produkt | 5M | ~$5 | ~$10.50 | ~$2 | $25 |
| Scale-up | 50M | ~$50 | ~$140 | ~$20 | $220 |

Przy 5M requestach miesięcznie (~170k/dzień): $25/mies. To punkt gdzie warto rozważyć optymalizacje (cache, min-instances, batching).

---

## Koszt vs alternatywy

### Gdybyśmy wybrali Google Global Load Balancer + CDN + Cloud Armor

```
Google Cloud Load Balancing: ~$18/mies (forwarding rules + ingress)
Cloud CDN:                   ~$2/mies (minimal traffic)
Cloud Armor (Security tier): ~$5/mies (minimum)
Łącznie:                     ~$25/mies

Vs Cloudflare Free:          $0/mies
Oszczędność:                 ~$25/mies = ~$300/rok
```

### Gdybyśmy wybrali GKE Autopilot

```
GKE Autopilot minimum:  ~$70/mies (cluster management fee)
Node pool (2 vCPU):     ~$50/mies
Łącznie:                ~$120/mies

Vs Cloud Run:           $0–$0.50/mies
Oszczędność:            ~$120/mies = ~$1,440/rok
```

### Gdybyśmy wybrali Cloud SQL zamiast Firestore

```
Cloud SQL db-f1-micro:  ~$7/mies (zawsze uruchomiona)
Storage 10GB SSD:       ~$1.70/mies
Backup:                 ~$0.50/mies
Łącznie:                ~$9/mies

Vs Firestore:           $0/mies
Oszczędność:            ~$9/mies = ~$108/rok
```

---

## Monitoring kosztów

### Billing budget (jeśli masz uprawnienia)

```hcl
# tf/monitoring/main.tf
resource "google_billing_budget" "monthly" {
  count           = var.billing_account_id != "" ? 1 : 0
  billing_account = var.billing_account_id

  amount {
    specified_amount {
      currency_code = "USD"
      units         = "20"  # $20/mies limit
    }
  }

  threshold_rules {
    threshold_percent = 0.5   # alert przy $10
  }
  threshold_rules {
    threshold_percent = 1.0
    spend_basis       = "FORECASTED_SPEND"  # prognoza przekroczy limit
  }
}
```

### Ręczny monitoring

```bash
# Bieżący koszt miesiąca
gcloud beta billing projects describe PROJECT_ID

# Szczegółowy breakdown w konsoli
# → GCP Console → Billing → Cost breakdown → per service
```

---

## Ścieżki optymalizacji kosztów

| Gdy... | Rozważ... | Oszczędność |
|--------|-----------|-------------|
| Ruch wzrośnie powyżej Free tier API GW | Cache responses w Cloudflare | 50-80% mniej API calls |
| Cold start jest problemem | `min_instance_count = 1` | +$7/mies, -cold start |
| Ruch wyraźnie dobowy | `max_instance_count = 1` w nocy | nieistotne przy min=0 |
| Dużo Firestore reads | Agregacja w Cloud Run (cache w pamięci) | Redukcja reads |
| VPC za drogi przy małym projekcie | Usuń VPC Connector, Firestore przez publiczne API | -$7/mies |
