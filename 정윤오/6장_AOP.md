# 목차    
- [6. AOP](6.-AOP)
    - [6.1 트랜잭션 코드의 분리](6.1-트랜잭션-코드의-분리)
    - [6.2 고립된 단위 테스트](6.2-고립된-단위-테스트)
    - [6.3 다이내믹 프록시와 팩토리 빈](6.3-다이내믹-프록시와-팩토리-빈)
    - [6.4 스프링의 프록시 팩토리 빈](6.4-스프링의-프록시-팩토리-빈)
    - [6.5 스프링 AOP](6.5-스프링-AOP)
    - [6.6 트랜잭션 속성](6.6-트랜잭션-속성)
    - [6.7 애노테이션 트랜잭션 속성과 포인트컷](6.7-애노테이션-트랜잭션-속성과-포인트컷)
    - [6.8 트랜잭션 지원 테스트](6.8-트랜잭션-지원-테스트)

<BR>

# **6. AOP**
IoC/DI, 추상화에 이어 스프링의 3대 기반기술 중 하나인 AOP에 대해 알아보자.     

<BR>

## **6.1 트랜잭션 코드의 분리**
```java
public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }

        this.transactionManager.commit(status);
    } catch (Exception e) {
        this.transactionManager.rollback(status);
        throw e;
    }
}
```
비즈니스 로직을 처리해야하는 userService에 지저분한 트랜잭션 코드가 더 크게 붙어있었다.    
우선 이를 트랜잭션을 처리하는 부분과 실제 비즈니스를 처리하는 부분으로 분리해보자.

```java
public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
        upgradeLevelsInternal();
        this.transactionManager.commit(status);
    } catch (Exception e) {
        this.transactionManager.rollback(status);
        throw e;
    }
}

private void upgradeLevelsInternal() {
    List<User> users = userDao.getAll();
    for (User user : users) {
        if (canUpgradeLevel(user)) {
            upgradeLevel(user);
        }
    }
}
```
분리하고보니 이전보단 낫지만, 결국 Transaction을 처리하는 부분과 비즈니스를 처리하는 부분이 하나의 클래스 안에서 함수만 다르게 적용되고 있다.     
서로 데이터를 주고받지 않는 시점에서 굳이 같은 곳에 위치할 필요는 없다.    
두 부분을 분리해서 각자의 부분을 신경쓰지 않도록 해보자.

같은 로직을 적용할 수 있게 userService를 인터페이스로 분리하고, userServiceImpl과 userServiceTx로 분리하여 각 함수의 트랜잭션 부분과 비즈니스 로직 부분을 구분할 수 있을것이다.

client인 userServiceTest는 userService를 참조한 채 userServiceTx를 주입받고, userServiceTx는 userService 인터페이스를 참조한 채, userServiceImpl을 주입해주는 형태로 변경해보자.

> client -> userServiceTx -> userServiceImpl 의 형태가 된다. 

```java
public interface UserService {
    void add(User user);
    void upgradeLevels();
}
```

```java
public class UserServiceImpl implements UserService {
    UserDao userDao;
    MailSender mailSender;

    public void upgradeLevels() {
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
    }   
    ...
}

```

```java
@Setter
public class UserServiceTx implements UserService {
    UserService userService;
    PlatformTransactionManager transactionManager;

    public void upgradeLevels() {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            userService.upgradeLevels(); // UserServiceImpl 호출
            this.transactionManager.commit(status);
        } catch (Exception e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```
<BR>

## **6.2 고립된 단위 테스트**
UserService를 테스트하려고 하면 아래와 같은 오브젝트의 동작까지 동시에 고려해야한다.     
- UserDaoJdbc
    - XXXDataSource
- DSTransactionManager
- JavaMailSenderImpl
    - JavaMail

UserService를 테스트하는 것 처럼 보이지만, 실제론 그 뒤에 있는 수 많은 오브젝트와 환경 서버 등을 동시에 테스트하고, 이를 구현한 로직에 문제가 있다면 UserService 테스트가 실패하게 된다.    

이를 해결하기 위해 테스트의 대상이 외부 서버, 다른 클래스의 코드에 종속되고 영향 받지 않도록 고립시킬 필요가 있다.    

<BR>

### **Mock을 이용한 테스트 구조**
upgradeLevels() 메소드는 리턴 값이 없는 void형이다.   
따라서, 메소드를 실행하고 결과를 검증하는 것은 불가능하다.    
DB에 결과를 반영하므로, 일반적으로는 DB를 검증하지 않으면 테스트 하는 것은 불가능하다고 할 수 있다.

UserService를 UserServiceTx와 UserServiceImpl로 분리해보면, 비즈니스 로직인 UserServiceImpl의 입장에서 MockUserDao와 MockMailSender를 이용하여 각각의 오브젝트가 어떤 요청을 수행했는지 확인하는 과정을 통해 테스트 결과를 직접 반영하지 않고도 결과가 반영될 것이라고 결론을 내릴 수 있게 되는 것이다.

### **테스트 성능 향상**
이전과 비교해서 테스트 성능이 월등히 좋아진 것을 확인할 수 있다.    
단순하게 생각해보아도 DB에 연결해서 데이터를 직접 가져오지 않은 채로 검증을 수행했기 때문이다.  

### **Mock Framework**
단위 테스트 수행을 위해서는 Mock Object의 사용이 필수적이다.    
다른 로직과의 인터랙션이 없이 오로지 하나의 작업 단위로만 테스트를 수행해야하기 때문이다.

