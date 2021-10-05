---
title: 그래들 태스크 정리
date: 2021-10-03 12:00:00 +0900
categories: [DevOps,gradle]
tags: [kotlin,jvm,gradle,kotlindsl]
---

## Gradle Task정리

이전까지 Gradle(wrapper)를 쓰면서, 한정적인 명령어를 감을 의존해 사용했었던 것 같다. 이번 기회에 gradle task의 주요 기능 및 build과정시 진행되는 것들을 살펴보려 한다.

### build vs bootJar
둘 다 많이 쓰고, 봐왔던 태스크에 해당한다. 이둘의 정확한 차이점을 이야기 해보는 것이 목표이다.

<br/>

1. ./gradlew build --dry-run
>(Asciidoctor 이라는 명령어가 있는데, 이는 프로젝트에서 사용하기 위해 따로 정의한 task다. test에 의존)

<img src="/assets/img/gradle/1.png">

눈에 띄는 것은 bootJar 이라는 과정이 포함되어 있는 것이다. 예상가능 하지만, ./gradlew bootJar을 하면 bootJar이 마지막 태스크로 위치하게 된다. java 플러그인이 포함된 프로젝트인 경우, 기존 assemble task는 이 bootJar에 의존성을 갖게되어 assemble에 의존하고 있는 build 과정에 bootJar작업이 들어가게 되는 것이다. 
> build는 플러그인이 없더라도 자동으로 내장된(Base Plugin) Lifecycle task에 해당한다.

<br>

빌드시엔 inspectClassesForKotlinIC ~ build 라는 과정이 추가되어 들어간다. 이것들이 어떠한 작업을 하는지 알면, 둘의 차이를 명확히 알 수 있다. 이전에 공통으로 수행하는 task들을 먼저 알아보자

#### 공통으로 들어가는 task들
앞선 이미지로 볼 수 있듯이, 컴파일과정(테스트 포함) + processResources + test + bootJar 이 들어간다. 
> 이외의 task들은 의존성을 부여하거나 받는 aggregating task에 해당한다. 이 task들에는 다른 플러그인이 와서 새로운 의존 task들을 추가할 수 있다.

- processResources: Copy작업에 해당한다. 생성된 리소스들을 복사해, resources directory에 넣어준다.
> processTestResources도 존재한다.

- 컴파일 과정이 들어있으므로 JDK가 필요하다.
- bootJar만 실행하였을 때에 해당한다. Java plugin을 이용하였을때, build와 비교해 좀 더 compact하게 프로젝트 구성을 할 수 있게 도와준다.
- 실행이 가능한 jar 파일을 생성해 주는 것으로 마무리 된다.

#### build 시에만 들어가는 task들
핵심적인 task들로 jar, assemble, check가 들어있다. 이 task들 역시 BasePlugin으로 깔려있다.

- jar: aggregate task인 classes에 의존한다. jar파일을 빌드해주는 라이프사이클 태스크에 해당한다. bootJar은 이 jar태스크를 확장하여 만든 것이다.
> _스프링 부트 2.5.0 이후에 jar파일을 생성할 경우 -plain이 붙은 jar파일이 추가로 생성이 된다._*

- assemble: jar task에 의존한다. aggregating task에 해당한다.
> java plugin이 추가될시, bootJar에도 의존한다.

- check: test에 의존하고 있다. verification을 진행하는 task들의 aggregating task에 해당한다. 여기에 다른 test를 의존성을 갖게해주면 check라는 과정으로 한번에 verification을 진행해 버릴 수 있다. 

#### 정리
- 즉, bootJar의 경우 java에 특화된 build과정을 추가해 준것에 해당한다. build시에 추가되는 aggregating task는 이외의 다른 task(verification / resource)들이 추가될 때 의존성을 걸어주면 통합으로 이를 진행시켜준다.
- 부트 버전 2.5.0 이후에 추가된 plain.jar 파일은 뭘까? 이 파일은 jar을 실행했을 때만 생성이 된다.
> 공식 문서를 보니, 이파일은 runnable하지 않은 zip파일에 해당한다. vi 명령어로 파고 들어가보면 다음과 같이 자바 파일과 함께, 코드를 확인할 수 있다. 

`vi /build/lib/multilang-0.0.1-plain.jar`

<img src="/assets/img/gradle/2.png">

각각의 파일에 엔터를 치고 들어가보면

<img src="/assets/img/gradle/3.png">

다음과 같이 코드를 확인할 수 있었다.