# Android View 렌더링 파이프라인 정리

---

## 한 줄 요약

`ViewGroup.addView()`는 그 자리에서 화면을 다시 그리지 않는다. **`requestLayout`(위로)로 measure/layout 재실행을 예약하고, `invalidate`(위로)로 다시 그릴 dirty 영역을 표시**한 뒤 다음 프레임 traversal 1회로 몰아서 처리한다. 이후 `measure → layout → draw(record)`가 메인 스레드(CPU)에서 display list를 기록하고, **합성(composite)은 RenderThread(GPU)가 각 View의 RenderNode를 조립**해 한 장의 프레임으로 만든다.

> 핵심 축: **요청(위로) vs 실행(아래로)** → **재기록(CPU, 바뀐 뷰만) vs 재합성(GPU, 캐시 재사용)** → **메인 스레드 ↔ RenderThread(sync로 묶임)**

---

## 1. `addView()` 내부의 `requestLayout` / `invalidate` 호출

```java
public void addView(View child, int index, LayoutParams params) {
    ...
    requestLayout();      // 부모(this)를 layout 요청 상태로 마킹
    invalidate(true);     // 부모를 redraw 대상으로 마킹
    addViewInner(child, index, params, false);  // 실제로 child를 mChildren에 추가
}
```

**핵심은 순서에 있다.**

- `addViewInner()`는 내부에서 `child.setLayoutParams()`를 호출하고, 이게 다시
  `child.requestLayout()`를 트리거한다.
- `View.requestLayout()`에는 다음 가드가 있다:
  ```java
  if (mParent != null && !mParent.isLayoutRequested()) {
      mParent.requestLayout();  // 부모가 아직 요청 안 했을 때만 위로 전파
  }
  ```
- 그래서 `addViewInner()` **전에** 부모의 `requestLayout()`를 미리 호출해두면,
  child의 내부 `requestLayout()`가 위로 전파될 때 부모가 이미
  `isLayoutRequested() == true`라서 **루트까지 올라가는 중복 전파가 조기 차단**된다.

**두 호출의 역할:**
1. **의미상** — 자식이 추가되면 부모의 크기/배치가 바뀔 수 있으니 부모 자신을
   re-measure/layout(`requestLayout`) + redraw(`invalidate`) 대상으로 표시.
2. **최적화상** — 순서를 앞에 둠으로써 `addViewInner` 내부에서 발생하는 자식의
   `requestLayout` 전파가 트리 전체를 다시 훑지 않도록 단락(short-circuit).

---

## 2. 요청(request) vs 실행(execution) — 방향이 반대

| 단계 | 방향 | 대상 |
|---|---|---|
| **요청** `requestLayout()` | 자식 → 부모 → … → ViewRootImpl (**위로**) | addView **즉시** |
| **실행** `measure` | 부모 → 자식 (**아래로**, DFS) | 다음 프레임 |
| **실행** `layout` | 부모 → 자식 (**아래로**, DFS) | measure 직후 |
| **실행** `draw` | 부모 → 자식 (**아래로**) | layout 직후 |

- "measure는 자식으로, layout은 부모로 전파" 는 **부정확한 표현.**
  - `requestLayout`(요청)은 **부모로(위로)** 전파
  - `measure`/`layout`(실행)은 **자식으로(아래로)** 전파
- measure도 layout도 **실행은 둘 다 부모→자식(아래로)** 흐른다.

---

## 3. 계층 구조에서 `addView` 시 순서

구조: `Root → A → B → C`, 동작: `C.addView(D)`

### 1단계: 요청이 위로 (addView 시점, *즉시*)

```
C.addView(D)
 ├ C.requestLayout()          → C에 FORCE_LAYOUT, 부모로 전파
 │   └ B.requestLayout()      → B, 부모로
 │       └ A.requestLayout()  → A, 부모로
 │           └ ... → ViewRootImpl.scheduleTraversals()  ← 다음 프레임 예약(즉시 실행 X)
 └ addViewInner(D)
     └ D.setLayoutParams() → D.requestLayout()
         └ 부모 C로 전파하려다 C.isLayoutRequested()==true → 멈춤
```

