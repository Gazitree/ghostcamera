# Ghost Camera (귀신앱)

CCTV 필터 + 귀신 합성 셀카 웹앱. 단일 HTML 파일.

## 실행

```bash
npx serve . -p 3000
# http://localhost:3000
```

`getUserMedia`는 `file://`에서 차단됩니다. 반드시 로컬 서버 또는 HTTPS로 실행.

## 파일 구조

```
귀신앱/
├── index.html          # 전체 앱 (HTML/CSS/JS 한 파일)
├── 귀신.png            # 기본 귀신 이미지
├── 귀신1.png ~ 귀신10.png  # 추가 귀신 이미지 (있는 만큼만 로드)
└── ghost-camera-PRD.md # 원본 기획서
```

귀신 이미지 파일명 규칙은 고정입니다 — `귀신.png`, `귀신1.png` … `귀신10.png`. 페이지 로드 시 `loadGhostImage()`가 모두 시도하고 onerror는 무시. 캡처마다 `pickRandomGhost()`로 랜덤 1개 선택.

## 화면 구조

3개 화면을 `.screen.active` 클래스 토글로 전환:
1. `#screen-onboarding` — 카메라 권한 트리거 버튼
2. `#screen-camera` — 라이브뷰 + 셔터
3. `#screen-result` — 캡처 이미지 + 저장/다시찍기

## 렌더링 파이프라인

라이브뷰는 매 프레임 다음 레이어를 그립니다 (귀신은 **라이브뷰에 없음**, 캡처에만 합성):

```
Layer 1: 배경 (mirrored video frame)
Layer 2: 사람 (segmentation mask로 클리핑된 인물)
Layer 3: CCTV 필터 (블루톤 → 스캔라인 → 노이즈 → 비네팅 → 타임스탬프 → 글리치)
```

캡처 시(`buildCaptureCanvas`)는 Layer 2 앞에 **귀신 레이어**가 추가됩니다:

```
Layer 1: 배경
Layer 2: 귀신 (랜덤 이미지, 사람 옆 / 얼굴 옆)
Layer 3: 사람 (mask) — 귀신 위에 덮여 자연스럽게 뒤에 있는 느낌
Layer 4: CCTV 필터
```

## MediaPipe 사용 패턴 (중요)

`@mediapipe/selfie_segmentation@0.1.1675465747` (구버전 고정 — 최신 CDN 불안정).

**자기조절 루프** 패턴 필수. rAF 자유 호출하면 `send()` 큐가 폭주해 첫 프레임에서 멈춥니다:

```
send() → onResults 콜백 → drawFrame() → requestAnimationFrame(sendFrame)
```

`onResults`는 `initSegmentation()`에서 **딱 한 번만** 등록.

## 좌표계 / 미러링

- 비디오는 셀카 미러로 좌우 반전해서 표시 (`ctx.translate(w,0); ctx.scale(-1,1)`)
- MediaPipe segmentationMask는 **원본(미러 X) 좌표계**로 옴
- 사람 바운드 / 얼굴 위치 계산 후, 캔버스 좌표로 변환 시 `mirX = w - origX` 적용

## 귀신 배치 알고리즘 (`computeGhostPlacement`)

1. 사용자 얼굴(상단 25% 슬라이스)의 가로폭 검출
2. 귀신 너비 = `clamp(faceWidth * 2.5, w*0.22 ~ w*0.30)` × 랜덤(0.9~1.15)
3. 좌/우 방향: 공간 넓은 쪽 우선, 동률이면 랜덤
4. 귀신 머리 X = 사람 몸통 *바깥쪽* + 5% 랜덤 겹침 (사람 마스크에 가려지지 않게)
5. 수직 = 사용자 얼굴 높이 ± 35% jitter
6. **얼굴 잘림 방지** strict clamp: 귀신 상단 30% × 가로 80% 영역이 항상 화면 안

`귀신.png` 머리 위치는 이미지 상단 ~13%, 가로 중앙 50% 폭으로 가정. 다른 이미지를 쓰면 클램프 비율 조정 필요.

## 주요 튜닝 값 (`index.html` 내)

| 항목 | 위치 | 현재값 |
|---|---|---|
| 블루톤 | `applyCCTVFilter` 픽셀 조작 | r×0.15, g×0.28, b×1.5+28 |
| 스캔라인 | `applyCCTVFilter` | opacity 0.30, 4px 간격 |
| 노이즈 | `applyCCTVFilter` | 픽셀 수 비례 (1280×720 기준 2400) |
| 비네팅 | `applyCCTVFilter` | radial gradient, 외곽 0.85 |
| 타임스탬프 | `applyCCTVFilter` 하단 | `h-36*scale` / `h-14*scale`, 좌하단 (반응형) |
| 귀신 alpha | `buildCaptureCanvas` Layer 2 | 0.55 |
| 귀신 filter | 동상 | `blur(2px) contrast(1.3)` |
| 귀신 wobble | 동상 | ±4px 랜덤 |
| 글리치 빈도 | `startGlitchTimer` | 700ms마다 일반 14% / 강한 4% |

## 글리치 효과

라이브뷰 전용 CSS 기반. `setInterval`로 700ms마다 확률 체크 → `#screen-camera`에 `.glitch` / `.glitch-hard` 클래스 추가 → 80~160ms 후 제거.

캔버스 자체에는 transform + filter 적용, 스캔라인 오버레이는 `::before` pseudo로.

## 자주 만나는 함정

1. **MediaPipe send 큐 폭주** → 자기조절 루프 패턴 깨지 말 것
2. **mask 좌표 미러 변환 누락** → 귀신이 사람 반대쪽에 나타남
3. **귀신 이미지 색상에 따른 블렌드 모드 호환성** — `screen` 블렌드는 어두운 귀신 이미지에서 거의 안 보임. 현재는 `source-over` 사용
4. **`canvas` CSS `object-fit: cover`** → 화면 비율과 비디오 비율이 다르면 일부 잘려 보임 (저장 시엔 비디오 원본 해상도 그대로)
5. **`귀신N.png` 누락** → onerror로 무시. 콘솔에 `[Ghost Cam] 로드된 귀신 이미지: N개` 확인

## 의존성

- 브라우저 내장: `getUserMedia`, Canvas 2D
- CDN: `@mediapipe/selfie_segmentation@0.1.1675465747`
- 빌드 도구: 없음 (정적 HTML)
