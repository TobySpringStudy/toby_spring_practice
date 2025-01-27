# 5장 - 서비스 추상화

## Intro

* 자바에는 표준 스펙, 상용 제품, 오픈소스를 통틀어서 사용 방법과 형식은 다르지만 기능과 목적이 유사한 기술이 존재
* 5장에서는 지금까지 만든 DAO에 트랜잭션을 적용해보면서 스프링이 어떻게 성격이 비슷한 여러 종류의 기술을 추상화하고 이를 일관된 방법으로 시용할 수 있도록 지원하는지를 살펴본다.

## 5.1. 사용자 레벨 관리 기능 추가

* 지금까진 User의 간단한 CRUD(Create-Read-Update-Delete) 기능만 추가함
* 여기에 간단한 비즈니스 로직을 추가
* 정기적으로 사용자의 활동내역을 참고해서 레벨을 조정해주는 기능
  * 사용자의 레벨은 BASIC, SILVER, GOLD 3가지
  * 사용자가 처음 가입하면 BASIC 레벨, 이후 활동에 따라 한단계씩 업그레이드
  * 가입 후 50회 이상 로그인시 BASIC -> SILVER
  * SILVER 레벨이면서 30번 이상 추천을 받으면 GOLD 레벨
  * 사용자 레벨의 변경작업은 일정한 주기를 가지고 일괄적으로 진행됨, 변경 작업 전에는 조건을 충족하더라도 레벨의 변경이 일어나지 않음

### 5.1.1. 필드 추가

* 위에 언급된 레벨은 값이 한정된 있음 - Java 의 Enum으로 처리하는 것이 안전하고 편리함
  * 임의의 정수형을 사용할 경우에는 다음과 같은 문제 발생
  * User 에 추가
  ```java
    class User {
        private static final int BASIC = 1;
        private static final int SILVER = 2;
        private static final int GOLD = 3;

        int level;

        public void setLevel(int level) {
            this.level = level;
        }
    }
  ```
  * 다음과 같이 실수로 넣을수도 있다. 직접 디버깅 해보기 전에는 찾기 힘들다.
  ```java
  user1.setLevel(other.getSum())
  ```
  * 아예 범위를 벗어나는 위험한 값도 넣을 수 있다. 직접 디버깅 해보기 전에는 찾기 힘들다.
  ```java
  user1.setLevel(1000);
  ```
  * Enum은 저런 문제가 생기지 않도록 원천 봉쇄한다.
* User 필드 추가(소스코드 참조)
* UesrDaoTest 수정(소스코드 참조)
  * JDBC가 사용하는 SQL은 컴파일 과정에서는 자동으로 검증이 되지 않는 단순한 문자열에 불과하다
  * 따라서 SQL 문장이 완성되서 DB에 전달되기 전까지는 문법 오류나 오타조차 발견하기 힘들다
  * 미리미리 DB까지 연동되는 테스트를 잘 만들어뒀기 때문에 SQL 문장에 사용될 필드 이름의 오타를 아주 빠르게 잡아낼 수 있다
* UserDaoJdbc 수정(소스코드 참조)

### 5.1.2. 사용자 수정 기능 추가

* 수정 기능 테스트 추가
  * 수정 기능을 위한 픽스처 추가(소스코드 참조)
* UserDao와 UserDaoJdbc 수정(소스코드 참조)
  * Tip - 테스트를 먼저 만들면, 아직 준비되지 않은 클래스나 메소드를 테스트 코드 내에서 먼저 사용하는 경우가 있다. 이 경우 IDE의 자바 코드 에디터에서 에러가 표시되면 자동고침 기능을 이용해 클래스나 메소드를 생성하도록 만들면 매우 편리하다.
* 수정 테스트 보완
  * JDBC 개발에서 리소스 반환과 같은 기본 작업을 제외하면 가장 많은 실수가 일어나는 곳은 바로 SQL 문장!
    * UPDATE 문장에서 WHERE 절을 빼먹는 경우
    * WHERE 절이 없어도 아무 경고 없이 정상적으로 동작하는 것처럼 보임
  * 해결책은?
    * 첫째, JdbcTempate 의 update() 가 돌려주는 int 리턴 값을 확인한다. 이 값은 테이블의 내용이 변경되는 SQL을 실행하면 영향을 받은 row의 갯수를 돌려준다.
    * 둘째, 테스트를 보강해서 원하는 사용자 외의 정보는 변경되지 않았음을 직접 확인하는 것. 사용자를 두 명 등록해놓고, 그 중 하나만 수정한 뒤에 수정된 사용자와 수정하지 않은 사용자의 정보를 모두 확인하면 됨.
    * 소스코드에서는 두번째 방법을 사용

### 5.1.3. UserService.upgradeLevels()

* 사용자 관리 로직은 어디에 두는 것이 좋을까?
  * UserDaoJdbc는 적당하지 않음. DAO는 데이터를 어떻게 가져오고 조작할지를 다루는 곳.
  * 사용자 관리 비즈니스 로직을 담을 클래스를 하나 추가함.
  * 클래스 이름은 UserService로 함
* UserService 클래스와 빈 등록

```java
// UserService.java

public class UserService {
	UserDao userDao;
	
	public void setUserDao(UserDao userDao) {
		this.userDao = userDao;
	}
}
```

```xml
<!-- applicationContext.xml -->

<bean id="userService" class="springbook.user.service.UserService">
 <property name="userDao" ref="userDao" /> 
</bean>

<bean id="userDao" class="springbook.user.dao.UserDaoJdbc">
 <property name="dataSource" ref="dataSource" /> 
</bean>
```

* UserServiceTest 테스트 클래스
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/applicationContext.xml")
public class UserServiceTest {
	@Autowired
	UserService userService;
}
```

* upgradeLevels() 메소드
* upgradeLevels() 테스트
