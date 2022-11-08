# 목차    
- [6. AOP](6.-AOP)
    - [6.1 트랜잭션 코드의 분리](6.1-트랜잭션-코드의-분리)
    - [6.2 고립된 단위 테스트](6.2-고립된-단위-테스트)
    - [6.3 다이내믹 프록시와 팩토리 빈](6.3-다이내믹-프록시와-팩토리-빈)
    - [6.4 스프링의 프록시 팩토리 빈](6.4-스프링의-프록시-팩토리-빈)




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
 