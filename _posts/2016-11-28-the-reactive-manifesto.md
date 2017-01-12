---
layout: post
title: 리액티브 선언문 (The Reactive Manifesto)
---

본 글은 [The Reactive Manifesto](http://www.reactivemanifesto.org/)를 번역한 것입니다.

서로 다른 분야에 종사하는 조직들이 유사한 소프트웨어 구축 패턴을 독립적으로 발견한다. 이 시스템들은 더 견고하고 더 복원력이 뛰어나고 더 유연하며 현대의 요구조건을 충족하기 위한 우위에 서 있다.

이러한 변화는 최근 몇 년 사이 급격하게 변한 애플리케이션 요구사항에 기인한다. 불가 몇 년 전만 해도 큰 애플리케이션은 수십대의 서버와 초 단위의 반응 시간, 수 시간의 오프라인 유지보수 그리고 기가바이트의 데이터를 처리를 의미했다. 오늘날 애플리케이션은 모바일 기기부터 수천개의 멀티 코어 프로세서를 사용하는 클라우드 기반 클러스터에 이르기까지 모든 곳에 배포된다. 사용자는 밀리 초의 반응시간과 100% 가동 시간을 기대한다. 데이터는 페타바이트 단위로 측정된다. 이러한 오늘날의 요구사항들은 단순히 기존의 소프트웨어 구조로 충족될 수 없다.

우리는 시스템 아키텍처에 대한 일관된 접근이 필요하다고 믿으며, 필요한 모든 측면이 이미 개별적으로 인식되고 있다고 믿는다: 우리는 반응이 빠르고 복원력이 좋으며 탄력적인 메시지 기반의 시스템을 원한다. 우리는 이러한 것들을 리액티브 시스템이라고 부른다.

리액티브 시스템으로 구축된 시스템들은 더욱 유연하고 느슨하게 연결되며 [확장 가능](/the-reactive-manifesto-glossary/#Scalability)하다. 이러한 점들은 소프트웨어를 더 쉽게 개발하고 변경할 수 있게 한다. 리액티브 시스템들은 [장애](/the-reactive-manifesto-glossary/#Failure)에 훨씬 더 견고하며 장애가 발생했을 때 재앙을 일으키기 보다는 우아하게 대응한다. 리액티브 시스템은 반응이 매우 빨라 [사용자들](/the-reactive-manifesto-glossary/#User)에게 효과적인 상호작용을 제공한다.

*리액티브 시스템들은:*

* <a name="Responsive"></a>**즉각 반응한다(Responsive)**: [시스템](/the-reactive-manifesto-glossary/#System)은 가능한 적시에 응답한다. 응답성은 사용성과 유용성의 기초일 뿐만 아니라 그 이상으로 문제가 신속하게 감지되고 효과적으로 처리 될 수 있음을 의미한다. 반응이 빠른 시스템은 신속하고 일관된 응답 시간을 제공하고 신뢰할 수있는 상한을 설정하는데 집중하여 일관된 서비스 품질을 제공한다. 이러한 일관된 동작은 오류 처리를 단순화하고 최종 사용자의 신뢰를 높이며 상호 작용을 촉진한다.
* <a name="Resilient"></a>**복원력이 좋다(Resilient)**: 시스템은 [장애](/the-reactive-manifesto-glossary/#Failure) 발생 시에도 응답을 유지한다. 이는 고 가용성의 절대 멈춰서는 안되는 시스템(mission critical systems)에만 적용되는 것은 아니다 — 복원력이 좋지 않은 시스템은 장애 발생 후 응답하지 않는다. 복원성은 [복제](/the-reactive-manifesto-glossary/#Replication), 억제, [분리](/the-reactive-manifesto-glossary/#Isolation) 및 [위임](/the-reactive-manifesto-glossary/#Delegation)에 의해 달성된다. 장애는 각 [컴포넌트](/the-reactive-manifesto-glossary/#Component) 내부에 억제되며, 컴포넌트 서로를 분리하여 시스템의 일부에 장애가 발생하더라도 시스템 전체를 손상시키지 않고 복구 할 수 있도록 한다. 각 컴포넌트의 복구는 다른 (외부) 컴포넌트로 위임되며 필요한 경우 복제를 통해 고 가용성을 보장한다. 컴포넌트의 클라이언트는 장애 처리를 부담하지 않는다.
* <a name="Elastic"></a>**탄력적이다(Elastic)**: 시스템은 다양한 작업 부하에서도 빠르게 응답한다. 리액티브 시스템은 입력을 처리하도록 할당 된 [자원](/the-reactive-manifesto-glossary/#Resource)을 늘리거나 줄임으로써 입력량의 변화에 ​​대응할 수 있다. 이는 데이터 경합점이 없거나 중앙 병목 현상이 없는 설계을 의미하고 결과적으로 컴포넌트를 분할하거나 복제할 수 있게 만들고 입력을 컴포넌트들로 분산할 수 있게 한다. 리액티브 시스템들은 의미있는 실시간 성능 측정을 제공하여 예측가능한 반응형 확장 알고리즘을 지원한다. 이들은 범용 하드웨어 및 소프트웨어 플랫폼에서 비용적으로 효과적인 방식으로 [탄력성](/the-reactive-manifesto-glossary/#Elasticity)을 달성한다.
* <a name="Message-Driven"></a>**메시지 기반이다(Message Driven)**: 반응형 시스템은 [비동기](/the-reactive-manifesto-glossary/#Asynchronous) [메시지 전달](/the-reactive-manifesto-glossary/#Message-Driven)에 기반하여 느슨한 결합, 분리, [위치 투명성](/the-reactive-manifesto-glossary/#Location-Transparency)을 보장하는 컴포넌트 간의 경계를 형성한다. 이 경계는 또한 [장애](/the-reactive-manifesto-glossary/#Failure)를 메시지로 할당하는 수단을 제공한다. 명시적인 메시지 전달을 사용하는 것은 시스템의 메시지 큐를 구축하고 모니터링하며 필요 시 [역압](/the-reactive-manifesto-glossary/#Back-Pressure)을 적용하여 부하 관리, 탄력성 그리고 흐름 제어를 가능하게 한다. 통신 수단으로서 위치 투명성 메시징은 장애 관리가 클러스터 전체 또는 단일 호스트 내에서 동일한 구문과 의미로 작동 할 수 있게 한다. [논 블로킹](/the-reactive-manifesto-glossary/#Non-Blocking) 통신은 수신자가 활성화 시에만 [자원](/the-reactive-manifesto-glossary/#Resource)을 소비할 수 있게 하여 시스템 오버헤드를 줄인다.

대형 시스템들은 작은 시스템들로 구성되므로 구성 요소의 리액티브 특성에 의존한다. 즉, 리액티브 시스템은 설계 원칙을 적용하여 이러한 특성이 모든 크기의 수준에서 적용될 수 있도록 하여 시스템들을 조합할 수 있게 만든다. 세상에서 가장 큰 시스템은 이러한 속성을 기반으로하는 아키텍처에 의존하여 매일 수십억 명의 사람들의 요구에 부합한다. 이러한 설계 원칙을 매번 재발견하는 대신 처음부터 의식적으로 적용해야 할 때이다.

[선언문에 서명하기](http://www.reactivemanifesto.org/#sign-button)
