---
title: Event Driven Architecture on Linux
category: Theory, Analysis
date: 2017-09-20T12:00:00Z
lastmod: 2017-09-20T12:00:00Z
comment: true
adsense: true
---

Event Driven Architecture를 분석하고 Linux에서 동작하는 Event Driven Architecture를 설계한다.

### 1. Event Driven Architecture

![]({{site.baseurl}}/images/theory_analysis/Event_Driven_Architecture_on_Linux/Event_Driven_Architecture.PNG){: width="500px"}

Event Driven Architecture는 **Event**와 해당 Event를 처리하는 **Event Handler**로 구성된다. 또한 **Main Loop**라는 Single Thread만 이용한다는 점이 특징이다. Main Loop Thread는 평소에 Blocking되어 있다가 Event가 발생하면 해당 Event가 어떤 Event인지 파악한 후 해당하는 Event Handler를 실행한다. 그 후 다시 Blocking 상태가 되어 다음 Event가 올 때까지 대기한다. 이렇게 평소에 Blocking 되어 있다가 발생한 Event를 알려주는 역활을 하는 함수를 **I/O Multiplexer**라고 한다. Main Loop Thread가 Event 발생순으로 매우 빠르게 Event Handler들을 실행하기 때문에 Concurrent 프로그램처럼 동작하게 된다.

{% highlight C++ %}
int event_handler1(event *ev){
  // Non-blocking
}

int event_handler2(event *ev){
  // Non-blocking
}

int main()
{
  // Init I/O multiplexer
  IOMultiplexer multiplexer;

  // Registe event to multiplexer
  multiplexer.Add(ev1)
  multiplexer.Add(ev2)

  // Run main loop
  while(ture){
    ev_list = multiplexer.wait() // Only blocked here

    for(ev:ev_list){
      switch(ev){
        case: ev1
          event_handler1();
          break;
        case: ev2
          event_handler2();
          break;    
      }
    }
  }
}
{% endhighlight %}

Event Drivent Architecture는 Main Loop라는 Single Thread를 이용하기 때문에 Race Condtion이 발생하지 않는다. 따라서 Lock을 이용한 자원 동기화가 불필요 하기 때문에 프로그래밍이 간단하다는 큰 장점이 있다. 하지만 몇가지 단점도 가지고 있다.

먼져 Main Loop는 Event Handler안에서 Blocking상태가 되면 안된다. 오직 Event를 대기하는 곳에서만 Blocking 상태가 되어야 한다. Event Handler안에서 Blocking 상태가 되면 그 사이에 발생한 Event는 처리되지 못하기 때문에 프로그램의 반응성이 크게 떨어진다. Event Handler안에서 발생하는 Blocking 동작은 대부분은 I/O 동작을 위한 read(), write() 같은 System Call에서 발생한다. 이러한 Blocking 현상을 막기 위해서는 Non-blocking Option을 통해서 System Call을 호출하거나 AIO(Async I/O)를 이용해야 한다.

또 하나의 단점은 Single Thread로 동작하기 때문에 Mult-Core CPU를 100% 이용할 수 없다는 단점이 있다. 따라서 Event Driven Architecture만 이용해서는 CPU Bound 프로그램에 제대로 대응 할 수 없다.

### 2. On Linux

#### 2.1. I/O Multiplexer

Linux에서는 **select(), poll(), epoll()** 3개의 I/O Multiplexer를 제공한다. Linux에서 모든 프로그램은 fd(File Descriptor)를 이용하여 I/O를 처리한다. 이러한 특징을 이용하여 I/O Multiplexer는 fd의 상태 변화를 감지하고 프로그램에게 알려주어 Multiplexing을 수행한다. 일반적으로 성능이 가장 좋은 epoll()이 가장많이 이용된다. 하지만 epoll()은 POSIX 표준이 아니이고 Linux Kernel 2.6 이후에서만 지원하기 때문에 개발 환경에 맞게 select()나 poll()을 선택해야 한다.

#### 2.2. fd Heler functions

Linux에서 지원하는 I/O Multiplexer는 모두 fd를 기반으로 동작한다. 이러한 특징 때문에 프로그램으로 전달되는 Event들을 fd로 받거나 또는 fd로 전달 할 수 있어야 한다. Linux Kernel에서는 이러한 Event / fd 전환을 도와주는 timerfd(), signalfd(), eventfd() 함수들이 존재한다.

timerfd() 함수는 일정한 주기로 fd를 읽기 가능 상태로 바꾸어 준다. 따라서 timerfd()를 이용하면 일정한 주기로 Event Handler를 실행시킬 수 있다. signalfd()는 Linux Kernel로 부터 오는 Signal을 fd의 변화로 전환해주는 역활을 한다. signalfd()를 통해서 signal 처리를 event handler에서 수행 할 수 있다.

eventfd()는 pipe()의 개량형으로 fd를 통해서 Data를 주고 받을 수 있다. eventfd()는 pipe()와 다르게 하나의 fd를 통해서 Data를 주고 받을 수 있다. 또한 User/User, User/Kernel, Kernel/Kernel Thread간의 또는 동일 Thread간의 Data 전달이 가능하여 Thread 제한이 없다는 장점을 갖고 있다. 또 하나의 중요한 특징중 하나는 전송할 수 있는 Data의 크기가 최대 64byte라는 점이다. 따라서 64Byte보다 큰 Data를 전달하기 위해서는 Shared Memory 공간을 만들고 그 위에 큐를 만들어 전달해야 한다. 64Byte의 제한이 있지만 pipe()와 다르게 전달 할 Data의 Kernel Memory에 복사과정이 없기 때문에 성능이 빠르다. eventfd()를 이용하여 Event Handler간의 통신이나 다른 Thread간의 통신을 구현 할 수 있다.

#### 2.3. Architecture Design

![]({{site.baseurl}}/images/theory_analysis/Event_Driven_Architecture_on_Linux/Event_Driven_Architecture_on_Linux.PNG){: width="500px"}

epoll()과 fd Helper Function들을 이용하면 위의 그림과 같은 Architecture 설계가 가능하다. Handler 간의 통신의 경우 전달해야 할 Data의 크기가 64Byte보다 크다면, Share Memory를 필요없고 전역 공간에 Queue를 만들어 이용하면 된다.
