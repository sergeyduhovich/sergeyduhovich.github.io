---
layout: single
title: "Что прячется за subscribe"
date: 2019-05-31 12:00:00 +0300
categories: rxswift
---

Не имеет особого значения то, как много операторов будет использовано в описании цепочки Observable. Если в конце цепочки нет вызова `subscribe`, ресурсы не будут выделены, работа не будет сделана. Не важно, таймер это, URL-запрос, или биндинг вью-моделей на таблицу. Почему `subscribe` так влияет на Observable и почему результат подписки нужно дополнительно обработать, обычно `.disposed(by: disposeBag)`? 

Давайте начнем разбор с класса `DisposeBag`:

```swift
//DisposeBag.swift (simplified)
public final class DisposeBag: DisposeBase {
	fileprivate var _disposables = [Disposable]()
	
	public func insert(_ disposable: Disposable) {
		self._disposables.append(disposable)
	}
	    
	deinit {
	    for disposable in _disposables {
	        disposable.dispose()
	    }
	}
}
```

Это всего лишь контейнер для объектов, реализующих протокол `Disposable`. Его задача - хранение сильных ссылок, чтобы предотвратить удаление объектов. Как только контейнер удаляется, у всех объектов, хранившихся в контейнере, вызывается метод `dispose()`. Кстати, у протокола `Disposable` кроме этого метода больше ничего и нет.

Связь между `DisposeBag` и `Disposable` более-менее понятна. Дальше нам предстоит уточнить, что из себя предоставляют реализации протокола `Disposable`. Начнем мы распутывать клубок начав с метода `subscribe`. Именно он возвращает объекты данного типа, значит где-то в этом методе они должны и создаваться:

```swift
//Observable.swift
public class Observable<Element> : ObservableType {
	public func subscribe<O: ObserverType>(_ observer: O) -> Disposable where O.E == E {
		rxAbstractMethod()
	}
}

//Rx.swift
func rxAbstractMethod(file: StaticString = #file, line: UInt = #line) -> Swift.Never {
	rxFatalError("Abstract method", file: file, line: line)
}
```

Базовая реализация метода нам ничем не помогла. Здесь мы видим, что данный метод является обязательным для классов-наследников `ObservableType`. Что-ж, давайте искать наследников.

### Observable

Пройдясь по классам, которые переопределяют метод `subscribe` я попытался составить UML-диаграмму для них. Для некоторых классов и протоколов я попытался выстроить цепочку вплоть до корневого класса или протокола, для прояснения картины:

