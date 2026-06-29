---
type: reference
status: active
publish: true
date: 2026-06-09
tags:
  - type/reference
  - topic/homelab
  - topic/proxmox
  - topic/linux
  - topic/docker
  - status/active
topics:
  - disk-usage
  - lvm
  - growpart
  - resize2fs
  - docker-prune
  - proxmox-resize
related:
  - "[[2026-06-09_Proxmox-VM-복제-골든템플릿]]"
  - "[[2026-06-09_Proxmox-Docker-Compose-부팅-자동시작-Harbor-로그순서]]"
aliases:
  - 리눅스 디스크 꽉 참
  - growpart No space left
  - LVM lvextend resize2fs
  - Proxmox VM 디스크 증설
---

# Linux / Proxmox VM 디스크 용량 진단 & 확장

디스크가 꽉 찼을 때: **① 원인 추적 → ② 정리 → ③ 그래도 부족하면 증설**. Proxmox VM 은 물리 서버보다 훨씬 쉽게 늘린다.

## 1. 어디서 용량이 쌓이는지 추적

```bash
df -h                                   # 어느 파티션이 100% 인지 (Use%, Mounted on)
sudo du -h --max-depth=1 / | sort -hr   # 그 파티션에서 큰 디렉토리 (꼬리 물고 하위로)
sudo find / -type f -size +500M -exec ls -lh {} \;   # 대용량 파일 직격
sudo ncdu /                             # 대화형 (방향키 탐색) — 강력 추천
```

> [!warning] 함부로 `rm` 금지
> 로그(`/var/log/*.log`)는 지우지 말고 `> file` 로 비우거나 `logrotate` 점검. 패키지 캐시는 `apt clean` / `yum clean all`.

## 2. 흔한 범인: Docker

`/var/lib/docker/overlay2`(이미지·컨테이너 레이어)와 `/var/lib/docker/containers`(컨테이너 로그)가 자주 부풀어 오른다. **긴 이름 폴더를 직접 `rm -rf` 하면 Docker 내부가 꼬여 서비스가 망가진다** — 전용 명령만 사용.

```bash
docker system df            # 항목별(이미지/컨테이너/볼륨/빌드캐시) 사용량
docker system prune         # 중지 컨테이너·미사용 네트워크·dangling 이미지 (안전)
docker system prune -a      # 미사용 이미지 전부 (재pull 가능하면 OK)
docker builder prune -a     # 빌드 캐시 (RECLAIMABLE 100% 인 경우 즉시 회수)
docker system prune --volumes   # 볼륨까지 (데이터 손실 주의)
```

### 컨테이너 로그 폭주 처리
```bash
sudo du -h --max-depth=1 /var/lib/docker/containers | sort -hr   # 로그 크기 확인
sudo sh -c 'truncate -s 0 /var/lib/docker/containers/*/*-json.log'  # 내용만 0바이트로
```
근본 처방 — `/etc/docker/daemon.json` 으로 로그 크기 제한(신규 컨테이너부터 적용, 기존은 recreate 필요):
```json
{ "log-driver": "json-file", "log-opts": { "max-size": "10m", "max-file": "3" } }
```
```bash
sudo systemctl restart docker
```

## 3. 디스크 증설 (2단계)

### 3-1. Proxmox GUI 에서 가상 디스크 늘리기 (VM 켜진 채 가능)
VM 선택 → **Hardware** → Hard Disk(scsi0 등) 선택 → **Disk Action → Resize** → **Size Increment(GiB)** 에 *더할 용량* 입력(최종값 아님: 16→32 면 16 입력) → Resize disk.

### 3-2. 게스트 OS 안에서 인식시키기
하드웨어를 늘려도 리눅스는 옛 파티션 크기만 안다. `lsblk` 로 구조 확인 후 확장.

```bash
lsblk   # sda/vda 가 커졌는지, 늘릴 파티션 이름 확인
```

**일반(non-LVM) — ext4:**
```bash
sudo growpart /dev/sda 1     # 디스크와 번호 사이 띄어쓰기 필수
#   growpart 없으면: sudo apt install cloud-guest-utils
sudo resize2fs /dev/sda1     # xfs 면: sudo xfs_growfs /
df -h
```

**LVM (Ubuntu Server 기본값 — `ubuntu--vg-ubuntu--lv` 가 보이면):**
Ubuntu 설치 기본값은 LV 가 디스크 일부만 차지(예: 30GB 중 10GB 만). 남은 공간을 루트로 끌어온다.
```bash
sudo growpart /dev/sda 3     # 물리 파티션(예: sda3) 먼저 확장 (필요 시)
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
df -h
```

## 4. 함정: 디스크가 꽉 차서 `growpart` 자체가 실패

`growpart` 가 임시 디렉토리를 못 만들어 죽는 catch-22:
```
mkdir: cannot create directory '/tmp/growpart.xxxx': No space left on device
FAILED: failed to make temp dir
```

```bash
# A) 임시파일을 RAM(/dev/shm)으로
sudo TMPDIR=/dev/shm growpart /dev/sda 2

# B) 몇 MB 확보
sudo apt-get clean    # 또는 yum clean all

# 그 후 resize2fs /dev/sdaN (ext4) 또는 xfs_growfs / (xfs)
```

> [!note] 늘렸는데 `lsblk` 가 옛 크기 그대로면 — 커널 rescan
> 하이퍼바이저/클라우드(AWS·GCP·VMware)에서 디스크를 키웠는데 OS 가 모를 때:
> ```bash
> echo 1 | sudo tee /sys/class/block/sda/device/rescan
> ```
> 후 다시 `lsblk` → 커진 게 보이면 growpart/resize 진행.

---
홈랩 인덱스: [[02-Areas/homelab/index|homelab]]