이 책에서는 MockObject를 직접 구현해서 해결하고 있지만, Mock Framework를 이용하여 MockObject를 별도로 구현하지 않고도 이용할 수 있다.

<BR>

### **예제**
```java
UserDao mockUserDao = mock(UserDao.class);
```
위와 같이 선언하면 MockObject가 만들어진다.    
그러나 아무 기능이 없다.    

행동에 따른 결과물을 리턴하도록 Stub을 추가해주어야 한다. 
```java
when(mockUserDao.getAll()).thenReturn(this.users);
```
**`mockUserDao.getAll()이 호출되면, users를 return하라`** 라는 의미의 선언이다.

update()의 호출이 있는지를 검증하고 싶다면 다음과 같이 확인하면 된다.
```java
verify(mockUserDao, times(2)).update(any(User.class));
```
User 타입의 Object를 Parameter로 받으며 update() 함수가 2번 호출됐는지 확인하라는 의미이다.

실제 Mockito의 Mock Object는 다음의 4단계를 거쳐서 사용된다.

1. 인터페이스를 이용해 MockObject를 만든다.
2. Mock Object가 Return값이 있으면 이를 지정해준다. 예외를 강제로 던지게 할 수도 있다.
3. 테스트 대상 Object에 DI해서 Mock Object가 테스트 도중 사용되도록한다.
4. 테스트 대상 오브젝트를 사용한 후 Mock Object의 특정 Method가 호출됐는지, 어떤 값을 가지고 몇 번 호출 됐는지를 검증한다.

다음은 전체 흐름을 구현한 예제이다.

```java
@Test
public void mockUpgradeLevels() {
    UserServiceImpl userServiceImpl = new UserServiceImpl();
    
    UserDao mockUserDao = mock(UserDao.class);
    when(mockUserDao.getAll()).thenReturn(this.users);
    userServiceImpl.setUserDao(mockUserDao); // DI

    MailSender mockMailSender = mock(MailSender.class);
    userServiceImpl.setMailSender(mockMailSender); // return 값이 없는 경우 더 쉽게 만들 수 있음

    userServiceImpl.upgradeLevels(); // 함수 수행

    verify(mockUserDao, times(2)).update(any(User.class));
    verify(mockUserDao).update(users.get(1)); // user.get(1)을 파라미터로 호출 된 적이 있는지?
    assertThat(users.get(1).getLevel(), is(Level.SILVER));
    verify(mockUserDao).update(users.get(3));
    assertThat(users.get(3).getLevel(), is(Level.GOLD));
}
```

<BR>

## **6.3 다이내믹 프록시와 팩토리 빈**
### **Proxy**
부가기능 코드(Tx)에서 핵심 기능(Impl)으로 요청을 위임해주는 과정에서 자신이 가진 부가적인 기능을 적용해 줄 수 있다. 위의 경우에서는 Tx 클래스가 Impl의 upgradeLevel을 호출하는 과정에서 Transaction을 적용해주는 과정이다.    
client는 Tx클래스를 주입받은 UserService interface의 upgradeLevel을 호출한다.

이처럼 마치 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것을 대리인의 역할을 한다고 해서 프록시(Proxy)라고 부른다.    

프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트를 Target 혹은 Real Subject 라고 부른다.    

프록시는 사용 목적에 따라 2가지로 구분할 수 있다.    
1. 클라이언트가 타겟에 접근하는 방법을 제어하기 위해
2. 타겟에 부가적인 기능을 제공하기 위해

두 가지 모두 프록시를 적용한다는 사실은 동일하지만, 목적에 따라 다른 디자인 패턴으로 구분한다.    

<BR>

### **데코레이터 패턴**
타겟에 부가적인 기능을 런타임 시에 다이나믹하게 부여하기 위해 프록시를 적용한 케이스다.    
케이크에 여러 데코레이터를 붙일 수 있는 것처럼, 프록시가 꼭 한개로 제한되지 않는다.

데코레이터 패턴은 인터페이스를 통해 위임하는 방식이기 때문에, 어느 데코레이터에서 타겟으로 연결될지 코드 레벨에선 알 수 없다.     
UserServiceTx도 UserService라는 인터페이스를 통해 다음 오브젝트로 연결되도록 되어 있다. UserServiceImpl이라는 특정 클래스를 가리키지 않는다는 것이다.    
또한 언제든지 Tx와 Impl 클래스 사이에 부가적인 데코레이터를 만들어서 추가해줄수도 있다.

<BR>

### **프록시 패턴**
일반적으로 용어 "프록시" 와 "프록시 패턴"은 구분할 필요가 있다.    
전자는 클라이언트와 타겟 사이에 대리를 맡은 오브젝트를 총칭하는 것이라면, 후자는 프록시 방법 중에서 타겟에 접근하는 방법을 제한한 케이스만을 가리킨다.

프록시 패턴의 프록시는 타겟의 기능을 확장하거나 추가하지 않는다, 이 대신 클라이언트가 타겟에 접근하는 방식을 변경한다.    

타겟 오브젝트 생성 프로세스가 복잡하거나, 당장 메모리를 차지할 필요가 없다면 꼭 필요한 시점에 생성해주는 형태이다. 실제 타겟 오브젝트 대신 프록시를 넘겨주고, 프록시의 메소드를 통해 타겟을 이용하려고 하면 그 때 프록시가 타겟 오브젝트를 생성하고 요청을 위임한다.

