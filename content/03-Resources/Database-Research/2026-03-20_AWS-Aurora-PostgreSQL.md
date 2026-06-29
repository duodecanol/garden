---
type: research
status: draft
date: 2026-03-20
tags:
  - type/research
  - topic/dev
  - topic/database
topics:
  - aurora
  - postgresql
  - aws
  - serverless
  - database
related:
  - "[[2026-03-14_Managed-Serverless-Postgres-Hosting]]"
  - "[[2026-03-14_Aiven-Deep-Dive]]"
  - "[[2026-03-20_Aiven-PG-Extensions]]"
  - "[[2026-03-20_Aiven-Fork-vs-Neon-Branch]]"
sources:
  - https://aws.amazon.com/rds/aurora/pricing/
  - https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html
  - https://www.vantage.sh/blog/aurora-serverless-v2-vs-neon-pricing
  - https://cloudfix.com/blog/aurora-io-optimized-vs-standard/
notebooklm_notebook: AWS Aurora PostgreSQL - 제품 조사
publish: true
---

# AWS Aurora PostgreSQL 제품 조사

관련: [[2026-03-14_Managed-Serverless-Postgres-Hosting]] · [[2026-03-14_Aiven-Deep-Dive]] · [[2026-03-20_Aiven-Fork-vs-Neon-Branch]]

---

## 결론

> [!important] 핵심 요약
> Aurora PostgreSQL Serverless v2는 **엔터프라이즈급 AWS 워크로드에 최적화**된 관리형 PostgreSQL이다.
>
> - **강점**: 무중단 오토스케일링, 3-AZ 분산 스토리지, 최대 15개 Read Replica, AWS 생태계 통합
> - **약점**: 최소 0.5 ACU 비용(~$43.80/월)으로 진정한 Scale-to-zero 미지원, 최신 PG 버전 도입 지연
> - **Neon 대비**: 개발/프리뷰 환경엔 Neon이 유리, 프로덕션 AWS 스택엔 Aurora가 적합
> - **Aiven 대비**: Aurora는 AWS 생태계 통합 우위, Aiven은 멀티클라우드·최신 PG 버전 도입 속도 우위

---

## 아키텍처 & 핵심 특징

### Aurora vs 일반 RDS 차이

```
Aurora 아키텍처:
  컴퓨팅(Writer/Reader) ←→ 분산 스토리지 레이어 (3 AZ, 자동 복제)
  스토리지: 최대 128TB 자동 확장, 사전 프로비저닝 불필요
```

| 항목           | Aurora PostgreSQL         | 일반 RDS PostgreSQL   |
| ------------ | ------------------------- | ------------------- |
| 스토리지         | 3 AZ 분산, 자동 확장 (최대 128TB) | EBS 볼륨, 사전 프로비저닝 필요 |
| 스케일링         | 무중단 오토스케일링                | 인스턴스 재부팅 필요         |
| Read Replica | 최대 15개                    | 최대 5개               |
| 페일오버 속도      | 매우 빠름 (스토리지 레이어 HA)       | 더 느림 (EBS 동기화)      |
| 비용           | 컴퓨팅+스토리지+I/O 별도 과금        | 인스턴스+EBS+IOPS       |

### Serverless v2 스케일링

> [!tip] Serverless v2의 스케일링 특징
> - **최소 단위**: 0.5 ACU (최근 업데이트로 0 ACU Auto-pause 지원)
> - **확장 속도**: 즉각적, 트랜잭션 중단 없음
> - **축소 속도**: 이전 v1 대비 최대 15배 빠름
> - 초당 수십만 TPS 수준까지 확장 가능

---

## 가격 구조

### 기본 단가 (us-east-1 기준)

| 항목       | Standard                         | I/O-Optimized         |
| -------- | -------------------------------- | --------------------- |
| **컴퓨팅**  | ~$0.12/ACU-hr                    | ~$0.156/ACU-hr (+30%) |
| **스토리지** | ~$0.10/GB-월                      | ~$0.225/GB-월 (+125%)  |
| **I/O**  | ~$0.20/백만 요청                     | **무료**                |
| **백업**   | 클러스터 크기 100% 무료, 초과분 $0.021/GB-월 | 동일                    |

### Standard vs I/O-Optimized 선택 기준

> [!warning] I/O 비용 폭탄 주의
> Standard 구성에서 I/O가 예기치 않게 급증하면 청구서가 폭발할 수 있다.
> **I/O 비용이 전체 Aurora 청구액의 25%를 초과하면 I/O-Optimized로 전환**하라.
> 전환은 다운타임 없이 콘솔에서 즉시 적용 가능.

#### 월 비용 시뮬레이션 (500GB 스토리지, 컴퓨팅 기본 $211.70 가정)

