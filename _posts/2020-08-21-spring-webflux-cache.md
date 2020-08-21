---
layout: post
title: Spring Webflux Cache       # 페이지 최상단 타이틀
author: dreamchaser                  # authors.yml 에 작성한 본인의 이
thumbnail: thumbnail/webflux.jpg    # thumbnail
permalink: /staging-webflux/        # staging url 로 공유할 링크. 원하는대로 입력
staging: true                       # staging 페이지의 경우, staging true 옵션을 !!!무조건!!! 줄것.
---
# 개요
--------------
<br>

스프링 웹플럭스와 Reactor를 사용하여 웹서버를 개발할 때 고민이 되는 부분이 있습니다.
바로 **'캐시'**입니다.
기존 mvc 모델에서는 스프링 캐시를 사용하여 캐싱을 쉽게 할 수 있었지만, 웹플럭스 모델에서는 스프링 캐시를 이용하여 캐싱을 할 수 없습니다. 
다음과 같이 `@Cacheable` 어노테이션을 이용하여 캐싱을 한다고 하면,

<br>
<br>

### mvc 모델
```java
@Cacheable
public List<Integer> list() {
	// Integer List를 리턴
}
```
<br>

### 웹플럭스 모델
```java
@Cacheable
public Mono<List<Integer>> listMono() {
	// Mono 체인을 리턴
}
```

mvc 모델에서는 캐시하고자 하는 값 자체를 캐싱할 수 있지만, 웹플럭스 모델에서는 위처럼 `Mono` 객체가 캐싱됩니다. 
리액터의 `Mono/Flux`는 구독자가 subscribe()를 하기 이전까지는 내부 로직이 실제로 실행되지 않고 조립 단계만 수행합니다.
따라서 위 예제에서 캐싱된 `Mono` 객체는 우리가 실제로 캐시하고자 하는 값이 아니라 `Mono` 체인을 조립한 객체일 뿐입니다. 
이처럼 웹플럭스 모델에서 스프링 캐시를 사용하려면 다른 방법이 필요합니다.

<br>

