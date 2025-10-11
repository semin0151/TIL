## What & Why
- 정의: `TextureView`는 `View` 트리 안에서 GPU 텍스처를 그려주는 뷰로, 내부에 `SurfaceTexture`(버퍼 큐 Consumer) 를 보유합니다. `Producer`(디코더/카메라/GL)는 `Surface`(SurfaceTexture)에 프레임을 써 넣고, `TextureView`는 이를 일반 View처럼 합성합니다.
- 목적: 알파/회전/스케일/라운드 코너/클리핑/트랜스폼 같은 View 변형을 미디어 내용에 직접 적용하면서도, 다른 UI와 자연스럽게 레이어링하기 위함.
- 주요 사용처: 인피드 자동재생(`RecyclerView`), 미니플레이어/썸네일, 라운드 카드형 미디어, WebView 하이브리드(영상은 네이티브, UI/트래킹은 JS) 등.

## Mental Model
- `TextureView`는 자체 `SurfaceTexture(Consumer)`를 통해 프레임을 받아 텍스처로 유지합니다.
- 이후 `TextureView`가 `View` 트리 내부에서 이 텍스처를 다른 `View`와 함께 합성합니다(알파/트랜스폼 적용 가능).
- 최종적으로 앱 윈도우 `Surface`가 시스템의 `SurfaceFlinger`에 전달되어 화면으로 출력됩니다.
  (SurfaceView처럼 별도 레이어로 ‘바로’ 올라가지 않고, 앱 내부 합성 → 윈도우 합성을 거친다는 점이 핵심)

## 핵심 구성 요소
- `TextureView` : 
  - `View` 트리의 노드(레이아웃/가시성/터치/애니메이션 적용 가능).
  - 내부에 `SurfaceTexture`를 보유하고, 텍스처를 `View`처럼 그린다.
- `SurfaceTexture` (Consumer)
  - `BufferQueue`의 소비자. `Producer`가 쓴 프레임을 받아 GL 텍스처로 제공.
  - `Surface`를 만들어 `Producer`에 넘길 수 있음: `Surface(surfaceTexture)`.
- `Surface` (Producer 핸들)
  - 디코더/카메라/GL 같은 `Producer`가 프레임을 쓰는 출력 단.
  - `MediaCodec.configure(..., surface, ...)`, `player.setVideoSurface(surface)`.
- `Producer`
  - `MediaPlayer`/`ExoPlayer`/`MediaCodec`/`Camera`/`GL` 등 → `Surface(SurfaceTexture)`에 직접 렌더.
- `SurfaceFlinger` (시스템 합성자)
  - 앱 윈도우, 상태바 등 여러 `Surface`를 최종 합성.
  - `TextureView`는 앱 윈도우 내에서 먼저 그려진 후, 윈도우 단위로 `Flinger` 합성에 참여.

### 개념 지도
```text
App (Process)
┌──────────────────────────────────────────────────────────┐
│ View Hierarchy (UI Thread)                               │
│  ┌───────────────┐                                       │
│  │  TextureView  │ -- has --> SurfaceTexture (Consumer)  │
│  └───────────────┘                 ▲                     │
│                                     │ (Surface from ST)  │
│                              Surface ◄──────────────── Producer
│                                 ▲             (MediaCodec/Player/Camera/GL)
│                                 │
│   TextureView draws the texture as a normal View (alpha/transform/clip)
└──────────────────────────────────────────────────────────┘
                     │
                     ▼
System (SurfaceFlinger)  ← 앱 윈도우의 결과를 다른 Surface들과 합성
```
## 라이프사이클 & 기본 패턴
```kotlin
class TvHost @JvmOverloads constructor(
    ctx: Context, attrs: AttributeSet? = null
) : FrameLayout(ctx, attrs), TextureView.SurfaceTextureListener {

    private val tv = TextureView(ctx).apply { surfaceTextureListener = this }
    private var surface: Surface? = null

    init { addView(tv, LayoutParams(MATCH_PARENT, MATCH_PARENT)) }

    override fun onSurfaceTextureAvailable(st: SurfaceTexture, w: Int, h: Int) {
        surface = Surface(st)
        // 플레이어/코덱에 Surface 연결
        // exoPlayer.setVideoSurface(surface)  /  codec.configure(..., surface, ...)
        // exoPlayer.prepare(); exoPlayer.playWhenReady = true
    }

    override fun onSurfaceTextureSizeChanged(st: SurfaceTexture, w: Int, h: Int) {
        // 비율/크기 변화 시 transform(matrix) 재계산(CENTER_CROP/FIT 등)
    }

    override fun onSurfaceTextureUpdated(st: SurfaceTexture) {
        // 프레임 도착(HUD/FPS 업데이트 등)
    }

    override fun onSurfaceTextureDestroyed(st: SurfaceTexture): Boolean {
        // 재사용 풀링 시 false 반환 + 풀에 보관, 아니면 true로 파기
        exoPlayer.clearVideoSurface()
        surface?.release(); surface = null
        return true
    }
}
```
## 연결 방식 예시
- `ExoPlayer`: setVideoSurface(Surface(textureView.surfaceTexture))
- `MediaCodec`: configure(format, Surface(st), null, 0)
- `Camera2`: CaptureSession에 Surface(st) 등록

## 렌더링 경로

1. 하드웨어 경로(일반적)
  `Producer` → `Surface(SurfaceTexture)`에 프레임 enqueue → `SurfaceTexture`가 수신 →
  `TextureView가` 텍스처를 View처럼 그려서 앱 윈도우 `Surface`에 출력 → `Flinger` 최종 합성.

2. GL 직접 경로(응용)
  `SurfaceTexture를` GL에서 `updateTexImage()`로 업데이트 → 커스텀 셰이더/매트릭스로 후처리 가능.

## Z-Order & 합성
- `TextureView`는 View 트리 내부 합성이므로 다른 View와 자유롭게 레이어링됩니다.
- 알파/회전/스케일/라운드 코너/클리핑이 미디어 내용에 직접 적용됩니다(이 점이 SurfaceView와 가장 다름).
- 반대로, 최저 레이턴시·최대 성능·일부 DRM 경로에서는 SurfaceView가 유리/요구될 수 있습니다.
