---
title: "Spring Cache와 로컬 캐시, 분산 캐시 이해하기"
categories:
  - Backend
  - Spring
tags:
  - Spring
  - SpringBoot
  - Cache
  - Cacheable
  - CachePut
  - CacheEvict
  - LocalCache
  - DistributedCache
  - Redis
  - Performance
---

웹 애플리케이션에서 모든 요청마다 DB나 외부 API를 조회하면 응답 속도가 느려지고,
트래픽이 증가할수록 서버와 DB에 부담이 커진다.

이때 자주 사용되는 방법이 `Cache`이다.
Cache는 자주 조회되는 데이터를 더 빠른 저장소에 임시로 저장해두고,
다음 요청에서 다시 계산하거나 조회하지 않도록 도와준다.

Spring에서는 `Spring Cache Abstraction`을 통해 Cache 기능을 추상화해서 사용할 수 있다.
대표적으로 다음 세 가지 애노테이션을 많이 사용한다.

```text
1. @Cacheable
2. @CachePut
3. @CacheEvict
```

또한 Cache 저장 위치에 따라 `로컬 캐시(Local Cache)`와 `분산 캐시(Distributed Cache)`로 나눌 수 있다.
이 글에서는 Spring Cache 애노테이션의 차이와 Cache 구조 선택 기준을 함께 정리한다.

---

## 1. Spring Cache 기본 설정

Spring Cache를 사용하려면 먼저 Cache 기능을 활성화해야 한다.

```java
@EnableCaching
@Configuration
public class CacheConfig {
}
```

`@EnableCaching`을 추가하면 Spring이 Cache 관련 애노테이션을 인식하고,
대상 메서드 호출 전후에 Cache 동작을 적용한다.

Spring Boot에서는 Cache 구현체가 클래스패스에 있으면 적절한 `CacheManager`를 자동 구성할 수 있다.
예를 들어 Caffeine을 사용하면 로컬 캐시를,
Redis Cache 설정을 사용하면 Redis 기반 분산 캐시를 적용할 수 있다.

중요한 점은 Spring Cache가 기본적으로 `Proxy` 기반으로 동작한다는 것이다.
따라서 같은 클래스 내부에서 Cache 메서드를 직접 호출하면 Cache가 적용되지 않을 수 있다.

```java
@Service
public class ProductService {

    public ProductResponse getProductForInternalUse(Long productId) {
        return getProduct(productId);
    }

    @Cacheable(cacheNames = "products", key = "#productId")
    public ProductResponse getProduct(Long productId) {
        // ...
    }
}
```

위처럼 같은 객체 내부에서 `getProduct()`를 직접 호출하면 Spring Proxy를 거치지 않는다.
실무에서는 Cache가 필요한 메서드를 별도 Bean으로 분리하거나,
외부 Bean을 통해 호출되도록 구조를 잡는 것이 안전하다.

---

## 2. Cache를 사용하는 이유

Cache의 목적은 단순히 데이터를 저장하는 것이 아니라,
반복되는 비용을 줄이는 것이다.

예를 들어 다음과 같은 작업은 매 요청마다 수행하기 부담스러울 수 있다.

- 자주 조회되는 상품 목록 조회
- 변경이 적은 공통 코드 조회
- 외부 API를 호출해 받아오는 환율, 날씨, 요금제 정보
- 복잡한 통계 쿼리 결과

Cache를 사용하면 다음과 같은 이점을 얻을 수 있다.

```text
응답 속도 개선
DB 부하 감소
외부 API 호출 비용 감소
반복 계산 비용 감소
```

하지만 Cache는 원본 데이터와 별도로 존재하는 복사본이다.
따라서 데이터가 변경되었을 때 Cache를 어떻게 갱신하거나 제거할지 함께 설계해야 한다.

---

## 3. @Cacheable

`@Cacheable`은 조회 결과를 Cache에 저장하고,
동일한 요청이 다시 들어오면 메서드를 실행하지 않고 Cache에서 값을 반환한다.

가장 일반적인 조회 Cache에 사용한다.

