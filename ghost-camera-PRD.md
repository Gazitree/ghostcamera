# 👻 Ghost Camera - 구현 기획서
> Claude Code를 위한 기술 구현 문서

---

## 1. 제품 개요

**한 줄 정의:** 실시간 카메라 화면에 CCTV 필터를 씌우고, 촬영 시 귀신이 사람 뒤에 합성되는 웹 카메라 서비스

### 핵심 컨셉
- 공포 X → **미묘한 이상함**
- 과한 연출 X → **진짜 있을 것 같은 느낌**
- 찍는 순간보다 → **찍고 나서 발견하는 재미**

### 레퍼런스 이미지
> 첨부 이미지 2장 참고
- `image_original.png` : 원본 셀카 (필터 없음) → **입력 상태**
- `image_filtered.png` : CCTV 필터 적용 결과 → **목표 출력 상태**

---

## 2. 기술 스택

```
HTML 단일 파일 (index.html)
├── getUserMedia (카메라 스트림)
├── MediaPipe Selfie Segmentation (사람/배경 분리)
├── Canvas API (필터 합성 + 렌더링)
└── requestAnimationFrame (실시간 루프)
```

**왜 HTML 단일 파일?**
- 별도 서버 불필요
- 브라우저에서 직접 실행
- 카메라 접근 완벽 작동
- 추후 Next.js로 이전 용이

---

## 3. 화면 구성

### Screen 1 - 온보딩
```
[앱 로고 / 타이틀]
"01_CCTV_document.mov를 로드합니다."
[카메라 시작 버튼]
```
- 카메라 권한 요청 트리거

### Screen 2 - 메인 카메라 뷰 (핵심)
```
┌─────────────────────────┐
│ NOV 03 2011             │  ← 좌상단 타임스탬프
│ 01:26:18 AM             │
│                         │
│   [실시간 카메라 피드]    │
│   + CCTV 필터 실시간     │
│   + 스캔라인 살아서 움직임│
│                         │
│        [● 셔터]          │  ← 하단 중앙
└─────────────────────────┘
```

### Screen 3 - 결과 뷰
```
┌─────────────────────────┐
│  [캡처된 이미지]          │
│  (귀신 합성 포함)         │
│                         │
│  [💾 저장]  [🔄 다시찍기] │
└─────────────────────────┘
```

---

## 4. 레이어 구조 (렌더링 순서)

```
Layer 1: 배경 (카메라 원본 영상)
Layer 2: 👻 귀신 이미지 (배경 위, 사람 아래)  ← MediaPipe로 위치 결정
Layer 3: 사람 (MediaPipe 마스크로 분리)
Layer 4: CCTV 필터 효과 (전체에 적용)
Layer 5: UI (타임스탬프, 버튼 등)
```

---

## 5. CCTV 필터 상세 스펙

> 레퍼런스: 첨부 `image_filtered.png` 참고

### 5-1. 블루톤
```javascript
// Canvas pixel 조작
r = r * 0.3   // Red 감소
g = g * 0.5   // Green 감소
b = b * 1.2   // Blue 증가 (최대 255)
```

### 5-2. 스캔라인 (핵심 - 살아서 움직여야 함)
```javascript
// requestAnimationFrame 루프 안에서
scanlineOffset += 0.8  // 매 프레임 Y축 이동

// 가로선을 위→아래로 천천히 흘려내림
for (let y = 0; y < height; y += 4) {
  const lineY = (y + scanlineOffset) % height
  ctx.fillRect(0, lineY, width, 2)  // 반투명 검정 가로선
}
```

### 5-3. 노이즈/Grain
```javascript
// 매 프레임 랜덤 픽셀 주입
// → 화면이 살아있는 느낌
for (let i = 0; i < 3000; i++) {
  const x = Math.random() * width
  const y = Math.random() * height
  ctx.fillRect(x, y, 1, 1)  // 랜덤 위치에 흰 점
}
```

### 5-4. 비네팅
```javascript
// radial-gradient로 가장자리 어둡게
const gradient = ctx.createRadialGradient(
  centerX, centerY, innerRadius,
  centerX, centerY, outerRadius
)
gradient.addColorStop(0, 'rgba(0,0,0,0)')
gradient.addColorStop(1, 'rgba(0,0,0,0.7)')
```

### 5-5. 타임스탬프
```javascript
// 좌하단 고정
ctx.font = '14px monospace'
ctx.fillStyle = '#7eb8ff'
ctx.fillText('NOV 03 2011', 20, height - 40)
ctx.fillText('01:26:18 AM', 20, height - 20)
// 실제 현재 시간으로 교체 가능
```

### 5-6. 추가 효과 (선택)
- **가끔 글리치**: 0.5% 확률로 프레임 순간 흔들림
- **해상도 저하**: canvas를 작게 그렸다가 늘리기 (픽셀 뭉개짐)

---

## 6. MediaPipe 사람 감지 + 귀신 배치

### 6-1. MediaPipe 설정
```html
<!-- CDN 로드 -->
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation/selfie_segmentation.js"></script>
```

```javascript
const selfieSegmentation = new SelfieSegmentation({
  locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation/${file}`
})

