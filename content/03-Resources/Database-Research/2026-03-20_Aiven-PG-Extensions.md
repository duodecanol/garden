---
type: research
status: draft
date: 2026-03-20
tags:
  - type/research
  - topic/dev
  - topic/database
topics:
  - aiven
  - postgres
  - extension
  - uuid7
related:
  - "[[2026-03-14_Aiven-Deep-Dive]]"
  - "[[2026-03-14_Managed-Serverless-Postgres-Hosting]]"
sources:
  - https://aiven.io/docs/products/postgresql/reference/list-of-extensions
  - https://aiven.io/blog/exploring-postgresql-18-new-uuidv7-support
  - https://aiven.io/changelog/65b55699-6536-465c-a655-8b7c544053a5
publish: true
---

# Aiven PostgreSQL Extension 목록 및 uuid7 지원 여부

관련: [[2026-03-14_Aiven-Deep-Dive]] · [[2026-03-14_Managed-Serverless-Postgres-Hosting]]

---

## ⚡ uuid7 결론 (핵심)

> [!important] uuid7 사용 가능 여부
> - **`pg_uuidv7` 확장**: ❌ Aiven 공식 Extension 목록에 **없음**
> - **PostgreSQL 18 네이티브 `uuidv7()` 함수**: ✅ **사용 가능**
>
> Aiven은 PostgreSQL 18을 지원하며, PG18에서 `uuidv7()`는 별도 Extension 없이 내장 함수로 제공된다.
> PG17 이하에서는 uuid7을 사용할 수 없다.

```sql
-- PG18에서 Extension 없이 바로 사용
CREATE TABLE items (
  id UUID PRIMARY KEY DEFAULT uuidv7()
);
```

Aiven CLI로 PG18 서비스 생성:
```bash
avn service create -t pg --cloud do-nyc --plan pg:free-1-1gb 'my-service' -c 'pg_version=18'
```

---

## 지원 PostgreSQL 버전 (2026년 기준)

Aiven은 현재 다음 버전을 지원한다:

| 버전 | 상태 |
|------|------|
| **PG 18.2** | ✅ 지원 (최신) |
| **PG 17.8** | ✅ 지원 (신규 서비스 기본값) |
| **PG 16.12** | ✅ 지원 |
| **PG 15.16** | ✅ 지원 |
| **PG 14.21** | ✅ 지원 |
| PG 13 이하 | ⚠️ EOL 예정 또는 종료 |

---

## 전체 Extension 목록

### 감사 (Auditing)
- `pgaudit` — 세션/객체 감사 로깅 (금융·규정 준수)
- `tcn` — Triggered change notifications

### 연결 (Connectivity)
- `dblink` — 다른 PostgreSQL DB에 연결
- `postgres_fdw` — 원격 PostgreSQL 서버 Foreign Data Wrapper

### 데이터 타입 (Data Types)
- `citext` — 대소문자 무시 문자열
- `cube` — 다차원 큐브 데이터 타입
- `hll` — HyperLogLog 타입 (PG11+)
- `hstore` — key-value 쌍 저장
- `ip4r` — IPv4/IPv6 주소 범위 타입
- `isn` — 국제 상품 번호 표준 타입
- `ltree` — 계층 트리 구조 데이터 타입
- `seg` — 선분/부동소수점 구간 타입
- `timescaledb` — 시계열 데이터 확장 ✅
- `unit` — SI 단위 확장
- `uuid-ossp` — UUID 생성 (v1~v5)

> [!note] uuid-ossp vs uuidv7
> `uuid-ossp`는 UUIDv1~v5만 생성. UUIDv7은 PG18 네이티브 `uuidv7()` 또는 `pg_uuidv7` Extension 필요. Aiven에서는 **PG18 네이티브만 지원**.

### 지리 정보 (Geographical)
- `address_standardizer`, `address_standardizer_data_us`
- `earthdistance` — 지구 표면 대원거리 계산
- `h3`, `h3_postgis` — 육각형 지리공간 인덱싱
- `pgrouting` — 지리공간 라우팅
- `postgis`, `postgis_legacy`, `postgis_raster`, `postgis_sfcgal`, `postgis_tiger_geocoder`, `postgis_topology`

### AI / 머신러닝
- `pgvector` — 벡터 유사도 검색 (PG13+) ✅
- `pgvectorscale` — pgvector 보완 벡터 확장 (PG16+) ✅

### 절차적 언어 (Procedural Language)
- `plperl` — PL/Perl
- `plpgsql` — PL/pgSQL