프록시는 다음과 같은 2가지 기능으로 구성된다.
- 타깃과 같은 메소드를 구현하고 있다가 메소드 호출시 타깃 오브젝트로 위임
- 지정된 요청에 대한 부가 기능 수행

앞의 예제를 통해 부가 기능과 타겟 오브젝트로 구분해보자.

```java
public class UserServiceTx implements UserService {
    UserService userService; // 타겟 오브젝트
    PlatformTransactionManager transactionManager;

    public void upgradeLevels() {
        // 부가 기능
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {

            // 위임
            userService.upgradeLevels(); // UserServiceImpl 호출

            // 부가 기능
            this.transactionManager.commit(status);            
        } catch (Exception e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```

위처럼 프록시의 역할은 위임과 부가 기능으로 확인할 수 있다.    
그렇지만 프록시를 만들기는 아래의 이유로 굉장히 번거롭다.
1. 타겟의 인터페이스를 구현하고 위임하는 코드 작성의 번거로움
2. 부가 기능 코드의 중복 가능성

위 문제를 해결하기 위해 등장한 것이 다이나믹 프록시이다.

<BR>

### **다이나믹 프록시**
다이나믹 프록시는 리플렉션을 통해 구현된다.    
프록시 팩토리에 의해 런타임시 동적으로 생성되는 오브젝트다.    
다이나믹 프록시 오브젝트는 타겟의 인터페이스와 같은 타입으로 만들어지고, 클라이언트는 타겟 인터페이스를 통해 프록시 오브젝트를 사용할 수 있다.    
다만, 부가 기능 제공 코드는 직접 작성해야한다.    

프록시 오브젝트와 독립적으로 InvocationHandler를 구현한 오브젝트에 담아서 구현한다. InvocationHandler는 invoke() 하나만 가지고 있는 인터페이스이며, 리플렉션의 Method 인터페이스를 파라미터로 받는다.

아래와 같은 형태로 구현한다.

```java
public class UppercaseHandler implements InvocationHandler {
    Hello target;

    public UppercaseHandler(Hello target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String ret = (String) method.invoke(target, args); // 타겟으로 위임
        return ret.toUpperCase(); // 대문자로 변환하는 부가기능 추가
    }
}
```

이제 프록시를 생성해보자.
```java
Hello proxiedHello = (Hello) Proxy.newProxyInstance(
    getClass().getClassLoader(), // 다이나믹 프록시 클래스 로더 
    new Class[] {Hello.class}, // 구현할 인터페이스
    new UppercaseHandler(new HelloTarget())); // 부가기능 & 위임 코드를 담은 InvocationHandler
```

현재는 리턴값이 모두 스트링이지만, 리턴값이 달라지는 경우, 혹은 그 외 함수를 구분하고 싶은 경우 아래와 같이 invoke를 작성할 수 있다.

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object ret = method.invoke(target, args);
        if (ret instanceof String && method.getName().startsWith("say")) {
            return ((String)ret).toUpperCase();  // String이면서 say로 시작하는 경우 부가기능 추가
        } else {
            return ret; // String이 아닐 경우
        }    
    }
```

위와 같은 구조를 트랜잭션에 적용해보자.
```java
@Setter
public class TransactionHandler implements InvocationHandler {
    private Object target; // 부가 기능 제공할 타겟 오브젝트
    private PlatformTransactionManager transactionManager; // 트랜잭션 매니저
    private String pattern; // 트랜잭션 적용할 메소드 이름 패턴

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().startWith(pattern)) {
            return invokeInTransaction(method, args); // 패턴에 해당되는 것만 트랜잭션 적용 
        } else {
            return method.invoke(target, args);
        }
    }

    private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
        Transaction status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            Object ret = method.invoke(target, args);
            this.transactionManager.commit(status);
            return ret;
        } catch (InvocationTargetException e) {
            this.transactionManager.rollback(status);
            throw e.getTargetException();
        }
    }
}
```

위와 같은 오브젝트를 스프링에서 어떻게 생성할까?    
다이나믹 프록시의 경우, 코드 실행시 생성되므로 사전에 등록해둘 수가 없다.    
이런 경우 팩토리 빈을 이용해 객체를 생성할 수 있다.

<BR>

### **프록시 팩토리 빈**

```java
public interface FactoryBean<T> {
    T getObject() throws Exception; // Bean Object
    Class<? extends T> getObjectType(); // 생성하는 오브젝트의 타입 제공
    boolean isSingleton();
}
```

```java
public class TxProxyFactoryBean implements FactoryBean<Object> {
    Object target; // FactoryBean의 타입과 맞춰준다
    private PlatformTransactionManager transactionManager; // 트랜잭션 매니저
    private String pattern; // 트랜잭션 적용할 메소드 이름 패턴
    Class<?> serviceInterface;

    ...

    // FactoryBean
    public Object getObject() throws Exception {
        TransactionHandler txHandeler = new TrnasactionHandler();
        // set txHandler params...

        ...

        return Proxy.newProxyInstance(
            getClass().getClassLoader(), new Class[] {serviceInterface},
            txHandler
        ); // 이 부분에서 타겟이 생성된다
    }

