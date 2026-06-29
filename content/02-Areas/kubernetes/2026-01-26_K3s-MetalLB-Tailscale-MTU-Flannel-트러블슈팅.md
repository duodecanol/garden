---
type: research
status: active
publish: true
date: 2026-01-26
tags:
  - type/research
  - topic/kubernetes
  - topic/k3s
  - topic/metallb
  - topic/tailscale
  - topic/networking
  - status/active
topics:
  - k3s
  - metallb
  - tailscale
  - flannel-mtu
  - mss-clamping
  - vxlan
related:
  - "[[2026-03-24_K3s-클러스터-Terraform-Terragrunt-삽질]]"
  - "[[2026-01-27_MetalLB-L2-좀비-DNS-OPNsense-Unbound]]"
  - "[[02-Areas/vivident-office-network/index]]"
aliases:
  - Tailscale K3s MTU
  - flannel MTU 1200
  - TLS handshake hang Tailscale
  - redis Timeout connecting to server K8s
---

# K3s + MetalLB over Tailscale — MTU/Flannel/MSS 트러블슈팅

서버 노드가 있는 네트워크는 **`10.78`** 서브넷이고, 에이전트 노드 중 2개는 **`10.53`**(그리고 `10.79`) 서브넷에 있다. 두 원격 네트워크가 **OPNsense 내장 Tailscale(WireGuard)** 로 묶인 특이한 토폴로지. 동일 구조로 여러 번 배포에 성공했었기 때문에 MetalLB 버전/ServiceLB 충돌 같은 일반적 원인은 처음부터 배제하고, "라우터를 사이에 둔 서로 다른 서브넷"이라는 이번 환경의 가장 큰 변수에 집중했다. 결론적으로 모든 증상은 하나의 근본 원인 — **Tailscale 터널(MTU 1280)을 통과하지 못하는 큰 패킷** — 에서 비롯되었다.

이 노트는 IP Pool 생성 실패부터 Redis 간헐 타임아웃까지, 증상이 한 꺼풀씩 벗겨진 진단 여정을 순서대로 기록한다.

---

## 0. 일반 점검 (이번 건은 해당 없음, 그래도 체크리스트로 보존)

멀티노드 K3s에서 MetalLB IP Pool 생성 실패의 흔한 원인들:

1. **K3s 기본 ServiceLB(Klipper-lb) 미비활성화** — K3s는 `servicelb`를 내장한다. MetalLB를 쓰려면 설치 시 비활성화해야 포트/IP 충돌이 없다.
   ```bash
   cat /etc/systemd/system/k3s.service
   cat /etc/rancher/k3s/config.yaml
   ```
2. **MetalLB 버전별 설정 방식 불일치** — v0.13.0부터 `ConfigMap` → **CRD(IPAddressPool + L2Advertisement)** 로 완전히 바뀌었다. 최신 버전에 구버전 ConfigMap 예제를 따라 하면 Pool이 무시된다. **IP Pool만 만들고 L2Advertisement를 안 만들면 IP가 할당되지 않는다**(흔한 누락).
   ```yaml
   apiVersion: metallb.io/v1beta1
   kind: IPAddressPool
   metadata: { name: first-pool, namespace: metallb-system }
   spec:
     addresses: [ 192.168.1.240-192.168.1.250 ]   # 실제 물리 네트워크 대역
   ---
   apiVersion: metallb.io/v1beta1
   kind: L2Advertisement
   metadata: { name: example, namespace: metallb-system }
   spec:
     ipAddressPools: [ first-pool ]
   ```
3. **Webhook/CRD 상태** — v0.13+는 Validating Webhook으로 유효성 검사. `kubectl get pods -n metallb-system`.
4. **Speaker 파드 / 대역 충돌** — `kubectl logs -l component=speaker -n metallb-system`.

> 이번 케이스는 **1·2번 배제**(이전에 수 차례 성공). 따라서 차이점은 오직 네트워크 구조였다.

---

## 1. 멀티서브넷 + 라우터 = MetalLB가 흔들리는 3가지 지점

L3로 분리된 노드 구조에서 발생 가능한 핵심 원인:

### (a) Layer 2 모드의 구조적 한계 (Broadcast Domain 분리)
MetalLB L2 모드는 **ARP(브로드캐스트)** 에 응답해 작동하는데, **ARP는 라우터(L3)를 넘지 못한다.**
- `10.78` 대역 IP Pool을 만들면, `10.53`의 Speaker 파드는 그 IP에 대한 ARP 요청을 받지도 응답하지도 못한다.
- LoadBalancer 서비스가 `10.53` 노드에 스케줄되면 `10.78` IP를 광고해야 하는데 라우터가 "자기 서브넷 아닌 IP"라며 드랍할 수 있다.

### (b) Validating Webhook 통신 실패 (생성 과정 에러 시 유력)
`kubectl apply -f ip-pool.yaml`이 **타임아웃/Internal Error**로 실패하면 이쪽이다.
- 메커니즘: K3s 마스터(API Server, `10.78`)가 CRD 생성 시 유효성 검사를 위해 MetalLB Controller 파드(Webhook)를 호출한다.
- 시나리오: Controller 파드가 에이전트 노드(`10.53`)에 스케줄됨 → 마스터가 `10.53.x.x:443/9443`으로 호출 → 중간 라우터/방화벽이 그 방향 포트를 막음 → 승인 안 떨어짐.
- 점검: 라우터/방화벽에서 `10.78 ↔ 10.53` 간 K8s 필수 포트(Overlay VXLAN, 443, 9443 등) 개방 여부.

### (c) Memberlist(Gossip) 프로토콜 차단
Speaker 파드들은 `Memberlist`(TCP/UDP **7946**)로 리더/IP 소유 정보를 교환한다. 라우터가 이 포트를 막으면 양쪽 Speaker가 서로를 못 봐 "Split Brain"이 되거나 클러스터 형성에 실패한다.

### 이 구조에서 MetalLB를 제대로 쓰는 정공법 두 가지
- **방법 A: BGP 모드(권장)** — 서브넷이 다를 때 가장 깔끔. 라우터가 BGP를 지원해야 하며, MetalLB가 라우터와 BGP 피어링을 맺어 양 노드가 LoadBalancer IP 경로를 라우터에 전파.
- **방법 B: L2 유지 시 'Split Pool' + nodeSelector** — IP Pool을 서브넷별로 쪼개고 L2Advertisement에 `nodeSelector`로 해당 서브넷 노드만 선택.
  ```yaml
  apiVersion: metallb.io/v1beta1
  kind: L2Advertisement
  metadata: { name: pool-server-subnet }
  spec:
    ipAddressPools: [ pool-10-78 ]
    nodeSelectors:
    - matchLabels: { kubernetes.io/hostname: server-node-name }
  ---
  apiVersion: metallb.io/v1beta1
  kind: L2Advertisement
  metadata: { name: pool-agent-subnet }
  spec:
    ipAddressPools: [ pool-10-53 ]
    nodeSelectors:
    - matchLabels: { kubernetes.io/hostname: agent-node-name }
  ```

---

## 2. 증상 ①: `kubectl apply`(IP Pool CRD) 타임아웃 → Webhook MTU

두 라우터가 Tailscale로 연결되어 있고, `kubectl apply` 시 **타임아웃**이 난다 → **"Tailscale VPN 환경에서의 MTU 불일치로 인한 Webhook 통신 실패"** 가 핵심.

`apply` 타임아웃 = API Server가 MetalLB 설정 검증을 위해 Controller 파드(Webhook)에 요청했는데 응답이 안 돌아온다는 뜻. Tailscale 위에서 K3s(Flannel CNI)를 돌릴 때 매우 빈번하다.

**왜 실패하는가**
1. 경로: Master(API Server) → (Tailscale VPN) → Agent(MetalLB Controller Pod).
2. 패킷 크기: Webhook은 HTTPS(TLS)라 핸드셰이크 패킷이 크다.
3. MTU 충돌: **Tailscale 인터페이스 MTU ≈ 1280**(WireGuard 기반) vs **K3s/Flannel 기본 ≈ 1450**(VXLAN).
4. 결과: K3s 내부 패킷이 Tailscale 터널을 통과하기엔 너무 커서 조각화/폐기 → Webhook 요청 타임아웃.

