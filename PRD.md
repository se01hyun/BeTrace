# BeTrace PRD
> 공간의 서사를 기록하는 실시간 디지털 방명록

---

## 1. 제품 개요

**한 줄 정의:** 특정 GPS 위치에서만 열리는 공유 수첩 — 그 공간을 거쳐 간 이들의 흔적(텍스트·사진·드로잉)을 열람하고 나의 흔적을 남기는 서비스.

**핵심 가치:**
- **장소성** — 반경 100m 이내에서만 접근 가능한 희소성
- **자율성** — 텍스트 / 사진 / 손그림을 자유롭게 조합
- **연결감** — 지금 이 순간 같은 공간에 있는 사람들과의 실시간 동질감

---

## 2. 기술 스택 결정

### Frontend
- **Next.js 14 (App Router)** — SSR + 정적 생성, 파일 기반 라우팅
- **TypeScript** — 전체 타입 안전성
- **Tailwind CSS** — 유틸리티 기반 스타일링
- **Framer Motion** — 수첩 넘기기 / 토스트 애니메이션
- **Fabric.js** — Canvas 드로잉 + 이미지 레이어 병합 (Canvas API 추상화)

### Backend & Infra
- **Firebase Firestore** — 장소 메타데이터, 방명록 페이지 저장
- **Firebase Realtime Database** — 실시간 접속자 수 / 작성 중 상태 동기화
- **Firebase Storage** — 사진 업로드 및 캔버스 결과물(PNG) 저장
- **Firebase Auth (Anonymous)** — 회원가입 없는 익명 세션 관리
- **Firebase Functions** — 신고 접수 시 AI 심사 트리거 (서버리스)

### AI 모더레이션
- **Claude API (`claude-haiku-4-5`)** — 신고된 콘텐츠 자동 심사. 빠른 응답과 낮은 비용을 위해 Haiku 사용

### 알고리즘
- **Haversine Formula** — 클라이언트 측 반경 필터링 (서버 부하 최소화)

---

## 3. 데이터 모델

```
Firestore
├── places/{placeId}
│   ├── name: string
│   ├── lat: number
│   ├── lng: number
│   ├── category: "cafe" | "restaurant" | "landmark" | "other"
│   └── pageCount: number  // 집계 카운터
│
├── places/{placeId}/pages/{pageId}
│   ├── authorId: string        // Firebase Anonymous UID
│   ├── createdAt: Timestamp
│   ├── type: "text" | "photo" | "drawing" | "mixed"
│   ├── status: "active" | "deleted"  // 삭제된 게시글은 열람 제외
│   ├── reportCount: number           // 누적 신고 수
│   ├── content: {
│   │   ├── text?: string
│   │   ├── photoUrl?: string   // Firebase Storage URL
│   │   └── canvasUrl?: string  // PNG export URL
│   │   }
│   └── drawingStrokes?: StrokeData[]  // 리플레이용 궤적 데이터
│
└── reports/{reportId}
    ├── pageId: string
    ├── placeId: string
    ├── reporterId: string      // 신고자 Anonymous UID
    ├── reason: "spam" | "sexual" | "abuse" | "other"
    ├── createdAt: Timestamp
    └── verdict: {              // AI 심사 결과 (Function 처리 후 기록)
        ├── result: "deleted" | "kept"
        ├── confidence: number  // 0~1
        └── reason: string      // AI 판단 근거 (내부용)
        }

Realtime Database
└── presence/{placeId}
    ├── count: number
    └── writing: boolean
```

---

## 4. 주요 기능 명세

### 4.1 위치 기반 장소 진입
- `navigator.geolocation.getCurrentPosition()` 으로 좌표 획득
- Haversine으로 반경 100m 이내 장소 필터링 후 리스트 + 지도 핀 표시
- 장소 선택 시 Firestore `places/{placeId}` 구독 시작

### 4.2 실시간 인터랙션
- 진입 시 Realtime DB `presence/{placeId}/count` 증가, 이탈 시 감소 (`onDisconnect`)
- 작성 시작 시 `writing: true` → 다른 사용자에게 토스트 노출
- 작성 완료/취소 시 `writing: false`

### 4.3 멀티모달 수첩 작성
| 모드 | 입력 | 저장 방식 |
|------|------|-----------|
| 텍스트 | `<textarea>` | Firestore `content.text` |
| 사진 | `<input type="file">` + 카메라 | Storage → `content.photoUrl` |
| 드로잉 | Fabric.js Canvas | PNG export → Storage → `content.canvasUrl` + `drawingStrokes[]` |
| 혼합 | 사진 위에 드로잉 오버레이 | 병합 PNG → Storage |

### 4.5 신고 및 AI 자동 모더레이션

**신고 UX:**
- 각 페이지에 `···` 메뉴 → '이 흔적 신고하기' 버튼
- 신고 사유 선택: 스팸 / 선정적 콘텐츠 / 욕설·혐오 / 기타
- 동일 사용자가 같은 게시글을 중복 신고 불가 (UID 기준)

