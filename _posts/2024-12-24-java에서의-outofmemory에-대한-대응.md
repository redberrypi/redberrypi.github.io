---
layout: post
title: Java에서의 OutOfMemory에 대한 대응
date: 2024-12-24 23:24 +0900
categories: [Java]
tags: [Java, Jvm]
---

## OutOfMemoryError

OutOfMemoryError(이하 oome)에 대한 악명은 개발자라면 익히 들어서 익숙할 것이다.  
하지만 익숙한만큼 이 정도는 별 거 아니지 라고 생각하게 되고, JVM 위에서 돌아가는 Java는 메모리 관리를 알아서 해주기에 실제로 oome와 마주칠 기회는 그리 많지 않았던 것 같다.  
그러나 언제나 예외는 있는 법이다.  
꼭 내가 만든 어플리케이션에서만 이러한 에러가 나는 것도 아니고, 평소에는 잘만 작동하는 프로그램이라도 예상 외의 부하가 걸린다면 얄짤없이 oome와 마주쳐야 할 수도 있다.  
이 포스트에서는 이러한 oome의 원인이나 해결 방법에 앞서 이러한 oome와 마주쳤을 때 당장 이를 어떻게 제어하고 임시 조치를 취할 수 있을지를 알아보고자 한다.  

## GC(Garbage Collector) 선정

JVM 위에서 돌아가는 GC를 바꿔주는 것은 근본적인 해결법은 아니지만 비교적 손쉽게, 코드를 건드리지 않고도 사태를 호전시켜줄 수 있다.  
Java Heap Space 오류의 경우, 실제 메모리 누출의 경우가 아니라 단순히 JVM의 성능이 GC의 속도를 따라가지 못해서 일어나는 경우가 있는데, 이러한 경우 GC를 바꿔주는 것이 효과적인 방법이 될 수 있다.  
GC에는 여러 종류가 있다.  

* Serial GC
* Parallel GC
* Concurrent Mark-Sweep (CMS) GC
* G1 (Garbage First) GC
* ZGC

Serial GC는 가장 기본적인, 단일 스레드를 위한 GC이다. 과거의 유산으로서 현재는 거의 쓰이지 않는다.  
Parallel GC는 Java 8버전의 기본 GC이다.  
CMS GC는 비효율적이며 메모리 파편화 문제가 있어 현재는 거의 쓰이지 않는다고 한다.  
G1 GC는 Java 9+버전의 기본 GC로서, 아마 대부분의 시스템에서 쓰이고 있지 않나 싶다.  
ZGC는 현재 최신의 GC이며, Java 15버전부터 release되었지만 그 이전 버전에서도 실험 기능으로 사용할 수 있는 경우가 있는 것 같다.  

Java 버전에 따라 사용할 수 있는 GC가 있고 없는 GC가 있다.  
상황에 따라 적절한 GC를 고를 수 있을 것이다.  

## JVM args 설정

사실 솔직히 말하자면, 대부분의 경우 GC를 바꾸는 것은 임시방편에 불과하다.  
OOME의 성질상 메모리는 계속 쌓여만 갈 뿐이고, 그것은 GC 성능이 아무리 좋든 상관이 없기 때문이다.  
위에서 언급했듯, GC 처리 속도가 어플리케이션의 요구를 따라가지 못하는 일부 상황에서나 효과가 있다고 할 수 있다.  
이 경우 JVM이 터지는 것은 피할 수 없으며, JVM의 argument 중에는 이에 대처할 수 있는 몇 가지의 옵션이 존재한다.  

1. -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath
2. -XX:OnOutOfMemoryError
3. -XX:+ExitOnOutOfMemoryError
4. -XX:+CrashOnOutOfMemoryError -XX:ErrorFile

### -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath

`-XX:+HeapDumpOnOutOfMemoryError` 옵션은 말 그대로 OOME가 발생한 시점에 어플리케이션의 힙 덤프를 남기는 옵션이다.  
다시 말해 해당 시점에서의 메모리의 스냅샷을 남기는 것이라고도 할 수 있다.  
이러한 스냅샷은 Eclipse Memory Analyzer (MAT)나 VisualVM 등의 분석툴로 분석할 수 있으며, 이를 통해 OOME의 핵심 원인을 찾아내는 것이 가능하다.  
`-XX:HeapDumpPath` 옵션을 통해 해당 힙 덥프의 path를 지정하는 것도 가능하다.  
예를 들면 다음과 같이 사용할 수 있다.  

``` Jvm
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/logs/heapdump.hprof
```

힙 덤프는 hprof의 확장자를 지니며, 이진 파일이므로 일반적인 텍스트 편집기로는 열 수 없다.  

### -XX:OnOutOfMemoryError

