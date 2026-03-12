# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

BeTrace — GPS 기반 실시간 디지털 방명록. 특정 위치에서만 열리는 공유 수첩에 텍스트·사진·드로잉을 남기는 서비스. 상세 기능 명세는 `PRD.md` 참조.

**현재 상태:** 설계 완료, 구현 전. `package.json`에 `next`, `react`, `react-dom`만 설치된 상태이며 Next.js 초기 세팅(`next.config.ts`, `tsconfig.json`, `tailwind.config.ts` 등)이 아직 없음.

## Tech Stack

- **Next.js 14 (App Router)** + TypeScript + Tailwind CSS
- **Firebase**: Firestore (데이터), Realtime Database (presence), Storage (파일), Auth (Anonymous)
- **Firebase Functions** — 신고 트리거 + Claude Haiku AI 모더레이션
- **Fabric.js** — 드로잉 캔버스 및 이미지 레이어 병합
- **Framer Motion** — 수첩 스와이프 및 토스트 애니메이션
- **Claude API** (`claude-haiku-4-5`) — 신고 콘텐츠 자동 심사

## Commands

```bash
# 개발 서버
npm run dev

# 빌드
npm run build

# Firebase Functions (functions/ 디렉토리에서)
cd functions && npm run build
firebase deploy --only functions

# 전체 Firebase 배포
firebase deploy
```

## Architecture

### 핵심 데이터 흐름

1. **위치 → 장소**: `useGeolocation` → Haversine 반경 100m 필터 → Firestore `places` 쿼리
2. **열람**: Firestore `places/{id}/pages` (`status == "active"`) → `startAfter` 커서 페이지네이션 → 스와이프 UI
3. **작성**: Fabric.js 캔버스 → PNG export (max 1920×1080, quality 0.8) → Storage 업로드 → Firestore 저장
4. **실시간**: Realtime DB `presence/{placeId}` (접속자 수, 작성 중 상태, `onDisconnect` 자동 감소)
5. **신고**: Firestore `reports` 생성 → Firebase Function 트리거 → Claude API → `pages.status` 업데이트

### 주요 디렉토리

- `app/` — Next.js App Router 페이지 (`/`, `/map`, `/place/[placeId]`, `/place/[placeId]/write`)
- `components/` — UI 컴포넌트 (NoteBook, Page, DrawingCanvas, PresenceBadge, WritingToast, ReportModal)
- `lib/` — Firebase 초기화 및 유틸 (firebase.ts, firestore.ts, presence.ts, storage.ts, haversine.ts, report.ts)
- `hooks/` — useGeolocation, useNearbyPlaces, usePresence
- `functions/src/` — Firebase Functions (moderateReport.ts)
- `types/` — 공통 타입 정의

### Firestore 스키마

```
places/{placeId}
  name: string
  lat: number
  lng: number
  category: "cafe" | "restaurant" | "landmark" | "other"
  pageCount: number  // 집계 카운터

places/{placeId}/pages/{pageId}
  authorId: string          // Firebase Anonymous UID
  createdAt: Timestamp
  type: "text" | "photo" | "drawing" | "mixed"
  status: "active" | "deleted"   // 소프트 삭제. 열람 쿼리는 항상 where("status","==","active") 포함
  reportCount: number
  content: {
    text?: string
    photoUrl?: string        // Firebase Storage URL
    canvasUrl?: string       // PNG export URL
  }
  drawingStrokes?: StrokeData[]  // 리플레이용 궤적 데이터

reports/{reportId}
  pageId: string
  placeId: string
  reporterId: string         // 신고자 Anonymous UID
  reason: "spam" | "sexual" | "abuse" | "other"
  createdAt: Timestamp
  verdict: {                 // AI 심사 결과 (Function 처리 후 채워짐)
    result: "deleted" | "kept"
    confidence: number       // 0~1
    reason: string           // 내부용
  }

Realtime Database
presence/{placeId}
  count: number
  writing: boolean
```

### AI 모더레이션 규칙

판정 임계값:
- confidence ≥ 0.85 → `pages.status = "deleted"` 즉시 적용
- 0.5 ≤ confidence < 0.85 → `reportCount` 누적, 3회 초과 시 삭제
- confidence < 0.5 → 기각, `verdict.result = "kept"`

동일 `pageId`에 대한 중복 심사 방지는 Function 내에서 처리.

Claude 프롬프트 (JSON 응답 강제):
```
당신은 UGC 콘텐츠 모더레이터입니다.
아래 콘텐츠가 다음 중 하나에 해당하는지 판단하세요:
- spam: 반복적 무의미 텍스트, 광고, 도배
- sexual: 성적으로 노골적인 텍스트 또는 이미지
- abuse: 욕설, 혐오 표현, 타인 비하

반드시 아래 JSON 형식으로만 응답하세요:
{"violated": true/false, "category": "spam"|"sexual"|"abuse"|null, "confidence": 0.0~1.0}

[콘텐츠]: {content}
```

### 구현 상의 주요 제약

- **GPS 정확도**: 실내 오차 50~200m 가능 → 장소 선택 UI에서 수동 보정 허용
- **익명성**: 작성자 식별 정보 노출 없음. UID는 중복 신고 방지에만 사용
- **캔버스 크기**: 업로드 전 클라이언트 측 리사이즈 (max 1920×1080, quality 0.8)
- **오프라인**: Firestore offline persistence 활성화로 열람은 오프라인 지원, 작성은 온라인 필수
- **Claude API 비용**: 신고 폭주 시 동일 `pageId` 중복 심사를 Function에서 반드시 차단

### 구현 순서 (Phase)

1. **Phase 1 (MVP)**: Firebase 세팅, useGeolocation + Haversine, Firestore CRUD, 텍스트 수첩 + 스와이프 UI
2. **Phase 2 (멀티모달)**: 사진 업로드, Fabric.js 드로잉, 혼합 모드
3. **Phase 3 (실시간)**: Realtime DB presence, 작성 중 토스트, 드로잉 리플레이
4. **Phase 4 (모더레이션)**: ReportModal, Firebase Function + Claude API, status 필터링
5. **Phase 5 (완성도)**: 스케우어모픽 UI, PWA, 성능 최적화