    ...
}
```

### **장점**
- 타겟 인터페이스를 일일이 구현하지 않아도 된다
- 부가기능 코드의 중복이 사라진다

### **한계**
- 트랜잭션과 같은 비즈니스 로직을 담은 많은 클래스에 적용해야 한다면 거의 비슷한 프록시 팩토리 빈이 중복된다
- 하나의 타겟에 여러 부가 기능을 적용할 때 부가기능의 개수만큼 프록시 팩토리 빈을 붙여야만 한다

<BR>

## **6.4 스프링의 프록시 팩토리 빈**
위의 문제에 대해 세련된 스프링의 해법을 알아보자.

스프링의 프록시 팩토리 빈은 순수하게 프록시를 생성하는 역할만을 담당하고, 타겟과 부가 기능은 외부에서 제작 후 이용할 수 있다. 
```java
ProxyFactoryBean pfBean = new ProxyFactoryBean();
pfBean.setTarget(new HelloTarget());
pfBean.addAdvice(new UppercaseAdvice());

Hello proxiedHello = (Hello) pfBean.getObject();
...
```

```java
static class UppercaseAdvice implements MethodInterceptor {
    public Object invoke(MethodInvocation invocation) throws Throwable {
        String ret = (String) invocation.proceed();
        return ret.toUpperCase();
    }
}
```

<BR>

### **어드바이스 : 타겟이 필요 없는 순수 부가기능**
타겟 오브젝트에 적용할 부가 기능을 담은 오브젝트를 어드바이스라고 한다.     
InvocationHandler를 구현했을 때와는 달리, MethodInterceptor를 이용하면 타겟 오브젝트가 등장하지 않는다. MethodInterceptor 안에 메소드 정보와 함께 타겟 오브젝트가 담긴 MethodInvocation 오브젝트가 전달된다. 이를 기반으로, 부가 기능을 제공하는 데만 집중할 수 있다.

proceed()를 실행하면, 타겟 오브젝트의 메소드를 내부적으로 실행한다.

<BR>

### **포인트컷 : 부가 기능 적용 대상 메소드 선정 방법**
앞선 예시의 startWith 함수처럼 부가 기능을 어떤 대상에게 적용할 것인지를 정하는 오브젝트를 의미한다.    
어드바이스와 포인트컷은 모두 프록시에 DI로 주입되어 사용된다.

프록시는 클라이언트로부터 요청을 받으면, 포인트컷에게 부가기능을 부여할 메소드인지 확인해달라고 요청한 후, 조건에 따라 어드바이스를 호출하는 구조이다.

간단한 예시를 보도록 하자.

```java
@Test
public void pointcutAdvisor() {
    ProxyFactoryBean pfBean = new ProxyFactoryBean();
    pfBean.setTarget(new HelloTarget());

    NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
    pointcut.setMappedName("sayH*");

    pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice())); // pointcut과 advice를 쌍으로 DI

    Hello proxiedHello = (Hello) pfBean.getObject();

    ... test
}
```

위에서 본 것처럼, Advice와 Pointcut을 합친 것을 Advisor라고 부른다.

<BR>

### **어드바이스와 포인트컷의 재사용**
pointcut과 advice를 모두 DI하는 형태로 개발을 하기 떄문에, 어드바이스와 포인트컷을 자유롭게 재사용할 수 있게 되었다.    
그 덕에 독립적이며 여러 프록시가 공유할 수 있는 어드바이스와 포인트 컷이 된 것이다.    
예시로 트랜잭션 같은 경우는 여러곳에서 동시에 사용할 수 있다는 이점이 생긴 것이다.

<BR>

## **6.5 스프링 AOP**
이제 부가 기능들을 AOP를 통해 자연스럽게 적용할 수 있게 되었다.    
그런데, 원하는 기능을 여러 Bean에 적용하고 싶다면, 위처럼 하나씩 어드바이저와 포인트컷을 DI해줘야 한다는 문제가 있다.     
한 번에 여러 종류의 Bean에 Proxy를 적용할 방법은 없는 것일까?

<BR>

### **Bean 후처리기를 이용한 자동 프록시 생성기**
DefaultAdvisorAutoProxyCreator라는 Bean 후처리기를 통해 위 문제를 해결할 수 있다.    
이름 그대로, 어드바이저를 이용한 자동 프록시 생성기이다.     
이녀석이 Bean으로 등록되어 있으면, Bean Object가 생성될 때 마다 자동으로 후처리기에 보내서 작업을 요청한다.    

```java
public interface Pointcut {
    ClassFilter getClassFilter(); // 프록시를 적용할 클래스인지?
    MethodMatcher getMethodMatcher(); // 어드바이스를 적용할 메소드인지?
}
```
위처럼 Pointcut을 통해 두 가지를 판단한다.

실제로는 AspectJExpressionPointcut을 이용하여 많이 판단한다.

<BR>

### **AOP란 무엇인가?**
앞서 문제를 해결해왔던 과정을 돌아보도록 하자.

1. 트랜잭션 경계 설정 코드를 비즈니스 로직에 담으면서 생긴 특정 트랜잭션 기술에 종속적인 문제    
    - DI를 이용해 트랜잭션을 추상화    
2. 여전히 코드 내에 트랜잭션 코드가 적용중인 사실이 드러나고, 트랜잭션이 필요한 코드마다 이를 다 적용해주어야함. 
    - DI를 이용한 데코레이터 패턴 적용으로 비즈니스 로직을 담은 클래스에 영향을 주지 않고 트랜잭션을 적용할 수 있는 구조로 변경
    - 프록시 역할을 하는 트랜잭션 데코레이터를 통해 타겟에 접근함으로서 비즈니스 로직과 트랜잭션이 서로 자유로운 관계가 됨
3. 비즈니스 로직 인터페이스의 모든 메소드마다 트랜잭션을 부여하는 프록시 클래스를 만들어야 하는 문제는 여전히 존재함
    - 다이나믹 프록시를 통해 특정 어드바이스와 포인트컷을 지정하여 자동으로 프록시 객체를 생성할 수 있게 해줌
    - 어드바이스와 포인트 컷을 프록시와 분리할 수 있게 되었음.
4. 트랜잭션의 적용 대상이 되는 Bean마다 프록시 팩토리 빈을 설정해주어야 한다는 부담이 아직 남아있음. 
    - Bean 생성 후처리 기법을 통해 조건에 맞는 class와 method에만 부가기능을 적용할 수 있게 됨

<BR>

### **부가기능의 모듈화**
트랜잭션과 같이 수 많은 기능에 부분부분 붙어 있는 부가 기능의 경우 기존의 코드를 분리하고, 모으고, 인터페이스를 도입하고, DI를 통해 의존관계를 만들어서 해결하기가 어렵다.    
이에 따라 특별한 기법을 도입해서 이를 해결해보고자 한다.

<BR>

### **AOP : Aspect Oriented Programming**
트랜잭션과 같은 부가기능을 어떻게 모듈화 할 것인가를 고민하다, 기존의 객체 지향 패러다임과는 구분되는 새로운 특성이 있다고 생각했고, 오브젝트와는 다르게 Aspect라고 부르기 시작했다.    
Aspect는 부가 기능을 정의한 어드바이스와, 어드바이스를 어디에 적용할지 결정하는 포인트컷을 함께 가지고 있다.    
AOP는 어플리케이션의 핵심적인 기능에서 부가적인 기능을 분리해서 Aspect라는 모듈로 만들어 설계하고 개발하는 방법을 이야기한다.    

<BR>

### **프록시를 이용한 AOP**

스프링은 독립적으로 개발한 부가기능 모듈을 다양한 타겟 오브젝트의 메소드에 Dynamic하게 적용하기 위해 프록시를 이용하고 있다.    
프록시로 만들어서 DI로 연결된 Bean 사이에 적용해 타겟의 메소드 호출 과정에 참여해서 부가기능을 제공하도록 만들었다.    
스프링의 AOP는 프록시 방식의 AOP라고 할 수 있다. 

<BR>

### **바이트코드 조작을 통한 AOP**
프록시 방식이 아닌 AOP도 있다.    
바로 바이트코드를 조작하는 것이다.  
AspectJ는 프록시를 사용하지 않는 대표적인 AOP 기술이다.     
*, .. 의 정규표현식을 토대로 대상이 되는 타겟 오브젝트를 뜯어 고쳐서 부가기능을 직접 넣어주는 방식으로 AOP를 실현한다.  
타겟 오브젝트의 소스코드를 수정할 수는 없으니, 컴파일된 타겟의 클래스파일을 수정하거나 클래스가 JVM에 로딩되는 시점을 가로채서 바이트코드를 조작하는 방법을 사용한다.       

왜 이런 복잡한 방법을 쓰는걸까?     
1. 바이트코드를 조작해서 타겟 오브젝트를 직접 수정하면 스프링과 같은 DI 컨테이너의 도움을 받지 않아도 AOP를 실현할 수 있기 때문. 즉, 스프링과 같은 컨테이너가 사용되지 않는 환경에서도 손쉽게 AOP 적용 가능하다.
2. 프록시 방식보다 훨씬 유연하고 강력하기 때문. 바이트코드를 수정하게 되면 오브젝트의 생성, 필드값 조회, 조작, 정적 필드 초기화 등 다양한 작업이 가능하다. 특히, Proxy 적용이 불가능한 private 메소드의 호출이나 필드 입출력 등에 부가기능을 부여하려고 하면 위 방법 밖에는 없다.       

하지만, **대부분의 AOP 작업은 프록시로 수행이 가능하다.**       
따라서 일반적인 AOP를 적용하는데에는 프록시 방식인 스프링 AOP도 충분하다.       
이 이상의 요구사항이 있을때에 AspectJ를 다시 알아봐도 충분하다는 것이다.        

<BR>

### **AOP의 용어**

- 타겟
    - 부가기능을 부여할 대상
    - 핵심 기능일 수도 있지만, 경우에 따라 프록시 오브젝트도 가능함
- 어드바이스
    - 타겟에 제공할 부가기능을 담은 모듈 
    - 오브젝트일수도 있고, 메소드일수도 있음
- 조인 포인트
    - 어드바이스가 적용될 수 있는 위치
    - 스프링 프록시 AOP에서 조인 포인트는 메소드의 실행 단계뿐임
- 포인트 컷
    - 어드바이스를 적용할 조인 포인트를 선별하는 작업, 혹은 그 기능을 정의한 모듈
    - 스프링의 조인 포인트가 메소드의 실행이므로, 스프링의 포인트 컷은 메소드를 선정하는 기능을 의미
- 프록시
    - 클라이언트와 타겟 사이에 투명하게 존재하면서 부가기능을 제공하는 오브젝트
    - DI를 통해 타겟 대신 클라이언트에 주입되며 클라이언트의 메소드 호출을 대신 받아서 타겟에 위임하고, 그 과정에서 부가기능을 부여
    - 스프링은 프록시를 이용해 AOP를 지원
- 어드바이저
    - 포인트컷과 어드바이스를 하나씩 갖고 있는 오브젝트
- 애스펙트
    - 한 개 또는 그 이상의 포인트컷과 어드바이스의 조합으로 만들어짐
    - 보통 싱글톤 형태의 오브젝트로 존재

<BR>

### **스프링의 프록시 방식 AOP 적용하기**
최소 4가지 Bean을 등록해야한다.
- 자동 프록시 생성기
    - 스프링의 DefaultAdvisorAutoProxyCreator 클래스를 Bean으로 등록
    - Application Context가 Bean Object를 생성하는 과정에 후처리기로 참여하여 Bean으로 등록된 어드바이저를 이용하여 프록시를 자동으로 생성
- 어드바이스
    - 부가 기능을 구현한 클래스를 Bean으로 등록
- 포인트컷
    - AspectJExpressionPointcut을 Bean으로 등록하고 expression Property에 포인트컷 표현식을 넣어주기
- 어드바이저
    - 스프링의 DefaultPointcutAdvisor 클래스를 Bean으로 등록해서 사용
    - 어드바이저와 포인트컷을 프로퍼티로 참조하는 것 외에 기능은 없음

어드바이스를 제외한 나머지는 스프링이 직접 제공하는 클래스를 Bean으로 등록하고 Property만 설정해주면 된다.      
이 세가지 클래스를 이용해 선언한 Bean은 AOP를 적용하면 반복적으로 등장하게 된다.

<BR>

스프링은 AOP를 위해 기계적으로 적용하는 Bean들을 간단히 적용할 수 있게 AOP 네임스페이스를 제공한다.

```xml
<aop:config>
    <aop:pointcut id ="transactionPointcut" expression="execution(* *..*ServiceImpl.upgrade*(..))" />
    <aop:advisor advice-ref="transactionAdvice" pointcut-ref="transactionPointcut" />
