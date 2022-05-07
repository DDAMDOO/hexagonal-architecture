# Chap 3. 코드 구성하기

### 계층으로 코드 구성하기

코드 구성의 첫 번째는 계층을 이용하는 것으로 웹, 도메인, 영속성 계층에 각각 `web`, `domain`, `persistence`로 나타냈다.



> buckpal
> * domain
>    * Account
>    * Activity
>    * AccountRepository
>    * AccountService
> * persistence
>   * AccountReposityroImpl
> * web
>  * AccountController

이러한 구조는 의존성 역전 원칙을 적용하여 의존성이 domain 패키지에 있는 도메인 코드를 향하도록 되어있어서 1장에서 설명한 것과 같이 repository와 repository의 구현부가 따로 있는 것을 볼 수 있다. 하지만 이 구조는 최적의 구조가 아닌 3가지 이유가 있다.

* ##### 어플리케이션의 기능 조각이나 특성을 구분짓는 패키지가 없다.

  * 기능이 추가되는 경우 추가적인 구조가 없다면 구분짓기가 어려워진다.

* ##### 어떤 유스케이스를 제공하는지 파악할 수 없다.

* ##### 패키지 구조를 통해서 우리가 목표로하는 아키텍처를 파악할 수 없다.



### 기능으로 구성하기

계층 구성하기에서 몇 가지 문제를 해결하여 만든 구성이다.

> buckpal
> * account
>  * Account
>  * AccountController
>  * AccountRepository
>  * AccountRepositoryImpl
>  * SendMoneyService

계층 패키지를 없애고, 각 기능을 묶어 account라는 같은 래벨의 패키지에 들어가고, 외부에서 접근이 안되는 클래스들에 대해서는 package-private 접근 수준을 이용할 수 있다.

또한 AccountService에서 책임을 좁히기 위해 SendMoneyService로 변경하여 클래스명만 보고도 유스케이스를 파악 가능하다. 이러한 아키텍처를 `소리치는 아키텍처(screaming architecture)`라고 명명한다.

기능에 의한 패키징은 **패키지명이나 인커밍포트, 아웃고잉 포트를 확인 불가능**하고, **가시성이 떨어진다**. 또한 도메인코드와 영속성 코드 간의 의존성 역전에 대해 구현체는 알 수 없도록 했지만 package-private을 이용해 도메인코드가 실수로 영속성 코드에 의존하는 것을 막을 수 없다.



### 아키텍처적으로 표현력 있는 패키지 구조

> buckpal
>
> * account
>   * adapter
>     * in
>       * web
>         * AccountController
>     * out
>       * persistence
>         * AccountPersistenceAdapter
>         * SpringDataAccountRepository
>   * domain
>     * Account
>     * Activity
>   * application
>     * SendMoneyService
>     * port
>       * in
>         * SendMoneyUsecase
>       * out
>         * LoadAccountPort
>         * UpdateAccountStatePort



각 요소들은 패키지에 하나씩 매핑되어 여러명이 함께 이야기 나눌 때 이해하기 쉽다. 이러한 구조를 `아키텍처-코드 갭` 또는 `모델-코드 갭`을 효과적으로 다룰 수 있는 요소이다.

패키지가 많아서 서로를 public으로 패키지간의 접근을 허용해야 할 것 같지만, 모든 클래스들은  application 패키지 내에 있는 포트 인터페이스를 통해서만 밖으로 호출되기 때문에 package-private 접근수준을 유지해도 된다.

이 구조는 DDD 개념을 대응시켜 상위 래벨 패키지는 다른 `bounded context(도메인 모델이 적용될 수 있는 범위)`와 통신할 진입점과 출구를 포함하는 바운디드 컨텍스트이므로 우리가 원하는 어떤 도메인 모델도 만들 수 있다.



### 의존성 주입의 역할

모든 계층의 의존성을 가진 중립적인 컴포넌트를 하나 도입하여 아키텍처를 구성하는 대부분의 클래스를 초기화 하는 역할에 의존성 주입을 활용할 수 있다.



### 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?

육각형 아키텍처 패키기 구조에서 아키텍처 다이어그램의 박스 이름을 따라 검색하면 되기 때문에 의사소통이나 협업, 유지보수가 더 수월해진다.



## Q&A

#### usecase 중 내부에 하나의 구현만 있을 때도 인터페이스로 두어야 하는게 형식적으로 둔 것인가.

* sendMoneyUsecase로 뺀 이유는 기능 단위로 나누려고 하다보니 나뉘어진 것 같다.
* 테스트코드를 짜기 위해 서비스의 메소드가 많을수록 어떻게든 처리를 해야하기 때문에 인터페이스를 구현하는 구현체 하나만 모킹해서 테스트하기 좋아진다.
* 컨트롤러에서 sendMoneyUsecase와 sendMoneyService중 어느 것을 의존하는게 컨트롤러의 변경이 적어질지에 대해서 생각해보면 Usecase가 더 적기 때문.



#### 클린아키텍처란?

과거의 규칙들을 지속적으로 테스트할 수 있어야 한다

* 시장에서 행동이 변화되면 이것을 녹일 수 있는 설계를 가지고 가야한다.

잘 쪼개진 인터페이스가 필요하지만, 이 책의 예제들은 단순히 내용을 설명하기 위한 내용이고, 현실적인 실무에서는 이보다 큰 시스템과 내용들이 있기 때문에 이 부분에 대해서 생각해봐야할 것 같다.



#### 책에서는 단일 도메인으로만 설명되어 있는데 다른 도메인이 추가되면 어떻게 해야 할 것인가.

상호작용 하는 것 자체가 MSA로 쪼개기 어렵기 때문에 큰 헥사고날 아키텍처로 구성한다.

각각을 헥사고날로 그리고 서로가 서로에게 의존을 할 때에는 A가 B에 의존할 때는 하나의 어댑터라고 생각하여 포트로 의존하면 된다.

