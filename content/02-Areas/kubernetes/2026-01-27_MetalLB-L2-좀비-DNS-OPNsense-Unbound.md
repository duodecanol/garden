---
type: research
status: active
publish: true
date: 2026-01-27
tags:
  - type/research
  - topic/kubernetes
  - topic/metallb
  - topic/dns
  - topic/opnsense
  - status/active
topics:
  - metallb
  - dns-stale-record
  - opnsense-unbound
  - externaldns
  - dangling-record
related:
  - "[[2026-01-26_K3s-MetalLB-Tailscale-MTU-Flannel-트러블슈팅]]"
  - "[[2026-03-26_ArgoCD-Replace-ELB-재생성-ExternalDNS]]"
  - "[[02-Areas/vivident-office-network/index]]"
aliases:
  - MetalLB L2Advertisement 계속 잡힘
  - dangling DNS record
  - OPNsense Unbound Override 삭제
---

# "MetalLB L2Advertisement가 계속 잡힌다" — 사실은 좀비 DNS 레코드

클러스터의 모든 manifest를 destroy하고 노드들을 reboot했는데도 MetalLB의 L2Advertisement가 계속 잡히는 것 같다. `dig`로 확인:
```
dig gateway.imagegen-thrifty.intranet.moelive.tech
;; ANSWER SECTION:
gateway.imagegen-thrifty.intranet.moelive.tech. 3600 IN A 10.78.147.2
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Tue Jan 27 18:55:37 KST 2026
```

> [!note] 결론 먼저
> MetalLB가 좀비처럼 살아있는 게 **아니라**, DNS 레코드가 남아있거나 캐싱된 문제. `dig`는 MetalLB가 관장하는 네트워크(L2/ARP) 계층이 아니라 **DNS 서버의 데이터베이스**를 조회하는 도구다.

## 1. 개념 분리: DNS(dig) vs MetalLB(L2)

- **DNS (`dig` 결과)**: "이 도메인(`gateway...`)의 IP는?"에 대한 답. 외부 DNS 서버나 로컬 캐시에 저장된 **텍스트 정보**일 뿐, 그 IP가 네트워크상에 살아있는지는 확인하지 않는다.
- **MetalLB (L2Advertisement)**: "이 IP(`10.78.147.2`)를 가진 기기가 누구인가?"라는 ARP 요청에 "나(내 MAC)야!"라고 손드는 행위.

**결론**: `dig`가 IP를 반환했다는 건 **DNS 서버가 아직 그 IP 정보를 가지고 있다**는 뜻이지, MetalLB가 실제로 그 IP를 광고(Advertise)하고 있다는 뜻이 아니다.

## 2. 좀비인지 단순 잔여물인지 검증

### Step 1 — 실제 통신 가능 여부 (Ping)
```bash
ping -c 4 10.78.147.2
```
- **응답 없음(Unreachable/timeout)**: MetalLB는 정상 종료됨. `dig` 결과는 남아있는 DNS 레코드(Stale Record)일 뿐, 안심.
- **응답 있음**: 정말 네트워크상에 그 IP를 점유한 기기가 있음 → Step 2.

### Step 2 — ARP 테이블 (누가 IP를 점유 중?)
```bash
arp -an | grep 10.78.147.2
ip neigh show | grep 10.78.147.2
```
- 출력 MAC이 클러스터 노드 인터페이스 MAC과 일치하면 정리 안 된 네트워크 설정 잔존 가능. 단 노드를 reboot까지 했다면 **다른 장비가 우연히 그 IP를 쓰는 중**일 가능성도 배제 못 함.

### Step 3 — DNS 레코드 출처
`dig` 결과의 TTL이 3600(1시간), `SERVER: 127.0.0.53`은 로컬 캐싱 DNS다.
```bash
dig @<사내_DNS_IP> gateway.imagegen-thrifty.intranet.moelive.tech
```

## 3. `resolvectl flush-caches` 했는데도 또 나오는 이유

가장 유력한 이유: **로컬 캐시는 지워졌지만 상위(Upstream) DNS 서버에 기록이 살아있어 즉시 다시 받아왔기 때문.** (브라우저 방문기록을 지워도 다시 접속하면 서버에서 새로 받아오는 것과 동일.)

캐시를 비운 직후 `dig`를 치면 0.1초 만에:
1. `resolvectl flush-caches`로 내부 메모를 비움.
2. `dig` 질의 → `systemd-resolved`는 캐시가 비었으므로 상위 DNS(사내 DNS, 8.8.8.8 등)에 물음.
3. 상위 DNS가 아직 레코드(`10.78.147.2`)를 가지고 있으므로 응답.
4. `systemd-resolved`가 응답을 받자마자 **다시 캐싱**하고 보여줌.

### 확인: 로컬 캐시를 거치지 않고 상위 DNS에 직접
```bash
resolvectl status | grep "DNS Server"     # 상위 DNS 주소 확인
dig @<상위_DNS_IP> gateway.imagegen-thrifty.intranet.moelive.tech
```
여기서도 응답이 오면 **내 PC가 아니라 사내/외부 DNS 서버에 레코드가 안 지워지고 남은 것**(ExternalDNS 등이 삭제 처리 못 하고 죽었을 확률 99%).