</aop:config>

<!-- 또는 아래와 같이 Advisor에 내장된 Pointcut도 사용 가능. -->
<!-- 하나의 포인트컷을 공유하는 형태가 아니라면 아래의 형태가 간결함. -->

<aop:config>
    <aop:advisor advice-ref="transactionAdvice" pointcut="execution(* *..*ServiceImpl.upgrade*(..))" />
</aop:config>
```

<BR>

## **6.6 트랜잭션 속성**
트랜잭션의 동작 방식을 제어할 수 있는 네 가지 속성에 대해 알아보자.

<BR>

### **트랜잭션 전파**
트랜잭션 전파란, 트랜잭션의 경계에서 이미 진행중인 트랜잭션이 있을 때, 혹은 없을 때 어떻게 동작할 것인가를 결정하는 방식을 의미한다.

예를 들어, 아래와 같은 구조가 있다고 해보자.

```java
public void methodA() {
    // begin transaction

    methodB();

    // end transaction
}

public void methodB() {
    // begin transaction
    
    // do something

    // end transaction
}
```

함수 A가 실행되고, 함수 B가 실행될 때 함수 B는 어떤 트랜잭션의 흐름을 따라야 할까?      
- A에 편승하거나
- 독자적으로 B의 트랜잭션을 타서 A에 영향을 주지 않거나

이렇게 진행중인 트랜잭션이 어떻게 영향을 미칠 수 있는가를 정의하는 것이 트랜잭션 전파 속성이다.         
아래와 같은 옵션들이 있다.

- **PROPAGATION_REQUIRED**      
    - 가장 많이 사용되는 트랜잭션 전파 속성
    - 진행중인 트랜잭션이 없으면 새로 시작하고, 이미 시작된게 있으면 이에 참여
    - DefaultTransactionDefinition의 트랜잭션 전파 속성이 이것임
- **PROPAGATION_REQUIRES_NEW**      
    - 항상 새 트랜잭션 시작
    - 독립적인 트랜잭션이 보장되어야 하는 코드에 사용함
- **PROPAGATION_NOT_SUPPORTED**
    - 트랜잭션 없이 동작하도록 만들기 
    - 일반적으로 트랜잭션 경계설정의 경우 AOP를 이용해 많은 메소드에 동시에 적용하는 형태인데, 그 중 특정 Method만 트랜잭션에서 제외할 때 사용 

그렇기 때문에 트랜잭션 시작지점의 함수가 begin()이 아닌 getTransaction()인 것이다.      
항상 시작의 의미를 담는 것이 아니라, 위의 옵션에 따라 트랜잭션의 형태가 달라지기 때문에 가져와서 확인하는 형태인 것이다.        

<BR>

### **트랜잭션 격리 수준**
모든 DB 트랜잭션은 격리 수준(isolation level)을 갖고 있어야 한다.       
서버 환경에서는 여러 트랜잭션이 동시에 진행될 수 있는데, 적절한 교통정리를 위해 하나씩 실행되면 정말 다행이지만, 실제로는 성능이 크게 떨어지게 된다.        
따라서 적절하게 격리 수준을 조절해서 가능한 많은 트랜잭션을 동시에 실행되도록 하는 것이 중요하다.       
격리 수준은 DB로 설정하기도 하지만, JDBC Driver나 DataSource에서 지정할 수도 있다.

<BR>

### **Timeout**
트랜잭션을 수행하는 시간을 설정할 수 있다.      
DefaultTransactionDefinition의 기본 설정은 제한시간이 없는 것이다.      
트랜잭션을 직접 시작할 수 있는 **PROPAGATION_REQUIRED**나 **PROPAGATION_REQUIRES_NEW**와 함께 사용해야 의미가 있다.

<BR>

### **Read Only**
읽기 전용 (Read Only)으로 설정해두면 트랜잭션 내에서 데이터를 조작하는 시도를 막아줄 수 있다. 또한 데이터 액세스 기술에 따라 성능이 향상될 수 있다.

TransactionDefinition 타입 오브젝트를 사용하면 네 가지 속성(트랜잭션 전파, isolation level, timeout, read only)을 을 이용해 트랜잭션의 동작 방식을 제어할 수 있다.      

트랜잭션 정의를 수정하려면 어떻게 할까?         
디폴트 속성을 가지고 있는 DefaultTransactionDefinition을 사용하는 대신, 외부에서 정의된 TransactionDefinition Type의 Bean을 정의해두면 Property를 통해 원하는 속성을 지정해 줄 수 있다.         

<BR>

하지만 이 방법으로 트랜잭션 속성을 변경한다면, TransactionAdvice를 사용하는 모든 트랜잭션의 속성이 한꺼번에 바뀐다는 문제가 있다.       
원하는 Method를 선택해서 독자적인 트랜잭션 정의를 이용할 수 없을까?

<BR>

### **Transaction Interceptor**
트랜잭션 인터셉터가 이 문제를 해결해준다.       
여태 사용했던 TransactionAdvice의 경우 RuntimeException이 발생하는 경우 rollback, checkedExeption이 발생하는 경우 commit을 하는 등 형태가 정해져있었다.     
하지만 Transaction Interceptor의 경우 **메소드 이름 패턴**을 통해 트랜잭션 속성을 직접 지정할 수 있다.      
트랜잭션 전파를 제외하고는 모두가 선택 옵션이고, 트랜잭션 속성 외에 아래 옵션도 추가 가능하다.
- -Exception1 : 체크 예외 중 롤백할 것 추가
- +Exception2 : 런타임 예외 중 커밋할 것 추가

트랜잭션 속성 예시
```xml
<bean id="transactionAdvice" class="org.springframework.transaction.interceptor.TransactionInterceptor" />
    <property name="transactionManager" ref="transactionManager"/>
    <property name="transactionAttributes">
        <props>
            <prop key="get*">PROPAGATION_REQUIRED,readOnly,timeout_30</prop>
            <prop key="upgrade*">PROPAGATION_REQUIRES_NEW,ISOLATION_SERIALIZABLE</prop>
            <prop key="*">PROPAGATION_REQUIRED</prop>
        </props>
    </property>
