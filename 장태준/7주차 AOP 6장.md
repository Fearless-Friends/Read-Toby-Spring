# 6장 aop

aop를 도입하여 트랜잭션 경계설정 기능을 깔끔하게 바꾸고 도입해야 했던 이유도 알아보자.

## 6.1 트랜잭션 코드의 분리

트랜잭션 경계설정을 넣은 코드때문에 찜찜했다.

### 6.1.1 메소드 분리

```java
public void upgradeLevels() throws Exception { // 트랜잭션 경계설정
	TransactionStatus status = this.transactionManager
			.getTransaction(new DefaultTransactionDefinition());
	try{
		List<User> users: userDao.getAll();  // 비즈니스 로직
		for (User user: users) {
			if (canUpgradeLevel(user)){
				upgradeLevel(user);
			}
		}
		this.transactionManager.commit(status);
	} catch (Exceoption e){ // 트랜잭션 경계 설정
			this.transactionManager.rollback(status);
			throw e;
	}
}
```

분리되어 있다 + 서로 주고 받는 내용이 없다 → 분리 가능

```java
public void upgradeLevels() throws Exception { // 트랜잭션 경계설정
	TransactionStatus status = this.transactionManager
			.getTransaction(new DefaultTransactionDefinition());
	try{
		upgradeLevelsInternal();
		this.transactionManager.commit(status);
	} catch (Exceoption e){ // 트랜잭션 경계 설정
			this.transactionManager.rollback(status);
			throw e;
	}
}

private void upgradeLevelsInternal(){
	List<User> users: userDao.getAll();
		for (User user: users) {
			if (canUpgradeLevel(user)){
				upgradeLevel(user);
			}
		}
}
```

### 6.1.2 DI를 이용한 클래스의 분리

서로 직접 주고 받는 데이터가 없으므로 아예 분리해내자.

#### DI 적용을 이용한 트랜잭션 분리

![스크린샷 2022-11-07 오후 9.51.11.png](imgs/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-11-07_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_9.51.11.png)

강한 결합도를 가지고 있다.

![스크린샷 2022-11-07 오후 9.51.30.png](imgs/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-11-07_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_9.51.30.png)

UserService 인터페이스를 통해 약한 결합으로 바꿔졌다.

구현클래스를 바꿔가며 쓰기 위해 런타임 시에 DI를 적용시킨다. 

![스크린샷 2022-11-07 오후 9.53.37.png](imgs/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-11-07_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_9.53.37.png)

UserService를 구현한 다른 구현 클래스를 만든다. 트랜잭션의 경계설정이라는 책임만 맡고 있는 클래스를 통해 로직 처리 작업을 위임하는 것이다.

#### UserService 인터페이스 도입

처음에 사용했던 로직인 upgradeLevels만 남기고 UserSeviceImpl를 만든다.

#### 분리된 트랜잭션 기능

비즈니스 트랜잭션 로직을 처리를 담은 UserServiceTx를 만든다.

#### 트랜잭션 적용을 위한 DI 설정

![스크린샷 2022-11-07 오후 10.22.26.png](imgs/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-11-07_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_10.22.26.png)

UserServiceTest 빈은 UserServiceTx, UserServiceTx빈은 UserServuceImpl을 DI하게 만든다.

#### 트랜잭션 경계설정 코드 분리의 장점

- 비즈니스 로직을 담당하는 부분을 작성할 때 트랜잭션과 같은 기술적인 내용을 신경쓰지 않아도 된다.
- 비즈니스 로직에 대한 테스트를 손쉽게 만들어낼 수 있다.

## 6.2 고립된 단위 테스트

테스트는 가능한 작은 단위로

### 6.2.1 복잡한 의존관계 속의 테스트

UserServiceTest는 UserDao, TransactionManager, MailSender까지 세 가지 의존관계를 가지고 있다.

### 6.2.2 테스트 대상 오브젝트 고립시키기

