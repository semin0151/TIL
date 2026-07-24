# Android Activity 실행 과정 (Zygote → onCreate)

---

## 한 줄 요약

앱 아이콘을 탭하면, 부팅 때부터 대기하던 **Zygote가 자신을 `fork()`해 앱 프로세스를 찍어내고**, 그 프로세스의 진입점 `ActivityThread.main()`이 실행된다. 이후 **AMS가 Binder IPC로 Activity 실행 명령을 보내면**, 앱은 이를 `MessageQueue`를 거쳐 메인 스레드로 넘겨 최종적으로 `onCreate()`를 호출한다.

> 핵심 3축: **Zygote(fork로 빠른 프로세스 생성)** → **AMS(생명주기 관제탑)** → **ActivityThread(메인 스레드에서 실제 처리)**

---

## 개요 — 주요 클래스 정리

### OS / 커널 레벨

| 개념 | 설명 |
|---|---|
| **Linux Kernel** | OS의 핵심 부품(커널). Android, Ubuntu 모두 이 커널을 기반으로 한다. |
| **init (PID 1)** | 커널이 직접 실행하는 첫 번째 프로세스. `init.rc` 스크립트로 Zygote를 실행한다. |

### 프로세스 레벨

| 클래스/프로세스 | 위치 | 설명 |
|---|---|---|
| **Zygote** | Zygote 프로세스 | 부팅 시 프레임워크 클래스를 미리 메모리에 로드한 뒤 `fork()` 대기. 앱 실행 요청마다 자신을 복제해 앱 프로세스를 찍어내는 공장. |
| **SystemServer** | SystemServer 프로세스 | Zygote의 첫 번째 fork. AMS/WMS/PMS 등 수십 개의 시스템 서비스를 실행하는 컨테이너. 죽으면 Android 전체가 재부팅. |

### 시스템 서비스 (SystemServer 프로세스 내부, 각 1개씩만 존재)

> **주의**: 여기서의 "Service"는 Android 컴포넌트(`android.app.Service`)와 **완전히 다른 개념**.
> 일반 Java 객체/스레드로 구현되며 `ServiceManager`에 Binder로 등록된다.
> 개발자가 사용하는 `android.app.Service`는 AMS가 관리하는 앱 컴포넌트.

| 클래스 | 약자 | 설명 |
|---|---|---|
| **ActivityManagerService** | AMS | 앱/Activity 생명주기 관제탑. 앱 실행 요청 처리, Activity 스택 관리, 프로세스 관리, ANR 감지. |
| **ActivityTaskManagerService** | ATMS | Android 10부터 AMS에서 분리. Activity 태스크/스택 전담 관리. |
| **WindowManagerService** | WMS | 화면에 그려지는 Window(창) 관리. |
| **PackageManagerService** | PMS | APK 설치/삭제, 패키지 정보·권한·컴포넌트 조회, Intent 해석. AMS가 Activity 실행 전 패키지 정보를 요청하는 대상. |
| **InputManagerService** | IMS | 터치, 키 입력 이벤트 처리 및 분배. |
| **PowerManagerService** | - | 화면 켜기/끄기, 절전 모드, WakeLock 관리. |
| **NotificationManagerService** | - | 알림 표시, 알림 채널 관리. |
| **LocationManagerService** | - | GPS/네트워크 위치 정보 제공. |
| **ConnectivityService** | - | 네트워크 연결 상태 관리 (Wi-Fi, 셀룰러 등). |
| **AudioService** | - | 볼륨, 오디오 포커스, 오디오 경로 관리. |
| **SensorService** | - | 가속도계, 자이로 등 센서 데이터 제공. |
| **BatteryService** | - | 배터리 상태 모니터링. |
| **AppOpsService** | - | 앱 권한 사용 이력 추적 및 제어. |

### 앱 프로세스 내부

| 클래스 | 스레드 | 설명 |
|---|---|---|
| **ActivityThread** | 메인 스레드 | 앱 프로세스의 진입점(`main()`). 메인 스레드 전체를 관리하며 Activity 생명주기 실제 처리를 담당. 앱 하나 = 프로세스 하나 = ActivityThread 하나. |
| **ApplicationThread** | Binder 스레드 | `ActivityThread`의 내부 클래스. AMS의 Binder IPC 명령을 수신하는 창구. 수신 후 `H`에 메시지를 던져 메인 스레드로 전달. |
| **H (Handler)** | 메인 스레드 | `ActivityThread` 내부 Handler. Binder 스레드 → 메인 스레드 간 메시지 전달 담당. |
| **Looper** | 메인 스레드 | MessageQueue를 무한 순환하며 메시지를 꺼내 처리. `Looper.loop()`이 메인 스레드를 살아있게 유지. |
| **MessageQueue** | 메인 스레드 | 메시지 적재 공간. `H.sendMessage()`로 넣고 `Looper`가 꺼냄. |
| **Instrumentation** | 메인 스레드 | Activity 생성 중간 레이어. 테스트 환경에서 Activity 생성을 가로채거나 교체 가능. |
| **ActivityManager** | 앱 | 개발자가 사용하는 AMS 공개 API 래퍼. 내부적으로 Binder Proxy를 통해 AMS를 원격 호출. |

