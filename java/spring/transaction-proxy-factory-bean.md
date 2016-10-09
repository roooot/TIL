# Transaction Proxy Factory Bean
아래 코드는 Transaction Handler 를 이용하는 dynamic proxy 를 생성하는 factory bean class 이다.

### TxProxyFactoryBean.java
```java
package springbook.user.service;

import java.lang.reflect.Proxy;

import org.springframework.beans.factory.FactoryBean;
import org.springframework.transaction.PlatformTransactionManager;

// 생성할 object type 을 지정할 수 도 있지만, 범용적으로 사용하기 위해서 Object class 로 했다.
public class TxProxyFactoryBean implements FactoryBean<Object> {
	// TransactionHandler 생성할 때 필요
	Object target;
	PlatformTransactionManager transactionManager;
	String pattern;

	// Dynamic proxy 를 생성할 때 필요하다. UserService 외의 interface 를 가진 타깃에도 적용 할 수 있다.
	Class<?> serviceInterface;

	public void setTarget(Object target) {
		this.target = target;
	}

	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}

	public void setPattern(String pattern) {
		this.pattern = pattern;
	}

	public void setServiceInterface(Class<?> serviceInterface) {
		this.serviceInterface = serviceInterface;
	}

	// FactoryBean interface 구현 method
	// DI 받은 정보를 이용해서 TransactionHandler 를 사용하는 dynamic proxy 를 생성한다.
	public Object getObject() throws Exception {
		TransactionHandler txHandler = new TransactionHandler();
		txHandler.setTarget(target);
		txHandler.setTransactionManager(transactionManager);
		txHandler.setPattern(pattern);
		return Proxy.newProxyInstance(
			getClass().getClassLoader(),new Class[] { serviceInterface }, txHandler);
	}

	public Class<?> getObjectType() {
		// factory bean 이 생성하는 object type 은 DI 받은 interface type 에 따라 달라진다.
		// 따라서 다양한 type 의 proxy object 생성에 재사용 할 수 있다.
		return serviceInterface;
	}

	public boolean isSingleton() {
		// Singleton bean 이 아니라는 뜻이 아니라,
		// getObject() 가 매번 같은 object 를 return 하지 않는다는 의미이다.
		return false;
	}
}
```

Factory bean 이 만드는 dynamic proxy 는 구현 interface 나 target 의 종류에 제한이 없다. (DI를 통해서 바꿀 수 있다.) 따라서 UserService 외에도 transaction 부가 기능이 필요한 object 를 위한 proxy 를 만들 때 얼마든지 재사용이 가능하다. 설정이 다른 여러 개의 TxProxyFactoryBean bean 을 등록하면 된다.

```xml
<bean id="userService" class="springbook.user.service.TxProxyFactoryBean">
	<property name="target" ref="userServiceImpl" />
	<property name="transactionManager" ref="transactionManager" />
	<property name="pattern" value="upgradeLevels" />
	<property name="serviceInterface" value="springbook.user.service.UserService" />
</bean>
```
target, transaction property 는 다른 bean 을 가리키는 것이니 ref attribute 로 설정했고, pattern 은 string 이니 value attribute 를 사용해 값을 지정했다. serviceInterface 는 숫자, 문자와 같은 단순한 type 이 아니라 Class type 이다. 이 경우 Class Type은 value attribute 를 이용해 Class 또는 Interface 이름을 넣어주면 된다. Spring 은 setter method 의 parameter 의 type 을 확인해서 property 의 type 이 Class 인 경우는 value 로 설정한 이름을 가진 Class object 로 자동 변환해준다.

### Transaction proxy factory bean Test
Exception 발생 시 transaction 이 rollback 되는 것을 확인하려면 Exception 을 발생시키는 TestUserService object 를 target object 로 대신 사용해야 한다. Spring 설정에는 정상적인 UserServiceImpl object 로 지정되어 있지만 test method 에서는 TestUserService object 가 동작하도록 해야 한다.

하지만, Spring bean 에서 생성되는 proxy object 에 대한 reference 는 TransactionHandler object 가 갖고 있는데, TransactionHandler object 는 TxProxyFactoryBean 내부에서 만들어져 Dynamic proxy 생성에 사용될 뿐 별도로 참조할 방법이 없다. 즉, 이미 Spring bean 으로 만들어진 transaction proxy object target 을 변경하기에 어렵다.

그렇다면, bean 으로 등록된 TxProxyFactoryBean 을 직접 가져와서 proxy 를 만들어보면 된다. TxProxyFactoryBean 자체를 가져와서 target property 를 재구성 해준 뒤에 다시 proxy object 를 생성하도록 요청한다. 이렇게 하면 context 의 설정을 변경하니까, `@DirtiesContext` 를 등록해주도록 한다.

```java
public class UserServiceTest {
	...
	// FactoryBean 을 가져오려면 ApplicationContext 가 필요하다.
	@Autowired ApplicationContext context;
	...

	@Test
	// Dynamic proxy factory bean 을 직접 만들어 사용할 때는 context 무효화 annotation
	@DirtiesContext
	public void upgradeAllOrNothing() throws Exception {
		TestUserService testUserService = new TestUserService(users.get(3).getId());
		testUserService.setUserDao(userDao);
		testUserService.setMailSender(mailSender);

		// Factory bean 자체를 가져오도록 '&' 를 붙인다.
		TxProxyFactoryBean txProxyFactoryBean =
			context.getBean("&userService", TxProxyFactoryBean.class);
		// Test 용 target 주입
		txProxyFactoryBean.setTarget(testUserService);
		// 변경된 target 으로 설정하여 transaction dynamic proxy object 를 생성 한다.
		UserService txUserService = (UserService) txProxyFactoryBean.getObject();

		userDao.deleteAll();			  
		for(User user : users) userDao.add(user);

		try {
			txUserService.upgradeLevels();   
			fail("TestUserServiceException expected");
		}
		catch(TestUserServiceException e) {
		}

		checkLevelUpgraded(users.get(1), false);
	}

	...
}
```
