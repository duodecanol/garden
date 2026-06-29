---
type: index
status: active
publish: true
date: 2026-05-05
tags:
  - type/index
  - type/reference
  - topic/dev-tools
  - topic/version-manager
  - status/active
topics:
  - dev-tools
  - version-manager
related:
  - "[[03-Resources/Dev-Tools/index|Dev-Tools]]"
---

# version-manager-comparison

> 2026-05-05 1차 비교 종료. `01-Projects/`에서 `03-Resources/Dev-Tools/`로 이관(reference). 재방문 트리거: aqua 2.59+ 정식 release 또는 ARM 호스트에서 bun을 묶어야 하는 신규 모노레포 의사결정.

`mise` vs `proto` vs `aqua` — 같은 시나리오(python/go/bun/terraform/uv 5종, root pin → subdir override → 재현)를 격리된 Docker 환경 3개에서 실행하고 차이를 정리. 결론은 `2026-05-05_findings.md` 참조.

## 코드 위치

vault 외부: `~/xxDev/version-manager-comparison/`
- `mise/`, `proto/`, `aqua/` 각 하위에 `Dockerfile` + 매니저별 native 설정 파일 + `scenarios.sh`.
- 최상위 `compare.sh`로 세 컨테이너를 차례로 빌드·실행.

## 비교 축

| 축 | 무엇을 본다 |
|---|---|
| 설치 명령 한 줄 | `mise install` / `proto install` / `aqua install --all` |
| 설정 문법 | `.mise.toml` (TOML) / `.prototools` (TOML, flat) / `aqua.yaml` (YAML, registry 기반) |
| 활성화 방식 | PATH 조작(mise) / shim(proto) / `bin/` 디렉터리(aqua) |
| 디렉터리 오버라이드 | `tools/python/` 진입 시 python 버전이 3.12→3.11로 바뀌는가 |
| 도구 커버리지 | python을 실제 런타임으로 다룰 수 있는가 (aqua는 설계상 불가) |

## 노트

```dataview
TABLE WITHOUT ID
  file.link AS "Note",
  date AS "Date",
  status AS "Status"
FROM "03-Resources/Dev-Tools/version-manager-comparison"
WHERE file.name != "index"
SORT date DESC
```
