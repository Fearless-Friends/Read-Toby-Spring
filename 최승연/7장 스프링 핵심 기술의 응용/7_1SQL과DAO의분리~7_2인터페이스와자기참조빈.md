# 범위
7장 스프링 핵심 기술의 응용

## 학습목표
IoC/DI, 서비스 추상화, AOP를 애플리케이션 개발에 활용해서 새로운 기능 만들어보기  
이를 통해, 스프링의 개발철학과 추구하는 가치, 스프링 사용자에게 요구되는 게 무엇인지 알아보기

## 다루는 범위
7.1 SQL과 DAO의 분리  
7.2 인터페이스와 자기참조 빈

# WHY
DB의 테이블, 필드 이름, SQL문은 바뀔 수 있다.  
현재 `UserDao` 로직은 sql문을 포함하기 때문에, 변화가 발생하면 코드를 수정해야 한다.

이런 변화에 유연하게 대처할 순 없을까?

<details>
<summary>현재 UserDao 로직 보기</summary>

```java
public interface UserDao {
    void add(User user);

    User get(String id);

    List<User> getAll();

    void deleteAll();

    int getCount();

    void update(User user);
}
```

```java
public class UserDaoJdbc implements UserDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;


    private final RowMapper<User> userMapper =
            (rs, rowNum) -> {
                User user = new User();
                user.setId(rs.getString("id"));
                user.setName(rs.getString("name"));
                user.setPassword(rs.getString("password"));
                user.setLevel(Level.valueOf(rs.getInt("level")));
                user.setLogin(rs.getInt("login"));
                user.setRecommend(rs.getInt("recommend"));
                return user;
            };


    public void add(final User user) throws DuplicateUserIdException {
        try {
            this.jdbcTemplate.update("insert into users(id, name, password, level, login, recommend) " +
                            "values(?, ?, ?, ?, ?, ?)",
                    user.getId(), user.getName(), user.getPassword(),
                    user.getLevel().intValue(), user.getLogin(), user.getRecommend());
        } catch (DataAccessException e) {
            throw new DuplicateUserIdException(e);
        }
    }

    public User get(String id) {
        return this.jdbcTemplate.queryForObject("select * from users where id = ?",
                new Object[]{id},
                this.userMapper
        );
    }

    public List<User> getAll() {
        return this.jdbcTemplate.query("select * from users order by id",
                this.userMapper
        );
    }

    public void deleteAll() {
        this.jdbcTemplate.update(
                con -> con.prepareStatement("delete from users")
        );
    }

    public int getCount() {
        return this.jdbcTemplate.queryForObject("select count(*) from users", Integer.class);
    }

    @Override
    public void update(User user) {
        this.jdbcTemplate.update(
                "update users set name = ?, password = ?, level = ?, login = ?, " +
                        "recommend = ? where id = ?", user.getName(), user.getPassword(),
                user.getLevel().intValue(), user.getLogin(), user.getRecommend(), user.getId()
        );
    }
}
```

</details>

# WHAT
DAO에 있는 `DB테이블과 필드정보를 담고 있는 SQL`을 분리하자!

# HOW
## sol1: XML 설정을 이용한 분리
### 구현전략 v0)  개별 SQL 프로퍼티 방식
각 SQL 쿼리문을 스프링의 xml 설정파일로 분리
- 빈으로 주입받아 사용하기

```java
// DAO 클래스
private String sqlAdd;

public void setSqlAdd(String sqlAdd){
    this.sqlAdd=sqlAdd;
}

@Override
public void add(User user){
    jdbcTemplate.update(sqlAdd,
        user.getId(),user.getName(),user.getPassword(),
        user.getLevel().intValue(),user.getLogin(),user.getRecommend(),user.getEmail());
}
```
```xml
// 설정 파일
<bean id="userDao" class="dao.UserDaoJdbc">
  <property name="dataSource" ref="dataSource"/>
  <property name="sqlAdd" value="insert into users(id, name, password, level, login, recommend, email) values(?,?,?,?,?,?,?)"/>
</bean>
```
#### 단점
1. SQL이 필요할 때마다 매번 프로퍼티를 추가하고 -> DI를 위한 변수와 수정자 메서드를 추가해야함  
== 귀찮다

