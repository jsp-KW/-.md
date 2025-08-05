
# JWT Refresh Token 재발급 시 `ROLE_null` 문제 해결 기록


##  문제 요약

Spring Boot 기반의 JWT 인증 시스템에서,  
AccessToken 만료 시 RefreshToken을 통해 새로운 AccessToken을 발급받는 과정에서 다음과 같은 문제가 발생:

- 재발급된 AccessToken에 사용자 역할(`role`)이 `null`로 들어감
- 이로 인해 `ROLE_null`이 부여되어 `403 Forbidden` 발생
- 로그:  
  ```
  jwt 필터 실행됨  
  사용자 이메일: a@example.com  
  추출된 role: null  
  부여된 권한: [ROLE_null]
  ```

---

## 원인 분석

1. 로그인 시 `accessToken`, `refreshToken` 모두 생성:
   ```java
   String accesstoken = jwtUtil.createToken(request.getEmail(), role);
   String refreshToken = jwtUtil.createRefreshToken(request.getEmail());
   ```

2. 문제는 `createRefreshToken()` 메서드 내부에서 `role` 정보를 넣지 않음:
   ```java
   public String createRefreshToken(String email) {
       return Jwts.builder()
           .setSubject(email)
           .setIssuedAt(new Date())
           .setExpiration(new Date(System.currentTimeMillis() + 604800000)) // 7일
           .signWith(privateKey, SignatureAlgorithm.RS256)
           .compact();
   }
   ```

3. `/api/auth/refresh` 호출 시:
   - Redis에 저장된 RefreshToken은 유효
   - 하지만 RefreshToken 내에는 role 정보가 없어,  
     `getUserRole(refreshToken)` 호출 시 `null` 반환 → 문제 발생

---

## 해결 방법

### 1. RefreshToken에도 `role` 포함

`JwtTokenProvider.java` 수정:
```java
public String createRefreshToken(String email, String role) {
    return Jwts.builder()
        .setSubject(email)
        .claim("role", role)
        .setIssuedAt(new Date())
        .setExpiration(new Date(System.currentTimeMillis() + 604800000)) // 7일
        .signWith(privateKey, SignatureAlgorithm.RS256)
        .compact();
}
```

###  2. AuthController 수정

```java
String refreshToken = jwtUtil.createRefreshToken(request.getEmail(), role);
```

---

##  부가적으로 확인한 내용

- DB에는 `USER`, `ADMIN` 역할이 Enum으로 잘 들어가 있었음
- `@Enumerated(EnumType.STRING)` 어노테이션도 정상 적용
- 로그인 시에는 정상 발급되지만, Refresh 시에는 누락된 정보로 인한 문제

---

## 깨달은 점


- 토큰을 만드는 쪽과 파싱하는 쪽의 key-value 구조 일관성을 반드시 확인할 것
- 보안관련 공부를 통해 보안성을 조금 보완하기 위해 성급하게 공부를 하다 필드하나를 제대로 신경을 쓰지 못하였습니다.
- 디버깅과 테스트 코드 작성으로 더욱 꼼꼼히 신경써 구조적인 안정성을 책임질 수 있는 능력을 길러야함을 깨달았습니다.

---


---


