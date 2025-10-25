---
layout: post
title: Windows ARM64에서 Gradle Tooling API 환경변수 전달 문제 해결기
category: gradle
---

Windows ARM 환경에서 개발하던 중 예상치 못한 문제를 마주했다. 단순한 디버깅 문제였지만, Gradle 내부 구조와 ARM 지원 메커니즘을 파헤치는 여정이 되었다.

## 문제의 발견

Windows ARM 서브 머신의 IntelliJ에서 디버깅을 걸었는데 브레이크포인트가 전혀 동작하지 않았다. IDE 설정 문제라고 생각해서 캐시를 지우고, 프로젝트를 다시 임포트하고, IntelliJ를 재시작해봤지만 마찬가지였다.

문제를 재현하면서 흥미로운 패턴을 발견했다. 일반 Java 프로젝트에서는 디버깅이 안 됐지만, Spring Boot 프로젝트에서는 정상 동작했다. 더 이상한 점은 같은 JetBrains의 IDE인 Fleet에서는 모든 프로젝트에서 디버깅이 잘 됐다는 것이다.

이런 패턴을 보니 환경 변수나 시스템 레벨의 문제일 가능성이 높았다.

## 문제 추적

JetBrains YouTrack에 [IDEA-380072](https://youtrack.jetbrains.com/issue/IDEA-380072) 이슈를 등록했다. 재현 방법, 환경 정보, 시연 영상, 로그를 제출하고 Fleet에서는 동작한다는 점을 강조했다. 당연히 IntelliJ 버그라고 생각했다.

그런데 JetBrains 담당자의 답변은 의외였다. **IntelliJ 문제가 아니라 서드파티인 Gradle의 Tooling API 문제이니 Gradle 측에 문의해보라**는 것이었다. Fleet에서는 동작하기 때문에 여전히 IntelliJ 버그를 의심했고, 책임을 회피하는 것은 아닌가 하는 생각도 들었다.

**재현 환경:**
- Windows 11 ARM64
- IntelliJ IDEA 2025.2.2
- Gradle 8.14
- JDK 21 (Azul JDK)

## Gradle Tooling API의 이해

Gradle Tooling API는 서드파티 도구들이 Gradle과 상호작용할 수 있게 해주는 인터페이스다. IntelliJ는 이 API를 통해 프로젝트 구조, 의존성, 빌드 설정 등의 정보를 가져온다.

![Gradle Tooling API 동작 구조](/image/d2.png)

다이어그램에서 볼 수 있듯이, IntelliJ는 Tooling API를 통해 Gradle Daemon과 통신하고, Daemon은 프로젝트 정보를 수집해서 IntelliJ로 전달한다. 핵심은 Gradle Daemon이 환경 정보를 수집해서 Tooling API를 통해 IntelliJ로 전달한다는 점이다.

로그를 살펴보니 Gradle Daemon이 프로세스 환경 변수를 조회할 때 null을 반환하고 있었다. 환경 변수가 없으니 디버거가 필요로 하는 정보를 IntelliJ로 전달할 수 없었고, 디버깅이 동작하지 않는 것이었다.

## 근본 원인의 발견

[Gradle 프로젝트](https://github.com/gradle/gradle)를 클론해서 환경 변수를 다루는 코드를 추적했다. `NativeServices.java`의 `createProcessEnvironment` 메서드에 도달했다.

```java
@Provides
protected ProcessEnvironment createProcessEnvironment(OperatingSystem operatingSystem) {
    if (useNativeIntegrations) {
        try {
            // native-platform을 사용해서 시스템 환경 변수를 가져온다
            // 이 라이브러리는 JNI를 통해 각 플랫폼의 네이티브 API를 호출한다
            net.rubygrapefruit.platform.Process process = nativeIntegration.get(Process.class);
            return new NativePlatformBackedProcessEnvironment(process);
        } catch (NativeIntegrationUnavailableException ex) {
            LOGGER.debug("Native-platform process integration is not available. Continuing with fallback.");
        }
    }
    return new UnsupportedEnvironment();
}
```

Gradle은 환경 변수를 가져오기 위해 `native-platform` 라이브러리를 사용한다. 이 라이브러리는 [JNI(Java Native Interface)](https://en.wikipedia.org/wiki/Java_Native_Interface)를 통해 각 플랫폼의 네이티브 코드를 호출해서 시스템 정보를 가져온다.

문제는 이 라이브러리가 Windows x86, x64, Linux, macOS는 지원하지만 **Windows ARM64는 지원하지 않는다**는 것이었다. native-platform이 Windows ARM64를 지원하지 않으니 `NativeIntegrationUnavailableException`이 발생하고, catch 블록을 거쳐 `UnsupportedEnvironment`를 반환한다.

`UnsupportedEnvironment`는 환경 변수를 제공하지 않는 구현체다.
1. Gradle Daemon이 환경 변수를 가져오지 못한다
2. Tooling API를 통해 IntelliJ로 빈 환경 정보가 전달된다
3. 디버거가 필요로 하는 JVM 옵션과 경로 정보가 누락된다
4. 브레이크포인트가 동작하지 않는다

## 임시 해결책의 구현

문제를 찾았으니 해결 방법을 찾아야 했다. Java 리플렉션을 사용해서 Windows 환경 변수를 가져오는 fallback 구현체를 만들기로 했다.

Windows는 `System.getenv()`로 환경 변수를 가져올 수 있지만, Gradle이 필요로 하는 것은 단순 환경 변수 맵이 아니라 `ProcessEnvironment` 인터페이스의 구현체다. 이 인터페이스는 환경 변수를 설정하고, 수정하고, 새로운 프로세스를 시작할 때 전달하는 기능을 제공한다.

[WindowsReflectiveProcessEnvironment](https://github.com/gunkim/gradle/blob/2551b46b33ac12a7961f98c6686cbd3a876f6175/platforms/core-runtime/native/src/main/java/org/gradle/internal/nativeintegration/jna/WindowsReflectiveProcessEnvironment.java) 클래스를 구현했다. 이 클래스는 Java 리플렉션 API를 사용해서 Windows의 환경 변수를 읽어온다. native-platform을 완전히 대체할 수는 없지만, 환경 변수를 읽는 용도로는 충분했다.

```java
@Provides
protected ProcessEnvironment createProcessEnvironment(OperatingSystem operatingSystem) {
    if (useNativeIntegrations) {
        try {
            net.rubygrapefruit.platform.Process process = nativeIntegration.get(Process.class);
            return new NativePlatformBackedProcessEnvironment(process);
        } catch (NativeIntegrationUnavailableException ex) {
            LOGGER.debug("Native-platform process integration is not available. Continuing with fallback.");
        }
    }
    // native-platform의 지원을 받지 못하는 경우 리플렉션 fallback으로 환경변수 전달
    if (operatingSystem.isWindows()) {
        LOGGER.info("Using Windows reflection-based process environment (native-platform unavailable)");
        return new WindowsReflectiveProcessEnvironment();
    }
    return new UnsupportedEnvironment();
}
```

이 변경사항을 로컬에서 빌드하고 테스트하니 디버깅이 정상 동작했다. 환경 변수가 IntelliJ로 전달되면서 브레이크포인트도 제대로 걸렸다.

## Gradle 프로젝트에 기여

Gradle 이슈 트래커를 보니 [Windows ARM64 관련 이슈들](https://github.com/gradle/gradle/issues/21703)이 여전히 Open 상태였다. 많은 사람들이 같은 문제를 겪고 있었다.

리플렉션 기반 fallback을 포함한 Pull Request를 제출했다. 하지만 PR을 올리고 나서 다시 생각해보니 이것은 근본적인 해결책이 아니었다. 리플렉션은 임시방편이고, 제대로 된 해결책은 native-platform이 Windows ARM64를 지원하는 것이었다.

메인테이너에게 "리플렉션 fallback은 임시 해결책이고, native-platform의 Windows ARM64 지원이 근본적인 해결책이다"라고 의견을 전달했다. 그리고 놀라운 답변을 받았다. **"Gradle 9.2.0-rc-2 버전에서는 이미 해결되었다"**는 것이었다. 딱 PR 올리기 일주일 전에 내 문제를 해결한 Gradle 9.2.0-rc-2 버전이 출시되는 기묘한 타이밍이었다.

## 근본적인 해결 확인

Gradle 9.2.0-rc-2 버전을 다운로드해서 테스트했다. 모든 것이 정상 동작했다. native-platform이 Windows ARM64를 지원하면서 더 이상 환경 변수 전달에 실패하지 않았다. 디버깅도 완벽하게 동작했다.

테스트 결과를 Gradle 메인테이너에게 전달했다. Windows ARM 머신에서 직접 테스트한 결과 문제가 해결되었다고 보고하니, 빠르게 피드백을 줘서 고맙다는 답변과 함께 관련 이슈들이 모두 Close 처리됐다. Gradle 팀에는 Windows ARM 머신이 없어서 제대로 테스트하지 못했고, 그래서 관련 이슈들이 오래 열려있었던 것이다. (그렇게 빠르게 베타테스터가 되었다.)

IntelliJ 담당자에게도 문제가 해결되었음을 알렸다. 결국 IntelliJ의 문제가 아니라 Gradle의 문제였고, 새 버전에서 해결되었으니 정식 릴리즈를 기다리면 된다고 전달했다.

## Gradle 9.2.0-rc2에서의 해결 방법

해결 방법을 찾아보니 Gradle 9.2.0-rc2가 아니라 Gradle 9.2.0-rc1에서 이미 수정된 것으로 보인다.

[Gradle 9.2.0-rc-1 Release Notes](https://docs.gradle.org/9.2.0-rc-1/release-notes.html)를 보면 다음과 같이 명시되어 있다.

> "Gradle now supports running builds on Windows ARM (ARM64) devices. This makes it possible to run Gradle on Windows virtual machines hosted on ARM-based systems."

관련 이슈를 찾아보니 [Support ARM64 Windows](https://github.com/gradle/gradle/issues/21703)가 해당 이슈다. native-platform 라이브러리를 통해 Windows ARM64용 빌드를 제공했기 때문에 해결됐다.

## Fleet과 IntelliJ 특정 프로젝트에서는 왜 디버깅이 됐을까?

IntelliJ 빌드 툴이 Gradle이 아닌 IntelliJ로 설정된 경우, Gradle 데몬을 사용하지 않고 직접 Gradle 프로젝트를 빌드한다. 이렇게 되면 Gradle을 우회하기 때문에 관련 문제가 발생하지 않는다. (즉 IntelliJ는 Windows ARM64 지원이 이미 되고 있다.)

## 마치며

단순한 디버깅 문제로 시작했지만, 결국 Gradle의 내부 구조와 native-platform의 플랫폼 지원 메커니즘까지 파헤치게 됐다.

비록 내가 만든 PR은 머지되지 않았지만, Gradle 팀에 Windows ARM 테스트 결과를 전달하고 관련 이슈를 닫을 수 있었다. 오픈소스 기여는 코드 머지만이 전부가 아니라는 걸 다시 한번 느꼈다.

Gradle 9.2.0 정식 버전이 릴리즈되면 이 문제는 완전히 해결된다. 그 전까지는 IntelliJ 빌드 툴을 IntelliJ로 사용하거나 Gradle-9.2.0-rc2 버전을 사용하면 된다.

마지막으로 이슈만 볼 게 아니라 관련 PR이나 정식 버전의 릴리즈만이 아닌, 서브 버전들의 릴리즈 노트까지 세심히 살폈다면 삽질을 덜 하지 않았을까 싶다. (그러면 내부까지 안 파봤을 테니 손해일지도)