---
author: 위영종
pubDatetime: 2026-08-14T22:00:00+09:00
title: "JPA 매핑을 orm.xml로 분리했는데, 왜 @Entity는 못 지울까"
description: "Kotlin에서 @Entity는 매핑 메타데이터인 동시에 컴파일러 스위치다. orm.xml로 옮길 수 있는 것과 없는 것을 세 개의 벽으로 나눠 정리한다."
featured: false
draft: false
tags: ["kotlin", "jpa", "hibernate", "architecture"]
---

결론부터: Kotlin에서는 못 지운다. 벽이 셋이고, 그중 하나는 Kotlin에만 있다.

DDD나 헥사고날 아키텍처를 하다 보면 이런 규칙을 세우게 된다.

> 도메인 엔티티에 프레임워크 어노테이션을 덕지덕지 붙이지 말자.
> 테이블명, 컬럼명, 인덱스 같은 상세 매핑은 `orm.xml`로 빼자.

JPA는 이걸 정식으로 지원한다. `META-INF/orm.xml`에 매핑을 선언하면 어노테이션 없이도 엔티티를 매핑할 수 있고, 같은 속성이 양쪽에 있으면 **XML이 이긴다**. `metadata-complete="true"`를 주면 아예 어노테이션을 전부 무시시킬 수도 있다.

그래서 이렇게 시작한다.

```kotlin
// Before
@Entity
@Table(name = "orders", indexes = [Index(name = "idx_user", columnList = "user_id")])
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,
    @Column(name = "user_id", nullable = false)
    val userId: Long,
)
```

```kotlin
// After — 상세 매핑은 XML로
@Entity
class Order(
    @Id val id: Long? = null,
    val userId: Long,
)
```

여기까지는 잘 된다. 그런데 **끝까지 밀어붙이면 어느 지점에서 멈춘다.** `@Entity`와 `@Id`마저 지우려는 순간부터 설명하기 어려운 실패들이 나온다. 이 글은 그 벽이 정확히 어디에 있고, 왜 Kotlin에서 더 두꺼운지에 대한 기록이다.

## 목차

---

## 벽 1 — 컴파일러 플러그인 (Kotlin 한정, 가장 결정적)

Kotlin으로 JPA를 쓰면 보통 이 플러그인을 넣는다.

```kotlin
plugins {
    kotlin("plugin.jpa")     // no-arg 합성 + final 해제 (2.3.20+, 뒤에서 확인한다)
    kotlin("plugin.spring")  // Spring 빈의 final 해제. 엔티티와는 무관하다
}
```

`kotlin-jpa`는 no-arg 플러그인을 JPA용으로 특수화한 것이다. 어떤 어노테이션을 트리거로 삼는지는 플러그인 jar 안에 그대로 들어 있다.

```
$ unzip -p kotlin-noarg-compiler-plugin-embeddable-<version>.jar \
      org/jetbrains/kotlin/noarg/NoArgPluginNames.class | strings
jpa
javax.persistence.Entity      javax.persistence.Embeddable      javax.persistence.MappedSuperclass
jakarta.persistence.Entity    jakarta.persistence.Embeddable    jakarta.persistence.MappedSuperclass
```

이 세 어노테이션이 붙은 클래스에 파라미터 없는 생성자를 합성해 준다. Kotlin은 주 생성자에 파라미터가 있으면 no-arg 생성자가 존재하지 않고, Hibernate는 리플렉션으로 no-arg 생성자를 호출해 인스턴스를 만들기 때문에 이게 없으면 엔티티를 만들 수 없다.

여기서 핵심은 이것이다.

> **컴파일러 플러그인은 `orm.xml`을 읽지 않는다. 읽을 수도 없다.**
> XML은 런타임에 Hibernate가 파싱하는 리소스이고, 플러그인은 컴파일 시점에 소스 트리만 본다.
> **따라서 클래스에는 어떤 형태로든 어노테이션이 남는다.**

도입부에서 언급한 `metadata-complete="true"`도 여기서는 소용이 없다. 그건 **런타임에 Hibernate가 어노테이션을 읽지 않게 하는** 스위치다. 컴파일 시점에 플러그인이 어노테이션을 보는 것과는 무관하다. 매핑 메타데이터로서의 역할은 무시시킬 수 있어도, 컴파일러 스위치로서의 역할은 남는다.

