# 템플릿 메서드 패턴 (Template Method Pattern)

## 정의
> 알고리즘의 **전체 흐름(뼈대)** 은 상위 클래스에서 정의하고,  
> **일부 단계의 구현** 은 하위 클래스에 위임하는 패턴

핵심은  
**“순서는 고정하고, 구현만 확장 가능하게 만든다”** 이다.

## 설계 철학 요약
템플릿 메서드 패턴의 설계는 코드를 예쁘게 만드는 일이 아니라  
**변경에 안전한 구조를 만드는 일**이다.

### 1. 변경은 전제다
- 요구사항은 반드시 바뀐다
- 설계의 목적은 변경을 막는 것이 아니라  
  **변경 지점을 명확히 분리하는 것**

### 2. 중요한 것은 덜 중요한 것에 의존하지 않는다
- UI는 바뀌어도
- 비즈니스 로직은 흔들리지 않도록
- 의존성은 항상 안쪽으로 향하게 설계한다

### 3. 흐름과 구현을 분리한다
- 전체 흐름과 정책은 상위에서 통제하고
- 세부 구현은 하위로 위임한다
- 순서는 고정하고, 내용만 바뀌게 만든다

### 4. 상속은 재사용이 아니라 제약이다
- 상속은 자유로운 확장이 아니라
- **허용된 방식으로만 확장하게 만드는 수단**
- 알고리즘 구조를 고정해야 할 때만 사용한다

### 5. 책임은 작고 명확하게 나눈다
- 한 클래스는 하나의 이유로만 변경되도록
- 책임이 섞이면 수정 범위와 리스크가 커진다

### 6. 암묵적인 규칙보다 명시적인 구조가 낫다
- “알아서 이 순서로 호출해야 한다”는 규칙 ❌
- 코드만 읽어도 흐름이 보이는 구조 ⭕

## 구조적 특징
- 상위 클래스
  - 알고리즘 전체 흐름을 가진 `templateMethod`
  - 흐름을 구성하는 단계 메서드들
- 하위 클래스
  - 일부 단계의 구체 구현만 담당

일반적으로:
- `templateMethod`는 `final`
- 확장 지점은 `abstract` 또는 `open` 메서드

## Android에서의 대표 사례

- Activity Lifecycle (onCreate → onStart → onResume)
- RecyclerView.Adapter의 생명주기 메서드

👉 Android 프레임워크 자체가 템플릿 메서드 패턴의 집합에 가깝다.

## 간단한 예제 (Kotlin)
### 추상 클래스
```
abstract class DataLoader {

    fun load() { // template method
        readData()
        processData()
        saveData()
    }

    protected abstract fun readData()

    protected open fun processData() {
        println("기본 데이터 처리")
    }

    private fun saveData() {
        println("데이터 저장")
    }
}
```

### 하위 클래스
```
class NetworkDataLoader : DataLoader() {
    override fun readData() {
        println("네트워크에서 데이터 읽기")
    }
}

class LocalDataLoader : DataLoader() {
    override fun readData() {
        println("로컬 파일에서 데이터 읽기")
    }

    override fun processData() {
        println("로컬 데이터 전용 처리")
    }
}
```
### 사용
```
val loader: DataLoader = NetworkDataLoader()
loader.load()
```

👉 load()의 순서는 절대 바뀌지 않음
