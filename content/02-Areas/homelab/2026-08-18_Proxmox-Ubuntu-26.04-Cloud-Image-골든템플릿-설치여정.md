---
type: reference
status: active
publish: true
date: 2026-08-18
tags:
  - type/reference
  - topic/homelab
  - topic/proxmox
  - topic/ubuntu
  - topic/cloud-init
  - topic/docker
  - status/active
topics:
  - proxmox-ve
  - ubuntu-26.04
  - cloud-image
  - cloud-init
  - vm-template
  - docker-engine
  - homebrew-linux
  - mise
related:
  - "[[2026-06-09_Proxmox-VM-복제-골든템플릿]]"
  - "[[2026-06-09_Proxmox-Docker-Compose-부팅-자동시작-Harbor-로그순서]]"
aliases:
  - Ubuntu 26.04 Proxmox 템플릿 설치
  - Ubuntu Cloud Image 설치 실패
  - Proxmox Cloud Image 부팅
  - Ubuntu VM 골든 템플릿 설치 여정
---

# Proxmox Ubuntu 26.04 Cloud Image 골든 템플릿 설치 여정

Proxmox VE 9.1 계열에서 Ubuntu Server 26.04 Cloud Image를 사용해 Cloud-Init 기반 골든 템플릿을 만드는 과정의 기록.

목표는 Ubuntu VM에 다음 개발 도구를 준비해 복제하는 것이다.

- Docker Engine + Compose plugin
- Homebrew on Linux
- mise
- QEMU Guest Agent
- Cloud-Init 기반 hostname·SSH key·네트워크 초기화

## 1. 결론

Proxmox VE 9.1 계열에서 Ubuntu 26.04 LTS VM 템플릿을 만들 수 있다.

Ubuntu 공식 이미지:

```text
ubuntu-26.04-server-cloudimg-amd64.img
```

주의할 점: 파일 확장자는 `.img`지만 실제 포맷은 QCOW2다. 공식 이미지의 확인값은 다음과 같다.

```text
file format: qcow2
virtual size: 3.5 GiB
compressed disk size: about 821 MiB
```

따라서 Proxmox에 import할 때 파일 확장자만 믿지 말고 `qemu-img info` 또는 `qm importdisk`로 실제 포맷을 확인한다.

## 2. Proxmox GUI의 Guest OS 선택

VM 생성 화면에서 다음 항목만 보이는 것은 정상이다.

```text
Linux 6.x - 2.6 Kernel
```

이 값은 Ubuntu의 실제 커널 버전을 선택하는 것이 아니라 Proxmox의 Linux 게스트 유형(`ostype: l26`)이다. Ubuntu 26.04에 별도의 `Linux 26.x` 선택 항목은 필요하지 않다.

## 3. 첫 번째 부팅 실패

### 증상

SeaBIOS에서 다음 메시지가 출력됐다.

```text
Booting from Hard Disk...
Boot failed: not a bootable disk
Booting from ROM...
iPXE initializing devices...
```

### 분석

Graphics 또는 Serial Console이 원인이 아니었다. SeaBIOS가 부팅 가능한 OS 디스크를 찾지 못해 PXE 네트워크 부팅으로 넘어간 상태였다.

당시 VM에는 빈 16G `scsi0` 디스크와 Cloud-Init Drive가 붙어 있었거나, Ubuntu Cloud Image가 부팅 디스크로 올바르게 연결되지 않은 상태였다.

Cloud-Init Drive는 설정 데이터를 담는 ISO이며 OS 부팅 디스크가 아니다.
실제 import 중간 산출물은 다음 경로에 있었다.

```text
/mnt/pve/vivident-server/import/ubuntu-26.04-server-cloudimg-amd64.img.raw
```

`ls -lah`에서 약 821M/824M로 보이는 것만으로는 raw 변환 성공 여부를 판단할 수 없다. 반드시 포맷과 가상 크기를 확인한다.

```bash
qemu-img info \
  /mnt/pve/vivident-server/import/ubuntu-26.04-server-cloudimg-amd64.img.raw
```

정상적인 원본 QCOW2:

```text
file format: qcow2
virtual size: 3.5 GiB
disk size: about 821 MiB
```

정상적으로 변환된 raw도 `virtual size`는 약 3.5 GiB여야 한다. `file format: raw`이면서 `virtual size`가 824 MiB라면 이미지가 잘못 처리된 것이다.

Cloud Image의 `.img` 확장자만 보고 raw로 취급하지 않는다. Ubuntu 공식 `.img`는 실제로 QCOW2일 수 있으므로 `qemu-img info`로 확인하거나 `.qcow2` 이름으로 명시해 import한다.

Cloud Image를 올바르게 import한 뒤 부팅에 성공했다.

### 해결 방향

1. 빈 `scsi0` 디스크를 분리
2. Ubuntu Cloud Image를 실제 디스크로 import
3. import된 디스크를 `scsi0`으로 연결
4. 부팅 순서를 `scsi0`으로 지정
5. `VirtIO SCSI` 컨트롤러 사용

