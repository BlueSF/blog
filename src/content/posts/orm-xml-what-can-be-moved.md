---
author: 위영종
pubDatetime: 2026-08-16T23:00:00+09:00
title: "orm.xml로 옮길 수 있는 것과 없는 것"
description: "@DynamicUpdate는 XML로 옮길 수 있고 @CreatedDate는 없다. 'XSD에 없다'와 '우리 XML에 안 썼다'를 구별하는 문제다."
featured: false
draft: false
tags: ["kotlin", "jpa", "hibernate", "hexagonal-architecture", "orm-xml"]
---

[앞선 글](/posts/kotlin-entity-cannot-be-removed/)에서 Kotlin의 `@Entity`는 지울 수 없다는 걸 확인했다. 그렇다면 나머지 어노테이션은 어떤가.

`@Column`이나 `@Table`처럼 명백히 매핑 정보인 것들은 물론 XML로 나간다. 문제는 그 사이에 있는 것들이다 — `@DynamicUpdate`, `@Converter`, `@CreatedDate`, `@Transient`.

여기서 내가 두 번 틀렸다. 그리고 틀린 방식이 흥미로웠다.

결론부터 말하면 **"이 어노테이션을 지울 수 있는가"는 하나의 질문이 아니라 네 개다.** 어노테이션마다 막히는 지점이 다르고, 지점이 다르면 해결책도 다르다. 아래 네 관문을 순서대로 통과시키면 된다.

## 목차

---

## 1관문 · 컴파일러가 읽는 것

Kotlin에서 `@Entity`, `@Embeddable`, `@MappedSuperclass`는 매핑 정보이기 이전에 컴파일러 플러그인의 트리거다. `orm.xml`은 런타임 리소스라 컴파일러에게 아무 말도 걸 수 없으므로, 여기 걸린 어노테이션은 **어떤 형태로든 클래스에 남는다.** 근거는 [1편](/posts/kotlin-entity-cannot-be-removed/)에 있다.

이 관문은 Kotlin에만 있다. Java라면 그냥 통과다.

## 2관문 · XSD에 대응 요소가 없는 것

여기서부터는 언어와 무관한, 명세 차원의 문제다.

```kotlin
@MappedSuperclass
@EntityListeners(AuditingEntityListener::class)
abstract class BaseEntity {
    @CreatedDate      var createdAt: LocalDateTime? = null
    @LastModifiedDate var updatedAt: LocalDateTime? = null
}
```

`@CreatedDate`, `@LastModifiedDate`는 **JPA 명세가 아니라 Spring Data의 어노테이션**이다. `orm.xml` XSD에 대응 요소가 존재할 이유가 없고, 실제로 없다. `AuditingEntityListener`가 런타임에 필드의 어노테이션을 리플렉션으로 직접 읽기 때문에 XML로 옮길 방법이 원천적으로 없다.

판별 기준은 단순하다. **JPA/Hibernate가 아닌 제3의 라이브러리가 어노테이션을 직접 읽는가?** 그렇다면 XML로 못 옮긴다.

그리고 이 범주는 **생각보다 훨씬 좁다.** 여기가 내가 틀린 지점이다.

## 3관문 · XSD엔 있는데 우리 XML이 안 쓴 것

가장 흔하고, 가장 위험하고, 유일하게 **해결책이 있는** 관문이다.

`<transient>`, `<post-persist>`, `<entity-listeners>` 같은 요소는 XSD에 **존재한다.** 그래서 "옮길 수 있는 것"으로 분류하기 쉽다. 하지만 실제로 XML에 그 요소를 써넣지 않았다면 어노테이션이 **유일한 선언**이므로, 지우는 순간 매핑이 사라진다.

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

### 내가 두 번 틀린 곳도 여기다

2관문(XSD에 없다)이라고 믿었는데 실은 3관문(우리가 안 썼다)이었던 것이 두 개 있다. 초고에 이렇게 적었다.

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

두 경우 모두 증상은 3관문의 `@Transient`와 똑같다 — 어노테이션을 지우면 깨진다. 하지만 원인이 다르고, 따라서 **해결책도 다르다.**

| 원인                                    | 해결책                                              |
| --------------------------------------- | --------------------------------------------------- |
| 2관문 · XSD에 요소가 없다 (Spring Data) | 없음. 어노테이션을 남긴다                           |
| 3관문 · XSD에 있는데 우리 XML이 안 썼다 | **XML에 요소를 추가하면 어노테이션을 지울 수 있다** |

