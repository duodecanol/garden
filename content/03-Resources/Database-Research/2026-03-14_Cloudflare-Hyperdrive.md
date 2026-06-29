---
type: research
status: draft
date: 2026-03-14
tags:
  - type/research
  - topic/dev
  - topic/cloudflare
  - topic/database
  - topic/serverless
topics:
  - Cloudflare
  - Database
  - Edge Computing
  - Connection Pooling
  - Serverless
related: []
publish: true
---

# Cloudflare Hyperdrive

> **한 줄 요약:** Cloudflare Workers 환경에서 기존 PostgreSQL/MySQL 데이터베이스를 글로벌 엣지에서 빠르게 사용할 수 있게 해주는 DB 가속 프록시 서비스

## 핵심 컨셉

### 문제: 왜 만들었나

Cloudflare Workers는 전 세계 분산 엣지에서 실행되지만, 기존 DB는 단일 리전에 위치한다. 이로 인해 두 가지 문제 발생:

**1. 연결 오버헤드**
새 DB 연결 수립 시 필요한 round-trip:
- TCP handshake
- TLS negotiation
- 인증(authentication)

→ 총 **7회 이상 round-trip**, 쿼리 실행 전 연결 수립에만 **1초 이상** 소요 가능

**2. 높은 연결 비용**
PostgreSQL은 연결당 메모리 소비가 높아, 서버리스 환경에서 수백 개의 동시 연결이 메모리를 급격히 소진함.

### 해결책

> [!tip] 핵심 아이디어
> "Regional 데이터베이스를 Globally Distributed처럼 느끼게 만드는 것"
> - **Connection Pooling**: Cloudflare 네트워크 내에 warm 연결 풀 유지
> - **Smart Query Caching**: DB 와이어 프로토콜 파싱으로 읽기 쿼리 자동 캐싱

## 기술 아키텍처

### 연결 흐름

```
[Workers (글로벌)] → [Hyperdrive Pool (CF 네트워크)] → [DB (단일 리전)]
```

**Connection Pooling (Transaction Mode):**
- 트랜잭션 동안 단일 연결 유지
- 완료 후 연결이 풀로 반환
- 세션 레벨 설정 자동 초기화(reset)
- 트래픽에 따라 연결 수 Auto Scaling

**Intelligent Query Caching:**

| 구분 | 예시 | 캐시 여부 |
|------|------|---------|
| 읽기 쿼리 | SELECT | ✅ |
| IMMUTABLE 함수 | 상수 반환 함수 | ✅ |
| 쓰기 쿼리 | INSERT, UPDATE, DELETE | ❌ |
| VOLATILE 함수 | NOW(), RANDOM() | ❌ |

	기본 캐시 설정: `max_age: 60초`, `stale_while_revalidate: 15초` (최대 1시간)

## 지원 데이터베이스

### PostgreSQL (v9.0 ~ 17.x)
- AWS RDS/Aurora, Google Cloud SQL, AlloyDB, Azure DB
- Neon, Supabase, CockroachDB, Timescale, Materialize

### MySQL (v5.7 ~ 8.x)
- AWS RDS/Aurora, Google Cloud SQL, Azure DB
- PlanetScale, MariaDB

> [!warning] 미지원
> SQL Server, MongoDB는 현재 지원하지 않음

## 성능 벤치마크

> [!success] 성능 향상
> | 시나리오 | 직접 연결 대비 향상 |
> |---------|-----------------|
> | 캐시된 쿼리 | **17~25배 빠름** |
> | 캐시 미적중/쓰기 | **6~8배 빠름** |
>
> Workers Placement 결합 시: 쿼리 지연 20~30ms → **1~3ms**

## 가격 정책

### Free Plan
- Hyperdrive 쿼리: **100,000건/일**
- Hyperdrive 설정: **최대 10개/계정**
- Workers Free Plan 포함

### Paid Plan ($5/월)
- Hyperdrive 쿼리: **무제한**
- Hyperdrive 설정: **최대 25개/계정**
- Workers Paid Plan 포함

> [!info] 중요
> - **Connection Pooling은 영구 무료** (Cloudflare 공식)
> - **데이터 전송(egress) 비용 없음**

## 주요 제한사항

| 항목 | Free | Paid |
|------|------|------|
| DB 설정 수 | 10개/계정 | 25개/계정 |
| 오리진 DB 연결 수 | ~20개 | ~100개 |
| 연결 타임아웃 | 15초 | 15초 |
| 유휴 타임아웃 | 10분 | 10분 |
| 최대 쿼리 실행 | 60초 | 60초 |
| 최대 캐시 크기 | 50MB | 50MB |

