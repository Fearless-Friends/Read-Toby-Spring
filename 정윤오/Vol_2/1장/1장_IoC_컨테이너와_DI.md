# 목차    
- [1. IoC 컨테이너와 DI](1.-IoC-컨테이너와-DI)
    - [1.1 IoC 컨테이너: 빈 팩토리와 애플리케이션 컨텍스트](1.1-IoC-컨테이너:-빈-팩토리와-애플리케이션-컨텍스트)
    - [1.2 IoC/DI를 위한 빈 설정 메타정보 작성](1.2-IoC-DI를-위한-빈-설정-메타정보-작성)
    - [1.3 프로토타입과 스코프](1.3-프로토타입과-스코프)
    - [1.4 기타 빈 설정 메타정보](1.4-기타-빈-설정-메타정보)

<BR>

# **1.1 IoC 컨테이너: 빈 팩토리와 애플리케이션 컨텍스트**
스프링 애플리케이션에서는 오브젝트의 생성, 관계설정, 사용, 제거의 작업을 애플리케이션 코드 대신 독립된 컨테이너가 담당함        
컨테이너가 코드 대신 제어권을 가진다 -> IoC (Inversion of Control)      
스프링 컨테이너가 위 역할을 수행하기 때문에, 스프링 컨테이너를 IoC 컨테이너라고 부르는 것       

스프링에서 IoC를 담당하는 컨테이너를 **빈 팩토리** 혹은 **애플리케이션 컨텍스트** 라고 부르는데,        
이를 런타임 DI 지정의 관점으로 보면 빈 팩토리의 역할을 수행하고, 여기에 개발을 위해 필요한 여러 컨테이너 기능을 추가한 것을 애플리케이션 컨텍스트 라고 부른다.      
애플리케이션 컨텍스트 = IoC, DI를 위한 빈 팩토리이면서 그 이상의 기능을 수행함 

코드의 관점에서 이야기하면?     
-> ApplicationContext interface를 구현한 클래스의 오브젝트를 의미       

<BR>

### **설정 메타정보**
Printer 라는 인터페이스가 있고, 이 인터페이스의 구현체인 StringPrinter, ConsolePrinter가 있다고 해보자.     
User라는 독립된 클래스가 Printer 인터페이스를 이용하려고 할 때, 이 클래스들 중에서 애플리케이션이 어떤 클래스를 사용할지 정하는 과정을 보자.        

스프링 컨테이너가 관리하는 오브젝트를 Bean이라고 한다.      
스프링의 설정 메타정보는 BeanDefinition 인터페이스로 표현되는 추상 정보로, XML, Annotation, Source 등 다양한 정보로부터 생성될 수 있다.     
스프링 IoC 컨테이너는 설정 메타정보를 읽어들인 후에 이를 참조해서 Bean 오브젝트를 생성한다.     
이것이 IoC 컨테이너의 역할이다.     

요약해서, 스프링 애플리케이션이란?        
> **POJO 클래스와 설정 메타정보를 이용해 IoC 컨테이너가 만들어주는 오브젝트의 조합**

<BR>

### **스프링 애플리케이션 동작 과정**
현재 스프링 프로젝트는 대부분 WebApplicationContext를 이용한다. (스프링 IoC 컨테이너가 구현한 컨텍스트)      
웹 환경에서 스프링이 동작하는 흐름을 알아보자.      

1. Client Request를 통해 Servlet Container에 요청
2. Servlet Container가 ApplicationContext 생성
3. Servlet이 ApplicationContext 조회
4. ApplicationContext가 설정 메타정보 조회 후 POJO 생성 
5. Servlet이 POJO 호출 및 이용

일반적으로는 위와 같이 진행된다.
위 과정을 스프링에서는 DispatfcherServlet 이라는 이름의 Servlet이 수행해준다.       

<BR>

