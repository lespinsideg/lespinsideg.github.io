---
layout: post
title: 리액티브하게 리팩토링하기 - JDBC 마이그레이션 해부
---

본 글은 [Nicolae Marasolu](https://www.infoq.com/author/Nicolae-Marasoiu)의 [Refactoring to Reactive - Anatomy of a JDBC migration](https://www.infoq.com/articles/Refactoring-Reactive-JDBC)를 번역한 것입니다.

## 핵심 요약

* 리액티브 프로그래밍은 러닝 커브가 있으며 완전한 이해를 위해서 경험, 연습 그리고 열린 마음이 필요하다.
* 어떠한 애플리케이션이든 점진적으로 리액티브 프로그래밍을 도입할 수 있다.
* 리액티브 프로그래밍과 Netty와 같은 논 블로킹 라이브러리들은 확장성과 탄력성을 증가시키고 개발, 운영 비용을 낮춘다.(자세한 내용은 [리액티브 선언문](http://www.reactivemanifesto.org/) 참고 - [번역](/the-reactive-manifesto/))
* 퓨처 값의 스트림을 다른 퓨처 스트림으로 변형하는 것은 강력한 프로그래밍 패러다임이며 연습에 따른 큰 보상이 따른다.
* 결합성은 함수형 프로그래밍의 특징이다. 이 글에서 우리는 Observable의 flatMap 연산자를 이용한 모나드 결합을 살펴본다.

리액티브 프로그래밍은 최근에 등장하여 동시성 관리와 흐름 제어와 같은 프로그래밍에서 가장 어려운 개념들에 대한 해결책을 제공하고 있다. 당신이 만약 리액티브를 사용하지 않는 애플리케이션 개발팀에서 일하고 있다면 이것은 좋은 기회이다. 리액티브 프로그래밍을 어떻게 시작할 수 있을지, 어떻게 테스트할지, 단계별로 도입할 수 있을지와 같은 궁금증이 생길 것이다.

이 글에서 우리는 (웹서버와 데이터베이스 백엔드를 가지는 고전적인) 실제 레거시 애플리케이션을 리액티브 모델로 변경할 것이다. 이러한 변경은 3가지 이점을 가진다.

a. 함수형 방식으로 일하는 것은 코드를 결합할 수 있도록 하여 코드를 재사용하게 하고 높은 추상화, 불변성, 플루언트(fluent, 유창한) 형태로 가독성을 높일 수 있다. 예를 들어 `Function<T,Observable<U>>`는 `Observable`의 `flatMap`과 같은 연산자를 이용하여 비슷한 여러 함수를 연결할 수 있다.

b. 탄력성이 높은 서버를 구축할 수 있다. 실행해야 할 코드들을 스레드 풀의 대기열에 대기시키지 않고 동시에 실행할 수 있도록 하여 서버가 실제 사용 가능한 스레드보다 더 많은 동시 사용자를 유지할 수 있다.

우리는 데이터베이스에서 데이터를 조회하는 동안 스레드를 대기시키는 방식이 아닌 데이터 덩어리를 받아올 때마다 애플리케이션이 반응하도록 하여 위 내용을 구현할 것이다. 이것은 풀 방식(블로킹 I/O)에서 푸시 방식으로 변형하는 것이다. 푸시 방식은 데이터가 사용 가능할 때나 시간 초과, 예외와 같은 어떠한 이벤트가 발생할 때마다 실행하도록 하는 것이다.

c. 데이터베이스에서 첫 바이트가 반환되는 순간 브라우저에 업데이트될 수 있도록 하여 더 반응성이 높은 애플리케이션을 개발할 수 있다. 이것은 서블릿 3.1의 스트리밍 API로 구현 가능하며 스프링 Reactor와 연관된 스프링 스택의 웹 확장 라이브러리들로 가능해질 것이다.

이 글의 예제는 [링크](https://github.com/nmarasoiu/Spring-JDBC/tree/pgasync)에서 받을 수 있다.

우리는 점진적 리팩토링을 사용하여 작은 단계로 우리 프로그램에 리액티브를 적용하려고 한다.

시작하기에 앞서 레거시 애플리케이션에 대한 테스트를 작성하자. 우리는 이 테스트에 의존하여 우리가 함수형으로 리액티브하게 변경한 프로그램이 올바르게 동작하는지 확인할 수 있다.

테스트가 완성되었다면 첫 번째 리팩토링을 진행해보자. 메서드의 반환형을 RxJava에 있는 `Observable`, `Single`, `Completable`으로 변경한다. 특히 `T` 타입의 값을 하나만 반환할 경우 `Single<T>`를 반환할 수 있다. `Single<T>`는 정확하게 하나의 값이나 에러를 반환하는 `Observable<T>` 이다. `List<T>`의 경우 `Observable<T>`로 변경할 수 있고 `void` 메서드는 `Completable`으로 변경한다. `Completable`은 `Single<Void>`로 전환할 수 있다.

모든 계층의 메서드를 전부 변경하는 것 보다는 한 계층만 선택하여 진행해보도록 하자. `T`를 `Sinlge<T>`로 `List<T>`를 `Observable<T>`로 `void`를 `Completable`으로 변경하는 것을 외부에 알려진 메서드에 적용할 필요는 없다. 메서드의 기존 시그니처를 변경하는 대신 `Observable<T>`를 반환하는 새로운 위임 메서드를 구현하자. (기존 메서드 시그니처를 유지하는) 위임 메서드는 observable의 `toBlocking`을 호출하여 값을 반환하기 전까지 기존의 동기 방식을 강제하도록 한다. 가능한 작은 부분에서 시작하는 것은 러닝 커브를 극복하고 꾸준하게 점진적으로 진행할 수 있도록 도와주는 훌륭한 마이그레이션 전략이다. RxJava를 사용하여 점진적인 리팩토링을 적용해보자.

구체적인 예제를 보자. [이곳](https://github.com/nmarasoiu/Spring-JDBC)에서 모든 코드와 히스토리를 볼 수 있다. 나는 [tkssharma/Spring-JDBC](https://github.com/tkssharma/Spring-JDBC)의 고전적인 스프링 애플리케이션을 RxJava로 변경하는데 두가지 방식을 사용하였다. (rx-jdbc 브랜치에 있는) RxJDBC 라이브러리를 이용하는 것과 (pgasync 브랜치에 있는) pgasync 라이브러리를 이용하는 것이다. Spring-JDBC 프로젝트에 있는 다음 메서드를 보자 :

```java
List<Student> getStudents();
Student getStudent(int studentId);
```

위에서 이야기 한 마이그레이션 전략에 따라 메서드 시그니처를 유지하고 구현에 약간의 변화를 줄 것이다 :

```java
public List<Student> getStudents() {
  return getStudentStream().toList().toBlocking().single();
}
```

추가적인 계층를 도입하였다:

```java
Observable<Student> getStudentStream();
Single<Student> getStudent(int studentId);
```

계층 구현:

```java
public Observable<Student> getStudentStream() {
  List<Student> students = getStudentsViaJDBC();
  return Observable.from(students);
}
```

`getStudentsViaJDBC`이 최초의 구현이다.

우리는 기존의 리액티브가 아닌 시그니처를 유지한 상태로 리액티브 계층을 도입하였다. 그리고 원래의 구현을 새로운 리액티브 호출로 교체한 것이다. 우리는 데이터 접근 계층에 대해 몇 차례 수정을 더 할 것이다. 그리고 컨트롤러 계층으로 리액티브를 옮겨 가서 최종적인 목표는 전체적으로 리액티브한 시스템을 만드는 것이다.
`Observable.toBlocking`은 기존 세상과 리액티브 세상을 연결하는 다리 역할을 한다. 이것은 서블릿과 JDBC를 양쪽 끝단으로 사용하는 대형 시스템의 고전적인 코드에 (API만이라도) 리액티브 코드를 적용하는 데 필요하다. 서블릿과 JDBC 양쪽 끝을 리액티브로 변경하기 전까지 우리는 이 메서드가 필요하다.

물론 리팩토링이 끝나면 동기 방식의 `toBlocking` 요청은 비동기 방식의 리액티브 패러다임에 부합하지 않으므로 최종 운영 코드에서는 해당 호출을 제거할 것이다.

`List<Student> getStudents()`라는 메서드가 있다고 해보자. `Observable<Student> getStudentStream()`이라는 새로운 메서드를 생성하여 앞서 말한 메서드의 구현을 옮길 수 있다. 이때 `Observable.from(Iterable)`을 이용하면 결과 리스트를 Observable로 감쌀 수 있다. 기존의 메서드는 새로운 메서드를 호출할 것이고 `studentStream.toList().toBlocking().single()`를 통해 다시 `List`로 변경된다. 이것은 블로킹 호출이지만 이미 `getStudentStream`이 블로킹이기 때문에 현시점에서는 괜찮다.

리액티브를 배우는 데 가장 큰 어려움은 Observable의 관점으로 생각하는 것이다. 리스트가 (Observable과 같은) 스트림이 된다는 것은 직관적으로 알 수 있을 것이다. 하지만 개별적인 값에 이 개념을 적용하는 것은 직관적이지 않다. 이를 개념화하기 위해 전통적인 자바의 Future를 생각해보자. Future는 이후에 발생하는 값들을 받아올 수 있는 스트림 중에서 하나의 값을 가지고 있거나 어떠한 값도 가져올 수 없어 에러가 발생하는 특정한 경우로 볼 수 있다.

첫 번째 단계에서는 실행 특성을 변경하지 않고 반환형을 `Observable`로 감쌌다. 이는 여전히 동기화 방식이고 JDBC처럼 블로킹 I/O를 수행한다.

우리의 리팩토링 중 `List<Student> getStudents()`의 시그니처를 `Observable<Student> getStudents()`로 변경하는 한 단계가 끝나간다. 흥미로운 점은 한 명의 학생을 반환하는 `Student getStudent()` 메서드도 `Observable<Student> getStudent()` 혹은 `Single<Student> getStudent()`로 리팩토링된다는 것이다. 나아가 `void` 메서드는 `Completable`을 반환하는 메서드로 변경된다. 리액티브 패러다임은 위에서 시작하여 아래로 적용해 나갈 수 있다. 프로그램의 크고 작은 일부분을 리액티브 API로 감싼 다음 비동기 방식이나 논 블로킹 I/O가 필요한 곳에서 세분화하여 적용하는 방식이다.

새로운 시그니처를 구현하기 위해 `studentList`를 반환하는 대신 `Observable.just(studentList)`를 반환하도록 한다.

이 시점에 우리는 `Observable`을 몇 군데 도입했다. 하지만 단순히 리스트를 감싸거나 푸는 것 외에는 기본적으로 변경된 것이 없다. 하지만 이것은 우리의 코드를 결합 가능하게 만는다는 점에서 중요하다. 그리고 우리는 이제 `Observable`의 강력한 기능인 느긋한 계산법(lazy evaluation)을 사용할 준비가 되었다. 느긋한 계산법은 자바 스트림에서도 사용할 수 있다. `Observable.just(studentList)`를 반환하는 대신 아래와 같이 반환하자.

```java
Observable.defer(
   ()->Observable.just(actuallyProduceStudentsViaTripToDatabase()
))
```

`actuallyProduceStudentsViaTripToDatabase`은 우리가 처음에 봤던 `List<Student>`를 반환하는 래거시 메서드이다. 이것을 `Observable.defer` 혹은 `Observable.fromCallable`로 감싸는 것으로 우리는 느긋한(lazy) Observable를 생성할 수 있다. 느긋한 Observable은 구독자(subscriber)가 해당 데이터를 구독하는 그 순간 데이터베이스로 질의를 보낸다.

지금까지는 데이터 접근계층 API만 `Observable`을 반환하도록 수정하였다. 컨트롤러 메서드는 아직 변경하지 않았다. 그렇기 때문에 컨트롤러 메서드는 observable을 소비(구독)하기 위해 같은 스레드에서 동기 I/O로 결과를 기다려야만 한다. 하지만 이 글에서 우리의 목표는 전체적으로 비동기 처리를 적용하는 것이므로 컨트롤러 메서드 또한 (탬플릿이 랜더링되는 시점에 데이터가 이미 사용 가능한) `Result`를 반환하는 것이 아닌 `DefferedResult` 클래스를 이용하여 비동기 스프링 MVC로 끝나길 원한다. `DefferedResult`는 스프링 MVC에서 제공하는 비동기 객체이다. (스프링은 스프링 Reactor 에코 시스템을 이용한 리액티브 웹에서 스트리밍을 지원할 예정이다) 이러한 접근 방식을 이용하여 컨트롤러 메서드는 완료된 `Result`를 반환하는 것이 아닌 결과가 사용가능할 때 이전에 반환한 `DefferedResult`에 데이터가 넘겨질 것이라는 약속(promise)을 반환한다. 컨트롤러의 메서드 반환을 `Result`에서 `DefferedResult`로 변경하는 것만으로 어느 정도 비동기를 제공할 수 있다.

```java
@RequestMapping(value = "/student.html", method = RequestMethod.GET)
public DeferredResult<ModelAndView> showStudents(Model model) {
   Observable<ModelAndView> observable = studentDAO.getAllStudents()
           .toList()
           .map(students -> {
               ModelAndView modelAndView = new ModelAndView("home");
               modelAndView.addObject("students", students);
               return modelAndView;
           });
   DeferredResult<ModelAndView> deferredResult = new DeferredResult<>();
   observable.subscribe(result -> deferredResult.setResult(result),
                             e -> deferredResult.setErrorResult(e));
   return deferredResult;
}
```

우리는 비동기에 대한 중요한 작업을 마쳤다. 하지만 놀랍게도 메서드는 여전히 데이터베이스가 결과를 반환할 때까지 기다린다. 즉 여전히 블로킹 상태이다. 왜 그럴까? 기억하고 있을지도 모르겠지만 데이터 접근계층에서 반환되는 Observable이 호출하는 스레드에서 구독되기 때문에 `DeferredResult`를 사용하더라도 메서드는 Observable이 데이터를 전달할 때까지 스레드 자원을 소비하며 블로킹 상태에 머물게 된다.

다음 순서는 Observable를 변경하여 구독 시점에 현재 스레드를 블로킹하지 않도록 하는 것이다. 이것은 두 가지 방식으로 해결할 수 있다. 하나는 native-reactive 라이브러리를 사용하는 것이고 다른 하나는 `Observable.subscribeOn(scheduler)`와 `observeOn(scheduler)`를 이용하는 것이다. 두 번째 방식은 subscribe과 observe를 다른 스케줄러(스레드 풀이라고 생각하라)에서 실행하도록 해준다.

observe 메서드는 `map`, `flatMap`, `filter` 와 같은 observable를 다른 observable로 변형하는 메서드들과 `doOnNext`와 같이 스트림에서 새로운 요소가 발생할 때 실행되는 행동을 포함한다. (`subscribeOn`을 사용하는) 두 번째 접근 방법은 완전히 논 블로킹을 지원하는 라이브러리로 가기 전 작은 중간 단계이다. 이것은 단순히 `subscribe`와 `observe`를 다른 스레드에서 실행하도록 해주는데 이것은 여전히 결과가 가능해지기 전 까지 스레드를 블로킹한다.(다른 스레드를 블로킹할 뿐이다) subscriber에게 데이터를 푸시하면 다시 그것을 `DeferredResult`로 푸시하는 방식이다. JDBC위에 RxJava를 구현한 라이브러리들은 이러한 블로킹 스레드(설정에 따라 호출 스레드 혹은 다른 스레드)의 방식을 사용한다. JDBC가 블로킹 API이기 때문에 현시점에서는 이러한 방식이 필요하다. 일반적으로 이러한 접근 방식은 완전히 비동기를 지원하는 라이브러리로 가기 위한 중간 단계로 사용된다. 궁극적으로는 native-reactive 방식이 목표이다. 이것은 확장성이 향상되어 사용 가능한 스레드 수보다 더 많은 수의 사용자 동시 작업(flows)을 제공할 수 있다.

RxJDBC를 이용한 `getStudent` 구현:

```java
public Observable<Student> getStudents() {
   Class<String> stringClass = String.class;
   return database
           .select("select id,name from student")
           .getAs(Integer.class, stringClass)
           .map(row->{
                   Student student = new Student();
                   student.setId(row._1());
                   student.setName(String.valueOf(row._2()));
                   return student;
               });
}
```

RxJDBC라이브러리를 사용하기 위해서 메이븐 프로젝트에 의존성을 추가하자:

```xml
<dependency>
   <groupId>com.github.davidmoten</groupId>
   <artifactId>rxjava-jdbc</artifactId>
   <version>0.7.2</version>
</dependency>
```

세 번째 단계는 진짜 리액티브 라이브러리를 도입하는 것이다. 관계형 데이터베이스에서조차도 리액티브 라이브러리가 잘 없지만 postgres와 같은 특정 데이터베이스에 초점을 맞추면 좀 더 많은 라이브러리를 찾을 수 있다. 그 이유는 데이터베이스 접근 라이브러리가 각각의 데이터베이스의 저수준 프로토콜마다 다르기 때문이다. 우리가 사용한 것은 RxJava를 사용하는 [postgres-async-driver project](https://github.com/alaisi/postgres-async-driver)이다.

이번에는 pgasync 라이브러리를 사용한 `getStudent` 구현이다:

```java
public Observable<Student> getStudents() {
     return database
     .queryRows("select id,name from student")
     .map(row -> {
         Student student = new Student();
         int idx = 0;
         student.setId(row.getLong(idx++));
         student.setName(row.getString(idx++));
         return student;
     });
 }
```

pgasync라이브러리를 사용하기 위해 의존성을 추가하자:

```xml
<dependency>
    <groupId>com.github.alaisi.pgasync</groupId>
    <artifactId>postgres-async-driver</artifactId>
    <version>0.9</version>
</dependency>
```

우리는 진정한 리액티브(비동기, 이벤트 기반, 논 블로킹) 백엔드를 구현했다. 또한 JVM의 스레드 수보다 많은 사용자 행위를 (I/O flow 수준에서) 동시에 처리할 수 있는 전체 비동기 솔루션을 만들었다.

다음으로 트랜잭션 작업을 진행해보자. INSERT나 UPDATE 같은 DML(data modification language) 연산을 이용하여 데이터를 수정하는 시나리오를 생각해보자. 하나의 DML로 이루어진 가장 간단한 트랜잭션에서조차 비동기 방식을 도입하는 것은 복잡하다. 이것은 우리가 스레드를 블로킹하는 트랜잭션에 익숙하기 때문이다. 여러 문장(statement)을 가지는 더 현실적인 트랜잭션의 경우에는 더 복잡해진다.

트랜잭션은 아래와 같이 생겼다:

```java
public class T {
 private Observable<Long> dml(String query, Object... params) {
   return database.begin()
           .flatMap(transaction ->
                   executeDmlWithin(transaction, query, params)
                           .doOnError(e -> transaction.rollback()));
 }

 private Observable<Long> executeDmlWithin(
       Transaction transaction, String query, Object[] params) {
   return transaction.querySet(query, params)
        .flatMap(resultSet -> {
            Long updatedRowsCount = resultSet.iterator().next().getLong(0);
            return commitAndReturnUpdateCount(transaction, updatedRowsCount);
        });
 }

 private Observable<Long> commitAndReturnUpdateCount(
       Transaction transaction, Long updatedRowsCount) {
   return transaction.commit()
        .map(__ -> updatedRowsCount);
 }
}
```

이것은 하나의 DML 구문으로 이루어진 트랜잭션이지만 비동기 리액티브 API에서 어떻게 트랜잭션이 이루어지는지 잘 나타내고 있다. 트랜잭션 시작, 커밋, 롤백은 모두 `Observable`을 반환하고 `flatMap`을 통해 연결될 수 있는 모나드(monad) 함수들이다.

예제의 시그니처부터 살펴보도록 하자. `dml`은 `UPDATE`나 `INSERT`와 같은 DML 구문과 파라미터를 받는 실행 함수이며 넘겨받은 DML 구문의 실행을 "예정(schedule)"한다. `db.begin`이 `Observable<Transaction>`을 반환한다는 것에 주목하라. 트랜잭션은 데이터베이스에 대한 I/O를 포함하므로 즉시 생성되지 않는다. 그래서 이것은 데이터베이스에 실행이 완료되면 요청한 SQL 쿼리에 대한 `Transaction` 객체를 반환하는 비동기 연산이다. `Transaction` 객체는 필요에 따라 `commit`하거나 `rollback` 할 수 있다. 이 `Transaction`은 자바의 클로저(closure)에서 다른 클로저로 전달된다. 위의 예제에서 볼 수 있듯이 `transaction`은 먼저 `flatMap`의 인자로 전달할 수 있다. 이것은 세 지점에서 사용된다:

* 먼저 이것은 `transaction`안에서 DML을 실행한다. DML을 수행하는 `querySet` 명령의 결과는 Observable이다. 이 Observable은 DML의 실행 결과(일반적으로 업데이트 된 행 수를 포함하는 `Row`)를 가진다. 그리고 이는 `flatMap`을 통해 다른 `Observable`로 변경된다.

* 두 번째 `flatMap`은 트랜잭션 객체를 트랜잭션 커밋에 사용한다. 트랜잭션 변수를 포함하는 람다 함수가 두 번째 `flatMap` 의 인자로 제공된다. 이 방식으로 비동기 흐름의 한 부분에서 다른 부분으로 데이터를 전달할 수 있다. 어휘(lexical) 범위의 변수를 람다 함수를 생성할 때 포함하여 이후에 다른 스레드에서 실행되도록 하는 것이다. 이것이 자바 클로저로써 람다 표현식의 중요성이다. 클로저는 표현식 안에 사용된 변수들을 포함한다. 람다뿐만 아니라 다른 자바 클로저에서도 이러한 방식으로 데이터를 보낼 수 있다.

* `transaction` 변수의 세 번째 사용은 트랜잭션이 롤백되는 `doOnError`이다. 다시 한번  `transaction` 변수가 어떻게 자바의 어휘 범위 지정을 통해 세 지점에 들어가는지 보라. 비록 코드 일부(호출하는 스레드 내 메소드 실행의 일부)는 동기적으로 실행되지만, 나머지는 데이터베이스의 응답과 같은 어떠한 이벤트가 발생할 때 비동기적으로 다른 스레드에서 실행된다. 하지만 `transaction` 변수는 이 모든 컨텍스트에서 사용할 수 있다. 이상적으로 공유되는 값들은 불변(immutable)이거나 무 상태(stateless) 혹은 스레드 안전(thread safe)해야 한다. 자바는 효과적인 방법으로 공유되는 값이 `final`이길 요구하지만 원시(primitive)형이 아닌 경우 이것으로 충분하지 않다.
성공적이라면 트랜잭션의 커밋 결과는 호출자가 사용할 수 있는 업데이트 행수로 변환(mapped)될 것이다. 업데이트/삽입 된 행 수를 트랜잭션 메서드 외부의 호출자에게 전달하기 위해서 우리는 자바의 클로저를 이용할 수 없다. 피 호출자는 호출자와 같은 어휘 범위에 존재하지 않기 때문이다. 이 경우에 우리는 결과를 Observable의 데이터형으로 캡슐화할 필요가 있다. 만약 다수의 결과를 가져와야 한다면 우리는 불변 클래스나 배열, 수정할 수 없는 컬렉션을 사용할 수 있다. 에러가 발생할 경우에는 트랜잭션에서 롤백이 호출된다. "이 Observable은 에러가 있으니 다른 Observable을 사용하거나 나중에 동일한 Observable로 재시도해." 라는 구체적인 Observable 명령을 통해 중단하지 않는 한 이 에러는 (호출 스택이 아니라) 연결된 Observable를 타고 전파된다.

이 트랜잭션 업데이트가 flatMap 연쇄 연결(chaining)의 첫 예제이다. 여기서 우리가 한 일은 여러 단계를 이벤트 방식으로 연결한 것이다. 트랜잭션이 시작되면 쿼리를 시작할 수 있다. 쿼리의 결과가 사용 가능하면 결과를 파싱하고 트랜잭션 커밋을 수행한다. 트랜잭션이 끝나면 결과는 (아무 정보도 포함하지 않는) 커밋성공과 결과(업데이트 행수)로 변경된다. 만약에 마지막 observable이 `Observable<Void>`가 아니라 `Observable<T>`였다면 데이터 전달 객체를 통해 `Long`과 함께 `T`를 받을 것이다.

리액티브 세상에서 우리는 블로킹 애플리케이션을 논 블로킹 상태로 변경시키는 것에 중점을 둔다. (블로킹 애플리케이션은 TCP 커넥션을 여는 것과 같은 I/O를 수행할 때 블로킹이 발생하는 것이다) 소켓을 열거나, 데이터베이스와 통신하거나(JDBC) 파일, 입력 스트림, 출력 스트림을 수행하는 대부분의 레거시 자바 API들은 블로킹 API이다. 예전의 서블릿 API들을 포함한 다른 많은 자바 구현체들 또한 마찬가지이다.

시간이 지나면서 논 블로킹에 대응하기 위한 것들을 도입하기 시작했다. 예를 들어 서블릿 3.x는 비동기와 스트리밍과 같은 몇 가지 개념을 통합하였다. 하지만 일반적인 J2EE 애플리케이션에서 쉽게 찾아볼 수 있는 블로킹 호출이 항상 나쁜것은 아니다. 블로킹 구조는 명시적으로 작성하는 비동기 API보다 이해하기 쉽다. C#, Scala, Hasekll과 같은 언어들은 블로킹 코드로부터 투명하게 논 블로킹을 구현할 수 있는 구조로 되어 있다. C#과 Scala의 비동기 고차 함수를 예로 들 수 있다. 내가 아는 한 자바에서 논 블로킹을 수행할 수 있는 가장 견고한 방법은 Reactive Streams이나 RxJava을 사용하거나 혹은 Netty와 같은 논 블로킹 라이브러리를 사용하는 것이다. 하지만 명시적으로 작성하는 것은 진입 장벽이 높을 수 있다. 하지만 스레드의 수보다 많은 동시 사용자를 처리해야 하거나 애플리케이션이 I/O 영역에 있어 비용을 최소화하고 싶을 때 논 블로킹을 도입하면 확장성, 탄력성, 비용 감소 측면에서 추가적인 향상을 얻을 수 있다.

탄력성이나 견고함을 이야기할 때 모든 스레드가 I/O를 기다리고 있는 상황을 고려해보는 것이 도움된다. 예를 들어 최신 JVM이 5000개의 스레드를 제공할 수 있다고 가정해 보자. 이것은 여러 웹 서비스에 5000개의 요청이 들어와 블로킹 애플리케이션의 각 스레드에서 대기 중이라면 그 시점에서 더는 사용자 요청을 처리할 수 없다는 의미이다. (단지 대기열에 넣는 일을 수행하는 특수한 스레드에 의해 대기될 수 있다) 기업 인트라넷과 같은 제어된 환경에서는 괜찮겠지만 스타트업에서 제품을 구매하려는 사용자가 갑자기 10배로 증가했을 때 발생하길 바라는 일은 아닐 것이다.

물론 갑작스러운 트래픽 증가를 해결하는 한 방식으로 수평적 확장이 있다. 더 많은 서버를 도입하는 것은 비용을 고려하지 않더라도 충분히 탄력적이라고 할 수 없다. 다시 말해 이것은 애플리케이션이 수행하는 I/O의 종류에 따라 다르다. 인터넷 서비스에서 HTTP만이 매우 느린 I/O이고 그 외의 내부 데이터베이스나 서비스의 모든 I/O가 HA(고 가용성)와 낮은 지연율을 가진다고 하더라도, 지구 반대편 클라이언트에게 전달되는 데이터는 HTTP를 통해 느리게 전송될 것이다.

이 문제는 전문적인 로드 밸런서의 일이지만 가장 "가용성이 높은" 내/외부 서비스가 언제 중단될지 알 수 없으며 가장 "지연시간이 낮은 실시간에 가까운" 서비스가 가비지 컬렉션때문에 느리게 응답할 수 있다는 것이다. 스택 일부만 블로킹되더라도 이 블로킹은 전체로 전파될 것이다. 이것은 가장 느린 I/O에서 스레드가 블로킹되기 시작한다는 것이고 트래픽의 5%를 차지하는 비즈니스에 중요하지 않은 단 하나의 느린 블로킹 접근때문에 자원을 가져오는 것이 중단될 수도 있다는 것을 의미한다.

애플리케이션을 논 블로킹으로 만드는 것이 많은 경우에 가치를 더할 수 있다는 것을 이해했을 것이다. 이제 우리의 레거시 애플리케이션으로 돌아가 보자. HTTP와 데이터베이스 접근에 대한 모든 계층이 블로킹 방식이다. 여기서부터 시작하도록 하자. 모든 수직적인 계층(HTTP와 데이터베이스 접근 계층)을 비동기로 만들기 전까지 전체 흐름은 비동기가 될 수 없다.

비동기와 논 블로킹 사이에는 차이점이 있다. 논 블로킹은 비동기를 포함하는 반면 비동기는 블로킹 호출을 단순히 다른 스레드로 이동시키는 것만으로 달성할 수 있다. 이것은 처음의 블로킹 방식과 같은 문제를 가지고 있지만, 점진적 접근방식에서 최종 목표로 가기 위한 한 단계로 사용할 수 있다. HTTP 측면에서 스트리밍은 아니지만, 비동기를 제공하는 서블릿 명세와 스프링 MVC의 현 상태에 대해서도 일부 이야기하였다.

비동기는 데이터베이스가 응답을 완료할 때 처리가 시작되는 것을 의미한다. 처리가 끝나면 웹 계층은 렌더링을 시작한다. 웹페이지 (혹은 JSON 페이로드)가 그려지면 HTTP 계층은 전체 응답 페이로드를 표시한다.

다음 단계는 스트리밍이다. 데이터베이스가 처리 계층에 "처리할 데이터가 있어"라고 이야기하면 처리 계층은 이것을 받는다. 이 전달에는 전용 스레드가 필요하지 않다. 예를 들어 NIO나 리눅스의 epoll은 논 블로킹이다. 이 개념은 백만개의 연결이 하나의 스레드를 통해 백만개의 연결중에 새로운 게 있는지 OS에게 물어보는 것이다. 그러면 처리 계층은 결과물을 "학생들"과 같은 의미 있는 단위로 변경한다. 때로는 데이터베이스 데이터가 한 학생 정보 중 일부분만을 가지고 있을때 처리 계층의 버퍼에 부분 정보를 유지하는 것이 유용하다. 대량의 데이터를 데이터베이스에서 가져온 뒤 그 학생의 모든 정보를 모아 상위 계층에 렌더링을 위해 보낼 수 있다. 이러한 파이프라인에서는 모든 컴포넌트가 세분화되어 스트림이 될 수 있다. 어떤 컴포넌트는 바이트를 왼쪽에서 오른쪽으로 복사만 하고 다른 컴포넌트는 전체 학생 객체를 전송한다. Spring MVC의 `DeferredResult`와 같이 HTTP 응답을 쓰기 전에 모든 결과물이 필요한 경우와 다르게 전체 중 일부만 전송하는 경우도 있다.

다시 리팩토링의 단계로 돌아가 보자:

1. 시그니처에 Observable 을 추가한다.
2. `observable.subscribeOn`과 `observable.observeOn(scheduler)` 를 이용하여 블로킹 연산(예: JDBC 요청)을 다른 스레드 풀로 이동시킨다.
3. 비동기 Spring MVC를 사용하여 비동기로 만든다.
4. 사용하는 데이터베이스에 특화된 논 블로킹 라이브러리를 사용하여 백엔드를 논 블로킹으로 만든다.
5. RxJava나 선호하는 리액티브 라이브러리를 사용하여 논 블로킹 구현을 감싼다.(우리와 같이 이미 싸여있지 않은 경우)
6. Vert.x를 사용하여 스트리밍으로 만든다.
7. writes를 한다.
8. 트랜잭션을 만든다.
9. 에러 처리를 검증한다.

앱을 실행하기 위해서:
우리는 PostgreSQL 데이터베이스를 사용하였다. PostgreSQL를 설치하고 postgres 사용자의 비밀번호를 "mysecretpassword"로 변경하거나 간단하게 docker를 설치하고 아래를 실행하라.

```sh
sudo docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=mysecretpassword -d postgres
```

이제 student.sql을 실행하여 테이블을 생성하고 샘플 데이터를 추가하라.
그리고 mvn install을 실행한 뒤 war 파일을 tomcat이나 jetty에 배포하라.
URL은 [이곳](http://localhost:8080/jdbctraining-1.0.0-BUILD-SNAPSHOT/)이며 "students"를 누르면 된다.

### 결합성에 대한 더 자세한 이야기
우리는 이미 결합성에 대하여 많은 것을 이야기했고 이것의 의미를 이해해야 한다. 리액티브 프로그래밍 특유의 맥락에서
이러한 결합이 어떻게 동작하는지 살펴보자. 이 결합을 "시퀀싱"이라고 부르도록 하자. 이것은 어떠한 입력을 출력으로 만들어내는 일이 단계적으로 일어나는 처리(processing) 파이프라인을 의미한다. 각 단계는 동기(예: 연산이나 값 변형)일 수 있고 비동기(예: 웹 서비스나 데이터베이스에서 데이터를 가져오는 것)일 수 있다. 파이프라인은 소비자로부터 당겨오거나(pull) 생성자로부터 푸시될 수 있다.

다른 예제를 생각해보자. 데이터 처리를 위해 데이터를 백엔드 서버에 보내고 결과를 응답으로 받는 논 블로킹 웹 서버를 만든다고 가정하자. 요청을 보내는 사용자에 대한 인증과 권한 적용도 필요하다. 백앤드(와 기타 시스템)에서 상태를 변경하고 최종 사용자에게 응답하는 요청 처리에 대한 파이프라인을 호출하는 것에 이미 처리 단계 중 일부가 나타나고 있다.

이러한 파이프라인을 결합할 때에는 이것이 동작이 동기 혹은 비동기인지 또는 얼마나 많은 재시도 횟수를 사용하는지와 같은 세부사항을 알 필요가 없는 것이 이상적이다. 비동기의 경우 어떠한 스레드 풀에서 처리할지 혹은 어떠한 논 블로킹 프레임워크가 사용할지와 같은 세부사항을 이야기한다.

반대로 처리 과정을 동기에서 비동기로 변경할 경우에는 외부의 변경 없이 모나드 함수의 내부 구현만 수정하면 된다.

설명을 위해 리소스 서버(앱 서버) 내부에서 JWT 토큰을 검증하는 단계가 필요하다고 해보자. 이 것은 토큰 페이로드에서 데이터를 확인하는 라이브러리로 해결할 수 있다. 또는 네트워크 호출을 통해 인증 프로바이더(IdP)에게 해당 사용자가 여전히 유효한지 알아보는 것과 같은 더 많은 것을 검증하여 해결할 수 있다.

모나드 함수를 정의해보자(flatMap를 가지고 있는 반환형이 모나드이다):

```java
Observable<Boolean> isValid(String token)
```

우리는 이것을 메모리상에서 CPU 집약적인 연산을 사용하는 토큰 복호화 라이브러리를 사용하여 토큰 서명과 토큰 만료 기간, id와 같은 정보를 검증하도록 구현할 수 있다.

혹은 구글을 IdP 서버로 사용한다면 구글에 요청을 보낼 수 있다.

두가지 경우에서 파이프라인를 포함하여 모든 함수 외부에서는  `Observable<Boolean>`이 어떻게 구현되었는지 알지 못한다. 그저 동일한 스레드에서 구독자를 호출할 뿐이다. 메모리 상에서 라이브러리를 사용하여 해결하는 구현에서는 `boolean isValid(token)` 함수와 동일하다. 혹은 구글과 I/O 통신한 결과를 파싱하여 응답을 boolean으로 반환하는 것일 수 있다. 설계는 구현을 알 수 없다.

우리는 또한 함수를 같은 시그니처(`String`->`Observable<Boolean>`)의 다른 함수로 감쌀 수 있다. 이를 통해 검증 상단에 재시도 구조를 더 할 수 있다.(구글에 요청을 하는 경우 HTTP 요청이 유실되거나 지연시간이 길 때 사용할 수 있다) 혹은 구글과 같은 외부 데이터 센터에 네트워크 접근이 불가능한 경우 라이브러리를 이용한 검증을 시도하는 우아한 성능 저하(degradation) 기능을 추가할 수 있다.

이러한 모든 대안 해결책이나 데코레이터들을 추가할 수 있으며 그렇다 하더라도 여전히 함수는 `String` 을 받아 `Observable<Boolean>` 을 반환할 것이다.

동기에서 비동기로 변경하거나 되돌리는 것이 API에 영향을 주지 않기 때문에 커플링이 낮다.

자바의 `Future`와 다르게 `Observable`은 결합이 가능하다. 토큰 검증의 경우에서 일반적인 `Response`와 `ErrorResponse` 둘 중 하나를 반환하는 함수가 있다고 해보자.

(여러 개의 토큰을 기다리지 않고 `Future<String>`과 같이 하나의 토큰만 기다리는) `Observable<String>`이 있다고 해보자. 우리는 이 "토큰 observable"에 `flatMap`을 적용하는데 `isValid` 함수를 사용하여 "boolean observable"을 얻을 것이다. 이 "boolean observable"에 "if" 구문을 사용하는 람다 함수를 `flatMap`로 적용했다. 만약 토큰이 유효하면 `Observable<Response>`를 반환하고 아니면 다른 `Observable<ErrorResponse>`를 반환한다.

코드는 아래와 같이 작성할 수 있다:

```java
responseObservable = tokenObservable.flatMap(token -> isValid(token)
  .flatMap(valid -> valid? process(request) :
                           Observable.just(new ErrorResponse("invalid"))));
```

`Observable<T>` 형의 값에 flatMap을 적용하면 다른 `Observable<U>`를 얻을 수 있다는 것을 볼 수 있다. 여기서 T와 U는 같은 형일 수도 있고 다른 형일수도 있다.

 결합은 중요한 속성이다. 특정한 모양의 작은 컴포넌트들을 결합하여 같은 모양의 큰 컴포넌트들을 이룰 수 있다. 여기서 모양이라는 것은 무엇일까?

모나드는 형 파라미터 T를 가지는 형이라는 것과 `flatMap`, `lift` 라는 두 함수로 나타낼 수 있다. 후자는 쉽다. T 형의 객체를 모나드의 형으로 변경하는 것이다.  `Observable.just(value)` 와 `Option.ofNullable(value)` 이 이러한 모나드의 두 예제이다.

`flatMap`은 어떤가? 이것은 고차 함수로 소스(source)라고 하는 `observable<T>` 객체와 모나드 함수 `f(T->Observable<T>)`가 주어졌을 때 `Observable<U>`의 형을 가지는 새로운 Observable을 `newObservable = sourceObservable.flatMap(t->f(t))`로 생성할 수 있다. Observable의 경우 소스의 T형 요소가 사용 가능 해질 때 함수 `f`가 호출된다. 이것은 새로운 observable을 생성하는데 요소마다 새로운 Observable이 생성된다. 이들은 `newObservable`의 요소가 되어 생성된 순서에 따라 결과 `Observables<U>`에서 나타나게 된다. 왜 `Observables<U>`일까? sourceObservable이 3개의 요소를 내보낼 때 각각의 요소에 함수 f가 적용되어 3개의 Observable을 만들어내기 때문이다. 이들은 병합되거나 연결될 수 있다. 병합은 3개의 observable의 모든 요소가 발생하는 순서로 그 시점에 newObservable "출력"에 더해진다는 것을 의미한다. 3 observable의 결과를 병합하는 것이 `flatMap`이 하는 일이다. 다른 방식으로는 첫 번째 observable에서 모든 결과가 나오길 기다린 후에 두 번째 observable의 요소를 연결할 수 있다. 이처럼 모든 observable의 결과를 연결하는 것이 `concatMap`이 하는 일이다.

이러한 Observable의 특징을 이용하여 하나의 Observable 값에 처리 단계를 추가하거나 재시도, 대체 시스템 메커니즘과 같은 데코레이터 기능을 추가한 새로운 Observable을 만들 수 있다. 이것이 내가 결합성이라고 말하는 것의 주요 부분이다.

### 논 블로킹의 표면 아래
앞서서 논 블로킹 비동기 I/O 라이브러리를 사용하면 사용 가능한 스레드보다 많은 흐름을 유지할 수 있다고 언급했었다. 어떻게 이러한 일이 가능한지 궁금할 것이다. Netty(Vert.x와 PgAsync에서 사용하는 논 블로킹 I/O 라이브러리)와 같은 라이브러리가 어떻게 동작하는지 살펴보도록 하자.

자바는 NIO라는 API가 있다. 이 API는 적은 개수의 스레드에서 더 많은 연결을 처리하는 것에 중점을 두고 있다. NIO는 특정한 OS 시스템 호출(리눅스의 경우 epoll과 poll)을 이용하여 동작한다. 예를 들어 1000개의 연결이 열려있다고 가정해보자. 한 스레드는 selector.select라는 NIO 메서드를 호출할 것이다. 이것은 블로킹 요청으로 마지막 요청 이후 쌓여있던 "더 많은 데이터 사용 가능", "연결이 닫힘"과 같은 이벤트들이 발생한 연결을 10개 가져온다. 이제 그 스레드는 10개의 이벤트를 다른 스레드들로 전달하고 계속 새로운 요청을 폴링할 것이다. 그렇게 이 첫 스레드는 무한 루프를 돌면서 열려있는 연결들에 대한 새로운 이벤트 질의를 하는 것이다. 받아온 10개의 이벤트는 처리를 위해 스레드 풀이나 이벤트 루프에 전달될 것이다. Netty는 이벤트 처리를 위한 대기열 제한이 없는 스레드풀이 있다. 이벤트 처리는 (계산 집중적인) CPU 기반이다. 모든 I/O는 다시 NIO로 위임될 것이다.

이 세가지 기술에 대해 깊게 다루고 있는 자료로 Tomasz Nurkiewicz와 Ben Christensen이 쓴 [Reactive Programming with RxJava Creating Asynchronous, Event-Based Applications](http://shop.oreilly.com/product/0636920042228.do#tab_04_2)이 있다.

### 글쓴이에 대하여
*Nicolae Marasoiu* 는 열정적인 소프트웨어 개발자로 스타트업부터 잘 정립된 회사에 이르기까지 다양한 아웃소싱 회사들과 제품의 고성능 서버사이드 애플리케이션 구축에 대한 수년간의 경험을 가지고 있다. 그는 제품 개발의 다양한 분야에 기여하는 것과 팀의 기술적인 모험에 영감을 주는 것을 좋아한다.