![img](http://uploads.dukhovich.by/articles/subscribe_diagram.png)

Как оказалось, довольно много классов переопределяют данный метод. Какие-то из этих классов являются частью RxSwift, какие-то RxCocoa и RxRelay. Но поиск изначально задумывался по классам-наследникам `Observable`. Их-то мы и попытаемся оценить. Сходу можно выделить 3 группы наследников: Producer-класс и его наследники, Subject-ы, и классы, которые участвуют в [трансформации холодных сигналов в горячие](http://dukhovich.by/ru/15-connectable-observable). Пожалуй, правильным решением будет разбор простейших случаев подписки у `Producer` и его наследников, т.к. там реализуется простейший тип подписки 1 к 1 без дополнительного шаринга ресурсов подписки.

Сокращеный файл `Producer.swift`:

```swift
class Producer<Element> : Observable<Element> {
    override func subscribe<O : ObserverType>(_ observer: O) -> Disposable where O.E == Element {
        let disposer = SinkDisposer()
        let sinkAndSubscription = self.run(observer, cancel: disposer)
        disposer.setSinkAndSubscription(sink: sinkAndSubscription.sink, subscription: sinkAndSubscription.subscription)
        return disposer
    }

    func run<O : ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == Element {
        rxAbstractMethod()
    }
}
```

### Subscribe

Давайте разберем что происходит в методе `subscribe` построчно:

1. Создается сущность класса `SinkDisposer`;
1. `disposer` вместе с `observer` передаются в метод `run`, откуда возвращается 2 других объекта - `sink` и `subscription`, оба реализуют `Disposable`;
1. Оба объекта передаются в метод `setSinkAndSubscription` свежесозданного `disposer`;
1. Из метода возвращается `disposer`;

#### SinkDisposer

`SinkDisposer` реализует протокол `Cancelable`, который в свою очередь реализует `Disposable`:

```swift
fileprivate final class SinkDisposer: Cancelable {
    private var _sink: Disposable?
    private var _subscription: Disposable?

    func setSinkAndSubscription(sink: Disposable, subscription: Disposable) {
        self._sink = sink
        self._subscription = subscription
        //...
    }
    
    func dispose() {
		//...
	    sink.dispose()
	    subscription.dispose()
	
	    self._sink = nil
	    self._subscription = nil
	}
}
```

У метода `setSinkAndSubscription` одна задача, сохранить сильные ссылки на объекты `sink` и `subscription`. Именно с этим классом - `SinkDisposer` мы работаем всякий раз, когда ипользуем результат выполнения `subscribe`, вызывая `dispose()` или `.disposed(by: disposeBag)`.

#### Run

Что бы понять, что происходит в методе `run`, придется заглянуть в классы-наследники `Producer`. Абсолютно все разбирать мы не будем, думаю будет вполне достаточно посмотреть цепочку из 3 операторов. Я выберу `interval`, `filter` и `skip` для примера:

```swift
let eventHandler = { (event: Event<Int>) -> Void in
  print(event)
  }

Observable<Int>.interval(1, scheduler: MainScheduler.instance)
  .filter { $0 % 2 == 0 }
  .skip(2)
  .subscribe(eventHandler)
  .disposed(by: disposeBag)
```

Запись немного необычна, я специально выделил замыкание `eventHandler`, что бы подчеркнуть, что именно в нашем случае является observer-ом. И если бы я добавил `.debug` после каждого оператора, то в консоли я бы увидел следующее:

```swift
//interval -> Event next(0)
//filter -> Event next(0)
//interval -> Event next(1)
//interval -> Event next(2)
//filter -> Event next(2)
//interval -> Event next(3)
//interval -> Event next(4)
//filter -> Event next(4)
//skip -> Event next(4)
//interval -> Event next(5)
//interval -> Event next(6)
//filter -> Event next(6)
//skip -> Event next(6)
//interval -> Event next(7)
//interval -> Event next(8)
//filter -> Event next(8)
//skip -> Event next(8)
```

Просмотрев файлы реализаций этих 3 операторов я заметил один и тот же подход. Есть наследник Producer, а есть еще непосредственно класс, содержащий логику оператора - `Sink`. Обычно Producer создает sink, когда вызывается метод `run`. У наследника Producer может быть сколько угодно дополнительных аргументов, что бы полноценно проинициализировать Sink.

Ок. Дальше мы будем разбирать эти пары классов:

* `Timer`+`TimerOneOffSink`;
* `Filter`+`FilterSink`;
* `SkipCount`+`SkipCountSink`;

Начнем с `Timer`+`TimerOneOffSink`:

```swift
//Timer.swift
final private class Timer<E: RxAbstractInteger>: Producer<E> {
	override func run<O: ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == E {
		let sink = TimerOneOffSink(parent: self, observer: observer, cancel: cancel)
	    let subscription = sink.run()
	    return (sink: sink, subscription: subscription)
	}
}

final private class TimerOneOffSink<O: ObserverType>: Sink<O> where O.E: RxAbstractInteger {
    func run() -> Disposable {
        return self._parent._scheduler.scheduleRelative(self, dueTime: self._parent._dueTime) { [unowned self] _ -> Disposable in
            self.forwardOn(.next(0))
            self.forwardOn(.completed)
            self.dispose()

            return Disposables.create()
        }
    }
}

//DispatchQueueConfiguration.swift
func scheduleRelative<StateType>(_ state: StateType, dueTime: Foundation.TimeInterval, action: @escaping (StateType) -> Disposable) -> Disposable {
    let deadline = DispatchTime.now() + dispatchInterval(dueTime)

    let compositeDisposable = CompositeDisposable()

    let timer = DispatchSource.makeTimerSource(queue: self.queue)
    timer.schedule(deadline: deadline, leeway: self.leeway)

    timer.setEventHandler(handler: {
        if compositeDisposable.isDisposed {
            return
        }
        _ = compositeDisposable.insert(action(state))
        cancelTimer.dispose()
    })
    timer.resume()

    _ = compositeDisposable.insert(cancelTimer)

    return compositeDisposable
}
```

`Timer`-Producer создает `TimerOneOffSink`, который в свою очередь создает `DispatchSourceTimer` для выполнения работы. И чем больше раз у `TimerOneOffSink` будет вызван метод `subscribe`, тем больше таймеров будет создано. Я рассказывал об этом в посте про [холодные/горячие сигналы](http://dukhovich.by/ru/15-connectable-observable).

Следующая пара классов - `Filter`+`FilterSink`:

```swift
//Filter.swift
final private class Filter<Element>: Producer<Element> {
    override func run<O: ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == Element {
        let sink = FilterSink(predicate: self._predicate, observer: observer, cancel: cancel)
        let subscription = self._source.subscribe(sink)
        return (sink: sink, subscription: subscription)
    }
}

final private class FilterSink<O: ObserverType>: Sink<O>, ObserverType {   
    func on(_ event: Event<Element>) {
        switch event {
        case .next(let value):
            do {
                let satisfies = try self._predicate(value)
                if satisfies {
                    self.forwardOn(.next(value))
                }
            }
            catch let e {
                self.forwardOn(.error(e))
                self.dispose()
            }
        case .completed, .error:
            self.forwardOn(event)
            self.dispose()
        }
    }
}
```

Producer-ская часть оператора `filter` тоже создает Sink сущность. `FilterSink` ничего не создает, он пробрасывает ивенты. Т.к. он реализует протокол `ObserverType`, который требует реализации `func on(_ event: Event<E>)` он может выступать в качестве аргумента подписки у вышестоящего источника `self._source.subscribe(sink)`. Всякий раз, когда источник получает ивент, он обрабатывается в вышеуказанно методе, дальше в зависимости от того, выполнилось ли условие, ивент будет проброшен по цепочке.

И последний оператор на сегодня - `SkipCount`+`SkipCountSink`:

```swift
//Skip.swift
final private class SkipCount<Element>: Producer<Element> {
    override func run<O : ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == Element {
        let sink = SkipCountSink(parent: self, observer: observer, cancel: cancel)
        let subscription = self.source.subscribe(sink)

        return (sink: sink, subscription: subscription)
    }
}

final private class SkipCountSink<O: ObserverType>: Sink<O>, ObserverType {
    func on(_ event: Event<Element>) {
        switch event {
        case .next(let value):
            
            if self.remaining <= 0 {
                self.forwardOn(.next(value))
            }
            else {
                self.remaining -= 1
            }
        case .error:
            self.forwardOn(event)
            self.dispose()
        case .completed:
            self.forwardOn(event)
            self.dispose()
        }
    }
}
```

`SkipCount` создает `SkipCountSink` в методе `run`. Работает по принципу проброса ивентов, в реализации можно найти и саму логику `if self.remaining <= 0 { self.forwardOn(.next(value)) }`.

### Подведем итоги

Еще раз посмотрим на нашу цепочку:

```swift
let eventHandler = { (event: Event<Int>) -> Void in
  print(event)
  }

Observable<Int>.interval(1, scheduler: MainScheduler.instance)	//3
  .filter { $0 % 2 == 0 }	//2
  .skip(2)	//1
  .subscribe(eventHandler)
  .disposed(by: disposeBag)
```

Используя отступы, я попробую воссоздать очередность вызова методов и уровни стека. Когда мы вызываем `subscribe` после последнего оператора `skip`:

* создается сущность`SinkDisposer`(1);
* у оператора `SkipCount` вызывается `run` метод;
* создается sink(1) класса `SkipCountSink`;
* `self.source.subscribe(sink)` вызывает следующее: 
	* создается сущность`SinkDisposer`(2);
	* у оператора `Filter` вызывается `run` метод;
	* создается sink(2) класса `FilterSink`;
	* `self._source.subscribe(sink)` вызывает следующее:
		* создается сущность`SinkDisposer`(3);
		* у оператора `Timer` вызывается `run` метод;
		* создается sink(3) класса `TimerOneOffSink`;
		* у созданного sink вызывается метод `run`;
		* в замыкании создается подписка subscription(3), с тем самым таймером `DispatchSourceTimer` который и будет делать нашу работу;
		* сохраняются сильные ссылки на sink(3) и subscription(3) в `SinkDisposer`(3)
		* `SinkDisposer`(3) возвращается;
	* `SinkDisposer`(3) под видом подписки subscription(2) сохраняется сильной ссылкой вместе с sink(2) у объекта `SinkDisposer`(2);
	* `SinkDisposer`(2) возвращается;
* `SinkDisposer`(2) под видом подписки subscription(1) сохраняется сильной ссылкой вместе с sink(1) у объекта `SinkDisposer`(1);
* `SinkDisposer`(1) возвращается;

`SinkDisposer`(1) будет той самой сущностью, у которой вызывается метод `.disposed(by: disposeBag)`

Каждый sink также содержит сильную ссылку на observer:

* для sink(1) observer-ом является замыкание `{ (event: Event<String>) -> Void in print(event) }`;
* для sink(2) observer-ом является `skip` sink(1);
* для sink(3) observer-ом является `filter` sink(2);

Давайте посмотрим на обратный процесс освобождения ресурсов, если мы у `SinkDisposer`(1) вызовем метод `dispose()`:

* `SinkDisposer`(1) освобождает sink(1) и subscription(1)
* subscription(1) это `SinkDisposer`(2), он освобождает sink(2) и subscription(2);
* subscription(2) это `SinkDisposer`(3), он освобождает sink(3) и subscription(3);
* subscription(3) освобождает непосредственно сам таймер;

