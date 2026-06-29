---
type: research
status: draft
date: 2026-04-07
tags:
  - type/research
  - topic/dev
  - topic/database
  - topic/aws
  - topic/gcp
  - topic/postgresql
topics:
  - aurora
  - rds
  - cloud-sql
  - alloydb
  - neon
  - postgresql
  - serverless
related:
  - "[[2026-03-20_AWS-Aurora-PostgreSQL]]"
  - "[[2026-03-14_Managed-Serverless-Postgres-Hosting]]"
sources:
  - https://aws.amazon.com/rds/aurora/pricing/
  - https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Replication.html
  - https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html
  - https://aws.amazon.com/rds/postgresql/pricing/
  - https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html
  - https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PostgreSQL.Replication.ReadReplicas.html
  - https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIOPS.Autoscaling.html
  - https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIT.html
  - https://cloud.google.com/sql/pricing
  - https://cloud.google.com/sql/docs/postgres/replication
  - https://cloud.google.com/sql/docs/postgres/high-availability
  - https://cloud.google.com/sql/docs/postgres/managed-connection-pooling
  - https://cloud.google.com/alloydb/pricing
  - https://cloud.google.com/alloydb/docs/backup/overview
  - https://cloud.google.com/alloydb/docs/backup/restore-pitr
  - https://cloud.google.com/alloydb/docs/instance-read-pool-scale
  - https://neon.com/pricing
  - https://neon.com/docs/introduction/branching
  - https://neon.com/docs/introduction/read-replicas
notebooklm_notebook: Managed Postgres on AWS and GCP vs Neon
publish: true
---

# AWS Aurora, RDS, GCP Database Products vs Neon

관련: [[2026-03-20_AWS-Aurora-PostgreSQL]] · [[2026-03-14_Managed-Serverless-Postgres-Hosting]]

---

## 범위

이 문서는 **PostgreSQL 워크로드 관점**에서 아래 제품만 비교한다.

- AWS: `Aurora PostgreSQL-Compatible`, `RDS for PostgreSQL`
- GCP: `Cloud SQL for PostgreSQL`, `AlloyDB for PostgreSQL`
- 비교군: `Neon`

> [!note]
> GCP의 전체 DB 포트폴리오(Spanner, Bigtable, Firestore 등)를 모두 비교하지는 않는다.
> 이 문서의 관심사는 "PostgreSQL 계열 운영형 DB"다.

---

## 결론

> [!important] 핵심 요약
> Neon을 기준점으로 보면, 각 제품의 포지션은 꽤 분명하다.
>
> - **Neon**: 개발/프리뷰/브랜칭/저비용 서버리스에 가장 유리
> - **Aurora PostgreSQL**: AWS 안에서 HA, 빠른 failover, 낮은 replica lag, Serverless가 중요할 때 가장 강함
> - **RDS PostgreSQL**: 가장 "표준 PostgreSQL"에 가까워 예측 가능성이 높고, 운영 감각도 익숙함
> - **Cloud SQL PostgreSQL**: GCP 기본 선택지. 비용 구조와 운영 모델이 단순하고 GCP 통합이 편함
> - **AlloyDB**: GCP의 프리미엄 PostgreSQL. 성능, HA, 읽기 확장, 운영 자동화는 강하지만 가격 바닥이 높음

> [!tip]
> 실무 선택을 한 줄로 줄이면:
> - **개발 생산성 / ephemeral DB / scale-to-zero**: Neon
> - **AWS 프로덕션 표준**: Aurora 또는 RDS
> - **GCP 프로덕션 표준**: Cloud SQL
> - **GCP 고성능 프리미엄**: AlloyDB

---

## Neon을 비교군으로 삼는 이유

Neon은 단순히 또 하나의 PostgreSQL 호스팅이 아니라, 다음 질문을 던지게 만든다.

- 정말 **항상 켜져 있는 프로비저닝 DB**가 필요한가?
- 개발/테스트 환경에 왜 **전체 인스턴스 비용**을 계속 내야 하는가?
- 복구를 왜 "새 인스턴스 복원"으로만 생각하는가? 왜 **branch from past**가 안 되는가?
- 읽기 확장을 왜 "별도 데이터 복제본"으로만 생각해야 하는가?

즉, Neon은 비교 대상들을 "전통적인 managed PostgreSQL"과 "서버리스/분기형 PostgreSQL"로 갈라서 보게 만든다.

