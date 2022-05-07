---
title: "Scala의 예외 처리 - Option, Either, Try"
date: 2020-04-12 18:10 +0900
categories: Programming
toc: true
---

Scala에서는 JVM 기반 언어 최대의 적인 `NPE (NullPointerException)`를 functional하게 handling 할 수 있는 다양한 수단을 제공하고 있습니다. Scala의 exception handling 3인방인 `Option, Either, Try` 에 대해 알아보도록 하겠습니다!

### Option
Java에서는 빈 List에 `find` 를 할 때 값이 없으면 `null`을 반환하거나 `exception throw`를 하는 것이 일반적인 상황입니다. 이 일반적인 상황은 매우 위험한 상황이 될 수 있는데 잘못해서 null을 reference 했다간 큰 일이 날 수 있고, exception throw 또한 호출부에서 잘 해결해주지 않으면 프로그램이 뻗어버리기 때문입니다. `exception` 이 발생했다고 해서 프로그램이 뻗어버리는 것은 좋은 상황은 아니죠.

Scala에서는 이러한 일을 좀 더 우아하게 처리해줄 수 있게 `Option[T]` 라는 type을 제공합니다. 값이 있으면 `Some(value)`, 값이 없으면 `None`을 반환하는 녀석입니다. 여기서 `None`은 `Option[Nothing]`과 동치입니다. 즉, Scala에서 금기시되는 `null` 대신 `None`을 한 번이라도 사용하셨다면, 알게 모르게 `Option`을 사용하고 계셨던 겁니다. 함수형 프로그래밍에 익숙하신 분들은 눈치채셨겠지만 특정 값을 `포장하고, 꺼내는` 방법을 제공하는 `Option[T]`는 Scala에서 제공하는 대표적인 `Monad` 중 하나입니다. 어쨌든 `Option[T]`를 통해 길고, 때로는 반복적인 작업이 될 수 있는 전통적 try-catch statement을 좀 더 우아하고 깔끔하게 해결할 수 있습니다. 또한, 다음과 같이 함수를 작성한다면 받는 쪽에서 반드시 예외 처리를 하게 됩니다:

```scala
def upperString(value: String): Option[String] = {
if (value.isEmpty) None
else Some(value.upper)
}
```

왜냐면 `validateName`의 return type은 `Option[String]` 이므로, 받는 쪽에서는 `get` 혹은 `getOrElse`, `fold`와 같은 함수를 이용해서 해당 값을 안전하게 받는 처리를 해야하기 때문입니다. 이렇게 function signature만 보고도 parameter와 return value를 손쉽게 유추할 수 있으니 아주 편리해집니다.

### Either
위의 예제에서 특정 함수가 예외 상황을 만났을 때 어떻게 기본값을 주는지 살펴보았습니다. caller가 None을 받게 된다면 그에 맞춰 에러를 내든, 기본값으로 serving 하든, 자유도있게 처리해주면 되겠지요. 하지만 좀 더 강력한 sign을 주고 싶으면 어떻게 할까요? 예를 들면, `exception` 자체를 반환한다던가, `exception의 이유`를 반환한다던가, 다른 type을 반환한다던가요. 그럴 땐 Scala의 `Either` 를 사용하면 됩니다. Scala의 Either는 `Either[Left, Right]` 로 표현되며, 주로 `Left`에는 error가, `Right`에는 올바른 값이 들어갑니다.

```scala
def upperString(value: String): Either[String, String] = {
if (value.isEmpty) Left("Value cannot be empty")
else Right(value.upper)
}
```

그리고 받는 쪽에서는 다음과 같이 처리해주면 됩니다. `case match`나 `fold`를 강제함으로서 좀 더 exception handling에 신경쓰게 할 수 있습니다. 혹은, 아예 다른 값을 주는 것도 가능하니 자유롭게 받아서 써도 되고요.

```scala
upperString(inputValue) match {
  case Left(error) => s"Upperstring failed: $error",
  case Right(result) => s"Upperstring succeeded: $result"
}
```

### Try
하지만 Either는 Modern Scala에서 거의 쓰이지 않습니다. Either에 대해서는 여러 논쟁이 많습니다만, Scala 2.10부터 Either의 자리를 `Try`라는 녀석이 대체하고 난 이후부터는 거의 대부분에 상황에서 `Either` 대신 `Try`를 사용할 수 있습니다. 일단 Either가 좋지 않은 이유는 `Monad`가 아니기 때문입니다. 그래서 `flatMap`과 같은 연산도 없고, Scala의 개념인 functional programming과 약간 동떨어진 느낌도 납니다. 반면에 Try는 Either와 사용 방법이 거의 똑같으면서 `Monadic` 입니다. (Monad의 성격을 완전히 만족하지는 않지만 호환은 가능한 성격입니다) Haskell에선 Either가 Monad이기 때문에 Try 같은 개념이 없지만, Scala에선 Try를 통해 Either가 부족한 점을 보완하고 있습니다. 다음과 같이 사용합니다.

```scala
Try {
    upperString(str)
}

match {
    case Success(_) => ...
    case Failure(_) => ...
}
```

거의 Either와 사용법이 똑같죠? 하지만 Try를 쓰면 다음과 같은 문법도 사용할 수 있습니다:

```
Try(upperString).toOption
```

감이 오시나요? Try의 결과를 evaluation 하지 않고 Option으로 감싸서 처리하는 순간을 뒤로 미루는 겁니다. (lazy evaluation 처럼요!) Either의 경우에는 Left, Right에서 명시적으로 evaluation이 일어나기 때문에, 위처럼 사용하기 쉽지 않습니다. 바깥에서 error를 제대로 처리해주지 않는다면 결국 프로그램이 뻗을 수도 있고요. Monadic인 Try를 사용한 아름다운 코드를 하나 공유해봅니다!

```scala
def getURLContent(url: String): Try[Iterator[String]] =
  for {
    url <- parseURL(url)
    connection <- Try(url.openConnection())
    is <- Try(connection.getInputStream)
    source = Source.fromInputStream(is)
  } yield source.getLines()

getURLContent("http://danielwestheide.com/foobar") match {
  case Success(lines) => lines.foreach(println)
  case Failure(ex) => match {
    case e: FileNotFoundException => ...
    case e: MalformedURLException => ...
  }
}
```

위처럼 Try의 결과를 evaluation해서 사용하는 pattern을 `Monad transfomer` 라고 합니다. Monad로 wrapping 되어 있는 값들을 꺼내서 판단하고 다시 바깥으로 반환하는 거죠. map이나 flatMap을 사용할 수도 있지만 for yield를 사용해서 결과도 한 눈에 들어오고, case match도 줄이는 좋은 코드라고 생각해서 공유해봤습니다.

## 결론
> Scala에서 예외 처리를 할 땐 `Option`, 좀 더 강한 처리를 해주고 싶으면 `Try`를 이용하면 된다.

### References
- https://xebia.com/blog/try-option-or-either/
- https://stackoverflow.com/questions/25467760/scalas-either-not-having-flatmap-meaning-of-either-left-right
