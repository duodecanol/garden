---
type: research
status: draft
date: 2026-03-14
tags:
  - type/research
  - topic/dev
  - topic/infrastructure
  - topic/database
topics:
  - serverless
  - postgres
  - hosting
  - database
related: []
sources:
  - https://dev.to/philip_mcclarence_2ef9475/best-postgresql-hosting-in-2026-rds-vs-supabase-vs-neon-vs-self-hosted-5fkp
  - https://www.leanware.co/insights/supabase-vs-neon
  - https://northflank.com/blog/best-postgresql-hosting-providers
  - https://designrevision.com/blog/supabase-vs-neon
  - https://makerkit.dev/blog/tutorials/best-database-software-startups
  - https://www.bytebase.com/blog/understanding-aws-aurora-pricing/
  - https://aiven.io/pricing
publish: true
---

# Managed Serverless Postgres Hosting 비교

편의성과 가격 면에서 우수한 서비스를 중심으로 비교 분석한 리서치 노트.

## Summary

> [!abstract]
> **편의성 + 가격 최적 추천:**
> - **순수 DB만 필요** → ==Neon== (scale-to-zero, 브랜칭, 사용량 기반 과금)
> - **풀스택 백엔드 필요** → ==Supabase== (Auth + Storage + Realtime + API 포함)
> - **예산 최소화** → ==DigitalOcean== (고정 요금, 숨겨진 비용 없음)

---

## Top 3 추천 서비스

### 1. Neon — 진정한 Serverless Postgres

> [!tip] 최고의 편의성 + 비용 효율
> Scale-to-zero로 유휴 시간에는 비용이 0. 사이드 프로젝트나 변동 트래픽에 최적.

- **핵심 기능**: Compute와 Storage 완전 분리, scale-to-zero, DB 브랜칭 (Git처럼)
- **Free Tier**: 100 compute-hours/월, 0.5GB 스토리지, 최대 10 프로젝트
- **유료**: Launch $19/월 (300 compute-hours 포함)
- **Cold start**: 300-500ms (scale-to-zero 사용 시)
- **지원 Extension**: `pgvector`, `PostGIS`, `pg_stat_statements`
- **강점**: Vercel 통합 우수, CI/CD에서 브랜칭 활용 가능
- **약점**: Write-heavy 워크로드에서 레이턴시 증가 가능, TimescaleDB 미지원

**가격표:**

| 구성 | 월 비용 |
|------|---------|
| Dev: Free tier (0.25 CU, 0.5GB) | $0 |
| Prod: Launch (최대 4 CU, 50GB) | ~$40-80 |
| Scale: Scale (최대 10 CU, 200GB) | ~$150-400 |

---

### 2. Supabase — PostgreSQL + 풀스택 BaaS

> [!info] DB 이상의 풀 백엔드가 필요하다면
> Auth, Storage, Realtime, Edge Functions, auto-generated REST API 모두 포함.

- **핵심 기능**: PostgREST 자동 API, GoTrue 인증, Realtime WebSocket, File Storage
- **Free Tier**: 500MB DB, 50,000 MAU, 2 프로젝트 (1주 비활성 시 pause)
- **유료**: Pro $25/월 (8GB DB, 100K MAU, 일일 백업)
- **지원 Extension**: `pgvector`, `PostGIS`, `pg_cron`, `pg_stat_statements` 등 다수
- **강점**: 코드 없이 즉시 CRUD API 생성, 클라이언트 SDK (JS/Python/Dart/Swift/Kotlin)
- **약점**: Free tier 1주 비활성 시 pause, Multi-AZ 미지원, 고정 compute (scale-to-zero 없음)

**가격표:**

| 구성 | 월 비용 |
|------|---------|
| Dev: Free tier (shared CPU, 500MB) | $0 |
| Prod: Pro + Small (2 vCPU, 4GB, 50GB) | ~$75 |
| Scale: Pro + XL (8 vCPU, 32GB, 200GB) | ~$350 |

---

### 3. Aiven for PostgreSQL — 멀티클라우드 매니지드 Postgres