**PostgreSQL 미지원 기능:** Prepared statements SQL 관리, Advisory locks, LISTEN/NOTIFY, Per-session 상태 변경

**MySQL 미지원 기능:** Non-UTF8, `USE` 구문, Multi-statement 쿼리

## 개발자 경험 (DX)

### 설정 (3단계)

**Step 1: Hyperdrive 설정 생성**
```bash
npx wrangler hyperdrive create MY_DB \
  --connection-string="postgres://user:pass@host:5432/dbname"
```

**Step 2: wrangler.jsonc 바인딩**
```json
{
  "hyperdrive": [{ "binding": "HYPERDRIVE", "id": "<ID>" }],
  "compatibility_flags": ["nodejs_compat"]
}
```

**Step 3: Worker 코드 (기존 드라이버 그대로)**
```typescript
import { Client } from "pg";
const sql = new Client({ connectionString: env.HYPERDRIVE.connectionString });
```

기존 ORM 호환: Drizzle, Prisma, Sequelize 등

## 장점 / 단점 분석

### ✅ 장점
1. **Zero code change** - 연결 문자열만 교체, 기존 도구 그대로
2. **Connection pooling 영구 무료**
3. **egress 비용 없음**
4. **최대 25배 성능 향상** (캐시 쿼리 기준)
5. **다양한 DB 지원** - PostgreSQL, MySQL 및 호환 DB
6. **Workers Placement 결합** 가능
7. **TLS 강제** - 보안 기본값

### ❌ 단점
1. **Workers 전용** - Cloudflare Workers 외 사용 불가
2. **쿼리 제한 (Free)** - 100,000건/일로 프로덕션 부족 가능
3. **연결 수 제한** - Free ~20개, Paid ~100개
4. **캐시 수동 invalidation 없음** - 자동 만료만 가능 (최대 1시간)
5. **SQL Server/MongoDB 미지원**
6. **Session-level 기능 제한**
7. **설정 수 제한** - Paid 기준 25개

## 경쟁 제품 비교

| 기능 | Hyperdrive | PgBouncer | Neon 내장 | Supabase Pooler |
|------|-----------|-----------|-----------|----------------|
| 대상 환경 | Serverless (Workers) | 범용 | Serverless | 범용 |
| 쿼리 캐싱 | ✅ | ❌ | ❌ | ❌ |
| 글로벌 분산 | ✅ CF 네트워크 | ❌ | 제한적 | ❌ |
| 설정 복잡도 | 낮음 | 중간 | 낮음 | 낮음 |
| 가격 | Free tier 있음 | 오픈소스 | DB 비용 포함 | DB 비용 포함 |
| Workers 전용 | 예 | 아니오 | 아니오 | 아니오 |

> [!tip] 핵심 차별점
> Hyperdrive는 단순 Connection Pooler가 아닌, **글로벌 분산 캐싱 레이어를 겸하는 엣지 DB 가속기**

## 주요 사용 사례

1. **글로벌 SaaS** - 단일 리전 DB로 전 세계 낮은 지연 제공
2. **서버리스 API** - Workers + 기존 PostgreSQL/MySQL 조합
3. **기존 DB 마이그레이션 없이 Workers 전환** - 최소한의 코드 변경
4. **읽기 중심 워크로드** - 캐싱으로 DB 부하 대폭 감소

## 보안

- TLS 강제 (평문 연결 불허)
- TLS 모드: `require` (기본) / `verify-ca` / `verify-full`
- PostgreSQL 인증: md5, SCRAM-SHA-256
- Private DB: Cloudflare Tunnel 통해 연결 가능

## 관련 노트

- [[2026-03-14_Managed-Serverless-Postgres-Hosting]]
- [[2026-03-14_Aiven-Deep-Dive]]

## 출처

- [Hyperdrive 공식 문서](https://developers.cloudflare.com/hyperdrive/)
- [지원 DB 목록](https://developers.cloudflare.com/hyperdrive/reference/supported-databases-and-features/)
- [Connection Pooling 설정](https://developers.cloudflare.com/hyperdrive/configuration/connection-pooling/)
- [Query Caching 설정](https://developers.cloudflare.com/hyperdrive/configuration/query-caching/)
- [플랫폼 제한](https://developers.cloudflare.com/hyperdrive/platform/limits/)
- [Workers 가격 정책](https://developers.cloudflare.com/workers/platform/pricing/)
- [블로그: Making regional databases feel distributed](https://blog.cloudflare.com/hyperdrive-making-regional-databases-feel-distributed/)
- [시작 가이드](https://developers.cloudflare.com/hyperdrive/get-started/)
