# Flow Project — PC방 시리즈 시각화 알고리즘 (투스탭 = `37da949` 기준)

> **시각화 알고리즘 단일 진실 소스**. 작성일 2026-05-10.
> 5가지 사용자 룰을 7번 시도 끝에 충족하기 위해 도입한 알고리즘 구조를 기록.

## 사용자 5가지 룰

1. **레퍼런스를 따를 것** — StatView 채널 voronoi 영상 스타일
2. **맵(정사각형) 안 빈칸 없음** — cell들이 컨테이너 빈틈없이 채움
3. **다음 데이터로 점프 없는 transition** — 영역 증가/감소만으로 변화
4. **영역들이 가능하면 맵 벽에 붙어있게** — 외곽 cover 강제
5. **rank-in/out 자연스럽게** — 새 게임 0% → target%, 빠진 게임 from% → 0%

## 사용 중인 알고리즘 — `d3-voronoi-map-tween`

라이브러리 (npm): `d3-voronoi-map-tween@0.0.1`. d3-voronoi-map v2.1.1 + d3-weighted-voronoi 의존.

### 동작 원리

1. **각 시점마다 voronoiMapSimulation 만들기** — d3-voronoi-map의 iteration 알고리즘으로 cell area를 weight 비례로 정확히 분할 (final state까지 tick)
2. **두 시점 사이 voronoiMapTween** — starting/ending simulation을 받아서 site characteristics(x, y, weight) 보간 + 자체 weighted-voronoi 호출로 매 frame cell polygon 새로 계산
3. **vertex-level interpolation** — polygon vertex들이 starting/ending 사이 자동 매핑 + lerp
4. **ENTER/UPDATE/EXIT 자동 분류**:
   - ENTER: ending에만 있음 (rank-in) — 0%에서 점차 등장
   - UPDATE: 양쪽 다 있음 — area 보간
   - EXIT: starting에만 있음 (rank-out) — 점차 사라짐

## 데이터 흐름

```
history (raw JSON)
  │
  ▼
snapshots (SeriesSnapshot[]) — useMemo
  │
  ▼ useEffect (mount 시 1회)
sims = snapshots.map(s => buildSim(s))   ← 모든 시점 voronoiMapSimulation 미리 계산
  │
  ▼ useMemo (currentIndex/segmentT 변경 시)
tween = voronoiMapTween(start, end)       ← 시점 i 앞/뒷 절반 분기
  │
  ▼ useMemo (segmentT 매 frame)
lerpedCells = tween.mapInterpolator()(t)  ← 매 frame 새 polygon 계산
  │
  ▼ props
<VoronoiPolygon cells={lerpedCells} />   ← dumb component, SVG 렌더만
```

## 12 anchor 시스템

### 위치 (시계방향 0~11)

```
점0 ──── 점1 ──── 점2 ──── 점3
 │                          │
점11                        점4
 │                          │
점10                        점5
 │                          │
점9 ──── 점8 ──── 점7 ──── 점6
```

- 꼭짓점 4개: 점0, 점3, 점6, 점9
- 각 변 3등분 점 8개: 점1·2 (top), 점4·5 (right), 점7·8 (bottom), 점10·11 (left)
- 시계방향 인접 정의: 점N과 점(N+1)%12

### key → anchor 매핑 (deterministic + linear probing)

```ts
function assignAnchors(data: LeafData[], anchorCount: number) {
  const sorted = [...data].sort((a, b) => a.key.localeCompare(b.key));  // alphabetic 안정 정렬
  const occupied = new Set<number>();
  const result: Record<string, number> = {};
  for (const d of sorted) {
    let idx = keyToAnchorIndex(d.key, anchorCount);  // hash
    while (occupied.has(idx)) idx = (idx + 1) % anchorCount;  // linear probing
    result[d.key] = idx;
    occupied.add(idx);
  }
  return result;
}
```

- 같은 key set이면 항상 같은 매핑 (sort + hash deterministic)
- 새 key 추가 시 빈 anchor에만 할당 (기존 cell anchor 영향 X)
- 같은 게임은 매번 같은 anchor에서 등장 → 자리 안정 효과

### initialPosition

```ts
.initialPosition((d: LeafData) => {
  const idx = anchorMap[d.key];
  return ANCHORS[idx];  // 정사각형 외곽 12점 중 하나
})
```

voronoi-map iteration이 cell center를 weight balance 위해 이동시키지만, 매 시점 같은 init이라 cell center가 같은 영역에 안착.