---

## 비교표

| 항목 | Neon | Aurora PostgreSQL | RDS PostgreSQL | Cloud SQL PostgreSQL | AlloyDB |
| --- | --- | --- | --- | --- | --- |
| 기본 포지션 | 서버리스 Postgres | AWS 고가용성 PG 호환 엔진 | 표준 관리형 PostgreSQL | GCP 기본 관리형 PostgreSQL | GCP 프리미엄 PG |
| 비용 바닥 | 가장 낮음 | 중간~높음 | 중간~높음 | 중간 | 높음 |
| 가격 구조 | 사용량 중심, free tier 존재 | 컴퓨팅 + 스토리지 + I/O | 인스턴스 + 스토리지 + IOPS | 인스턴스 + 스토리지 | CPU/메모리 + 스토리지, I/O 별도 과금 없음 |
| compute autoscaling | 강함 | 강함 (`Serverless v2`) | 사실상 없음 | 사실상 없음 | 제한적, 수동 중심 |
| scale-to-zero | 지원 | 제한적 지원 (`auto-pause`) | 불가 | 불가 | 불가 |
| read scaling | read-only endpoints | Aurora Replicas 최대 15개 | read replica 수동 생성 | read replica / cross-region / cascading | read pool |
| replica lag | 스토리지 분리 아키텍처상 유리 | 보통 100ms 미만 | native PG async 복제 | async 복제 기반 | read pool 기반 저지연 지향 |
| 운영 편의성 | 개발자 경험 최상 | AWS 통합 최상 | PG 친화성 최상 | GCP 운영 단순성 우수 | 자동화/성능 기능 강함 |
| PITR/복구 | restore branch from past | PITR + 백업 + Global DB | PITR + 백업 | PITR + 백업 | 기본 14일 PITR |
| 멀티리전 DR | 제한적 | 강함 | 보통 | 보통 | 강함 |

---

## 1. 가격

### 가격 철학

- **Neon**: 가장 낮은 비용 바닥. 개발용/저활성 워크로드에 유리
- **Aurora**: 유연하지만 과금 항목이 가장 복잡함
- **RDS**: 전통적인 인스턴스 과금 구조. 예측은 쉽지만 유휴 비용이 큼
- **Cloud SQL**: 구조가 단순하고 예측 가능
- **AlloyDB**: 프리미엄 과금이지만 I/O 과금이 없어 대규모 워크로드 예측성은 좋음

### 공식 가격표에서 바로 확인되는 포인트

| 제품 | 공식 가격 포인트 |
| --- | --- |
| Aurora Serverless v2 | `us-east-1` 기준 Aurora Standard 약 `$0.12 / ACU-hour`, I/O-Optimized 약 `$0.156 / ACU-hour` |
| Aurora 스토리지 | Standard 약 `$0.10 / GB-month`, I/O-Optimized는 더 비쌈 |
| Aurora I/O | Standard는 read/write I/O 과금, I/O-Optimized는 I/O 별도 과금 없음 |
| RDS PostgreSQL | 인스턴스 클래스별 시간당 과금 + 스토리지 + 필요 시 IOPS. scale-to-zero 없음 |
| Cloud SQL | SSD 스토리지 약 `$0.000232877 / GiB-hour`, 백업 약 `$0.000109589 / GiB-hour` |
| Cloud SQL HA | Google 문서상 **standalone 대비 약 2배 비용** |
| AlloyDB | CPU/메모리 + 스토리지 + 백업. Google이 "opaque I/O charges 없음"을 명시 |
| Neon | free tier 존재, 사용량 기반. 개발/저활성 워크로드의 비용 바닥이 가장 낮음 |

> [!warning]
> RDS PostgreSQL은 정확한 월 비용을 단일 숫자로 일반화하기 어렵다.
> 인스턴스 패밀리와 Single-AZ / Multi-AZ 구성이 월 비용을 좌우하기 때문이다.
> 대신 중요한 판단 기준은 이것이다:
> **RDS는 인스턴스를 켜 둔 시간 전체에 대해 계속 비용이 발생한다.**

### Neon 기준 가격 해석

- **Neon**은 "안 쓰면 거의 안 내는" 모델에 가장 가깝다.
- **Aurora Serverless v2**는 Neon에 가장 근접하지만, 여전히 AWS 인프라 가격대와 스토리지/I/O 과금 구조를 따른다.
- **RDS / Cloud SQL / AlloyDB**는 기본적으로 "프로비저닝된 인스턴스를 계속 보유"하는 모델이다.

