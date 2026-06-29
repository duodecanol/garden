---
type: research
status: active
publish: true
date: 2026-03-26
tags:
  - type/research
  - topic/kubernetes
  - topic/argocd
  - topic/aws
  - topic/cloudflare
  - topic/dns
  - status/active
topics:
  - argocd
  - elb-recreation
  - externaldns
  - cloudflare
  - dns-outage
related:
  - "[[2026-01-27_MetalLB-L2-좀비-DNS-OPNsense-Unbound]]"
aliases:
  - ArgoCD Force Replace ELB
  - DNS_PROBE_FINISHED_NXDOMAIN
  - ExternalDNS Cloudflare 연동
---

# ArgoCD Force+Replace → ELB 재생성 → Cloudflare CNAME 끊김

## 사건 경위

AWS에 배포된 Kubernetes 클러스터의 ArgoCD 관리자 화면에서 Sync가 제대로 안 되는 것 같아 **Force + Replace** Sync를 실행했더니 서비스 도메인 접속이 불가능해졌다.
- 브라우저: `DNS_PROBE_FINISHED_NXDOMAIN`
- 터미널 `dig`: IP가 나오지 않음.
- 확인해보니 **Elastic Load Balancer의 DNS가 바뀌어 있었다.** 어디서 바꿔야 하나 뒤지다가, 인증서 도메인 CNAME 등록이 **Cloudflare**로 되어 있는 것을 발견 → 바뀐 ELB DNS를 해당 도메인 CNAME 자리에 넣었더니 복구됨.

`DNS_PROBE_FINISHED_NXDOMAIN` → `dig`로 IP 누락 확인 → ELB 교체 인지 → Cloudflare CNAME 수정. 정석적인 트러블슈팅 흐름이었다.

## 1. 왜 ELB 주소가 바뀌었나 (The "Why")

ArgoCD의 `Force`/`Replace`는 강력하지만 **LoadBalancer 타입 Service**나 **Ingress**에 쓸 때 주의해야 한다.
- **일반 Sync**: 기존 리소스 설정을 변경(Patch/Update)만 함 → AWS ELB 유지, 설정만 바뀜.
- **Replace 옵션**: 리소스를 **삭제(Delete)하고 다시 생성(Create)**.

**[사건의 재구성]**
1. **Replace 실행**: K8s 상의 Service(LoadBalancer) 리소스가 삭제됨.
2. **AWS 연동**: 리소스 삭제로 AWS Cloud Controller가 연결된 **기존 ELB를 삭제**.
3. **재생성**: ArgoCD가 리소스를 다시 생성.
4. **새 ELB 탄생**: AWS가 새 ELB 프로비저닝 → **새 DNS 주소** 할당.
5. **DNS 불일치**: Cloudflare CNAME은 여전히 '삭제된 옛 ELB'를 가리킴 → `NXDOMAIN`.

## 2. 응급 복구

Cloudflare에서 CNAME을 새 ELB 주소로 바꿔준 것이 **가장 빠르고 확실한 응급조치**였다. (AWS Route53이면 Alias를 쓰겠지만, Cloudflare를 앞단에 두었으니 CNAME 변경이 정답.)

## 3. 재발 방지: ExternalDNS로 자동화

매번 ELB가 재생성될 때마다(혹은 실수로 지워졌을 때마다) 수동으로 DNS를 바꾸는 건 위험하고 번거롭다. **ExternalDNS**는 클러스터 내부의 Service/Ingress 리소스를 관찰하다가 변경 시 DNS Provider(Cloudflare, Route53 등)의 레코드를 자동 업데이트해준다.

**도입 시나리오**: Ingress에 `external-dns.alpha.kubernetes.io/hostname: myapp.example.com` 주석을 달면, `Replace`로 ELB가 바뀌어도 ExternalDNS가 감지 → Cloudflare API로 CNAME을 새 ELB 주소로 즉시 변경.

