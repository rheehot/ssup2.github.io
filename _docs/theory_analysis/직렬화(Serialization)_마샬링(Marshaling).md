---
title: 직렬화(Serialization), 마샬링(Marshaling)
category: Theory, Analysis
date: 2017-02-20T00:07:00Z
lastmod: 2017-02-20T00:07:00Z
comment: true
adsense: true
---

직렬화(Serialization)과 마샬링(Marshaling)은 거의 비슷한 의미를 가지고 있는 용지만 약간의 Semantic 차이를 가지고 있다.

### 1. 직렬화(Serialization)

* 직렬화는 **Transform(변경)**의 의미를 가지고 있다. 구조화된 Data를 외부로 전달할 수 있게 Byte Stream같은 Primitive 형태로 변경하는 과정을 의미한다. 반대로 Primitive 형태의 Data를 구조화된 Data로 변경하는 과정을 역직렬화(Unserialization)라고 한다.
* 구조화된 Data를 외부에 전달하기 위해 JSON/XML로 변경하는 과정을 직렬화 과정이라고 하고 반대로 JSON/XML 정보를 구조화된 Data로 변경하는 과정을 역직렬화 과정이라고 한다.

### 2. 마샬링(Marshaling)

* 마샬링은 **Move(전달)**를 의미를 가지고 있다. 구조화된 Data를 외부로 전달하는 과정을 의미한다. 반대로 내보내진 정보를 구조화된 Data로 전달하는 과정을 언마샬링(Unmarshalling)이라고 한다.
* 마샬링/언마샬링 과정에는 Data의 변경이 발생할 수 밖에 없다. 따라서 마샬링/언마샬링은 직렬화/역직렬화의 상위 개념이라고 할 수 있다. Python이나 Golang의 경우 구조화된 Data와 JSON/XML 사이의 변경 과정을 마샬링/언마샬링 이라고 표현한다. Java에서 직렬화는 객체의 상태만 Byte Stream으로 변경하는 과정을 의미하지만, 마샬링은 객체 자체를 전달하는 상위 과정을 의미한다.

### 4. 참조

* [http://softwareengineering.stackexchange.com/questions/149493/whats-the-difference-between-a-marshaller-and-a-serializer](http://softwareengineering.stackexchange.com/questions/149493/whats-the-difference-between-a-marshaller-and-a-serializer)
* [http://stackoverflow.com/questions/770474/what-is-the-difference-between-serialization-and-marshaling](http://stackoverflow.com/questions/770474/what-is-the-difference-between-serialization-and-marshaling)