### 비용 판단

> [!tip]
> 비용만 보면 보통 아래 순서다.
>
> - **최저 비용 바닥**: Neon
> - **전통형 중 단순 예측**: Cloud SQL, RDS
> - **탄력성 포함 시 중상위**: Aurora
> - **프리미엄 성능/HA 비용**: AlloyDB

---

## 2. 운영 편의성

### Neon

- 개발자 경험이 가장 좋다
- 브랜치 생성, 임시 환경, 과거 시점 분기 복구가 매우 강력하다
- "DB를 환경 단위로 복제하는 비용"을 거의 없애 준다

### Aurora PostgreSQL

- AWS 생태계 안에서는 가장 편하다
- IAM, CloudWatch, 보안, DR, replica/failover 구성이 자연스럽다
- 대신 PostgreSQL 자체를 운영한다기보다 **Aurora라는 AWS 엔진을 운영**하는 감각이 필요하다

### RDS PostgreSQL

- PostgreSQL 운영 경험이 있는 팀에게 가장 익숙하다
- Aurora보다 단순하고, PostgreSQL 표준 동작에 대한 기대치가 잘 맞는다
- 고급 HA/DR/elasticity는 상대적으로 약하다

### Cloud SQL PostgreSQL

- GCP 환경에서 "기본 선택"으로 가장 무난하다
- Cloud SQL Auth Proxy, IAM, Managed Connection Pooling 등 운영 편의 기능이 잘 붙는다
- Cloud SQL Enterprise Plus에서 Managed Connection Pooling을 쓸 수 있다

### AlloyDB

- GCP가 제공하는 고성능/고가용성 PostgreSQL 운영 경험
- 자동화와 성능 기능은 강하지만 제품 자체가 더 프리미엄이고 구조도 더 무겁다
- Cloud SQL보다 "쉽다"기보다는, **더 강력하지만 더 비싼 상위 옵션**에 가깝다

---

## 3. 오토스케일링

### Neon

- 서버리스 compute autoscaling이 핵심 강점
- autosuspend / low idle cost가 가능해 scale-to-zero 시나리오에 가장 가깝다

### Aurora PostgreSQL

- `Aurora Serverless v2`는 Aurora 제품군 중 가장 강한 탄력성을 제공한다
- AWS 가격 문서는 즉각적 scaling과 auto-pause 지원을 강조한다
- Neon처럼 가볍게 껐다 켜는 개발용 DB보다는, **프로덕션 탄력성**에 더 초점이 있다

### RDS PostgreSQL

- compute autoscaling은 사실상 없다
- 인스턴스 클래스 변경은 다운타임/페일오버 가능성이 있다
- 가능한 오토스케일링은 주로 **storage autoscaling**뿐이다

### Cloud SQL PostgreSQL

- storage 측면 자동 확장은 가능하지만, Neon/Aurora 같은 compute serverless autoscaling은 아니다
- 즉, "용량 부족 방지"에는 좋지만 "트래픽 피크 자동 흡수" 제품은 아니다

### AlloyDB

- 진짜 serverless 제품은 아니다
- primary/read pool의 스케일링은 되지만 대부분 계획적 수동 조정이다
- Google 문서상 read pool의 node count는 인스턴스 수준 무중단으로 늘릴 수 있다

> [!warning]
> "오토스케일링"이라는 단어만 보면 Aurora/AlloyDB/Cloud SQL가 비슷해 보일 수 있다.
> 하지만 Neon과 Aurora Serverless는 **compute가 탄력적으로 변하는 제품**이고,
> RDS/Cloud SQL/AlloyDB는 대체로 **프로비저닝 리사이즈 중심**이다.

---

## 4. Read Replica / Read Scaling

### Neon

- read replica는 **별도 compute**이며, 데이터를 복제해 따로 저장하는 전통적 replica와 감각이 다르다
- 같은 데이터 기반 위에 read-only endpoint를 늘리는 모델이라 개발/읽기 분산에 유리하다

### Aurora PostgreSQL

- 클러스터당 **최대 15개 Aurora Replica**
- AWS 문서상 replica lag는 **보통 100ms 미만**
- writer 장애 시 reader를 새 writer로 승격 가능
- 이 항목은 Aurora가 가장 강력한 축 중 하나다

