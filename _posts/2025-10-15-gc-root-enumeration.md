---
layout: post
title: GC 기본 알고리즘 - 루트 노드 열거(Root Enumeration)
category: java
---

이 포스트는 GC 기본 알고리즘에 대해 다루며, G1 GC, ZGC 등 세부 구현을 학습하기 전 기초 자료로써 활용할 수 있다.

## 알고 있다 가정하는 것

- Classfile이 무엇인지
- JIT 컴파일러가 무엇을 하는 녀석인지

## GC의 3단계 과정

**GC(Garbage Collection)**는 Root Set을 기준으로 참조 가능한 객체와 참조 불가능한 객체를 구분하여, 참조 불가능한 객체를 Garbage로 판단하고 메모리에서 해제함으로써 메모리를 효율적으로 관리하는 과정이다.

![](https://gunkim.gitbook.io/wiki/~gitbook/image?url=https%3A%2F%2F1622038435-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-x-prod.appspot.com%2Fo%2Fspaces%252FdJraD68HJlMF6Wf3h0TB%252Fuploads%252Fgit-blob-885a6011c9c6c179e3cf8dedcfd5cd4a4b1cb80a%252F03-4.png%3Falt%3Dmedia&width=300&dpr=4&quality=100&sign=d53feed0&sv=2)

Root Set에서 화살표가 도달하지 못하는 객체들은 GC 대상이 된다.

이 때 GC는 3가지 과정으로 이해할 수 있다.

1. **루트 열거(Root Enumeration)** - 도달 가능한 객체를 찾기 위한 탐색 시작점인 Root Set을 구성하는 단계이다.

2. **도달 가능성 분석(Reachability Analysis)** - Root Set에서 시작하여 참조를 따라가며 도달 가능한 객체와 도달 불가능한 객체(Garbage)를 식별하는 작업이다.

3. **메모리 회수(Memory Reclamation)** - 식별된 Garbage 객체들을 메모리에서 제거하고 사용 가능한 메모리 공간을 확보하는 작업이다.

이 포스트에서는 첫번째 과정인 루트 열거 과정에 대해 다룬다. 메모리 회수 단계는 GC 구현체마다 다르기 때문에 별도의 포스트로 다루겠다.

## 루트 열거(Root Enumeration)

예제 코드로 살펴보자.

```java
public class Example {
    public static Example example = new Example(); // Root - 정적 변수
     
    private Example field = null;
      
    public void method() {
        Example localRef = new Example(); // Root - 지역 변수
        Example temp = new Example();     // Root - 지역 변수
        localRef.field = temp;
    
        System.gc(); // GC 요청 - 실제 GC는 모든 스레드가 Safe Point에 도달한 후 시작
    }
}
```

### Root Set에 포함되는 주요 요소들

- **정적 변수**: 클래스의 static 필드 - 클래스 로딩부터 프로그램 종료까지 생존하는 참조
- **지역 변수**: 메서드 스택의 로컬 변수 - 메서드 실행 중 스택에 존재하는 참조
- **JNI 글로벌 참조**: 네이티브 코드에서 명시적으로 생성/삭제 관리되는 참조

여기서 Root Set은 정적 변수 `example`과 지역 변수 `localRef`, `temp`다. GC는 이들을 시작점(Root)으로 도달 가능성 분석을 시작한다.

그렇다면 이 시작점을 기반으로 탐색하면 될텐데, 루트 열거라는 단계가 존재하는 이유가 뭘까?

### 바이트코드로 들여다보기

위 클래스를 컴파일하면 다음 바이트코드가 생성된다.

```
public void method();
  descriptor: ()V
  flags: (0x0001) ACC_PUBLIC
  Code:
    stack=2, locals=3, args_size=1
       0: new           #8                  // class Example
       3: dup
       4: invokespecial #13                 // Method "<init>":()V
       7: astore_1                          // localRef를 로컬 변수 슬롯 1에 저장
       8: new           #8                  // class Example
      11: dup
      12: invokespecial #13                 // Method "<init>":()V
      15: astore_2                          // temp를 로컬 변수 슬롯 2에 저장
      16: aload_1
      17: aload_2
      18: putfield      #7                  // Field field:LExample;
      21: invokestatic  #14                 // Method java/lang/System.gc:()V
      24: return
```

`astore_1`, `astore_2`는 생성된 객체 참조를 로컬 변수 테이블의 슬롯 1, 2에 저장한다. `locals=3`에서 보듯이 슬롯 0은 `this`, 슬롯 1은 `localRef`, 슬롯 2는 `temp`다.

바이트코드를 보면 결국 컴파일 타임에 슬롯에 번호가 붙어 어디에 저장되는지가 확정되는 걸 알 수 있다. 그렇다면 JVM은 이 정보를 토대로 어떻게 Heap을 찾아갈 수 있을까?

**이상하다.** 어떤 객체가 Heap 내 어느 위치에 저장될지는 런타임에 결정이 된다. 하지만 컴파일 타임에 정해진다는건 말이 안되지 않나?

### OopMap의 등장

컴파일 타임에 결정되는 건 **변수의 저장 위치 정보**다. "localRef는 슬롯 1에, temp는 슬롯 2에 저장된다"

**OopMap**은 특정 실행 지점에서 "어떤 위치에 객체 참조가 저장되어 있는지" 기록한 메타데이터 테이블이다.

하지만 문제가 있다.

### JIT 최적화의 함정

JIT 컴파일러는 이 바이트코드를 읽어 런타임에 최적화된 네이티브 코드로 변환한다. 문제는 여기서 시작된다.

```java
public void optimizedMethod() {
    Example localRef = new Example();  // 바이트코드: astore_1, 아직 oopMap이 없어 이 변수만으론 Heap 객체를 찾을 수 없다.
    Example temp = new Example();      // 바이트코드: astore_2, 아직 oopMap이 없어 이 변수만으론 Heap 객체를 찾을 수 없다.
    
    localRef.field = temp;
    
    // 여기서 GC가 발생한다면?
    System.gc(); // <- Safe Point: 이 때 OopMap 조회
                 // OopMap: "RAX, RBX에 객체 참조 있음"
}
```

컴파일 시점에 확정된 바이트코드에서는 변수 위치가 명확하지만, JIT 컴파일러가 작동하는 건 런타임이기 때문에 문제가 생긴다. 변수를 CPU 레지스터로 옮기거나 메모리 위치를 바꾸기 때문이다.

다행히 JIT 컴파일러는 최적화를 수행하면서 OopMap도 함께 업데이트한다. 변수가 레지스터로 이동하면 "레지스터 RAX에 객체 참조 있음"으로 OopMap 정보를 갱신한다.

```
// OopMap 업데이트 과정:
// 1. 바이트코드 컴파일 시: "슬롯 1, 2에 객체 참조"  
// 2. JIT 최적화 후: "레지스터 RAX, RBX에 객체 참조"
```

하지만 이렇게 업데이트된 OopMap 정보를 신뢰할 수 있는 시점이 필요하다. JIT 최적화는 지속적으로 진행되고 변수 위치가 동적으로 바뀔 수 있기 때문이다.

그래서 이걸 안전하고 정확하게 파악할 수 있는 지점이 **Safe Point**이다.

## Safe Point에서 일어나는 일

Safe Point는 JVM이 스레드의 실행 상태를 정확히 파악할 수 있는 특별한 지점들이다.

### Safe Point가 선택되는 주요 지점들

- **메서드 호출 지점**: 호출 스택이 명확한 상태
- **루프 백엣지**: 루프가 다시 시작되는 지점
- **예외 발생 지점**: 실행 흐름이 명확한 상태
- **메서드 리턴 지점**: 메서드 종료 시점
- **긴 시간 실행되는 루프**: 특정 간격으로 Safe Point 삽입
- **JNI 호출 지점**: 네이티브 코드 전환 지점

이 지점들을 선택하는 이유는 실행 상태가 예측 가능하고 OopMap 정보가 정확히 구성될 수 있기 때문이다.

### Safe Point에서 실제로 뭐가 일어날까?

```java
// 이런 코드가 있다고 하자
public void safePointExample() {
    Object obj1 = new Object();  // 힙에 객체 생성
    Object obj2 = new Object();  // 힙에 객체 생성
    
    // JIT 최적화 후:
    // obj1 -> 레지스터 RAX (0x7f8a1c004000 주소)
    // obj2 -> 레지스터 RBX (0x7f8a1c004020 주소)
    
    someMethod(); // <- 메서드 호출 지점이 Safe Point 후보
}
```

### Safe Point에서 JVM이 하는 일

1. **모든 스레드 일시정지**: 잠깐, 모두 멈춰!

2. **OopMap 조회**: 현재 지점에서 RAX, RBX에 객체 참조 있음

3. **실제 주소 수집**: RAX에서 0x7f8a1c004000 추출, RBX에서 0x7f8a1c004020 추출

4. **Root Set 완성**: 수집한 주소들이 GC의 시작점

5. **다음 단계로**: 완성된 Root Set을 기반으로 도달 가능성 분석 시작

JVM은 모든 명령어 지점에 대해 OopMap을 생성하지 않는다. 모든 지점에 생성하면 막대한 메타데이터로 인해 성능에 악영향을 미치기 때문이다. 대신 Safe Point에서만 선별적으로 OopMap을 생성하여 효율성과 정확성을 동시에 확보한다.

---
## 참고

- JVM 밑바닥 파헤치기 도서