### **웹 애플리케이션의 IoC 컨테이너 구성**
웹 애플리케이션의 컨텍스트 구성 방법은 여러가지가 있다.     
- 서블릿 컨텍스트와 루트 애플리케이션 컨텍스트 계층구조
- 루트 애플리케이션 컨텍스트 단일 구조
- 서블릿 컨텍스트 단일 구조

<BR>

# **1.2 IoC/DI를 위한 빈 설정 메타정보 작성**
컨테이너는 자신이 만들 오브젝트가 무엇인지 어떻게 알 수 있을까?          
빈 설정 메타정보는 여러개가 있지만, 그 중에 가장 중요한 것은 클래스 이름이다.       
다른 값은 필수 항목이 아니지만, 클래스 이름은 추상 빈으로 정의하지 않는 이상 필수 값이기 때문이다.              

<BR>

## **빈 등록 방법**
가장 원시적인 방법은 **BeanDefinition 오브젝트를 직접 구현**하는 것이다.        
그러나 스프링을 확장해서 새 프레임워크를 만드는게 아니라면, 이 방법은 추천하지 않는다.      
일반적으로 사용하는 방법은 다음과 같다.
- XML
- Stereotype Annotation
- @Configuration Class의 @Bean 
- 일반 Class(POJO)의 @Bean

<BR>

결국 XML을 이용하거나, JAVA 코드에 의한 설정으로 Bean을 등록하는 방법이 있는 것이다.        
대부분은 아래와 같은 장점으로 JAVA 코드에 의한 설정(스프링 3.0부터 가능)을 이용한다.     
- 컴파일러나 IDE를 통한 타입 타입 검증
- IDE의 자동 완성 기능 이용
- 직관적인 이해
- 복잡한 Bean 설정이나 초기화 작업 손쉽게 적용      

<BR>

## **빈 의존관계 설정 방법**
크게 2가지 방법이 있고, Bean 등록 방법까지 고려하면 총 8가지 방법이 된다.       
XML이 아닌 JAVA Code의 관점에서 보면 아래와 같다.     
- 명시적으로 DI할 대상을 지정
    - @Resource
    - @Qualifier
- 규칙에 따라 자동으로 선정
    - @Autowired

<BR>

## **프로퍼티 값 설정 방법**    
값을 넣는 방법도 빈 등록과 마찬가지로 4가지로 구분할 수 있다.       
- XML
- 일반 JAVA code의 @Value
- Configuration Class의 @Value
- PropertyEditor & ConversionService

<Br>

## **컨테이너가 자동 등록하는 빈**
아래 빈은 언제든지 일반 빈에서 DI 받아서 사용할 수 있다.        
- ApplicationContext
- BeanFactory
- ResourceLoader
- ApplicationEventPublisher
- systemProperties (Properties type)
- systemEnvironment (Map type)      

아래 두개는 타입이 아니라 이름을 통해 접근 가능하다.        

<BR>

# **1.3 프로토타입과 스코프**
기본적으로 스프링 Bean은 싱글톤으로 만들어짐        
그러나 프로토타입 스코프로 빈을 선언하면, 컨테이너에게 빈을 요청할 때 마다 매번 새로운 오브젝트를 생성하는 형태     

<BR>

## **프로토타입 빈의 용도**
new로 오브젝트를 생성하는 것을 대신하기 위한 용도.          
DTO 같은 경우를 의미하는 것이 아니라, 사용자 요청에 따라 매번 독립적인 오브젝트를 만들어야하는데, 매번 새롭게 만들어지는 오브젝트가 컨테이너 내의 빈을 사용해야 하는 경우       
프로토타입 빈 이용      

하지만 이 방법이 과연 더 편할까? ...        
그냥 이런 방법도 있다. 정도

<BR>

## **DI와 DL**
DI(Dependency Injection) : 의존성 주입      
DL(Dependency Lookup) : 의존성 검색    
프로토타입 빈 생성은 DL로, DI는 Bean Object가 처음 생성될 때 한 번만 진행되기 때문      

