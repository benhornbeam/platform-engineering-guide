# Platform Engineering Guide

Jak zbudować produkcyjną platformę cloud od zera — krok po kroku, bez pomijania trudnych decyzji.

---

## Co znajdziesz w tym przewodniku

Kompletny walkthrough budowania platformy na Google Cloud Platform, w pełni zautomatyzowanej przez Infrastructure as Code i CI/CD bez kluczy serwisowych.

```
[Użytkownik] → Cloudflare CDN → GCS bucket (SPA)
                                      │ JWT
                                      ▼
                              API Gateway (walidacja JWT)
                                      │ OIDC
                                      ▼
                              Cloud Run (FastAPI)
                                      │ VPC
                                      ▼
                              Firestore

GitHub Actions (WIF, keyless) → deploy całej infrastruktury
```

**Stos:** Terraform · Google Cloud Run · API Gateway · Identity Platform · Firestore · GCS · Cloudflare CDN · GitHub Actions (WIF)

**Koszt:** ~$7–8/mies (praktycznie tylko VPC Connector — cała reszta mieści się w Free tier)

---

## Rozdziały

=== "Fundamenty"

    | Rozdział | Tematyka |
    |---------|---------|
    | [Bootstrap — keyless CI/CD](guide/02-bootstrap.md) | Workload Identity Federation, zero kluczy SA, separacja warstw TF |
    | [Sieć — VPC i izolacja](guide/03-networking.md) | VPC, firewall deny-all, VPC Connector, IAM vs sieciowa izolacja |

=== "Warstwa aplikacyjna"

    | Rozdział | Tematyka |
    |---------|---------|
    | [Backend — Cloud Run + FastAPI](guide/04-backend.md) | Kontener, Artifact Registry, JWT decode, chicken-and-egg deploy |
    | [Auth — Identity Platform + JWT](guide/05-auth.md) | Google SSO, Firebase SDK, JWT flow, pułapki |
    | [API Gateway](guide/06-api-gateway.md) | OpenAPI spec, CORS preflight, walidacja JWT, google-beta |
    | [Baza danych — Firestore](guide/07-database.md) | NoSQL vs SQL, 1 zasób TF, model danych, free tier |
    | [Frontend — GCS + Cloudflare CDN](guide/08-frontend.md) | CNAME trick, SPA routing, cache-control |

=== "Automatyzacja i operacje"

    | Rozdział | Tematyka |
    |---------|---------|
    | [CI/CD — GitHub Actions](guide/09-cicd.md) | deploy.yml, SHA tagging, auto-deploy per branch |
    | [Staging](guide/10-staging.md) | Multi-env, suffix pattern, branch→env mapping |
    | [Monitoring](guide/11-monitoring.md) | Dashboard, alert policies, billing budget jako kod |
    | [Security Review](guide/12-security.md) | IAM checklist, świadome kompromisy |
    | [Analiza kosztów](guide/13-cost-analysis.md) | Free tier, porównanie z alternatywami |

---

## Zacznij tutaj

Jeśli zaczynasz od zera:

[Wymagania wstępne →](guide/01-prerequisites.md){ .md-button }
[Czego się nauczysz →](guide/00-intro.md){ .md-button .md-button--primary }