### 구현전략 v1)  XML 설정을 이용한 분리 - SQL 맵 프로퍼티 방식
sol1 + SQL을 하나의 컬렉션으로 담아두기
- 맵을 이용하면 키 값을 이용해 SQL 문장을 가져옴 -> 프로퍼티는 하나만 만들어도 되기 때문에 DAO의 코드는 간결해짐

```java
// DAO 클래스
private Map<String, String> sqlMap;

public void setSqlMap(Map<String, String> sqlMap) {
    this.sqlMap = sqlMap;
}

@Override
public void add(User user) {
        jdbcTemplate.update(sqlMap.get("add"),
        user.getId(), user.getName(), user.getPassword(),
        user.getLevel().intValue(), user.getLogin(), user.getRecommend(), user.getEmail()
        );
}
```

```xml
// 설정 파일
<bean id="userDao" class="dao.UserDaoJdbc">
  <property name="dataSource" ref="dataSource"/>
  <property name="sqlMap">
    <map>
      <entry key="add" value="insert into users(id, name, password, level, login, recommend, email) values(?,?,?,?,?,?,?)"/>
      <entry key="get" value="select * from users where id = ?"/>
      <entry key="getAll" value="select * from users order by id"/>
      <entry key="deleteAll" value="delete from users"/>
      <entry key="getCount" value="select count(*) from users"/>
      <entry key="update" value="update users set name = ?, password = ?, level = ?, login = ?, recommend = ?, email = ? where id = ?"/>
    </map>
  </property>
</bean>
```
#### 단점
1. 메서드에서 SQL을 가져올 때 문자열로 된 키 값을 사용해서 오타같은 실수가 발생될 경우, 이 메서드가 실행되기 전까지 모름
2스프링 설정파일에 SQL과 DI설정정보가 섞여 있어 직관적이지 못하고, 유지보수가 어려움.
3데이터 엑세스 로직의 일부인 SQL 문장을 애플리케이션 구성정보를 가진 설정정보와 함께 두는 건 별로임

### 구현전략 v2) 독립된 SQL 제공 서비스
요구사항: DAO가 사용할 SQL에 대한 키 값을 전달하면, 그에 해당하는 SQL을 돌려주기 
```java
public interface SqlService {
    String getSql(String key) throws SqlRetrievalFailureException;
}

public class SimpleSqlService implements SqlService {
    private Map<String, String> sqlMap;

    public void setSqlMap(Map<String, String> sqlMap) {
        this.sqlMap = sqlMap;
    }

    @Override
    public String getSql(String key) throws SqlRetrievalFailureException {
        String sql = sqlMap.get(key);
        if (sql == null) {
            throw new SqlRetrievalFailureException(key + "에 대한 SQL을 찾을 수 없습니다");
        } else {
            return sql;
        }
    }
}
```

```xml
<bean id="userDao" class="dao.UserDaoJdbc">
    <property name="dataSource" ref="dataSource"/>
  <property name="sqlService" ref="sqlService"/>
</bean>

<bean id="sqlService" class="sql.SimpleSqlService">
  <property name="sqlMap">
    <map>
      <entry key="userAdd" value="insert into users(id, name, password, level, login, recommend, email) values(?,?,?,?,?,?,?)"/>
      <entry key="userGet" value="select * from users where id = ?"/>
      <entry key="userGetAll" value="select * from users order by id"/>
      <entry key="userDeleteAll" value="delete from users"/>
      <entry key="userGetCount" value="select count(*) from users"/>
      <entry key="userUpdate" value="update users set name = ?, password = ?, level = ?, login = ?, recommend = ?, email = ? where id = ?"/>
    </map>
  </property>
</bean>
```
#### 장점
1. UserDao를 포함한 모든 DAO는 SQL을 어디에 저장해두고/ 가져오는지에 대해서 신경 쓰지 않아도 됨!! << sol1 구현방안2랑 비슷해보이지만 아주 큰 장점
    - ==  구체적인 구현 방법과 기술에 상관없이 SqlService 인터페이스 타입의 빈을 DI 받아서 필요한 SQL을 가져다 쓰기면 하면 됨