### RDS PostgreSQL

- PostgreSQL native async replication 기반
- read replica는 **수동 생성**해야 하고 autoscaling은 없다
- replica lag는 WAL 기반 특성상 write 부하에 민감하다
- AWS 문서상 idle 상태에서도 replica lag가 5분까지 보일 수 있다

### Cloud SQL PostgreSQL

- read replica, cross-region read replica, cascading read replica 지원
- Google은 direct replica 수를 **10개 이하 권장**
- DR 시 replica를 standalone primary로 promote 가능
- load balancing은 Cloud SQL이 자동 제공하지 않는다

### AlloyDB

- read pool 기반으로 읽기 확장
- Cloud SQL보다 읽기 확장 모델이 더 제품 중심적으로 설계되어 있다
- read pool node count는 수평 확장이 가능하고, 증가 시 연결 영향 없이 확장 가능

---

## 5. 복구 가능성, 백업, DR

### Neon

- 브랜치를 과거 시점에서 생성할 수 있다
- restore를 "새로운 복구 브랜치 생성"으로 접근하기 쉬워 개발/실수 복구 UX가 좋다
- 기본 restore window는 Free 6시간, 유료 기본 1일이며 상위 플랜에서 더 확장 가능

### Aurora PostgreSQL

- 자동 백업과 PITR 지원
- 읽기 복제와 failover가 강하고, `Aurora Global Database`로 멀티리전 DR 구성이 가능하다
- AWS 프로덕션 DR 시나리오에 가장 잘 맞는다

### RDS PostgreSQL

- PITR 지원
- 단, AWS 문서상 **PITR은 primary(writer)에만 적용**되며 read replica에는 직접 적용되지 않는다
- DR은 read replica promote 패턴을 쓸 수 있지만 Aurora만큼 자연스럽지는 않다

### Cloud SQL PostgreSQL

- 백업과 PITR 지원
- cross-region replica 기반 DR 가능
- 고가용성 구성은 regional persistent disk 기반이며, Google 문서상 HA 인스턴스는 standalone의 약 2배 비용이다

### AlloyDB

- continuous backup and recovery가 기본 제공
- 기본 PITR window는 **14일**, 최대 **35일**
- continuous backup 기반 복구 모델이 강점이다
- regional primary는 active/standby 구조로 HA 제공

> [!tip]
> 복구 UX만 보면 Neon이 가장 "개발자 친화적"이고,
> 엔터프라이즈 DR 역량은 Aurora와 AlloyDB가 더 강하다.

---

## 6. Neon 기준 제품별 해석

### Aurora PostgreSQL vs Neon

- **Aurora 우위**
  - AWS 통합
  - 강한 HA/DR
  - 최대 15 replica와 낮은 lag
  - 프로덕션 운영 신뢰도
- **Neon 우위**
  - 비용 바닥
  - 브랜칭
  - scale-to-zero 성격
  - 개발 환경 자동화

### RDS PostgreSQL vs Neon

- **RDS 우위**
  - 표준 PostgreSQL 친화성
  - 운영 예측 가능성
  - 기존 PostgreSQL 마이그레이션 용이성
- **Neon 우위**
  - 서버리스
  - 브랜칭
  - 자동 suspend
  - 개발/테스트 총비용

### Cloud SQL PostgreSQL vs Neon

- **Cloud SQL 우위**
  - GCP 통합
  - 운영 단순성
  - 엔터프라이즈 IAM/네트워크 정책
- **Neon 우위**
  - 더 낮은 비용 바닥
  - ephemeral DB
  - 브랜치 기반 워크플로

### AlloyDB vs Neon

- **AlloyDB 우위**
  - 프리미엄 성능
  - HA/DR
  - read pool
  - Google 관리 자동화
- **Neon 우위**
  - 개발 생산성
  - 비용 민첩성
  - 작은 팀/초기 제품 적합성

---

## 7. 선택 가이드

> [!tip]
> 아래처럼 고르면 된다.

- **초기 스타트업 / 프리뷰 환경 / CI DB**
  - `Neon`
- **AWS 중심 프로덕션 SaaS**
  - `Aurora PostgreSQL`
- **AWS에서 표준 PostgreSQL 우선**
  - `RDS PostgreSQL`
- **GCP의 기본 관리형 PostgreSQL**
  - `Cloud SQL PostgreSQL`
