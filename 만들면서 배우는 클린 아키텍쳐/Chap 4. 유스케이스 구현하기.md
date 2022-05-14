# Chap 4. 유스케이스 구현하기

### 도메인 모델 구현하기

어플리케이션, 웹, 영속성 계층이 느슨하게 결합되어 있기 때문에 필요한대로 도메인 코드를 자유롭게 모델링할 수 있다.

헥사고날 아키텍처는 도메인 중심의 아키텍처에 적합하기 때문에 도메인 엔티티를 먼저 만들어 도메인 중심의 유스케이스를 구현한다.

```java
package io.reflectoring.buckpal.account.domain;

import java.time.LocalDateTime;
import java.util.Optional;

import lombok.AccessLevel;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.Value;

/**
 * An account that holds a certain amount of money. An {@link Account} object only
 * contains a window of the latest account activities. The total balance of the account is
 * the sum of a baseline balance that was valid before the first activity in the
 * window and the sum of the activity values.
 */
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class Account {

	/**
	 * The unique ID of the account.
	 */
	@Getter private final AccountId id;

	/**
	 * The baseline balance of the account. This was the balance of the account before the first
	 * activity in the activityWindow.
	 */
	@Getter private final Money baselineBalance;

	/**
	 * The window of latest activities on this account.
	 */
	@Getter private final ActivityWindow activityWindow;

	/**
	 * Creates an {@link Account} entity without an ID. Use to create a new entity that is not yet
	 * persisted.
	 */
	public static Account withoutId(
					Money baselineBalance,
					ActivityWindow activityWindow) {
		return new Account(null, baselineBalance, activityWindow);
	}

	/**
	 * Creates an {@link Account} entity with an ID. Use to reconstitute a persisted entity.
	 */
	public static Account withId(
					AccountId accountId,
					Money baselineBalance,
					ActivityWindow activityWindow) {
		return new Account(accountId, baselineBalance, activityWindow);
	}

	public Optional<AccountId> getId(){
		return Optional.ofNullable(this.id);
	}

	/**
	 * Calculates the total balance of the account by adding the activity values to the baseline balance.
	 */
	public Money calculateBalance() {
		return Money.add(
				this.baselineBalance,
				this.activityWindow.calculateBalance(this.id));
	}

	/**
	 * Tries to withdraw a certain amount of money from this account.
	 * If successful, creates a new activity with a negative value.
	 * @return true if the withdrawal was successful, false if not.
	 */
	public boolean withdraw(Money money, AccountId targetAccountId) {

		if (!mayWithdraw(money)) {
			return false;
		}

		Activity withdrawal = new Activity(
				this.id,
				this.id,
				targetAccountId,
				LocalDateTime.now(),
				money);
		this.activityWindow.addActivity(withdrawal);
		return true;
	}

	private boolean mayWithdraw(Money money) {
		return Money.add(
				this.calculateBalance(),
				money.negate())
				.isPositiveOrZero();
	}

	/**
	 * Tries to deposit a certain amount of money to this account.
	 * If sucessful, creates a new activity with a positive value.
	 * @return true if the deposit was successful, false if not.
	 */
	public boolean deposit(Money money, AccountId sourceAccountId) {
		Activity deposit = new Activity(
				this.id,
				sourceAccountId,
				this.id,
				LocalDateTime.now(),
				money);
		this.activityWindow.addActivity(deposit);
		return true;
	}

	@Value
	public static class AccountId {
		private Long value;
	}
}
```

> https://github.com/wikibook/clean-architecture/blob/main/src/main/java/io/reflectoring/buckpal/account/domain/Account.java 에 있는 예시코드

`Account` 엔티티에서는 실제 계좌의 현재 스냅샷을 제공하여 모든 입/출금을 포착한다. 한 계좌에 대한 모든 활동들에 대해서 메모리에 한꺼번에 올리는 것은 좋은 방법이 아니므로 `ActivityWindow` value object에서 포착한 일정 기간의 범위에 해당하는 활동만 보유한다.

계좌의 현재 잔고를 계산하기 위해서 Account 엔티티에서는 `baselineBalance` 속성을 가지고 있다. 현재 총 잔고는 `baselineBalance`에 활동창의 모든 활동들의 잔고 합이 된다.

즉, 계좌에서 일어나는 모든 입/출금은 메소드처럼 새로운 활동을 활동창에 추가하고, 이를 중심으로 유스케이스를 규현하기 위해 바깥방향으로 나아갈 수 있다.



### 유스케이스 둘러보기