테스트의 대상이 환경이나, 외부 서버, 다른 클래스의 코드에 종속되지않고 영향을 받지 않도록 고립시킬 필요가 있다.

#### 테스트를 위한 UserServiceImpl 고립

UserServiceImpl만 고립시키면 두개의 목 오브젝트에 의존하는 테스트 구조를 만들 수 있다.

#### 고립된 단위 테스트 활용

1. 테스트 실행 중에 UserDao를 통해 가져올 테스트용 정보를 DB에 넣는다. UserDao는 결국 DB를 이용해 정보를 가져오기 때문에 최후의 의존 대상인 DB에 직접 정보를 넣어줘야 한다.
2. 메일 발송 여부를 확인하기 위해 MailSender 목 오브젝트를 DI  해준다.
3. 실제 테스트 대상인 userService의 메소드를 실행한다.
4. 결과가 DB에 발영됐는지 확인하기 위해서 UserDao를 이용해 DB에서 데이터를 가져와 결과를 확인한다.
5. 목 오브젝트를 통해 UserService에 의한 메일 발송이 있었는지를 확인하면 된다.

#### UserDao 목 오브젝트

```java
static class MockUserDao implements UserDao {
	private List<User> users;
	private List<User> updated = new ArrayList();

	private MockUserDao(List<User> users) {
		this.users = users;
	}
	
	public List<User> getUpdated() {
		return this.updated;
	}

	public List<User> getAll() {
		return this.users;
	}

	public void update(User user){
		updated.add(user);
	}

	public void add(User user) {throw new UnsupportedOperationException();}
	public void deleteAll() {throw new UnsupportedOperationException();}
	public User get(String id) {throw new UnsupportedOperationException();}
	public int getCount() {throw new UnsupportedOperationException();}
}
```

사용되지 않을 메소드는 exception을 던진다.

#### 테스트 수행 성능의 향상

UserServiceTest를 고립된 테스트로 실행할 경우 매우 빠른 결과가 나온다.

### 6.2.3 단위 테스트와 통합 테스트

테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 테스트하는 것이 단위 테스트

두개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나, 외부의 DB나 파일, 서비스 등의 리소스가 참여하는 테스트는 통합 테스트

- 단위 테스트를 먼저 고려해야한다.
- 하나의 클래스나 성격과 목적이 같은 긴밀한 클래스를 모아서 외부와의 의존관계를 모두 차단하고 스텁이나 목 오브젝트 등의 테스트 대역을 이용하도록 테스트를 만든다.
- 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트로 만든다.

### 6.2.4 목 프레임워크

Mockito

- 인터페이스를 이용해 목 오브젝트를 만든다.
- 목 오브젝트가 리턴할 값이 있으면 이를 지정해준다. 메소드가 호출되면 예외를 강제로 던지게 만들 수도 있다.
- 테스트 대상 오브젝트에 DI 해서 목 오브젝트가 테스트 중에 사용되도록 만든다.
- 테스트 대상 오브젝트를 사용한 후에 목 오브젝트의 특정 메소드가 호출됐는지, 어떤 값을 가지고 몇번 호출됐는지를 검증한다.

## 6.3 다이내믹 프록시와 팩토리 빈

### 6.3.1 프록시와 프록시 패턴, 데코레이터 패턴

#### 프록시 패턴

자신이 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것을 대리자, 대리인과 같은 역할을 한다고 해서 프록시라고 부른다.

- 클라이언트가 타깃에 접근하는 방법을 제어하기 위해
- 타깃에 부가적인 기능을 부여해주기 위해

#### 데코레이터 패턴

타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴

프록시와 타깃이 어떤 방법과 순서로 연결되는지 정해져 있지 않다. 

타깃의 코드를 손대지 않고, 클라이언트가 호출하는 방법도 변경하지 않은 채로 새로운 기능을 추가할 때 유용한 방법

#### 디자인 패턴과 용어 차이

프록시 패턴의 프록시는 타깃의 기능을 확장하거나 추가하지 않는다. 대신 클라이언트가 타깃에 접근하는 방식을 변경해준다.