<BR>

## **스코프의 종류**
- 요청(request)
- 세션(session)
- 글로벌세션(globalSession)
- 애플리케이션(application) 

4개 모두 웹 환경에서만 의미가 있음.     

<BR>

### **요청 스코프 빈**
하나의 웹 요청 안에서 해당 요청이 끝날 때 제거      
요청 단위로 생성 및 제거되므로 상태를 가져도 괜찮음     

<BR>

### **(글로벌) 세션 스코프 빈**
HTTP 세션과 같은 존재 범위를 갖는 빈으로 만들어주는 스코프      
사용자별로 생성, 브라우저를 닫거나 세션 타임 종료까지 유지         

<Br>

### **애플리케이션 스코프 빈**      
서블릿 컨텍스트에 저장되는 빈 오브젝트      
서블릿 컨텍스트는 웹 애플리케이션마다 만들어짐, 따라서 컨텍스트가 존재하는 동안 유지되는 싱글톤 스코프와 비슷한 존재 범위를 가짐.       
상태를 갖지 않거나, 상태가 있더라도 읽기 전용이거나 하는 등 멀티스레드에서 안전한 환경으로 만들어야함       

<BR>

## **스코프 빈의 사용방법**
```java
@Scope("session")
public class LoginUser {
    String loginId;
    String name;
    Date loginTime;
    ...
}
```

<BR>

# **1.4 기타 빈 설정 메타정보**

### **빈의 식별자**     
- id
- name      

위 두가지로 식별할 수 있다.
id는 유니크한 값, name은 여러 값을 가질 수 있다.        

<BR>

### **애노테이션의 빈 이름**
@Component와 같은 스테레오 타입의 애노테이션을 부여하고, 빈 스캐너에 의해 자동인식 되도록 만든 경우에는 보통 클래스 이름을 그대로 빈 이름으로 사용하는 방법을 선호한다.     
```java
@Component
public class UserService {...}
```

물론 지정할 수 있다.        
```java
@Component("myUserService")
public class UserService {...}
```

@Configuration 내 @Bean 메소드를 이용해 빈을 정의하는 경우도 마찬가지이다.
```java
@Configuration
public class Config {
    @Bean
    public void UserDao userDao() {...}
}
```

위와 동일하게 지정할 수 있다.
```java
@Configuration
public class Config {
    @Bean(name={"myUserDao", "userDao"})
    public void UserDao userDao() {...}
}
```

<BR>

## **빈 생명주기 메소드**       
### **초기화 메소드**   
Bean Object가 생성되고 DI 작업까지 마친 다음에 실행되는 메소드      
초기화 메소들르 지정하는 방법은 4가지가 있다.       
- 초기화 콜백 인터페이스        
    InitializingBean 인터페이스를 구현해서 빈을 작성하는 방법       
    프로퍼티 설정을 마친 후에 afterPropertiesSet() 메소드가 호출된다

- init-method 지정      
    XML을 이용해 빈을 등록한다면 <bean> 태그에 init-method 애트리뷰트를 넣어서 초기화 작업을 수행할 메소드 이름을 지정  

- @PostConstruct            
    초기화를 담당할 메소드에 @PostConstruct 애노테이션을 부여

- @Bean(init-method)        
    @Bean 애노테이션의 init-method 엘리먼트를 사용해서 초기화 메소드 지정 가능

ex)
```java
@Bean(init-method="initResource")
public void MyBean myBean() {...};
```

<BR>

### **제거 메소드**
컨테이너가 종료될 때 호출해서 빈이 사용한 리소스를 반환하거나 종료 전 처리해야할 작업 수행      
- 제거 콜백 인터페이스      
    DisposableBean 인터페이스를 구현해서 destroy()를 구현하는 방법      
    스프링 API에 종속되는 코드를 만드는 단점이 있음

- destroy-method        
    <bean> 태그에 destroy-method를 넣어서 제거 메소드 지정