</bean>

<!-- 또는 -->

<tx:advice id="transactionAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="get*" propagation="REQUIRED" read-only="true" timeout="30" />
        <tx:method name="upgrade*" propagation="REQUIRES_NEW" isolation="SERIALIZABLE" />
        <tx:method name="*" propagation="REQUIRED" />
    </tx:attributes>
<tx:advice>
```
- get으로 시작하는 method에 대한 속성 지정
- upgrade로 시작하는 method에 대한 속성 지정
- 모든 함수에 대한 속성 지정        

우선순위는 가장 match가 잘되는 순서대로 적용됨.

만약, 읽기 전용이 아닌 method에서 get으로 시작하는 함수를 호출하는 경우에는 어떻게 될까? 쓰기와 읽기를 동시에 발생시키지는 않을까?      
다행히도, readOnly나 Timeout의 경우 처음 시작되는 경우가 아니라면 적용되지 않는다.      
이전에 진행중인 트랜잭션의 속성을 따라가므로, 위와 같은 문제는 없다.        

<BR>

### **포인트 컷과 트랜잭션 속성 적용 전략**
1. 트랜잭션 포인트컷 표현식은 타입 패턴이나 Bean 이름을 이용한다.
2. 공통된 메소드 이름 규칙을 통해 최소한의 트랜잭션 어드바이스와 속성을 정의한다.
    - 데이터를 읽어오는 get* 형태의 method에 read-only 옵션을 준다면 성능상 이점을 누릴 수 있다.
3. 프록시 방식 AOP는 같은 타겟 오브젝트 내의 메소드를 호출할 때 적용되지 않는다.
    - 예를들어, 프록시를 통해 delete나 update를 호출하는 경우는 문제가 없지만, 타겟 오브젝트가 자체적으로 update를 호출하는 경우 프록시의 Transaction 부가기능이 적용되지 않는다.
    - 스프링 API를 이용해 프록시 오브젝트에 대한 레퍼런스를 가져오고, 같은 오브젝트의 메소드 호출도 프록시를 거치도록 강제하는 방법이 있으나, 복잡한 로직이 다시 수면위로 드러나는 것이므로 바람직하지는 않다.
    - 위 방법보다는 AspectJ와 같이 타겟의 바이트코드를 조작하는 형태로 AOP를 적용하는 방식을 적용하는 것이 보다 바람직한 해결책이다.

트랜잭션 경계설정의 부가기능을 여러 계층에서 중구난방으로 적용하는 것은 좋지 않다.      
일반적으로, 비즈니스 로직을 담고 있는 서비스 계층 오브젝트의 메소드가 트랜잭션 경계를 부여하기에 가장 적절하다.      

<BR>

## **6.7 애노테이션 트랜잭션 속성과 포인트컷**

### **@Transactional**
```java
@Target({ElementType.METHOD, ElementType,TYPE}) // 애노테이션을 사용할 대상 지정. 여기처럼 메소드, 타입 등 여러가지를 동시에 지정할 수 있다.
@Retention(RetentionPolicy.RUNTIME) // 애노테이션 정보가 언제까지 유지되는지? -> Runtime으로 지정하면 런타임때도 애노테이션 정보를 리플렉션을 통해 획득 가능
@Inherited // 상속을 통해서 애노테이션 정보를 얻을 수 있음
@Documented
public @interface transactional {
    String value() default "";
    Propagation propagation() default Propagation.REQUIRED;
    Isolation isolation() default Isolation.DEFAULT;
    int timeout() default TransactionDefinition.TIMEOUT_DEFUALT;
    boolean readOnly() default false;
    Class<? extends Throwable>[] rollbackFor() default {};
    String[] rollbackForClassName() default {};
    Class<? extends Throwable>[] noRollbackFor() default {};
    String[] noRollbackForClassName() default {};
} 
```
@Transactional의 타겟이 메소드와 타입이다.       
따라서, 메소드, 클래스, 인터페이스에 사용이 가능하다.       
기본값들이 다 존재한다.

<BR>

### **@Transactional의 대체(fallback) 정책**
스프링은 @Transactional을 적용할 때 4단계의 대체 정책을 이용하게 된다.      
메소드의 속성을 확인할 때, 타겟 메소드, 타겟 클래스, 선언 메소드, 선언 타입의 순서에 따라서 @Transactional이 적용되었는지 확인한다.

우선순위 예시
```java
// @Transactional : 4순위
public interface Service {
    // @Transactional : 3순위
    void m1();
    // @Transactional : 3순위
    void m2();
}