"불가능"과 "안 해놨음"을 구별하지 못하면, 할 수 있는 일을 못 한다고 결론 내리게 된다. XSD는 jar에서 꺼내 grep 한 번이면 확인된다. 추측하지 말자.

```bash
# 우리 XML이 실제로 쓰고 있는 요소인지 확인
grep -l "<transient\|post-persist\|entity-listener" src/main/resources/META-INF/*.xml
```

### metadata-complete는 이 관문을 건너뛰게 해주지 않는다

[1편](/posts/kotlin-entity-cannot-be-removed/) 도입부에 나온 `metadata-complete="true"`를 떠올릴 수 있다. 어노테이션을 **전부 무시**시키는 스위치이니, 지우지 않고도 XML만 보게 만들면 되지 않나?

정확히 반대다. 이 관문의 문제는 **XML이 불완전하다는 것**인데, `metadata-complete`는 그 불완전한 XML을 유일한 진실로 선언해 버린다. 무시된 어노테이션의 자리를 XML이 대신 채워주지는 않는다.

이번엔 추측하지 않고 돌려 봤다. 클래스는 하나로 고정하고 XML만 세 벌 만들었다.

|     | `metadata-complete` | XML의 `<transient>` | 결과                                       |
| --- | ------------------- | ------------------- | ------------------------------------------ |
| A   | 없음                | 없음                | 기동 성공 — 클래스의 `@Transient`가 읽힌다 |
| B   | `true`              | 없음                | **기동 실패**                              |
| C   | `true`              | 있음                | 기동 성공                                  |

```
[기동 실패] metadata-complete="true"      + XML에 <transient> 없음
            Schema-validation: missing column [cachedId] in table [orders]
```

B가 요점이다. `@Transient`는 클래스에 그대로 남아 있는데도 무시되어, 어노테이션을 지웠을 때와 **똑같이** 기동에 실패한다. 그리고 B와 C의 차이는 `<transient name="cachedId"/>` 한 줄뿐이다.

그러니 `metadata-complete`는 3관문을 우회하는 수단이 아니라, **3관문을 이미 통과한 뒤에야 켤 수 있는 스위치**다. 켜는 순간 "우리 XML에 빠진 선언이 하나도 없다"를 약속하는 셈이고, 그 약속이 거짓이면 B가 된다.