#### 유스케이스의 일반적 단계

1. 입력을 받는다
2. 비즈니스 규칙을 검증한다
3. 모델 상태를 조작한다
4. 출력을 반환한다



##### 입력

유스케이스는 인커밍 어댑터로부터 입력을 받는다. 이 책의 지은이는 '유스케이스 코드가 **도메인 로직에만 신경 써야 하고 입력 유효성 검증으로 오염되면 안 된다**고 생각한다' 라고 했다. 그래서 입력 유효성 검증은 다른 곳에서 처리한다.



##### 비즈니스 규칙 검증

유스케이스는 **비즈니스 규칙을 검증할 책임**이 있고 도메인 엔티티와 이 책임을 공유한다. 챕터의 후반부에서는 입력 유효성 검증과 비즈니스 규칙 검증의 차이점에 대해 다룬다.



##### 모델 상태 조작

비즈니스 규칙을 충족하면 유스케이스는 입력을 기반으로 어떤 방법으로든 모델의 상태를 변경한다. 일반적으로 **도메인 객체의 상태**를 바꾸고 영속성 어댑터를 통해 구현된 포트로 이 상태를 전달해서 저장할 수 있게 한다.

또 다른 아웃고잉 어탭터를 호출 할 수도 있다.



##### 출력 반환

아웃고잉 어댑터에서 온 출력값을 유스케이스를 호출한 어댑터로 반환할 출력 객체로 변환한다.

서비스는 인커밍 포트 인터페이스인 `SendMoneyUseCase`를 구현하고, 계좌를 불러오기 위해 아웃고잉 포트 인터페이스인 `LoadAccountPort`를 호출한다. 그리고 데이터베이스의 계좌 상태를 업데이트하기 위해 `UpdateAccountStatePort`를 호출한다.

```java
package io.reflectoring.buckpal.account.application.service;

import io.reflectoring.buckpal.account.application.port.in.SendMoneyCommand;
import io.reflectoring.buckpal.account.application.port.in.SendMoneyUseCase;
import io.reflectoring.buckpal.account.application.port.out.AccountLock;
import io.reflectoring.buckpal.account.application.port.out.LoadAccountPort;
import io.reflectoring.buckpal.account.application.port.out.UpdateAccountStatePort;
import io.reflectoring.buckpal.common.UseCase;
import io.reflectoring.buckpal.account.domain.Account;
import io.reflectoring.buckpal.account.domain.Account.AccountId;
import lombok.RequiredArgsConstructor;

import javax.transaction.Transactional;
import java.time.LocalDateTime;

@RequiredArgsConstructor
@UseCase
@Transactional
public class SendMoneyService implements SendMoneyUseCase {

	private final LoadAccountPort loadAccountPort;
	private final AccountLock accountLock;
	private final UpdateAccountStatePort updateAccountStatePort;
	private final MoneyTransferProperties moneyTransferProperties;

	@Override
	public boolean sendMoney(SendMoneyCommand command) {

		checkThreshold(command);

		LocalDateTime baselineDate = LocalDateTime.now().minusDays(10);

		Account sourceAccount = loadAccountPort.loadAccount(
				command.getSourceAccountId(),
				baselineDate);

		Account targetAccount = loadAccountPort.loadAccount(
				command.getTargetAccountId(),
				baselineDate);

		AccountId sourceAccountId = sourceAccount.getId()
				.orElseThrow(() -> new IllegalStateException("expected source account ID not to be empty"));
		AccountId targetAccountId = targetAccount.getId()
				.orElseThrow(() -> new IllegalStateException("expected target account ID not to be empty"));

		accountLock.lockAccount(sourceAccountId);
		if (!sourceAccount.withdraw(command.getMoney(), targetAccountId)) {
			accountLock.releaseAccount(sourceAccountId);
			return false;
		}

		accountLock.lockAccount(targetAccountId);
		if (!targetAccount.deposit(command.getMoney(), sourceAccountId)) {
			accountLock.releaseAccount(sourceAccountId);
			accountLock.releaseAccount(targetAccountId);
			return false;
		}

		updateAccountStatePort.updateActivities(sourceAccount);
		updateAccountStatePort.updateActivities(targetAccount);

		accountLock.releaseAccount(sourceAccountId);
		accountLock.releaseAccount(targetAccountId);
		return true;
	}

	private void checkThreshold(SendMoneyCommand command) {
		if(command.getMoney().isGreaterThan(moneyTransferProperties.getMaximumTransferThreshold())){
			throw new ThresholdExceededException(moneyTransferProperties.getMaximumTransferThreshold(), command.getMoney());
		}
	}
}
```