- @PreDestroy       
    컨테이너가 종료될 때 실행될 메소드에 @PreDestroy를 붙여주면 됨

- @Bean(destroyMethod)
    @Bean 애노테이션의 destroyMethod 엘리먼트를 이용해서 제거 메소드를 지정할 수 있음       

<BR>

## **팩토리 빈과 팩토리 메소드**
생성자 대신 오브젝트를 생성해주는 코드의 도움을 받아서 빈 오브젝트를 생성하는 것을 팩토리 빈이라고 부름     
- FactoryBean Interface         
- Static Factory Method
- Instance Factory Method   
- @Bean 메소드

<BR>

# **1.5 스프링 3.1의 IoC 컨테이너와 DI**
스프링의 가장 큰 장점 : 버전이 업그레이드 되어도 **과거 기술과 완벽에 가깝게 호환**     
-> 스프링 3.1에서 2.x나 1.x 방식 (xml 등)을 사용해도 작동한다

스프링 3.1에 아래 2가지 IoC/DI 기술이 새롭게 도입
- 강화된 자바 코드 빈 설정
    - XML을 더 이상 사용하지 않아도 됨
- 런타임 환경 추상화

자바 코드 빈 설정이 강화됐다는데, 어떻게 강화했을까?        
빈의 종류부터 한 번 보도록 하자

<BR>

### **빈의 종류**
- **애플리케이션 로직 빈**
    - 스프링 IoC/DI 컨테이너에 의해 생성되고 관리되는 Bean
    - 일반적으로 우리가 만드는 Bean들 대부분 해당
    - DAO, Service, Repository, etc...
- **애플리케이션 인프라 빈**
    - 스프링 컨테이너에 등록되긴 하지만 성격이 다른 Bean
    - 애플리케이션의 로직을 담당하지 않고, 애플리케이션의 로직 빈을 지원하는 형태
    - DataSource, DataSourceTransactionMananger 등이 해당
- **컨테이너 인프라 빈**
    - 스프링 컨테이너에 등록되지만, 애플리케이션 로직을 담고 있지도 않고 다른 애플리케이션 로직을 담은 빈과 관계를 맺고 있지 않는 Bean
    - DefaultAdvisorAutoProxyCreator 등이 해당 (AOP 지원)

<BR>

## **컨테이너 인프라 빈과 전용 태그**   
컨테이너 인프라 빈은 <bean> 태그를 이용해 직접 빈을 등록할 수 있음          
하지만 대부분의 컨테이너 인프라 빈은 스프링 3.1 이전까지 전용 태그를 이용해서 등록했음      
ex) 전용 태그 예시
- `<context:component-scan>`
- `<context:annotation-config>`

전용태그를 추가하면 하나 이상의 빈이 추가로 등록됨          
@Configuration을 통해 Bean을 등록하는 것 또한       
`<context:annotation-config>`가 있어야 아래 Bean들이 등록되어 기능이 작동하는 형태      
- ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor
- ConfigurationClassPostProcessor
- AutowiredAnnotationBeanPostProcessor
- RequiredAnnotationBeanPostProcessor
- PersistenceAnnotationBeanPostProcessor



## **컨테이너 인프라 빈을 위한 자바 코드 메타정보**
스프링 빈을 역할에 따라 3가지로 구분하려는 이유가 무엇일까?     
-> 빈 설정 메타정보를 작성하는 방법과 전략을 각각 다르게 가져가기 위함 (이었다)

<BR>

