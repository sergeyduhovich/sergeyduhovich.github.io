---
layout: single
title: "Создание Observable"
date: 2019-05-03 12:00:00 +0300
categories: rxswift
---

В этом посте я опишу все операторы создания, которые доступны в соответствующем [разделе](http://reactivex.io/documentation/operators.html#creating) на сайте reactivex.

## Create 

Создает Observable с нуля, мы можем  напрямую вызывать ивенты у observer. Очень часто используется, когда нужно сделать Observable из замыкания:

```swift
struct Hotel: Codable {
  let title: String
  let rating: Float
}

enum APIError: Error {
  case noResponse
  case invalidFormat
  case invalidEndpoint
}

let intObservable = Observable<Int>.create { observer -> Disposable in
  observer.onNext(1)
  observer.onNext(2)
  observer.onCompleted()
  return Disposables.create()
}

let hotelsObservable = Observable<[Hotel]>.create { observer -> Disposable in
  guard let url = URL(string: "https://cool.api.hotels.com/v1/hotels") else {
    observer.onError(APIError.invalidEndpoint)
    return Disposables.create()
  }
  let task = URLSession.shared
    .dataTask(with: url) { (data, response, error) in
      do {
        guard let data = data else {
          observer.onError(APIError.noResponse)
          return
        }
        let json = try JSONDecoder().decode([Hotel].self, from: data)
        observer.onNext(json)
        observer.onCompleted()
      } catch {
        observer.onError(error)
      }
  }
  
  task.resume()
  
  return Disposables.create {
    task.cancel()
  }
}

intObservable
  .subscribe(onNext: { number in
    print(number)
  })
  .disposed(by: disposeBag)

hotelsObservable
  .subscribe(onNext: { hotels in
    print(hotels)
  })
  .disposed(by: disposeBag)
```

## Defer

Предотвращает выполнение кода из замыкания, до того момента, пока не появится хотя бы один подписчик. 

Давайте посмотрим на следующую проблему:

```swift
enum MyError: Error {
  case invalidCondition(String)
}

class ClassWithHeavyInit {
  init?() {
    print("initializer")
    sleep(5)
    print("initializer after sleep")
    if Int.random(in: 0...5) != 5 {
      return nil
    }
  }
}

func slowObservable() -> Observable<ClassWithHeavyInit> {
  if let result = ClassWithHeavyInit() { //выполнится и без вызова subscribe
  	//выполнится только после вызова subscribe
    return .just(result)
  } else {
  	//выполнится только после вызова subscribe
    return .error(MyError.invalidCondition(String(describing: ClassWithHeavyInit.self))) //executed only with subscribers
  }
}

slowObservable()
```
Несмотря на то, что у нас пока нет подписчика, абсолютно весь код, кроме `return .just(...)` или `return .error(...)` будет выполнен. Лог в консоли тому подтверждение:

```
initializer
initializer after sleep
```

Что бы избежать такого поведения, мы заворачиваем участок, отвечающий за создание Observable в `deferred` замыкание. Давайте посмотрим как будет выглядеть рабочий вариант `slowObservable`:

```swift
func slowObservable() -> Observable<ClassWithHeavyInit> {
  return Observable.deferred {
    if let result = ClassWithHeavyInit() {
      return .just(result)
    } else {
      return .error(MyError.invalidCondition(String(describing: ClassWithHeavyInit.self)))
    }
  }
}
```

И в консоли теперь ничего нет, до тех пор, пока не появится хотя бы один подписчик. Что и является ожидаемым поведением `slowObservable()`.

Я довольно часто использовал такой подход, когда работал с [Action](https://github.com/RxSwiftCommunity/Action). По мере создания сущностей нужно очень часто возвращать в замыканиях что-то типа `return .just(...)`.

## Empty

Создает пустой Observable, который завершается сразу после подписки на него:

```swift
_ = Observable<Int>
  .empty()
  .debug("empty")
  .subscribe()
```

в консоли:

```swift
empty -> subscribed
empty -> Event completed
empty -> isDisposed
```

Довольно частый сценарий использования `empty`:

```swift
class ViewController: UIViewController {

  override func viewDidLoad() {
    super.viewDidLoad()

    viewModel.hotels
      .flatMap { [weak self] hotels -> Observable<Float> in
        guard let self = self else { return Observable<Float>.empty() }
        return self.calculateRating(hotels: hotels)
      }
      .subscribe(onNext: { rating in
        print(rating)
      })
      .disposed(by: disposeBag)
  }
  
  func calculateRating(hotels: [Hotel]) -> Observable<Float> { ... }
}
```

Когда `self` фигурирует в замыкании с ключевым словом `weak`, но в самом замыкании мы не можем вернуть `nil`, а только конкретный Observable, нам нужно вызвать `guard` блок, и в нем вернуть некий Observable, который устроит и нас и компилятор.
`.empty()` здесь идеально вписывается.

## Never

Создает Observable, который никогда не завершается:

```swift
Observable<Int>
  .never()
  .debug("never")
  .subscribe()
  .disposed(by: disposeBag)
```

в консоли:

```swift
never -> subscribed
```

Я бы использовал этот оператор с предельной осторожностью. Это один из случаев, когда Observable не будет завершен нормальным путем с освобождением ресурсов. Нам в обязательном порядке нужен `disposeBag`.

Я не вижу очень много мест, где можно применить этот оператор, разве что в мок-классах, что бы удовлетворить требования протокола, где нам нужно вернуть что-то осмысленное.

## Error

Возвращает Observable, который в момент подписки на него завершается с указанной ошибкой.

```swift
enum MyError: Error {
  case customError
}

_ = Observable<Int>
  .error(MyError.customError)
  .debug("error")
  .subscribe()
```

в консоли:

```swift
error -> subscribed
error -> Event error(customError)
error -> isDisposed
```

Мы использовали этот оператор в примере для Defer:

```swift
func slowObservable() -> Observable<ClassWithHeavyInit> {
  if let result = ClassWithHeavyInit() {
    return .just(result)
  } else {
    return .error(MyError.invalidCondition(String(describing: ClassWithHeavyInit.self)))
  }
}
```

## From

Трансформирует объект или массив в Observable, объект может быть опциональным типом. После подписки на него генерирует `next` ивенты из переданных объектов и завершается. Есть возможность указать `scheduler` вторым аргументом.

Для массивов и последовательностей:

```swift
Observable.from([1, 2, 3, 4], scheduler: MainScheduler.instance)
  .debug("from array")
  .subscribe()
  .disposed(by: disposeBag)
```

в консоли:

```swift
from array -> subscribed
from array -> Event next(1)
from array -> Event next(2)
from array -> Event next(3)
from array -> Event next(4)
from array -> Event completed
from array -> isDisposed
```

Для `optional` аргумента:

```swift
Observable.from(optional: "QWERTY", scheduler: MainScheduler.instance)
  .debug("from qwerty")
  .subscribe()
  .disposed(by: disposeBag)
```

в консоли:

```swift
from qwerty -> subscribed
from qwerty -> Event next(QWERTY)
from qwerty -> Event completed
from qwerty -> isDisposed
```

Если в качестве `optional` аргумента передать `nil`, то мы не увидим `next` ивента, только `completed`, как в `empty`:

```swift
Observable<Int>.from(optional: nil, scheduler: MainScheduler.instance)
  .debug("from nil")
  .subscribe()
  .disposed(by: disposeBag)
```

в консоли:

```swift
from nil -> subscribed
from nil -> Event completed
from nil -> isDisposed
```

Еще один способ трансформировать объект в Observable - это `of` оператор.

```swift
Observable.of([1, 2, 3, 4])
  .debug("of array")
  .subscribe()
  .disposed(by: disposeBag)
```

в консоли:

```swift
of array -> subscribed
of array -> Event next([1, 2, 3, 4])
of array -> Event completed
of array -> isDisposed
```

Сравнивая с `from`, в частности для массива, мы можем заметить, что в этом случае у нас будет всего 1 `next` ивент. `from` генерирует n `next` ивентов по количеству элементов в массиве.

## Interval 

Генерирует Observable, который генерирует целочисленные значения, начиная с 0, увеличивая на 1, с заданным интервалом:

```swift
Observable<Int>.interval(0.5, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
  .debug("background")
  .subscribe()
  .disposed(by: disposeBag)

Observable<Int>.interval(1, scheduler: MainScheduler.instance)
  .debug("UI")
  .subscribe()
  .disposed(by: disposeBag)

sleep(3)
```

в консоли(пока мы не остановим приложение):

```swift
background -> subscribed
UI -> subscribed
background -> Event next(0)
background -> Event next(1)
background -> Event next(2)
background -> Event next(3)
background -> Event next(4)
UI -> Event next(0)
background -> Event next(5)
background -> Event next(6)
UI -> Event next(1)
background -> Event next(7)
background -> Event next(8)
background -> Event next(9)
UI -> Event next(2)
background -> Event next(10)
background -> Event next(11)
UI -> Event next(3)
...
```

## Just 

Трансформирует объект в Observable, который после подписки на него сразу отправляет  `next` ивент и завершается:

```swift
let just = Observable<Int>.just(1)

let justBg = Observable<Int>.just(1, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
```

## Range 

Создает Observable, который генерирует ивенты исходя из диапазона range:

```swift
Observable<Int>.range(start: 1, count: 7, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
  .debug("background")
  .subscribe()
  .disposed(by: disposeBag)

Observable<Int>.range(start: 1, count: 7, scheduler: MainScheduler.instance)
  .debug("UI")
  .subscribe()
  .disposed(by: disposeBag)

sleep(3)
```

в консоли:

```swift
background -> subscribed
background -> Event next(1)
UI -> subscribed
UI -> Event next(1)
background -> Event next(2)
background -> Event next(3)
background -> Event next(4)
background -> Event next(5)
background -> Event next(6)
background -> Event next(7)
background -> Event completed
background -> isDisposed

//and after few seconds

UI -> Event next(2)
UI -> Event next(3)
UI -> Event next(4)
UI -> Event next(5)
UI -> Event next(6)
UI -> Event next(7)
UI -> Event completed
UI -> isDisposed
```

Странное поведение для main потока, несмотря на то что он заблокирован (`sleep(3)`), первый ивент мы получаем в любом случае.

## Repeat 

Создает Observable, который генерирует указанное число без остановки и задержки:

```swift
Observable<Int>.repeatElement(1)
  .debug("UI")
  .subscribe()
  .disposed(by: disposeBag)

Observable<String>.repeatElement("bla", scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
  .debug("background")
  .subscribe()
  .disposed(by: disposeBag)
```

в консоли(пока мы не остановим приложение):

```swift
UI -> subscribed
UI -> Event next(1)
UI -> Event next(1)
UI -> Event next(1)
UI -> Event next(1)
UI -> Event next(1)
UI -> Event next(1)
UI -> Event next(1)
UI -> Event next(1)
...
```

Как Вы можете видеть, никаких признаков присутствия второго Observable, т.к. main поток заблокирован нашим оператором, который очень похож по поведению на `while true { }`. Крайне не рекомендую его использовать без операторов `until`, `take(..)`.

## Timer

Создает Observable, который генерирует 1 ивент после определенной задержки, либо бесконечное количество ивентов, если мы укажем `period`:

```swift
Observable<Int>.timer(5, scheduler: MainScheduler.instance)
  .debug("once after 5s")
  .subscribe()
  .disposed(by: disposeBag)

Observable<Int>.timer(3, period: 0.5, scheduler: MainScheduler.instance)
  .debug("UI")
  .subscribe()
  .disposed(by: disposeBag)

Observable<Int>.timer(1, period: 0.5, scheduler: ConcurrentDispatchQueueScheduler(qos: .background))
  .debug("background")
  .subscribe()
  .disposed(by: disposeBag)
```

в консоли:

```swift
once after 5s -> subscribed
UI -> subscribed
background -> subscribed
background -> Event next(0)
background -> Event next(1)
background -> Event next(2)
background -> Event next(3)
UI -> Event next(0)
background -> Event next(4)
UI -> Event next(1)
background -> Event next(5)
UI -> Event next(2)
background -> Event next(6)
UI -> Event next(3)
background -> Event next(7)
once after 5s -> Event next(0)
once after 5s -> Event completed
once after 5s -> isDisposed
UI -> Event next(4)
background -> Event next(8)
UI -> Event next(5)
background -> Event next(9)
UI -> Event next(6)
background -> Event next(10)
```

Как мы видим, без `period` поведение как у метода `asyncAfter(deadline:)`. С параметром `period` работает как оператор `interval`, с указанной задержкой.