그리고 문장을 "어떤 형태로든"이라고 쓰는 데는 이유가 있다. 정확히 말하면 필요한 것은 `@Entity` **자체가 아니다.** noarg와 allopen 플러그인은 프리셋 대신 임의의 어노테이션을 지정할 수 있다.

```kotlin
noArg { annotation("com.example.NoArgConstructor") }   // 내가 만든 어노테이션도 트리거가 된다
```

그러니 이론상 `@Entity`를 떼고 자체 마커 어노테이션을 붙이는 것도 가능하다. 하지만 그건 **어노테이션을 다른 어노테이션으로 바꾼 것**이지, 클래스에서 어노테이션을 없앤 게 아니다. 게다가 그 대가로 IDE 지원(벽 3)까지 잃는다. 실익이 없다.

즉 벽의 정체는 "`@Entity`가 특별해서"가 아니라 **"XML은 컴파일러에게 아무 말도 걸 수 없어서"** 다.

### 버전을 바꿔 보면 명확해진다

`@Entity` 하나만 다른 두 클래스를 준비했다.

```kotlin
@Entity
class Order(
    @Id val id: Long? = null,
    val userId: Long,
)

class OrderSummary(
    val id: Long? = null,
    val userId: Long,
)
```

이걸 Kotlin 2.2.20과 2.3.21로 각각 컴파일하고 `javap`로 열어 봤다.

```
=== Kotlin 2.2.20 ===
[Order       ]  public final class lab.jpa.Order          ← ACC_FINAL 있음
[OrderSummary]  public final class lab.jpa.OrderSummary   ← ACC_FINAL 있음
[Order       ]  no-arg 생성자: 있음
[OrderSummary]  no-arg 생성자: 없음

=== Kotlin 2.3.21 ===
[Order       ]  public class lab.jpa.Order                ← ACC_FINAL 없음
[OrderSummary]  public final class lab.jpa.OrderSummary   ← ACC_FINAL 있음
[Order       ]  no-arg 생성자: 있음
[OrderSummary]  no-arg 생성자: 없음
```

읽는 방법은 두 축이다.

**가로 — `@Entity`가 만드는 차이.** no-arg 생성자는 두 버전 모두에서 갈린다. 이건 예전부터 `kotlin-jpa`가 하던 일이다.

**세로 — 버전이 만드는 차이.** 2.3.21에서 `Order`의 `ACC_FINAL`만 사라졌다. 대조군인 `OrderSummary`는 두 버전 모두 그대로다. 즉 컴파일러 전반의 변화가 아니라 **`@Entity`가 트리거하는 동작이 하나 늘어난 것**이다.

`allOpen { ... }` 설정은 어디에도 없다. `kotlin("plugin.jpa")` 한 줄이 전부다.

### 왜 final이 문제인가

Kotlin 클래스는 기본이 `final`이다. Hibernate는 지연 로딩을 위해 엔티티의 **서브클래스 프록시**를 만들어야 하므로, final 엔티티는 프록시를 만들 수 없다. `@ManyToOne(fetch = LAZY)`를 걸어도 의도대로 동작하지 않는다.

그래서 2.3.20 이전에는 이 설정을 직접 넣어야 했다.

```kotlin
allOpen { annotation("jakarta.persistence.Entity") }
```