2. sqlService 빈에는 DAO에는 전혀 영향을 주지 않은 채로 다양한 방법으로 구현된 SqlService 타입 클래스를 적용이 가능함

### sol1의 단점
스프링의 XML 설정파일에서 태그 안에 SQL 정보를 넣어놓고 활용하는 것 그자체  
-> SQL을 저장해두는 전용 포멧을 가진 독립적인 파일을 이용하는 편이 좋다!

## sol2: SQL 전용 파일 분리
검색용 키와 SQL 문장 두 가지를 담을 수 있는 XML 문서 설계 후 이 XML 파일에서 SQL을 읽어뒀다가 DAO에게 제공하기
- 관련 기술: JAXB(Java Architecture for XML Binding)

```java
    public XmlSqlService() throws JAXBException {
        String contextPath = Sqlmap.class.getPackage().getName();
        System.out.println("contextPath = " + contextPath);
        try {
            JAXBContext context = JAXBContext.newInstance(contextPath);
            Unmarshaller unmarshaller = context.createUnmarshaller();
            InputStream is = getClass().getResourceAsStream("/sqlmap.xml");
            Sqlmap sqlmap = (Sqlmap) unmarshaller.unmarshal(is);

            for (SqlType sql : sqlmap.getSql()) {
                sqlMap.put(sql.getKey(), sql.getValue());
            }
        } catch (JAXBException ex) {
            throw new RuntimeException(ex);
        }
    }
```

### 장점
1. SQL 문을 스프링의 빈 설정에서 완전히 분리!
2. 독자적인 스키마를 갖는 깔끔한 XML 문서이므로 작성하고 검증하기에도 편리함. 필요시 다른 툴에서도 불러서 사용할 수 있다.
    - DBA에 의한 SQL 리뷰나 튜닝이 필요하다면 sqlmap.xml 파일만 제공하면 됨

### 단점
생성자에서 예외가 발생할 수 있는 복잡한 초기화 작업을 다루는 것은 위험함.  
-> 객체를 생성하는 중에 발생하는 예외는 다루기 힘들고, 상속하기 불편하며, 보안에도 문제가 생길 수 있다.

## sol3: 빈 초기화 적용
`@PostConstruct`을 사용해 빈 후처리기에 의해 빈 초기화를 수행하도록 한다.
```java
public class XmlSqlService implements SqlService {
    private String sqlmapFile;
    public void setSqlmapFile(String sqlmapFile) {
        this.sqlmapFile = sqlmapFile;
    }

    @PostConstruct
    public void loadSql() throws JAXBException {
        // 생략
        InputStream is = getClass().getResourceAsStream('/' + sqlmapFile);
    }
}
```

### 단점
1. 두 가지 서로 다른 관심사가 내부 구현에 고정되어 있다. 
    - XmlSqlService는 특정 포맷의 XML에서 SQL 데이터를 가져오는 방법
    - 이것을 HashMap 타입의 맵 객체에 저장하는 방법

-> 따라서 각각에 변경이 생기면 지금까지 만든 코드를 직접 고치거나 새로 만들어야함

## sol4: 인터페이스 분리
변하는 것과 변하지 않는 것을 분리하여, 유연하게 확장가능하도록 인터페이스 화
- 기존의 SqlService를 SQL 정보를 외부의 리소스로부터 읽어오는 역할 -> `SqlReader`
- SQL을 보관해두고 있다가 클라이언트의 요청이 있으면 제공해주는 역할 -> `SqlRegistry`

이때, SqlReader는 읽어온 SQL문을 등록하는 과정에서 SqlRegistry이 필요하다. SqlRegistry은 전략패턴으로 넘겨주기!

## sol5: DI
### 구현전략 v0: 자기참조 빈 - 하나의 클래스에서 모든 인터페이스 구현하기
굳이 DI를 적용하지 않더라도, 자신이 사용하는 객체의 클래스가 어떤 것인지 알지 못하게 만드는 편이 좋음  
-> 얻을 수 있는 장점: 구현 클래스를 바꾸고 의존 객체를 변경하는 등의 자유로운 확장이 가능함