| 시나리오      | Standard                           | I/O-Optimized                        | 결론                         |
| --------- | ---------------------------------- | ------------------------------------ | -------------------------- |
| 5억 I/O/월  | $211.70 + $50 + $100 = **$361.70** | $275.21 + $112.50 + $0 = **$387.71** | Standard 유리                |
| 10억 I/O/월 | $211.70 + $50 + $200 = **$461.70** | **$387.71** (고정)                     | I/O-Optimized 유리 (~16% 절감) |
| 20억 I/O/월 | $211.70 + $50 + $400 = **$661.70** | **$387.71** (고정)                     | I/O-Optimized 유리 (~40% 절감) |

### 소규모 개발 환경 최소 비용

> [!note] 개발 환경 월 최소 비용
> Serverless v2, 0.5 ACU 고정, 최소 스토리지 기준:
> - 컴퓨팅: 0.5 ACU × $0.12 × 730시간 = **$43.80/월**
> - 스토리지: 별도 추가
> - → **사실상 월 $45+ 최소 발생** (Neon 무료 티어, Aiven Free 대비 불리)

---

## Google Cloud 비교

### Aurora vs AlloyDB vs Cloud SQL

| 항목               | Aurora PostgreSQL               | AlloyDB (GCP)                     | Cloud SQL (GCP)        |
| ---------------- | ------------------------------- | --------------------------------- | ---------------------- |
| **성능**           | 즉각적 오토스케일링, 수십만 TPS             | OLTP 4배↑, OLAP 100배↑ (vs 표준 PG)   | OLTP ~4,800 TPS (벤치마크) |
| **컴퓨팅 단가**       | 기준                              | Cloud SQL Enterprise Plus 대비 +39% | 기준                     |
| **스토리지 단가**      | $0.10/GB (Standard)             | ~$0.338/GB                        | ~$0.17/GB (SSD)        |
| **I/O 과금**       | 별도 과금 (Standard) / 무료 (I/O-Opt) | **무료**                            | **무료**                 |
| **예산 예측성**       | I/O 폭탄 리스크 있음                   | 높음                                | **가장 높음**              |
| **Read Replica** | 최대 15개                          | 최대 20개                            | 최대 10개                 |
| **관리형 연결 풀**     | RDS Proxy (별도 비용)               | 내장                                | **PgBouncer 기본 내장**    |
| **GCP 통합**       | ✗                               | ✅                                 | ✅                      |
| **AWS 통합**       | ✅                               | ✗                                 | ✗                      |

> [!tip] 선택 가이드
> - **AWS 스택**: Aurora PostgreSQL
> - **GCP + 일반 워크로드**: Cloud SQL (예측 가능한 비용, PgBouncer 내장)
> - **GCP + 극한 성능 필요**: AlloyDB (HTAP, 대규모 엔터프라이즈)

### AlloyDB vs Aurora 세부

- **AlloyDB**: OLTP 성능 4배, OLAP 100배(컬럼형 엔진), I/O 무료, 단가 높음
- **Aurora**: AWS 에코시스템 통합(IAM 인증, Lambda, S3, Bedrock), Serverless v2 오토스케일링
- **공통**: 멀티-AZ HA, 교차 리전 복제, 128TB 스토리지 확장

---

## Aurora vs Neon Serverless

| 항목                | Aurora Serverless v2                 | Neon Serverless                     |
| ----------------- | ------------------------------------ | ----------------------------------- |
| **Scale-to-zero** | ✅ (최근 Auto-pause 추가, 이전엔 0.5 ACU 하한) | ✅ (완벽 지원, 콜드스타트 300~500ms)          |
| **월 최소 비용**       | ~$43.80 (0.5 ACU 기준)                 | **무료** (Free 티어: 컴퓨팅 100hr, 0.5GB)  |
| **유료 시작 가격**      | ~$43.80+                             | $19/월 (Launch 플랜)                   |
| **브랜칭**           | 클론(Cloning) 지원                       | ✅ Copy-on-Write 브랜칭 (수초, CI/CD 최적화) |
| **AWS 통합**        | ✅ 깊은 통합                              | ✗                                   |
| **개발/프리뷰 적합성**    | ⚠️ 비용 발생                             | ✅ 압도적 유리                            |
| **프로덕션 확장성**      | ✅ 엔터프라이즈 수준                          | 상대적으로 제한적                           |

> [!warning] 결론
> 개발/테스트/프리뷰 환경이라면 **Neon**이 압도적으로 유리.
> AWS 프로덕션 스택과 함께 쓴다면 **Aurora**가 자연스러운 선택.

---

## Aurora vs Aiven PostgreSQL