```java
@Service
public class ProductService {

    @Cacheable(cacheNames = "products", key = "#productId")
    public ProductResponse getProduct(Long productId) {
        return productRepository.findById(productId)
                .map(ProductResponse::from)
                .orElseThrow(() -> new IllegalArgumentException("상품을 찾을 수 없습니다."));
    }
}
```

처음 `getProduct(1L)`을 호출하면 DB를 조회하고 결과를 Cache에 저장한다.
이후 같은 `productId`로 다시 호출하면 DB를 조회하지 않고 Cache 값을 반환한다.

흐름은 다음과 같다.

```text
1. Cache에 key가 있는지 확인
2. 있으면 Cache 값 반환
3. 없으면 메서드 실행
4. 실행 결과를 Cache에 저장
5. 결과 반환
```

`@Cacheable`은 다음과 같은 상황에 적합하다.

- 조회가 자주 발생한다.
- 데이터 변경 빈도가 낮다.
- 같은 파라미터에 대해 같은 결과가 반환된다.
- 약간의 지연된 갱신을 허용할 수 있다.

예를 들어 카테고리 목록, 공통 코드, 상품 상세, 요금제 상세처럼 읽기 비중이 높은 데이터에 잘 어울린다.

---

## 4. @CachePut

`@CachePut`은 메서드를 항상 실행하고,
그 실행 결과를 Cache에 저장한다.

`@Cacheable`과 가장 큰 차이는 메서드 실행 여부이다.

```text
@Cacheable = Cache에 값이 있으면 메서드 실행 안 함
@CachePut = Cache에 값이 있어도 메서드 항상 실행
```

예를 들어 상품 정보를 수정한 뒤,
수정된 결과를 Cache에 바로 반영하고 싶을 때 사용할 수 있다.

```java
@Service
public class ProductService {

    @CachePut(cacheNames = "products", key = "#productId")
    public ProductResponse updateProduct(Long productId, ProductUpdateRequest request) {
        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new IllegalArgumentException("상품을 찾을 수 없습니다."));

        product.update(request.name(), request.price());

        return ProductResponse.from(product);
    }
}
```

이 경우 상품 수정 메서드는 반드시 실행되어야 한다.
DB에 변경 내용을 반영한 뒤, 반환된 최신 값을 Cache에 저장한다.

`@CachePut`은 다음 상황에 적합하다.

- 데이터 변경 후 Cache도 즉시 최신 값으로 맞추고 싶다.
- 수정 결과를 그대로 조회 Cache에 재사용할 수 있다.
- Cache를 삭제하는 대신 최신 값으로 갱신하고 싶다.

다만 실무에서는 `@CachePut`보다 `@CacheEvict`를 더 자주 쓰는 경우도 많다.
수정 로직의 반환값이 조회 응답과 다르거나,
여러 Cache key가 함께 영향을 받으면 단순 갱신보다 제거 후 재조회가 더 안전할 수 있기 때문이다.

---

## 5. @CacheEvict

`@CacheEvict`는 Cache에 저장된 값을 제거한다.

데이터가 변경되었을 때 기존 Cache 값이 더 이상 신뢰할 수 없다면,
해당 Cache를 비워 다음 조회에서 다시 원본 데이터를 읽게 만들 수 있다.

```java
@Service
public class ProductService {

    @CacheEvict(cacheNames = "products", key = "#productId")
    public void deleteProduct(Long productId) {
        productRepository.deleteById(productId);
    }
}
```

상품이 삭제되면 기존 상품 상세 Cache도 제거해야 한다.
그렇지 않으면 이미 삭제된 상품이 Cache 때문에 계속 조회될 수 있다.

목록 Cache처럼 여러 데이터가 묶여 있는 경우에는 전체 Cache를 제거해야 할 수도 있다.

```java
@CacheEvict(cacheNames = "productList", allEntries = true)
public ProductResponse createProduct(ProductCreateRequest request) {
    Product product = productRepository.save(request.toEntity());
    return ProductResponse.from(product);
}
```

