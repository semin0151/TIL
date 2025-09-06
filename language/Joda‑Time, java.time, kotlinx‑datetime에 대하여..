#  Joda‑Time, java.time, kotlinx‑datetime에 대하여..

*Android · Kotlin · 시간/날짜 API*
**개념 대응표, 권장 사용처, 마이그레이션 팁, DST(일광절약) 이슈까지**

---

## 요약

* **Joda-Time**: Java 8 이전 사실상 표준 라이브러리였으나 현재는 **유지보수 모드**. → Java 8+에서는 **java.time(JSR-310)** 권장.
* **java.time**: Joda의 철학(불변, 의미 분리)을 표준화.

  * `Instant` (절대 시점),
  * `LocalDate` (날짜),
  * `LocalTime` (시간),
  * `ZonedDateTime` (시점+타임존),
  * `Duration/Period` (시간량/기간).
* **kotlinx-datetime**: 멀티플랫폼용. 시간량은 `kotlin.time.Duration` DSL이 가장 직관적 (`5.hours + 30.minutes`).
* **도메인 설계**: Instant/LocalDate/LocalTime을 명확히 구분. 저장/전송은 \*\*Instant(epoch)\*\*로 표준화.
* **DST/타임존 이슈**: `LocalDate → 시점` 변환 시 안전 API (`atStartOfDay(zone)`)를 사용할 것.

---

## 용어 & 핵심 타입 지도

| 개념          | Joda-Time   | java.time       | kotlinx-datetime / kotlin.time | 설명                                 |
| ----------- | ----------- | --------------- | ------------------------------ | ---------------------------------- |
| 절대 시점       | `Instant`   | `Instant`       | `Instant`                      | UTC 기준 한 순간(Epoch 초/나노). 타임존 없음    |
| 날짜만         | `LocalDate` | `LocalDate`     | `LocalDate`                    | 연-월-일만. 타임존 없음                     |
| 시간만         | `LocalTime` | `LocalTime`     | `LocalTime`                    | 시-분-초(나노). 타임존 없음                  |
| 날짜+시간(+타임존) | `DateTime`  | `ZonedDateTime` | `LocalDateTime` + `TimeZone`   | 사람이 보는 현지 시각 + 존 의미                |
| 시간 양        | `Duration`  | `Duration`      | `kotlin.time.Duration`         | 초/나노 기반. Kotlin DSL 제공 (`3.hours`) |
| 사람 친화적 기간   | `Period`    | `Period`        | —                              | 년/월/일 단위. 영업일 계산 등에 사용             |

---

## Joda-Time의 위치

* **공식 JSR 표준 아님**, Java 8 이전에는 사실상 표준.
* Java 8 이후 \*\*JSR-310(java.time)\*\*이 공식 표준으로 채택됨.
* 현재 Joda-Time은 **유지보수 모드** (새 기능 없음, 버그/보안 위주).
  👉 새 코드/도메인 설계는 **java.time** 또는 **kotlinx-datetime(KMP)** 권장.

---

## 플랫폼별 권장 사용

### Android

* **minSdk 26+** → `java.time` 바로 사용.
* **minSdk < 26** → *Core Library Desugaring* 활성화 후 `java.time` 백포트 사용.

```groovy
android {
  compileOptions {
    sourceCompatibility JavaVersion.VERSION_17
    targetCompatibility JavaVersion.VERSION_17
    coreLibraryDesugaringEnabled true
  }
}

dependencies {
  // 버전 카탈로그 사용 권장 (예: libs.desugar)
  coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:<version>")
}
```

### Kotlin Multiplatform

* 공용 로직: `kotlinx-datetime`
* 시간량: `kotlin.time.Duration` DSL

---

## 주요 사용 예시

### 1) 절대 시점 ↔ 현지 시각

```kotlin
// java.time
val instant = java.time.Instant.now()
val seoul = java.time.ZoneId.of("Asia/Seoul")
val zdt = java.time.ZonedDateTime.ofInstant(instant, seoul)

// kotlinx-datetime
import kotlinx.datetime.*
val kInstant = Clock.System.now()
val tz = TimeZone.of("Asia/Seoul")
val local = kInstant.toLocalDateTime(tz)
val back: Instant = local.toInstant(tz)
```

### 2) Duration DSL

```kotlin
import kotlin.time.Duration.Companion.hours
import kotlin.time.Duration.Companion.minutes

val work = 5.hours + 30.minutes
if (work >= 8.hours) { /* ... */ }
```

### 3) Joda → java.time 마이그레이션

```kotlin
val joda: org.joda.time.DateTime = org.joda.time.DateTime.now()
val instant = java.time.Instant.ofEpochMilli(joda.millis)
val zone = java.time.ZoneId.of(joda.zone.id)
val zdt = java.time.ZonedDateTime.ofInstant(instant, zone)
```

### 4) java.time ↔ kotlinx-datetime 변환

```kotlin
import kotlinx.datetime.toKotlinInstant
import kotlinx.datetime.toJavaInstant

val jInst: java.time.Instant = java.time.Instant.now()
val kInst = jInst.toKotlinInstant()
val jBack = kInst.toJavaInstant()
```

---

## DST/타임존 이슈 (IllegalInstant 등)

* **문제**: DST로 인해 `LocalDate`의 00:00이 존재하지 않거나 겹칠 수 있음.
* **안전한 방법**:

  * `LocalDate.atStartOfDay(zone)` 사용 (존 규칙을 고려).
  * 필요 시 `ZoneRules`로 오프셋 조회 후 정책적 선택.

---

## 도메인/설계 가이드 (Clean Architecture 관점)

1. **개념 분리**: Instant / LocalDate / LocalTime 명확히 구분.
2. **저장/전송**: 서버·DB·캐시는 Instant(epoch) 통일.
3. **UI 변환**: 표시 직전에 TimeZone/Locale 적용.
4. **Value Class**: 의미 강화.
5. **테스트**: Clock 주입으로 시간 고정.

```kotlin
@JvmInline value class UtcInstant(val value: kotlinx.datetime.Instant)
@JvmInline value class CalendarDate(val value: kotlinx.datetime.LocalDate)
@JvmInline value class ClockTime(val value: kotlinx.datetime.LocalTime)

interface TimeProvider {
  fun now(): kotlinx.datetime.Instant
}

class SystemTimeProvider : TimeProvider {
  override fun now() = kotlinx.datetime.Clock.System.now()
}
```

---

## 체크리스트 & 안티패턴

* ✅ 절대 시점 저장은 **Instant/epoch**
* ✅ 입력/표시는 **LocalDate/LocalTime**
* ✅ DST 경계 시 안전 API 사용
* ✅ Android 하위 호환은 **Desugaring**
* ✅ KMP는 **kotlinx-datetime + kotlin.time.Duration**
* 🚫 절대 시점 저장에 `LocalDateTime` 사용 금지
* 🚫 문자열 파싱/포맷 시 타임존 의미 누락 금지
* 🚫 서버/클라이언트 서로 다른 존 가정 금지

