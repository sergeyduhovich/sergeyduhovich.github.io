---
layout: single
title: "Connectable операторы"
date: 2019-05-10 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

На сайте [reactivex](http://reactivex.io/documentation/operators.html#connectable) всего 4 оператора в разделе connectable. В первом посте я упомянул, что есть 2 типа Observable: холодный(пассивный) и горячий(активный). По умолчанию все операторы из раздела [creating](http://reactivex.io/documentation/operators.html#creating) возвращают холодный сигнал. Что это значит?

Давайте посмотрим на следующий пример. Я создал таймер, который срабатывает каждую секунду:

```swift
Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background)) // creation
  .debug("interval 1")
  .subscribe() // subscription
  .disposed(by: disposeBag)
```

Непосредственно сам Observable создается оператором `interval`, он ничего не делает до тех пор, пока мы на него не подпишемся. После вызова `subscribe` он начинает отправлять `next` ивенты. Пока все идет по плану, в логе мы видим:


```swift
// interval 1 -> subscribed
// interval 1 -> Event next(0)
// interval 1 -> Event next(1)
// interval 1 -> Event next(2)
// interval 1 -> Event next(3)
// interval 1 -> Event next(4)
```

Теперь представим, что мы хотим переиспользовать тот же таймер и для другой подписки. Возможно код мы обновим до следующего:

```swift
let tickObservable: Observable<Int> = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
    .debug("interval 1")

let strObservable: Observable<String> = tickObservable
  .map { "currently we at \($0) tick" }

tickObservable
  .debug("tickObservable")
  .subscribe()
  .disposed(by: disposeBag)

strObservable
  .debug("strObservable")
  .subscribe()
  .disposed(by: disposeBag)
```

После создания, Observable мы присвоили `tickObservable` переменной. Вторая переменная у нас получилась в результате применения оператора `map`. Дальше мы вызываем `subscribe` у обеих переменных. Смотрим в консоль:

```swift
//1
// tickObservable -> subscribed
// interval 1 -> subscribed
//2
// strObservable -> subscribed
// interval 1 -> subscribed
//3
// interval 1 -> Event next(0)
//4
// tickObservable -> Event next(0)
//5
// interval 1 -> Event next(0)
//6
// strObservable -> Event next(currently we at 0 tick)
//
// interval 1 -> Event next(1)
// interval 1 -> Event next(1)
// strObservable -> Event next(currently we at 1 tick)
// tickObservable -> Event next(1)
// interval 1 -> Event next(2)
// strObservable -> Event next(currently we at 2 tick)
// interval 1 -> Event next(2)
// tickObservable -> Event next(2)
```

### Давайте пройдемся по первым шести комментариям

Для обеих переменных `tickObservable` и `strObservable` мы видим `subscribed`. Каждая из переменных - это результат 2 цепочек `debug`, поэтому мы в том числе мы видим и `interval 1 -> subscribed` для каждой из них.

```swift
tickObservable
  .debug("tickObservable")
  .subscribe()
  .disposed(by: disposeBag)
  
//эквивалентно
Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
  .debug("interval 1")
  .debug("tickObservable")
  .subscribe()
  .disposed(by: disposeBag)
  
//and
strObservable
  .debug("strObservable")
  .subscribe()
  .disposed(by: disposeBag)
  
//эквивалентно
Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
  .debug("interval 1")
  .map { "currently we at \($0) tick" }
  .debug("strObservable")
  .subscribe()
  .disposed(by: disposeBag)
```

3 и 4 комментарий - `next` ивент `tickObservable`, 5 и 6 комментарий `next` ивент `strObservable`.

### В чем проблема?

Секундочку. 2 вывода в консоль для `next` ивентов для каждой переменной `tickObservable` и `strObservable`. Это не совсем то, что я ожидал от кода, написанного ранее. Я хотел получить 1 вывод `interval 1`, 1 вывод `tickObservable` и 1 `strObservable`, т.е. суммарно 3. Вместо этого получил 4. Это означает, что у нас в системе сейчас работает не один расшаренный таймер на 2 переменных, а 2 таймера, по одному на каждую переменную. Каждый из вызовов `subscribe` вызвал создание `interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))`. 

### Решение

В [доках](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/GettingStarted.md#sharing-subscription-and-share-operator) мы можем найти секцию с применением `share()` оператора. Добавим в наш пример данный оператор, и посмотрим. что получится.

```swift
let tickObservable: Observable<Int> = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
    .debug("interval 1")
    .share() // добавляем share() оператор

let strObservable: Observable<String> = tickObservable
  .map { "currently we at \($0) tick" }

tickObservable
  .debug("tickObservable")
  .subscribe()
  .disposed(by: disposeBag)

strObservable
  .debug("strObservable")
  .subscribe()
  .disposed(by: disposeBag)
```

Let's check ou the console:

```swift
// tickObservable -> subscribed
// interval 1 -> subscribed
// strObservable -> subscribed
//
// interval 1 -> Event next(0)
// tickObservable -> Event next(0)
// strObservable -> Event next(currently we at 0 tick)
//
// interval 1 -> Event next(1)
// tickObservable -> Event next(1)
// strObservable -> Event next(currently we at 1 tick)
//
// interval 1 -> Event next(2)
// tickObservable -> Event next(2)
// strObservable -> Event next(currently we at 2 tick)
```

Теперь у нас по 3 вывода в консоль на каждый `next` ивент, один на расшаренный таймер и по одному на `tickObservable` и `strObservable`. Проблема решена.

### Connectable операторы

Теперь давайте взглянем на сами операторы:

* Connect — указывает connectable Observable начинать работу вне зависимости, есть подписчики или нету
* Publish — конвертирует обычный Observable в connectable Observable
* RefCount — конвертирует connectable Observable в обычный Observable, который умеет считать количество подписчиков
* Replay — присылает прошедшие ивенты для только что подписавшихся

Ничего не видно про оператор `share`. Заглянем в файл `ShareReplayScope.swift`

```swift
public func share(replay: Int = 0, scope: SubjectLifetimeScope = .whileConnected) -> Observable<E> {
  switch scope {
  case .forever:
    switch replay {
    case 0: return self.multicast(PublishSubject()).refCount()
    default: return self.multicast(ReplaySubject.create(bufferSize: replay)).refCount()
    }
  case .whileConnected:
    switch replay {
    case 0: return ShareWhileConnected(source: self.asObservable())
    case 1: return ShareReplay1WhileConnected(source: self.asObservable())
    default: return self.multicast(makeSubject: { ReplaySubject.create(bufferSize: replay) }).refCount()
    }
  }
}
```

Итак, `share` это просто обертка для операторов `multicast` + `refCount`. Так же есть 2 оптимизированные верси для `share(replay: 1)` и `share()`.

### Multicast / Publish

Откроем файл `Multicast.swift`.

```swift
public func publish() -> ConnectableObservable<E> {
  return self.multicast { PublishSubject() }
}
```

Publish это multicast с использованием `PublishSubject` в качестве subject. Publish и multicast конвертируют обычный Observable в ConnectableObservable, у которого есть всего один метод:

```swift
public protocol ConnectableObservableType : ObservableType {
  func connect() -> Disposable
}
```

Connectable Observable не будет ничего отправлять своим подписчикам до того момента как будет вызван `connect` или `refCount`.

### Жизненный цикл Observable

Есть 2 разных подхода управлением жизненным циклом Observable в RxSwift.

### Connect 

#### Сценарий 1, S2 подписывается до того как S1 отписывается:

![connect](http://dukhovich.by/assets/images/articles/connect_1.jpg)

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(PublishSubject<Int>())

    _ = connectable.connect()

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 1.5) {
        connectable
          .debug("S1")
          .subscribe()
          .disposed(by: self.disposeBagS1)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 3.5) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 2.5) {
        connectable
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 6.5) {
        self.disposeBagS2 = DisposeBag()
    }
```

в консоли:

```swift
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S1 -> subscribed
//interval 1 -> Event next(1)
//S1 -> Event next(1)
//S2 -> subscribed
//interval 1 -> Event next(2)
//S1 -> Event next(2)
//S2 -> Event next(2)
//S1 -> isDisposed
//interval 1 -> Event next(3)
//S2 -> Event next(3)
//interval 1 -> Event next(4)
//S2 -> Event next(4)
//interval 1 -> Event next(5)
//S2 -> Event next(5)
//S2 -> isDisposed
//interval 1 -> Event next(6)
//interval 1 -> Event next(7)
//interval 1 -> Event next(8)
//interval 1 -> Event next(9)
```

#### Сценарий 2, S2 подписывается после того как S1 отписывается:

![connect](http://dukhovich.by/assets/images/articles/connect_2.jpg)

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(PublishSubject<Int>())

    _ = connectable.connect()

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 1.5) {
        connectable
          .debug("S1")
          .subscribe()
          .disposed(by: self.disposeBagS1)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 3.5) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 4.5) {
        connectable
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 6.5) {
        self.disposeBagS2 = DisposeBag()
    }
```

в консоли:

```swift
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S1 -> subscribed
//interval 1 -> Event next(1)
//S1 -> Event next(1)
//interval 1 -> Event next(2)
//S1 -> Event next(2)
//S1 -> isDisposed
//interval 1 -> Event next(3)
//S2 -> subscribed
//interval 1 -> Event next(4)
//S2 -> Event next(4)
//interval 1 -> Event next(5)
//S2 -> Event next(5)
//S2 -> isDisposed
//interval 1 -> Event next(6)
//interval 1 -> Event next(7)
//interval 1 -> Event next(8)
//interval 1 -> Event next(9)
```

Итак, как мы можем описать поведение оператора `connect`?

* Connect указывает connectable Observable начать доставлять ивенты подписчикам;
* Connectable Observable превращается в горячий сигнал, ему не нужно ждать вызовов `subscribe`, чтобы начать работу;
* Не прекращает работу, даже если количество подписчиков уменьшается до 0;
* Не пересоздает Observable, к которому он был применен, в момент когда появляется первый подписчик;

### RefCount

Давайте посмотрим на поведение `refCount`:

#### Сценарий 1, S2 подписывается до того как S1 отписывается:

![refCount](http://dukhovich.by/assets/images/articles/ref_count_2.jpg)

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(PublishSubject<Int>())

    let refCount = connectable.refCount()

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 1.5) {
        refCount
          .debug("S1")
          .subscribe()
          .disposed(by: self.disposeBagS1)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 4) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 3) {
        refCount
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 7) {
        self.disposeBagS2 = DisposeBag()
    }
