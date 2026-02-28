# Sieć — VPC i izolacja

Sieć jest zazwyczaj pierwszą warstwą Terraform po bootstrapie — kolejne warstwy (backend, VPC Connector) do niej odwołują się przez nazwę.

---

## Cel warstwy

Izolacja backendu od publicznego internetu. Cloud Run ma `ingress=ALL` (wymagane przez API Gateway), ale przez VPC Connector wychodzi tylko do prywatnej sieci. Użytkownicy nigdy nie docierają do Cloud Run bezpośrednio.

```
Internet
    │ tylko port 443 — przez API Gateway
    ▼
[API Gateway] → IAM check → [Cloud Run ingress=ALL]
                                     │ VPC Connector egress
                                     │ (10.8.0.0/28)
                                     ▼
                              [prototype-vpc]
                              subnet-warsaw: 10.0.1.0/24
                                     │ private_ip_google_access
                                     ▼
                              [Firestore / Secret Manager]
                              (Google APIs bez publicznego IP)
```

---

## Co tworzy warstwa `tf/infra/`

Trzy zasoby. To celowo mała warstwa — fundament zmienia się rzadko.

```hcl
# tf/infra/main.tf

# VPC z wyłączonym auto_create_subnetworks
resource "google_compute_network" "main_vpc" {
  name                    = "prototype-vpc"
  auto_create_subnetworks = false  # (1)
}

# Podsieć w Warszawie — /24 = 254 adresy
resource "google_compute_subnetwork" "subnet_pl" {
  name          = "subnet-warsaw"
  ip_cidr_range = "10.0.1.0/24"
  region        = var.region          # europe-central2
  network       = google_compute_network.main_vpc.id

  private_ip_google_access = true     # (2)
}

# Jawna reguła deny-all — dokumentuje intent w kodzie
resource "google_compute_firewall" "deny_all_ingress" {
  name      = "deny-all-ingress"
  network   = google_compute_network.main_vpc.name
  priority  = 65534                   # (3)
  direction = "INGRESS"

  deny {
    protocol = "all"
  }

  source_ranges = ["0.0.0.0/0"]
}
```

1. `auto_create_subnetworks = false` — bez tego GCP tworzy subsieci w każdym regionie (default VPC). Utrudnia audyt sieci i zwiększa ryzyko nieoczekiwanego egress między regionami.
2. `private_ip_google_access = true` — instancje bez publicznego IP mogą łączyć się z Google APIs (Firestore, Secret Manager, Artifact Registry). Bez tego musiałbyś używać Cloud NAT lub publicznych IP dla każdej maszyny w podsieci.
3. `priority = 65534` — GCP ma implicit deny-all na priorytecie 65535. Ta reguła (65534) jest o jeden wyżej, ale jest **eksplicytna** — pojawia się w audytach IAM, raportach compliance i `gcloud compute firewall-rules list`. Implicit reguły są niewidoczne.

---

## VPC Connector — łącznik Cloud Run z VPC

VPC Connector tworzy się w warstwie `tf/backend/`, ale koncepcyjnie jest częścią sieci.

```hcl
# tf/backend/main.tf — fragment

resource "google_vpc_access_connector" "connector" {
  name           = "backend-connector"
  network        = "prototype-vpc"
  ip_cidr_range  = "10.8.0.0/28"   # (1)
  region         = var.region        # europe-central2
  min_throughput = 200               # Mbps (minimum reserved)
  max_throughput = 300               # Mbps (maximum)
}
```

1. `/28` = 16 adresów. VPC Connector potrzebuje dedykowanej podsieci, która **nie nakłada się** na `subnet-warsaw` (10.0.1.0/24). Używamy `10.8.0.0/28` — inna pula RFC 1918.

```hcl
# Konfiguracja w Cloud Run service
vpc_access {
  connector = google_vpc_access_connector.connector.id
  egress    = "ALL_TRAFFIC"  # (1)
}
```

1. `ALL_TRAFFIC` — cały ruch wychodzi przez VPC, nie tylko RFC 1918. Alternatywa: `PRIVATE_RANGES_ONLY` — tylko ruch do prywatnych IP przez VPC, publiczny przez internet. Wybraliśmy `ALL_TRAFFIC` dla pełnej izolacji i kontroli.

---

## Dlaczego IAM zamiast sieci do izolacji Cloud Run?

Pierwotna architektura zakładała `ingress=INTERNAL_LOAD_BALANCER`. Problem: API Gateway **nie jest** Load Balancerem w sensie GCP — nie może routować ruchu do Cloud Run z `ingress=INTERNAL`.

Możliwe rozwiązanie alternatywne: Private Service Connect (PSC) — bardziej zaawansowane, wymaga dodatkowej konfiguracji. Dla prototypu overengineering.

Nasze rozwiązanie: `ingress=ALL` + izolacja przez IAM.

```
Atakujący z internetu
    │ próbuje wywołać Cloud Run URL bezpośrednio
    ▼
Cloud Run sprawdza IAM: czy caller ma roles/run.invoker?
    → NIE (brak tokenu api-gateway-sa)
    → 403 Forbidden
```

Cloud Run jest technicznie dostępny z internetu, ale bez tokenu SA `api-gateway-sa` z rolą `roles/run.invoker` — każde żądanie zwróci 403. To **izolacja przez IAM**, nie sieciowa. Dla compliance-heavy środowisk z PCI-DSS lub ISO 27001 — rozważ Private Service Connect lub `ingress=INTERNAL_LOAD_BALANCER` z Cloud Load Balancing.

---

## Pułapki

!!! danger "API Gateway ≠ region VPC Connector"
    API Gateway działa tylko w wybranych regionach. `europe-central2` (Warszawa) **nie jest wspierany**. Gateway musi być w `europe-west1` (Belgia). VPC Connector i Cloud Run są w `europe-central2`. Komunikacja między nimi idzie przez Google backbone — nie przez publiczny internet (~10ms dodatkowe opóźnienie).

!!! warning "VPC Connector: stały koszt ~$7/mies"
    VPC Connector nalicza opłatę nawet przy 0 requestach (`min_throughput = 200` Mbps = reserved capacity). To największy stały koszt tej architektury. Bez VPC Connectora (np. Firestore przez publiczne API bezpośrednio) — koszt infrastruktury = ~$0/mies.

!!! tip "Firewall rules nie dotyczą Cloud Run"
    Reguła `deny-all-ingress` **nie wpływa** na Cloud Run. Cloud Run to usługa zarządzana — działa poza VPC. Reguły firewall dotyczą zasobów wewnątrz VPC (Compute Engine, GKE nodes). VPC Connector jest bramą egress z Cloud Run **do** VPC, nie bramą ingress.