상품이 새로 추가되면 상품 목록 결과가 달라질 수 있다.
이때 특정 key 하나만 제거하기 어렵다면 `allEntries = true`로 해당 Cache 영역 전체를 비울 수 있다.

`@CacheEvict`는 다음 상황에 적합하다.

- 데이터 수정, 삭제 후 기존 Cache가 오래된 값이 된다.
- 수정 결과와 조회 Cache 구조가 다르다.
- 하나의 변경이 여러 조회 결과에 영향을 준다.
- Cache 정합성을 우선하고 싶다.

실무에서는 다음 기준으로 많이 판단한다.

```text
정확히 어떤 Cache key를 갱신할 수 있다 = @CachePut 고려
영향 범위가 넓거나 애매하다 = @CacheEvict 고려
```

---

## 6. 세 애노테이션 비교

| 구분 | 메서드 실행 여부 | Cache 동작 | 주 사용 상황 |
| --- | --- | --- | --- |
| `@Cacheable` | Cache miss일 때만 실행 | 조회 결과 저장 | 반복 조회 최적화 |
| `@CachePut` | 항상 실행 | 실행 결과로 Cache 갱신 | 수정 후 최신 값 반영 |
| `@CacheEvict` | 보통 항상 실행 | Cache 제거 | 수정, 삭제 후 오래된 값 제거 |

정리하면 다음과 같다.

```text
읽기 최적화 = @Cacheable
쓰기 후 갱신 = @CachePut
쓰기 후 무효화 = @CacheEvict
```

Cache를 적용할 때는 조회 성능만 보면 안 된다.
데이터 변경 시점에 Cache가 어떻게 갱신되는지까지 함께 봐야 한다.

---

## 7. Cache Key 설계

Spring Cache는 기본적으로 메서드 파라미터를 기준으로 key를 만든다.
하지만 실무에서는 명시적으로 key를 지정하는 것이 더 안전한 경우가 많다.

```java
@Cacheable(cacheNames = "products", key = "#productId")
public ProductResponse getProduct(Long productId) {
    // ...
}
```

검색 조건이 여러 개인 목록 조회라면 key 설계를 더 신중하게 해야 한다.

```java
@Cacheable(
        cacheNames = "productSearch",
        key = "#condition.category + ':' + #condition.page + ':' + #condition.size"
)
public Page<ProductResponse> searchProducts(ProductSearchCondition condition) {
    // ...
}
```

Cache key가 부정확하면 서로 다른 요청이 같은 Cache 값을 공유하는 문제가 생길 수 있다.

예를 들어 사용자별 권한이 다른 데이터에 단순히 `productId`만 key로 사용하면,
권한이 다른 사용자에게 같은 응답이 내려갈 수 있다.

```text
Cache key에는 결과를 달라지게 만드는 조건이 반드시 포함되어야 한다.
```

---

## 8. 로컬 캐시란?

`로컬 캐시(Local Cache)`는 애플리케이션 서버 내부 메모리에 Cache를 저장하는 방식이다.

대표적으로 `Caffeine`, `Ehcache`, `ConcurrentMapCache` 등이 있다.

구조는 다음과 같다.

```text
Client
  -> Application Server
      -> Local Cache
      -> DB
```

로컬 캐시는 같은 서버 안에서 바로 접근하므로 매우 빠르다.
네트워크 호출이 없고 구현도 비교적 단순하다.

장점은 다음과 같다.

- 접근 속도가 빠르다.
- 네트워크 장애 영향을 받지 않는다.
- 구조가 단순하다.
- 작은 규모의 서비스나 단일 서버 환경에서 적용하기 쉽다.

단점은 다음과 같다.

- 서버마다 Cache 데이터가 다를 수 있다.
- 서버를 여러 대로 늘리면 Cache 정합성 관리가 어렵다.
- 서버 재시작 시 Cache가 사라진다.
- 서버 메모리를 사용하므로 메모리 관리가 필요하다.

예를 들어 서버가 3대라면 각 서버가 자기 메모리에 Cache를 따로 가진다.
1번 서버의 Cache를 지워도 2번, 3번 서버의 Cache는 그대로 남아 있을 수 있다.