### 해결 ①-1: MetalLB Controller를 마스터 노드로 강제 이동 (즉시 우회, 추천)
Controller가 마스터(`10.78`)에 있으면 Webhook 트래픽이 Tailscale을 건널 필요가 없다(로컬 통신) → MTU/라우팅 문제 즉시 우회.
```bash
kubectl get nodes                       # 마스터 노드 이름 확인
kubectl get nodes --show-labels         # 마스터 라벨이 master인지 control-plane인지 확인

kubectl patch deployment controller -n metallb-system --type merge -p \
'{"spec":{"template":{"spec":{"nodeSelector":{"node-role.kubernetes.io/control-plane":"true"},
 "tolerations":[{"key":"node-role.kubernetes.io/control-plane","operator":"Exists","effect":"NoSchedule"}]}}}}'
```
> K3s 버전에 따라 마스터 라벨이 `node-role.kubernetes.io/master`일 수 있으니 `--show-labels`로 확인 후 적용. 파드가 마스터에서 `Running`되면 `apply` 재시도 → 성공 확률 매우 높음.

### 해결 ①-2: K3s Flannel MTU 조정 (근본책 예고)
이후 에이전트 노드와 통신하는 모든 파드(로그/모니터링 등)에서 같은 문제가 날 수 있으므로 K3s MTU를 Tailscale에 맞춰 낮추는 게 좋다. Tailscale MTU(1280) − VXLAN 오버헤드(50) = **1230 이하** → 안전하게 **1200**. (정확한 적용법은 §5에서 — 처음엔 `flannel-mtu`로 시도했다가 실패한다.)

---

## 3. 증상 ②: TCP는 붙는데 TLS Client Hello에서 hang

배포는 성공했는데 **Ingress 브라우저 접속이 안 됨**. control-plane이 있는 네트워크에선 `curl` 성공, 그렇지 않은 네트워크에선 handshake에서 멈춘다.
```
user@vnode-06:~$ hostname -I
10.79.160.6 10.42.6.0 10.42.6.1
user@vnode-06:~$ curl -k -v -H "Host: gateway.imagegen-thrifty.intranet.moelive.tech" https://10.78.147.2/docs
* Trying 10.78.147.2:443...
* Connected to 10.78.147.2 (10.78.147.2) port 443      ← TCP 3-way handshake 성공
* TLSv1.3 (OUT), TLS handshake, Client hello (1):       ← 여기서 멈춤
```

> [!important] 진단 시그니처
> **"TCP 연결(Connected)은 성공했는데 TLS Client Hello에서 멈춘다" = 작은 패킷은 통과하지만 큰 패킷이 Tailscale 터널을 통과 못 하고 드랍**되고 있다는 확실한 증거. 라우팅 경로는 뚫려 있다.

**원인 분석**
1. Tailscale은 WireGuard 기반, 가상 인터페이스(`tailscale0`) MTU가 **1280 고정**.
2. K3s(Flannel)는 호스트 감지로 보통 **1450(VXLAN)** 설정.
3. `curl`의 `Client Hello`는 인증서 정보 등으로 패킷이 크다 → K3s는 1450까지 보낼 수 있다고 보고 송신 → Tailscale 인터페이스에서 1280 초과로 폐기(Drop).
4. 보통 라우터가 "패킷이 크니 줄여라"는 ICMP(Fragmentation Needed)를 보내야 하나, 방화벽/VPN 특성상 전달이 안 돼 무한 대기(Hang).

검증 보조: 문제 노드에서 `ping -s 100 10.78.147.2`(작은 패킷)로 도달 자체를 확인.

---

## 4. 증상 ③: Ping도 안 되고 Curl도 hang → MSS Clamping

"Ping도 안 되고 Curl은 Handshake에서 멈춘다"는 상황의 해석:
1. **Ping 실패**: ICMP가 방화벽에 막혔거나 라우팅 불안정.
2. **Curl "Connected" 성공**: TCP 3-way handshake(작은 패킷) 성공 → **라우팅 경로는 뚫려 있다.**
3. **TLS Handshake 멈춤**: 큰 데이터 패킷이 터널을 못 지남.

