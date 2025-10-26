# 🧠 Android ART 메모리 구조 (정리본)

## 1️⃣ 개요

Android의 ART(Android Runtime)는 JVM과 유사한 구조를 가지며,

앱 실행 시 다음 3가지 주요 메모리 영역으로 나뉩니다.

| 구분 | 주요 역할 | 예시 | 특징 |
| --- | --- | --- | --- |
| **Method Area** | 클래스 정의, static 변수, object 저장 | `class`, `object`, `companion object` | 앱 실행 시 로드, 코드/메타데이터 저장 |
| **Heap** | 동적으로 생성되는 객체 저장 | `Bitmap`, `View`, `List`, `String` | 런타임 중 크기 변동 가능 |
| **Stack** | 함수 호출, 지역 변수 저장 | `val x = 10`, `add(a, b)` 호출 | 함수 단위로 push/pop, 크기 고정 |

---

## 2️⃣ 각 영역 상세

### 📘 Method Area

- **역할:** 클래스 로드 시 메모리에 저장되는 코드/메타데이터 공간
- **포함 내용:**
    - 클래스 이름, 메서드, 필드, 상수풀(Constant Pool)
    - `static` 변수, `object`, `companion object`
- **특징:**
    - 앱 실행 시 로드되어 종료 시까지 유지됨
    - 클래스 단위로 관리되며 코드 정의 및 상수 정보 저장
- **예시:**
    
    ```kotlin
    class User(val name: String)
    val user = User("Semin") // user는 Heap, User 클래스 정보는 Method Area
    ```
    

### 📗 Heap

- **역할:** 런타임 중 생성되는 객체, 배열, 문자열 등이 저장되는 영역
- **특징:**
    - 앱 내 대부분의 참조 타입(Bitmap, View, List 등)이 저장됨
    - 런타임 중 크기가 가변적으로 확장됨
    - GC(Garbage Collector)의 관리 대상
- **예시:**

```kotlin
val bitmap = Bitmap.createBitmap(100, 100, Bitmap.Config.ARGB_8888)
val list = mutableListOf<String>()
```

### 📙 Stack

- **역할:** 함수 호출 시 생성되는 스택 프레임(Stack Frame)에 지역 변수, 매개변수, 리턴 주소가 저장되는 영역
- **특징:**
    - 함수가 호출될 때마다 프레임이 생성되고, 종료 시 자동 해제됨
    - 크기가 고정되어 있으며 초과 시 StackOverflowError 발생
    - 접근 속도가 빠르고 시스템에 의해 자동 관리됨
- **예시:**

```kotlin
fun add(a: Int, b: Int): Int {
	val sum = a + b // a, b, sum → Stack
	return sum
}
```

## ✅ 요약

| 구분 | 저장 대상 | 수명 | 크기 | 관리 주체 |
| --- | --- | --- | --- | --- |
| **Method Area** | 클래스 정보, static/object | 앱 실행~종료 | 고정 | 시스템 |
| **Heap** | 객체 인스턴스, 배열 | 런타임 | 가변 | ART (GC) |
| **Stack** | 지역 변수, 함수 호출 정보 | 함수 실행 시 | 고정 | 시스템 자동 관리 |

## 👉 핵심 포인트 정리

- **Method Area:** 클래스 코드 및 정의가 저장되는 영역 → 상단 고정
- **Heap:** 객체가 생성되는 동적 메모리 영역 → 중간
- **Stack:** 함수 호출 시마다 생기는 제한된 메모리 영역 → 하단
