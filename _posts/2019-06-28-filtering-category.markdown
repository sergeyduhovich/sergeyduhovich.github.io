---
layout: single
title: "Фильтрация"
date: 2019-06-28 12:00:00 +0300
categories: rxswift
---

В этой статье я пройдусь по операторам фильтрации, которые можно найти на сайте [reactivex](http://reactivex.io/documentation/operators.html#filtering).

### Filter

Оператор очень похож на стандартный `filter` оператор, который применяется к коллекциям. Он игнорирует все ивенты, значения которых не удовлетворяют условию кложуры.

```swift
Observable
  .of(0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55)
  .filter { $0 > 5 }
  .debug("numbers greater than 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
numbers -> subscribed
numbers -> Event next(8)
numbers -> Event next(13)
numbers -> Event next(21)
numbers -> Event next(34)
numbers -> Event next(55)
numbers -> Event completed
numbers -> isDisposed
```

### Take

#### Take count

Мы уже видели этот оператор (`.take(1).asSingle()`), когда разбирали тему [traits](http://dukhovich.by/ru/17-traits). Он берет `count` количество ивентов и после них эмитит `completed`.

```swift
Observable
  .of(0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55)
  .take(5)
  .debug("take 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
take 5 -> subscribed
take 5 -> Event next(0)
take 5 -> Event next(1)
take 5 -> Event next(1)
take 5 -> Event next(2)
take 5 -> Event next(3)
take 5 -> Event completed
take 5 -> isDisposed
```

#### Take last

Возвращает указанное количество элементов с конца **завершенной** последовательности. 

```swift
Observable
  .of(0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55)
  .takeLast(5)
  .debug("take last 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
take last 5 -> subscribed
take last 5 -> Event next(8)
take last 5 -> Event next(13)
take last 5 -> Event next(21)
take last 5 -> Event next(34)
take last 5 -> Event next(55)
take last 5 -> Event completed
take last 5 -> isDisposed
```

Очень важный момент в определении выше - Observable не может быть бесконечной, иначе результат применения этого оператора бесполезен:

```swift
Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)
  .takeLast(5)
  .debug("take last 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
take last 5 -> subscribed
```

#### TakeWhile

Возвращает элементы до тех пор пока условие в кложуре равно `true`, как только условие станет `false` Observable завершится `completed` ивентом.

Давайте представим, что есть Observable, который эмитит следующие значения целых чисел:

[0, 1, 2, 3, 4, 0, 1, 2, 3, 4, 0, 1, 2, 3, 4, ...]

```swift
Observable<Int>
  .interval(0.5, scheduler: MainScheduler.instance)
  .map { num -> Int in
    if num >= 5 {
      return num % 5
    } else {
      return num
    }
  }
  .takeWhile { $0 < 4 }
  .debug("takeWhile < 4")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
takeWhile < 4 -> subscribed
takeWhile < 4 -> Event next(0)
takeWhile < 4 -> Event next(1)
takeWhile < 4 -> Event next(2)
takeWhile < 4 -> Event next(3)
takeWhile < 4 -> Event completed
takeWhile < 4 -> isDisposed
```

В случае, если в кложуре всегда возвращается `true` (например, `.takeWhile { $0 < 5 }` в примере выше), результирующая Observable просто повторяет поведение исходной.

#### Take duration

На протяжении указанного времени пробрасывает значения из исходной Observable, потом завершается.

```swift
Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)
  .take(5, scheduler: MainScheduler.instance)
  .debug("take duration = 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
take duration = 5 -> subscribed
take duration = 5 -> Event next(0)
take duration = 5 -> Event next(1)
take duration = 5 -> Event next(2)
take duration = 5 -> Event next(3)
take duration = 5 -> Event completed
take duration = 5 -> isDisposed
```

#### Take until

Пробрасывает значения из исходной Observable до тех пор пока другая Observable не пришлет хотя бы один `next` ивент.

```swift
let stopSequence = Observable<Int>
  .just(1)
  .delay(5, scheduler: MainScheduler.instance)

Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)
  .takeUntil(stopSequence)
  .debug("take until")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
take until -> subscribed
take until -> Event next(0)
take until -> Event next(1)
take until -> Event next(2)
take until -> Event next(3)
take until -> Event completed
take until -> isDisposed
```

### Skip

#### Skip count

Игнорирует указанное количество ивентов, дальше пробрасывает значения исходной Observable.

```swift
Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)
  .debug("source")
  .skip(4)
  .debug("result after skip 4")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
result after skip 4 -> subscribed
source -> subscribed
source -> Event next(0)
source -> Event next(1)
source -> Event next(2)
source -> Event next(3)
//four events above were skipped
source -> Event next(4)
result after skip 4 -> Event next(4)
source -> Event next(5)
result after skip 4 -> Event next(5)
source -> Event next(6)
result after skip 4 -> Event next(6)
```

#### Skip duration

На протяжении указанного времени игнорирует ивенты, дальше пробрасывает значения исходной Observable.

```swift
Observable<Int>
  .interval(0.5, scheduler: MainScheduler.instance)
  .debug("source")
  .skip(3, scheduler: MainScheduler.instance)
  .debug("result skip duration 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
result skip duration 5 -> subscribed
source -> subscribed
source -> Event next(0)
source -> Event next(1)
source -> Event next(2)
source -> Event next(3)
source -> Event next(4)
source -> Event next(5)
result skip duration 5 -> Event next(5)
source -> Event next(6)
result skip duration 5 -> Event next(6)
source -> Event next(7)
result skip duration 5 -> Event next(7)
```

#### Skip while

Игнорирует ивенты до тех пор, пока условие в кложуре равно `true`, дальше пробрасывает значения исходной Observable игнорируя условие в кложуре.

```swift
Observable<Int>
  .interval(0.5, scheduler: MainScheduler.instance)
  .map { num -> Int in
    if num >= 5 {
      return num % 5
    } else {
      return num
    }
  }
  .skipWhile { $0 < 4 }
  .debug("skipWhile < 4")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
skipWhile < 4 -> subscribed
skipWhile < 4 -> Event next(4)
skipWhile < 4 -> Event next(0)
skipWhile < 4 -> Event next(1)
skipWhile < 4 -> Event next(2)
skipWhile < 4 -> Event next(3)
skipWhile < 4 -> Event next(4)
skipWhile < 4 -> Event next(0)
skipWhile < 4 -> Event next(1)
skipWhile < 4 -> Event next(2)
skipWhile < 4 -> Event next(3)
```

#### Skip until

Игнорирует ивенты до тех пор, пока другая Observable не пришлет хотя бы один `next` ивент, дальше пробрасывает значения исходной Observable.

```swift
let startSequence = Observable<Int>
  .just(1)
  .delay(5, scheduler: MainScheduler.instance)

Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)
  .skipUntil(startSequence)
  .debug("skipUntil startSequence")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
skipUntil startSequence -> subscribed
skipUntil startSequence -> Event next(5)
skipUntil startSequence -> Event next(6)
skipUntil startSequence -> Event next(7)
skipUntil startSequence -> Event next(8)
```

#### Skip vs Take

В какой-то мере, оператор `skip` является антагонистом оператору `take`. Если бы они были применены к одной и той же исходной Observable с одинаковыми параметрами, они бы ее разделили на 2 части.

Давайте еще раз посмотрим на пример, который был выше:

[0, 1, 2, 3, 4, 0, 1, 2, 3, 4, 0, 1, 2, 3, 4, ...] - source

[**0, 1, 2, 3**, 4, 0, 1, 2, 3, 4, 0, 1, 2, 3, 4, ...] - `takeWhile`

[0, 1, 2, 3, **4, 0, 1, 2, 3, 4, 0, 1, 2, 3, 4, ...**] - `skipWhile`

### Distinct

`distinctUntilChanged` оператор игнорирует элемент, если он не отличается от предыдущего.

```swift
Observable
  .of(0, 0, 0, 1, 0, 0, 0, 0, 0, 1)
  .distinctUntilChanged()
  .debug("distinctUntilChanged")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
distinctUntilChanged -> subscribed
distinctUntilChanged -> Event next(0)
distinctUntilChanged -> Event next(1)
distinctUntilChanged -> Event next(0)
distinctUntilChanged -> Event next(1)
distinctUntilChanged -> Event completed
distinctUntilChanged -> isDisposed
```

### Debounce и Throttle

Давайте рассмотрим распространенный, но немного упрощенный сценарий: пользователь набирает текст в строке поиска, с каждой набранной буквой выполняется поисковой API запрос. После применения операторов `debounce` и `throttle` количество API запросов будет уменьшено.

#### Debounce

Игнорирует элементы, если временной интервал между ними меньше порогового значения. Если между какими-то 2 ивентами интервал больше указанного значения, эмитится первый ивент.

Пример ниже аналогичен быстрому набору цифр, но в интервале от 5 до 10, пауза вдвое больше обычной.

```swift
Observable<Int>
  .interval(0.2, scheduler: MainScheduler.instance)
  .filter { num in
    if num > 5 && num < 10 {
      return num % 2 == 0
    } else {
      return true
    }
  }
  .debug("source")
  .debounce(0.3, scheduler: MainScheduler.instance)
  .debug("debounce applied")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
debounce applied -> subscribed
source -> subscribed
source -> Event next(0)
source -> Event next(1)
source -> Event next(2)
source -> Event next(3)
source -> Event next(4)
source -> Event next(5)
source -> Event next(6)
debounce applied -> Event next(6)
source -> Event next(8)
debounce applied -> Event next(8)
source -> Event next(10)
source -> Event next(11)
source -> Event next(12)
```

#### Throttle

Пробрасывает ивенты исходного Observable, начиная с первого, если к моменту прихода следующего ивента прошло достаточно времени, он будет проброшен, иначе он будет проигнорирован.

```swift
Observable<Int>
  .interval(0.2, scheduler: MainScheduler.instance)
  .debug("source")
  .throttle(1, scheduler: MainScheduler.instance)
  .debug("throttle applied")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
throttle applied -> subscribed
source -> subscribed
source -> Event next(0)
throttle applied -> Event next(0)
source -> Event next(1)
source -> Event next(2)
source -> Event next(3)
source -> Event next(4)
source -> Event next(5)
throttle applied -> Event next(5)
source -> Event next(6)
source -> Event next(7)
source -> Event next(8)
source -> Event next(9)
source -> Event next(10)
throttle applied -> Event next(10)
```

### ElementAt

Эмитит только указанный _n_-й элемент.

```swift
Observable<Int>
  .interval(0.2, scheduler: MainScheduler.instance)
  .debug("source")
  .elementAt(5)
  .debug("elementAt 5")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
elementAt 5 -> subscribed
source -> subscribed
source -> Event next(0)
source -> Event next(1)
source -> Event next(2)
source -> Event next(3)
source -> Event next(4)
source -> Event next(5)
elementAt 5 -> Event next(5)
elementAt 5 -> Event completed
elementAt 5 -> isDisposed
source -> isDisposed
```

### IgnoreElements

Игнорирует все элементы и завершается (или фейлится) вместе с завершением (или ошибкой) исходного Observable. Равнозначен применению оператора `filter { _ in false }`.

```swift
Observable<Int>
  .interval(0.2, scheduler: MainScheduler.instance)
  .take(5)
  .debug("source")
  .ignoreElements()
  .debug("ignoreElements")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
ignoreElements -> subscribed
source -> subscribed
source -> Event next(0)
source -> Event next(1)
source -> Event next(2)
source -> Event next(3)
source -> Event next(4)
source -> Event completed
source -> isDisposed
ignoreElements -> Event completed
ignoreElements -> isDisposed
```

### Sample

На каждый ивент сэмплер-Observable проверяется, были ли элементы у исходного Observable, если да, то они пробрасываются дальше.

```swift
let sampler = Observable<Int>
  .interval(1, scheduler: MainScheduler.instance)

Observable<Int>
  .interval(0.3, scheduler: MainScheduler.instance)
  .debug("source")
  .sample(sampler)
  .debug("sample applied")
  .subscribe()
  .disposed(by: disposeBag)
```

```swift
sample applied -> subscribed
source -> subscribed
source -> Event next(0)
source -> Event next(1)
source -> Event next(2)
sample applied -> Event next(2)
source -> Event next(3)
source -> Event next(4)
source -> Event next(5)
sample applied -> Event next(5)
source -> Event next(6)
source -> Event next(7)
source -> Event next(8)
source -> Event next(9)
sample applied -> Event next(9)
```