## **IoC/DI 설정 방법의 발전**
| 버전 | 애플리케이션 로직 빈 | 애플리케이션 인프라 빈 | 애플리케이션 컨테이너 빈 |
| :---: | :--------------- | :---------------- | :------------------ |
| 스프링 1.x | `<bean>` | `<bean>` | `<bean>` |
| 스프링 2.0 | `<bean>` | `<bean>` | 전용태그 |
| 스프링 2.5 | `<bean>`, 빈 스캔 | `<bean>` | 전용태그 |
| 스프링 3.0 | `<bean>`, 빈 스캔, 자바 코드 | `<bean>`, 자바 코드 | 전용태그 |
| 스프링 3.1 | `<bean>`, 빈 스캔, 자바 코드 | `<bean>`, 자바 코드 | `<bean>`, 자바 코드 |

현재 Spring latest release는 6.x.       
참고 : https://github.com/spring-projects/spring-framework

현재는 그냥 아래와 같이 사용하면 빈 등록이 된다.        

```java
@Configuration
@ComponentScan("base.package")
public class AppConfig {...}
```
- 위처럼 base package를 등록하고 하위 패키지의 빈을 스캔하거나      
- 마커 인터페이스를 지정해서 빈 스캐닝의 기준 페이지를 지정할수도 있다
- 대부분 base package를 등록한다

<BR>

### **@Import**     
다른 @Configuration 클래스를 Bean 메타 정보에 추가할 때 사용한다.       

Config를 만들 때 BaseConfig를 작성해놓고 여러 Config(App, Data ...)들에서 공통으로 이용하고자 할때 사용하는 편     

<BR>

### **@ImportResource** 
XML에서 사용되던 주요한 전용 태그들을 자바 클래스에서 애노테이션과 코드로 대체할 수 있게 해줬지만, 스프링 시큐리티와 같은 서브 프로젝트 등 자주 사용되지 않는 일부 전용 태그는 자바 코드 방식을 지원하지 않는 경우가 있다.      
이 경우 xml로 작성하고 @ImportResource를 통해 Bean 설정을 가져올 수 있다.

```java
@Configuration
@ImportResource("/myproject/config/security.xml")
public class AppConfig {}
```

<BR>

### **@EnableXXX**
전용 태그를 이용할 수 있게 해주는 애노테이션   
- EnableAsync
- EnableWebMvc
- EnableTransactionManagement 

등이 있다.

<BR>

## **웹 애플리케이션의 새로운 IoC 컨테이너 구성**
웹 환경에서는 보통 루트 애플리케이션 컨텍스트와 서블릿 애플리케이션 컨텍스트로 분리해서 구성하는데, 루트 애플리케이션 컨텍스트는 contextClass 파라미터를 이용하여 아래와 같이 구성이 가능하다.

```xml
<context-param>
    <param-name>contextClass<param-name>
    <param-value>org.springframework.web.context.support.AnnotationConfigWebApplicationContext</param-value>
</context-param>
```

@Configuration 클래스는 contextConfigLocation를 통해 지정 가능하다.
```xml
    <param-name>contextConfigLocation<param-name>
    <param-value>myproject.config.AppConfig</param-value>
```

서블릿 컨테이너도 동일하다.

```xml
<servlet>
    ...
    <param-name>contextConfigLocation<param-name>
    <param-value>myproject.config.AppConfig</param-value>
    ...
</servlet>
```

**핵심은 위 처럼 작성하는 형태도 가능 하다는 것.**     

<BR>

## **런타임 환경 추상화와 프로파일**
스프링 3.1에 등장한 또 다른 기술인 런타임 환경 추상화에 대해 알아보자.      
대부분의 Bean Properties는 바뀌지 않지만, 애플리케이션 동작 환경에 따라 바뀌어야 하는 것들이 있다.          
-> 대표적으로 외부 리소스나 서버 환경 등       

DB를 생각해보자.        
운영 환경, 테스트환경, 개발 환경마다 각기 다른 DB를 이용하도록 Bean을 다르게 가져가야 하는 경우가 필요하다.     

<BR>

### **프로퍼티 치환지를 이용한 방법**
스프링 3.0까지는 이렇게 했다.
```xml
<bean id="dataSource" class="...SimpleDriverDataSource">
    <property name="driverClass" value="${db.driverclass}" />
    <property name="url" value="${db.url}" />
    <property name="username" value="${db.username}" />
    <property name="password" value="${db.password}" />
</bean>
```