```

в консоли:

```swift
//S1 -> subscribed
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S1 -> Event next(0)
//S2 -> subscribed
//interval 1 -> Event next(1)
//S1 -> Event next(1)
//S2 -> Event next(1)
//S1 -> isDisposed
//interval 1 -> Event next(2)
//S2 -> Event next(2)
//interval 1 -> Event next(3)
//S2 -> Event next(3)
//interval 1 -> Event next(4)
//S2 -> Event next(4)
//S2 -> isDisposed
//interval 1 -> isDisposed
```

#### Сценарий 2, S2 подписывается после того как S1 отписывается:

![refCount](http://dukhovich.by/assets/images/articles/ref_count_1.jpg)

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(PublishSubject<Int>())

    let refCount = connectable.refCount()

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 1.5) {
        refCount
          .debug("S1")
          .subscribe()
          .disposed(by: self.disposeBagS1)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 4) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 5.5) {
        refCount
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 8) {
        self.disposeBagS2 = DisposeBag()
    }
```

в консоли:

```swift
//S1 -> subscribed
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S1 -> Event next(0)
//interval 1 -> Event next(1)
//S1 -> Event next(1)
//S1 -> isDisposed
//interval 1 -> isDisposed
//S2 -> subscribed
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S2 -> Event next(0)
//interval 1 -> Event next(1)
//S2 -> Event next(1)
//S2 -> isDisposed
//interval 1 -> isDisposed
```