> [blog-lab/003-metadata-complete](https://github.com/BlueSF/blog-lab/tree/main/003-metadata-complete)에서 `./verify.sh`로 재현할 수 있다. 위 메시지가 `cached_id`가 아니라 `cachedId`인 것은 그 모듈이 순수 Hibernate라서다. Spring Boot에서는 네이밍 전략이 붙어 `cached_id`를 찾는다. 컬럼명은 환경마다 다르고, 컬럼을 찾다가 실패한다는 사실은 같다.

## 4관문 · XML과 중복인 것

세 관문을 다 통과해서 남은 어노테이션이 있다. `@Id`, `@Access`, `@Version`, `@GeneratedValue` 같은, **XML에 이미 똑같이 선언돼 있는데도 클래스에 남아 있는** 것들이다. 런타임에는 아무 영향이 없다. 순수하게 IDE 때문이다.

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

`@Entity`는 [1편](/posts/kotlin-entity-cannot-be-removed/)에서 본 이유로 반드시 남아야 하므로 IDE는 계속 그 클래스를 엔티티로 인식한다. 그 상태에서 `@Id`가 없으면 이번엔 _"Entity should have primary key"_ 가 뜬다. **오류가 사라지는 게 아니라 다른 오류로 바뀔 뿐이다.**

대안도 각각 대가가 있다.

| 방법                                     | 문제                                                                                      |
| ---------------------------------------- | ----------------------------------------------------------------------------------------- |
| IntelliJ JPA facet에 매핑 파일 수동 등록 | `.idea/`를 gitignore하는 팀이면 공유 불가, 각자 설정해야 함                               |
| `persistence.xml` 도입                   | Spring Boot 자동 설정과 이중화된다. 커스텀 DataSource(읽기/쓰기 분리 등)를 쓰면 충돌 가능 |
| 그냥 남긴다                              | 중복 선언이 늘고, "왜 남았는지" 아는 사람이 사라진다 ← 이 글을 쓰게 된 이유               |

---

## 그래서 어떻게 할 것인가

지금까지의 네 관문을 한 장으로 줄이면 이 표다.

| 관문                        | 예시                                                               | 지우면                           | 해소 방법                         |
| --------------------------- | ------------------------------------------------------------------ | -------------------------------- | --------------------------------- |
| 1 · 컴파일러 트리거         | `@Entity` `@Embeddable` `@MappedSuperclass`                        | 인스턴스화 실패 / 지연 로딩 파손 | 없음 (Kotlin의 구조적 제약)       |
| 2 · XSD에 요소 없음         | `@CreatedDate` `@LastModifiedDate`                                 | 해당 기능 소실                   | 없음                              |
| 3 · XSD엔 있으나 XML 미기재 | `@Transient`, 라이프사이클 콜백, `@DynamicUpdate`, 미기재 연관관계 | `validate` 환경에서 기동 실패    | **XML에 요소 추가하면 제거 가능** |
| 4 · XML과 중복              | `@Id` `@Access` `@Version`                                         | 런타임 무해, IDE 지원만 상실     | 굳이 지울 이유 없음               |

실무에서 중요한 건 **2관문과 3관문을 구별하는 것**이다. 둘은 증상이 똑같다. 어노테이션을 지우면 깨진다. 하지만 한쪽은 손쓸 방법이 없고, 다른 쪽은 XML에 한 줄 추가하면 끝난다. 이 둘을 뭉뚱그리면 할 수 있는 일을 못 한다고 결론 내리게 된다.

구별하는 방법은 추측이 아니라 `hibernate-core` jar에서 `mapping-*.xsd`를 꺼내 grep하는 것이다.

### 팀에 남길 것

이 분류를 코딩 컨벤션 문서에 **근거와 함께** 적어 두는 걸 권한다. 흔히 "IDE 인식용으로 이 어노테이션들은 허용"이라고 뭉뚱그려 적는데, 이러면 실제로는 런타임 필수인 `@Entity`나 `@Transient`까지 "IDE 편의를 위한 장식"으로 읽힌다. 누군가 선의로 정리하다가 운영 기동 실패를 만든다. 그것도 개발 환경에서는 재현되지 않는 형태로.

위 표에서 실제로 손댈 수 있는 것은 3관문 한 줄뿐이라는 점도 같이 적어 두는 편이 낫다. 나머지 세 줄은 "지우지 마라"가 아니라 "지울 수 없다"거나 "지워도 이득이 없다"이기 때문이다.

### 마지막으로, 가장 쓸모 있었던 세 가지 도구

두 글의 모든 판단은 결국 세 개의 명령으로 수렴했다. 셋 다 **jar와 클래스 파일을 직접 여는 것**이다.

```bash
# 1. 어노테이션이 컴파일 결과에 무엇을 만드는가
javap -v build/classes/kotlin/main/com/example/YourEntity.class | grep -E "flags|YourEntity\("

# 2. 이 어노테이션이 XML에 대응 요소가 있는가
unzip -p hibernate-core-*.jar org/hibernate/xsd/mapping/mapping-3.1.0.xsd | grep 'name="batch-size"'

# 3. 그 플러그인이 실제로 무슨 일을 하는가
javap -c org/jetbrains/kotlin/noarg/gradle/KotlinJpaSubplugin.class
```

이 글의 초고에서 나는 `@DynamicUpdate`와 `@Converter`를 "XML로는 표현할 수 없는 것"으로 분류했다. 둘 다 틀렸다. XSD를 열어 보지 않고 "Hibernate 전용이니까", "클래스 등록이니까" 하고 추론했기 때문이다. 2번 명령 한 줄이면 30초 만에 확인됐을 일이다.

[1편에서 final 해제의 원인을 찾을 때](/posts/kotlin-entity-cannot-be-removed/)도 같았다. 문서를 읽어서는 끝내 알 수 없었고, 3번 명령으로 플러그인 진입점을 열고서야 풀렸다.

이 시리즈의 검증 장치는 AI 에이전트로 만들었다. 그래도 결론은 같다. 직접 열어보기 전까지는 이것도 문서다. `verify.sh`는 200줄이 안 되고, 보여주는 건 `javap`와 Hibernate의 출력이다. 어디까지가 AI의 것인지는 [blog-lab README](https://github.com/BlueSF/blog-lab#이-레포와-ai)에 적어 뒀다.

**문서나 블로그(이 글 포함)를 믿는 것보다, 자기 프로젝트의 jar와 산출물을 직접 여는 쪽이 언제나 빠르고 정확하다.** 막혔을 때 답은 대개 한 단계 아래에 있다.

어노테이션을 어디까지 허용할 것인가라는 판단 자체에 대해서는 [도메인 클래스에 무엇을 허용할 것인가](/posts/what-to-allow-in-domain-class/)에서 다뤘다.