![스크린샷 2022-11-08 오후 5.18.03.png](imgs/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-11-08_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_5.18.03.png)

### 6.3.2 다이내믹 프록시

일일이 프록시 클래스를 정의하지 않고 몇가지 API를 이용해 프록시처럼 동작하는 오브젝트를 다이내믹하게 생성하는 것이다.

첫번째는 타킷의 인터페이스를 구현하고 위임하는 코드를 작성하기 번거롭다

두번째는 부가기능 코드가 중복될 가능성이 많다

#### 리플렉션

자바의 코드 자체를 추상화해서 접근하도록 만든다.

```java
public class UppercaseHandler implements InvocationHandler {
	Object target;

	public UppercaseHandler(Object target){
		this.target = target;
	}

	public Object invoke(Object proxy, Method method, Object[] args)
		throws Throwable {
		if (ret instanceof String){
			return ((String)ret).toUpperCase();
		} else {
			return ret
		}
	} 
}
```

### 6.3.3 다이내믹 프록시를 이용한 트랜잭션 부가기능

```java
public class TransactionHandler implements InvocationHandler {
	private Object target;  
	private PlatfornTrensactionManager transactionManager;
	private String pattern;
	public void setTarget(Object target) { 
		this.target = target;
	}
	public void setTransactionManager(PlatformTiroanMnasnagcetr transactionManager) {
		this.transactionManager = transactionManager;
	}
	public void setPattern(String pattern) {
		this pattern = pattern;
	}
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable { 
		if (method.getName().startsWith(pattern)) {
			return invokeInTransaction(method, args);
		} else { 
			return method, invoke(target, args);
		}
	private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
		TransactionStatus status = this. transactionManager.getTransaction(new DefaultTransactionDefinition());
		try {
			Object ret = method.invoke(target, args); 
			this.transactionManager.commit(status);
			return ret
		} catch (InvocationTargetException e) {
			this.transactionManager.rollback(status);
			throw e.getTargetException();
		}
	}
}
```

### 6.3.4 다이내믹 프록시를 위한 팩토리 빈

```java
public interface FactoryBean<T> {
    T getObject() throws Exception; // 빈 오브젝트 리턴
    Class<? extends T> getObjectType(); // 생성하는 오브젝트의 타입
    boolean isSingleton();
}
```

![스크린샷 2022-11-08 오후 6.46.50.png](imgs/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-11-08_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_6.46.50.png)

팩토리 빈은 다이내믹 프록시가 위임할 타킷 오브젝트인 UserServiceImpl에 대한 레퍼런스를 프로퍼티를 통해 DI 받아줘야 한다.

생성 후 생성 후 클라이언트가 받는다.

### 6.3.5 프록시 팩토리 빈 방식의 장점과 한계

#### 프록시 팩토리 빈의 장점

- 프록시를 적용할 대상의 모든 메소드를 구현하지 않아도 된다.
- 부가기능 코드의 중복을 해결할 수 있다.

#### 프록시 팩토리 빈의 한계

- 비즈니스 로직을 담은 많은 클래스의 메소드에 적용되려면 설정이 중복된다.
- 하나의 타깃에 여러 개의 부가기능을 적용할때 부가기능 수만큼 설정해줘야한다.

## 6.4 스프링의 프록시 팩토리 빈

### 6.4.1 ProxyFactoryBean

스프링에서는 앞선 서비스 추상화를 제공해준다.

포인트 컷

부가기능 적용 대상을 선정하는 방법이다.

어드바이스

타깃이 필요없는 순수한 부가기능

포인트컷 + 어드바이스 → 어드바이저

### 6.4.2 ProxyFactoryBean 적용

어드바이스와 포인트컷도 모두 독립적인 기능이으로 추상화기법이 적용되어 있기 때문에 재사용하여 메소드 선정 방식이 바뀔 경우 포인트컷의 설정을 따로 등록하는 방식으로 조합해서 적용해 줄 수 있다.