---
type: reference
status: active
publish: true
date: 2026-01-07
tags:
  - type/reference
  - topic/kubernetes
  - topic/rudimentary
  - status/active
topics:
  - replicaset
  - deployment
  - service
  - rolling-update
  - rollback
  - revision-history-limit
aliases:
  - ReplicaSet Deployment Service
  - 롤링 업데이트 롤백
  - revisionHistoryLimit
---

# \[기초] ReplicaSet · Deployment · Service

K8s 애플리케이션 운영의 3총사. **Deployment(관리자) → ReplicaSet(반장) → Pod**, 그리고 접속을 담당하는 **Service(안내데스크)**.

## ReplicaSet — "지정 개수의 파드 복제본을 항상 유지"
- **라벨 셀렉터**로 관리 대상 파드 식별 → 조정 루프(observe→diff→act): 부족하면 템플릿으로 생성, 과하면 최근 것부터 제거.
- 핵심 기능: 자가치유(Self-healing), 수평 확장.
- **직접 작성할 일은 거의 없다** — 업데이트 기능이 없어서 상위 **Deployment**로 관리하는 게 표준.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
spec:
  replicas: 3
  selector: { matchLabels: { app: frontend } }   # 관리 대상
  template:                                       # 부족 시 이 모양으로 생성
    metadata: { labels: { app: frontend } }
    spec:
      containers: [ { name: nginx, image: nginx:1.14.2 } ]
```

## Deployment — 롤링 업데이트 / 롤백
파드를 직접 죽이지 않고 **구 ReplicaSet과 신 ReplicaSet 두 개를 동시 운용**, 신규를 Scale Up + 기존을 Scale Down 하며 무중단 교체("달리는 차 바퀴 갈아끼우기").

- 업데이트 완료 후 **구 ReplicaSet은 `DESIRED:0`으로 남겨둠**(삭제 X) → 롤백 보험.
- 속도/가용성 전략:
  ```yaml
  spec:
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 25%        # 정원 초과 허용 (속도↑)
        maxUnavailable: 25%  # 부족 허용 (0이면 새 파드 Ready 전까지 기존 안 죽임 = 최고 안전)
  ```
- 실습 흐름:
  ```bash
  kubectl create deployment nginx-demo --image=nginx:1.14.2 --replicas=3
  kubectl set image deployment/nginx-demo nginx=nginx:1.16.1   # 롤링 시작
  kubectl get rs        # 구 RS(줄어듦) + 신 RS(늘어남) 공존 ← The Dance
  kubectl rollout status deployment/nginx-demo
  kubectl rollout undo deployment/nginx-demo                   # 즉시 롤백 (구 RS 다시 Scale Up)
  ```

### ReplicaSet이 10개 넘게 쌓인다? — `revisionHistoryLimit`
- 영원히 안 쌓인다. 기본 **10개** 보관, 11번째부터 가장 오래된 것 자동 GC.
- `revisionHistoryLimit: 5` 로 줄이면 즉시 정리. `0` 이면 롤백 불가(비추천).
- 수동 삭제는 **`DESIRED/CURRENT/READY` 모두 0인 것만** (`kubectl delete rs <name>`) — 단 그 버전으로는 롤백 불가.

## Service — 바뀌는 Pod IP 앞의 "고정 대표번호 + 교환원"
파드는 일회용이라 재생성 시 IP가 바뀜. Service는 **고정 ClusterIP**를 부여받고, **라벨 셀렉터**로 살아있는 파드들에 트래픽 분산.

| Type | 설명 | 비유 |
|---|---|---|
| **ClusterIP**(기본) | 클러스터 내부 전용 | 사내 내선 |
| **NodePort** | 노드IP:30000~32767 로 외부 노출 | 회사 대표전화 |
| **LoadBalancer** | 클라우드 LB + 공인 IP 자동 할당 | 고객센터 1588 |

> [!important] Service `selector` 와 Pod `labels` 가 정확히 일치해야 트래픽이 흐른다.
```yaml
kind: Service
spec:
  type: LoadBalancer
  selector: { app: nginx }     # Deployment 템플릿의 labels 와 일치
  ports: [ { protocol: TCP, port: 80, targetPort: 80 } ]
```

## 역할 분담 한 줄 요약
- **Deployment**: "파드 3개 유지하고 v2로 업데이트해" (선언)
- **ReplicaSet**: "하나 죽었네? 즉시 보충" (유지)
- **Service**: "파드 IP 뭐든 나한테 오면 연결" (접속)

> 다음 퍼즐: 도메인/경로 기반 라우팅은 **Ingress** (이 클러스터는 Traefik + cert-manager + MetalLB 조합 → [[2026-03-24_K3s-클러스터-Terraform-Terragrunt-삽질]]).
