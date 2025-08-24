
---

# Spring LazyInitializationException Error (JPA LAZY + DTO 변환)

## 1. 문제 배경

OpenBanking 프로젝트에서 거래내역을 조회한 뒤 **Controller에서 DTO 변환** 중,
연관 엔티티(`Transaction.account`)가 **LAZY**라 세션 종료 후 접근할 때 예외 발생.

* ❌ Service에서 엔티티만 반환 → Controller에서 `getAccount().getAccountNumber()` 접근
* ❌ 트랜잭션 범위 밖에서 **프록시 초기화 시도**


➡️ **필요한 연관만 트랜잭션 안에서 로딩**하거나 **DTO로 직접 조회**해야 함.

---

## 2. 증상 (에러 메시지)

```
org.hibernate.LazyInitializationException: 
could not initialize proxy [com.fintech.api.domain.Account#4] - no Session
```

---

## 3. 재현 코드 (문제 흐름)

```java
// [Controller]
@GetMapping("/account/{accountId}")
public ResponseEntity<List<TransactionWithAccountDto>> getTransactionsByAccountId(
        @AuthenticationPrincipal UserDetails userDetails,
        @PathVariable Long accountId) {
    String email = userDetails.getUsername();
    List<Transaction> txList = transactionService.getTransactionsByAccountId(email, accountId);
    // ❌ 여기서 DTO 변환 (세션 이미 종료)
    List<TransactionWithAccountDto> dto = txList.stream()
            .map(TransactionWithAccountDto::from)   // from() 안에서 tx.getAccount().getAccountNumber()
            .toList();
    return ResponseEntity.ok(dto);
}
```

---

## 4. 원인 분석

* `@ManyToOne(fetch = LAZY)` 기본값으로 **Account는 즉시 로딩되지 않음**(프록시).
* Service 트랜잭션 종료 후 Controller에서 **LAZY 필드 접근** → Hibernate가 DB 접근 못하고 예외.

---

## 5. 해결 전략 요약

1. **fetch join**으로 Service에서 미리 로딩 + **Service 내부에서 DTO 변환**
2. **DTO 프로젝션**(JPQL `new` / 인터페이스 기반)으로 **바로 DTO 반환**
3. 간단 케이스는 **@EntityGraph**로 연관 로딩


---

## 6. 해결 방법 상세

### 6.1 방법 A — fetch join (+ Service에서 DTO 변환)

**Repository**

```java
@Query("""
    select t
    from Transaction t
    join fetch t.account a
    where a.id = :accountId
      and a.user.email = :email
    order by t.transactionDate desc
""")
List<Transaction> findAllByAccountIdWithAccount(String email, Long accountId);
```

**Service**

```java
@Transactional(readOnly = true)
public List<TransactionWithAccountDto> getTransactionsByAccountId(String email, Long accountId) {
    // (권한 체크) 본인 계좌 여부 확인
    accountRepository.findById(accountId)
        .filter(a -> a.getUser().getEmail().equals(email))
        .orElseThrow(() -> new IllegalArgumentException("본인의 계좌가 아닙니다."));

    // fetch join으로 Account 함께 로딩
    List<Transaction> txList = txRepository.findAllByAccountIdWithAccount(email, accountId);

    // ✅ 트랜잭션 범위 안에서 DTO 변환
    return txList.stream().map(TransactionWithAccountDto::from).toList();
}
```

**Controller (간결)**

```java
@GetMapping("/account/{accountId}")
public ResponseEntity<List<TransactionWithAccountDto>> getTransactionsByAccountId(
        @AuthenticationPrincipal UserDetails userDetails, @PathVariable Long accountId) {
    return ResponseEntity.ok(
        transactionService.getTransactionsByAccountId(userDetails.getUsername(), accountId)
    );
}
```

---

### 6.2 방법 B — DTO 프로젝션으로 바로 조회 (세션 독립)

**Repository (DTO 생성자 프로젝션)**

```java
@Query("""
  select new com.fintech.api.dto.TransactionWithAccountDto(
    t.id, t.amount, t.type, t.description, t.transactionDate, t.balanceAfter, a.accountNumber
  )
  from Transaction t
  join t.account a
  where a.id = :accountId
    and a.user.email = :email
  order by t.transactionDate desc
""")
List<TransactionWithAccountDto> findDtoByAccountId(String email, Long accountId);
```

**Service**

```java
@Transactional(readOnly = true)
public List<TransactionWithAccountDto> getTransactionsByAccountId(String email, Long accountId) {
    // (선택) 본인 계좌 검사
    accountRepository.findById(accountId)
        .filter(a -> a.getUser().getEmail().equals(email))
        .orElseThrow(() -> new IllegalArgumentException("본인의 계좌가 아닙니다."));
    return txRepository.findDtoByAccountId(email, accountId); // 변환 불필요
}
```

> 장점: **LAZY 문제 원천 차단**, 불필요한 필드 미로딩 → 성능/메모리 유리

---

### 6.3 방법 C — @EntityGraph

```java
@EntityGraph(attributePaths = {"account"})
@Query("""
  select t from Transaction t
  where t.account.id = :accountId and t.account.user.email = :email
  order by t.transactionDate desc
""")
List<Transaction> findAllByAccountIdWithGraph(String email, Long accountId);
```

> 단순 연관 한두 개 로딩엔 간단. 복잡한 조건/페이징에선 fetch join/프로젝션을 선호.

---

## 7. 현재 코드 기준 “최소 수정안”

* **Service에서 DTO 변환**으로 이동
* Repository 하나 추가 (A 또는 B 택1)

```java
// 기존 DTO
@Getter
@AllArgsConstructor
public class TransactionWithAccountDto {
    private Long id;
    private Long amount;
    private String type;
    private String description;
    private LocalDateTime transactionDate;
    private Long balanceAfter;
    private String accountNumber;

    public static TransactionWithAccountDto from(Transaction tx) {
        return new TransactionWithAccountDto(
            tx.getId(), tx.getAmount(), tx.getType(), tx.getDescription(),
            tx.getTransactionDate(), tx.getBalanceAfter(),
            tx.getAccount() != null ? tx.getAccount().getAccountNumber() : null
        );
    }
}
```

> Controller에서는 **Service가 반환한 DTO**만 그대로 내려주면 된다.

---

## 8. 테스트/검증 시나리오

### 8.1 재현 테스트 (@DataJpaTest)

```java
var tx = txRepository.findById(1L).orElseThrow();
// 트랜잭션 밖에서 아래 호출 → LazyInitializationException 기대
assertThrows(LazyInitializationException.class,
    () -> tx.getAccount().getAccountNumber());
```

### 8.2 해결 확인 (fetch join/프로젝션 적용 후)

* **예외 미발생** 확인
* 쿼리 수 감소 확인 (`logging.level.org.hibernate.SQL=debug`)

---


---

## 9. 깨달은 점

* LAZY 필드는 세션이 열려 있을 때만 안전하게 접근이 가능하기에 db 세션이 닫히고 재접근하는 부분을 조심해야합니다.
* 해결은 “세션을 늘리는 것”이 아니라 **필요 데이터만 명시적으로 로딩** 함으로써 해결방향을 잡아야합니다.
* dto 의 코드에서 연관 엔티티 관계 체크와 동작을 꼼꼼히 확인하여 서비스 로직부분을 디버깅해야 한다는 것을 많이 깨닫게 된 에러였습니다.


---