> [!note] 멀티클라우드 + 오픈소스 데이터 플랫폼
> AWS/GCP/Azure/DO/UpCloud 어디서든 동일한 경험. Kafka, Redis 등 통합 데이터 스택.

- **핵심 기능**: 멀티클라우드 배포, HA(2~3 VM 클러스터), 동적 디스크 사이징, 벡터 검색 지원
- **Free Tier**: 1 CPU, 1GB RAM, 1GB 스토리지 (무기한, 카드 불필요)
- **Developer Tier**: $5/월 (1 CPU, 1GB RAM, 8GB 스토리지)
- **지원 Extension**: `pgvector`, `PostGIS`, `pg_stat_statements` 등 전체 호환
- **강점**: 클라우드 벤더 락인 없음, Kafka/Redis/ClickHouse 등 통합 관리, 99.99% SLA (Startup+)
- **약점**: Serverless 아님 (고정 VM 기반), Free tier 스토리지 1GB로 매우 제한적, scale-to-zero 미지원

**가격표:**

| 플랜 | VM | 스펙 | 월 비용 |
|------|-----|------|---------|
| Free | 1 | 1 CPU, 1GB RAM, 1GB | $0 |
| Developer | 1 | 1 CPU, 1GB RAM, 8GB | $5 |
| Hobbyist | 1 | 1 CPU, 1GB RAM, 8GB | $12 |
| Startup-4 | 1 | 2 CPU, 4GB RAM, 80GB | $75 |
| Business-4 | 2 (HA) | 2 CPU, 4GB RAM, 80GB | $180 |
| Premium-4 | 3 (HA) | 2 CPU, 4GB RAM, 80GB | $270 |

---

### 4. Aurora Serverless v2 — AWS 네이티브 Serverless

> [!warning] 강력하지만 비싸다
> AWS 생태계에 깊이 통합된 엔터프라이즈급 serverless. 단, 최소 비용 ~$43/월.

- **핵심 기능**: ACU 기반 자동 스케일링 (0.5~128 ACU), Multi-AZ 기본, 15개 읽기 레플리카
- **Free Tier**: 없음
- **최소 비용**: 0.5 ACU 상시 가동 ≈ **~$43/월** (compute만, storage/IO 별도)
- **과금 단위**: $0.12/ACU-hour (1 ACU ≈ 2GB RAM), Storage $0.10/GB-month, IO $0.20/백만건
- **지원 Extension**: `pgvector`, `PostGIS`, `pg_stat_statements`, `pg_cron` 등 RDS 수준
- **강점**: AWS 서비스 통합 (IAM, Lambda, S3), 128TB 자동 스토리지, 초 단위 과금
- **약점**: ==scale-to-zero 불가== (최소 0.5 ACU 상시), 가격 구조 복잡 (compute+storage+IO+transfer), 비용 예측 어려움

**가격 시뮬레이션:**

| 구성 | 월 비용 (us-east-1) |
|------|---------------------|
| 최소 유휴 (0.5 ACU, 20GB) | ~$45 (compute $43 + storage $2) |
| 소규모 Prod (평균 2 ACU, 50GB) | ~$180 |
| 중규모 Prod (평균 8 ACU, 200GB, IO 많음) | ~$720+ |

> [!danger] 비용 주의
> Aurora Serverless v2는 0.5 ACU가 최소이므로 **진정한 scale-to-zero가 아님**. Neon 대비 유휴 시 비용이 ~$43/월 vs $0. IO가 많으면 비용이 급증할 수 있으므로 I/O-Optimized 모드 전환 검토 필요.

---

### 5. DigitalOcean Managed Databases — 심플하고 예측 가능

> [!note] 숨겨진 비용 없는 고정 요금제
> 가격에 compute, storage, data transfer 모두 포함. IOPS 별도 과금 없음.

- **핵심 기능**: 고정 가격, 간편 설정 (5분 내 프로덕션 클러스터), 빌트인 커넥션 풀링
- **가격**: Dev $25/월 ~ Prod $60/월 ~ Scale $375/월
- **강점**: 가격 투명성, 세팅 간편함
- **약점**: 최대 64GB RAM 한계, 리전 12개로 제한, 기본 모니터링만 제공