```text
로컬 캐시의 핵심 문제 = 서버 간 Cache 불일치
```

---

## 9. 분산 캐시란?

`분산 캐시(Distributed Cache)`는 애플리케이션 서버 외부의 별도 Cache 저장소를 여러 서버가 함께 사용하는 방식이다.

대표적으로 `Redis`, `Memcached`가 있다.

구조는 다음과 같다.

```text
Client
  -> Application Server 1
  -> Application Server 2
  -> Application Server 3
      -> Distributed Cache
      -> DB
```

분산 캐시는 여러 애플리케이션 서버가 같은 Cache 저장소를 바라본다.
따라서 서버를 여러 대 운영할 때 Cache 정합성을 맞추기 쉽다.

장점은 다음과 같다.

- 여러 서버가 같은 Cache를 공유할 수 있다.
- 서버를 증설해도 Cache 구조를 유지하기 쉽다.
- 서버 재시작과 Cache 데이터가 분리된다.
- Redis 자료구조, TTL, Pub/Sub 같은 기능을 활용할 수 있다.

단점은 다음과 같다.

- 네트워크 호출 비용이 발생한다.
- Redis 같은 별도 인프라 운영이 필요하다.
- Cache 서버 장애 대응이 필요하다.
- 직렬화, 역직렬화 비용이 발생한다.
- 잘못 사용하면 DB 부하 대신 Redis 부하가 병목이 될 수 있다.

분산 캐시는 운영 환경에서 서버가 여러 대이거나,
Cache 정합성이 중요한 경우에 많이 사용된다.

---

## 10. 로컬 캐시와 분산 캐시 비교

| 구분 | 로컬 캐시 | 분산 캐시 |
| --- | --- | --- |
| 저장 위치 | 애플리케이션 서버 메모리 | 외부 Cache 서버 |
| 속도 | 매우 빠름 | 네트워크 비용 존재 |
| 서버 확장 | 서버별 Cache 분리 | 여러 서버가 Cache 공유 |
| 정합성 | 서버 간 불일치 가능 | 상대적으로 관리 쉬움 |
| 장애 영향 | 앱 서버 재시작 시 사라짐 | Cache 서버 장애 고려 필요 |
| 운영 복잡도 | 낮음 | 높음 |
| 대표 기술 | Caffeine, Ehcache | Redis, Memcached |

단순히 분산 캐시가 항상 좋은 것은 아니다.
서비스 규모와 데이터 특성에 따라 선택해야 한다.

---

## 11. 실무에서 Cache 선택 기준

Cache 선택은 다음 기준으로 판단할 수 있다.

### 단일 서버이거나 작은 규모라면

단일 서버 MVP나 관리자 도구처럼 트래픽이 크지 않은 서비스라면 로컬 캐시로도 충분할 수 있다.

```text
단일 서버
변경이 적은 기준 데이터
정합성 요구가 낮은 조회 데이터
짧은 TTL로 관리 가능한 데이터
```

이런 경우에는 `Caffeine` 같은 로컬 캐시를 사용하면 구조가 단순하고 성능도 좋다.

### 여러 서버가 같은 데이터를 봐야 한다면

운영 환경에서 서버가 여러 대이고, 사용자 요청이 로드밸런서를 통해 여러 서버로 분산된다면 분산 캐시를 우선 고려한다.

```text
다중 서버 운영
서버 간 Cache 정합성 필요
로그인 세션, 인증 관련 임시 데이터
외부 API 응답 공유
여러 인스턴스가 같은 조회 결과를 재사용해야 하는 경우
```

이런 경우에는 Redis 같은 분산 캐시가 적합하다.

### 데이터 정합성이 매우 중요하다면

결제 금액, 재고 차감, 포인트 잔액처럼 정확성이 중요한 데이터는 Cache만 믿고 처리하면 위험하다.

이런 데이터는 DB 트랜잭션과 Lock, 제약 조건을 기준으로 처리하고,
Cache는 조회 성능 개선용으로 보조적으로 사용해야 한다.

