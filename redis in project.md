
# Redis 연동 도입기 (Spring Boot + RedisTemplate)

## 도입 배경

Spring Boot 기반 OpenBanking 프로젝트에서, 인증 코드와 예약 이체 대기열, 알림 등 실시간 처리가 필요한 기능이 증가하며 기존 RDBMS만으로는 한계 발생:

- ❌ 인증 코드 TTL 직접 삭제 쿼리 필요
- ❌ 예약 이체 DB polling 방식의 부하
- ❌ 동일 데이터 반복 조회 시 비효율

➡️ 실시간성, TTL 관리, 캐싱 등 개선을 위해 Redis 도입 결정

---

##  Redis 연동 절차 요약

1. Docker로 Redis 컨테이너 실행
2. `build.gradle`에 Redis 의존성 추가
3. `application.yml`에 Redis 설정
4. `RedisTemplate` 수동 Bean 등록
5. 테스트용 Service/Controller 작성
6. `/redis/**` Security 허용
7. Postman으로 저장/조회 테스트
8. Redis 도입 전후 비교

---

##  Docker Desktop으로 Redis 실행

```bash
docker pull redis
docker run -d -p 6379:6379 --name redis redis
```
<img width="975" height="392" alt="image" src="https://github.com/user-attachments/assets/39b80e07-9c38-4935-aa8d-3f1159b8e2fa" />

---

##  build.gradle 의존성 추가

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'org.apache.commons:commons-pool2'
}
```

---

##  application.yml 설정

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

>  `spring.redis`는 deprecated, `spring.data.redis` 사용 권장

---

##  RedisTemplate 수동 Bean 등록

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

---

## 테스트용 Service & Controller

###  RedisTestService.java

```java
@Service
@RequiredArgsConstructor
public class RedisTestService {
    private final RedisTemplate<String, Object> redisTemplate;

    public void saveTest() {
        redisTemplate.opsForValue().set("myKey", "Hi redis~!");
    }

    public String getTest() {
        return (String) redisTemplate.opsForValue().get("myKey");
    }
}
```

###  RedisTestController.java

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/redis")
public class RedisTestController {

    private final RedisTestService redisTestService;

    @PostMapping("/save")
    public ResponseEntity<Void> save() {
        redisTestService.saveTest();
        return ResponseEntity.ok().build();
    }

    @GetMapping("/get")
    public ResponseEntity<String> get() {
        return ResponseEntity.ok(redisTestService.getTest());
    }
}
```

---

##  Spring Security 설정

```java
http
  .csrf(AbstractHttpConfigurer::disable)
  .authorizeHttpRequests(auth -> auth
    .requestMatchers("/redis/**").permitAll()
    .anyRequest().authenticated()
  );
```

---

##  Postman 테스트 결과

| Method | Endpoint                     | 설명           |
|--------|-------------------------------|----------------|
| POST   | `/redis/save`                | Redis에 값 저장 |
| GET    | `/redis/get`                 | Redis에서 조회 |

📤 응답 결과:

```json
"Hi redis~!"
```

---

##  RedisTemplate 주입 오류 발생

```text
No qualifying bean of type 'RedisTemplate<String, Object>' available
```

###  원인
- Spring Boot는 기본적으로 `RedisTemplate<String, Object>`를 자동 등록하지 않음

###  해결
- `RedisConfig.java`에서 **직접 Bean 등록** 필요 (위 코드 참고)

###  문제 상황

Spring Boot에서 `RedisTemplate<String, String>`을 직접 등록하고 주입하려 했더니 다음과 같은 에러 발생:

```
No qualifying bean of type 'RedisTemplate<String, String>' available: expected single matching bean but found 2
```

###  원인 분석

- Spring Boot는 내부적으로 기본 `StringRedisTemplate`을 자동 등록함  
- 여기에 내가 수동으로 `RedisTemplate<String, String>`을 등록하면서 **같은 타입의 Bean이 2개** 생성됨  
- Spring은 어떤 Bean을 주입해야 할지 몰라 `NoUniqueBeanDefinitionException` 발생

###  해결 방법

####  방법 1: `@Primary` 사용
```java
@Configuration
public class RedisConfig {

    @Bean
    @Primary  // 우선 순위 명시
    public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        return template;
    }
}
```

####  방법 2: `@Qualifier`로 명시
```java
@Bean
@Qualifier("stringRedisTemplate")
public RedisTemplate<String, String> stringRedisTemplate(...) {
    ...
}

@Qualifier("stringRedisTemplate")
private final RedisTemplate<String, String> redisTemplate;
```
---


### 깨달은 점
- 기존의 backend 서버 구축시 단일 RDBMS 만으로는 실시간성, TTL 기반 데이터처리, 고속 데이터 접근 등의 실무에서(?) 자주 발생하여 고려해야하는 문제점들을 보완하긴 어렵다는 점을 깨달았습니다.
  
- 또한 여러 증권,은행 앱을 사용하다보면 일정 시간이 지난 이후 재로그인을 하라는 요청을 경험한 기억을 떠올렸고 이런 기능을 프로젝트에 적용하기 위해서 도입해야할 것을 공부하던중 로그인 세션 만료 알림, 인증코드 만료등 일시성 데이터처리에 Redis가 많이 활용된다는 점을 발견하였습니다.
  
- 이를 계기로 Redis에 대해 직접 학습하고 도입해보며 RedisTemplate을 직접 Bean으로 등록해 발생한 충돌 이슈와 해결법에 대해서 알게 되었습니다.
  
- Spring 은 IOC (제어의 역전) 을 통해 객체의 생성과 객체의 생명주기를 직접 관리합니다. 하지만 동일한 타입의 Bean이 여러개 존재할 경우, 어떤 Bean 을 주입해야 할지 결정할 수 없어 NoUniqueBeanDefinitionException 에러가 발생합니다.
  
- 이 에러 발생을 통해 Spring에서는 @Primary, @Qualifier 를 활용하여 의존성 주입의 대상인 Bean을 명확히 지정해야함을 깨달았습니다.