## 시점 lerp 룰 — 두 절반 분할

시점 i 노출 시간 T 동안 segmentT가 0→1로 변함.

### 분기

```ts
if (segmentT < 0.5) {
  // 시점 i 앞 절반 — 시점 i-1 → i transition의 후반부
  startSim = sims[currentIndex - 1];
  endSim   = sims[currentIndex];
  t        = 0.5 + segmentT;   // 0.5 ~ 1.0
} else {
  // 시점 i 뒷 절반 — 시점 i → i+1 transition의 전반부
  startSim = sims[currentIndex];
  endSim   = sims[currentIndex + 1];
  t        = segmentT - 0.5;   // 0 ~ 0.5
}
```

### 연속성 보장

- 시점 i 정중앙 (segmentT = 0.5): 분기 변경 instant. 양 분기 모두 t=0.5 (분기 A: t=0.5+0.5=1.0 시점 i, 분기 B: t=0+0=0 시점 i) → 둘 다 시점 i 정확한 값.
- 시점 i 끝 (segmentT≈1) → 시점 i+1 시작 (segmentT≈0): 분기 B의 t=0.5 → 분기 A의 t=0.5. 같은 transition (i → i+1)의 정중앙. **연속, 점프 X**.

### Cell 처리

| Cell 종류 | 시점 i-1 뒷 절반 | 시점 i 앞 절반 | 시점 i 정중앙 | 시점 i 뒷 절반 |
|---|---|---|---|---|
| **기존 (양쪽 존재)** | i-1 → i 보간 (t 0~0.5) | i-1 → i 보간 (t 0.5~1) | i_value 정확 | i → i+1 보간 (t 0~0.5) |
| **Rank-in (i에만)** | tween의 ENTER로 0%부터 등장 | tween 자체 처리 (점차 커짐) | i_value 도달 | UPDATE로 다음 transition |
| **Rank-out (i에만)** | UPDATE로 i-1 → i | UPDATE 마지막 | i_value | tween의 EXIT로 0%로 사라짐 |

## React 컴포넌트 구조

### `PcbangView.tsx` (smart component)

- props: history, events, colorMap
- useState: vizSize (ResizeObserver), sims (precomputed)
- useEffect: containerRef ResizeObserver → vizSize 결정
- useEffect: vizSize 결정되면 모든 snapshot에 buildSim() 호출 → setSims
- useMemo (tween): segmentT 분기 + voronoiMapTween 객체 생성
- useMemo (lerpedCells): tween.mapInterpolator()(t) 호출 + polygon → CellPolygon 변환
- <VoronoiPolygon cells={lerpedCells} /> 렌더 (ShortsFrame 안 + 하단 카드)

### `VoronoiPolygon.tsx` (dumb component)

- props: cells (CellPolygon[]), size, topKey, highlightKey, valueLabel
- 자체 voronoi 호출 X. cells 그대로 SVG로 렌더만.
- safeCells filter (defensive: NaN 좌표/missing key 등 스킵)
- SVG 구조: `<defs><pattern>` (픽셀 grid) → cell polygon × N (단색 fill) → 픽셀 grid overlay × N → 라벨 텍스트 × N (1위 👑 + 흰 라벨 + 금색 % italic, 검은 stroke)

### `useAutoplay.ts` (RAF hook)

- snapshots, totalDurationSec, customWeights 받음
- requestAnimationFrame으로 매 frame currentIndex/segmentT/progress/isMajorEvent 업데이트
- ?duration=N URL로 길이 조정, ?loop=true로 반복
- 변곡점 시점은 customWeights로 5x weight (더 길게 머무름)

## 디자인 디테일

- **컨테이너**: 정사각형 (사용자가 '맵'이라 명명). inset 2%
- **cell 경계**: 직선 polygon (곡선 시도들 다 빈틈/모서리 깎임 부작용)
- **cell 색**: src/data/pcbang/game-colors.json (게임별 brand color)
- **cell 픽셀 그리드**: SVG `<pattern>`으로 100×100 grid line (`rgba(0,0,0,0.22)`, strokeWidth 0.4) → cell 위에 overlay → StatView 채널 모자이크 텍스처 효과
- **라벨**: 검은 stroke + 흰 cell 라벨 + 금색(`#fbbf24`) italic % value. 1위에 👑 위에. 박스 배경 X.
- **변곡점 강조**: drop-shadow 글로우 (해당 시점에서만)
- **1위 교체**: AnimatePresence로 👑 토스트 1.2초 표시

