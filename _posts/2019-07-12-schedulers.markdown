---
layout: single
title: "Шедулеры"
date: 2019-07-12 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

В предыдущем посте я рассмотрел нюансы работы 2 операторов, которые работают с шедулерами. Это `observeOn` и `subscribeOn`. Я писал, что шедулеры можно рассматривать как очередь в контексте работы этих операторов.

Но что из себя представляют шедулеры на самом деле?

В техническом плане - это реализация `SchedulerType` протокола, который  предполагает несколько вариантов выполнения кложур:

* выполнить кложуру;
* выполнить кложуру с delay;
* зациклить выполние кложуры с delay;

```swift
func schedule<StateType>(_ state: StateType, action: @escaping (StateType) -> Disposable) -> Disposable
func scheduleRelative<StateType>(_ state: StateType, dueTime: RxTimeInterval, action: @escaping (StateType) -> Disposable) -> Disposable
func schedulePeriodic<StateType>(_ state: StateType, startAfter: RxTimeInterval, period: RxTimeInterval, action: @escaping (StateType) -> StateType) -> Disposable
```

В RxSwift достаточное количество [готовых шедулеров](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Schedulers.md#builtin-schedulers):

### CurrentThreadScheduler

Выполняет кложуры на текущем потоке. Шедулер последовательный(serial). Реализует только `ImmediateSchedulerType` протокол. 

### SerialDispatchQueueScheduler

Используется `DispatchQueue` для выполнения кложур. Для простых случаев используется `queue.async`, когда нужно указать delay, или повторяющееся событие -  `DispatchSourceTimer`.

Для `observeOn` оператора на этом шедуелере есть небольшая оптимизация в виде `ObserveOnSerialDispatchQueueSink` и `ObserveOnSerialDispatchQueue`.

#### MainScheduler

`MainScheduler` является сабклассом `SerialDispatchQueueScheduler`, в качестве очереди используется `DispatchQueue.main`.

Если `schedule` метод вызывается с Main очереди, то кложура будет выполнена сразу без `queue.async`.

### ConcurrentDispatchQueueScheduler

Похож на `SerialDispatchQueueScheduler`, используется `queue.async` или `DispatchSourceTimer` с той лишь разницей, что очередь в данном случае конкурентная.

### ConcurrentMainScheduler

Этот шедулер использует `MainScheduler` для повторяющихся событий и для выполнения кложуры с delay. Для простого выполнения кложуры используется `DispatchQueue.main`

Шедулер оптимизирован для работы с оператором `subscribeOn`.

### OperationQueueScheduler

В случае, когда нужно явно указать количество потоков, для асинхронного выполнения, используется `OperationQueueScheduler`. `DispatchQueue` не иммеет API для ограничения количества потоков, в то время как `NSOperationQueue` имеет такой функционал. 

Поскольку этот шедулер реализует только `ImmediateSchedulerType`, повторяющиеся события и работа с delay не могут быть выполнены.

### VirtualTimeScheduler

Несмотря на то, что этот шедулер реализует протокол `SchedulerType`, он не похож на описанные выше шедулеры. Каждый вызов метода `schedule` создает `VirtualSchedulerItem`, который хранит предполагаемое время выполнения. В `RxTest`
можно найти его сабкласс - `TestScheduler`.

### Свой шедулер

Реализовав протокол `SchedulerType`, можно создать свой уникальный шедулер. Причем, не обязательно привязываться ко времени, как это делает большинство стандартных шедулеров.

## Операторы:

Список операторов, которые так или иначе используют шедулеры:

* window;
* interval;
* timeout;
* throttle;
* take(duration:);
* subscribeOn; 
* skip(duration:);
* just;
* repeatElement;
* range;
* delay;
* delaySubscription;
* from(optional:);
* debounce;
* buffer

Очень [подробная статья](https://www.uraimo.com/2017/05/07/all-about-concurrency-in-swift-1-the-present/) про многопоточность в swift. [Свободный перевод](https://medium.com/@alexey_nenastev/%D0%B2%D1%81%D1%91-%D0%BE-%D0%BC%D0%BD%D0%BE%D0%B3%D0%BE%D0%BF%D0%BE%D1%82%D0%BE%D1%87%D0%BD%D0%BE%D1%81%D1%82%D0%B8-%D0%B2-swift-%D1%87%D0%B0%D1%81%D1%82%D1%8C-1-%D0%BD%D0%B0%D1%81%D1%82%D0%BE%D1%8F%D1%89%D0%B5%D0%B5-f0b4d5718877) на русский язык.