Итак, как мы можем описать поведение оператора `refCount`?

* Начинает работу только когда появляется первый подписчик, т.е. с первым вызовом `subscribe`
* Прекращает работу, после того, как количество подписчиков уменьшается до 0;
* Создает новый Observable, к которому он был применен, в момент когда появляется первый подписчик;

### Replay

Мы разобрали 3 из 4 операторов в секции connectable.

В примерах выше мы использовали PublishSubject. Чтобы посмотреть как работает `replay`, мы должны передать `ReplaySubject` в качестве параметра для `multicast`.  Replay работает по разному для операторов  `connect` и `refCount`. Давайте посмотрим на примеры.

#### Сценарий 1

Через 2.5 секунды после вызова `connect` и `refCount` добавляем подписчика. В обоих случаях `ReplaySubject` имеет буфер на 2 элемента:

![replay](http://dukhovich.by/assets/images/articles/replay_1.jpg)

`RefCount` ждет подписчиков, и к моменту, когда появляется первый, нет ни одного ивента для replay: 

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(ReplaySubject<Int>.create(bufferSize: 2))

    let refCount = connectable.refCount()

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 2.5) {
        refCount
          .debug("S1")
          .subscribe()
          .disposed(by: self.disposeBagS1)
    }
```

```swift
//S1 -> subscribed
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S1 -> Event next(0)
//interval 1 -> Event next(1)
//S1 -> Event next(1)
//interval 1 -> Event next(2)
//S1 -> Event next(2)
//interval 1 -> Event next(3)
//S1 -> Event next(3)
```

`Connect` не ждет подписчиков, сразу начинает работу. Поскольку у него была фора в 2.5 секунды, у первого подписчика будет 2 ивента в момент подписки:

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(ReplaySubject<Int>.create(bufferSize: 2))

    _ = connectable.connect()

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 2.5) {
        connectable
          .debug("S1")
          .subscribe()
          .disposed(by: self.disposeBagS1)
    }
```