### 입력 유효성 검증

어플리케이션 계층에서 입력 유효성을 검증해야 하는 이유는, 그렇게 하지 않을 경우에 어플리케이션 코어의 바깥쪽으로부터 유효하지 않은 입력값을 받게되고, 모델의 상태를 해칠 수 있기 때문이다.

입력모델의 생성자 내에서 입력 유효성을 검증한다. 입력해야하는 조건 중 한 가지라도 위배된다면 객체 생성시에 예외를 던져 생성을 막으면 된다.

`SendMoneyCommand`의 필드에 final을 지정하여 불변 필드로 만들었기 때문에 생성이 성공하면 유효하게 변경되지만 그렇지 않은 경우 변경이 일어나지 않는다. 또한 `SendMoneyCommand`는 유스케이스 API의 일부이기 때문에 인커밍 포트 패키지에 위치하기 때문에 유스케이스 코드를 오염시키지도 않는다.

자바에서는 `Bean Validation API`를 사용하여 필드를 검증할 수도 있다.

```java
package io.reflectoring.buckpal.account.application.port.in;

import io.reflectoring.buckpal.account.domain.Account.AccountId;
import io.reflectoring.buckpal.account.domain.Money;
import io.reflectoring.buckpal.common.SelfValidating;
import lombok.EqualsAndHashCode;
import lombok.Value;

import javax.validation.constraints.NotNull;

@Value
@EqualsAndHashCode(callSuper = false)
public
class SendMoneyCommand extends SelfValidating<SendMoneyCommand> {

    @NotNull
    private final AccountId sourceAccountId;

    @NotNull
    private final AccountId targetAccountId;

    @NotNull
    private final Money money;

    public SendMoneyCommand(
            AccountId sourceAccountId,
            AccountId targetAccountId,
            Money money) {
        this.sourceAccountId = sourceAccountId;
        this.targetAccountId = targetAccountId;
        this.money = money;
        this.validateSelf();
    }
}
```

`SelfValidating` 추상클래스는 `validateSelf()`메소드를 제공하여 생성자 마지막에서 이 메소드를 호출한다. 이 메소드가 필드에 지정된 `Bean Validation`어노테이션을 검증하고, 위반한 경우 예외를 던진다.

입력 모델에 있는 유효성 검증 코드를 통해 유스케이스 그현체 주위에 **사실상 오류 방지 계층**을 만들어 잘못된 입력을 호출자에게 돌려주는 보호막 역할을 한다.



### 생성자의 힘

입력 모델 `SendMoneyCommand`는 클래스가 불변이기 때문에 파라미터 리스트에 클래스의 각 속성에 해당하는 파라미터들이 포함되어있고, 생성자에서 파라미터의 유효성 검증도 하기 때문에 많은 책임을 지고 있다.

때로는 좋은 IDE를 사용하면 긴 파라미터 리스트도 충분히 포매팅할 수 있고 파라미터명 힌트도 주기 때문에 이러한 도움을 받는 것도 좋다.



### 유스케이스마다 다른 입력 모델

불변 커맨드 객체의 필드에 대해서 `null`을 유효한 상태로 받아들이는 것 자체가 `code smell`이다. 그보다 더 문제는 서로 다른 유스케이스에 대해 어떻게 입력 유효성을 검증하느냐이다.

이러한 경우 각 유스케이스 전용 입력 모델을 사용해야 한다. 전용 입력 모델은 유스케이스를 훨씬 명확하게 만들고, 다른 유스케이스와 결합도 제거하여 불필요한 부수효과를 발생하지 않게 한다.



### 비즈니스 규칙 검증하기

비즈니스 규칙을 검증하는 것은 도메인 모델의 현재 상태에 접근해야 하고, 입력 유효성 검증은 그럴 필요가 없다.

입력 유효성을 검증하는 것은 **구문상의(syntactical)** 유효성 검증이라고 할 수 있고 비즈니스 규칙은 유스케이스의 맥락속에서 **의미적인(semantical)** 유효성 검증이라고 할 수 있다.

- ex) 출금 계좌는 초과 출금되어서는 안 된다
  - 모델의 현재 상태에 접근해야하므로 **비즈니스 규칙**

- ex) 송금되는 금액은 0보다 커야 한다
  - 모델에 접근하지 않고도 검증될 수 있는 **입력 유효성 검증**