`-XX:OnOutOfMemoryError` 옵션은 메모리 오류가 발생했을시 어떠한 행동을 할지를 정한다.  
예를 들면 메모리 오류가 발생했을 경우 해당 시점을 log로 남기고 싶거나, 메일로 담당자에게 오류 사실을 전달하고 싶은 경우가 있을 수 있다.  
혹은 어플리케이션 자체를 재실행시키고 싶을 수도 있다.  
이럴 때 이 옵션을 사용하면 된다.  

``` Jvm
-XX:OnOutOfMemoryError=/scripts/log.sh
```

### -XX:+ExitOnOutOfMemoryError & -XX:+CrashOnOutOfMemoryError

`-XX:+ExitOnOutOfMemoryError`와 `-XX:+CrashOnOutOfMemoryError` 두 옵션 모두 OOME 발생 시점에 어플리케이션을 그 즉시 종료한다.  
만약 이러한 옵션이 지정되지 않는다면 어플리케이션은 기본적으로 메모리 오류가 난 채로 작동을 계속하는데, 메모리 오류가 난 부분과 상관이 없는 부분이라면 상관이 없겠지만 메모리 오류가 난 부분은 계속해서 오류가 난 채로 존재하게 된다.  
메모리 오류가 어플리케이션의 어떤 부분에 어떤 식으로 발생할지는 알 수가 없기 때문에, 개인적으로는 이러한 상황에서 어플리케이션을 아예 종료하고 다시 시작하는 것이 좋다고 생각한다.  
이때 위의 옵션이 도움된다.  
덧붙여 두 옵션의 차이점은, `ExitOnOutOfMemoryError`의 경우 단순히 어플리케이션이 종료되고 끝이지만, `CrashOnOutOfMemoryError`의 경우에는 어플리케이션 종료시 간략한 로그 파일을 남긴다는 점이다.  
때문에 `ExitOnOutOfMemoryError`보다는 `CrashOnOutOfMemoryError`이 더 나은 선택이라고 본다.  

`CrashOnOutOfMemoryError`의 경우 로그 파일의 위치는 다음 옵션으로 지정할 수 있다.  

``` Jvm
-XX:+CrashOnOutOfMemoryError -XX:ErrorFile=/logs/hs_err_pid%p.log
```

로그 파일은 텍스트 파일의 형식을 지닌다.  

## 도커에서의 사용에 대해

위와 같이 OOME가 발생하였을 때 제일 좋은 조치는 해당 어플리케이션을 종료하고 재시작시키는 것이라고 생각한다.  
재기동 시간에 요청이 들어오면 오류가 발생할 가능성이 있지만, 그보다도 메모리 오류에 의해 발생되는 미지의 오류가 더 문제가 될 것이기 때문이다.  
그렇기 때문에 일반적으로는 `OnOutOfMemoryError` 옵션을 통해 어플리케이션을 재실행시키면 된다.  
하지만 도커 컨테이너를 사용할 경우, `restart=always` 등의 옵션을 사용하면 어플리케이션이 종료되었을 때 자동으로 컨테이너를 재실행시키는 것이 가능하다.  
따라서 다음과 같이 설정하는 게 편리할 수 있다.  

``` jvm
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/heapdump.hprof 
-XX:OnOutOfMemoryError=/scripts/log.sh
-XX:+CrashOnOutOfMemoryError
-XX:ErrorFile=/logs/hs_err.log
```

log.sh 파일에 date 로깅이나 메일 전송 등의 로직을 넣으면 된다.  
다만 이와 같이 설정하였다 하더라도 OOME가 너무 자주 발생한다면(1~2시간 간격?) 이것도 결국 근본적인 해결은 되지 않을 것이다.  
본인의 경우 며칠에 한 번 정도 꼴로 발생하였으므로 이러한 설정이 큰 문제가 되지는 않았다.  

## 마무리

OutOfMemory가 발생하는 시점에 어떠한 식으로 대처할 수 있을지 적절한 대응에 대해서 고찰해보았다.  
물론 서버 환경이나 스펙, 프레임워크 등이 다르므로 어떠한 조치를 취할 수 있을지는 그때그때 다를 것이다.  
여담이지만 힙 덤프나 로그 파일 등은 이름이 같을 경우 이전 파일을 덮어씌우지 않는다.  
만약 새로운 파일을 생성해야 할 경우 파일 이름에 동적으로 날짜 등을 넣어야 할 수도 있다.  
다만 log파일이라면 모르겠으나 hprof 힙 덤프 파일은 해당 시점의 메모리만큼의 저장 공간을 차지하기 때문에 저장 공간이 남아도는 것이 아니라면 부담될 수도 있을 것이다.  

## 참고 사이트

<https://medium.com/tier1app-com/outofmemoryerror-related-jvm-arguments-5403e2d17f55>  
