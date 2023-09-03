# 목차    
- [4. 스프링 @MVC](4.-스프링-@MVC)
    - [4.1 @RequestMapping 핸들러 매핑](4.1-RequestMapping-핸들러-매핑)
    - [4.2 Controller](4.2-Controller)

<BR>

# 4. 스프링 @MVC

## 4.1 @RequestMapping 핸들러 매핑
```java
// 아래와 같이 사용
@RequestMapping()
void method() { ... }

// String[] value()
@RequestMapping("/hello")

// String[] method() 
@RequestMapping(value="/hello", method=RequestMethod.GET)

// String[] params
@RequestMapping(value="/user/edit", params="type=nember")

// String[] headers 
@RequestMapping(value = "/view", headers = "content-type=text/*")
```

### **타입 레벨과 메소드 레벨의 결합**
```java
@RequestMapping("/hello")
public class ABController {
    @RequestMapping("/add") public String add( ... ) { }
    @RequestMapping("/edit") public String edit( ... ) { } 
    @RequestMapping("/delete") public String delete( ... ) { }

}
```

**@RequestMapping이 정의된 Controller를 상속받는다면?**         
- 기본적으로 정보를 그대로 상속받는다
- 서브클래스에서 재정의시 무시한다
- class -> method 순으로 Annotation이 적용된다      

<BR>

### **제네릭을 이용한 컨트롤러 작성**           
아래는 전형적인 컨트롤러이다.       
```java
public class UserController {
    UserService service;
    public void add(User user) { ... } 
    public void update(User user) { ... } 
    public User view(Integer id) { ... }
    public void delete(Integer id) { ... } 
    public List<User> list() { ... }
}
```

이를 아래와 같이 설계할 수 있다.       
물론 예시이고, 실제로는 더 신경써야한다.
```java
public abstract class GenericController<T, K, S> { 
    S service;
    public void add(T entity) { ... } 
    public void update(T entity) { ... } 
    public T view(K id) { ... }
    public void delete(P id) { ... } 
    public List<T> list() { ... }
}
```

```java
public class UserController extends GenericController<User, Integer, UserService> {
    public String login(String userld, String password) { ... }
}
```

<BR>

## **4.2 Controller**
### **메소드 파라미터의 종류**
- HttpServletRequest, HttpServletResponse
- HttpSession
- WebRequest, NativeWebRequest
- Locale
- InputStream, Reader
- OutputStream, Writer
- @PathVariable
```java
@RequestMapping("/member/{membercode}/order/{orderid}")
public String lookup(@PathVariable('membercode') String code,
@PathVariable("orderid") int orderid) { ... }
```
- @RequestParam     
여러개를 지정할 수 있다.
```java
public void view(@RequestParam(value="id", required=false, defaultValue="1") int id) { ... }

public String add(@RequestParam Map<String, String> params) { ... }

public String view(@RequestParam int id) { ... }

```
- CookieValue
- @RequestHeader        
```java
public void header(@RequestHeader('Host') String host, @RequestHeader("Keep-Alive") Long keepAlive)
```
- Map, Model, ModelMap
- @ModelAttribute 
    - @RequestParam과 비슷한 역할
    - 요청 파라미터를 메소드 파라미터에서 1:1로 받으면 @RequestParam 
    - 도메인 오브젝트나 DTO의 프로퍼티에 요청 따라미터를 바인딩해 서 한 번에 받으면 @ModelAttribute
    - 간결한 코드
    - Validation 과정 포함
```java
@RequestMapping("/user/search")
public String search(@RequestParam int id, @RequestParam String name,
@RequestParam int level, @RequestParam String email, Model
model) {
    List<User> list = userService.search(id, name, level, email);
    model.addAttribute("userList", list);
}

@RequestMapping("/user/search")
public String search(ModelAttribute UserSerarch userSearch) {
    List<User> list = userService.search(userSearch); 
    model.addAttribute("userList", list);
}
```
- Errors, BindingResult
- SessionStatus
- @RequestBody
```java
// Json Mapping
public void message(@RequstBody AClass body) { ... }
```
- @Value
    - 값 주입시
```java
public class HelloController {
    @Value('#{systemProperties['os.name')}') 
    String osName

    @RequestMapping( ... ) 
    public String hello() {
        String osName = this.osName;
    }
}
```
- @Valid
    - 데이터 검증

<BR>

### **리턴 타입의 종류**
최종적으로 DispatcherServlet에 돌아갈 때엔 ModelAndView로 return        
아래 Object는 return type에 상관없이 모델에 자동으로 추가
- @ModelAttribute Object
- @ModelAttribute Method
- Map, Model, ModelMap Params
- BindingResult

아래는 리턴 가능 타입
- String            
    - 이 return값은 View 이름으로 사용
- void      
    - RequestToViewNameResolver 전략을 통해 자동 생성되는 View 이름 적용
- Map/Model/ModelMap
- View
- @ResponseBody

<BR>

### **@SessionAttributes와 SessionStatus**
HTTP 요청에 의해 동작하는 서블릿은 기본적으로 상태를 유지하지 않는다.           
따라서 매 요청이 독립적으로 처리된다.           
하나의 HTTP 요청을 처리한후에는 사용했던 모든 리소스를 정리해버린다. 

### **배경**
- User를 Update하는 경우
- 실제 User의 모든 정보를 Update하지는 않는다 (ID 등 고윳값 제외)
- 일부 값만 업데이트할 때 User를 그대로 이용하는게 옳은가?

### **Spring의 해결방법 @SessionAttributes**
```java

@Controller
@SessionAttributes('user') 
public class UserController {
    @RequestMapping(value="/user/edit", method=RequestMethod.GET)
    public String form(@RequestParam int id, Model model) {
        model.addAttribute("user" , userService.getUser(id));
        return "user/edit";
    }

    @RequestMapping(value='/user/edit', method=RequestMethod.POST) 
    public String submit(ModelAttribute User user) { ... }
}
```
@SessionAttribute를 이용해 세션에 User 정보를 저장          
@ModelAttribute가 새 오브젝트를 바인딩 하기 전 세션에 같은 이름의 오브젝트가 존재하는지 확인 후 사용

![Alt text](image.png)

<BR>

### **SessionStatus**
위에서 이용한 데이터는 자동으로 제거되지 않음        
여러번 이용 될 가능성이 있기 때문       
그렇기에 submit같이 작업을 마무리하는 코드에서 setComplete()를 통해 세션 정리

```java
@RequestMapping(value='/user/edit', method=RequestMethod.POST)
public String submit(ModelAttribute User user, SessionStatus sessionStatus) {
    this.userService.updateUser(user);
    sessionStatus.setComplete(); // 현재 컨트툴러에 의해 세션에 저징된 정보를 모두 제거
    return "user/editsuccess";
}
```