```swift
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//interval 1 -> Event next(1)
//S1 -> subscribed
//S1 -> Event next(0)
//S1 -> Event next(1)
//interval 1 -> Event next(2)
//S1 -> Event next(2)
//interval 1 -> Event next(3)
//S1 -> Event next(3)
//interval 1 -> Event next(4)
//S1 -> Event next(4)
//interval 1 -> Event next(5)
//S1 -> Event next(5)
//interval 1 -> Event next(6)
//S1 -> Event next(6)
```

#### Сценарий 2

Добавляем второго подписчика спустя 3.5 секунды после того как был вызван `connect` и `refCount` и подписан первый подписчик. В обоих случаях `ReplaySubject` имеет буфер на 2 элемента:

![replay](http://dukhovich.by/assets/images/articles/replay_2.jpg)

```swift
let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(ReplaySubject<Int>.create(bufferSize: 2))

    _ = connectable.connect()

    connectable
      .debug("S1")
      .subscribe()
      .disposed(by: self.disposeBagS1)

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 3.5) {
        connectable
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }
```

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(ReplaySubject<Int>.create(bufferSize: 2))

    let refCount = connectable.refCount()

    refCount
      .debug("S1")
      .subscribe()
      .disposed(by: self.disposeBagS1)

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 3.5) {
        refCount
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }
```

Поведение в обоих случаях одинаковое. После подписки сначала приходят 2 прошлых ивента:

```swift
//interval 1 -> subscribed
//S1 -> subscribed
//interval 1 -> Event next(0)
//S1 -> Event next(0)
//interval 1 -> Event next(1)
//S1 -> Event next(1)
//interval 1 -> Event next(2)
//S1 -> Event next(2)
//S2 -> subscribed
//S2 -> Event next(1)
//S2 -> Event next(2)
//interval 1 -> Event next(3)
//S1 -> Event next(3)
//S2 -> Event next(3)
//interval 1 -> Event next(4)
//S1 -> Event next(4)
//S2 -> Event next(4)
//interval 1 -> Event next(5)
//S1 -> Event next(5)
//S2 -> Event next(5)
//interval 1 -> Event next(6)
//S1 -> Event next(6)
//S2 -> Event next(6)
```

#### Сценарий 3

Добавляем второго подписчика спустя 14.5 секунд после того как был вызван `connect` и `refCount`. Первый Observer был отписан после 10-го ивента:

![replay](http://dukhovich.by/assets/images/articles/replay_3.jpg)

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(ReplaySubject<Int>.create(bufferSize: 2))

    _ = connectable.connect()

    connectable
      .debug("S1")
      .subscribe()
      .disposed(by: self.disposeBagS1)

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 11.5) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 14.5) {
        connectable
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }
```

С - стабильность. `Connect` оператор с `replay` во всех 3х случаях вел себя одинаково. В консоли для 3-го сценария:

```swift
//interval 1 -> Event next(7)
//S1 -> Event next(7)
//interval 1 -> Event next(8)
//S1 -> Event next(8)
//interval 1 -> Event next(9)
//S1 -> Event next(9)
//interval 1 -> Event next(10)
//S1 -> Event next(10)
//S1 -> isDisposed
//interval 1 -> Event next(11)
//interval 1 -> Event next(12)
//interval 1 -> Event next(13)
//interval 1 -> Event next(14)
//S2 -> subscribed
//S2 -> Event next(13)
//S2 -> Event next(14)
//interval 1 -> Event next(15)
//S2 -> Event next(15)
//interval 1 -> Event next(16)
//S2 -> Event next(16)
//interval 1 -> Event next(17)
//S2 -> Event next(17)
//interval 1 -> Event next(18)
//S2 -> Event next(18)
//interval 1 -> Event next(19)
//S2 -> Event next(19)
```

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .multicast(ReplaySubject<Int>.create(bufferSize: 2))

    let refCount = connectable.refCount()

    refCount
      .debug("S1")
      .subscribe()
      .disposed(by: self.disposeBagS1)

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 11.5) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 14.5) {
        refCount
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }
```
А теперь давайте обсудим, что происходит с оператором `refCount` в данной ситуации. Когда количество подписчиков уменьшается до нуля, Observable, к которому был применен `multicast` + `refCount` завершается, ресурсы освобождаются, но ReplaySubject продолжает держать 2 последних события. И когда вновь появляется первый подписчик, он сначала получает эти 2 события, потом "свежесозданные". Выглядит довольно интересно - 9, 10, 0, 1, 2, ... :

```swift
//interval 1 -> Event next(7)
//S1 -> Event next(7)
//interval 1 -> Event next(8)
//S1 -> Event next(8)
//interval 1 -> Event next(9)
//S1 -> Event next(9)
//interval 1 -> Event next(10)
//S1 -> Event next(10)
//S1 -> isDisposed
//interval 1 -> isDisposed
//S2 -> subscribed
//S2 -> Event next(9)
//S2 -> Event next(10)
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S2 -> Event next(0)
//interval 1 -> Event next(1)
//S2 -> Event next(1)
//interval 1 -> Event next(2)
//S2 -> Event next(2)
//interval 1 -> Event next(3)
//S2 -> Event next(3)
```

### Share

Мы рассмотрели почти все возможные сценарии для `connect`, `refCount` и `replay`. Теперь давайте посмотрим на аргумент `scope` в методе `share(replay: scope:)`. 

Если мы укажем в первом аргументе 0, т.е. `share(replay: 0)` что эквивалентно `share()`, то второй аргумент никак не повлияет на результат. Второй аргумент меняет поведение метода только при `replay > 0`.

Давайте продублируем последний сценарий-3, но заменим `multicast`.`refCount` на:

#### `.share(replay: 2, scope: .forever)`

что эквивалентно:

```swift
self.multicast(ReplaySubject.create(bufferSize: replay)).refCount()
```

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .share(replay: 2, scope: .forever)

    connectable
      .debug("S1")
      .subscribe()
      .disposed(by: self.disposeBagS1)

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 11.5) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 14.5) {
        connectable
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }
```

Вывод в консоли такой же как в сценарии-3:

```swift
//interval 1 -> Event next(7)
//S1 -> Event next(7)
//interval 1 -> Event next(8)
//S1 -> Event next(8)
//interval 1 -> Event next(9)
//S1 -> Event next(9)
//interval 1 -> Event next(10)
//S1 -> Event next(10)
//S1 -> isDisposed
//interval 1 -> isDisposed
//S2 -> subscribed
//S2 -> Event next(9)
//S2 -> Event next(10)
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S2 -> Event next(0)
//interval 1 -> Event next(1)
//S2 -> Event next(1)
//interval 1 -> Event next(2)
//S2 -> Event next(2)
//interval 1 -> Event next(3)
//S2 -> Event next(3)
```

#### `.share(replay: 2, scope: .whileConnected)` 

что эквивалентно:

```swift
self.multicast(makeSubject: { ReplaySubject.create(bufferSize: replay) }).refCount()
```

```swift
    let connectable = Observable<Int>.interval(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
      .debug("interval 1")
      .share(replay: 2, scope: .whileConnected)

    connectable
      .debug("S1")
      .subscribe()
      .disposed(by: self.disposeBagS1)

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 11.5) {
        self.disposeBagS1 = DisposeBag()
    }

    DispatchQueue.main
      .asyncAfter(deadline: DispatchTime.now() + 14.5) {
        connectable
          .debug("S2")
          .subscribe()
          .disposed(by: self.disposeBagS2)
    }
```