CLI 형태:

```bash
qm importdisk 9900 \
  /mnt/pve/<storage>/import/ubuntu-26.04-server-cloudimg-amd64.qcow2 \
  <storage>

qm set 9900 --scsihw virtio-scsi-pci
qm set 9900 --boot order=scsi0
```

실제 import 후에는 반드시 다음으로 디스크 구성을 확인한다.

```bash
qm config 9900
```

기대 구조:

```text
scsi0: <storage>:vm-9900-disk-...
ide1: <storage>:cloudinit
scsihw: virtio-scsi-pci
boot: order=scsi0
```

## 4. Ubuntu Cloud Image 부팅 성공

올바르게 import한 뒤 Ubuntu가 부팅됐다. Serial Console에서 systemd 초기화 로그가 보였으며, 다음 로그는 정상적인 초기화 과정이다.

```text
Reached target cryptsetup.target
Finished systemd-modules-load.service
Started multipathd.service
Started systemd-journald.service
```

초기 화면에 로그가 더 나오지 않는다고 즉시 멈춘 것으로 판단하지 않는다. Cloud Image는 Serial Console을 기본 출력으로 사용하고, 첫 부팅에서는 Cloud-Init 작업 때문에 시간이 더 걸릴 수 있다.

확인은 SSH 또는 다음 명령으로 한다.

```bash
cloud-init status --long
```

정상 상태:

```text
status: done
```

## 5. Cloud Image 파티션 구조

현재 Ubuntu VM의 파티션은 다음과 같다.

```text
sda       11.5G  disk
├─sda1     2.4G  part  /
├─sda13  1023M  part  /boot
├─sda14     4M  part
└─sda15   106M  part  /boot/efi
```

이 구조는 Ubuntu Cloud Image가 BIOS와 UEFI 부팅을 함께 지원하기 위해 사용하는 정상적인 GPT 파티션 배치다.

- `/dev/sda1`: root filesystem
- `/dev/sda13`: `/boot`
- `/dev/sda14`: BIOS boot 영역
- `/dev/sda15`: EFI System Partition

VM을 다시 만들어도 같은 Cloud Image를 사용하면 같은 파티션 번호가 반복된다. `sda13`을 `sda3`으로 바꾸려고 파티션 테이블을 재작성하지 않는다.

스크립트에서는 장치 번호를 하드코딩하지 않고 mount point·UUID를 사용한다.

```bash
findmnt -no SOURCE /
findmnt -no SOURCE /boot
findmnt -no SOURCE /boot/efi
lsblk -f
blkid
```

root 확장 대상은 현재 구조에서 다음이다.

```bash
sudo growpart /dev/sda 1
sudo resize2fs /dev/sda1
```

## 6. 확정한 Hardware 설정

현재 정상 부팅한 VM의 Hardware 설정:

| 항목 | 설정 | 판단 |
|---|---|---|
| Memory | 4 GiB | 기본 개발 VM에 적절 |
| CPU | 2 vCPU, 1 socket / 2 cores | 적절 |
| CPU type | `x86-64-v2-AES` | 현대적인 동일 계열 노드에서 적절 |
| BIOS | Default (SeaBIOS) | 현재 Cloud Image에서 정상 동작 |
| Display | Serial terminal 0 (`serial0`) | 서버 VM에 적절 |
| Machine | Default (`i440fx`) | 현재 목적에 충분 |
| SCSI Controller | VirtIO SCSI | 적절 |
| Cloud-Init Drive | `ide1` | 문제 없음 |
| Network | VirtIO, `vmbr0`, firewall enabled | 적절 |
| Serial Port | `socket` | Serial Console에 필요 |

### 유지하는 설정

- SeaBIOS를 유지한다. 현재 정상 부팅되므로 OVMF로 바꿀 이유가 없다.
- Serial terminal 0과 serial socket을 유지한다.
- i440fx는 GPU passthrough·vTPM·특수 PCIe 요구가 없으면 유지한다.
- VirtIO SCSI를 유지한다.

### 조정할 설정

현재 OS 디스크는 약 3.584G로 너무 작다.

```text
일반 개발 VM: 32G 이상
Docker 빌드·이미지 사용: 64G 권장
Harbor/Registry: OS 디스크와 별도 데이터 디스크 100G 이상 권장
```

Proxmox 호스트:

```bash
qm resize 9900 scsi0 32G
```

템플릿에 Docker 데이터를 넣지 않고, Harbor나 대형 이미지 저장소는 별도 `scsi1` 데이터 디스크로 분리한다.

## 7. QEMU Guest Agent

Proxmox VM 설정에서 QEMU Guest Agent를 활성화한다.

```bash
qm set 9900 --agent enabled=1
```

VM 내부 설치:

```bash
sudo apt-get update
sudo apt-get install -y qemu-guest-agent
sudo systemctl restart qemu-guest-agent
systemctl is-active qemu-guest-agent
```