// @Transactional : 2순위
public class ServiceImpl implements Service {
    // @Transactional : 1순위 
    public void m1() {

    }
    // @Transactional : 1순위
    public void m2() {

    }
}
```
이를 통해 코드 반복을 줄일수가 있다, 예를 들어 클래스(2순위)에 @Transactional을 선언하고, 추가적인 트랜잭션 속성이 필요한 Method(1순위)에만 별도로 선언하는 등의 형태로 수행할 수 있다. 

적용 예제
```java
@Transactional // 전체에 기본 옵션 적용
public interface UserService {
    void add(User user);
    void deleteAll();
    void update(User user);
    void upgradeLevels();

    @Transactional(readOnly=true) // 읽기 전용 지정하여 적용
    User get(String id);

    @Transactional(readOnly=true)
    List<User> getAll();
}
```

<BR>

## **6.8 트랜잭션 지원 테스트**
AOP를 이용해 코드 외부에서 트랜잭션의 기능을 부여해주고 속성을 지정할 수 있게 하는 방법을 **선언적 트랜잭션** 이라고 함.        
반대로, TransactionTemplate이나 개별 데이터 기술의 트랜잭션 API를 사용해 직접 코드 안에서 사용하는 방법은 프로그램에 의한 트랜잭션(Programmatic Transaction)이라고 한다.        

위 두가지 방법은 혼용이 가능하다.
예를 들면 아래와 같다.
```java
@Test
public void transactionSync() {
    DefaultTransactionDefinition txDefinition = new DefaultTransactionDefinition();
    TransactionStatus txStatus = transactionManager.getTransaction();

    userService.deleteAll(); // 선언적 트랜잭션을 포함하고 있는 Method
    userService.add(users.get(0)); 
    userService.add(users.get(1));

    transactionManager.commit(txStatus); 
}
```

<BR>

### **롤백 테스트**
현재 테스트의 가장 큰 문제점은, DB가 비워지고 임의의 데이터가 추가된다는 점이다.        
이를 롤백 기능을 통해 보다 자유롭게 테스트 할 수 있게 된다.

예를 들면, 
```java
@Test
public void transactionSync() {
    DefaultTransactionDefinition txDefinition = new DefaultTransactionDefinition();
    TransactionStatus txStatus = transactionManager.getTransaction();

    try {
        userService.deleteAll(); // 선언적 트랜잭션을 포함하고 있는 Method
        userService.add(users.get(0)); 
        userService.add(users.get(1));
    } finally {
        transactionManager.rollback(txStatus); // 어떤 짓을 해도 반영안됨
    }
}
```

### **트랜잭션 직접 지정하기**
아래와 같이 직접 지정할수도 있다.       
단, 테스트 함수에서는 Transactional 어노테이션이 있어도 강제로 롤백되는 옵션이 기본으로 설정되어 있다.
```java
@Test
@Transactional
public void test() {
    ...
}
```

이는 @Rollback을 통해 해결할 수 있다.       
강제 Rollback 옵션을 false로 지정하면 된다.
```java
@Test
@Transactional
@Rollback(false)
public void test() {
    ...
}
```

클래스 레벨에 지정된 경우 아래와 같이 해결 가능하다.
```java
@Transactional
@TransactionConfiguration(defaultRollback=false)
public class UserServiceTest {
    @Test
    @Rollback // 재설정 가능
    public void test() {
        ...
    }
}
```