> addView 순간엔 measure/layout이 돌지 **않는다.** `FORCE_LAYOUT` 플래그만
> 경로(C→B→A→root)에 찍고, 다음 프레임 traversal만 예약.

### 2단계: 실행이 아래로 (다음 프레임 `performTraversals`)

```
measure:  A.onMeasure → B.measure → B.onMeasure → C.measure → C.onMeasure → D.measure → D.onMeasure
layout:   A.onLayout(B배치) → B.onLayout(C배치) → C.onLayout(D배치) → D.layout
draw:     dirty 영역만
```

각 `measure()`는 `FORCE_LAYOUT`이거나 spec이 바뀌었을 때만 `onMeasure()` 실제 실행:
```java
final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
if (forceLayout || specChanged) {
    onMeasure(...);
}
```

**핵심:** ① addView는 실행을 즉시 하지 않고 다음 프레임에 몰아서 한다.
② `FORCE_LAYOUT`이 찍힌 경로만 재측정하므로 형제 서브트리는 `onMeasure`를 건너뛴다.

---

## 4. `requestLayout`이 무시되는 조건

### A. 완전히 early-return (진짜 무시)

**layout 패스 진행 중 같은 뷰가 다시 요청**
```java
if (viewRoot != null && viewRoot.isInLayout()) {
    if (!viewRoot.requestLayoutDuringLayout(this)) {
        return;   // 무시 (무한 layout 루프 방지)
    }
}
```

### B. 자기 플래그만 세팅, 상위 전파는 스킵

**부모가 이미 layout 요청 상태** — `!mParent.isLayoutRequested()`.
(addView 최적화가 이 케이스. 완전 무시 아님, 상위 전파만 단락)

### C. 스케줄 레벨에서 합쳐짐 (coalesce)

- **이미 traversal 예약됨** — `ViewRootImpl.scheduleTraversals()`의
  `if (!mTraversalScheduled)` 가드. 한 프레임에 100번 불러도 traversal은 1번.
- **layout-in-layout 처리 중** — `mHandlingLayoutInLayoutRequest == true`면 무시.

### D. 스케줄 대상 자체가 없음

- **Window 미부착** — `mAttachInfo == null` / `mParent == null`.
  플래그는 찍히나 전파·스케줄 안 됨. attach 시 반영.

> **주의:** 다른 스레드에서 호출하면 `checkThread()`가
> `CalledFromWrongThreadException`을 던진다 → "무시"가 아니라 **크래시**.

---

## 5. `FORCE_LAYOUT`이 찍히는 범위 = "수직 경로 + 새 자식"

```
        Root      ← FORCE_LAYOUT (조상)
         │
         A        ← FORCE_LAYOUT (조상)
         │
         B        ← FORCE_LAYOUT (조상)
         │
         C(this)  ← FORCE_LAYOUT (addView 대상)
        / \
      기존  D      ← FORCE_LAYOUT (새 자식)
      자식
       │
      (형제 서브트리는 플래그 안 찍힘)
```

- **찍힘**: 새 자식 D + `this`(C) + C의 조상들(B, A, Root)
- **안 찍힘**: C의 기존 자식·서브트리, A·B의 다른 형제 서브트리
- "모든 View에 requestLayout 발생"은 **틀림.** 추가 지점→루트 수직 경로 + 새 자식만.

---

## 6. `invalidate(boolean invalidateCache)`의 역할

`invalidateCache` = **"이 View의 그리기 결과물(display list / drawing cache)까지
폐기하고 다시 기록할지"** 를 결정.

```java
void invalidateInternal(..., boolean invalidateCache, boolean fullInvalidate) {
    mPrivateFlags |= PFLAG_DIRTY;              // 항상: 다시 그려야 함(dirty)
    if (invalidateCache) {
        mPrivateFlags |= PFLAG_INVALIDATED;            // display list 무효화 → 재기록
        mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;   // 캐시 무효화
    }
    // 부모로 dirty 영역 전파
}
```