종합하면 "연결은 되는데 패킷 크기(MTU/MSS) 문제로 데이터가 드랍". 단순히 Flannel MTU만 줄여서 안 되는 경우가 있는데 이는 **TCP MSS(Maximum Segment Size)** 때문. 클라/서버는 "1500까지 받을 수 있어"라 통신하지만 중간 Tailscale(1280)이 감당 못 해 버린다.

### 해결: iptables로 MSS Clamping (TCP 한정)
모든 노드(Master & Worker)에서:
```bash
# 즉시 적용 — 나가는 TCP SYN을 보고 경로상 최소 MTU(Tailscale 1280)에 맞춰 MSS 자동 축소
sudo iptables -t mangle -A POSTROUTING -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

# 위가 안 먹으면 수동 지정
sudo iptables -t mangle -A POSTROUTING -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1200

# 재부팅 시 초기화 방지 (성공 후 저장)
sudo apt-get install iptables-persistent
sudo netfilter-persistent save
```

> **왜 Ping도 실패했나(참고)**: `curl`이 "Connected"를 띄웠다면 TCP는 확실히 된 것이라 라우팅 문제는 아니다. Ping 실패는 방화벽(UFW/FirewallD)의 ICMP 차단이거나 L2 모드에서 ICMP 처리를 못 하는 경우(간혹). 지금 중요한 건 HTTP/TLS(TCP)이므로 Ping 실패는 무시하고 curl에 집중.

---

## 5. 증상 ④: 그래도 안 됨 → 파드 간 통신은 VXLAN(UDP)이라 MSS로 못 고친다

MSS Clamping을 OPNsense에도 이미 적용한 상태였고, 문제 서비스 포트를 강제 개방해 MetalLB IP를 거치지 않고 localhost에서 직접 접근해 보니 **일부 엔드포인트는 동작**했다. 하지만 **Webserver–Redis–Worker 구조에서 Redis 파드와 Worker 파드가 서로 다른 노드(네트워크)에 있을 때 파드 간 Redis 커넥션이 안 됨** → 결국 timeout 408.

### MetalLB L2 IP를 Tailscale 대역(100.x)으로 옮기면? → 불가
1. **ARP 부재**: L2 모드는 ARP 브로드캐스트로 "이 IP는 내 MAC으로"라고 알리는데, `tailscale0`는 P2P 터널이라 ARP 브로드캐스트가 안 된다.
2. **라우팅 혼선**: 물리 LAN 인터페이스에 Tailscale 대역 IP를 띄우면 OPNsense가 "Tailscale IP는 VPN 인터페이스로 가야 하는데 왜 LAN 포트에서?"라며 드랍/꼬임.

### OPNsense MSS Clamping을 켜도 안 되는 진짜 이유 — 이중 캡슐화
- 파드 간 통신 시 K3s(Flannel)는 TCP 패킷을 **VXLAN(UDP, Port 8472)** 으로 포장한다.
- OPNsense 라우터가 보기엔 이 트래픽은 TCP가 아니라 **거대한 UDP 패킷**.
- **MSS Clamping은 TCP 헤더(SYN)만 손댄다 → UDP엔 MSS 개념이 없어 무력.**
- 결과: 내부 데이터(Redis TCP) + VXLAN 헤더 + IP 헤더 ≈ 1500B UDP 패킷이 생성 → Tailscale(1280) 통과 시도 → UDP는 조각화 미지원 또는 `DF` 비트로 **그냥 버려짐** → 파드끼리 연결 시도만 하다 408.

> OPNsense MSS Clamping 위치(참고): **Firewall → Settings → Normalization → MSS Clamping**, 값 1200~1240(1280보다 작게). 버전별 UI 상이.

→ 따라서 **물리적으로 Flannel MTU를 낮추는 것이 유일한 해결책.**

---

## 6. 근본 해결: Flannel MTU 축소 — `flannel-mtu`는 존재하지 않는 옵션

