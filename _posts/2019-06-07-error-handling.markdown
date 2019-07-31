---
layout: single
title: "Обработка Ошибок"
date: 2019-06-07 12:00:00 +0300
categories: rxswift
---

**Нормальное завершение** Observable последовательности происходит в момент отправки ею **Completed** или **Error** ивента. После того как любой из них был получен, Observable не может ничего больше отправлять, также освобождаются все ресурсы, выделенные под Observable.

Абзац выше очень важен и является фундаментальной частью контракта ReactiveX. Но когда мы начинаем описывать цепочку, мы порой забываем про эту особенность. И последовательность, которая на первый взгляд должна работать без нареканий, на самом деле может привести к багам в некоторых исключительных ситуациях.

```swift
class ViewController: UIViewController {
	let apiCallObservable: Observable<Int>
	//...
	override func viewDidLoad() {
		super.viewDidLoad()
		
	    apiCallObservable
		  .map { "We've found \($0) hotels for you" }
		  .bind(to: label.rx.text)
		  .disposed(by: disposeBag)
	}
}
```

Код выше описывает некий Observable, предположим API-запрос, который возвращает числовое значение, с последующим преобразованием и биндингом в `text` свойство некоторого UILabel. Что если вместо ответа мы получим ошибку, например отсутствие интернета? 

Ответ прост: Observable тут же завершится, все ресурсы освободятся, а наш UILabel не будет обновлен.

Для обработки ошибок я использую следующие операторы, какой именно оператор применить, обычно зависит от контекста:

* `materialize` и `dematerialize`;
* `catch`;
* `retry`;
* `retryWhen`;

В остальной части поста я буду использовать `chatObservable`, который может прислать ошибку:

```swift
let chatObservable: Observable<String> = Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)
  .flatMap { num -> Observable<String> in
    if Int.random(in: 0..<3) == 0 {
      return .error(DataError.testError)
    } else {
      return .just("message \(num)")
    }
}
```

Давайте для него попробуем корректно обработать ошибки.

## 1. Внутренний Observable присылает error

Пожалуй, самый простой случай - это когда исходный Observable никогда не присылает ошибку, например клики на кнопку ([ControlEvent trait]({{ site.baseurl }}{% post_url 2019-05-17-traits %})), а вложенный оператором `flatMap` Observable может завершиться с ошибкой. Т.е. на каждый клик, создается новый Observable. Одной из особенностей `flatMap` является тот факт, что "наверх" пробрасываются как `next` так и `error` ивенты. А вот `completed` ивент не пробрасывается. Эту особенность можно использовать в обработке ошибок.

### Проблема

Давайте посомтрим сначала на Observable, который не содержит никаких обработок:

```swift
button.rx.tap.debug("source")
  .flatMapLatest { _ -> Observable<String> in
    return chatObservable.debug("inner")
  }
  .debug("result")
  .bind(to: label.rx.text)
  .disposed(by: disposeBag)
```

В консоли (в зависимости от `Int.random(in: 0..<3)`) мы время от времени получим:

```swift
result -> subscribed
source -> subscribed
//after button click
source -> Event next(())
inner -> subscribed
inner -> Event next(message 0)
result -> Event next(message 0)
inner -> Event next(message 1)
result -> Event next(message 1)
inner -> Event next(message 2)
result -> Event next(message 2)
inner -> Event next(message 3)
result -> Event next(message 3)
inner -> Event error(testError)
result -> Event error(testError)
result -> isDisposed
source -> isDisposed
inner -> isDisposed
```

Как видим, не обработав Observable, и `result` и `source` подписки завершились, освободив ресурсы. Другими словами, нажатие на кнопку больше ничего не делает.

### `catchError`

Я очень часто использую такой подход для обработки ошибки используя `catchError`:

```swift
button.rx.tap.debug("source")
  .flatMapLatest { _ -> Observable<String> in
    return chatObservable.debug("inner")
      .catchError { [weak self] error -> Observable<String> in
        self?.label.text = "something went wrong: \(error)"
        return .empty()
      }
  }
  .debug("result")
  .bind(to: label.rx.text)
  .disposed(by: disposeBag)
```

Observable даже после получения ошибок продолжает работать (точнее внутренний Observable всегда создается заново, обнуляя таймер в нашем случае). Ошибки обработаны, клики работают как ожидается.

```swift
result -> subscribed
source -> subscribed
//after button click
source -> Event next(())
inner -> subscribed
inner -> Event next(message 0)
result -> Event next(message 0)
inner -> Event next(message 1)
result -> Event next(message 1)
inner -> Event next(message 2)
result -> Event next(message 2)
inner -> Event error(testError)
inner -> isDisposed
//result and source подписки все еще актуальны
```

