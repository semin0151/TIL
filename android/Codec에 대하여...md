# Android Media + Codec 개념 정리
## 기본 용어
- Codec(코덱): COder/DECoder.
- 인코더: 원본 신호(영상 프레임/오디오 PCM) → 압축 비트스트림(H.264/HEVC/AV1, AAC/Opus 등)
- 디코더: 압축 비트스트림 → 복원 신호(비디오 프레임/오디오 PCM)
- Container(컨테이너): MP4/MKV 등. 트랙/인덱스/타임스탬프(PTS/DTS)/메타데이터를 담는 포장. 내용물은 코덱 비트스트림.
- ES (Elementary Stream): 컨테이너 내부의 순수 비트스트림(video ES / audio ES).
- EOS (End Of Stream): 스트림 종료 플래그.
- PTS/DTS: 표시 시간/디코드 시간. 재생 타이밍 결정에 PTS 사용.

## 재생 파이프라인(개념 흐름)
```text
[MP4] → (Demux) MediaExtractor → video ES ─→ MediaCodec(비디오 디코더) ─→ Surface(표시)
                                audio ES ─→ MediaCodec(오디오 디코더) ─→ PCM ─→ AudioTrack(출력)
                           ◄────────────── A/V Sync: 보통 오디오 시계 기준 ──────────────►
```
- MediaExtractor: 컨테이너에서 video/audio ES + PTS를 분리.
- MediaCodec(Decoder):
  - 비디오: ES → 표시 가능한 프레임(대개 Surface로 직접 렌더)
  - 오디오: ES → PCM(스피커 출력용)
- A/V 동기화: 일반적으로 오디오 타임라인 기준으로 비디오 표시 시점 정렬.

## MediaCodec 동작 모델(큐 기반)
- 입력 큐: 앱이 ES 샘플과 PTS를 디코더에 공급(“넣기”).
- 출력 큐: 디코더가 복원된 데이터를 제공(“꺼내기”).
  - 비디오: 출력 버퍼를 Surface에 표시
  - 오디오: 출력 버퍼는 PCM 데이터
- 상태 전이(디코더)
  - 1) 생성 → 2) configure(포맷/출력 대상) → 3) start(실행 가능)
  - 4) (입·출력 큐 운용) → 5) flush/stop/release

## 표시 경로와 합성(비디오)
- Surface: 디코더가 프레임을 써 넣는 출력 대상 핸들(Producer 엔드).
- SurfaceView vs TextureView
  - SurfaceView: Consumer가 SurfaceFlinger(시스템 컴포지터). 별도 레이어로 바로 합성.
  - TextureView: Consumer가 SurfaceTexture(앱 내부). 텍스처로 View 트리 안에서 합성된 뒤 앱 윈도우가 Flinger로 전달.
- SurfaceFlinger: 여러 Surface(윈도우/오버레이 등)를 최종 화면 프레임으로 합성.

##  인코딩 파이프라인(개념 흐름)
```text
[원본 프레임/PCM] ─→ MediaCodec(Encoder) ─→ video ES / audio ES ─→ MediaMuxer(컨테이너 작성, 예: MP4)
```

- 비디오 인코더 입력: 프레임(보통 Surface 입력 또는 YUV 버퍼)
- 오디오 인코더 입력: PCM
- 출력: 압축된 video/audio ES → 컨테이너로 묶어 배포/저장

## 시간/타이밍 개념
- PTS: 표시할 이상적 시간.
- DTS: 디코딩 순서 기준 시간(재정렬 있는 코덱에서 사용).
- 플레이어는 보통 오디오를 기준 시계로 삼고 비디오 표시를 PTS로 맞춤.

## 요약(TL;DR)
- 컨테이너(MP4)는 포장, 코덱은 압축/복원 방식.
- Extractor가 ES+PTS를 꺼내 MediaCodec에 공급하고, 비디오는 Surface, 오디오는 PCM→AudioTrack으로 나간다.
- SurfaceView는 시스템 합성 직결, TextureView는 앱 내부 합성 후 제출.
- 인코딩은 그 반대 방향: 원본 → Encoder → ES → Muxer(컨테이너).