**AI 심사 흐름:**
```
신고 접수 (Firestore reports/{reportId} 생성)
    ↓
Firebase Function onDocumentCreated 트리거
    ↓
콘텐츠 추출 (text + 이미지 URL)
    ↓
Claude API 호출 (claude-haiku-4-5)
  - 텍스트: 직접 전달
  - 이미지: URL을 vision으로 전달
  - 프롬프트: 스팸·선정성·욕설 여부를 JSON으로 반환 요청
    ↓
판정 결과 처리
  - 위반 확정 (confidence ≥ 0.85): pages/{pageId}.status = "deleted" 즉시 적용
  - 경계 (0.5 ≤ confidence < 0.85): reportCount 누적, 3회 초과 시 삭제
  - 정상 (confidence < 0.5): 신고 기각, verdict.result = "kept"
```

**Claude 프롬프트 설계:**
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

**열람 필터링:**
- `status === "deleted"` 페이지는 수첩 쿼리에서 제외 (`where("status", "==", "active")`)
- 삭제된 페이지 자리는 빈 페이지 없이 자동으로 건너뜀

### 4.4 수첩 열람
- Framer Motion `AnimatePresence` 기반 좌우 스와이프 (드래그 제스처 지원)
- 드로잉 리플레이: `drawingStrokes[]` 배열을 순서대로 재생 (requestAnimationFrame)
- 페이지네이션: Firestore `startAfter` 커서 기반 무한 로드

---

## 5. 파일 구조

```
/e/BeTrace/
├── app/
│   ├── page.tsx                  # 랜딩 + 위치 허용 요청
│   ├── map/page.tsx              # 장소 리스트 + 지도
│   ├── place/[placeId]/
│   │   ├── page.tsx              # 수첩 열람 (스와이프)
│   │   └── write/page.tsx        # 작성 화면
│   └── layout.tsx
├── components/
│   ├── NoteBook.tsx              # 수첩 스와이프 컨테이너
│   ├── Page.tsx                  # 개별 페이지 렌더러
│   ├── DrawingCanvas.tsx         # Fabric.js 드로잉 캔버스
│   ├── PresenceBadge.tsx         # 실시간 접속자 수
│   ├── WritingToast.tsx          # "누군가 흔적을 남기고 있어요..." 토스트
│   └── ReportModal.tsx           # 신고 사유 선택 모달
├── lib/
│   ├── firebase.ts               # Firebase 초기화
│   ├── firestore.ts              # Firestore CRUD 함수
│   ├── presence.ts               # Realtime DB presence 관리
│   ├── storage.ts                # 파일 업로드 유틸
│   ├── haversine.ts              # 반경 계산
│   └── report.ts                 # 신고 제출 함수
├── functions/                    # Firebase Functions (별도 배포)
│   └── src/
│       └── moderateReport.ts     # 신고 트리거 + Claude API 호출
├── hooks/
│   ├── useGeolocation.ts
│   ├── useNearbyPlaces.ts
│   └── usePresence.ts
└── types/
    └── index.ts                  # 공통 타입 정의
```

---

## 6. 구현 순서 (Phase)

### Phase 1 — 골격 (MVP)
1. Firebase 프로젝트 세팅 + `lib/firebase.ts`
2. `useGeolocation` + Haversine 반경 필터
3. Firestore 장소 CRUD + 페이지 작성/열람
4. 텍스트 전용 수첩 작성 + 스와이프 UI

### Phase 2 — 멀티모달
5. 사진 업로드 (Storage)
6. Fabric.js 드로잉 캔버스 + PNG export
7. 사진 위 드로잉 오버레이 (혼합 모드)

### Phase 3 — 실시간
8. Realtime DB presence (접속자 수)
9. 작성 중 상태 토스트
10. 드로잉 리플레이 애니메이션

### Phase 4 — 모더레이션
11. 신고 UI (`ReportModal.tsx`) + Firestore `reports` 컬렉션 연동
12. Firebase Function `moderateReport` 구현 (Claude API 연동)
13. `status === "deleted"` 필터링 + 열람 쿼리 적용

### Phase 5 — 완성도
14. 스케우어모픽 수첩 UI 다듬기 (페이지 텍스처, 그림자)
15. PWA 설정 (모바일 홈화면 추가)
16. 성능 최적화 (이미지 압축, Firestore 인덱스)

---

## 7. 성공 지표

| 지표 | 측정 방법 |
|------|-----------|
| 장소당 평균 페이지 수 | Firestore `places.pageCount` 집계 |
| 평균 체류 시간 | 장소 진입~이탈 시간 차 (presence 기반) |
| 재방문율 | Anonymous UID 기준 7일 내 재접속 여부 |
| 작성 전환율 | 열람 사용자 중 '흔적 남기기' 클릭 비율 |

---

## 8. 제약 및 고려사항

- **GPS 정확도** — 실내에서는 오차 50~200m 발생 가능. 장소 선택 UI에서 수동 보정 허용
- **익명성** — 작성자 식별 정보 없음. 신고는 Anonymous UID로만 기록하여 중복 방지에만 활용
- **AI 오판 대응** — confidence 임계값(0.85)을 보수적으로 설정해 오탐 최소화. 향후 누적 데이터로 튜닝
- **Claude API 비용** — Haiku 사용으로 신고 1건당 약 $0.0003 수준. 신고 폭주 시 Function에서 동일 pageId 중복 심사 방지 처리 필요
- **콘텐츠 크기** — Canvas PNG는 업로드 전 클라이언트 측에서 리사이즈 (max 1920×1080, quality 0.8)
- **오프라인** — Firestore offline persistence 활성화로 열람은 오프라인 지원, 작성은 온라인 필수