<BR>

### **런타임 환경 추상화를 이용한 방법**
```xml
<beans profile="dev">
    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
        <property name="driverClass" value="${db.driverclass}" />
        <property name="url" value="${db.url}" />
        <property name="username" value="${db.username}" />
        <property name="password" value="${db.password}" />
    </bean>
</beans>

<beans profile="spring-test">
    <jdbc:embedded-database id="dataSource" type="HSQL">
        <jdbc:script location="schema.sql">
    </jdbc:embedded-database>
</beans>

...
```
위와 같이 분리가 가능하다.  

<BR>

### **활성 프로파일 지정 방법**
활성 프로파일을 지정한 후에 로딩하는 방법
```java
GenericXmlApplicationContext ac = new GenericXmlApplicationContext();
ac.getEnvironment().setActiveProfiles("dev");
ac.load(getClass(), "applicationContext.xml");
ac.refresh();
```

커맨드라인 이용
```shell
-Dspring.profiles.active=dev
```

<BR>

### **프로파일 전략**
1. XML
```xml
<context-param>
    <param-name>spring.profiles.active<param-name>
    <param-value>dsDev, mockMailserver</param-value>
</context-param>

...

<beans profile="dev, test">
```

2. @Profile
```java
@Configuration
@Profile("dev")
public class DevConfig {...}
```

3. 중첩 @Configuration 
```java
@Configuration
public class AppConfig {

    @Configuration
    @Profile("spring-test")
    public static class SpringTestConfig {...}

    @Configuration
    @Profile("dev")
    public static class DevConfig {...}

    @Configuration
    @Profile("production")
    public static class ProductionConfig {...}
}
```

<BR>

## **프로퍼티 소스**        
DB 연결정보의 경우 환경에 따라서 달라지게 되는데, 이 값을 직접 읽어와서 지정해줄수도 있다.      

<BR>

### **프로퍼티**
기본적으로 key-value 쌍을 의미한다
```yaml
db.username=spring
db.password=book
```

<BR>

### **스프링에서 사용되는 프로퍼티 종류**
- 환경변수
    - System.getEnv()
- 시스템 프로퍼티
    - System.getProperties()
- JNDI
    - WAS에 여러 애플리케이션이 올라가고 그중 하나의 애플리케이션에만 프로퍼티 지정시
    - <jee:jndi-lookup id="db.username" jndi-name="db.username" />
- 서플릿 컨텍스트 파라미터
    - 웹 애플리케이션 레벨의 property
    - ServletContext Bean을 주입받아 사용
    - @Autowired ServletContext servletContext;
- 서블릿 컨픽 파라미터
    - 개별 서블릿을 위한 설정 값
    - ServletConfigAware 구현 or @Autowired로 주입
    - getInitParameter()

위 프로퍼티들은 각각 받아오는 방법이 다르다보니 하나로 통합하고자 하는 니즈가 생겼다.       

<BR>

## **프로파일 통합과 추상화**  
스프링 3.1에서는 이를 프로퍼티 소스라는 개념으로 추상화하고, 프로퍼티의 저장 위치에 상관없이 동일한 API를 이용해 가져올 수 있게 하였다.     
Environment 타입의 런타임 오브젝트를 이용하면 일관된 방식으로 프로퍼티 정보를 가져올 수 있다.       

```java
// Environment Object를 통해 가져오기
@Autowired
Environment env;

private String adminEmail;

// 매번 필요한 값이라면? @PostConstruct
@PostConstruct
public void init() {
    this.adminEmail = env.getProperty("admin.email")
}
```

위 과정이 번거롭다면 아래와 @Value와 프로퍼티 치환자를 이용할수도 있다.     
```java
@Value("${db.username}")
private String username;
```

<BR>

------