- **`true`** → `PFLAG_INVALIDATED` 세팅. 다음 프레임에 `draw()` 재실행 → display list 재기록.
- **`false`** → `PFLAG_DIRTY`만. display list는 재사용, **재합성/재출력만**. 내용·크기 안 바뀐 경우 최적화.

> `addView`가 `invalidate(true)`를 부르는 이유: 자식이 추가되면 ViewGroup이 실제로
> 그리는 내용이 바뀌므로 display list를 재기록해야 함.
> `invalidate(boolean)`은 `@hide` API — 프레임워크 내부 전용.

---

## 7. `dirty` 개념

**dirty = "변경은 됐는데 그에 따른 반영(재조정)이 아직 완료되지 않은 pending 상태"**

- CS 전반의 공통 은유 (OS dirty page, 캐시 dirty bit, ORM dirty checking 등과 동일 계열).
- **"실패/비정상"이 아니라 "아직 완료 안 됨"** 이 본질. 대부분 곧 처리될 정상 상태.

### View dirty vs DiskLruCache(Okio) journal DIRTY 비교

| | View dirty | DiskLruCache DIRTY |
|---|---|---|
| 문제 | 렌더 결과가 stale | 편집이 미커밋(원자성·내구성) |
| 저장 | 메모리 플래그 | journal에 영속 기록 |
| 해소 | 다시 그림 | commit(→CLEAN) 또는 abort(삭제) |
| 관심사 | 성능(불필요 redraw 회피) | 크래시 안전성 |

- 추상 개념("clean과 어긋나 재조정 필요")은 동일.
- View dirty = *"다시 그려라"* 신호 / journal DIRTY = *"이 편집은 아직 신뢰 불가"* 트랜잭션 상태.
- journal DIRTY는 재시작 시 짝(CLEAN)이 없으면 **미완성 편집으로 폐기** — 이때 비로소 "비정상".

---

## 8. `invalidate`도 부모로 전파된다

`requestLayout`과 전파하는 것·방식이 다르다.

하드웨어 가속(API 14+ 기본)에서 무효화는 루트까지 상승한다.
(**Android 8.0(API 26)+ 부터는 `onDescendantInvalidated`로 통합**됐고,
그 이전은 `invalidateChild()` / `invalidateChildInParent()`가 담당했다.)
```java
public void onDescendantInvalidated(View child, View target) {
    mPrivateFlags = (mPrivateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DIRTY;  // 조상은 DIRTY만
    mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
    if (mParent != null) {
        mParent.onDescendantInvalidated(this, target);
    }
}
```

| | requestLayout | invalidate |
|---|---|---|
| 전파하는 것 | `FORCE_LAYOUT` 플래그 | dirty **영역(Rect)** |
| 목적 | measure/layout 재실행 | **draw만** 재실행 |
| 조상 처리 | 조상도 재측정 | 조상은 `PFLAG_DIRTY`만(재합성), 재기록은 invalidate된 뷰만 |
| 조기 종료 | 부모가 이미 요청 상태면 단락 | 단락 없음(계속 위로) |

- 한 프레임의 여러 invalidate/requestLayout은 `mTraversalScheduled`로 **하나의 traversal로 합쳐짐.**

---

## 9. 예시에서 `invalidate` 흐름

`C.addView(D)`의 `invalidate(true)` 부분.

### 중요한 타이밍: invalidate 대상은 D가 아니라 C

```java
requestLayout();
invalidate(true);        // this = C. 이 시점엔 D가 아직 트리에 없음!
addViewInner(child, ...); // D는 이 다음에 추가됨
```

### 흐름 (HW 가속)

```
C.invalidate(true)
 └ C.invalidateInternal(..., invalidateCache=true, fullInvalidate=true)
     ├ C: PFLAG_DIRTY | PFLAG_INVALIDATED, DRAWING_CACHE_VALID 해제 → C display list 재기록
     └ B.invalidateChild(C, damage)
         └ (HW가속) B.onDescendantInvalidated(C, C)
             ├ B: PFLAG_DIRTY (재합성만)
             └ A.onDescendantInvalidated(B, C)
                 ├ A: PFLAG_DIRTY
                 └ ... Root ... → ViewRootImpl.invalidate() → scheduleTraversals()
```