> [참고]
>
> 리액터의 reactive stream은 아래와 같이 3단계의 생명 주기를 가지고 있습니다.
>
> 1. 조립 단계 (Assembly-time)
> 2. 구독 단계 (Subscription-time)
> 3. 런타임 단계 (Runtime)
>
> 보다 자세한 내용은 [이 글](https://dreamchaser3.tistory.com/16)을 참고하시면 좋습니다.

<br>
<br>
<br>

# 접근법
--------------
<br>

## 1. Blocking Way
```java
@Cacheable
public List<Integer> worstWay() {
        // Mono 체인의 끝에 block()을 붙임
	return Mono...block()
}
```
가장 단순하게 생각해볼 수 있는 방식은 `Mono/Flux` 뒤에 `block()` 메소드를 호출해서 블로킹 방식으로 데이터를 꺼내온 다음에 캐시하는 방법일 것입니다.
하지만 이 방식은 웹플럭스 모델의 논블로킹 리액티브 방식을 완전히 훼손하는 것으로, 절대 해서는 안되는 방식입니다.

결국 쟁점은 논블로킹하고 리액티브한 방식으로 추상화된 스프링 캐시의 이점을 살리면서, 리액터와 연결지을 수 있을까인데요. 
이를 해결하기 위해서 다음과 같은 2가지 방식을 생각해 볼 수 있습니다. 
앞으로 예제에 대한 모든 설명은 편의상 `Mono`를 기준으로 하려고 합니다. (`Flux`도 같기 때문에)

참고: [https://stackoverflow.com/questions/48156424/spring-webflux-and-cacheable-proper-way-of-caching-result-of-mono-flux-type](https://stackoverflow.com/questions/48156424/spring-webflux-and-cacheable-proper-way-of-caching-result-of-mono-flux-type)

(아래 예제도 위 글에서 가져왔습니다.)

<br>

## 2. Hack Way
스프링의 `@Cacheable`을 여전히 사용하면서 Mono/Flux의 `cache()` 메소드를 사용하는 방식입니다. 
즉, 캐시된 Mono의 레퍼런스를 캐시하는 것입니다.
```kotlin
@Repository
interface TaskRepository : ReactiveMongoRepository<Task, String>

@Service
class TaskService(val taskRepository: TaskRepository) {

    @Cacheable("tasks")
    fun get(id: String): Mono<Task> = taskRepository.findById(id).cache()
}
```

`Mono`의 [cache()](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#cache--)를 사용하면, 
실제 런타임 단계에서 수행될 때 `cache()` 직전 단계까지 수행된 source를 들고 있는 Mono 객체를 메모리에 일정 시간 올려두고 
기존 단계의 로직을 다시 수행하지 않고도 계속해서 재사용할 수 있습니다. 
그리고 `@Cacheable` 어노테이션은 이러한 Mono의 레퍼런스를 캐시하기 때문에, 캐시된 Mono 레퍼런스를 참조하여 구독하게 되고, 우리가 원하던 캐시의 효과를 얻을 수 있게 됩니다.

그러나 위와 같은 방식은 그 이름에서도 알 수 있듯이, 'hack' 방식이고 정석적인 방식이 아니기 때문에 다음과 같은 문제가 있습니다. 
`@Cacheable`을 통해 캐시한 값이 실제 값이 아닌 캐시된 Mono의 레퍼런스이기 때문에 스프링 캐시 설정으로 관리하던 캐시의 설정을 적용할 수가 없습니다. 
가장 대표적인 문제점이 ttl을 컨트롤할 수가 없다는 점인데, 스프링 캐시로 설정한 ttl은 캐시된 Mono의 레퍼런스의 캐시에만 적용되고, 
실제로 캐시된 Mono는 갱신되지 않는다는 문제가 있습니다. 
이러한 문제를 해결하기 위해서, `Mono`의 [cache(ttl)](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#cache-java.time.Duration-) 
메소드를 사용하여 ttl을 지정해줄 수 있지만 스프링 캐시로 설정한 ttl과 Mono.cache()의 ttl을 아무리 동일하게 맞춘다 할지라도 실행되는 시점이 서로 다르기 때문에 오차가 생길 수 밖에 없습니다.

<br>

## 3. Reactor Addons Way
Reactor Addons 라이브러리를 사용하면 리액티브 방식으로 `Mono`와 `Flux`를 캐시할 수 있습니다.
```kotlin
@Repository
interface TaskRepository : ReactiveMongoRepository<Task, String>

@Service
class TaskService(val taskRepository: TaskRepository) {

    @Autowire
    CacheManager manager;


    fun get(id: String): Mono<Task> = CacheMono.lookup(reader(), id)
                                               .onCacheMissResume(() -> taskRepository.findById(id))
                                               .andWriteWith(writer());

    fun reader(): CacheMono.MonoCacheReader<String, Task> = key -> Mono.<Signal<Task>>justOrEmpty((Signal) manager.getCache("tasks").get(key).get())
    fun writer(): CacheMono.MonoCacheWriter<String, Task> = (key, value) -> Mono.fromRunnable(() -> manager.getCache("tasks").put(key, value));
```

위 예제처럼 [CacheMono](https://projectreactor.io/docs/extra/snapshot/api/reactor/cache/CacheMono.html)와 
[CacheFlux](https://projectreactor.io/docs/extra/release/api/reactor/cache/CacheFlux.html)를 사용하여 
편리하게 스프링의 CacheManager를 리액티브 스트림 내부 로직에 녹아들게 할 수 있습니다. 
이 방식을 사용하면 실제로 스프링의 CacheManager를 이용하여 값을 `put() / get()`할 수 있습니다. 
즉, 캐시 추상화라는 스프링 캐시의 이점을 살릴 수 있고, 스프링 캐시를 리액티브하게 리액터와 연동해주는 방식이라고 할 수 있습니다.

<br>
<br>

# 스프링 + 어노테이션 방식으로 캐시하기
--------------
Reactor Addons 라이브러리를 사용하는 방식이 리액티브 방식으로 캐시하는 좋은 방법이지만, 
스프링 캐시를 사용하는 데 익숙해진 사람들에게는 `@Cacheable` 어노테이션의 사용법이 익숙하고 편리하며, 가독성도 좋습니다.
따라서 다음과 같이 새로운 어노테이션을 만들어, 스프링의 `@Cacheable`과 동일한 방식으로 캐시를 하는 방식을 구현해 볼 수 있습니다.

스프링 캐시 셋팅은 이미 되어 있다는 전제 하에 시작합니다.

<br>

## 1. Reactor Addons + spring boot aop starter 라이브러리 import
```groovy
implementation("io.projectreactor.addons:reactor-extra:3.3.0.RELEASE")
implementation("org.springframework.boot:spring-boot-starter-aop:2.3.0.RELEASE")
```

<br>

## 2. ReactiveCacheManager 만들기
Reactor Addons의 `CacheMono/CacheFlux`는 아래와 같은 구조로 되어있습니다.

```java
CacheMono.lookup(reader(), id)// reader()를 통해 캐시가 있는지 찾는다. 있으면 그 값을 리턴
    // 캐시가 없으면, 원본 데이터를 가져온다.
    .onCacheMissResume(() -> taskRepository.findById(id))
    // writer()를 통해 캐시를 넣는다.
    .andWriteWith(writer());
```
<br>

스프링의 `CacheManager`를 주입 받아서 다음과 같이 Reactor Addons 인터페이스에 맞게 로직을 구현하였습니다.


```java
@Component
public class ReactiveCacheManager {
    private CacheManager cacheManager;

    @Autowired
    public ReactiveCacheManager(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    public <T> Mono<T> findCachedMono(String cacheName, Object key, Supplier<Mono<T>> retriever, Class<T> classType) {
        Cache cache = cacheManager.getCache(cacheName);
        return CacheMono
                .lookup(k -> {
                    T result = cache.get(k, classType);
                    return Mono.justOrEmpty(result).map(Signal::next);
                }, key)
                .onCacheMissResume(Mono.defer(retriever))// retriever(원 메소드)의 수행을 지연시키기 위해 defer()로 감쌌음
                .andWriteWith((k, signal) -> Mono.fromRunnable(() -> {
                    if (!signal.isOnError()) {
                    	cache.put(k, signal.get());
                    }
                }));
    }

    public <T> Flux<T> findCachedFlux(String cacheName, Object key, Supplier<Flux<T>> retriever) {
        Cache cache = cacheManager.getCache(cacheName);
        return CacheFlux
                .lookup(k -> {
                    List<T> result = cache.get(k, List.class);
                    return Mono.justOrEmpty(result)
                            .flatMap(list -> Flux.fromIterable(list).materialize().collectList());
                }, key)
                .onCacheMissResume(Flux.defer(retriever))
                .andWriteWith((k, signalList) -> Flux.fromIterable(signalList)
                        .dematerialize()
                        .collectList()
                        .doOnNext(list -> {
                            cache.put(k, list);
                        })
                        .then());
    }
  }
```
<br>

## 3. 어노테이션 만들기
어노테이션은 다음과 같이 심플하게 만들어 볼 수 있습니다.

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ReactorCacheable {

    String name();
}
```

<br>

## 4. Aspect 만들기
이제 `@ReactorCacheable` 어노테이션을 붙인 메소드에 위에서 만든 `ReactiveCacheManager`의 캐시 로직이 동작할 수 있도록 Aspect를 만들어줍니다.

```java
@Aspect
@Component
public class ReactorCacheAspect {
    private final ReactiveCacheManager reactiveCacheManager;

    @Autowired
    public ReactorCacheAspect(ReactiveCacheManager reactiveCacheManager) {
        this.reactiveCacheManager = reactiveCacheManager;
    }

    @Pointcut("@annotation(reactor.cache.ReactorCacheable)")
    public void pointcut() {
    }

    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();

        ParameterizedType parameterizedType = (ParameterizedType) method.getGenericReturnType();
        Type rawType = parameterizedType.getRawType();

        // 어노테이션이 붙은 메소드의 리턴 타입이 Mono/Flux가 아니면 에러를 발생
        if (!rawType.equals(Mono.class) && !rawType.equals(Flux.class)) {
            throw new IllegalArgumentException("The return type is not Mono/Flux. Use Mono/Flux for return type. method: " + method.getName());
        }
        ReactorCacheable reactorCacheable = method.getAnnotation(ReactorCacheable.class);
        String cacheName = reactorCacheable.name();
        Object[] args = joinPoint.getArgs();
        
        // joinpoint.proceed()를 무조건 try/catch로 묶어줘야 해서 가독성을 위해 ThrowingSupplier로 감쌌음
        ThrowingSupplier retriever = () -> joinPoint.proceed(args);
        // 리턴타입이 Mono면
        if (rawType.equals(Mono.class)) {
            Type returnTypeInsideMono = parameterizedType.getActualTypeArguments()[0];
            Class<?> returnClass = ResolvableType.forType(returnTypeInsideMono).resolve();
            return reactiveCacheManager
                    .findCachedMono(cacheName, generateKey(args), retriever, returnClass)
                    .doOnError(e -> log.error("Failed to processing mono cache. method: " + method.getName(), e));
        }
        // 리턴타입이 Flux면 
        else {
            return reactiveCacheManager
                    .findCachedFlux(cacheName, generateKey(args), retriever)
                    .doOnError(e -> log.error("Failed to processing flux cache. method: " + method.getName(), e));
        }
    }
    
    // argument들의 조합으로 cache key를 생성
    private String generateKey(Object... objects) {
        return Arrays.stream(objects)
                .map(obj -> obj == null ? "" : obj.toString())
                .collect(Collectors.joining(":"));
    }
}
```
<br>

[참고] ThrowingSupplier

```java
@FunctionalInterface
public interface ThrowingSupplier<T> extends Supplier<T> {
    @Override
    default T get() {
        try {
            return getThrows();
        } catch (Throwable th) {
            throw new RuntimeException(th);
        }
    }

    T getThrows() throws Throwable;
}
```

<br>

## 5. 캐시하고자 하는 메소드에 어노테이션 붙이기
이제 캐시하고자 하는 `Mono/Flux`를 리턴하는 메소드에 `@ReactorCacheable` 어노테이션을 붙이면 됩니다.

```java
// 예시
@ReactorCacheable(name = "stringList")
public Mono<List<String>> selectStringList(String paramA, String paramB) {
	return mapper.selectStringList();
}
```

<br>
<br>

# 유의 사항
--------------
위와 같은 방식으로 리액티브하게 스프링 캐시를 사용하더라도, 스프링 캐시는 추상화된 캐시 인터페이스라는 점을 유의해야 합니다.
즉, `cache.get() / cache.put()`을 실행하면, 실제로 수행되는 로직은 스프링의 Cache 인터페이스를 구현하고 있는 
캐시 라이브러리(ex. Caffeine Cache, Redis, etc..)의 구현체 로직입니다. 
내부 구현체는 여전히 블로킹 모델이기 때문에, 스레드가 Network나 Disk I/O 작업으로 블로킹이 될 여지가 있는지 체크해봐야 합니다.

<br>

## 1. In-Memory 캐시 사용하는 경우 (ex. Caffeine cache)
인 메모리 캐시의 경우, I/O 블로킹이 문제가 되지 않으므로 **'4. Aspect 만들기'** 방식 그대로 사용해도 됩니다.

```java
	... (중략)
        // 리턴타입이 Mono면
        if (rawType.equals(Mono.class)) {
            Type returnTypeInsideMono = parameterizedType.getActualTypeArguments()[0];
            Class<?> returnClass = ResolvableType.forType(returnTypeInsideMono).resolve();
            return reactiveCacheManager
                    .findCachedMono(cacheName, generateKey(args), retriever, returnClass)
                    .doOnError(e -> log.error("Failed to processing mono cache. method: " + method.getName(), e));
        }
        // 리턴타입이 Flux면 
        else {
            return reactiveCacheManager
                    .findCachedFlux(cacheName, generateKey(args), retriever)
                    .doOnError(e -> log.error("Failed to processing flux cache. method: " + method.getName(), e));
        }
        ...
```
<br>

## 2. Network나 Disk I/O 작업이 있는 캐시를 사용하는 경우 (ex. 외부 Redis cache)
외부 Redis 서버에 캐시하거나, 디스크 캐시를 사용하는 경우는 아래처럼 **'4. Aspect 만들기'** 코드에서 `subscribeOn()`을 추가하여, 
현재 스레드가 블로킹되지 않도록 별도 스레드에 위임하는 작업이 필요합니다.

```java
	... (중략)
        // 리턴타입이 Mono면
        if (rawType.equals(Mono.class)) {
            Type returnTypeInsideMono = parameterizedType.getActualTypeArguments()[0];
            Class<?> returnClass = ResolvableType.forType(returnTypeInsideMono).resolve();
            return reactiveCacheManager
                    .findCachedMono(cacheName, generateKey(args), retriever, returnClass)
                    .doOnError(e -> log.error("Failed to processing mono cache. method: " + method.getName(), e))
                    // I/O 전용 스케쥴러에 위임
                    .subscribeOn(Schedulers.boundedElastic());
        }
        // 리턴타입이 Flux면 
        else {
            return reactiveCacheManager
                    .findCachedFlux(cacheName, generateKey(args), retriever)
                    .doOnError(e -> log.error("Failed to processing flux cache. method: " + method.getName(), e))
                    // I/O 전용 스케쥴러에 위임
                    .subscribeOn(Schedulers.boundedElastic());
        }
        ...
```
subscribeOn()을 이용한 블로킹 로직 감싸는 것과 관련된 내용은 [리액터 공식 레퍼런스](https://projectreactor.io/docs/core/release/reference/#faq.wrap-blocking)를 참고하면 좋습니다.

<br>

### 덧
위에서 언급한 바와 같이, 스프링 캐시를 아무리 리액티브하게 구현한다고 하더라도, 
각 캐시 라이브러리 구현체가 이를 리액티브하게 구현하지 않으면 모든 레이어가 완전히 리액티브하지는 못하게 됩니다. 
이러한 이유에서 스프링에서는 리액티브 캐시를 지원하지 않고 있습니다.

관련 이슈: [https://github.com/spring-projects/spring-framework/issues/17920](https://github.com/spring-projects/spring-framework/issues/17920)

<br>
<br>

# 참고
--------------
[https://stackoverflow.com/questions/48156424/spring-webflux-and-cacheable-proper-way-of-caching-result-of-mono-flux-type](https://stackoverflow.com/questions/48156424/spring-webflux-and-cacheable-proper-way-of-caching-result-of-mono-flux-type)

[https://projectreactor.io/docs/extra/snapshot/api/reactor/cache/CacheMono.html](https://projectreactor.io/docs/extra/snapshot/api/reactor/cache/CacheMono.html)

[https://github.com/spring-projects/spring-framework/issues/17920](https://github.com/spring-projects/spring-framework/issues/17920)