## 성능

- 페이지 mount 시 voronoi simulation 172개 × maxIter 50 = 약 8600 iteration. **client-side 1~3초 소요** (loading 표시 노출).
- 매 frame voronoiMapTween 생성 + interpolator 호출. 6 cell 정도라 빠름 (60fps 유지).
- precompute는 useEffect로 client-only (SSR 안 함, hydration 후 시작).

## 알려진 한계

1. **자리 안정 100%는 아님** — voronoi-map의 iteration이 매 시점 cell center를 weight balance 위해 약간 이동. anchor 기반 init으로 최소화했지만 완전 고정 X.
2. **rank-in/out 정확한 timing** — voronoi-map-tween이 자체 처리. 우리 룰의 "시점 i 앞 절반에만 등장"과 정확히 일치하지 않을 수 있음 (라이브러리 내부 알고리즘 따름).
3. **C-2 룰 채택**: raw weight 그대로, 작은 cell은 외곽 룰(인접 anchor 2점 cover) 위반 허용.

## 마커 (Stop Points)

- **원스탭 = `862a2f1`** (2026-05-10): voronoi-map의 originalObject 구조 정확히 파악 + initialPosition cache 작동 시점. 사용자가 "원스탭으로 돌아가자" 시 복원.
- **투스탭 = `37da949`** (2026-05-10): voronoi-map-tween 정착 + 12 anchor + 라벨 박스 제거 + '맵' 이름 통일. 사용자가 "투스탭으로 돌아가자" 시 복원.

## 의존성

```json
"d3-voronoi-map": "^2.1.1",
"d3-voronoi-map-tween": "^0.0.1",
"d3-weighted-voronoi": "^1.1.3",
"d3-hierarchy": "^3.1.2",
"d3-voronoi-treemap": "^1.1.2",
"framer-motion": "^12.38.0"
```

## 알고리즘 여정 (시도 → 결과)

| # | 알고리즘 | 결과 |
|---|---|---|
| 1 | d3-voronoi-treemap (b08d185 = 원스탭 직전 시점) | iteration 기반 area 정확. cell drift 점프 발생 |
| 2 | + alphabetic sort + prng 고정 | 자리 안정 효과 약함 |
| 3 | precompute + polygon points lerp + vertex resample | 시점 사이 부드럽게 morph. 매 시점 random init이라 자리 변동 |
| 4 | voronoi-map 직접 + initialPosition cache (prevCenter useRef) | 첫 시도 client crash (originalObject.data 미파악) → 두 번째 작동. 여전히 drift |
| 5 | voronoi-map originalObject 구조 정확히 파악 (.data가 input) | client crash 해결 (`862a2f1` = 원스탭) |
| 6 | + 12 anchor 시스템 | rank-in cell이 외곽 anchor에서 등장. UI 만족 |
| 7 | anchor permanent pin (prevCenter cache 무시) | iteration이 여전히 drift |
| 8 | d3-weighted-voronoi 직접 (anchor pinned sites) | iteration 없음 = 점프 사라짐. **단점: power diagram이라 weight 변해도 영역 거의 변하지 않음** |
| 9 | + inwardRatio 0.3 + weight²×100 | 영역 변화 약간 강해짐. 여전히 voronoi-map 수준 미달 |
| 10 | **d3-voronoi-map-tween** (현재) | voronoi-map 두 sim의 site characteristics 보간 + 자체 weighted-voronoi → vertex-level transition. 점프 X + area 정확. **사용자 만족 = 투스탭 `37da949`** |

## 다른 시리즈 확장 가능성

같은 알고리즘 (12 anchor + voronoi-map-tween)을 다른 시리즈에 적용 가능:

- 포켓몬(18 타입): anchor 12개로 분배. 일부 anchor에 2개 이상 cell 매핑 (linear probing) 또는 anchor 18개로 확장 (4 변 × 4 등분 = 16 + 꼭짓점 = 20)
- 국회(5 슬롯): 그대로 적용
- 메이플 서버(11): 그대로 적용
- 롤(7~8): 그대로 적용

## 참고 라이브러리 문서

- d3-voronoi-map: https://github.com/Kcnarf/d3-voronoi-map
- d3-voronoi-map-tween: https://github.com/Kcnarf/d3-voronoi-map-tween
- d3-weighted-voronoi: https://github.com/Kcnarf/d3-weighted-voronoi
