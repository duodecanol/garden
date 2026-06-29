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
  - neon
  - postgres
  - branching
  - devops
related:
  - "[[2026-03-14_Aiven-Deep-Dive]]"
  - "[[2026-03-14_Managed-Serverless-Postgres-Hosting]]"
  - "[[2026-03-20_Aiven-PG-Extensions]]"
sources:
  - https://aiven.io/docs/platform/concepts/service-forking
  - https://neon.com/docs/introduction/branching
publish: true
---

# Aiven Forking vs Neon Branching 비교

관련: [[2026-03-14_Aiven-Deep-Dive]] · [[2026-03-14_Managed-Serverless-Postgres-Hosting]]

---

## 결론

> [!important] 핵심 답변
> Aiven에도 **Forking** 기능이 있지만, Neon의 **Branching**과는 **기술적 구조와 실용성에서 근본적으로 다르다.**
>
> - **Neon Branching**: Copy-on-Write 스냅샷 → **초 단위 생성**, 저장 공간 공유, 수십 개 병렬 운영 가능
> - **Aiven Forking**: 백업 복원 → **분~수십 분 소요**, 완전히 독립된 새 서비스(VM) 생성, 비용 별도 발생

---

## Aiven Forking 상세

### 작동 방식

```
백업 스냅샷(PGHoard) → 새 VM 프로비저닝 → 데이터 복원 → 새 서비스 가동
```

- 백업(PGHoard)에서 새 PostgreSQL 인스턴스를 생성하는 방식
- PITR 지원: 최신 트랜잭션 또는 **특정 과거 시점**에서 Fork 가능
- Fork된 서비스는 원본과 **완전히 독립** (리소스 공유 없음)
- Fork 시 복사되는 것: 서비스 설정, DB, 서비스 사용자, 커넥션 풀

### 사용 가능 조건

> [!warning] Fork 조건
> - 서비스에 **백업이 최소 1개 이상** 있어야 함
> - Free/Developer 플랜은 PITR 미지원 → 단순 재해복구 백업만 사용 가능
> - **Startup 이상**만 PITR Fork 가능 (2일~30일 보존 기간)

### Aiven Forking 사용 사례

| 사용 사례 | 가능 여부 |
|-----------|-----------|
| 프로덕션 DB 개발/테스트 복사본 생성 | ✅ |
| 특정 시점(PITR)으로 복원 | ✅ (Startup 이상) |
| 메이저 버전 업그레이드 테스트 | ✅ |
| 다른 클라우드/리전으로 이전 | ✅ |
| CI/CD 파이프라인 자동화 | ⚠️ 가능하나 느리고 비용 발생 |
| Preview 환경 수십 개 병렬 생성 | ❌ 비용 및 속도 문제 |

### 제한사항

- Service Integration은 Fork에 복사되지 않음
- SSO 설정 복사 안 됨 (OpenSearch)
- Fork 초기에는 단일 노드로 시작, 백업 완료 후 나머지 노드 추가
- Fork된 서비스 = 별도 유료 서비스 (플랜 요금 별도 과금)

---

## Neon Branching 상세

### 작동 방식

```
Copy-on-Write(CoW) 스냅샷 → 즉시 완료 → 변경분만 저장
```

- Git 브랜치와 동일한 개념: 데이터를 복사하지 않고 **변경분(delta)만 저장**
- 브랜치 생성이 **수초 내 완료** (DB 크기와 무관)
- 부모 브랜치에 전혀 영향 없음
- 브랜치당 독립적인 compute endpoint 자동 할당

### Neon Branching 사용 사례

| 사용 사례 | 특징 |
|-----------|------|
| 개발자별 독립 DB 환경 | 수초 내 생성, 비용 최소 |
| PR 단위 Preview 환경 | Vercel 통합 → 자동 생성/삭제 |
| 병렬 E2E 테스트 | 각 브랜치에 독립 compute |
| PITR 데이터 복구 | 6시간(무료) ~ 30일(유료) |
| TTL 기반 임시 환경 | 자동 삭제 설정 가능 |

---

## 직접 비교

| 항목 | **Aiven Forking** | **Neon Branching** |
|------|-------------------|--------------------|
| **생성 속도** | 분~수십 분 (백업 복원) | 수 초 (CoW 스냅샷) |
| **기술 방식** | 백업 기반 전체 복원 | Copy-on-Write |
| **저장 공간** | 전체 복사 → 별도 스토리지 | Delta만 저장 → 효율적 |
| **비용** | Fork = 새 서비스 → 플랜 요금 발생 | 변경분 스토리지만 과금 |
| **PITR** | ✅ (Startup 이상) | ✅ (모든 유료 플랜) |
| **병렬 운영** | ❌ (비용/속도 제약) | ✅ (수십 개 가능) |
| **CI/CD 통합** | ⚠️ 가능하나 복잡 | ✅ Vercel/GitHub Actions 네이티브 |
| **원본 영향** | 없음 | 없음 |
| **독립성** | 완전 독립 (별도 VM) | 스토리지 공유, compute 독립 |
| **목적** | 스냅샷 분석, 버전 업그레이드 테스트, 리전 이전 | 개발/테스트/Preview 환경 대량 생성 |

---

## 평가

> [!tip] 언제 어떤 기능을 쓸까?

**Aiven Forking이 적합한 경우:**
- 메이저 버전 업그레이드 전 테스트 (1회성, 속도 무관)
- 재해 복구 드릴
- 다른 클라우드/리전으로 DB 이전
- 특정 시점 스냅샷을 장기 보관해야 할 때

**Neon Branching이 필요한 경우:**
- PR마다 격리된 DB Preview 환경 자동화
- 개발자 10명이 각자 독립 DB 필요
- CI/CD 파이프라인에서 빠른 DB 생성/삭제 반복
- 테스트 후 즉시 폐기하는 임시 환경 대량 운영

> [!warning] 결론
> Aiven의 Forking은 Neon Branching의 **대체재가 아니다.** 용도 자체가 다르다.
> Neon Branching이 필요한 CI/CD·Preview 환경 워크플로우라면 Aiven은 적합하지 않다.
> Aiven Forking은 "운영 수준의 스냅샷 관리" 도구에 가깝다.

---

## Sources

- [Aiven: Fork a service (공식 문서)](https://aiven.io/docs/platform/concepts/service-forking)
- [Neon: Branching (공식 문서)](https://neon.com/docs/introduction/branching)
- [Aiven: PostgreSQL Backup 문서](https://aiven.io/docs/products/postgresql/concepts/pg-backups)
