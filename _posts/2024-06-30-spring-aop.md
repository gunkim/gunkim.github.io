---
layout: post
title: 결국 Spring AOP는 프록시 패턴의 확장이다
tags: ["Spring", "Design Pattern"]
---

## 개요

Spring Framework의 관점 지향 프로그래밍(AOP, Aspect Oriented Programing)은 이름 때문에 복잡한 개념으로 오해될 수 있습니다. 그러나 실제로는 복잡한 프록시 패턴을 단순화하여, 프레임워크 레벨에서 구현함으로써 유지보수가 쉬운 코드를 작성할 수 있도록 도와줍니다. 

이 글에서는 AOP에서 강조하는 **관점**이 무엇인지, 이것이 **프록시 패턴**을 통해 어떻게 구현되는지, 그리고 AOP가 이를 어떻게 지원하는지를 알아봅니다.

![](/public/img/2024-06-30-1.png)

## 프록시 패턴 이해하기

[프록시 패턴](https://refactoring.guru/ko/design-patterns/proxy)은 기존 객체의 동작을 변경하지 않고, 추가적인 작업을 수행할 수 있게 해주는 디자인 패턴입니다. 이를 통해 클라이언트(:호출하는 객체)는 프록시 객체를 실제 객체처럼 사용할 수 있습니다.

프록시는 필요한 경우 로깅, 지연 로딩, 보안 검사 등 기존 로직에서 관심사를 벗어난 로직(:**횡단 관심사**)을 수행할 수 있습니다. 기본적으로 실제 객체와 동일한 인터페이스를 제공하여 클라이언트가 눈치채지 못하게 합니다.

다음 예제는 `UserService` 인터페이스에 대한 프록시 구현을 통해 로깅 기능을 추가하는 방법을 보여줍니다.

```java
public interface UserService {
    void loadUser(String userId);
}

//우리가 구현하는 애플리케이션 코드
public class UserServiceImpl implements UserService {
    @Override
    public void loadUser(String userId) {
        // 실제 사용자 정보를 데이터베이스에서 로드하는 로직
        System.out.println("Loading user from database for userId: " + userId);
    }
}


// 프록시 객체, Spring AOP에서 생성합니다.
public class UserServiceProxy implements UserService {
    private static final Logger logger = Logger.getLogger(UserServiceProxy.class.getName());
    
    // AOP가 적용된 실제 구현 클래스입니다.
    private UserServiceImpl userService;

    public UserServiceProxy(UserServiceImpl userService) {
        this.userService = userService;
    }

    @Override
    public void loadUser(String userId) {
        logger.info("Before invoking loadUser");
        userService.loadUser(userId);
        logger.info("After invoking loadUser");
    }
}
```

```java
public class UserController {
    //UserService 인터페이스를 의존하기 때문에 프록시의 존재 여부를 알지 못합니다.
    private UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    public void displayUser(String userId) {
        userService.loadUser(userId);
    }
}

public class Client {
    public static void main(String[] args) {
        // 실제 구현체를 생성합니다.
        UserService userServiceImpl = new UserServiceImpl();

        // 프록시를 통해 UserServiceImpl을 사용합니다.
        UserService userServiceProxy = new UserServiceProxy(userServiceImpl);

        // UserController는 UserService 인터페이스에 의존합니다.
        UserController userController = new UserController(userServiceProxy);

        // 프록시의 존재를 알지 못하고, 메서드를 호출합니다.
        userController.displayUser("12345");
    }
}
```

이 예제에서 프록시를 통해 실제 구현체의 변경 없이도 필요한 기능을 확장할 수 있으며, 이로 인해 소프트웨어의 유지 관리가 용이해집니다.

## AOP (Aspect-Oriented Programming)

프록시 패턴을 AOP로 구현하면 다음과 같이 간단해집니다.

```java
@Aspect
@Component
public class LoggingAspect {
    private static final Logger logger = Logger.getLogger(LoggingAspect.class.getName());

    @Before("execution(* com.example.UserService.loadUser(..))")
    public void logBeforeLoadUser(JoinPoint joinPoint) {
        logger.info("Before invoking loadUser");
    }

    @After("execution(* com.example.UserService.loadUser(..))")
    public void logAfterLoadUser(JoinPoint joinPoint) {
        logger.info("After invoking loadUser");
    }
}
```
니
- `@Before("execution(* com.example.UserService.loadUser(..))")`
- `@After("execution(* com.example.UserService.loadUser(..))")`

`@Before`와 `@After` 어노테이션은 메서드 호출 전후에 실행할 어드바이스를 지정하며, 내부의 문자열은 포인트컷을 정의합니다. `@Aspect`는 이 클래스가 어드바이스와 포인트컷을 포함하는 애스펙트임을 나타냅니다. 

어드바이스나 포인트컷에 대한 더 자세한 내용은 [여기](https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/advice.html)를 확인하면 됩니다.

## Spring에서의 프록시 패턴 적용

Spring은 AOP의 접근 방식을 사용하여, 인터페이스의 유무와 관계없이 프록시 객체를 생성할 수 있는 유연성을 제공합니다. 

![](/public/img/2024-06-30-2.png)

### JDK Dynamic Proxy

인터페이스를 구현한 클래스의 경우, JDK Dynamic Proxy는 Java의 [Reflection API](https://docs.oracle.com/javase/tutorial/reflect)를 사용하여 인터페이스 기반의 프록시 객체를 동적으로 생성합니다. 런타임에 AOP 대상 객체가 구현한 인터페이스를 상속하고 구현합니다.

이 방식은 앞선 예제 코드의 Proxy 구현을 그대로 런타임에 대신 해준다고 이해하면 됩니다.

### CGLIB Proxy

인터페이스가 없는 클래스의 경우, Spring은 [CGLIB](https://github.com/cglib/cglib) 라이브러리를 사용합니다. CGLIB은 자바 바이트 코드를 조작하여 런타임에 원본 클래스를 상속받는 서브 클래스를 생성함으로써 프록시를 구현합니다. 이 방식은 인터페이스 없이도 프록시 생성이 가능하게 합니다.

## 결론

Spring AOP(관점 지향 프로그래밍)에서 말하는 **관점**은 핵심 로직과 관련 없는 **횡단 관심사**를 의미합니다. 이러한 횡단 관심사들은 로깅, 보안 검사, 트랜잭션 처리 등과 같은 공통 기능을 포함하며, 이를 효과적으로 **모듈화**하고 **재사용**하기 쉽게 만드는 것이 AOP의 핵심 목표입니다. 

프록시 패턴은 이러한 횡단 관심사를 관리하기 위해 사용되며, 프록시 패턴을 이해하면 Spring에서의 AOP 구현도 더욱 명확해집니다.

또한, Spring AOP는 프레임워크 코드가 애플리케이션 코드에 침투하는 것을 방지합니다. 인터페이스 생성을 강제하지 않아 코드의 복잡성을 줄이면서도, AOP의 마법과 같은 동작은 때때로 코드의 직관성을 떨어뜨릴 수 있기 때문에 남용에 대한 주의가 필요합니다.