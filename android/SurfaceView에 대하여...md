## What & Why
- 정의: `SurfaceView`는 `View` 트리 안에 존재하지만 실제 픽셀 렌더링은 별도의 `Surface`(버퍼 큐)에 그리도록 해주는 호스트(브리지) 뷰.
- 목적: UI 스레드의 `onDraw()` 경로와 분리된 고성능 렌더링(하드웨어 디코더/GL)을 가능하게 함.
- 주요 사용처: 동영상 플레이어(전체화면), 카메라 프리뷰, 고프레임 실시간 그래픽.

## Mental Model (구멍 비유)
- `SurfaceView`는 레이아웃(위치/크기)만 담당하고, 실제 프레임은 별도 `Surface` 레이어에 그려진다.
- 합성(Composition)은 `SurfaceFlinger`가 처리.
- 실무적으로 **“앱 윈도우에 구멍을 내고, 그 뒤/앞의 별도 `Surface`에 그림”**으로 이해하면 정확함.
- Android 8.0+에서 내부 합성 방식이 개선되었지만, 별도 합성 레이어라는 본질은 동일.

## 핵심 구성 요소
- `SurfaceView`:
  - `View` 트리의 노드(위치/크기/가시성만).
  - `View` 트리에 존재하지만 그 자체가 픽셀을 그리지 않는 특수 `View`.
  - 화면에 “영역만 잡고” 실제 픽셀은 별도 **Surface**에 그려지도록 하는 호스트(브리지).
- `SurfaceHolder`:
  - `Surface` 접근/수명 관리 인터페이스. (surfaceCreated/Changed/Destroyed)
  - `SurfaceView`가 소유한 `Surface`에 대한 액세스/수명 관리 인터페이스.
- `Surface`:
  - `Surface`는 디코더·GL·카메라 같은 `Producer`가 프레임을 써 넣는 `Producer` 측 출력 핸들이다.
  - `Producer`가 `Surface`에 큐잉한 프레임은 `BufferQueue`의 `Consumer`(=`SurfaceFlinger`) 가 가져가 VSync에 맞춰 화면으로 합성한다.
- `Producer`:
  - `MediaPlayer`, `ExoPlayer`, `MediaCodec`, `Camera` 등 → `Surface`에 직접 렌더.
- `SurfaceFlinger` (시스템 합성자)
  - Android 시스템 프로세스의 합성(Compositor).
  - 앱·시스템이 제출한 여러 Surface(윈도우, SurfaceView, 상태바, 네비바 등)를 하나의 화면 프레임으로 합성해 디스플레이에 출력.

### 개념 지도
```text
App (Process)
┌──────────────────────────────────────────────────────────┐
│ View Hierarchy (UI Thread)                               │
│  ┌─────────────┐                                         │
│  │ SurfaceView │  ---(노출/관리)-->  SurfaceHolder       │
│  └─────────────┘                         │               │
│                                          ▼               │
│                              (handle to) Surface  <---  Producer
│                                          ▲         (MediaCodec, MediaPlayer,
│                                          │          ExoPlayer, Camera, GL 등)
└──────────────────────────────────────────────────────────┘
                     │
                     ▼
System (SurfaceFlinger)  ← 여러 Surface들을 합성(Composite)하여 화면(FrameBuffer)에 출력
```

## 라이프사이클 & 기본 패턴
```kotlin
class MySurfaceHost(context: Context) : FrameLayout(context), SurfaceHolder.Callback {
    private val sv = SurfaceView(context)
    private var surface: Surface? = null

    init {
        addView(sv, LayoutParams(MATCH_PARENT, MATCH_PARENT))
        sv.holder.addCallback(this)
    }

    override fun surfaceCreated(h: SurfaceHolder) {
        surface = h.surface
        // 플레이어/코덱에 Surface 연결
        // mediaPlayer.setSurface(surface) 또는 exoPlayer.setVideoSurface(surface)
    }

    override fun surfaceChanged(h: SurfaceHolder, format: Int, w: Int, hgt: Int) {
        // 비율/크기 조정, setFixedSize 필요 시 처리
    }

    override fun surfaceDestroyed(h: SurfaceHolder) {
        // 플레이어에서 Surface 해제 & 정리
        // mediaPlayer.clear... / codec.stop() etc.
        surface = null
    }
}
```

### 연결 방식 예시
- `MediaPlayer`: `setSurface(surface)` 또는 `setDisplay(holder)`
- `ExoPlayer`: `setVideoSurface(surface)` / `setVideoSurfaceHolder(holder)`
- `MediaCodec`: `configure(format, /*surface*/ surface, null, 0)`

## 렌더링 경로 (두 가지)

1. Canvas 경로
   
`SurfaceHolder.lockCanvas()` → 그리기 → `unlockCanvasAndPost()`
  - CPU 라스터 기반. 간단한 데모/도형 등.

2. 하드웨어 경로
   
디코더/GL이 Surface로 직접 출력
  - MediaCodec 디코더, MediaPlayer/ExoPlayer, Camera 프리뷰.
  - 고성능/저지연의 핵심.

## Z-Order & 합성
- `SurfaceView`는 일반 `View`와 동일한 레이어에서 그리지 않음(별도 Surface 레이어).
- UI를 영상 위에 겹쳐 보이게 하려면 같은 윈도우의 오버레이 View를 FrameLayout 상단에 놓는다.
- 일부 제어:
  - `setZOrderOnTop(true)` → Surface를 맨 위 합성(특수 상황)
  - `setZOrderMediaOverlay(true)` → 미디어 오버레이 용도

| ⚠️ 투명/변형 한계: alpha/rotate/scale 같은 View 변형이 영상 내용에 직접 적용되지 않음. 이런 트랜스폼이 필요하면 TextureView 사용 권장.

