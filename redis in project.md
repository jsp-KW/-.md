# Redis 연동 도입기 (Spring Boot + RedisTemplate)

##배경

Spring Boot 기반 OpenBanking 프로젝트에서, 인증 코드와 예약 이체 대기열, 알림 등 **실시간 처리가 필요한 기능**이 생기면서 RDBMS만으로는 한계가 있었다.

- 인증 코드 TTL 관리 불편
- 예약 이체를 주기적 DB polling으로 처리
- 캐시 없이 반복적인 DB 조회 발생

이러한 문제를 해결하고자 Redis를 도입하게 되었다.

<img width="1262" height="450" alt="image" src="https://github.com/user-attachments/assets/cdd78910-bd04-4797-93f7-1f3a7e3f8da7" />

---

##  RedisTemplate 주입 오류 발생

```text
No qualifying bean of type 'RedisTemplate<String, Object>' available

## 원인

- Spring Boot는 기본적으로 `RedisTemplate<String, Object>`를 자동으로 Bean으로 등록하지 않음
- 따라서 수동으로 Bean 등록이 필요함

---

## 해결 방법

직접 `RedisConfig.java` 파일을 생성하여 Bean 등록

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
