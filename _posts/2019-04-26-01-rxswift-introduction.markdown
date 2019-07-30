---
layout: single
title: "Реактивное программирование"
date: 2019-04-26 12:00:00 +0300
categories: rxswift
---

Реактивное программирование — парадигма программирования, ориентированная на потоки данных и распространение изменений. Цитата с [Wiki](https://www.wikiwand.com/ru/%D0%A0%D0%B5%D0%B0%D0%BA%D1%82%D0%B8%D0%B2%D0%BD%D0%BE%D0%B5_%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5), так же неплохой пост о реактивном программировании можно найти на [stackoverflow](https://stackoverflow.com/a/1030631) (en)

В этой статье я расскажу о фреймворке RxSwift, который является портом [ReactiveX](http://reactivex.io/) на Swift.

> ReactiveX - это комбинация паттерна Observer, паттерна Iterator и функционального программирования

В основе ReactiveX лежит 2 паттерна Observer и Iterator. Хотя на том же сайте можно найти упоминание [Reactor и Iterator](http://reactivex.io/documentation/observable.html). Но это не очень сильно меняет дело. Для простоты, далее я буду оперировать паттернами Observer и Iterator.

## Пример

Посмотрим на пример, взятый с репозитория [RxSwift](https://github.com/ReactiveX/RxSwift):

Сначала мы формируем Observable `searchResults` для URL запроса к API GitHub и получения списка репозиториев:

```swift
let searchResults = searchBar.rx.text.orEmpty
  .throttle(0.3, scheduler: MainScheduler.instance)
  .distinctUntilChanged()
  .flatMapLatest { query -> Observable<[Repository]> in
    if query.isEmpty {
      return .just([])
    }
    return searchGitHub(query)
      .catchErrorJustReturn([])
  }
  .observeOn(MainScheduler.instance)
```

После этого мы привязываем этот Observable к UITableView, что бы видеть результат выполнения запросов:

```swift
searchResults
  .bind(to: tableView.rx.items(cellIdentifier: "Cell")) {
    (index, repository: Repository, cell) in
    cell.textLabel?.text = repository.name
    cell.detailTextLabel?.text = repository.url
  }
  .disposed(by: disposeBag)
```

Полученный результат:

![tableview](http://uploads.dukhovich.by/articles/GithubSearch.gif)

## Плюсы RxSwift

Давайте взглянем на следующие 2 блока кода, с использованием RxSwift:

```swift
_ = Observable<String>
  .just("hello world") //2
  .subscribe(onNext: { element in //3
    print(element) //1
  })
```

```swift
observable //2
  .map { element in //4
    return "current element is \(element)"
  }
  .observeOn(ConcurrentDispatchQueueScheduler(qos: .background)) //4
  .subscribe(onNext: { element in //3
    print(element) //1
  })
  .disposed(by: disposeBag) //5
```

### 1. Компактная форма записи

На мой взгляд, большим плюсом RxSwift (как наверно и любого другого реактивного фреймворка) является линейная запись (сверху вниз) асинхронных вызовов. Примеры выше записаны приблизительно по одной и той же схеме:

1. У нас есть блок кода, выполняющий полезную работу;
1. Есть непосредственно сам Observable;
1. Есть метод `subscribe`, который связывает Observable с конкретным Observer;
1. Опционально может присутствовать блок трансформации исходного Observable;
1. Метод `disposed(by:)` помогает избежать утечек памяти, пока примем его как обязательной или стилистической фишкой RxSwift. Более подробно об этом в следующих постах.

Если у Вас есть опыт разработки под iOS / MacOS / watchOS, скорее всего Вы уже столкнулись с большим набором разношерстных API, которое предоставляет Apple. Data-source и delegate для таблиц, target-action для обработки кликов по кнопкам, подписки к Notification Center, KVO. Попытка использовать все эти API в одном файле, может обернуться тем, что файл будет сложно читать и поддерживать. 

В RxSwift, с его однотипной структурой записи, с этим немного проще. Я не стану отрицать, и в RxSwift можно написать цепочку таким образом, что в ней сможет разобраться только тот, кто ее написал. Но в среднем, однотипная форма записи по шагам, описанным выше, упрощает сопровождение исходников.

Еще немного примеров того, как может выглядеть RxSwift в проекте:

```swift
struct Hotel {
  let title: String
  let rating: Float
}

let hotelsObservable: Observable<[Hotel]> = ...

hotelsObservable
  .observeOn(MainScheduler.instance)
  .bind(to: tableView.rx.items(cellIdentifier: "cell", cellType: UITableViewCell.self)) { index, model, cell in
    cell.textLabel?.text = model.title
}
.disposed(by: disposeBag)

hotelsObservable
  .observeOn(MainScheduler.instance)
  .map { "we found \($0.count) in your range" }
  .bind(to: label.rx.text)
  .disposed(by: disposeBag)

tableView.rx.modelSelected(Hotel.self)
  .subscribe(onNext: { [weak self] hotel in
    self?.openDetails(hotel: hotel)
  })
  .disposed(by: disposeBag)

NotificationCenter.default.rx
  .notification(UIApplication.didEnterBackgroundNotification)
  .subscribe(onNext: { [weak self] _ in
    self?.saveData()
  })
  .disposed(by: disposeBag)

button.rx
  .tap
  .bind { [weak self] in
    self?.refreshData()
  }
  .disposed(by: disposeBag)
```

### 2. Кроссплатформенность

RxSwift является портом ReactiveX, все знания Вы сможете использовать в других языках. Ну или как минимум, Вы сможете обсудить детали реализации с коллегами, которые работают с ReactiveX вне зависимости от того, на каком языке они программируют.

### 3. Простота работы с потоками

Переключение между потоками очень простое, переключить тяжелый блок кода в фоновый поток проще простого. Основные методы для управления потоками - `.observeOn(...)` и `.subscribeOn(...)`.

### 4. Комьюнити

У RxSwift довольно неплохое комьюнити. Много [примеров](https://github.com/ReactiveX/RxSwift/tree/master/RxExample) на github, довольно много extensions: RxCocoa для UI, RxDataSources для таблиц, RxAlamofire для работы с сетью, RxCoreLocation и другие.

### 5. Идеально вписывается в MVVM-like архитектуры

RxSwift хорошо подходит для MVVM. В RxCocoa есть удобные биндинги на UI. Помимо этого, довольно легко комбинировать Observable, используя большой набор операторов. И в большинстве случаев для этого не нужно писать слишком много кода.
Продолжая архитектурную тему, в RxSwift так же есть типы, которые одновременно являются Observable и Observer - `BehaviourRelay` и `PublishRelay`, идеально подходят для стыковочных мест между императивным и декларативным миром.

## Минусы RxSwift

### 1. Избыточность

RxSwift довольно избыточный, написать приложение можно и без него. С RxSwift в некоторых ситуациях очень легко пишется код, но количество объектов в памяти, стек количества вызовов методов значительно возрастает.

### 2. Кривая обучечния

Начать использовать RxSwift с первой минуты знакомства без ошибок и избыточных преобразований не получится. В зависимости от интенсивности на более-менее уверенное использование фреймворка уйдет от 1 до 2 недель. Разбирая существующие примеры с github, Вы однозначно ускорите этот процесс.

### 3. Очень сложный дебаг

Дебаг в RxSwift на первый взгляд может показаться очень сложным, но и на второй взгляд он окажется довольно сложным. Да, `.debug()` помогает в некоторых случаях, но это явно не панацея. Размер стека вызовов при добавлении брейкпойнта тоже впечатляет, например, для того чтобы найти в стеке предыдущий вызов из Ваших исходников может понадобится какое-то время.

## Добавлять или нет?

Пару "за" и "против" добавления RxSwift в проект.

### За

Если у вас нет на носу дедлайна, и у Вас есть время поразбираться с че-то новым. RxSwift помогает в случаях, когда нужно комбинировать сложные асинхронные цепочки. Пожалуй, цепочку любой сложности можно получить, используя существующие операторы. В RxSwift так же есть такие типы, как Subject, своего рода мост между императивным и декларативным миром. Subject может выступать в роли Observable, и в то же время он может быть Observer, т.е. принимать объекты и эмитить ивенты. Подходит для MVVM архитектур.  

### Против

Если у Вас нет опыта или ментора, который мог бы Вам помогать в случае, если Вы застрянете. Вы работаете на проекте, у которого в скором времени дедлайн. Стоит помнить, что RxSwift не панацея, любая задача может быть решена и без него.

## Event

> ... ориентированная на потоки данных и распространение изменений

В RxSwift под данными подразумевается тип `enum Event<Element>`, который может принимать 3 возможных значения : `next`, содержащий значение, конкретного типа; `error`, ошибка и `completed`.


```swift
public enum Event<Element> {
	/// Next element is produced.
	case next(Element)
	
	/// Sequence terminated with an error.
	case error(Swift.Error)
	
	/// Sequence completed successfully.
	case completed
}
```

Каждый Observable может отправить 0 и более элементов. В большинстве случаев ивенты будут приходить только в том случае, если есть хотя бы один подписчик на Observable (имеется ввиду холодный Observable, об этом будет рассказано в следующих постах). Нет никаких ограничений, для типов, которые могут использоваться в ивентах, главное правило - тип должен быть один и только один. Observable не может быть сразу двух типов, например `Int` и `String`. В теории можно сделать Observable с неким протоколом, который будут реализовать 2 разных типа, но это обычно не используется.

Давайте посмотрим на marble диаграмму. По мере изучения RxSwift Вы будете часто их [встречать](https://rxmarbles.com/). Очень полезная утилита для визуального представления и понимания, что происходит.

![rxmarbles_debounce](http://uploads.dukhovich.by/articles/post_1_rx_marbles.png)

Кругами на диаграмме обозначаются `next` ивенты, вертикальной линией - `completed`, перекрестие - `error`. **Нормальное завершение** любого Observable - это наличие **Completed** или **Error** ивента. После того как любой из них был получен, Observable не может ничего больше отправлять, также освобождаются все ресурсы, выделенные под Observable.

На мой взгляд, это основа RxSwift, чем раньше Вы поймете это, тем проще Вам будет с более сложными цепочками преобразований. Еще раз - когда **Error** или **Completed** ивенты генерируются, весь **Observable останавливается**. Никаких ивентов больше не будет. Я расскажу подробнее о техниках обработки ошибок, что бы Observable мог жить настолько долго, насколько нам нужно, несмотря на приходящие ошибки.

Давайте еще раз взглянем на тип `enum Event<Element>`, почему только тип для `next` ивента передается как дженерик, а ошибка нет? Хорошее объяснение можно найти в [документации](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/DesignRationale.md). Вкратце, что бы было удобнее комбинировать сигналы между собой.

## Что такое DisposeBag и Disposable?

Несмотря на то, что у swift есть ARC, в RxSwift нужен дополнительный инструмент для управления памятью. DisposeBag - класс контейнер для подписок, который очищает существующие подписки в момент, когда он удаляется. По умолчанию, подписка (вызов `subscribe`) создает так называемый retain-cycle между Observable и Observer. Без добавления подписок в специальный класс контейнер мы будем получать утечки памяти. Не будет ошибкой добавлять все подписки в `disposeBag`, несмотря на то, что есть пару исключений, например `.just()`, `.empty()`, с которыми это делать не обязательно.

Каждый раз, когда Вы обнуляете или переприсваиваете новое значение переменной типа `DisposeBag`, Вы освобождаете все ресурсы, выделенные для этих подписок. Так же Вы принудительно завершаете Observable.

## Горячие / Холодные сигналы

Еще один важный момент в RxSwift, [понимание разницы между горячим и холодным сигналом](http://dukhovich.by/ru/15-connectable-observable). По-умолчанию, все Observable, созданные при помощи операторов [create](http://reactivex.io/documentation/operators/create.html), являются холодными. Что это значит? Когда Вы у переменной вызываете метод `subscribe`, процесс создания  Observable запускается по-новой, с выделением ресурсов. Например, если Вы создаете таймер используя метод `+interval`, вызвав 2 раза `subscribe` Вы создадите 2 таймера. В некоторых случаях нужно иметь общий Observable для проведения трансформаций и последующих подписок на них. 

Горячие сигналы в отличие от холодных не выделяют дополнительные ресурсы при увеличении количества подписчиков, если ресурсы уже выделены. Горячим сигналом может быть сокет, может быть `text` свойство у `UITextField`, или это может быть "Connectable" Observable.