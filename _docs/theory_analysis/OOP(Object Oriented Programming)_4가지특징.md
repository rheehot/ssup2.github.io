---
title: 병행성(Concurrency), 병렬성(Parallelism)
category: Theory, Analysis
date: 2017-01-23T13:42:00Z
lastmod: 2017-01-23T13:42:00Z
comment: true
adsense: true
---

### 1. 객체 (Object)

* Software로 구현할 실제 대상을 나타낸다. 각 객체는 상태(State)와 행동(Behavior)으로 이루어져 있다. 사람이라는 객체는 이름, 나이 같은 상태를 가지고 있으며 뛰거나, 걷거나 하는 동작을 할 수 있다.

### 2. 클래스 (Class)

* 객체를 나타내기 위한 설계도(BluePrint)이다. 하나의 객체는 수많은 상태와 행동으로 표현될 수 있다. 하지만 프로그래밍 과정에서 이러한 모든 상태와 행동을 프로그램으로 구현하지 않는다. 프로그램의 목적에 따라 필요한 상태와 행동만을 모델링하여 구현하게 된다. 특정 프로그램이 단순히 사람의 나이 정보만을 관리한다면 사람 Class는 나이를 변수수로 갖고 있고 나이를 증가시키는 Method를 가지고 있을 것이다. 하지만 프로그램이 나이 뿐만 아니라 사람의 신상 정보를 관리한다면 사람 Class는 나이 뿐만 아니라 성별, 주소 등 다양한 변수와 그에 따른 Method를 갖게 된다. 이렇게 프로그램의 목적에 따라 객체를 나타내 주는 설계도가 클래스이다.

### 3. 인스턴스 (Instance)

* 인스턴스는 Class로 설계된 객체가 프로그램 안에서 실체화 된 것을 의미한다. 각 인스턴스는 프로그램을 통해 메모리에 할당되면서 실체화 된다.

### 4. 참조

* [https://alfredjava.wordpress.com/2008/07/08/class-vs-object-vs-instance](https://alfredjava.wordpress.com/2008/07/08/class-vs-object-vs-instance)
* [http://cerulean85.tistory.com/149](http://cerulean85.tistory.com/149)
