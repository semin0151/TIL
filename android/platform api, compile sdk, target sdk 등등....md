# Android Platform API / compileSdk / targetSdk 정리

## 0. 한 줄 요약
- **Platform API (Device OS / API Level)**: 앱이 **실제로 실행되는 플랫폼**(디바이스 OS)의 API 집합
- **compileSdk**: 앱을 **어떤 API 정의서(헤더)** 기준으로 컴파일할지(코드에서 무엇을 호출할 수 있는지)
- **minSdk**: 앱이 **설치·실행 가능한 최소 API Level**(이보다 낮은 OS에서는 설치 자체가 차단됨)
- **targetSdk**: 앱에 **어느 버전의 동작/보안 모델(behavior change)** 을 강제 적용할지(정책 + 런타임 동작)

---

## 1. Platform API (Device OS / API Level)
### 의미
- 디바이스에 설치된 **Android OS 버전**이 제공하는 API 집합(Framework + System Services 포함)
- 앱은 항상 **디바이스 OS 위에서 실행**됨

### 핵심 포인트
- 디바이스가 API 36이면 앱은 **항상 플랫폼 36 위에서 실행**
- 디바이스가 API 34이면 앱은 **플랫폼 34 위에서만 실행**(OS보다 높은 플랫폼을 "바라볼" 수 없음)

### 영향 범위
- 실제 런타임에서 "존재하는 클래스/메서드/동작"의 상한선
- OS 내부 구현 변화(보안/성능/제약 포함)

---

## 2. minSdk (간단 정리)
- 앱이 지원하는 **최소 API Level**
- 이 값보다 낮은 OS의 디바이스에서는 **Play Store에서 설치 자체가 차단**됨
- compileSdk/targetSdk와 달리 "어떤 동작이 적용되느냐"와는 무관하고, 단순히 **설치 가능 범위의 하한선**

---

## 3. compileSdk
### 의미
- 앱을 빌드할 때 **어떤 Android SDK(API 정의서)** 로 컴파일할지 결정
- IDE 자동완성/컴파일 가능 여부/Lint 경고가 이 값 기준으로 결정

### 핵심 포인트
- compileSdk=34면 **API 34까지**의 타입/메서드만 코드에서 직접 사용 가능
- compileSdk를 올려도 **런타임 동작은 바뀌지 않음**
- 실행은 항상 **디바이스 OS** 기준

### 예시
- compileSdk=36이면 IDE에서 API 36 메서드가 보이고 컴파일도 됨
- 하지만 디바이스 OS가 34면, 분기 없이 API 36 메서드를 호출하면 런타임 에러 가능

---

## 4. targetSdk
### 의미
- OS가 앱을 실행할 때 "이 앱은 **어느 버전의 Android 동작 모델을 이해한다**"고 간주할 기준
- 즉, **보안 정책 + 런타임 behavior change** 적용 기준

### Android의 Behavior Change는 두 종류로 나뉜다

Android는 새 OS 버전을 출시할 때마다 behavior change 목록을 공개하는데, 항상 **두 카테고리**로 나뉜다:

#### 1) 모든 앱에 적용되는 변경 (All apps)
- 해당 OS 버전에서 실행되는 **모든 앱에 무조건 적용**
- targetSdk 값과 **무관** — targetSdk를 낮게 유지해도 회피 불가
- 주로 OS 내부 구현 변경, 보안 패치, 플랫폼 수준의 동작 변경
- 예: Android 12에서 도입된 Notification Trampoline 제한 (포그라운드가 아닌 상태에서 알림 → Activity 직접 실행 차단)

#### 2) 해당 버전을 타게팅하는 앱에만 적용되는 변경 (Apps targeting Android XX)
- targetSdk가 **해당 버전 이상인 앱에만 적용**
- targetSdk가 낮으면 OS가 **호환 모드(레거시 경로)** 로 실행하여 이전 동작을 유지해줌
- targetSdk를 올리면서 대응하지 않으면 **크래시 또는 비정상 동작** 발생
- 예: 런타임 권한(M), 백그라운드 제한(O), Scoped Storage(Q), exported 필수(S), 알림 권한(T)