- **GCP에서 성능/HA를 강하게 요구**
  - `AlloyDB`

> [!warning]
> Neon과 비교할 때 가장 자주 생기는 오판은 이것이다.
> "Aurora Serverless도 서버리스니까 Neon과 거의 같다"는 생각.
>
> 실제로는:
> - Neon은 **개발 생산성과 비용 바닥**
> - Aurora Serverless는 **엔터프라이즈형 AWS 프로덕션 탄력성**
>
> 에 최적화되어 있다.

---

## 8. 권장안 3개 시나리오

### 시나리오 1. 초기 스타트업 / 작은 팀 / 빠른 제품 반복

> [!important]
> **권장안: Neon**

#### 이런 조건일 때

- 개발자 수가 적고 배포 속도가 중요하다
- 프리뷰 환경, 브랜치별 테스트 DB, CI DB가 자주 필요하다
- 유휴 시간 비용을 줄이고 싶다
- 아직 멀티 AZ DR이나 복잡한 규정 준수보다 제품 실험 속도가 더 중요하다

#### 이유

- 브랜칭과 임시 환경 생성이 가장 강하다
- 비용 바닥이 가장 낮다
- "프로덕션 DB 1개 + 복제 환경 여러 개"가 아니라, "필요한 만큼 branch를 파생"하는 방식이 잘 맞는다
- 복구도 과거 시점 branch 생성 방식이라 개발/운영 실수 복구가 빠르다

#### 주의

- AWS/GCP 네이티브 통합은 약하다
- 복잡한 엔터프라이즈 네트워크/보안 통제에는 Aurora, Cloud SQL, AlloyDB보다 불리할 수 있다
- 아주 보수적인 DR/HA 요구가 생기면 장기적으로 재검토가 필요하다

### 시나리오 2. AWS 중심 B2B SaaS / 운영 신뢰성 우선

> [!important]
> **권장안: Aurora PostgreSQL**

#### 이런 조건일 때

- 애플리케이션이 AWS 위에 이미 올라가 있다
- 읽기 트래픽 확장과 failover 시간이 중요하다
- IAM, CloudWatch, 백업, DR, 보안 정책을 AWS 안에서 통합하고 싶다
- 장기적으로 멀티리전 DR이나 더 높은 운영 자동화가 필요할 가능성이 크다

#### 이유

- Aurora Replica, 빠른 failover, 낮은 replica lag가 강점이다
- Serverless v2를 쓰면 전통적 RDS보다 트래픽 변화 대응이 쉽다
- Aurora Global Database 등 AWS 고가용성 기능과의 결합이 자연스럽다
- "표준 PostgreSQL을 운영"하는 것보다 "AWS용 프로덕션 데이터 플랫폼"에 가깝다

#### 대안 판단

- PostgreSQL 표준 호환성과 단순 운영이 더 중요하면 `RDS PostgreSQL`
- 개발/프리뷰 환경만 따로 떼어내고 싶으면 `Neon + Aurora` 혼합도 가능

#### 주의

- 비용 구조가 RDS보다 복잡하다
- Neon처럼 가볍고 싼 개발 환경 대량 생성에는 적합하지 않다
- PostgreSQL과 완전히 같은 엔진 감각으로 운영하면 오해가 생길 수 있다

### 시나리오 3. GCP 중심 엔터프라이즈 / 안정성과 확장성 분리

> [!important]
> **권장안: Cloud SQL 또는 AlloyDB**

#### 이렇게 나눈다

- **기본 권장안: Cloud SQL PostgreSQL**
- **고성능/프리미엄 권장안: AlloyDB**

#### Cloud SQL이 맞는 경우

- GCP 기본 통합이 중요하다
- 운영 구조를 단순하게 유지하고 싶다
- 비용 예측 가능성이 중요하다
- PostgreSQL 운영을 무난하게 관리형으로 가져가고 싶다

#### AlloyDB가 맞는 경우

- 더 높은 성능과 더 강한 HA/복구 체계가 필요하다
- 읽기 확장과 프리미엄 운영형 PostgreSQL이 필요하다
- Cloud SQL보다 높은 비용을 감수할 수 있다

#### 이유

- Cloud SQL은 GCP의 기본 관리형 PostgreSQL로 포지션이 명확하다
- AlloyDB는 GCP가 제공하는 상위급 PostgreSQL 제품으로, 성능과 운영 자동화 측면에서 더 공격적이다
- Neon은 여전히 개발 생산성 면에서는 강하지만, GCP 엔터프라이즈 운영 기준점으로는 Cloud SQL/AlloyDB가 더 자연스럽다

