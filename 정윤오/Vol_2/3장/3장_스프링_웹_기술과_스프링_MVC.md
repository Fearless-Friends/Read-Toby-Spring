# 목차    
- [3. 스프링 웹 기술과 스프링 MVC](3.-스프링-웹-기술과-스프링-MVC)
    - [3.1 스프링의 웹 프레젠테이션 계층 기술](3.1-스프링의-웹-프레젠테이션-계층-기술)
    - [3.2 스프링 웹 애플리케이션 환경 구성](3.2-스프링-웹-애플리케이션-환경-구성)

<BR>

## **3.1 스프링의 웹 프레젠테이션 계층 기술**

### **스프링 웹 프레임워크**
- **스프링 서블릿/스프링 MVC** (아래는 스프링 서블릿 기반)
    - Spring Web Flow
    - Spring JavaScript
    - Spring Faces
    - Spring Web Service
    - Spring blazeDS Integration
- 스프링 포틀릿

### **스프링을 기반으로 두지 않는 웹 프레임워크**
- JSP/Servlet
    - new XXXHelper() 방식으로 비즈니스 로직을 처리했다면 위 형태
- Struts1
- Struts2
- Tapestry
- JSF/Seam

스프링 서블릿, 스프링 MVC 기반의 확장된 프레임 워크 권장

<BR>

### **스프링 MVC와 DispatcherServlet 전략**
프레임워크의 발전 방향은 2가지이다.
- 스프링과 같이 유연성, 확장성에 중점을 두고 특정 기술에 종속되지 않기 위한 형태
- 루비와 같이 제한적인 기술만을 사용하도록 강제하지만 기술의 장점을 극대화 시키는 형태

스프링의 유연한 확장성을 고려해 각 프로젝트에도 위와 같은 방향으로 적절히 적용하여 개발할 것!

<BR>

### **DispatcherServlet과 MVC 아키텍처**
스프링 웹 기술의 핵심이자 기반이 되는 DispatcherServlet           
스프링의 웹 기술을 구성히는 다양한 전략을 DI로 구성해서 확장하도록 만들어진 스프링 서블릿/MVC의·엔진과 같은 역할            

MVC는 아래 요소가 서로 협력하여 하나의 request를 처리하여 response를 내려주는 구조
- Model
- View
- Controller
![Alt text](image.png)

위 그림은 아래와 같은 순서로 수행된다.
- (1) DispatcherServlet의 HTTP 요청 접수
    - request를 전달받을 URL 패턴을 지정
    - 위에 해당할 경우 할당
    ```xml
        <servlet-mapping>
            <servlet-name>Spring MVC Dispatcher Servlet</servlet-name>
            <url-pattern>/app/*</url-pattern>
        </servlet-mapping>
    ```
- (2) DispatcherServlet에서 컨트롤러로 HTTP 요청 위임       
주로 Adapter Pattern 형태로 전달
![Alt text](image-1.png)
DispatcherServlet이 HanldeAdapter에 웹 요청을 전달시 HttpServletRequest 타입 오브젝트 전달      
이를 어댑터가 적절히 변환해서 컨트롤러에 전달하는 형태      
- (3) 컨트톨러의 모델 생성과 정보 등록
    - Model은 이름과 Object 쌍으로 생성 (Map)
- (4) 컨트롤러의 절과 리턴: 모멜과 뷰
    - 말 그대로 ModelAndView Object를 통해 Model, View 정보 return
- (5) DispatcherServlet의 뷰 호출과 (6) 모델 참조
    - 뷰 오브젝트에게 모댈을 전달, 클라이언트에게 돌려줄 최종 결괴물 생성 요청
    - 최종 결과물은 HttpServletResponse 오브젝트로 return
- (7) HTTP 응답 돌려주기
    - 뷰가 만들어준 HttpServletResponse를 서블릿 컨테이너에게 전달      
    - 서블릿 컨테이너는 HttpServletResponse에 담긴 정보를 HTTP 응답으로 만들어 클라이언트에 전달 및 작업 종료

<BR>

### **DispatcherServlet의 DI 가능한 전략**
- HandlerMapping
    - URL 기준 Controller 결정
- HandlerAdapter
    - Handler Mapping 정보 기준 결정
- HandlerExceptionResolver
    - 예외처리 결정
- ViewResolver
    - Controller가 return한 View 이름을 토대로 적절한 Object 탐색
- LocaleResolver
- ThemeResolver
- RequestToViewNameTranslator

<BR>

## **3.2 스프링 웹 애플리케이션 환경 구성**
서비스 계층과 데이터 액세스 계층의 코드는 실행환경에서 독립적으로 만들기 쉬움          
JavaEE 환경에서도 동작하지만 JavaSE에서 사용할 수도 있고, 간단한 테스트 환경에 서도 구동이 가능        
반면에 웹 프레젠테이션 계층은 최소한 서블릿 또는 포틀릿 컨 테이너가 제공되는 서버환경이 있어야만 동작      

아래 순서로 설정한다.
- 루트 웹 애플리케이션 컨텍스트
- 서블릿 웹 애플리케이션 컨텍스트 등록
    - URL 패턴의 경우 보통 아래와 같이 지정
        - 확장자로 구분히는 방법 (*.do. / *.action. / *.html 등)
        - 특정 url로 구분하는 방법 (RESTful 스타일에 적용)
        - /*로 지정해서 모든 request에 지정할 작업 설계 등
- Model / View / Controller 생성
- 작동 검증

```java
public interface Controller {
    ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception;
}
public class HelloController implements Controller {
    @Autowired
    HelloSpring helloSpring; 

    public ModelAndView handleRequest(HttpServletRequest req, HttpServletResponse res) throws Exception {
        String name = req.getParameter("name"); 
        String message =this.helloSpring.sayHello(name);
        Map<String, Object> model = new HashMap<String, Object>(); model.put("message", message);
        return new ModelAndView("/WEB-INF/view/hello.jsp", model);
    }
}
```

### **테스트**
- MockHttpServlet 이용
- 적절한 Model And View를 내려주는지 검증