| 항목                | Aurora PostgreSQL  | Aiven PostgreSQL              |
| ----------------- | ------------------ | ----------------------------- |
| **PostgreSQL 버전** | 최신 PG18 도입 지연 가능성  | ✅ PG18 지원 (uuid7 네이티브 사용 가능)  |
| **uuid7 지원**      | ⚠️ PG18 미지원 시 불가   | ✅ PG18 → `uuidv7()` 기본 함수     |
| **클라우드**          | AWS 전용             | AWS/GCP/Azure 멀티클라우드          |
| **BYOC**          | 없음 (AWS 자체 인프라)    | ✅ (AWS/GCP, commitment 계약 필요) |
| **서버리스**          | ✅ Serverless v2    | ❌ (프로비저닝 방식)                  |
| **브랜칭**           | 클론 지원 (Neon 수준 아님) | Forking (분~수십 분, 새 서비스 생성)    |
| **무료 티어**         | ✗                  | ✅ (Free 플랜)                   |
| **AWS 통합**        | ✅ 깊은 통합            | 제한적                           |

---

## Extension 지원

### Aurora 주요 지원 Extension

- `PostGIS` — 공간 데이터
- `pg_partman` — 파티션 관리
- `pg_cron` — 스케줄링
- `pgAudit` — 감사 로깅
- `pglogical` — 논리적 복제
- `pg_stat_statements` — SQL 실행 통계
- `postgres_fdw`, `log_fdw` — 외부 데이터 래퍼
- `Babelfish` — SQL Server → Aurora 마이그레이션

> [!note] TLE (Trusted Language Extensions)
> Aurora 환경에서는 임의 Extension을 소스 컴파일로 설치할 수 없다.
> 대신 **TLE**를 통해 엔진에 영향 없이 커스텀 Extension을 안전하게 구축·실행 가능.

### uuid7 현황

> [!warning] Aurora의 uuid7 현황
> - `pg_uuidv7` Extension: Aurora 미지원
> - PG18 네이티브 `uuidv7()`: **Aurora의 PG18 지원 여부에 따라 결정**
> - Aurora는 최신 PG 메이저 버전 도입이 Aiven보다 늦는 경향이 있음
> - PG17 이하 사용 중이라면 uuid7 사용 불가 → `uuid-ossp`(v1~v5)만 가능

---

## 평가 & 추천

> [!tip] Aurora PostgreSQL이 적합한 경우
> - AWS 생태계(Lambda, S3, IAM, Bedrock)와 깊게 통합된 프로덕션 서비스
> - 예측 불가한 트래픽 피크가 있는 워크로드 (Serverless v2 오토스케일링)
> - 글로벌 서비스 (Aurora Global Database, 다중 리전 DR)
> - 엔터프라이즈 HA/DR 요구사항 (Multi-AZ, 15 Read Replica)

> [!warning] Aurora PostgreSQL이 부적합한 경우
> - 개발/테스트/프리뷰 환경 (비용 → Neon/Aiven Free 권장)
> - 최신 PostgreSQL 기능(PG18 uuid7 등) 즉시 필요한 경우 (→ Aiven 권장)
> - AWS가 아닌 멀티클라우드 전략 (→ Aiven 권장)
> - 소규모 스타트업 초기 단계 (월 $45+ 최소 비용 부담)

---

## Sources

| # | 제목 | 출처 |
|---|------|------|
| 1 | Amazon Aurora Pricing | [AWS 공식](https://aws.amazon.com/rds/aurora/pricing/) |
| 2 | Using Aurora Serverless v2 | [AWS Docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html) |
| 3 | Aurora Serverless v2 vs Neon Pricing | [Vantage](https://www.vantage.sh/blog/aurora-serverless-v2-vs-neon-pricing) |
| 4 | Aurora I/O-Optimized vs Standard | [CloudFix](https://cloudfix.com/blog/aurora-io-optimized-vs-standard/) |
| 5 | Aurora DSQL vs Serverless v2 비용 분석 | DoiT |
| 6 | Aurora Serverless v2 Dev 환경 최소 비용 | AWS re:Post |
| 7 | AWS RDS vs Google Cloud SQL 비교 2026 | SQLFlash |
| 8 | Understanding Google Cloud AlloyDB Pricing | GCP 공식 |
| 9 | Best Managed PostgreSQL 2026 | DEV Community |
| 10 | PostgreSQL 18 UUIDv7 지원 탐구 | Aiven Blog |

---

## Development Metadata

| Field | Value |
|-------|-------|
| NotebookLM notebook | AWS Aurora PostgreSQL - 제품 조사 |
| NotebookLM notebook ID | 98ea6cd3-8c0c-442d-9377-2cc53135289b |
| NotebookLM status | ✅ Used (5 Q&A queries) |
| Sources analyzed | 12 |
| Processing date | 2026-03-20 |
