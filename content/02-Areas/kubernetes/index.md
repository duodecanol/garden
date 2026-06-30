---
title: Kubernetes Index
type: index
date: 2026-06-10
tags:
  - topic/kubernetes
  - topic/k3s
  - topic/homelab
publish: true
---

# Kubernetes

자체 운영하는 K3s 클러스터(**gpu-intranet** / imagegen ComfyUI GPU 워크로드)의 운영·트러블슈팅 지식을 종료 없이 지속 관리하는 영역. 두 내부 네트워크(`10.0.0.0/16` office.lan ↔ legacy 대역)가 **OPNsense 내장 Tailscale**로 연결된 특수 토폴로지 위에서 MetalLB·Traefik·cert-manager·GPU Operator·Terraform/Terragrunt 로 구성.

> [!note] 관련 영역·노트
> - 노드가 도는 가상화/홈랩: [[02-Areas/homelab/index|homelab]]
> - 두 네트워크를 잇는 라우터/방화벽: [[02-Areas/vivident-office-network/index|office-network]] (OPNsense + Tailscale)
> - GPU 드라이버 자동 업그레이드 장애: [[2026-06-03_Thrifty-K3s-NVIDIA-Driver-Mismatch-GPU-Worker-CrashLoop-Incident|gpu-intranet K3s NVIDIA mismatch CrashLoop]]

```dataview
TABLE WITHOUT ID
  file.link AS "Note",
  date AS "Date",
  status AS "Status",
  topics AS "Topics"
FROM "02-Areas/kubernetes"
WHERE file.name != "index"
SORT date DESC
```