이 변화는 [KT-28594](https://youtrack.jetbrains.com/issue/KT-28594)와 KT-79389가 2.3.20에 반영되면서 생겼다. KT-28594는 6년 넘게 열려 있던 이슈다. JetBrains가 2026년 1월에 낸 글에도 "`plugin.jpa`는 필요한 설정을 다 해주는 것처럼 보이지만 no-arg만 설정하고 all-open은 설정하지 않는다"고 적혀 있다. 그때는 맞는 설명이었다.

**그러니 이 글을 읽고 "`plugin.jpa`가 알아서 열어주는구나" 하고 넘어가면 안 된다.** 2.3.20 미만에서는 지연 로딩이 조용히 깨진 채로 간다. 자기 프로젝트의 Kotlin 버전부터 확인할 것.

> 위 출력은 [blog-lab/001-jpa-entity-final](https://github.com/BlueSF/blog-lab/tree/main/001-jpa-entity-final)에서 `./verify.sh --compare`로 재현할 수 있다. JDK 17 이상만 있으면 된다.

### 여담 — 이 문단이 가장 오래 걸렸다

처음엔 all-open의 spring 프리셋 목록만 보고 막혔다. 거기에 `jakarta.persistence.Entity`가 없었기 때문이다.

```
$ unzip -p kotlin-allopen-compiler-plugin-embeddable-<version>.jar \
      org/jetbrains/kotlin/allopen/AllOpenPluginNames.class | strings
spring
org.springframework.stereotype.Component
org.springframework.transaction.annotation.Transactional
...
```

"프리셋에 없는데 왜 열려 있지?"에서 한참을 헤맸다. 같은 파일 뒤쪽에 **jpa 프리셋이 따로 있었는데** grep 범위를 좁게 잡은 탓이다. 결국 Gradle 플러그인 진입점을 디컴파일하고서야 풀렸다.

```java
// KotlinJpaSubplugin.apply()
plugins.apply(NoArgGradleSubplugin.class)      // 1. no-arg 플러그인 적용
plugins.apply(AllOpenGradleSubplugin.class)    // 2. all-open 플러그인도 적용
noArgExtension.myPresets.add("jpa")            // 3. no-arg에 jpa 프리셋 등록
allOpenExtension.preset("jpa")                 // 4. all-open에도 jpa 프리셋 등록  ← 이것
```

참고로 우리 프로젝트의 버전 카탈로그에는 이렇게 적혀 있었다.

```toml
kotlin-jpa = { ... }      # JPA 엔티티 지원 (noarg 자동화)
kotlin-allopen = { ... }  # 클래스 open 처리 (Spring 프록시용)
```

내가 쓴 주석이다. 그리고 2.3.20 전까지는 정확한 설명이었다. **문서가 틀렸던 게 아니라, 산출물이 문서보다 먼저 바뀐 것이다.**

---

## 벽 2 — XML로 옮길 수 없는 것들

이건 언어와 무관한, 명세 차원의 문제다. 그리고 **여기서 대부분의 사람이 틀린다. 나도 틀렸다.**

### (1) 진짜로 XML에 없는 것 — 제3의 라이브러리가 읽는 어노테이션

```kotlin
@MappedSuperclass
@EntityListeners(AuditingEntityListener::class)
abstract class BaseEntity {
    @CreatedDate      var createdAt: LocalDateTime? = null
    @LastModifiedDate var updatedAt: LocalDateTime? = null
}
```

`@CreatedDate`, `@LastModifiedDate`는 **JPA 명세가 아니라 Spring Data의 어노테이션**이다. `orm.xml` XSD에 대응 요소가 존재할 이유가 없고, 실제로 없다. `AuditingEntityListener`가 런타임에 필드의 어노테이션을 리플렉션으로 직접 읽기 때문에 XML로 옮길 방법이 원천적으로 없다.

판별 기준은 단순하다. **JPA/Hibernate가 아닌 제3의 라이브러리가 어노테이션을 직접 읽는가?** 그렇다면 XML로 못 옮긴다. 이 범주는 생각보다 좁다.

### (2) "Hibernate 전용이라 XML 안 될 것" — 대부분 된다

내가 처음에 이렇게 적었다.

> `@DynamicUpdate`, `@BatchSize` 같은 Hibernate 고유 어노테이션은 XSD가 커버하지 않는다.

**틀렸다.** Hibernate 6의 `mapping-3.1.0.xsd`를 직접 열어 보면 나온다.

```bash
$ unzip -p hibernate-core-<version>.jar org/hibernate/xsd/mapping/mapping-3.1.0.xsd > mapping.xsd
$ grep -c 'name="dynamic-update"' mapping.xsd   # 1
$ grep -c 'name="batch-size"'     mapping.xsd   # 1
$ grep -oE '@org\.hibernate\.annotations\.[A-Za-z]+' mapping.xsd | sort -u
@org.hibernate.annotations.Cascade
@org.hibernate.annotations.ColumnDefault
@org.hibernate.annotations.Comment
@org.hibernate.annotations.JdbcTypeCode
@org.hibernate.annotations.NaturalId
@org.hibernate.annotations.OnDelete
@org.hibernate.annotations.Type
... (계속)
```

XSD 주석에 대응 어노테이션이 친절하게 적혀 있다. Hibernate 6의 매핑 XSD는 표준 JPA를 넘어 자사 확장까지 폭넓게 커버한다. "Hibernate 전용 어노테이션 = XML 불가"는 성립하지 않는다.

`@Converter`도 마찬가지다. `<convert converter="..."/>`(사용처 지정)만 있는 줄 알았는데, `entity-mappings` 최상위에 **컨버터 클래스를 등록하는 요소가 따로 있다.**

```xml
<entity-mappings ...>
    <converter class="com.example.MoneyConverter" auto-apply="true"/>
    ...
</entity-mappings>
```

### (3) "스키마엔 있지만 우리가 안 쓴" 것들 — 가장 흔한 착각

`<transient>`, `<post-persist>`, `<entity-listeners>` 같은 요소는 XSD에 **존재한다.** 그래서 "옮길 수 있는 것"으로 분류하기 쉽다. 하지만 실제로 XML에 그 요소를 써넣지 않았다면 어노테이션이 **유일한 선언**이므로 지우는 순간 매핑이 사라진다.

특히 위험한 게 `@Transient`다.

```kotlin
@MappedSuperclass
abstract class BaseEntity {
    @Transient
    private var cachedId: Long? = null   // 이 어노테이션을 지우면?
}
```

JPA는 명시적으로 제외하지 않은 필드를 전부 영속 대상으로 본다. `@Transient`를 지우면 Hibernate가 `cached_id` 컬럼을 찾다가 `ddl-auto: validate` 환경에서 **애플리케이션 기동 자체가 실패**한다.

> 여기서 진짜 함정은 환경별로 증상이 다르다는 것이다.
> 개발 환경이 `ddl-auto: none`이면 스키마 검증을 건너뛰므로 아무 일도 없어 보인다.
> 매핑이 깨진 채로 배포되고, `validate`가 켜진 운영에서 처음 터진다.

**교훈: "XSD에 요소가 있다"와 "우리 XML에 그 요소가 있다"는 완전히 다른 이야기다.** 어노테이션을 지우기 전에 대응 XML에 실제로 그 선언이 있는지 확인해야 한다.

```bash
# 우리 XML이 실제로 쓰고 있는 요소인지 확인
grep -l "<transient\|post-persist\|entity-listener" src/main/resources/META-INF/*.xml
```

그리고 (2)에서 내가 두 번 틀렸다는 사실이 이 교훈의 가장 좋은 증거다. `@DynamicUpdate`도 `@Converter`도 "XML로는 안 되는 것"이라고 굳게 믿었지만, 실은 **XSD에 다 있는데 우리 XML이 안 쓰고 있었을 뿐**이었다. 증상은 똑같다 — 어노테이션을 지우면 깨진다. 하지만 원인이 다르고, 따라서 **해결책도 다르다.**

| 원인                               | 해결책                                              |
| ---------------------------------- | --------------------------------------------------- |
| XSD에 요소가 없다 (Spring Data 등) | 없음. 어노테이션을 남긴다                           |
| XSD에 있는데 우리 XML이 안 썼다    | **XML에 요소를 추가하면 어노테이션을 지울 수 있다** |

"불가능"과 "안 해놨음"을 구별하지 못하면, 할 수 있는 일을 못 한다고 결론 내리게 된다. XSD는 jar에서 꺼내 grep 한 번이면 확인된다. 추측하지 말자.

---

## 벽 3 — IDE

여기까지 통과해서 남은 어노테이션이 있다. `@Id`, `@Access`, `@Version`, `@GeneratedValue` 같은, **XML에 이미 똑같이 선언돼 있는데도 클래스에 남아 있는** 것들이다. 런타임에는 아무 영향이 없다. 순수하게 IDE 때문이다.

원인은 대부분 이것이다.

> `orm.xml`을 `persistence.xml`이 아니라 Spring Boot 설정으로 등록했다.

```yaml
spring:
  jpa:
    mapping-resources:
      - META-INF/orm-order.xml
      - META-INF/orm-user.xml
```

이 방식은 런타임에는 완벽히 동작한다. 문제는 IDE다. IntelliJ의 JPA 지원은 persistence unit(즉 `persistence.xml`) 기준으로 매핑 파일을 수집한다. YAML에만 적힌 매핑 파일은 IDE 입장에서 **그냥 평범한 XML 파일**이다.

결과적으로 어노테이션을 지우면 IDE는 그 클래스를 엔티티로 인식하지 못하고, `@Query`의 JPQL 검증·파생 쿼리 메서드 검증·연관관계 탐색이 전부 깨진다. 그래서 "런타임에는 불필요하지만 IDE를 위해" 어노테이션을 남기게 된다.

### 그런데 이건 지워도 해결되지 않는다

`@Id`를 지운다고 IDE가 조용해지지 않는다.

`@Entity`는 벽 1 때문에 반드시 남아야 하므로 IDE는 계속 그 클래스를 엔티티로 인식한다. 그 상태에서 `@Id`가 없으면 이번엔 _"Entity should have primary key"_ 가 뜬다. **오류가 사라지는 게 아니라 다른 오류로 바뀔 뿐이다.**

대안도 각각 대가가 있다.

| 방법                                     | 문제                                                                                      |
| ---------------------------------------- | ----------------------------------------------------------------------------------------- |
| IntelliJ JPA facet에 매핑 파일 수동 등록 | `.idea/`를 gitignore하는 팀이면 공유 불가, 각자 설정해야 함                               |
| `persistence.xml` 도입                   | Spring Boot 자동 설정과 이중화된다. 커스텀 DataSource(읽기/쓰기 분리 등)를 쓰면 충돌 가능 |
| 그냥 남긴다                              | 중복 선언이 늘고, "왜 남았는지" 아는 사람이 사라진다 ← 이 글을 쓰게 된 이유               |

---

## Java와 Kotlin은 결정적으로 다르다

지금까지의 벽 중 **벽 1은 Kotlin에만 있다.** 이게 생각보다 큰 차이를 만든다.

| 항목                         | Java                                                                                                            | Kotlin                                                                                                                                                     |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **no-arg 생성자**            | 명시적 생성자가 없으면 컴파일러가 자동 생성. 있어도 `protected Order() {}` 한 줄이면 끝 — **어노테이션과 무관** | 주 생성자에 파라미터가 있으면 존재하지 않음. 플러그인이 **어노테이션을 보고** 합성                                                                         |
| **final / 지연 로딩 프록시** | 클래스·메서드 기본 non-final → 프록시 정상                                                                      | 기본 final → all-open 플러그인이 어노테이션 기준으로 해제. **2.3.20부터 `kotlin-jpa`가 jpa 프리셋을 자동 등록하며, 그 이전 버전에서는 수동 등록이 필요**   |
| **접근 타입(access type)**   | 필드에 붙이면 FIELD, getter에 붙이면 PROPERTY. 위치가 곧 의미                                                   | 프로퍼티가 항상 getter를 동반해 모호. `@Access(FIELD)`나 XML `<access>`로 명시하는 편이 안전                                                               |
| **어노테이션 적용 대상**     | 붙인 위치가 곧 대상                                                                                             | 생성자 파라미터의 어노테이션은 param→property→field 순으로 해석. JPA 어노테이션은 PARAMETER 타깃이 없어 field로 떨어짐 — 동작은 하지만 규칙이 눈에 안 보임 |

Java 진영에서는 no-arg 생성자를 이렇게 해결한다.

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)   // Lombok
@Entity
public class Order { ... }
```

생성자 확보가 **JPA 어노테이션과 완전히 분리**되어 있다. 그래서 Java 엔티티는 이론상 `@Entity`까지 지우고 XML만으로 매핑하는 게 가능하다(직접 검증해 보지는 않았다). Kotlin에서는 불가능하다.

### 정리하면

```
Java   : 어노테이션 = 매핑 메타데이터                    → 클래스를 완전히 비울 수 있음
Kotlin : 어노테이션 = 매핑 메타데이터 + 컴파일러 스위치  → 클래스에 무언가는 반드시 남음
```

**"XML 하이브리드 매핑으로 도메인을 순수하게 만들겠다"는 목표는 Kotlin에서 절반까지만 달성된다.** 상세 매핑(테이블명·컬럼명·인덱스·컨버터 지정)은 전부 XML로 뺄 수 있지만, 엔티티 선언 자체는 클래스에 남는다.

---

## 그래서 어떻게 할 것인가

어노테이션을 지우기 전에 세 가지를 순서대로 확인하면 된다.

**1단계 — 컴파일러 플러그인의 트리거인가?** (Kotlin)
`@Entity`, `@Embeddable`, `@MappedSuperclass`(또는 `noArg`/`allOpen`에 직접 등록한 어노테이션)라면 여기서 끝이다. XML은 컴파일러에게 말을 걸 수 없으므로 어떤 형태로든 어노테이션이 남아야 한다.

**2단계 — XSD에 대응 요소가 있는가?**
`hibernate-core` jar에서 `mapping-*.xsd`를 꺼내 grep한다. **추측하지 말고 열어 본다.** 없다면(주로 Spring Data처럼 제3의 라이브러리가 리플렉션으로 읽는 것) 지울 수 없다.

**3단계 — 대응 XML에 실제로 그 선언이 있는가?**
XSD에 있는 것과 우리 XML에 있는 것은 다르다. 없다면 지금은 지울 수 없지만, **XML에 요소를 추가하면 지울 수 있다** — 2단계와 결론이 같아 보여도 대응이 전혀 다르다.

세 관문을 다 통과한 어노테이션만이 "중복"이고, 그것들은 대체로 IDE를 위해 남겨 두는 편이 낫다.

### 팀에 남길 것

이 분류를 코딩 컨벤션 문서에 **근거와 함께** 적어 두는 걸 권한다. 흔히 "IDE 인식용으로 이 어노테이션들은 허용"이라고 뭉뚱그려 적는데, 이러면 실제로는 런타임 필수인 `@Entity`나 `@Transient`까지 "IDE 편의를 위한 장식"으로 읽힌다. 누군가 선의로 정리하다가 운영 기동 실패를 만든다. 그것도 개발 환경에서는 재현되지 않는 형태로.

| 분류                    | 예시                                                               | 지우면                           | 해소 방법                         |
| ----------------------- | ------------------------------------------------------------------ | -------------------------------- | --------------------------------- |
| 컴파일러 트리거         | `@Entity` `@Embeddable` `@MappedSuperclass`                        | 인스턴스화 실패 / 지연 로딩 파손 | 없음 (Kotlin의 구조적 제약)       |
| XSD에 요소 없음         | `@CreatedDate` `@LastModifiedDate`                                 | 해당 기능 소실                   | 없음                              |
| XSD엔 있으나 XML 미기재 | `@Transient`, 라이프사이클 콜백, `@DynamicUpdate`, 미기재 연관관계 | `validate` 환경에서 기동 실패    | **XML에 요소 추가하면 제거 가능** |
| XML과 중복              | `@Id` `@Access` `@Version`                                         | 런타임 무해, IDE 지원만 상실     | 굳이 지울 이유 없음               |

### 마지막으로, 가장 쓸모 있었던 세 가지 도구

이 글의 모든 판단은 결국 세 개의 명령으로 수렴했다. 셋 다 **jar와 클래스 파일을 직접 여는 것**이다.

```bash
# 1. 어노테이션이 컴파일 결과에 무엇을 만드는가
javap -v build/classes/kotlin/main/com/example/YourEntity.class | grep -E "flags|YourEntity\("

# 2. 이 어노테이션이 XML에 대응 요소가 있는가
unzip -p hibernate-core-*.jar org/hibernate/xsd/mapping/mapping-3.1.0.xsd | grep 'name="batch-size"'

# 3. 그 플러그인이 실제로 무슨 일을 하는가
javap -c org/jetbrains/kotlin/noarg/gradle/KotlinJpaSubplugin.class
```

이 글의 초고에서 나는 `@DynamicUpdate`와 `@Converter`를 "XML로는 표현할 수 없는 것"으로 분류했다. 둘 다 틀렸다. XSD를 열어 보지 않고 "Hibernate 전용이니까", "클래스 등록이니까" 하고 추론했기 때문이다. 2번 명령 한 줄이면 30초 만에 확인됐을 일이다.

final 해제의 범인을 찾을 때도 같았다. 프리셋 목록을 grep하다 막혔고, 결국 3번처럼 **플러그인 진입점 자체를 열어보고서야** `kotlin-jpa`가 all-open까지 적용한다는 걸 알았다. 공식 문서에도 "noarg의 특수화"라고만 적혀 있어 읽기만 해서는 알 수 없었다.

**문서나 블로그(이 글 포함)를 믿는 것보다, 자기 프로젝트의 jar와 산출물을 직접 여는 쪽이 언제나 빠르고 정확하다.** 막혔을 때 답은 대개 한 단계 아래에 있다.
