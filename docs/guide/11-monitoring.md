# Monitoring — alerty jako kod

Aplikacja bez monitoringu to aplikacja, w której awarie wykrywają użytkownicy. Cloud Monitoring z dashboardem, alertami i notyfikacjami email — wszystko jako kod Terraform.

---

## Co monitorujemy

```
┌──────────────────────────────────────────────────────────────┐
│  GPC Dashboard                                               │
├────────────────┬───────────────┬────────────────┬────────────┤
│ API GW         │ Cloud Run     │ Cloud Run      │ Cloud Run  │
│ req/min        │ latency p99   │ 5xx errors/min │ instances  │
├────────────────┴───────────────┴────────────────┴────────────┤
│ Firestore reads/s              │ Firestore writes/s          │
└─────────────────────────────────────────────────────────────-┘

Alerty:
  → 5xx errors > 5 w 5 min    → email
  → latency p99 > 3s przez 5 min → email
```

---

## Terraform — warstwa `tf/monitoring/`

### Notification channel — email

```hcl
resource "google_monitoring_notification_channel" "email" {
  display_name = "GPC Alerts — email"
  type         = "email"

  labels = {
    email_address = var.alert_email  # przekazywany przez TF_VAR_alert_email w CI
  }
}
```

### Alert — 5xx errors

```hcl
resource "google_monitoring_alert_policy" "alert_5xx" {
  display_name = "GPC — HTTP 5xx errors (>5/5min)"
  combiner     = "OR"

  conditions {
    display_name = "Cloud Run backend-api — 5xx count"

    condition_threshold {
      filter = join(" AND ", [
        "metric.type=\"run.googleapis.com/request_count\"",
        "resource.type=\"cloud_run_revision\"",
        "resource.labels.service_name=\"backend-api\"",
        "metric.labels.response_code_class=\"5xx\"",
      ])

      comparison      = "COMPARISON_GT"
      threshold_value = 5         # (1)
      duration        = "0s"      # (2)

      aggregations {
        alignment_period     = "300s"  # okno 5 minut
        per_series_aligner   = "ALIGN_SUM"
        cross_series_reducer = "REDUCE_SUM"
        group_by_fields      = []
      }

      trigger {
        count = 1
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.name]
}
```

1. `threshold_value = 5` — alert po 5 błędach 5xx w oknie 5 minut. Jeden błąd 5xx (np. cold start timeout) nie wysyła alertu.
2. `duration = "0s"` — warunek musi być spełniony przez 0 sekund przed alertem. W kombinacji z `alignment_period = 300s`: suma za ostatnie 5 minut > 5 → alert. Gdyby `duration = "300s"` i `alignment_period = "300s"` — alert tylko jeśli przez 10 minut > 5 błędów.

### Alert — latency p99

```hcl
resource "google_monitoring_alert_policy" "alert_latency" {
  display_name = "GPC — Cloud Run latency p99 > 3s"

  conditions {
    condition_threshold {
      filter = join(" AND ", [
        "metric.type=\"run.googleapis.com/request_latencies\"",
        "resource.type=\"cloud_run_revision\"",
        "resource.labels.service_name=\"backend-api\"",
      ])

      comparison      = "COMPARISON_GT"
      threshold_value = 3000      # milisekundy
      duration        = "300s"    # przez 5 minut ciągłe

      aggregations {
        alignment_period     = "300s"
        per_series_aligner   = "ALIGN_PERCENTILE_99"  # (1)
        cross_series_reducer = "REDUCE_MEAN"
      }
    }
  }
}
```

1. `ALIGN_PERCENTILE_99` — 99. percentyl latency. Nie średnia (która maskuje outliery). Jeśli 99% requestów odpowiada < 3s przez 5 minut — alert. Użytkownicy z powolnymi połączeniami nie triggerują fałszywych alertów.

### Budget alert — opcjonalny

```hcl
resource "google_billing_budget" "monthly" {
  count           = var.billing_account_id != "" ? 1 : 0  # (1)
  billing_account = var.billing_account_id

  budget_filter {
    projects = ["projects/${var.project_id}"]
  }

  amount {
    specified_amount {
      currency_code = "USD"
      units         = tostring(var.monthly_budget_usd)
    }
  }

  threshold_rules {
    threshold_percent = 0.5   # 50% budżetu
  }
  threshold_rules {
    threshold_percent = 0.8   # 80% budżetu
  }
  threshold_rules {
    threshold_percent = 1.0
    spend_basis       = "FORECASTED_SPEND"  # prognoza przekroczy limit
  }

  all_updates_rule {
    monitoring_notification_channels = [google_monitoring_notification_channel.email.name]
    schema_version                   = "1.0"
  }
}
```

1. `count = var.billing_account_id != "" ? 1 : 0` — zasób tworzony tylko gdy `billing_account_id` jest podany. Wymagane uprawnienia: `billing.budgetsAdmin` na billing account. Bez uprawnień: ustaw `GCP_BILLING_ACCOUNT_ID = ""` (lub nie ustawiaj) w GitHub Secrets.

### Dashboard — JSON w Terraform

```hcl
resource "google_monitoring_dashboard" "gpc" {
  dashboard_json = jsonencode({
    displayName = "GPC Dashboard"
    gridLayout = {
      columns = "2"
      widgets = [
        {
          title = "API Gateway — req/min"
          xyChart = {
            dataSets = [{
              timeSeriesQuery = {
                timeSeriesFilter = {
                  filter = "metric.type=\"apigateway.googleapis.com/http/request_count\""
                  aggregation = {
                    alignmentPeriod    = "60s"
                    perSeriesAligner   = "ALIGN_RATE"
                    crossSeriesReducer = "REDUCE_SUM"
                  }
                }
              }
            }]
          }
        },
        # ... pozostałe widgety (Cloud Run latency, 5xx, instances, Firestore)
      ]
    }
  })
}
```

Dashboard jako kod — wersjonowany, powtarzalny, widoczny w code review.

---

## Metryki kluczowe

| Metryka | Typ | Gdzie |
|---------|-----|-------|
| `apigateway.googleapis.com/http/request_count` | Counter | API Gateway |
| `run.googleapis.com/request_latencies` | Distribution | Cloud Run |
| `run.googleapis.com/request_count` | Counter | Cloud Run (z `response_code_class`) |
| `run.googleapis.com/container/instance_count` | Gauge | Cloud Run (aktywne instancje) |
| `firestore.googleapis.com/document/read_count` | Counter | Firestore |
| `firestore.googleapis.com/document/write_count` | Counter | Firestore |

---

## Free tier Cloud Monitoring

| Metryka | Limit Free tier |
|---------|----------------|
| Metryki ingested | 150 MB/mies |
| Custom metrics | Nie dotyczy (używamy system metrics) |
| Alert policies | Unlimited |
| Dashboard | Unlimited |
| Log-based metrics | 50 user-defined |

Dla tego projektu: wszystkie metryki są system metrics (GCP zbiera je automatycznie). Free tier w pełni wystarczy.

---

## Pułapki

!!! warning "disable_default_iam_alerts — usunięte w Google provider v5"
    Starsze przykłady z internetu zawierają `disable_default_iam_alerts = true` w `all_updates_rule`. W Google Terraform Provider v5 to pole zostało usunięte. Dodanie go zwróci błąd.

!!! tip "Eksportowanie dashboardu do JSON"
    Jeśli zmodyfikujesz dashboard ręcznie w konsoli i chcesz zapisać zmiany do TF:
    ```bash
    gcloud monitoring dashboards describe DASHBOARD_ID --format=json
    ```
    Skopiuj pole `gridLayout` do `dashboard_json` w TF.
