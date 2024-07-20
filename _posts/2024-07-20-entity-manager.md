---
layout: post
title: 스프링에서 EntityManager는 어떻게 관리될까?
tags: ["Spring", "JPA"]
---

## 개요

스프링 프레임워크에서 `spring-data-jpa` 의존성을 사용하여 JPA/Hibernate를 활용할 때, 직접적으로 `EntityManager`를 조작하는 경우는 드뭅니다. 대부분의 경우 `JpaRepository`를 통해 간접적으로 사용하며, 필요 시에는 `@PersistenceContext` 어노테이션을 통해 `EntityManager` 객체를 주입받아 사용합니다.

이 글에서는 스프링에서 `EntityManager`가 어떻게 생성되고 관리되는지, 그리고 트랜잭션과의 관계를 중점으로 설명하겠습니다.

---

## @PersistenceContext 살펴보기

기본적으로 `EntityManager`는 `EntityManagerFactory`를 통해 생성됩니다. 이는 JPA를 사용하는 개발자라면 대부분 알고 있는 사실입니다.

그렇다면, `@PersistenceContext` 어노테이션을 통해 주입받는 `EntityManager`는 어떻게 동작할까요?

```kotlin
@Repository
class PersonRepository(
    @PersistenceContext    
    private val entityManager: EntityManager
) {    
    fun find(id: Long) {        
        entityManager.find(Person::class.java, id)
    }
}
```

위의 코드에서 `PersonRepository` 객체는 싱글톤 객체로 스프링이 기동되는 시점에 초기화되어 스프링 컨텍스트에 등록됩니다. 그러면 `EntityManager`는 초기에 한 번 주입되며, 매 트랜잭션마다 생성되는 것이 아니라 최초 생성된 `EntityManager`가 재사용되는 것일까요?

## EntityManager의 정체는 프록시다!

스프링에서 `@PersistenceContext`를 통해 주입받는 `EntityManager`는 실제로는 프록시 객체입니다. 이 프록시 객체는 스레드 로컬(Thread Local) 변수를 사용하여 각 트랜잭션마다 다른 `EntityManager` 인스턴스를 제공합니다.

따라서, 실제로는 매 트랜잭션마다 새로운 `EntityManager`가 생성되고, 트랜잭션이 끝나면 해당 `EntityManager`는 닫힙니다.

다음은 `EntityManager` 프록시 객체가 어떻게 생성되는지를 보여주는 코드입니다

```java
public static EntityManager createSharedEntityManager(EntityManagerFactory emf, @Nullable Map<?, ?> properties,
    boolean synchronizedWithTransaction, Class<?>... entityManagerInterfaces) {
    ClassLoader cl = null;
    if (emf instanceof EntityManagerFactoryInfo emfInfo) {
        cl = emfInfo.getBeanClassLoader();
    }
    Class<?>[] ifcs = new Class<?>[entityManagerInterfaces.length + 1];
    System.arraycopy(entityManagerInterfaces, 0, ifcs, 0, entityManagerInterfaces.length);
    ifcs[entityManagerInterfaces.length] = EntityManagerProxy.class;
    return (EntityManager) Proxy.newProxyInstance(
            (cl != null ? cl : SharedEntityManagerCreator.class.getClassLoader()),
            ifcs, new SharedEntityManagerInvocationHandler(emf, properties, synchronizedWithTransaction));
}
```

이 코드는 프록시 `EntityManager`를 생성합니다. 프록시 `EntityManager`는 실제 `EntityManager`를 호출할 때마다 `InvocationHandler`의 `invoke` 메서드를 통해 적절한 `EntityManager`를 찾아 사용합니다.

## 실제 EntityManager는 어떻게 생성될까?

`SharedEntityManagerInvocationHandler`의 `invoke` 코드를 살펴보면 다음과 같은 코드가 있습니다:

```java
EntityManager target = EntityManagerFactoryUtils.doGetTransactionalEntityManager(this.targetFactory, this.properties, this.synchronizedWithTransaction);
```

이 코드는 `EntityManager`를 얻는 과정을 다음과 같이 세 가지로 나눕니다.

1. **현재 트랜잭션과 연관된 (이미 생성된) EntityManager를 얻기**  
    ```java
    EntityManagerHolder emHolder = (EntityManagerHolder) TransactionSynchronizationManager.getResource(emf);
    ```
    이 코드 내에서 실제로 리소스들이 스레드 로컬하게 관리되며, 동일한 스레드라도 트랜잭션이 다르다면 공유되지 않습니다.
    
2. **현재 스레드에 활성화된 EntityManager가 없을 때**  
    트랜잭션 스코프 밖에서 호출되었음을 의미하며, 엔티티 매니저를 생성할 수 없기 때문에 null을 반환합니다.
    ```java
    else if (!TransactionSynchronizationManager.isSynchronizationActive()) {
        return null;
    }
    ```
    
3. **현재 트랜잭션에 사용할 새로운 EntityManager를 생성**  
    ```java
    EntityManager em = null;
    if (!synchronizedWithTransaction) {
        try {
            em = emf.createEntityManager(SynchronizationType.UNSYNCHRONIZED, properties);
        } catch (AbstractMethodError err) {
        }
    }
    if (em == null) {
        em = (!CollectionUtils.isEmpty(properties) ? emf.createEntityManager(properties) : emf.createEntityManager());
    }
    ```

이 과정 덕분에 스프링에서는 `EntityManager`가 안전하게 트랜잭션과 함께 사용되고 관리될 수 있습니다. 스프링은 이러한 복잡한 작업들을 프록시와 스레드 로컬 변수를 사용하여 투명하게 처리해 줍니다. ~~대신 코드를 파악하기 어렵고 모르고 쓰게 된다~~

이를 통해 개발자는 복잡한 내부 구현을 신경 쓰지 않고 비즈니스 로직에 집중할 수 있습니다.

## 트랜잭션과 EntityManager의 관계

스프링에서는 `@Transactional` 어노테이션을 통해 트랜잭션을 관리합니다. 이 어노테이션이 적용된 메서드는 트랜잭션이 시작될 때 `EntityManager`가 생성되고, 메서드가 종료될 때 `EntityManager`가 닫힙니다.

```kotlin
@Service
class PersonService(
    private val personRepository: PersonRepository
) {
    @Transactional
    fun updatePerson(id: Long, name: String) {
        val person = personRepository.findById(id).orElseThrow()
        person.name = name
        personRepository.save(person)
    }
}
```

위의 코드에서 `updatePerson` 메서드가 호출되면 트랜잭션이 시작되고, 트랜잭션 범위 내에서 `EntityManager`가 생성되어 사용됩니다. 메서드가 종료되면 트랜잭션이 끝나고 `EntityManager`가 닫힙니다.

## 결론

스프링에서 `@PersistenceContext`를 통해 주입받는 `EntityManager`는 프록시 객체로, 각 트랜잭션마다 다른 `EntityManager` 인스턴스를 제공합니다.

이 프록시 객체는 스레드 로컬 변수를 사용하여 트랜잭션 범위 내에서 안전하게 `EntityManager`를 관리합니다.

또한, `JpaRepository`를 통해 사용할 경우에도 동일한 동작이 이루어집니다.