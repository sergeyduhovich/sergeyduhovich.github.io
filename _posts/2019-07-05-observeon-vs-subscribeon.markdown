---
layout: single
title: "Чем отличается observeOn от subscribeOn"
date: 2019-07-05 12:00:00 +0300
categories: rxswift
---

По умолчанию, Observable и все последующие за ним операторы преобразования выполняют свою работу и уведомляют подписчиков на той очереди, на которой был вызван метод `subsribe`.

Выражение выше является частью контракта [ReactiveX](http://reactivex.io/documentation/scheduler.html).

Т.е. если мы настраиваем биндинги в методе `viewDidLoad`, абсолютно все будет выполняться на Main очереди, и кложура создания Observable и доставка ивентов всем observer-ам.

### Subscribe на Main очереди

```swift
override func viewDidLoad() {
    super.viewDidLoad()
	
	Observable<Int>
	  .create { observer in
	    for i in 0..<20 {
	      sleep(1)
	      observer.onNext(i)
	    }
	    observer.onCompleted()
	    return Disposables.create()
	  }
	  .filter { num -> Bool in
	    return num % 2 == 0
	  }
	  .skipWhile { num -> Bool in
	    return num < 5
	  }
	  .map { num -> String in
	    return "current iteration is \(num)"
	  }
	  .bind(to: label.rx.text)
	  .disposed(by: disposeBag)
}
```

Поскольку в примере выше, метод `bind` (он же `subscribe`) был вызван на Main очереди, то, поставив брейкпойнты, в дебаг навигаторе увидим, что абсолютно все выполняется на Main очереди. А с учетом того, что в кложуре создания есть `sleep(1)`, тем самым мы еще и блокируем Main очередь.

Предлагаю протестировать реализацию контракта ReactiveX в RxSwift. Будут ли доставляться уведомления подписчикам в той же очереди, что и вызов `subscribe`, если `observer.onNext(i)` я поменяю на:

```swift
DispatchQueue.global().async {
	observer.onNext(i)
}
```

Все кложуры начали приходить на глобальной очереди, а не на Main, в котором был вызван метод `subscribe`.

### Subscribe на глобальной очереди

Теперь немного видоизменим пример, что бы метод подписки был вызван на глобальной очереди:

```swift
  override func viewDidLoad() {
    super.viewDidLoad()

    let observable = Observable<Int>
      .create { observer in
        for i in 0..<20 {
          sleep(1)
          observer.onNext(i)
        }
        observer.onCompleted()
        return Disposables.create()
      }
      .filter { num -> Bool in
        return num % 2 == 0
      }
      .skipWhile { num -> Bool in
        return num < 5
      }
      .map { num -> String in
        return "current iteration is \(num)"
    }

    DispatchQueue.global().async {
      observable
        .bind(to: self.label.rx.text)
        .disposed(by: self.disposeBag)
    }
  }
```

Observable переменная была "сконструирована" на Main очереди, а подписка на нее была оформлена на глобальной очереди. Все кложуры теперь вызываются на глобальной очереди (в т.ч. и обновление UI, что не очень хорошо). Контракт ReactiveX сохраняется.

Повторим эксперимент с заменой очереди для `observer.onNext(i)`, на этот раз отправим `onNext` в Main очереди:

```swift
DispatchQueue.main.async {
	observer.onNext(i)
}
```

Теперь все кложуры начали приходить на Main очереди, а не на глобальной, в которой был вызван метод `subscribe`.

Эти маневры с Dispatch вряд ли можно отнести к хорошему стилю использования RxSwift, но тем не менее, они дают базовое представление о том по какому принципу выбирается очередь, когда начинается проброс ивентов от первого к последующему оператору.

## observeOn оператор 

Этот оператор указывает шедулер (можно читать как очередь в этом контексте), в котором будут доставляться ивенты observer-ам в операторах, которые идут **ПОСЛЕ** его применения. 

Доставка/проброс ивента в операторе `map`:

```swift
.map { num -> String in
	//кложура
	return "current iteration is \(num)" 
}
```

### 1

Давайте посмотрим на следующий псевдокод:

```swift
observable
	.operator1()
	.operator2()
	.operator3()
	.operator4()
	.observeOnBackground()
	.operator5()
	.operator6()
	.operator7()
	.operator8()
	.subscribe()
```

Если предположить, что `subscribe` вызван на Main очереди, то не добавь мы `observeOnBackground`, абсолютно все кложуры операторов отработали бы на Main очереди. Но в примере выше, начиная с оператора 5, ивенты приходят на Background очереди.

### 2

Следующий пример:

```swift
observable
	.observeOnDefault()
	.operator1()
	.operator2()
	.operator3()
	.operator4()
	.observeOnBackground()
	.operator5()
	.operator6()
	.operator7()
	.observeOnMain()
	.operator8()
	.subscribe()
```

`observeOnDefault` указывает всем операторам после него, что ивенты будут приходить на глобальной очереди (QoS `.default`). Однако эта инструкция прерывается на операторе 4, т.к. перед оператором 5 присутствует новая инструкция, которая указывает, что последующие кложуры должны приходить на background очереди. Оператор 8 и финальная подписка выполнятся на Main очереди.

### 3

И чисто теоретический пример, вряд ли такое можно встретить в реальном проекте:

```swift
observable
	.operator1()
	.operator2()
	.operator3()
	.operator4()
	.operator5()
	.operator6()
	.operator7()
	.operator8()
	.observeOnBackground()
	.observeOnDefault()
	.observeOnMain()
	.subscribe()
```

На какой очереди будет вызвана финальная кложура `subscribe`? Ответ - на Main очереди. В такой странной цепочке `observeOn` операторов результирующим будет последний оператор, т.к. он отменяет инструкции предшествующих ему. 

## subscribeOn оператор

Этот оператор указывает шедулер (можно читать как очередь в этом контексте), в котором будет отрабатывать код создания операторов, которые идут **ПЕРЕД** ним.

О чем здесь идет речь, какой еще код создания операторов? В статье [про subscribe]({{ site.baseurl }}{% post_url 2019-05-31-what-subscribe-does %}) я рассказывал, что в момент подписки, объекты-producer-ы начинают создавать sink-объекты. Так вот, процесс инициализации операторов может быть оптимизирован и вынесен на другую очередь. Это же верно и для операторов из [категории create]({{ site.baseurl }}{% post_url 2019-05-03-creating-an-observable %}), но там немного иная схема.

Для операторов преобразования `subscribeOn` меняет очередь, на которой вызывается метод `run` у producer-части этого оператора. Для операторов создания `subscribeOn` указывает очередь, в которой вызывается кложура с `observer.onNext`.

```swift
 override func viewDidLoad() {
    super.viewDidLoad()

    Observable<Int>
      .create { observer in
        for i in 0..<20 {
          sleep(1)
          observer.onNext(i)
        }
        observer.onCompleted()
        return Disposables.create()
      }
      .filter { num -> Bool in
        return num % 2 == 0
      }
      .skipWhile { num -> Bool in
        return num < 5
      }
      .map { num -> String in
        return "current iteration is \(num)"
      }
      .subscribeOn(ConcurrentDispatchQueueScheduler(qos: .default))
      .bind(to: label.rx.text)
      .disposed(by: self.disposeBag)
}
```

В примере выше `subscribeOn` указывает producer-у `map` на какой очереди будет вызван метод `run`, который выглядит следующим образом:

```swift
let sink = MapSink(transform: self._transform, observer: observer, cancel: cancel)
let subscription = self._source.subscribe(sink)
return (sink: sink, subscription: subscription)
```

`subscribeOn` отразится не только на ближайшем операторе `map`, он повлияет на все вышестоящие операторы, т.е. у них метод `run` тоже будет вызван на глобальной очереди (QoS `.default`).

### 1 

И снова псевдокод:

```swift
observable
	.operator1()
	.operator2()
	.operator3()
	.operator4()
	.subscribeOnBackground()
	.operator5()
	.operator6()
	.operator7()
	.operator8()
	.subscribe()
```

В этом случае `subscribeOnBackground` укажет `operator1`-`operator4`  изменить очередь, где будет вызван метод `run`, а блоку `observable` укажет очередь, на которой должен вызваться `observer.onNext`.

### 2

```swift
observable
	.subscribeOnBackground()
	.operator1()
	.operator2()
	.operator3()
	.operator4()
	.subscribeOnGlobal()
	.operator5()
	.operator6()
	.operator7()
	.operator8()
	.subscribe()
```

В этом случае на background очереди будет вызван только `observer.onNext` в кложуре `observable`. У операторов с 1го по 4й метод `run` вызовется на глобальной очереди. У операторов с 5го по 8й метод `run` будет вызван на той же очереди, где будет выполняться непосредственно сама подписка, если предположим, что она в методе `viewDidLoad` - то это Main очередь.

### 3

```swift
observable
	.subscribeOnBackground()
	.subscribeOnDefault()
	.subscribeOnMain()
	.operator1()
	.operator2()
	.operator3()
	.operator4()
	.operator5()
	.operator6()
	.operator7()
	.operator8()
	.subscribe()
```

На какой очереди будет вызвана кложура `observable `? Ответ - на background очереди. В такой странной цепочке `subscribeOn` операторов результирующим будет первый оператор. 

### ObserveOn vs SubscribeOn

* `observeOn` применяется к последующим операторам;
* `subscribeOn` применяется к предшествующим операторам;
* `observeOn` влияет на метод `func on(_:)` Sink-объекта;
* `subscribeOn` влияет на метод `func run(_:cancel:)` Producer-объекта;

Статья на эту же тему [в блоге Marin](http://rx-marin.com/post/observeon-vs-subscribeon/).