### targetSdk 업데이트의 전체 흐름

```
targetSdk를 올린다
  → OS가 "이 앱은 새 규칙을 안다"고 간주
  → "타게팅하는 앱" 카테고리의 behavior change가 강제 적용됨
  → 대응 안 했으면 크래시/비정상 동작
  → 대응해야 정상 동작
```

```
targetSdk를 안 올린다
  → OS가 "이 앱은 아직 새 규칙을 모른다"고 간주
  → "타게팅하는 앱" 카테고리는 적용하지 않고 호환 모드로 실행
  → 하지만 "모든 앱" 카테고리는 여전히 적용됨
  → 그리고 Google Play 정책으로 targetSdk 최소 요구 버전이 매년 올라가므로, 영원히 버틸 수 없음
```

### OS와의 관계(중요)
- targetSdk가 OS보다 높아도, **OS가 모르는 버전의 정책/behavior는 적용될 수 없음**
  - 예: 디바이스 OS=34, targetSdk=36 → OS 34는 36 behavior change를 모르므로 36 전용 강제 변경은 적용 불가
- 반대로 OS가 충분히 최신이면(예: OS=36, targetSdk=36), 36 behavior change가 적용됨

---

## 5. 런타임에서 실제로 "무엇이 기준인지" 정리

### A) "이 API가 존재하냐?" (런타임 안전성)
- 런타임 존재 여부는 **디바이스 OS(Platform API)** 기준
- 코드에서 안전하게 쓰려면 보통:
  - compileSdk는 해당 API 이상
  - 런타임은 `Build.VERSION.SDK_INT`로 분기

### B) "이 동작/제약이 강제 적용되냐?"
- 주로 **targetSdk** 기준 (단, OS가 그 버전의 behavior를 알고 있어야 함)

### C) "IDE에서 보이냐/컴파일 되냐?"
- **compileSdk** 기준

---

## 6. 예시 시나리오

### 시나리오 1
- 디바이스 OS = API 36
- targetSdk = 34
- compileSdk = 34

결과:
- 실행 플랫폼은 36
- "모든 앱" 변경은 OS 36 기준으로 적용됨
- "타게팅하는 앱" 변경은 34 기준까지만 적용 (36의 타게팅 변경은 호환 모드로 회피)
- 코드에서는 API 34까지만 직접 사용 가능

### 시나리오 2
- 디바이스 OS = API 34
- targetSdk = 36
- compileSdk = 36

결과:
- 실행 플랫폼은 34
- OS 34는 35/36 전용 강제 변경을 알지 못하므로 적용 불가
- 코드에서 API 36 호출은 가능하게 컴파일되지만,
  런타임에서 `SDK_INT` 분기 없이 호출하면 크래시 가능

---

## 7. 최종 요약 표

| 항목 | 기준/정의 | 결정하는 것 |
|------|-----------|-------------|
| Platform API (Device OS API Level) | 디바이스에 설치된 OS | 실제 실행 플랫폼 / 런타임 API 존재 여부 / OS 내부 구현 |
| minSdk | 앱이 지원하는 최소 API Level | 설치 가능 디바이스 범위의 하한선 |
| compileSdk | 빌드 시 사용하는 SDK 정의서 버전 | IDE/컴파일 가능 API 범위(자동완성, 컴파일 에러, Lint) |
| targetSdk | 앱이 이해한다고 "선언"하는 동작 모델 버전 | 보안 정책 + behavior change 강제 적용 기준 (호환성 모드 분기점) |

---

## 8. 실무적으로 기억할 3문장
1) 앱은 항상 **디바이스 OS(Platform API)** 위에서 실행된다.
2) `compileSdk`는 "코드에서 무엇을 호출할 수 있나"를 정한다(컴파일 타임).
3) `targetSdk`는 "OS가 어떤 규칙/동작 변경을 강제할까"를 정한다(런타임).