비즈니스 규칙은 도메인 엔티티 안에 넣는 것이 좋다. 만약 도메인 엔티티에서 비즈니스 규칙을 검증하기가 여의치 않다면 유스케이스 코드에서 도메인 엔티티를 사용하기 전에 해도 된다.



### 풍부한 도메인 모델 vs 빈약한 도메인 모델

`풍부한 도메인 모델`에서는 어플리케이션의 코어에 있는 엔티티에서 가능한 한 많은 도메인 로직이 구현된다. 이 경우 유스케이스는 도메인 모델의 진입점으로 동작하여 체계화된 도메인 엔티티 메소드 호출로 변환하고, 많은 비즈니스 규칙이 유스케이스 구현체 대신 엔티티에 위치하게 된다.

`빈약한 도메인 모델`에서는 엔티티 자체가 굉장히 얇고, getter, setter 메소드만 포함하고 어떤 도메인 로직도 가지고 있지 않다. 그렇기 때문에 도메인 로직이 유스케이스 클래스에 구현되어있어 엔티티는 빈약할 수 있지만 유스케이스가 풍부하다고 볼 수 있다.



### 유스케이스마다 다른 출력 모델

유스케이스가 할 일을 다하고 나면 호출자에게는 각 유스케이스에 맞게 구체적으로 꼭 필요한 데이터만 반호나하는 것이 좋다. 유스케이스를 가능한 구체적으로 유지하기 위해서는 지속적인 질문이 필요하고 만약 의심스러운 경우에는 가능한 적게 반환하는 것이 좋다.

유스케이스들 간에 같은 출력 모델을 공유하게 되면 유스케이스들도 그만큼 강하게 결합된다. 공유 모델은 장기적으로 봤을 때 하나의 출력 모델에서 새로운 필드가 필요한 경우 다른 유스케이스에서도 이를 처리해야 하기 때문에 **단일 책임 원칙**을 적용하고 모델을 분리해서 유지하는 것이 결합을 제거하는데 도움이 된다.



### 읽기 전용 유스케이스는 어떨까?

읽기 전용 작업을 유스케이스라고 언급하기에는 조금 이상하지만, 어플리케이션 코어의 관점에서 이 작업은 간단한 데이터 쿼리다. 이 책의 아키텍처 스타일에서는 쿼릴르 위한 인커밍 전용 포트를 만들고 `쿼리 서비스`에 구현한다. 쿼리 서비스는 유스케이스 서비스와 동일한 방식으로 동작하고 이러한 방식은 `CQS(Command-Query Separation)`이나 `CQRS(Comand-Query Reponsibility Segregation)` 같은 개념과 잘 맞는다.



### 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?

입출력 모델을 독립적으로 모델링한다면 원치 않는 부수효과를 피할 수 있다. 유스케이스 간에 모델을 공유하는 것 보다 유스케이스별로 모델을 만들면 **명확하게 이해 가능**하고, 장기적으로 **유지보수에 용이성도 있다.**



## Q&A

#### 생성자를 항상 사용하는것 보다는 스프링에서 제공하는 AOP validate를 사용하는 것은 어떤가?

장점

* spring에서 validator제공
* lombok의 자동완성을 사용가능

단점

* 스프링 코드가 침투한다.

>https://velog.io/@_koiil/SpringBoot-Spring-Validation%EC%9D%84-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EC%9C%A0%ED%9A%A8%EC%84%B1-%EA%B2%80%EC%A6%9D

> @Validated + @Valid

validator는 취향에 맞게 사용하면 된다.



클린 아키텍처 관점으로 보면 파라미터가 많아지면 입력 모델이 단일 책임 원칙에 위배되지 않는지를 다시 한 번 파악해봐야한다. 또한 파라미터들을 객체로 묶어서 사용할 수 있는 방안을 생각해봐야 한다.

* 인자들을 따로 그룹핑해서 클래스를 쪼개는 편이 좋을 것 같다.



#### 입력 유효성 검증과 비즈니스 규칙은 도메인 모델의 상태에 접근 유무에 따라서 다른데 도메인 엔티티에 접근하는 경우에는 어떤 것으로 봐야할까?

결국엔 모호한 경계를 가져서 개발자의 판단에 맡기는 것이 낫다.



#### 풍부한 도메인 모델과 빈약한 도메인 모델

개발자의 취향이기는 한데 어떤 것이 더 선호되는지.

* 주로 빈약한 도메인 모델이 선호도가 높은 것 같다.
  * 코드 가독성이 떨어질 수 있다.