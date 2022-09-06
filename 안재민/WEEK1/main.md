# WEEK 1

## 1.1 초난감 DAO

### 1.1.1 User

User 정보를 저장하는 엔티티와, 테이블을 만든다.

```java
@Getter @Setter
public class User {
	String id;
	String name;
	String password;
}
```

**DDL**

```sql
create table user (
	id varchar(10) primary key,
	name varchar(20) not null,
	password varchar(10) not null
);
```

### 1.1.2 UserDao

User 정보를 DB에 넣고 관리할 수 있는 Dao 클래스를 만들었다

```java
public class UserDAO {

  public void add(User user) throws ClassNotFountException, SQLException {
    Class.forName('com.mysql.jdbc.Driver');
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/toby", "sa", null);

    PreparedStatement ps = 
			c.prepareStatement("insert into users(id, name, password) values (?, ?, ?)");
    ps.setString(1, user.getId());
    ps.setString(2, user.getName());
    ps.setString(3, user.getPassword());

    ps.executeUpdate();

    ps.close();
    c.close();
  }

  public User get(String userid) throws ClassNotFoundException, SQLException {
    Class.forName('com.mysql.jdbc.Driver');
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/toby", "sa", null);

    PreparedStatement ps = 
			c.prepareStatement("select * from users where id = ?");

    ps.setString(1, user.getId());

    ResultSet rs = ps.executeQuery();
    rs.next();
    User user = new User();
    user.setId(rs.getString("id");    
    user.setName(rs.getString("name");
    user.setPassword(rs.getString("password");

    rs.close();
    ps.close();
    c.close();

    return user;
  }
}
```

### 1.1.3 main()을 이용한 DAO 테스트 코드

위의 DAO 클래스가 정상적으로 동작하는지 메인 함수를  통해 테스트 코드를 작성하였다

```java
public static void main(String[] args) {
 UserDao dao = new UserDao();

 User user = new User();
 user.setId("whiteship");
 user.setName("백기선");
 user.setPassword('married');

 dao.add(user); 

    System.out.println(user.getId() + "등록 성공")

 User user2 = dao.get(user.getId()); 
 System.out.println(user2.getName());
 System.out.println(user2.getPassword());

    System.out.println(user2.getId() + "조회 성공")
}
```

## 1.2 DAO의 분리

### 1.2.1 관심사의 분리

***`Why`***

애플리케이션이 더 이상 사용되지 않아 폐기 처분 되기 전까진, 코드는 끊임없이 변화한다. 따라서 개발자는 항상 변화에 빠르고, 쉽게 대응할 수 있도록 설계하고 개발해야 한다.

***`How`***

이렇게 유지보수에 용이하게 코드를 작성하고 설계할 때엔 ***관심사에 따른 분리*** 를 해야한다

***`Then`***

관심사의 분리를 통해 객체간의 분리도를 높여 변화가 필요할 때 관련된 객체만을 수정하여 작업을 최소화 할 수 있다.

### 1.2.2 커넥션 만들기의 추출

***UserDao의 관심사항***

1. DB와의 커넥션을 가져오기
2. Statement를 만들고 실행하기
3. 리소스 오브젝트 닫아주기

```java
private Connection getConnection() throws ClassNotFountException, SQLException {
        Class.forName("com.mysql.jdbc.Driver");
        return DriverManager.getConnection("jdbc:mysql://localhost/toby", "sa", null);
    }
```

먼저 첫번째 관심사항을 담은 새로운 메소드를 만들어 관심사를 분리해주었다

이러한 리팩토링 방식을 `extract method` 라 한다.

### 1.2.3 DB 커넥션 만들기의 독립

그러나 위의 방식에서도, DB 커넥션 생성 방식이 변경됨에 따라 `getConnectinon` 메소드도 계속해서 변경되어야 한다는 문제가 있다.

따라서 메소드를 추상 메소드로 선언한 뒤, `UserDao`를 상속받은 클래스에서 각각 오버라이딩하여 사용하게 하였다.

```java
public abstract class UserDAO {
    ...
    public abstract Connection getConnection() throws ClassNotFountException, SQLException;
}
```

***디자인 패턴***

1. 템플릿 메소드 패턴
    
    > 부모 객체에서 변하지 않는 메소드들만 구현한 뒤, 구현에 따라 변경될 메소드들은 자식 객체에게 구현을 위임하는 패턴
    > 
    
    UserDao를 상속받은 클래스들이 `getConnection` 메소드들을 오버라이딩하여 사용한다.
    
2. 팩토리 메소드 패턴
    
    > 객체 생성을 직접하지 않고, 자식 클래스에게 생성 객체와 방식을 위임하는 패턴
    > 
    
    `Connection` 인터페이스 타입의 오브젝트를 `UserDao`를 상속받은 클래스들이 각 구현방식에 따라 반환한다.
    

그러나 다음과 같은 코드에도 문제점은 있다