### 그 외 가능성
- **`/etc/hosts` 정적 매핑**: DNS를 안 거치고 항상 응답. `grep "gateway.imagegen-thrifty" /etc/hosts` → 있으면 삭제.
- **Flush가 특정 링크에만 적용**: 전역 캐시가 안 지워졌을 수 있음.
  ```bash
  sudo systemd-resolve --flush-caches     # 또는 sudo resolvectl flush-caches
  resolvectl statistics                   # Current Cache Size가 0에 가까운지
  ```

## 4. 범인 확정: 상위 DNS = OPNsense (`10.78.142.1`)

`resolvectl status` 출력:
```
Link 2 (enp5s0)
    Current DNS Server: 10.78.142.1
    DNS Servers: 10.78.142.1
    DNS Domain: vivident.lan
```
→ 범인은 **`10.78.142.1`(상위 DNS = OPNsense 게이트웨이)**. 로컬 캐시를 아무리 비워도 `dig`를 치는 순간 이 서버에 질의가 가고, 이 서버가 여전히 "그 도메인은 `10.78.147.2`"라 응답하기 때문.
```bash
dig @10.78.142.1 gateway.imagegen-thrifty.intranet.moelive.tech    # 즉답 → 이 서버가 범인
```

### 왜 레코드가 안 지워졌나 (Dangling / 고아 레코드)
보통 ExternalDNS 파드가 Ingress/Service 생성 시 DNS 서버에 레코드를 자동 추가한다. `destroy` 시:
1. K8s가 Ingress와 ExternalDNS 파드를 거의 동시에 종료.
2. ExternalDNS가 "DNS 서버야 이 레코드 지워줘" 요청을 보내기도 전에 **파드 자체가 먼저 강제 종료(Kill)**.
3. DNS 서버는 삭제 요청을 받은 적이 없으므로 레코드를 영원히(또는 TTL 만료까지) 유지 → **Dangling DNS Record**.

## 5. 해결: OPNsense에서 레코드/캐시 제거

`10.78.142.1`이 OPNsense면, DNS 처리 방식별로:

### 방법 1 — BIND 플러그인 (ExternalDNS RFC2136 연동 시 유력)
1. **Services → BIND → Configuration → Zones**
2. 해당 도메인(예: `intranet.moelive.tech`) Zone 편집
3. `gateway` A 레코드가 있으면 휴지통 → **Save → Apply Configuration**
4. GUI엔 안 보이는데 `dig`는 계속되면 동적 업데이트(DDNS)가 저널(.jnl)에만 있는 것 → **BIND 서비스 재시작**.

### 방법 2 — Unbound DNS (기본값, **이번 실제 원인**)
별도 BIND 없이 OPNsense 기본 DNS에 **Host Override**로 등록돼 있던 경우.
1. **Services → Unbound DNS → Overrides**
2. Hostname `gateway`, Domain `imagegen-thrifty...` 항목 찾기
3. 휴지통 → **Apply**

> 이 Override는 "무조건 이 IP로 응답하라"는 **정적 매핑(Static Mapping)** 규칙이라, 클러스터를 끄거나 노드를 재부팅해도 사라지지 않고 계속 살아있었다 — 노드 reboot로도 안 죽은 이유.

### 방법 3 — 설정엔 없는데 캐시만 남은 경우 (SSH)
```bash
# Unbound
service unbound restart
unbound-control -c /var/unbound/unbound.conf reload
# BIND
service named restart
```

### 적용 후 검증
```bash
resolvectl flush-caches
dig @10.78.142.1 gateway.imagegen-thrifty.intranet.moelive.tech
# NXDOMAIN(그런 도메인 없음) 또는 무응답이면 성공
```

## 6. 조치 요약표

| 원인 | 조치 |
|---|---|
| 단순 DNS 잔여물 (Ping 실패) | 무시 가능. 또는 실제 DNS 서버(Route53/Bind/Unbound)에서 A 레코드 수동 삭제. 클라 캐시면 `systemd-resolve --flush-caches`. |
| ExternalDNS 문제 | 재배포 시 `--policy=sync`면 ExternalDNS가 "서비스 없는데 레코드 있네?" 하고 자동 삭제. 아니면 DNS 공급자 콘솔에서 수동 정리. |
| 실제 L2 광고 중 (Ping 성공) | 매우 드묾. 다른 물리장비/VM이 그 IP를 중복 할당받아 쓰는지 네트워크 담당자와 확인. |

## 교훈
- 노드 재부팅에도 살아남는 정적 매핑은 **앱(K8s)이 아니라 라우터/DNS 쪽**을 의심.
- DNS 캐시 적중 판별: `resolvectl query <host>` 출력의 `Data from: network` → `cache network` 변화로 확인. 네트워크 세팅 변경 후엔 `sudo resolvectl flush-caches` 습관화.

> 같은 dangling 레코드의 **AWS/Cloudflare 버전**: [[2026-03-26_ArgoCD-Replace-ELB-재생성-ExternalDNS]] · OPNsense 라우터 운영: [[02-Areas/vivident-office-network/index|vivident-office-network]] · 이 클러스터의 MTU 트러블슈팅: [[2026-01-26_K3s-MetalLB-Tailscale-MTU-Flannel-트러블슈팅]]