selfieSegmentation.setOptions({
  modelSelection: 1,  // 1 = 고정밀 모드
})
```

### 6-2. 렌더링 파이프라인
```javascript
function renderFrame() {
  // 1. 카메라 프레임 → MediaPipe 분석
  selfieSegmentation.send({ image: videoElement })

  // 2. 결과 콜백
  selfieSegmentation.onResults((results) => {
    // results.segmentationMask = 사람/배경 마스크
    
    // 3. 배경 그리기
    ctx.drawImage(results.image, 0, 0)
    
    // 4. 귀신 그리기 (배경 위)
    drawGhost(ctx)
    
    // 5. 사람만 마스크로 다시 그리기
    ctx.save()
    ctx.globalCompositeOperation = 'source-atop'
    // 마스크 적용해서 사람만 위에 올리기
    ctx.restore()
    
    // 6. CCTV 필터 전체에 씌우기
    applyCCTVFilter(ctx)
  })

  requestAnimationFrame(renderFrame)
}
```

### 6-3. 귀신 배치 위치
```javascript
// 사람의 오른쪽 어깨 뒤 (기본값)
const ghostX = canvasWidth * 0.6   // 화면 오른쪽
const ghostY = canvasHeight * 0.1  // 상단에서 10%
const ghostWidth = canvasWidth * 0.25
const ghostHeight = canvasHeight * 0.7

// opacity: 0.2 (미묘하게)
ctx.globalAlpha = 0.2
ctx.filter = 'blur(3px)'
ctx.drawImage(ghostImage, ghostX, ghostY, ghostWidth, ghostHeight)
ctx.globalAlpha = 1.0
ctx.filter = 'none'
```

---

## 7. 셔터 → 결과 플로우

```javascript
async function onShutter() {
  // 1. 셔터 플래시 효과
  flashEffect()  // 화면 흰색 깜빡임

  // 2. "현상 중..." 텍스트 표시
  showDevelopingText()

  // 3. 딜레이
  await sleep(500)

  // 4. 현재 Canvas 캡처 (귀신 포함)
  const imageData = canvas.toDataURL('image/png')

  // 5. 결과 화면으로 전환
  showResultScreen(imageData)
}
```

---

## 8. 귀신 에셋 스펙

> ⚠️ 이 파일은 직접 제작 후 제공 예정

```
파일명: ghost_01.png
배경: 완전 투명 (transparent)
색상: 흰색 or 연회색 실루엣
크기: 300 x 600px (세로형, 2:1 비율)
스타일: 서 있는 사람 실루엣
디테일: 없음 (얼굴 이목구비 X)
가장자리: Gaussian blur 2~3px 적용
```

**임시 대체:** 에셋 없을 시 Canvas로 타원 + 직사각형 조합으로 실루엣 그리기

---

## 9. UX 디테일

### 귀신 등장 방식
- MVP: **무조건 등장** (확률 없음, 항상 보임)
- 추후: 20~40% 확률로 랜덤 등장

### 긴장감 연출
```
셔터 클릭
  → 화면 흰색 플래시 (0.1초)
  → "저장 중..." 텍스트 (0.5초)
  → 결과 이미지 fade-in
  → (귀신은 처음부터 있지만 필터에 묻혀 천천히 눈에 들어옴)
```

### 재촬영 유도
```
결과 화면 하단:
[💾 저장]  [🔄 다시 찍기]
```

---

## 10. 디자인 방향

| 항목 | 값 |
|------|-----|
| 배경색 | `#000000` |
| 주조색 | `#1a1a2e` (딥 네이비) |
| 텍스트 | `#7eb8ff` (CCTV 블루) |
| 버튼 | 최소화, 반투명 |
| 레이아웃 | 전체 화면 몰입형 |
| 폰트 | monospace (타임스탬프), sans-serif (UI) |

---

## 11. MVP 구현 순서 (권장)

```
Step 1. HTML 기본 구조 + 카메라 스트림 연결
Step 2. Canvas 실시간 렌더링 루프 구성
Step 3. CCTV 필터 적용 (블루톤 → 스캔라인 → 노이즈 → 비네팅 → 타임스탬프)
Step 4. MediaPipe 연동 (사람/배경 분리)
Step 5. 귀신 이미지 합성 (사람 뒤에 배치)
Step 6. 셔터 → 딜레이 → 결과 화면 → 저장
Step 7. UX 디테일 (플래시, 현상중 텍스트, fade-in)
```

---

## 12. 파일 구조

```
ghost-camera/
├── index.html          # 메인 (전체 기능 포함)
├── ghost_01.png        # 귀신 에셋 (직접 제공 예정)
└── README.md
```

---

## 13. 주의사항

- `getUserMedia`는 **HTTPS 또는 localhost**에서만 작동
- 로컬 파일(`file://`)로 열면 카메라 차단될 수 있음
- → `npx serve .` 또는 `python -m http.server`로 로컬 서버 실행 권장
- MediaPipe CDN 로드 필요 (인터넷 연결 필수)

---

*이 문서 + 레퍼런스 이미지 2장을 Claude Code에게 함께 전달할 것*
