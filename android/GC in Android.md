# 🧠 GC in Android

---

## 1️⃣ GC 개요

- **GC(Garbage Collection)** 은 더 이상 참조되지 않는 객체를 자동으로 회수하여 메모리를 관리하는 기능.
- Kotlin은 JVM 위에서 실행되므로, 실질적인 메모리 관리는 **JVM 또는 Android ART의 GC 엔진**이 담당한다.
- 주요 목적:
    - 메모리 누수 방지
    - 수동 메모리 해제 부담 감소
    - 힙(Heap) 공간의 효율적 재사용

---

## 2️⃣ 메모리 구조

```markdown
┌──────────────────────────────────────────────────┐
│                      Heap                         │
│  ┌──────────────────────────┐  ┌────────────────┐ │
│  │        Young Gen         │  │     Old Gen    │ │
│  │  ┌───────────────┐       │  │                │ │
│  │  │     Eden      │       │  │                │ │
│  │  └───────────────┘       │  │                │ │
│  │  ┌───────────────┐       │  │                │ │
│  │  │      S0       │       │  │                │ │
│  │  └───────────────┘       │  │                │ │
│  │  ┌───────────────┐       │  │                │ │
│  │  │      S1       │       │  │                │ │
│  │  └───────────────┘       │  │                │ │
│  └──────────────────────────┘  └────────────────┘ │
│              MetaSpace / Class Data               │
└──────────────────────────────────────────────────┘
```

| 영역 | 설명 |
| --- | --- |
| **Eden** | 새로 생성된 객체가 저장되는 공간 |
| **Survivor 0 / 1** | Eden에서 살아남은 객체를 임시로 저장 |
| **Old Generation** | 여러 번 살아남은 객체가 승격(Promotion)되어 저장 |
| **MetaSpace** | 클래스 메타데이터 저장 (Java 8 이상, 이전 PermGen 대체) |

---

## 3️⃣ GC 구분

| 구분 | 대상 영역 | 발생 조건 | 특징 |
| --- | --- | --- | --- |
| **Minor GC** | Young Generation (Eden + Survivor) | Eden이 가득 찼을 때 | 빠르고 빈번, STW 짧음 |
| **Major GC (Full GC)** | Old Generation (또는 전체 Heap) | Old Gen 임계치 도달 시 | 느리고 비용 큼, STW 길음 |
| **Concurrent GC (ART)** | 전체 Heap (병행 처리) | 내부 정책에 따라 자동 수행 | 대부분 병행, STW 최소화 |

**흐름:**

`Eden full` → `Minor GC` → `살아남은 객체` → `Survivor` → `Old Generation (Promotion)`

`Old Gen full` → `Major GC (Full GC)`

---

## 4️⃣ GC 트리거 방식

- **이벤트 기반(Event-driven)** 으로 동작하며, 주기적이지 않음.
- 조건이 충족될 때만 수행:

```kotlin
if (eden.isFull()) triggerMinorGC()

if (oldGenUsage > threshold) triggerMajorGC()
```

- Minor GC: Eden이 가득 차면 즉시 실행.
- Major GC: Old Gen이 임계치를 초과하면 백그라운드로 스케줄되어 실행.
- GC는 한 번 실행 후 결과(성공/실패)를 반환하고 종료 → 무한 루프형 구조가 아님.

## 5️⃣ 메모리 회수 실패 시 (OOM 과정)

### ✅ 정상 흐름

```
Eden full → Minor GC
Survivor full → Old Gen Promotion
Old Gen full → Major GC
```

### ⚠️ 예외 상황

- **Promotion Failure**: Minor GC 중 Old Gen에 승격할 공간이 없음.
    
    → GC 엔진이 **Full GC를 강제 트리거**
    
    → 여전히 공간 부족 → **OutOfMemoryError (OOM)** 발생.
    

### 주요 원인

- Major GC가 아직 실행 전 (스케줄링 지연)
- Old Gen 단편화 (연속 공간 부족)
- Survivor 공간 부족
- 전역 객체 / 참조 누수로 공간 확보 불가

---

## 6️⃣ GC 스레드 구조

- GC는 **별도의 전용 스레드(GC Thread)** 에서 수행된다.
- 앱 스레드(Main/UI, Worker 등)와는 분리되어 백그라운드로 실행.
- 일부 단계에서는 앱 스레드를 잠시 멈추는 **STW(Stop-The-World)** 발생.

| GC 유형 | 스레드 구조 | 특징 |
| --- | --- | --- |
| **Serial GC** | 단일 스레드 | STW 길지만 단순 |
| **Parallel GC** | 다중 GC Worker | 병렬 Mark/Sweep |
| **G1 / CMS / ART** | GC 스레드 + App 스레드 병행 | Concurrent GC로 STW 최소화 |

GC 스레드는 OS 스케줄러에서 **높은 우선순위(High Priority)** 로 실행되어 메모리 확보를 우선시한다.

---

## 7️⃣ Stop-The-World (STW)

### 개념

- GC가 객체 그래프의 **일관성(consistency)** 을 보장하기 위해 모든 앱 스레드를 일시 정지시키는 단계.
    
- 이 구간 동안 UI 및 로직이 모두 멈춘다.

### 동작 단계

1. **Safepoint 진입** – 스레드들이 안전 지점에서 대기
2. **Root Scan** – Stack, Static, JNI 참조 수집
3. **Mark / Sweep / Compact** 수행
4. **스레드 재개 (Resume)**

### STW 시간 체감 기준

| STW 시간 | 사용자 체감 |
| --- | --- |
| < 5 ms | 체감 거의 없음 |
| 10 ~ 20 ms | 프레임 살짝 튐 |
| 50 ~ 100 ms | 눈에 띄는 렉 |
| 100 ms 이상 | 터치 지연, ANR 가능성 |

---

## 8️⃣ 요약

| 항목 | 내용 |
| --- | --- |
| **GC 실행 주체** | 전용 GC Thread (App Thread와 별도) |
| **GC 구분** | Minor / Major(Full) / Concurrent |
| **트리거 방식** | 메모리 임계점(Event-driven) |
| **메모리 회수 실패(OOM)** | Promotion Failure → Full GC 실패 시 발생 |
| **STW** | 루트 스캔, Compaction 시점에 짧게 발생 |
| **체감 영향** | Major GC가 50ms 이상이면 프레임 드랍 유발 |
| **Android ART 특징** | Concurrent Copying GC로 STW 최소화 |

> 💡 정리 문장
> 
> JVM / ART의 GC는 전용 스레드에서 실행되며, Eden이 가득 차면 Minor GC, Old Gen이 임계 도달 시 Major GC가 동작한다.
> 
> GC는 객체 그래프의 안정성을 위해 STW를 수행하고, 메모리 확보에 실패하면 Full GC 후에도 공간 부족 시 OOM이 발생한다.
> 
> Major GC의 긴 STW(수십~수백 ms)는 사용자에게 프레임 드랍으로 체감될 수 있다.
>