> [!warning] 함정
> `config.yaml`에 `flannel-mtu: 1200`을 넣으면 **K3s가 아예 기동하지 못한다.** 존재하지 않는 키이기 때문. K3s는 Flannel을 내장하지만 MTU 변경용 최상위 플래그를 제공하지 않는다.

올바른 방법은 **Flannel 설정(JSON)을 별도로 작성하고 `flannel-conf`로 참조**:

```bash
# /var/lib/rancher/k3s/flannel-config.json  (모든 노드)
sudo vi /var/lib/rancher/k3s/flannel-config.json
```
```json
{
  "Network": "10.42.0.0/16",
  "Backend": { "Type": "vxlan", "MTU": 1200 }
}
```
> `Network` 값은 K3s 기본 `--cluster-cidr`인 `10.42.0.0/16`. 배포 시 CIDR을 바꿨다면 그 값에 맞춘다. 포인트는 `Backend.MTU: 1200`.

```yaml
# /etc/rancher/k3s/config.yaml  (모든 노드) — 기존 flannel-mtu 줄은 삭제
flannel-conf: /var/lib/rancher/k3s/flannel-config.json
```

### 기존 인터페이스 삭제 (이게 빠지면 99% 안 먹힘)
> [!important] K3s 재시작만으로는 이미 생성된 `flannel.1`/`cni0`의 MTU가 안 바뀐다. **수동 삭제 후 재기동**해야 K3s가 새 MTU(1200)로 재생성한다.

```bash
# 1) K3s 정지
sudo systemctl stop k3s          # 워커 노드: k3s-agent

# 2) CNI 인터페이스 삭제 (핵심)
sudo ip link delete flannel.1
sudo ip link delete cni0         # 존재할 경우

# 3) K3s 시작
sudo systemctl start k3s         # 워커 노드: k3s-agent

# 4) 적용 확인
ip link show flannel.1           # mtu 1200(실측 1150) 확인
```

### 추가 점검: VXLAN 포트
MTU를 고쳐도 안 되면 OPNsense 방화벽에서 K3s 기본 VXLAN 포트 **UDP 8472**가 두 사이트(`10.78 ↔ 10.53`) 간에 열렸는지 확인. 막히면 노드 간 파드 통신이 완전 차단된다.

---

## 7. 검증: netshoot으로 노드 간 큰 패킷 통과 확인

MTU가 `1150`으로 잡힌 것을 확인(`ip link show flannel.1` → `mtu 1150`). Tailscale(1280) 기준 오버헤드 고려해도 안전한 수치.

```bash
# 파드 위치/IP 확인 — Target(Redis)과 Source가 서로 다른 노드여야 의미 있음
kubectl get pods -o wide -A

# 네트워크 도구 다 든 일회용 디버그 파드 (Target과 다른 노드에 뜨도록 유도)
kubectl run tmp-debug --rm -it --image nicolaka/netshoot -- /bin/bash
```
파드 내부에서:
```bash
# 1) TCP 연결(라우팅/방화벽)
nc -zv <REDIS_POD_IP> 6379
#   → Connection to <IP> 6379 port [tcp/redis] succeeded!

# 2) 큰 패킷 통과 (MTU 검증의 핵심) — MTU 1150이면 헤더 28B 빼고 ~1122B
ping -s 1100 -c 2 <REDIS_POD_IP>
#   → 1108 bytes from ... 0% packet loss   (성공 = 터널 오버헤드 문제 해결)

# 3) MTU 초과 패킷은 막혀야 정상
ping -s 1200 -M do -c 2 <REDIS_POD_IP>
#   → ping: sendmsg: Message too large    (OS가 MTU 1150 초과를 막아줌 = 적용 정상)
```

실측 결과 (vnode 환경, Redis 파드 `10.42.4.20`):
```
nc -zv 10.42.4.20 6379            → succeeded!
ping -s 1100 -c 2 10.42.4.20      → 0% packet loss
ping -s 1200 -M do -c 2 10.42.4.20 → Message too large (정상)
```
**완벽** — 네트워크 계층 문제 해결. (`netshoot`에는 `redis-cli`가 **없으므로** `nc`로 Redis 프로토콜 직접 검증 — Redis는 단순 텍스트 기반이라 가능.)
```bash
echo "PING" | nc 10.42.4.20 6379
#   → +PONG   (애플리케이션 계층까지 통신 확인)
```