1. DB 커넥션을 생성하는 코드를 다른 클래스에 적용할 수 없다
2. `UserDao` 객체 자체가 변화할 때 자식 클래스들에 대한 변경이 필요하다

## 1.3 DAO의 확장

### 1.3.1 클래스의 분리

따라서 종속관계로부터의 분리를 위해, DB 커넥션과 관련된 부분을 독립적인 클래스로 만들었다

```java
public class SimpleConnectionMaker {
    public Connection makeNewConnection() throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.jdbc.Driver");
        return DriverManager.getConnection("jdbc:mysql://localhost/toby", "sa", null);
    }
}
```

```java
public abstract class UserDAO {
	private SimpleConnectionMaker simpleConnectionMaker;

	public UserDAO() {
		simpleConnectionMaker = new SimpleConnectionMaker();
	}
	...
}
```

그러나 이러한 방식에도 문제점이 있다

1. `SimpleConnectionMaker` 가 변경되면, `UserDAO` 클래스 내부에서 사용되는 부분 또한 모두 변경해줘야 한다
2. `UserDAO` 클래스가 `SimpleConnectionMaker` 클래스에 종속적이기 때문에, 
DB 커넥션과 관련된 객체가 변경되면 `UserDAO` 의 변경( 현재 코드에서는 생성자에서 객체를 생성해주는 부분 )이 불가피해진다.

### 1.3.2 인터페이스의 도입

클래스는 분리하면서도 종속성을 낮추기 위해, 인터페이스를 사용한다

```java
public interface ConnectionMaker {
	public Connection makeConnection() throws ClassNotFoundException, SQLException;
}
```

```java
public class DConnectionMaker implements ConnectionMaker { ... }
public class NConnectionMaker implements ConnectionMaker { ... }
```

인터페이스 도입을 통해, 구현에 대한 자유로움이 생겼다. 

그러나 이러한 방식에도, 여전히 `UserDAO` 의 코드가 `Connection` 의 구체적인 구현클래스에 의존적인 문제가 남아있다.

### 1.3.3 관계설정 책임의 분리

의존관계에 대한 책임을 객체의 클라이언트에게 위임하여, 위와 같은 문제를 해결한다.
`UserDAO` 에서는 인터페이스만을 지니고 있기 때문에, 클라이언트에서 구체적인 구현객체와의 관계를 맺어주는 형태로 구현할 수 있다.

```java
public class UserDAO {
	...
	public UserDAO(ConnectionMaker connectionMaker) {
		this.connectionMaker = connectionMaker;
	}
}
```

이와같이 생성자를 통해 의존관계를 주입받는 형태로 변경한다.

```java
public class UserDAOTest {
	public static void main(String[] args) throws Exception {
		ConnectionMaker connectionMaker = new DConnectionMaker();
		
		UserDAO dao = new UserDAO(connectionMaker);
	}
}
```

클라이언트인 `UserDAOTest` 에서 `ConnectionMaker` 의 구현객체를 확정하여 `UserDAO` 의 생성자를 통해 두 객체를 연결한다.

이를 통해 DB 커넥션과 관련된 기능들을 `Connection` 인터페이스를 구현함으로써 자유롭게 변경이 가능해졌으며, `UserDAO` 클래스 또한 SQL을 생성하고 실행하는 작업에만 집중할 수 있게 되었다

### 1.3.4 원칙과 패턴

***OCP : 개방 폐쇄 원칙***

> 클래스나 모듈은 확장에는 열려있고, 변경에는 닫혀 있어야 한다
> 

처음의 DAO와는 달리, 현재의 DAO클래스는 SQL 생성과, 실행이라는 자신의 관심사에만 집중할 수 있으며, 연결된 다른 관심사에서의 변경이 일어나도 큰 영향을 받지 않기 때문에 OCP 원칙을 잘 지킨 것이라 할 수 있다

***높은 응집도와 낮은 결합도***

- 높은 응집도 : 하나의 모듈이나 클래스가 하나의 관심사에만 집중되어야한다
- 낮은 결합도 : 책임과 관심사가 다른 오브젝트, 모듈과는 느슨하게 연결되어 있어야한다

관심사의 분리를 통해 높은 응집도를 얻게 되어 각 객체들은 자신의 관심사에만 집중 할 수 있게 되었고, 추상화와 관계주입에 대한 책임 위임을 통해 낮은 결합도를 얻게 되어 하나의 관심사에 변경이 필요할 때 다른 관심사에 큰 영향을 미치지 않게 되었다.

***전략 패턴***

> 자신의 기능 context에서, 필요에 따라 변경이 필요한 알고리즘 클래스를 필요에 따라 통째로 외부로 분리시키고, 이를 구현한 구체적인 알고리즘 클래스를 필요에 따라 바꿔서 사용할 수 있게 하는 디자인 패턴
> 

위의 코드에서 DB 커넥션과 관련된 부분을 추상화하여 외부 클래스로 분리한 경우와 동일하다
