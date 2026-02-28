# Platform Engineering Guide

Jak zbudować produkcyjną platformę cloud od zera — krok po kroku.

**Live docs:** https://benhornbeam.github.io/platform-engineering-guide/

---

## Co znajdziesz w tym przewodniku

Kompletny walkthrough budowania platformy opartej na:
- **Terraform** (wielowarstwowy IaC, 8 warstw)
- **Google Cloud Platform** (Cloud Run, API Gateway, Firestore, GCS, Identity Platform)
- **GitHub Actions** (keyless CI/CD przez Workload Identity Federation)
- **Cloudflare** (CDN, SSL, DDoS protection za $0)

Każda decyzja architektoniczna jest udokumentowana — co wybraliśmy, co odrzuciliśmy i dlaczego.

## Rozdziały

| # | Rozdział | Tematyka |
|---|---------|---------|
| 00 | Intro | Architektura end-to-end, stos, koszty |
| 01 | Prerequisites | Konta, narzędzia |
| 02 | Bootstrap | WIF keyless CI/CD, separacja warstw TF |
| 03 | Networking | VPC, firewall, VPC Connector |
| 04 | Backend | Cloud Run, FastAPI, Docker, Artifact Registry |
| 05 | Auth | Identity Platform, Firebase Auth, JWT flow |
| 06 | API Gateway | OpenAPI spec, JWT validation, CORS |
| 07 | Database | Firestore, model danych, free tier |
| 08 | Frontend | GCS bucket, Cloudflare CDN, CNAME trick |
| 09 | CI/CD | GitHub Actions, SHA tagging, auto-deploy |
| 10 | Staging | Multi-env, branch→env mapping |
| 11 | Monitoring | Cloud Monitoring, alerty jako kod |
| 12 | Security | IAM checklist, świadome kompromisy |
| 13 | Cost Analysis | Rzeczywiste koszty, free tier, alternatywy |

## Szacowany koszt platformy

~$7–8/mies (praktycznie tylko VPC Connector). Cała reszta mieści się w Free tier.
