---
type: reference
status: active
publish: true
date: 2026-03-28
tags:
  - type/reference
  - topic/dev
  - topic/obsidian
  - topic/cloud-storage
topics:
  - Obsidian Know-how
  - Image Management
related:
  - "[[2026-03-27_Obsidian-Image-Cloud-Upload-Tools]]"
  - "[[2026-03-28_Image-Upload-test Wow!]]"
---

# Obsidian 이미지 클라우드 업로드 구축 기록

리서치 결과([[2026-03-27_Obsidian-Image-Cloud-Upload-Tools]])를 바탕으로 실제 구축한 내용.

---

## 1. 왜 이 구성을 선택했는가

리서치([[2026-03-27_Obsidian-Image-Cloud-Upload-Tools]])에서 10개 플러그인 + CLI/CI 방식을 비교한 결과, 핵심 요구사항은 다음 네 가지였다:

1. **Cloudflare R2 지원** — 이미 Cloudflare 스택을 사용 중
2. **모바일(iOS) 지원** — 데스크톱 전용은 불가
3. **붙여넣기 시 자동 업로드** — 수동 트리거 방식은 워크플로우에 맞지 않음
4. **로컬 이미지 정리** — Git vault 크기 관리

### 탈락 사유

| 플러그인 | 탈락 이유 |
|---|---|
| Image Auto Upload (PicGo) | PicGo 앱 별도 실행 필요, 모바일 미지원 |
| Image Upload Toolkit | 자동 업로드 아님 (수동 "Publish page" 커맨드), 원본 수정 안 함 |
| S3 Image Uploader | 모바일 미지원, 배치 업로드 제한적 |
| Image Uploader For Note | 모바일 미지원, 수동 트리거 방식 |
| Cloudflare Image Auto Uploader | 모바일 미지원, 배치 업로드 없음 |
| Cloudinary / Imgur | R2 미지원, 외부 서비스 종속 |
| NotePix | GitHub repo 크기 제한 (1~5GB), R2 미지원 |
| WebDAV Image Uploader | 모바일 미지원, 별도 WebDAV 서버 필요 |
| CLI/CI 방식 | 실시간 업로드 불가, 커스텀 스크립트 유지보수 부담 |

### 최종 선택

**Custom Image Auto Uploader**가 유일하게 R2 직접 지원 + 모바일 + 자동 업로드를 모두 만족했다. 여기에 **Paste Image Rename Convert**를 조합하여 붙여넣기 시 파일명 정규화와 WebP 변환을 자동 처리하도록 구성했다.

| 플러그인                           | 역할                                 |
| ------------------------------ | ---------------------------------- |
| **Custom Image Auto Uploader** | 이미지 자동 업로드 (Cloudflare Workers 경유) |
| **Paste Image Rename Convert** | 붙여넣기 시 자동 리네임 + WebP 변환            |

---

## 2. Custom Cloudflare Worker 구축 배경

### 왜 커스텀 Worker인가

Custom Image Auto Uploader 플러그인 제작자는 자체 API 게이트웨이 레포([haierkeys/custom-image-gateway](https://github.com/haierkeys/custom-image-gateway))를 제공한다. Go 기반 서버로 S3/R2/OSS 등 다양한 백엔드를 지원하지만, 별도 서버 운영이 필요하다. 이를 참조하여 **Cloudflare Workers + R2 바인딩만으로 동일한 역할을 하는 경량 Worker**를 직접 구축했다.

Custom Image Auto Uploader 플러그인은 R2에 직접 연결하는 옵션도 제공하지만, 다음 이유로 **Workers 프록시 방식**을 선택했다:

- **R2 직접 접근 시 S3 API credentials(Access Key + Secret Key)를 플러그인 설정에 평문 저장**해야 함 — 유출 시 버킷 전체에 대한 읽기/쓰기 권한이 노출
- Worker를 경유하면 **단일 AUTH_TOKEN만 플러그인에 저장**하고, R2 바인딩은 Worker 내부에서 처리 — credentials가 클라이언트에 노출되지 않음
- Worker가 업로드 경로 생성(`vault/YYYY/MM/DD/timestamp-filename`), 파일명 sanitize, CORS 처리 등을 담당하므로 클라이언트 로직이 단순해짐
- 향후 rate limiting, 이미지 변환, 접근 제어 등을 Worker 레벨에서 추가 가능

### 아키텍처

```
Obsidian (플러그인)
  → POST /journal  (Authorization: AUTH_TOKEN)
    → Cloudflare Worker (obsidian-image-worker)
      → R2 bucket (obsidian-images) 에 저장
      → 공개 URL 반환

브라우저/Obsidian (읽기)
  → GET /journal/2026/03/28/timestamp-filename.webp
    → Worker가 R2에서 조회 → 이미지 응답 (Cache-Control: immutable)
```

### 레포지토리

- **Repo**: [duodecanol/obsidian-image-worker](https://github.com/duodecanol/obsidian-image-worker)
- **스택**: Cloudflare Workers + R2 바인딩 (`obsidian-images` 버킷)
- **인증**: `AUTH_TOKEN` 환경변수 (Worker Secrets)로 업로드 보호, 읽기(GET)는 공개

---

## 3. Custom Image Auto Uploader 설정

- API 엔드포인트: 위 Worker의 URL (`/journal` 경로로 vault prefix 지정)
- 자동 업로드 활성화, 업로드 후 로컬 원본 삭제
- 이미지 압축: 최대 1200×1200, quality 1
- 기존 업로드 이미지 도메인은 exclude 처리 (중복 업로드 방지)

## 4. Paste Image Rename Convert 설정

- 파일명 패턴: `{{fileName}}` (노트 제목 기반)
- 중복 시 `-1`, `-2` 넘버링
- PNG → WebP 자동 변환 활성화

## 5. 테스트 결과

[[2026-03-28_Image-Upload-test Wow!]] 에서 이미지 업로드 및 URL 치환 정상 동작 확인.

---

## 6. Git 보안 설정

- **git-crypt** 도입: 플러그인 `data.json` (API 토큰 포함) 암호화
- **`.gitattributes`**: 플러그인 `main.js`, `styles.css`를 binary로 처리 (diff 생략)
- **`.gitignore`**: `.wrangler/` 디렉토리 제외

대칭키는 Bitwarden에 보관.

---

## 7. Remaining Tasks

- [ ] 모바일(iOS)에서 업로드 동작 확인
- [ ] **Image Uploader For Note** 플러그인 비교 테스트 (로컬 삭제 기능)
- [ ] Cloudflare Workers 업로드 API의 인증/rate limiting 검토