```swift
//interval 1 -> Event next(7)
//S1 -> Event next(7)
//interval 1 -> Event next(8)
//S1 -> Event next(8)
//interval 1 -> Event next(9)
//S1 -> Event next(9)
//interval 1 -> Event next(10)
//S1 -> Event next(10)
//S1 -> isDisposed
//interval 1 -> isDisposed
//S2 -> subscribed
//interval 1 -> subscribed
//interval 1 -> Event next(0)
//S2 -> Event next(0)
//interval 1 -> Event next(1)
//S2 -> Event next(1)
//interval 1 -> Event next(2)
//S2 -> Event next(2)
```

Я предпочитаю эту связку `multicast(makeSubject: { ReplaySubject.create(bufferSize: replay) }).refCount()` если сравнивать ее с `multicast(ReplaySubject.create(bufferSize: replay)).refCount()`. Для меня это кажется более уместным в большинстве случаев.

### Заключение

Кому-то может показаться, что мы разбирали какие-то совсем надуманные сценарии, никак не связанные с реальным использованием RxSwift в MVVM-подобных архитектурах. В свое оправдание могу сказать, что знание теории позволяет избегать ошибок на практике. 

Теперь давайте посмотрим на примеры / ошибки, которые встречаются в проектах:

```swift
    let hotels: Observable<[Hotel]> = apiClient.hotels()
    let count = hotels.map { "We've found \($0.count) hotels" }
    let rating = hotels.map { "Average rating is \($0.reduce(0.0, { $0 + $1.rating }) / Float($0.count))" }

    count
      .bind(to: countLabel.rx.text)
      .disposed(by: disposeBag)

    rating
      .bind(to: averageRatingLabel.rx.text)
    .disposed(by: disposeBag)
```

Мы не знаем природу Observable `hotels()`, но по контексту можно предположить, что он холодный (метод некого apiClient) и данный код содержит ошибку. В сеть уйдет 2 запроса вместо одного. Чтобы починить код, нужно добавить `share()`:

```swift
    let hotels: Observable<[Hotel]> = apiClient.hotels().share()
    let count = hotels.map { "We've found \($0.count) hotels" }
    let rating = hotels.map { "Average rating is \($0.reduce(0.0, { $0 + $1.rating }) / Float($0.count))" }

    count
      .bind(to: countLabel.rx.text)
      .disposed(by: disposeBag)

    rating
      .bind(to: averageRatingLabel.rx.text)
    .disposed(by: disposeBag)
```

Встречался и такой случай с излишней подстраховкой операторами `share`:

```swift
  let hotelsBehavior = BehaviorRelay<[Hotel]>(value: [])
  lazy var hotelsObservable = hotelsBehavior.asObservable()

  lazy var hotelsCount = hotelsObservable
    .share()
    .map { $0.count }

  lazy var hotelsFound = hotelsObservable
    .share()
    .map { "We've found \($0.count) hotels" }

  lazy var averageRating = hotelsObservable
    .share()
    .map { $0.reduce(0.0, { $0 + $1.rating }) / Float($0.count) }
```

В первом случае мы не знали `apiClient.hotels()` горячий или холодный, но здесь мы видим что `hotelsObservable` это `BehaviorRelay`. Даже не применяя оператор `share` мы не будем выделять лишних ресурсов на 3 подписчиков.

Но если мы, как в данном примере, будем использовать оператор `share` перед каждой трансформацией в новый Observable, мы будем проделывать лишнюю работу, давай посмотрим на эквивалент:

```swift
    hotelsCount
      .subscribe(onNext: { count in
        print(count)
      })
      .disposed(by: disposeBag)
      
    //тоже самое что
    let hotelsCountSubject = PublishSubject<[Hotel]>()

    hotelsBehavior
      .bind(to: hotelsCountSubject)
      .disposed(by: disposeBag)

    hotelsCountSubject
      .map { $0.count }
      .subscribe(onNext: { count in
        print(count)
      })
      .disposed(by: disposeBag)
```

Похоже на overhead. Чтобы починить код, удаляем лишние вызовы `share`.