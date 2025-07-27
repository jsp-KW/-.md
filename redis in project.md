
# 🚀 Redis 연동 도입기 (Spring Boot + RedisTemplate)

## ✅ 도입 배경

Spring Boot 기반 OpenBanking 프로젝트에서, 인증 코드와 예약 이체 대기열, 알림 등 실시간 처리가 필요한 기능이 증가하며 기존 RDBMS만으로는 한계 발생:

- ❌ 인증 코드 TTL 직접 삭제 쿼리 필요
- ❌ 예약 이체 DB polling 방식의 부하
- ❌ 동일 데이터 반복 조회 시 비효율

➡️ 실시간성, TTL 관리, 캐싱 등 개선을 위해 Redis 도입 결정

---

## 🧭 Redis 연동 절차 요약

1. Docker로 Redis 컨테이너 실행
2. `build.gradle`에 Redis 의존성 추가
3. `application.yml`에 Redis 설정
4. `RedisTemplate` 수동 Bean 등록
5. 테스트용 Service/Controller 작성
6. `/redis/**` Security 허용
7. Postman으로 저장/조회 테스트
8. Redis 도입 전후 비교

---

## 🐳 Docker Desktop으로 Redis 실행

```bash
docker pull redis
docker run -d -p 6379:6379 --name redis redis
```

---

## 📦 build.gradle 의존성 추가

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'org.apache.commons:commons-pool2'
}
```

---

## ⚙️ application.yml 설정

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
```

> ℹ️ `spring.redis`는 deprecated, `spring.data.redis` 사용 권장

---

## 🔧 RedisTemplate 수동 Bean 등록

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

## 🧪 테스트용 Service & Controller

### 📄 RedisTestService.java

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

### 📄 RedisTestController.java

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

## 🔐 Spring Security 설정

```java
http
  .csrf(AbstractHttpConfigurer::disable)
  .authorizeHttpRequests(auth -> auth
    .requestMatchers("/redis/**").permitAll()
    .anyRequest().authenticated()
  );
```

---

## 📮 Postman 테스트 결과

| Method | Endpoint                     | 설명           |
|--------|-------------------------------|----------------|
| POST   | `/redis/save`                | Redis에 값 저장 |
| GET    | `/redis/get`                 | Redis에서 조회 |

📤 응답 결과:

```json
"Hi redis~!"
```

✅ **200 OK**, 응답 속도 약 **60~80ms**

---

## ❗ RedisTemplate 주입 오류 발생

```text
No qualifying bean of type 'RedisTemplate<String, Object>' available
```

### 🔍 원인
- Spring Boot는 기본적으로 `RedisTemplate<String, Object>`를 자동 등록하지 않음

### ✅ 해결
- `RedisConfig.java`에서 **직접 Bean 등록** 필요 (위 코드 참고)

---