![error handling in rxswift 1](http://dukhovich.by/assets/images/articles/error_handling_rxswift.gif)

### `materialize` с последующим `dematerialize`

Такую обработку можно сделать, но с использованием [пары операторов](http://reactivex.io/documentation/operators/materialize-dematerialize.html) `materialize` and `dematerialize`. `error` и `completed` начнут приходить в виде `next` ивентов, например так - `.next(Event.error(testError))`. Но это не означает, что исходный Observable продолжит работать после ошибки, он все равно завершится. 

Пару `materialize` + `dematerialize` можно использовать во внутренних Observable вместо `catchError`. Код будет выглядеть следующим образом:

```swift
button.rx.tap.debug("source")
  .flatMapLatest { _ -> Observable<String> in
    return chatObservable.debug("inner")
      .materialize()
      .map { [weak self] event -> Event<String> in
        if case let .error(err) = event {
          self?.label.text = "something went wrong: \(error)"
          return .completed
        } else {
          return event
        }
      }
      .dematerialize()
  }
  .debug("result")
  .bind(to: label.rx.text)
  .disposed(by: disposeBag)
```

## 2. Исходный Observable присылает error

### `retry`

Единственная опция возобновить Observable без потери подписки - это оператор `retry`. В RxSwift есть 2 реализации: с ограничением - `retry(_ maxAttemptCount: Int)` и без ограничений - `retry()`. Несмотря на то, что они могут помочь, эти операторы несут потенциальную опасность. Например, если мы применим бесконечный `retry()` после API запроса, то в случае, когда пропадет соединение с интернетом, мы до бесконечности будем пытаться пересоздать `Sink`, подписаться на него и переотправить запрос на сервер. 

По правде говоря, мне не очень нравится стандартная реализация этого оператора. Чуть более гибкое решение можно найти в [RxSwiftExt репозитории](https://github.com/RxSwiftCommunity/RxSwiftExt#retry), эта версия поддерживает параметры `exponentialDelayed` и `customTimerDelayed`.

### `retryWhen`

`retryWhen` из стандартного набора RxSwift очень гибкий в плане обработки ошибки. В качестве внутренней последовательности можно вернуть Observable, который будет чем-то вроде ожидания взаимодействия с пользователем:

```swift
chatObservable
  .debug("chatObservable")
  .retryWhen { [weak self] error -> Observable<Void> in
    return error.flatMapLatest { error -> Observable<Void> in
      guard let self = self else { return .empty() }
      self.label.text = "something went wrong: \(error)"
      return self.button.rx.tap.asObservable()
    }
  }
  .debug("result")
  .bind(to: label.rx.text)
  .disposed(by: disposeBag)
```

## RxCocoa

Работая с UI мы чаще всего используем [Driver и Signal]({{ site.baseurl }}{% post_url 2019-05-17-traits %}). Эти traits обычно создаются из Observable, который может завершиться с ошибкой, и метод для преобразования под капотом использует `catch`.

### Driver:

```swift
public func asDriver(onErrorJustReturn: E) -> Driver<E>
public func asDriver(onErrorDriveWith: Driver<E>) -> Driver<E>
public func asDriver(onErrorRecover: @escaping (_ error: Swift.Error) -> Driver<E>) -> Driver<E>
```

### Signal:

```swift
public func asSignal(onErrorJustReturn: E) -> Signal<E>
public func asSignal(onErrorSignalWith: Signal<E>) -> Signal<E>
public func asSignal(onErrorRecover: @escaping (_ error: Swift.Error) -> Signal<E>) -> Signal<E>
```

## RxSwiftExt

Как уже было упомянуто выше, в [RxSwiftExt](https://github.com/RxSwiftCommunity/RxSwiftExt) репозитории можно найти несколько полезных операторов по теме:

* [retry](https://github.com/RxSwiftCommunity/RxSwiftExt#retry)
* [errors, elements](https://github.com/RxSwiftCommunity/RxSwiftExt#errors-elements)
* [catcherrorjustcomplete](https://github.com/RxSwiftCommunity/RxSwiftExt#catcherrorjustcomplete)

## Итоги

Обработку ошибок можно разделить на **Catch** и **Retry**.

Для сохранения исходной подписки и возобновления Observable подходит только `retry`-группа операторов. 

Для Observable, который используется внутри `flatMap` мы можем подключить группу операторов `catch`.