D는 이후 `addViewInner` → `D.setLayoutParams()` → `D.requestLayout()`로
`FORCE_LAYOUT | PFLAG_INVALIDATED`가 붙고, 다음 프레임 layout 후 draw에서 첫 렌더.

| 뷰 | draw() 재실행(재기록) | 최종 합성 참여 |
|---|---|---|
| C, D | ✅ | ✅ |
| C의 기존 자식 | ❌ (재사용) | ✅ |
| B, A, Root | ❌ (재사용) | ✅ (재합성) |

> **draw() 재실행(CPU 비용)은 C, D에만** 발생. GPU 합성은 dirty 영역 단위라
> 조상·겹치는 형제의 캐시된 display list도 재사용되어 조합됨(재기록보다 훨씬 쌈).

---

## 10. 합성(composite)이란

`measure → layout → draw`는 **CPU(메인 스레드)** 앞부분, 합성은 그 뒤
**GPU(RenderThread)** 단계.

### 소프트웨어 렌더링 (옛날)
`onDraw(Canvas)`에서 `drawText`/`drawRect`가 **즉시 픽셀을 칠함**. 합성 개념 거의 없음.

### 하드웨어 가속 렌더링 (API 14+ 기본) — `draw`가 둘로 쪼개짐

```
[메인 스레드 - CPU]
 measure ─ layout ─ draw(record)
                      │  onDraw() 명령이 픽셀이 아니라
                      │  "그리기 명령 목록" = display list(RenderNode)로 기록만
                      ▼
                    sync (display list를 RenderThread로 넘김)
                      │
[RenderThread - GPU]  ▼
              render / composite  ← '합성'
                 각 View의 RenderNode를 GPU로 실행하고
                 계층·투명도·transform을 적용해
                 '하나의 최종 화면 프레임'으로 조합
```

**합성 = 개별 RenderNode(그리기 명령 캐시) 조각들을 부모-자식 순서로 겹치고
위치·알파·transform을 적용해 GPU가 한 장의 프레임으로 조립하는 과정.**

- 재기록(CPU) 대상 = 내용 바뀐 뷰만 / 재합성(GPU) = 캐시된 RenderNode 재사용해 조합.
- 합성은 두 층: ① RenderThread가 View들의 RenderNode를 합쳐 앱 프레임 생성,
  ② SurfaceFlinger가 여러 Window Surface를 모아 최종 디스플레이 프레임 합성.

---

## 11. Surface / SurfaceFlinger 는 addView가 만드는 게 아님

- **하나의 Window = 하나의 Surface**(그리기 버퍼 한 장).
- 그 Window의 **전체 View 계층(Root~D)은 전부 이 하나의 Surface에 그려짐.**
- `addView`는 Surface를 새로 만들거나 건드리지 않음. **이미 존재하는 Window의
  Surface에 그려질 View 트리만 바꾸는 것.**
- **addView 관련 범위:** View 트리 변경 → display list 재기록 → RenderThread가
  그 Window의 (이미 있는) Surface에 렌더 **까지.**
- SurfaceFlinger는 addView와 무관하게 **매 프레임 항상 도는** 시스템 합성 단계.

---

## 12. RenderThread 생성과 동작

### 생성 시점
- **프로세스당 1개** 싱글턴 네이티브 스레드 (`RenderThread::getInstance()`, C++ libhwui).
- **지연 생성**: 첫 하드웨어 가속 윈도우가 `ThreadedRenderer`를 초기화할 때.
- 한 번 생성되면 **프로세스 종료까지 유지**, 모든 윈도우가 공유. 스레드명 `"RenderThread"`.

### 한 프레임 흐름
```
VSYNC (Choreographer)
   │
[메인]  measure → layout → draw(record)   ← display list 기록
   ▼  ThreadedRenderer.draw() → nSyncAndDrawFrame()
[sync]  기록한 RenderNode 트리를 RenderThread로 넘김
        ↳ 이 sync 동안만 메인 스레드 잠깐 블록(핸드오프)
   │
[RenderThread]  display list를 GPU 실행 → 버퍼 렌더 → SurfaceFlinger 큐잉
   │            (이 사이 메인은 해제되어 다음 프레임 준비 가능)
```