`systemctl enable --now qemu-guest-agent` 실행 시 다음 경고가 나올 수 있다.

```text
The unit files have no installation config ...
```

Ubuntu의 `qemu-guest-agent.service`가 static unit이기 때문에 나타나는 정상 메시지다. `enable` 가능 여부보다 실제 상태와 Proxmox 통신을 확인한다.

```bash
systemctl is-active qemu-guest-agent
```

Proxmox 호스트:

```bash
qm agent 9900 ping
qm agent 9900 get-fsinfo
```

다음 조건이면 정상이다.

```text
qm config 9900  → agent: enabled=1
systemctl is-active qemu-guest-agent  → active
qm agent 9900 ping  → 성공
```

## 8. Disk Cache

`cache=writeback`은 필수가 아니다. 데이터 안정성과 예측성을 우선하는 골든 템플릿에는 기본값 또는 다음 구성을 사용한다.

```text
Cache: No cache
cache=none
Discard: On
IO thread: 선택
```

`writeback`은 일부 쓰기 성능이 좋아질 수 있지만, 호스트 전원 장애나 강제 종료 시 캐시 데이터 유실 위험이 있다. UPS와 별도 장애 복구 체계가 있고 실제 workload benchmark로 확인한 경우가 아니면 사용하지 않는다.

## 9. Docker 공식 설치

Ubuntu 26.04에서는 Docker convenience script가 아니라 Docker 공식 APT 저장소 방식을 사용한다.

### 충돌 패키지 제거

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc docker-buildx podman-docker containerd runc | cut -f1)
```

### 공식 저장소 등록

```bash
sudo apt update
sudo apt install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

Ubuntu 26.04에서는 Suite가 `resolute`로 설정된다.

### Docker Engine 설치

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

sudo systemctl enable --now docker
```

일반 사용자 Docker 권한:

```bash
sudo usermod -aG docker "$USER"
```

재로그인 후 확인:

```bash
docker version
docker compose version
docker run hello-world
```

주의: Docker가 공개한 포트는 UFW/firewalld 규칙을 우회할 수 있으므로 운영용 VM에서는 `DOCKER-USER` 체인을 별도로 점검한다.

## 10. Homebrew와 mise

Homebrew on Linux는 root가 아닌 실제 사용자로 설치해야 한다. `/home/linuxbrew/.linuxbrew`를 사용하는 공식 설치 방식을 따른다.

- 대상 사용자로 설치한다.
- 대상 사용자는 초기 Homebrew 설치를 위해 passwordless sudo가 필요할 수 있다.
- Homebrew는 사용자별 설치이므로 여러 사용자가 쓰는 VM에서는 사용자별 구성이 필요하다.
- mise는 대상 사용자로 설치하고 `HOME`을 root로 두지 않는다.

mise 활성화 예시:

```bash
echo 'eval "$(mise activate bash)"' >> ~/.bashrc
```

Homebrew·mise 설치를 매 클론의 첫 부팅마다 수행하면 외부 네트워크와 최신 설치 스크립트에 의존하므로, 모든 개발 VM에 공통인 경우 템플릿에 한 번 설치하고 봉인한다.

## 11. 최종 봉인 체크리스트

### 현재 진행 상태

- [x] Ubuntu 26.04 Cloud Image import
- [x] 올바른 OS 디스크를 `scsi0`에 연결
- [x] SeaBIOS 부팅 성공
- [x] Serial Console 확인
- [x] Cloud Image 파티션 구조 확인
- [x] Docker 공식 APT 저장소 설치 절차 확정
- [ ] OS 디스크를 32G 이상으로 확장
- [ ] root filesystem 확장 확인
- [ ] Proxmox QEMU Guest Agent 옵션 활성화
- [ ] Guest Agent `active` 및 `qm agent ping` 확인
- [ ] Docker 설치 및 `hello-world` 확인
- [ ] Homebrew 설치
- [ ] mise 설치
- [ ] Docker 이미지·컨테이너·볼륨 정리
- [ ] Cloud-Init 캐시 정리
- [ ] machine-id 초기화
- [ ] SSH host key 제거
- [ ] VM 종료 후 Template 변환

### 봉인 명령

```bash
sudo cloud-init clean
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
sudo rm -f /etc/ssh/ssh_host_*
sudo poweroff
```

Proxmox 호스트에서 종료를 확인한 뒤:

```bash
qm template 9900
```

## 12. 공식 참고 문서

- [Proxmox Cloud-Init Support](https://pve.proxmox.com/wiki/Cloud-Init_Support)
- [Ubuntu 26.04 Cloud Images](https://cloud-images.ubuntu.com/releases/resolute/release/)
- [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Homebrew on Linux](https://docs.brew.sh/Homebrew-on-Linux)
- [mise Getting Started](https://mise.jdx.dev/getting-started.html)

---
홈랩 인덱스: [[02-Areas/homelab/index|homelab]]