### Binder IPC

| 개념 | 설명 |
|---|---|
| **Binder** | Android 프로세스 간 통신(IPC) 메커니즘. 앱 ↔ AMS처럼 서로 다른 프로세스 간 메서드 호출을 가능하게 함. |
| **IActivityManager** | AMS의 Binder 통신 계약 인터페이스(AIDL). `ActivityManager`(클라이언트)와 `AMS`(서버) 사이의 계약. |

### 빌드 산출물

| 파일 | 설명 |
|---|---|
| `.kt` / `.java` | 소스 코드 |
| `.class` | JVM 바이트코드 (컴파일 결과) |
| `.dex` | Android 전용 바이트코드. d8/R8이 `.class`를 변환. ART가 실행. |

---

## 1. 부팅 과정

```
커널 부팅
  └── init (PID 1)
        └── Zygote 시작
              ├── JVM(ART) 초기화
              ├── Android 프레임워크 클래스 전부 메모리에 로드
              └── fork() 대기 상태 진입
```

## 2. Zygote의 첫 번째 fork — SystemServer

```
Zygote
  └── fork()  ← 부팅 시 딱 한 번 (유일)
        └── SystemServer 프로세스 생성
              └── SystemServer.main()
                    ├── AMS / ATMS
                    ├── WMS
                    ├── PMS
                    ├── InputManagerService
                    ├── PowerManagerService
                    ├── NotificationManagerService
                    └── ... 수십 개의 시스템 서비스 시작
```

> - SystemServer 내부 서비스들은 독립 프로세스가 아닌 **같은 프로세스 안의 스레드/객체**.
> - 하나가 unhandled exception으로 죽으면 → 프로세스 전체 크래시 → Android 재부팅.
> - AMS, WMS, PMS 등은 시스템 전체에 하나씩만 존재 (싱글톤).

### SystemServer 안정성 보장 장치

| 방어 수단 | 설명 |
|---|---|
| **Watchdog** | SystemServer 내부 Watchdog 스레드가 각 서비스 응답을 주기적으로 체크. 무응답 시 SystemServer 강제 종료 후 재부팅. |
| **try-catch 철저히** | 각 서비스 내부에서 예외를 최대한 잡아 프로세스까지 전파되지 않도록 방어. |
| **SELinux** | 외부에서 서비스를 임의로 종료하지 못하도록 격리. |

> Watchdog가 재부팅을 선택하는 이유: 좀비 상태의 시스템보다 **빠른 재부팅이 사용자 경험상 낫다**는 설계 철학.

## 3. 사용자가 앱 아이콘을 탭 — AMS 개입

```
사용자 앱 아이콘 탭
  └── Launcher → AMS에 앱 실행 요청 (Binder IPC)
        └── AMS → PMS에 패키지 정보 요청
              └── AMS: 해당 앱 프로세스가 존재하는지 확인
                    └── 없으면 → Zygote에 fork 요청
```

## 4. Zygote의 두 번째 fork — 앱 프로세스 생성

```
Zygote
  └── fork()  ← 앱 실행 요청마다
        └── 새 앱 프로세스 생성
              └── ActivityThread.main()  ← 진입점 (Java main()과 동일 개념)
```

### fork()란?

- 현재 프로세스(Zygote)를 통째로 복사해 새 프로세스를 만드는 Linux 시스템 콜
- 반환값으로 부모/자식 구분

```c
pid_t pid = fork();

if (pid == 0) {
    // 자식 프로세스 (앱 프로세스) → ActivityThread.main() 실행
} else {
    // 부모 프로세스 (Zygote) → 계속 대기
}
```

### Copy-on-Write (CoW)

물리 메모리를 즉시 복사하지 않고, **쓰기가 발생하는 시점에만 해당 페이지를 복사**.

```
fork() 직후
  ├── Zygote    가상주소 0x1000 → 물리주소 0xABCD  (공유)
  └── 앱프로세스 가상주소 0x1000 → 물리주소 0xABCD  (공유)

쓰기 발생 시
  ├── Zygote    가상주소 0x1000 → 물리주소 0xABCD  (원본 유지)
  └── 앱프로세스 가상주소 0x1000 → 물리주소 0xEFGH  (새로 할당)
```

> 앱 프로세스 입장에서는 가상 주소가 바뀌지 않아 모름.
> OS가 뒤에서 조용히 물리 주소(MMU)만 바꿔치기.

이 덕분에 앱 시작이 빠름:

| 방식 | 시간 |
|---|---|
| JVM 새로 띄우고 클래스 로드 | 수초 |
| Zygote fork() | 수십ms |

## 5. 앱 프로세스 초기화 — ActivityThread.main()