### 검색 & 텍스트
- `bloom` — Bloom 인덱스
- `btree_gin`, `btree_gist` — 일반 타입 GIN/GiST 인덱싱
- `dict_int` — 정수 텍스트 검색 딕셔너리
- `fuzzystrmatch` — 문자열 유사도/거리
- `pg_similarity` — 유사도 쿼리 지원 (PG13+)
- `pg_trgm` — trigram 기반 텍스트 유사도
- `pgcrypto` — 암호화 함수
- `rum` — RUM 인덱스
- `unaccent` — 악센트 제거 텍스트 검색

### 유틸리티
- `aiven_extras` — non-superuser용 DB 기능 확장 (Aiven 자체 개발)
- `bool_plperl`, `jsonb_plperl` — plperl 타입 변환 (PG13+)
- `intagg` — 정수 집계 (obsolete)
- `intarray` — 1D 정수 배열 함수/연산자
- `lo` — Large Object 관리
- `pageinspect` — DB 페이지 저수준 검사
- `pg_buffercache` — 공유 버퍼 캐시 검사
- `pg_cron` — PostgreSQL 잡 스케줄러 ✅
- `pg_partman` — 시간/ID 기반 파티션 관리 ✅
- `pg_prewarm` — 릴레이션 데이터 프리워밍 (PG11+)
- `pg_prometheus` — Prometheus 메트릭 (PG12 이하만)
- `pg_repack` — 최소 락으로 테이블 재구성
- `pg_stat_statements` — SQL 실행 통계 ✅
- `pgrowlocks` — 행 수준 잠금 정보
- `pgstattuple` — 튜플 수준 통계
- `postgresql_anonymizer` — PII 마스킹 (PG15+)
- `sslinfo` — SSL 인증서 정보
- `tablefunc` — 테이블 조작 함수 (`crosstab` 포함)
- `timetravel` — 시간 여행 함수 (PG11 이하만)
- `tsm_system_rows`, `tsm_system_time` — TABLESAMPLE 메서드

---

## uuid7 상세: PG18 네이티브 함수

> [!info] PostgreSQL 18 내장 uuid 함수
> PG18부터 Extension 없이 다음 함수를 기본 제공:

```sql
-- UUIDv7 생성
SELECT uuidv7();

-- UUID 버전 확인
SELECT uuid_extract_version(uuidv7());  -- 7 반환

-- UUIDv4 (기존)
SELECT gen_random_uuid();

-- UUID 버전 확인
SELECT uuid_extract_version(gen_random_uuid());  -- 4 반환
```

### UUIDv7 vs UUIDv4 성능 비교 (Aiven 실측)

| 작업 | UUIDv4 | UUIDv7 | 차이 |
|------|--------|--------|------|
| 10,000건 INSERT | 89ms | 65ms | **27% 빠름** |
| 60,000건 ORDER BY | 52ms (Seq Scan) | 17ms (Index Scan) | **70% 빠름** |

**이유**: UUIDv7은 타임스탬프 기반 정렬 가능 → B-tree 인덱스 순차 삽입 → 인덱스 단편화 감소, 캐시 효율 증가

### UUIDv7 보안 주의사항

> [!warning] 외부 노출 금지
> UUIDv7은 상위 48비트가 Unix 타임스탬프 → **생성 시각이 UUID에 노출됨**
> - 외부 API나 사용자 공개 ID로 사용 시 타이밍 정보 유출 위험
> - 권장: 내부 PK는 UUIDv7, 외부 노출 ID는 별도 UUIDv4 사용

---

## 주요 미지원 Extension (타 서비스 대비)

| Extension | 용도 | 대안 |
|-----------|------|------|
| `pg_uuidv7` | UUIDv7 생성 (PG13~17용) | PG18 네이티브 `uuidv7()` 사용 |
| `plv8` | JavaScript 절차 언어 | 없음 |
| `citus` | 분산 PostgreSQL | 없음 (Azure만 지원) |
| `pg_partman` advanced triggers | 일부 기능 제한 | 기본 지원됨 |

---

## Sources

- [Aiven PostgreSQL Extension 공식 목록](https://aiven.io/docs/products/postgresql/reference/list-of-extensions)
- [Aiven 블로그: PostgreSQL 18 UUIDv7 지원 탐구](https://aiven.io/blog/exploring-postgresql-18-new-uuidv7-support)
- [Aiven Changelog: PG 18.2, 17.8, 16.12, 15.16, 14.21 업그레이드](https://aiven.io/changelog/65b55699-6536-465c-a655-8b7c544053a5)
- [GitHub: pg_uuidv7 Extension](https://github.com/fboulnois/pg_uuidv7)