#### 주의

- Cloud SQL은 Neon처럼 브랜치 기반 개발 워크플로에 강하지 않다
- AlloyDB는 Cloud SQL보다 비용 바닥이 높고, "무난한 기본값"보다는 "프리미엄 선택지"에 가깝다

> [!tip]
> 정리하면:
>
> - **개발 속도와 비용 민첩성**: Neon
> - **AWS 프로덕션 표준**: Aurora PostgreSQL
> - **GCP 프로덕션 표준**: Cloud SQL PostgreSQL
> - **GCP 프리미엄 성능/HA**: AlloyDB

---

## 최종 판단

Neon을 기준점으로 보면, AWS와 GCP의 관리형 PostgreSQL 제품은 결국 두 그룹으로 나뉜다.

1. **전통적 관리형 PostgreSQL**
   - `RDS PostgreSQL`
   - `Cloud SQL PostgreSQL`

2. **클라우드 전용 고가용성/고성능 PostgreSQL 계열**
   - `Aurora PostgreSQL`
   - `AlloyDB`

Neon은 이 둘과 다른 축에 있다.
**DB를 오래 켜 두는 인프라**가 아니라, **필요한 순간에 켜고 분기하고 복구하는 개발 플랫폼**에 가깝다.

그래서 질문은 "어느 DB가 제일 좋은가"가 아니라 보통 이렇게 바뀐다.

- 우리는 **항상 켜진 프로덕션 DB**가 필요한가?
- 아니면 **분기 가능한 서버리스 Postgres 플랫폼**이 필요한가?

프로덕션 신뢰성, 클라우드 통합, DR이 우선이면 `Aurora` 혹은 `AlloyDB`.
표준 PostgreSQL 친화성과 단순 운영이 우선이면 `RDS` 혹은 `Cloud SQL`.
개발 속도와 비용 민첩성이 우선이면 `Neon`이 가장 강하다.

---

## Sources

| # | 제목 | 출처 |
| --- | --- | --- |
| 1 | Amazon Aurora Pricing | [AWS 공식](https://aws.amazon.com/rds/aurora/pricing/) |
| 2 | Replication with Amazon Aurora | [AWS Docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Replication.html) |
| 3 | RDS for PostgreSQL Pricing | [AWS 공식](https://aws.amazon.com/rds/postgresql/pricing/) |
| 4 | Working with DB instance read replicas | [AWS Docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html) |
| 5 | Working with read replicas for Amazon RDS for PostgreSQL | [AWS Docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PostgreSQL.Replication.ReadReplicas.html) |
| 6 | Managing capacity automatically with Amazon RDS storage autoscaling | [AWS Docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIOPS.Autoscaling.html) |
| 7 | Restoring a DB instance to a specified time for Amazon RDS | [AWS Docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIT.html) |
| 8 | Cloud SQL pricing | [Google Cloud 공식](https://cloud.google.com/sql/pricing) |
| 9 | About replication in Cloud SQL | [Google Cloud Docs](https://cloud.google.com/sql/docs/postgres/replication) |
| 10 | About high availability | [Google Cloud Docs](https://cloud.google.com/sql/docs/postgres/high-availability) |
| 11 | Managed Connection Pooling overview | [Google Cloud Docs](https://cloud.google.com/sql/docs/postgres/managed-connection-pooling) |
| 12 | AlloyDB pricing | [Google Cloud 공식](https://cloud.google.com/alloydb/pricing) |
| 13 | Data backup and recovery overview | [Google Cloud Docs](https://cloud.google.com/alloydb/docs/backup/overview) |
| 14 | Use point-in-time recovery (PITR) | [Google Cloud Docs](https://cloud.google.com/alloydb/docs/backup/restore-pitr) |
| 15 | Scale an instance | [Google Cloud Docs](https://cloud.google.com/alloydb/docs/instance-read-pool-scale) |
| 16 | Neon Pricing | [Neon 공식](https://neon.com/pricing) |
| 17 | Neon Branching | [Neon Docs](https://neon.com/docs/introduction/branching) |
| 18 | Neon Read Replicas | [Neon Docs](https://neon.com/docs/introduction/read-replicas) |