---

## 전체 가격 비교표

| Provider | Dev (2 vCPU, 4GB, 20GB) | Prod (4 vCPU, 32GB, 100GB) | Scale (16 vCPU, 128GB, 500GB) |
|----------|------------------------|---------------------------|-------------------------------|
| **Neon** | $0 (free) | ~$40-80 | ~$150-400 |
| **Supabase** | $0 (free) | ~$75 | ~$350 |
| **Aiven** | $0 (free, 1GB) | ~$75 (Startup) | ~$375+ |
| **Aurora Serverless v2** | ~$45 (0.5 ACU) | ~$180 (2 ACU avg) | ~$720+ (8 ACU avg) |
| **DigitalOcean** | ~$25 | ~$60 | ~$375 |
| **AWS RDS** | ~$60 | ~$580 (Multi-AZ) | ~$2,100 (Multi-AZ) |
| **Google Cloud SQL** | ~$10 | ~$560 (HA) | ~$2,000 (HA) |
| **Azure Flexible** | ~$25 | ~$460 (HA) | ~$1,900 (HA) |
| **Render** | $0 (free) | ~$35 | ~$200 |
| **Self-hosted (Hetzner)** | ~$5 | ~$45 | ~$85 |

---

## 선택 가이드

> [!question] 어떤 상황에서 어떤 서비스?

| 상황 | 추천 |
|------|------|
| 사이드 프로젝트, 변동 트래픽 | **Neon** (scale-to-zero) |
| 풀스택 앱 (Auth+API 필요) | **Supabase** |
| 심플한 프로덕션 DB | **DigitalOcean** |
| CI/CD에서 DB 브랜칭 필요 | **Neon** |
| 검색 중심 앱 | **Xata** (Postgres + OpenSearch) |
| 엔터프라이즈/규정 준수 필수 | **AWS RDS** 또는 **Cloud SQL** |
| AWS 락인 OK + 자동 스케일링 | **Aurora Serverless v2** |
| 멀티클라우드 + 데이터 스택 통합 | **Aiven** |
| 최저 비용 + 풀 컨트롤 | **Self-hosted (Hetzner)** |
| Vercel 프로젝트 | **Neon** (공식 통합) |

---

## Key Insights

> [!important] 핵심 판단 기준
> - **"DB만 필요한가, 백엔드 전체가 필요한가?"** 가 가장 중요한 갈림길
> - Neon과 Supabase 모두 standard PostgreSQL이므로 나중에 마이그레이션 가능
> - 3대 클라우드(AWS/GCP/Azure)는 기능은 안정적이나 가격이 5~10배 비쌈
> - Free tier 비교: Neon(100 compute-hours) > Supabase(500MB, 1주 pause) > Aiven(1GB) > Render(256MB)
> - Aurora Serverless v2는 "serverless"이지만 ==scale-to-zero 불가== (최소 $43/월), Neon은 진정한 $0 유휴 가능
> - Aiven은 serverless가 아닌 전통적 managed DB이지만, 멀티클라우드 전략 시 유일한 선택지

## Sources

- [Best PostgreSQL Hosting in 2026 — DEV Community](https://dev.to/philip_mcclarence_2ef9475/best-postgresql-hosting-in-2026-rds-vs-supabase-vs-neon-vs-self-hosted-5fkp)
- [Supabase vs Neon Comparison — Leanware](https://www.leanware.co/insights/supabase-vs-neon)
- [Best PostgreSQL Hosting Providers — Northflank](https://northflank.com/blog/best-postgresql-hosting-providers)
- [Neon vs Supabase Benchmarks — DesignRevision](https://designrevision.com/blog/supabase-vs-neon)
- [Best Database Software for Startups — MakerKit](https://makerkit.dev/blog/tutorials/best-database-software-startups)
- [AWS Aurora Pricing Explained — Bytebase](https://www.bytebase.com/blog/understanding-aws-aurora-pricing/)
- [Aiven Pricing — Aiven](https://aiven.io/pricing)
- [Aiven Developer Tier — Aiven](https://aiven.io/developer-tier)