```java
// ActivityThread.java
public static void main(String[] args) {  // 일반 Java main()과 동일 개념
    Looper.prepareMainLooper();   // MessageQueue 생성
    ActivityThread thread = new ActivityThread();
    thread.attach();              // AMS에 자신을 Binder로 등록
    Looper.loop();                // 무한 대기 시작
}
```

## 6. AMS → Activity 실행 명령

```
AMS (SystemServer 프로세스)
  └── Binder IPC
        └── ApplicationThread (앱 프로세스, Binder 스레드)  ← 명령 수신 창구
              └── H (Handler).sendMessage(LAUNCH_ACTIVITY)
                    └── MessageQueue 적재
```

### 왜 MessageQueue를 거치나?

- `ApplicationThread`는 **Binder 스레드**에서 실행
- `onCreate()`는 반드시 **메인 스레드**에서 실행되어야 함
- 직접 호출 불가 → MessageQueue를 통해 메인 스레드로 전달

```
Binder 스레드 (ApplicationThread)
  └── H.sendMessage(LAUNCH_ACTIVITY)  ← 메인 스레드로 전달

메인 스레드 (Looper.loop())
  └── MessageQueue에서 꺼냄
        └── onCreate() 실행
```

## 7. onCreate() 호출

```
Looper.loop() → MessageQueue에서 LAUNCH_ACTIVITY 꺼냄
  └── ActivityThread.handleLaunchActivity()
        └── ActivityThread.performLaunchActivity()
              └── Instrumentation.callActivityOnCreate()
                    └── Activity.performCreate()
                          └── Activity.onCreate()  ← 우리가 오버라이드하는 곳
```

### super.onCreate() 강제 검증

```java
// ActivityThread.java
if (!activity.mCalled) {
    throw new SuperNotCalledException(
        "Activity did not call through to super.onCreate()"
    );
}
```

`Activity.onCreate()` 내부에서 `mCalled = true`를 세팅.
`ActivityThread`가 호출 직후 이 플래그를 확인 → super 미호출 시 런타임 크래시(`SuperNotCalledException`).

> `@CallSuper` 어노테이션은 lint 경고 수준이지만,
> Activity의 경우 프레임워크가 `mCalled` 플래그로 직접 강제 검증한다.

---

## 전체 흐름 요약

```
커널 부팅
  └── init (PID 1)
        └── Zygote (프레임워크 클래스 로드 후 대기)
              ├── fork() → SystemServer
              │             ├── AMS  (Activity 생명주기 관제탑)
              │             ├── WMS  (Window 관리)
              │             └── PMS  (패키지 정보 관리)
              │
              └── fork() → 앱 프로세스  ← 앱 실행 요청마다
                    └── ActivityThread.main()  ← 진입점
                          ├── Looper.prepareMainLooper()
                          ├── attach() → AMS에 Binder 등록
                          └── Looper.loop() 대기
                                ↑
                          AMS → Binder IPC
                                └── ApplicationThread (Binder 스레드)
                                      └── H.sendMessage(LAUNCH_ACTIVITY)
                                            └── MessageQueue
                                                  └── Looper.loop() 꺼냄
                                                        └── performLaunchActivity()
                                                              └── Instrumentation
                                                                    └── onCreate()
```

---

## Android OS 계층 구조

```
OneUI / MIUI / ColorOS  ← 제조사 커스터마이징 레이어 (OS 아님)
  └── Android OS
        ├── Android Framework (Java/Kotlin)
        ├── Android Runtime (ART)  ← .dex 실행
        ├── Native Libraries (C/C++)
        └── Linux Kernel  ← 실제 OS 커널 (프로세스/메모리/드라이버 관리)
```

| OS | 커널 | 비고 |
|---|---|---|
| Android | Linux Kernel | 오픈소스 |
| Ubuntu / Debian | Linux Kernel | 오픈소스 |
| macOS / iOS | XNU Kernel | Mach + BSD 기반, Apple 확장 |
| Windows | NT Kernel | Microsoft 완전 독자 개발 |

> "Linux"는 커널이지 OS가 아님. Ubuntu처럼 커널 + GNU 도구 묶음을 Linux OS(GNU/Linux)라 부르는 것은 관용적 표현.

---

## 실무적으로 기억할 3문장

1. **앱 프로세스는 매번 새로 만드는 게 아니라 Zygote를 `fork()`해서 복제**한다. 프레임워크 클래스가 이미 로드된 상태를 CoW로 공유하기 때문에 앱 시작이 수십 ms로 빠르다.
2. **`onCreate()`는 반드시 메인 스레드에서 실행**된다. AMS의 명령은 Binder 스레드(`ApplicationThread`)로 들어오지만, `H` 핸들러가 `MessageQueue`를 통해 메인 스레드로 넘기기 때문이다.
3. **`super.onCreate()`를 빼먹으면 lint 경고가 아니라 런타임 크래시**다. 프레임워크가 `mCalled` 플래그로 직접 검증해 `SuperNotCalledException`을 던진다.
