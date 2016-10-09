# Factory Bean
Spring 은 class 정보를 가지고 default constructor 를 통해 object 를 만드는 법 외에도 bean 을 만들 수 있는 여러가지 방법을 제공한다. 대표적으로 factory bean 을 이용한 bean 생성 방법을 들 수 있다. Factory Bean 이란 Spring 을 대신해서 object 의 생성 로직을 담당하도록 만들어진 특별한 bean 을 말한다. 
Factory bean 을 만드는 방법에는 여러 가지가 있는데, 가장 간단한 방법은 Spring 의 FactoryBean 이라는 interface 를 구현하는 것이다. FactoryBean interface 는 다음과 같이 세 가지 method 로 구성되어 있다.
```java
package org.springframework.beans.factory;

public interface FactoryBean<T> {
	// Bean object 를 생성해서 돌려준다.
	T getObject() throws Exception;
	// 생성되는 object 의 type 을 알려준다.
	Class<? extends T> getObjectType();
	// getObject() 가 return 하는 object 가 항상 같은 singleton object 인지 알려준다.
	boolean isSingleton();
}
```
FactoryBean interface 를 구현한 class 를 Spring bean 으로 등록하면 factory bean 으로 동작한다.

#### Message.java
먼저 Spring 에서 bean object 로 만들어 사용하고 싶은 Message class 가 있다. Message class 의 constructor 는 private 으로 되어 있기 때문에, Message class 의 object 를 만들려면 newMessage() 라는 static method 를 사용해야된다.
```java
package springbook.learningtest.spring.factorybean;

public class Message {
	String text;
	
	private Message(String text) {
		this.text = text;
	}
	
	public String getText() {
		return text;
	}

	public static Message newMessage(String text) {
		return new Message(text);
	}
}
```
따라서 Message class 를 직접 Spring bean 으로 등록해서 사용할 수 없다. 즉, 다음과 같은 방법으로는 사용할 수 없다.
```xml
<bean id="m" class="springbook.leaningtest.spring.factorybean.Message">
```
사실 Spring 에서는 private constructor 를 가진 클래스도 bean 으로 등록하면 reflection 을 이용해 object 를 만들어준다. 하지만 생성자를 private 으로 만들었다는 것은 static method 를 통해 object 가 만들어져야하는 이유가 있을 것이다. 

#### MeessageFactoryBean.java
Message class 의 object를 생성해주는 factory bean class 는 다음과 같다.
```java
package springbook.learningtest.spring.factorybean;

import org.springframework.beans.factory.FactoryBean;

public class MessageFactoryBean implements FactoryBean<Message> {
	String text;
	
	// Object 를 생성할 때 필요한 정보를 factory bean 의 property 로 설정해서 대신 DI 받을 수 있게 한다. 
	// Injected 된 정보는 object 생성 중에 사용된다.
	public void setText(String text) {
		this.text = text;
	}

	// 실제 bean 으로 사용될 object 를 직접 생성한다. 
	// 코드를 이용하기 때문에 복잡한 방식의 object 생성과 초기화 작업이 가능하다.
	public Message getObject() throws Exception {
		return Message.newMessage(this.text);
	}

	public Class<? extends Message> getObjectType() {
		return Message.class;
	}

	// getObject() method 가 돌려주는 object 가 singleton 인지를 알려준다.
	public boolean isSingleton() {
		return true;
	}
}
```

Factory bean 의 설정은 일반 bean 과 다르지 않다. id 와 class attribute 를 사용해 bean 을 생성한다는 면에서 차이가 없다.
```xml
...
<bean id="message" class="springbook.learningtest.spring.factorybean.MessageFactoryBean">
	<property name="text" value="Factory Bean" />
</bean>
...
```
다른 bean 설정과 다른 점은 message bean object 의 type 이 class attribute 에 정의된 MessageFactoryBean 이 아니라 Message Type 이라는 것이다. Message bean 의 type 은 MessageFactoryBean 의 getObjectType() method 가 돌려주는 타입으로 결정된다. 또, getObject() method 가 생성해주는 object 가 message bean 의 object 가 된다.

#### FactoryBeanTest.java
아래는 위의 MessageFactoryBean 이 동작하는지 확인하는 테스트이다.
```java
package springbook.learningtest.spring.factorybean;

import static org.hamcrest.CoreMatchers.is;
import static org.junit.Assert.assertThat;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration // 설정파일 이름을 지정하지 않으면 클래스이름+"-context.xml" 이 기본으로 사용된다.
public class FactoryBeanTest {
	@Autowired
	ApplicationContext context;
	
	@Test
	public void getMessageFromFactoryBean() {
		Object message = context.getBean("message");
		// Type 확인
		assertThat(message, is(Message.class));
		// 설정과 기능 확인
		assertThat(((Message)message).getText(), is("Factory Bean"));
	}
	
	@Test
	public void getFactoryBean() throws Exception {
		// & 를 bean 이름 앞에 붙여주면 factory bean 자체를 돌려준다.
		Object factory = context.getBean("&message");
		assertThat(factory, is(MessageFactoryBean.class));
	}
}
```