**포인트:**
1. **sync 구간만 짧게 블록** — 실제 GPU 렌더는 RenderThread 담당.
2. **파이프라이닝** — RenderThread가 N 렌더 중 메인은 N+1 traversal 시작 가능.
3. **메인 스레드 독립 애니메이션** — RenderNode 기반(Ripple, reveal,
   ViewPropertyAnimator 하드웨어 레이어)은 RenderThread가 자체 진행 → 메인이
   잠깐 바빠도 안 끊김.

---

## 13. sync 블로킹의 진짜 원인

### 맞는 직관: display list 이관은 가벼움
HWUI **스테이징 모델** — RenderNode가 staging(메인 기록) / active(RenderThread 사용)
두 display list 보유. sync는 `pushStagingDisplayListChanges`로 **staging→active
스왑/이관**이라 ops 전체 딥카피가 아님. "명령이 많아도 이관 자체는 가볍다"는 맞음.

### 그런데도 블록되는 이유

**① 랑데부(rendezvous) — 핵심**

sync는 메인·RenderThread의 **동기점.** 메인이 프레임 N을 sync하려는데
RenderThread가 아직 N-1 렌더 중이면 **메인이 sync에서 대기.**
```
RenderThread:  [── N-1 렌더(GPU) ──][── N 렌더 ──]
메인 스레드:    N sync요청 →★블록★대기┘ (N-1 끝나야 진행)
```
display list가 가벼워도 **직전 프레임이 무겁거나 GPU가 느리면(오버드로, 큰 서피스)**
백프레셔가 메인 sync를 밀어버림 = jank의 흔한 원인.

**② 리소스(Bitmap) GPU 업로드**

새 `Bitmap`은 sync 단계에서 **GPU 텍스처로 업로드**될 수 있음. display list엔
"이 비트맵 그려라"만 있지만 픽셀 전송은 sync에서 → **큰 비트맵이면 sync가 무거움.**

| 관점 | 무거운가? |
|---|---|
| display list ops 이관 | 가벼움 (스테이징 스왑, 딥카피 아님) |
| RenderThread 랑데부 | 이전 프레임 무거우면 **여기서 대기** ← 실제 블로킹 원인 |
| Bitmap 텍스처 업로드 | 큰 비트맵이면 무거움 |

### RenderThread 단일 → 프레임 직렬 → 메인과 결국 묶임
- RenderThread는 프로세스당 1개 → N-1, N, N+1 **직렬 처리.**
- RenderThread가 바쁘면 메인의 sync가 블록 (무한이 아니라 **백프레셔**로 파이프라인
  깊이 제한 → 생산 속도를 소비 속도에 맞춤).
- **SDK 관점 시사점:** 렌더 경로에서 GPU 부하(오버드로, 큰 비트맵/텍스처 업로드,
  복잡한 하드웨어 레이어)를 만들면 그 비용이 sync를 통해 **메인 스레드로 역류.**
  "백그라운드 스레드니 메인과 무관"하다고 방심 금물 — 프레임 단위로 메인과 묶여 있음.

---

## 실무적으로 기억할 3문장

1. **`addView`는 즉시 렌더링하지 않는다.** `FORCE_LAYOUT` 플래그를 추가 지점→루트의 수직 경로 + 새 자식에만 찍고, 다음 프레임 traversal 1회로 몰아서 처리한다. "모든 View가 다시 그려진다"는 오해다.
2. **재기록(CPU)과 재합성(GPU)은 다른 비용이다.** display list 재기록은 내용이 바뀐 뷰에만 발생하고, 조상·형제는 캐시된 RenderNode를 GPU가 그대로 재합성한다 — 그래서 재합성이 훨씬 싸다.
3. **RenderThread는 백그라운드지만 메인과 묶여 있다.** GPU 부하(오버드로, 큰 비트맵 업로드, 무거운 하드웨어 레이어)는 sync 랑데부를 통해 메인 스레드로 역류해 jank를 만든다.