책임에 따라 분리되지 않았던 XmlSqlService 클래스를 일단 `sol4` 를 바탕으로 구현
- 같은 클래스의 코드이지만 책임이 다른 코드는 직접 접근하지 않고 인터페이스를 통해 간접적으로 사용하는 코드로 변경!

```java
public class XmlSqlService implements SqlService, SqlRegistry, SqlReader {
    private SqlReader sqlReader;
    private SqlRegistry sqlRegistry;

    public void setSqlReader(SqlReader sqlReader) {
        this.sqlReader = sqlReader;
    }

    public void setSqlRegistry(SqlRegistry sqlRegistry) {
        this.sqlRegistry = sqlRegistry;
    }
```

```xml
<bean id="sqlService" class="sql.XmlSqlService">
  <property name="sqlmapFile" value="sqlmap.xml"/>
  <property name="sqlReader" ref="sqlService"/>
  <property name="sqlRegistry" ref="sqlService"/>
</bean>
```

### 구현전략 v1: 하나의 클래스에 전부 구현해둔 인터페이스를 분리하기(BaseSqlService)
SqlReader와 SqlRegistry를 완전히 분리하기!

```java
public class BaseSqlService implements SqlService {
    private SqlReader sqlReader;
    private SqlRegistry sqlRegistry;

    public void setSqlReader(SqlReader sqlReader) {
        this.sqlReader = sqlReader;
    }

    public void setSqlRegistry(SqlRegistry sqlRegistry) {
        this.sqlRegistry = sqlRegistry;
    }
    
    @Override
    public void loadSql() {
        sqlReader.read(sqlRegistry);
    }

    @Override
    public String getSql(String key) throws SqlRetrievalFailureException {
        return sqlRegistry.findSql(key);
    }
}
```

```xml
<bean id="sqlReader" class="sql.JaxbXmlSqlReader">
  <property name="sqlmapFile" value="sqlmap.xml"/>
</bean>
<bean id="sqlRegistry" class="sql.HashMapSqlRegistry"/>
<bean id="sqlService" class="sql.BaseSqlService">
  <property name="sqlReader" ref="sqlReader"/>
  <property name="sqlRegistry" ref="sqlRegistry"/>
</bean>
```

### 구현전략 v2: DI를 받지 않는 경우, 자동 적용되는 의존관계 미리 설정해두기(DefaultSqlService)
특정 의존 객체가 대부분의 환경에서 거의 디폴트라고 해도 좋을 만큼 기본적으로 사용될 가능성이 있을 경우  
-> 디폴트 의존관계를 갖는 빈을 만드는 것을 고려하기.  
-> 디폴트 의존관계가 없으면, 매번 최소 3개의 빈을 등록해야하는 점이 귀찮을 수도!

그런 상황에서는  외부에서 DI 받지 않은 경우 기본적으로 자동 적용되는 의존관계를 부여해두기!
- 주요 포인트: SqlServce를 직접 구현하는 것이 아니라, BaseSqlService를 상속받음으로써,   
필요에 따라 DI를 받을 수도 있고 / 그렇지 않은 경우는 디폴트 의존 관계를 주입해줄 수 있다.


```java
public class DefaultSqlService extends BaseSqlService {
    public DefaultSqlService() {
        setSqlReader(new JaxbXmlSqlReader()); // 디폴트 구현체를 미리 설정해둠
        setSqlRegistry(new HashMapSqlRegistry());
    }
}
```

```xml
<bean id="sqlService" class="sql.DefaultSqlService">
	<property name="sqlRegistry" ref="BurritoSqlRegistry"/>
</bean>
```

#### 단점
1. 필요에 따라 의존관계를 주입받는 경우, 불필요한 디폴트 의존객체가 생성됨
    - 해결법: 생성자 대신 @PostConstruct에서 주입된 의존객체가 있는지 확인 후 -> 없을 경우에만 디폴트 의존객체를 생성하기