### 1단계: Cloudflare API Token 생성
(Global API Key보다 **API Token** 권장)
1. Cloudflare 대시보드 → **My Profile → API Tokens**
2. **Create Token** → **Edit zone DNS** 템플릿 또는 Custom
   - Permissions: `Zone` - `Zone` - `Read`, `Zone` - `DNS` - `Edit`
   - Zone Resources: 적용 도메인(예: `example.com`) 또는 `All zones`
3. 생성된 **Token 값** 복사

### 2단계: Kubernetes Secret 등록
```bash
kubectl create secret generic cloudflare-api-token \
  --from-literal=apiKey="<발급받은_TOKEN>" \
  -n kube-system        # external-dns 설치할 네임스페이스
```

### 3단계: Helm(Bitnami)으로 설치
```yaml
# values.yaml
provider: cloudflare

cloudflare:
  secretName: cloudflare-api-token
  secretKey: apiKey
  proxied: true          # true: 오렌지구름(CDN/WAF), false: 회색구름(DNS Only)

# 안전 설정 (중요)
policy: sync             # k8s 리소스와 DNS 동기화 (생성/수정/삭제 모두 반영)
txtOwnerId: "my-k8s-cluster"   # DNS 레코드 소유자 식별 (다른 클러스터와 충돌 방지)

domainFilters:           # 특정 도메인만 관리 (사고 방지)
  - example.com

logLevel: debug          # 초기 구축 시 권장

sources:
  - service
  - ingress
```
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install external-dns bitnami/external-dns -f values.yaml -n kube-system
```

### 4단계: 사용 (Annotation 추가)
#### A. Service(LoadBalancer)에 적용
ELB 주소가 바뀌어도 이 설정만 있으면 자동으로 Cloudflare CNAME이 변경된다.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    external-dns.alpha.kubernetes.io/hostname: "myapp.example.com"
    # true=오렌지구름(IP 숨김), false=회색구름(IP 노출). values 기본값보다 우선.
    external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"
spec:
  type: LoadBalancer
  # ...
```
#### B. ArgoCD 관리 팁
ArgoCD로 이 Service를 관리한다면, Git 저장소의 YAML에 위 `annotations`만 추가해 Push.

### 5단계: 동작 확인
```bash
kubectl logs -f -l app.kubernetes.io/name=external-dns -n kube-system
```

## 4. ArgoCD Sync 옵션 안전 가이드

운영(Production) Sync 시 체크:
- **Force/Replace 지양**: `ConfigMap`/`Secret` 변경이 잘 안 먹힐 때만 제한적으로 사용.
- **Network 리소스 주의**: `Service(Type: LoadBalancer)`·`Ingress`는 재생성 시 클라우드 리소스(ELB)가 교체됨.
- **Prune 옵션**: Git에서 리소스가 삭제되면 클러스터에서도 삭제. 실수로 Git 파일을 지우고 Sync하면 ELB가 날아갈 수 있으니 주의.

## 꿀팁
- **Proxied(Orange Cloud)**: 웹 서비스(HTTP/HTTPS)면 `proxied: "true"` 권장 — Cloudflare가 앞단에서 받아 ELB 실IP 숨김 + DDoS 방어. TCP/UDP 등 비HTTP거나 직접 접속이 필요하면 `"false"`.
- **`txtOwnerId` 꼭 설정**: 안 하면 나중에 다른 클러스터(예: Staging)에 ExternalDNS를 띄웠을 때 같은 도메인을 서로 건드리며 DNS가 핑퐁을 치는 대참사가 발생.
- 적용 후엔 실수로 `Replace`를 눌러 ELB가 바뀌어도 **1분 내 자동으로 DNS가 따라가** 장애 시간을 획기적으로 줄인다.

> 같은 dangling/stale 레코드의 **사내(OPNsense) 버전**: [[2026-01-27_MetalLB-L2-좀비-DNS-OPNsense-Unbound]]
