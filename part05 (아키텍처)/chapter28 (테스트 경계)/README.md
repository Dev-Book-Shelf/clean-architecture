# 28장 테스트 경계

## 시스템 컴포넌트인 테스트

테스트도 시스템의 일부, 아키텍처 관점에서는 모든 테스트가 동일하다.

테스트는 태생적으로 의존성 규칙을 따른다.  
테스트는 아키텍처에서 가장 바깥쪽 원으로 생각할 수 있다. 시스템 내부의 어떤 것도 테스트에는 의존하지 않으며, 테스트는 시스템의 컴포넌트를 향해, 항상 원의 안쪽으로 의존한다.

또한 테스트는 독립적으로 배포 가능하다.

테스트는 시스템 컴포넌트 중 가장 고립되어 있다. 시스템 운영에 필수적이지 않으며, 사용자도 테스트에 의존하지 않기 때문이다.

## 테스트를 고려한 설계

테스트가 지닌 고립성과 테스트는 대체로 배포하지 않는 사실 때문에 개발자는 때때로 테스트가 시스템의 설계 범위 밖에 있다고 착각한다.

테스트가 시스템의 설계와 잘 통합되지 않으면 테스트는 깨지기 쉬워지고 시스템은 뻣뻣해진다.

시스템에 강하게 결합된 테스트라면 시스템이 변경될 때 함께 변경되어야 한다.(Fragile Tests Problem)

이 문제를 해결하려면 테스트를 고려해서 설계해야 한다.

## 테스트 API

테스트가 모든 업무 규칙을 검증하는 데 사용할 수 있도록 특화된 API를 만들 수 있다.  
이 API는 보안 제약사항을 무시할 수 있으며, 값비싼 자원은 건너뛸 수 있어야 한다.

이 API는 사용자 인터페이스가 사용하는 인터랙터와 인터페이스 어댑터들의 상위 집합이 될 것이다.

### 구조적 결합

상용 클래스에 대응하는 테스트 클래스가 각각 존재하고, 상용 메서드에 테스트 메서드 집합이 각각 존재하는 테스트 스위트가 있다고 가정해보자.  
상용 클래스나 메서드 중 하나라도 변경되면 딸려 있는 다수의 테스트가 변경되어야 한다.

테스트 API의 역할은 상용 코드를 리팩토링하거나 진화시킬 때도 테스트에 영향을 주지 않도록 만드는 것이다.  
따로따로 진화할 수 있다는 점은 아주 중요한데, 상용 코드는 더 추상적이고 범용적인 형태로 발전, 테스트는 더 구체적이고 더 특화된 형태로 변할 것이기 때문이다.

### 코드 예시
<details> <summary> 계좌 이체 로직 </summary>

``` java
// 도메인 엔티티
public class Account {
    private final String accountId;
    private Money balance;

    public Account(String accountId, Money initialBalance) {
        this.accountId = accountId;
        this.balance = initialBalance;
    }

    public String getAccountId() {
        return accountId;
    }

    public Money getBalance() {
        return balance;
    }

    public void withdraw(Money amount) {
        if (balance.isLessThan(amount)) {
            throw new IllegalStateException("잔액 부족");
        }
        this.balance = this.balance.minus(amount);
    }

    public void deposit(Money amount) {
        this.balance = this.balance.plus(amount);
    }
}

// 값 객체
public class Money {
    private final long amount;

    private Money(long amount) {
        this.amount = amount;
    }

    public static Money of(long amount) {
        return new Money(amount);
    }

    public Money plus(Money other) {
        return new Money(this.amount + other.amount);
    }

    public Money minus(Money other) {
        return new Money(this.amount - other.amount);
    }

    public boolean isLessThan(Money other) {
        return this.amount < other.amount;
    }

    public long getAmount() {
        return amount;
    }
}
```

``` java
// 도메인 Port (Repository)
public interface AccountRepository {
    Account findById(String accountId);
    void save(Account account);
}
```

``` java
// 유스케이스(업무 규칙)
public class TransferMoneyUseCase {

    private final AccountRepository accountRepository;

    public TransferMoneyUseCase(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    public void transfer(String fromAccountId, String toAccountId, Money amount) {
        Account from = accountRepository.findById(fromAccountId);
        Account to = accountRepository.findById(toAccountId);

        from.withdraw(amount);
        to.deposit(amount);

        accountRepository.save(from);
        accountRepository.save(to);
    }
}
```

그리고 테스트 API
``` java
/**
 * 테스트 전용 API.
 * - 여러 유스케이스를 한 번에 엮어서 사용하기 쉽게 만든다.
 * - 테스트에 특화된 편의 메서드를 제공한다.
 * - 보안, 인증, 복잡한 인프라 디테일을 생략할 수 있다.
 */
public class BankingTestApi {

    private final AccountRepository accountRepository;
    private final TransferMoneyUseCase transferMoneyUseCase;

    public BankingTestApi(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
        this.transferMoneyUseCase = new TransferMoneyUseCase(accountRepository);
    }

    // ====== 테스트에 특화된 헬퍼들 ======

    public String createAccountWithBalance(long initialBalance) {
        String accountId = "TEST-" + System.nanoTime();
        Account account = new Account(accountId, Money.of(initialBalance));
        accountRepository.save(account);
        return accountId;
    }

    public void deposit(String accountId, long amount) {
        Account account = accountRepository.findById(accountId);
        account.deposit(Money.of(amount));
        accountRepository.save(account);
    }

    public void transfer(String fromAccountId, String toAccountId, long amount) {
        transferMoneyUseCase.transfer(fromAccountId, toAccountId, Money.of(amount));
    }

    public long getBalance(String accountId) {
        return accountRepository.findById(accountId).getBalance().getAmount();
    }

    // 필요하다면 시간·이벤트 조작 같은 것도 여기에 들어간다.
}
```

테스트 코드에서 테스트 API 사용

``` java
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

public class TransferMoneyTest {

    private final InMemoryAccountRepository repo = new InMemoryAccountRepository();
    private final BankingTestApi api = new BankingTestApi(repo);

    @Test
    void 돈_이체_후_잔액이_변한다() {
        // given
        String from = api.createAccountWithBalance(10_000);
        String to   = api.createAccountWithBalance(0);

        // when
        api.transfer(from, to, 3_000);

        // then
        assertThat(api.getBalance(from)).isEqualTo(7_000);
        assertThat(api.getBalance(to)).isEqualTo(3_000);
    }

    @Test
    void 잔액_부족하면_예외_발생() {
        // given
        String from = api.createAccountWithBalance(1_000);
        String to   = api.createAccountWithBalance(0);

        // when & then
        org.junit.jupiter.api.Assertions.assertThrows(
            IllegalStateException.class,
            () -> api.transfer(from, to, 5_000)
        );
    }
}
```

테스트는 테스트용 진입점인 BankingTestApi 만 알고있기 때문에 시스템 내부 구조가 바뀌어도 BankingTestApi의 인터페이스만 유지되면 테스트는 깨지지 않음

</details>

### 보안

테스트 API가 지닌 강력한 힘(보안 제약사항 무시 등)을 운영 시스템에 배포하면 위험하니, 테스트 API 중 위험한 부분의 구현부는 독립적으로 배포할 수 있는 컴포넌트로 분리해야 한다.

## 결론

테스트도 시스템의 일부이다.  
그렇게 생각하지 않고 설계하면 테스트가 깨지기 쉽고 유지보수하기 어려워진다.