워커 재시작으로 기존 실패 커넥션 풀 비우기:
```bash
kubectl rollout restart deployment <워커-디플로이먼트-이름>
kubectl logs -f -l <워커-라벨-셀렉터>     # Timeout 408 사라졌는지 확인
```

---

## 8. 잔여: 간헐적 Redis 타임아웃 = 앱 레이어 튜닝

`nc`로 PING/PONG이 즉시 되는데도 Webserver–Redis 간 **한 번씩** redis timeout이 발생:
```
File ".../worker_gateway/main.py", line 100, in get_redis
    await client.ping()
...
redis.exceptions.TimeoutError: Timeout connecting to server
```

기본 네트워크 문제는 끝났고, 이건 **VPN(Tailscale) 특유의 레이턴시 변동(Jitter) + 방화벽의 유휴 연결(Idle Connection) 정리** 때문. `redis-py` 기본값이 LAN을 가정해 VPN 터널에서 너무 민감.

**원인 두 가지**
1. **타이트한 타임아웃**: 기본 연결 타임아웃이 VPN의 일시적 지연을 못 견디고 포기.
2. **유휴 연결 끊김(Stale Connection)**: Connection Pool의 연결을 재사용하려는데 중간 라우터/Tailscale이 오래 무패킷인 TCP 세션을 끊어둠 → 앱이 모르고 쓰다 타임아웃.

**해결: 클라이언트 옵션 강화**
```python
import redis.asyncio as redis

client = redis.Redis(
    host="redis-service-name", port=6379, db=0,
    # 1) 연결/응답 타임아웃 증가 (VPN 고려, 5~10초)
    socket_connect_timeout=10.0,
    socket_timeout=10.0,
    # 2) TCP Keepalive — 중간 방화벽이 idle 연결 끊는 것 방지
    socket_keepalive=True,
    # 3) 풀에서 꺼낼 때 30초+ 지난 연결은 PING으로 검사 (stale 재사용 방지)
    health_check_interval=30,
    # 4) 타임아웃 시 1회 자동 재시도
    retry_on_timeout=True,
)
```

| 옵션 | 설명 | 추천값 |
|---|---|---|
| `socket_connect_timeout` | 최초 TCP 연결 대기 | 5.0~10.0초 |
| `socket_timeout` | 명령 응답 대기 | 5.0~10.0초 |
| `socket_keepalive` | OS 레벨 주기적 빈 패킷으로 연결 유지 | True |
| `health_check_interval` | 풀에서 꺼낼 때 죽은 연결 선검사 | 30초 |

코드 변경 후 워커 재배포하면 간헐 타임아웃이 사라진다. ("연결이 끊긴 게 아니라 응답이 조금 늦는데 앱이 못 참고 끊은 상황".)

---

## 9. 3줄 요약

1. Tailscale 위 K3s에서 **"TCP 연결(Connected)은 되는데 TLS/데이터에서 hang"** = MTU. 파드 간 통신은 VXLAN(UDP)이라 **MSS clamping으로는 못 고친다**(TCP 전용).
2. **`flannel-mtu` 옵션은 없다** → `flannel-conf` JSON의 `Backend.MTU: 1200` + **`ip link delete flannel.1` 후 재기동**(재시작만으론 MTU 안 바뀜).
3. 네트워크 복구 후 남는 **간헐 타임아웃**은 `socket_keepalive`/`health_check_interval`/`retry_on_timeout` 등 **클라이언트 튜닝**으로 마무리.

> 같은 클러스터의 IaC 함정: [[2026-03-24_K3s-클러스터-Terraform-Terragrunt-삽질]] · 좀비 DNS 레코드: [[2026-01-27_MetalLB-L2-좀비-DNS-OPNsense-Unbound]] · 라우터/OPNsense 운영: [[02-Areas/vivident-office-network/index|vivident-office-network]]