```text
정합성이 중요한 쓰기 로직 = DB 기준
조회 성능 최적화 = Cache 보조
```

### Cache 만료 전략을 정할 수 있어야 한다면

Cache를 사용할 때는 TTL(Time To Live)을 함께 고민해야 한다.

```java
@Bean
public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
    RedisCacheConfiguration configuration = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10));

    return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(configuration)
            .build();
}
```

TTL이 너무 길면 오래된 데이터가 오래 남고,
너무 짧으면 Cache hit율이 낮아져 효과가 줄어든다.

따라서 데이터 변경 빈도, 정합성 요구 수준, 트래픽 패턴을 보고 TTL을 정해야 한다.

---

## 12. 실무에서 주의할 점

### Cache Stampede

동일한 Cache가 동시에 만료되면 많은 요청이 한꺼번에 DB로 몰릴 수 있다.
이를 `Cache Stampede`라고 한다.

예를 들어 인기 상품 목록 Cache가 만료되는 순간,
수백 개 요청이 동시에 DB를 조회하면 장애로 이어질 수 있다.

대응 방법은 다음과 같다.

- TTL에 약간의 랜덤 값을 섞는다.
- Lock을 사용해 한 요청만 Cache를 재생성하게 한다.
- 만료 전 미리 Cache를 갱신한다.

### Cache Penetration

존재하지 않는 데이터를 계속 조회하는 요청이 들어오면 Cache에 저장되지 않고 DB까지 계속 도달할 수 있다.

예를 들어 존재하지 않는 상품 ID를 반복 조회하면 매번 DB 조회가 발생한다.

이 경우 짧은 TTL로 빈 결과를 Cache하거나,
요청 검증과 rate limit을 적용하는 방법을 고려할 수 있다.

### Transaction과 Cache 갱신 시점

DB 트랜잭션이 아직 커밋되지 않았는데 Cache가 먼저 갱신되면,
트랜잭션 롤백 시 DB와 Cache가 불일치할 수 있다.

쓰기 로직에서 Cache를 갱신하거나 제거할 때는 트랜잭션 커밋 이후 시점을 고려해야 한다.

```text
DB 변경 성공
트랜잭션 커밋
Cache 갱신 또는 제거
```

Spring에서는 `TransactionSynchronization`이나 이벤트 기반 처리로 커밋 이후 Cache 무효화를 구성할 수 있다.

---

## 13. 정리

`@Cacheable`, `@CachePut`, `@CacheEvict`는 모두 Cache를 다루지만 목적이 다르다.

```text
@Cacheable = 조회 결과를 저장하고 재사용한다.
@CachePut = 메서드를 실행한 뒤 결과로 Cache를 갱신한다.
@CacheEvict = 더 이상 신뢰할 수 없는 Cache를 제거한다.
```

로컬 캐시는 빠르고 단순하지만 서버 간 정합성이 약하다.
분산 캐시는 여러 서버가 Cache를 공유할 수 있지만 별도 인프라와 장애 대응이 필요하다.

실무에서는 다음 기준으로 선택하는 것이 좋다.

```text
단일 서버, 변경 적음, 단순 조회 = 로컬 캐시
다중 서버, 공유 필요, 운영 확장 = 분산 캐시
정합성 중요 = DB를 기준으로 하고 Cache는 보조로 사용
```

Cache는 성능을 높이는 강력한 도구지만,
잘못 사용하면 오래된 데이터, 서버 간 불일치, 장애 전파의 원인이 될 수 있다.

따라서 Cache를 적용할 때는 항상 다음 질문을 함께 해야 한다.

```text
무엇을 Cache할 것인가?
언제 만료할 것인가?
데이터가 변경되면 어떻게 무효화할 것인가?
운영 중 Cache 장애가 나면 어떻게 대응할 것인가?
```

이 네 가지 질문에 답할 수 있을 때 Cache는 단순한 성능 최적화를 넘어,
운영 가능한 시스템 설계의 중요한 도구가 된다.
