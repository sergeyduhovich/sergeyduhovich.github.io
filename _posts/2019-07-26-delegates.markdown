---
layout: single
title: "Делегаты"
date: 2019-07-26 12:00:00 +0300
categories: rxswift
---

Операторы из категории create - не единственный способ создания Observable последовательности.

Иногда возникает необходимость преобразовать методы существующего делегата в Observable. Поскольку [паттерн делегирования](https://developer.apple.com/documentation/swift/cocoa_design_patterns/using_delegates_to_customize_object_behavior) очень распространен во фреймворках Apple, задача расширения делегатов за счет добавления Observable функционала, рано или поздно может понадобиться.

### UITableView

Давайте посмотрим на самый часто используемый класс, который настраивается делегатом и источником данных (dataSource), - `UITableView`.

Одна из простейших настроек таблицы с использованием Rx, может выглядеть следующим образом:

```swift
class ViewController: UIViewController {

  @IBOutlet var tableView: UITableView!

  private let disposeBag = DisposeBag()
  private let mostVisitedCities = Observable<[String]>.just([
    "Bangkok",
    "London",
    "Paris",
    "Dubai",
    "Singapore",
    "New York",
    "Kuala Lumpur",
    "Tokyo",
    "Istanbul",
    "Seoul",
    "Antalya",
    "Phuket",
    "Mecca",
    "Hong Kong",
    "Milan",
    "Palma de Mallorca",
    "Barcelona",
    "Pattaya",
    "Osaka",
    "Bali"
    ])

  override func viewDidLoad() {
    super.viewDidLoad()
    
    tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
    mostVisitedCities
      .bind(to: tableView.rx.items(cellIdentifier: "cell", cellType: UITableViewCell.self)) { index, model, cell in
        cell.textLabel?.text = model
      }
      .disposed(by: disposeBag)
  }
}
```

![countries](http://dukhovich.by/assets/images/articles/countries.png)

Запустив код, мы видим, что таблица отображается корректно, несмотря на то, что ни источник данных, ни делегат не был назначен на `ViewController`. Связь dataSource <-> вью контроллер была создана в тот момент, когда у таблицы был вызван метод `items(dataSource:)`. Реализацию можно посмотреть в файле `UITableView+Rx.swift`. Сразу же на глаза бросается класс `RxTableViewDataSourceProxy`, который предположительно отвечает за сайд эффект (неявная установка источника в таблице), который мы наблюдали выше.

Используем данный класс как начальную точку в поиске подробностей. Объявление класса `RxTableViewDataSourceProxy` выглядит следующим образом:

```swift
open class RxTableViewDataSourceProxy
    : DelegateProxy<UITableView, UITableViewDataSource>
    , DelegateProxyType 
    , UITableViewDataSource {
}
```

Мы видим протокол `UITableViewDataSource`, класс `DelegateProxy` с двумя конкретными типами и протокол `DelegateProxyType`.

После прочтения комментариев, оставленных в `DelegateProxyType.swift` понимаем причину, по которой были написаны все эти прокси и делегаты. Как разработчику, RxSwift предоставляет мне возможность использовать оба способа написания логики, в методе делегата и в Observable последовательности, которая является оберткой над этим самым делегатом.

Так же в файле можно увидеть заметки на тему того, как не нужно инициализировать `DelegateProxyType`:

> Реализации `DelegateProxyType` не должны инициализироваться напрямую.
> Для получения сущности реализованного типа `DelegateProxyType`,  должен быть вызван метод `proxy`.

Для некоторых классов, например `UITableView`, эта инициализация уже написана, все что нам нужно для настройки делегата - вызвать `tableView.rx.setDelegate(self)` где-нибудь во `viewDidLoad`.

### "Сломанный" делегат

Давайте проведем небольшой тест с использованием стандартного делегата. Я написал `didSelectRowAt` метод в расширении класса:

```swift
extension ViewController: UITableViewDelegate {
  func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
    print("call from extension")
  }
}
```

также, добавил подписку на rx-версию этого же метода во `viewDidLoad`:

```swift
tableView.rx.itemSelected
  .subscribe(onNext: { indexPath in
    print("call from rx.itemSelected")
  })
  .disposed(by: disposeBag)
```

Теперь интересная часть. В зависимости от того, в каком месте написать установку делегата `tableView.delegate = self` относительно rx-подписки, поведение будет разным, например:

Если я напишу `tableView.delegate = self` **перед** `tableView.rx.itemSelected` в консоли я увижу оба лога:

```swift
//call from extension
//call from rx.itemSelected
```

Если я напишу `tableView.delegate = self` **после** `tableView.rx.itemSelected` в консоли я увижу только лог из стандартной реализации делегата:

```swift
//call from extension
```

Что бы избавиться от такой неоднозначности в коде, нам нужно установить делегат не стандартным способом, а используя rx-версию:

```swift
tableView.rx.setDelegate(self)
  .disposed(by: disposeBag)
```

Код выше не чувствителен к тому, где он написан, перед или после rx-подписок.

### UINavigationController

В RxCocoa очень много расширений UIKit. Давайте посмотрим, как реализован `UINavigationController`. RxCocoa предоставляет 2 ControlEvent свойства: `willShow` и `didShow`, точно также, как и у обычного `UINavigationControllerDelegate`.

Повторим эксперимент выше. Сначала реализуем `willShow` у `UINavigationControllerDelegate`:

```swift
extension ViewController: UINavigationControllerDelegate {
  func navigationController(_ navigationController: UINavigationController, willShow viewController: UIViewController, animated: Bool) {
    print("call from extension")
  }
}
```

Также подпишемся на `willShow` ControlEvent:

```swift
navigationController?.rx.willShow
  .subscribe(onNext: { (controller, animated) in
    print("call from rx.willShow")
  })
  .disposed(by: disposeBag)
```

Теперь поэкспериментируем с местом, где установим сам делегат: `navigationController?.delegate = self`. 

Как и в случае с таблицей, поведение неочевидное.

Если я напишу `navigationController?.delegate = self` **перед** `willShow` в консоли я увижу оба лога:

```swift
//call from extension
//call from rx.itemSelected
```

Если я напишу `navigationController?.delegate = self` **после** `willShow` в консоли я увижу только лог из стандартной реализации делегата:

```swift
//call from extension
```

Решим эту проблему схожим способом. В RxCocoa уже есть `RxNavigationControllerDelegateProxy` класс. Можем добавить во `viewDidLoad` следующий код, чтобы проблема исчезла:

```swift
RxNavigationControllerDelegateProxy
  .installForwardDelegate(self,
                          retainDelegate: false,
                          onProxyForObject: navigationController!)
  .disposed(by: disposeBag)
```

Но я все же рекомендую добавить `setDelegate(_:)` метод в расширении и воспользоваться именно этим методом:

```swift
extension Reactive where Base: UINavigationController {
  func setDelegate(_ delegate: UINavigationControllerDelegate)
    -> Disposable {
      return RxNavigationControllerDelegateProxy
        .installForwardDelegate(delegate,
                                retainDelegate: false,
                                onProxyForObject: self.base)
  }
}
```

Тогда во `viewDidLoad` код будет выглядеть получше:

```swift
navigationController?.rx
  .setDelegate(self)
  .disposed(by: disposeBag)
```

### Кастомный делегат

Мы поговорили о проблемах настройки делегатов уже написанных в RxCocoa. Давайте напишем свой делегат, например для `CNContactPickerDelegate`.

Начнем мы с прокси класса `RxCNContactPickerDelegateProxy`:

```swift
/// For more information take a look at `DelegateProxyType`.
open class RxCNContactPickerDelegateProxy
  : DelegateProxy<CNContactPickerViewController, CNContactPickerDelegate>
  , DelegateProxyType
, CNContactPickerDelegate {

  /// Typed parent object.
  public weak private(set) var pickerController: CNContactPickerViewController?

  /// - parameter navigationController: Parent object for delegate proxy.
  public init(pickerController: ParentObject) {
    self.pickerController = pickerController
    super.init(parentObject: pickerController, delegateProxy: RxCNContactPickerDelegateProxy.self)
  }

  // Register known implementations
  public static func registerKnownImplementations() {
    self.register { RxCNContactPickerDelegateProxy(pickerController: $0) }
  }
}
```

Как по мне, так самый простой способ написать такого рода делегат - открыть любой существующий, например, `RxNavigationControllerDelegateProxy` и поменять `UINavigationController` на `CNContactPickerViewController`, а `UINavigationControllerDelegate` на `CNContactPickerDelegate`. 

Что бы воспользоваться этим делегатом, нам нужно добавить его в `Reactive` расширение. В нем нужно добавить, как минимум `delegate` свойство, ну и желательно реализовать сами делегатные методы, что бы расширение не было бесполезным:

```swift
extension Reactive where Base: CNContactPickerViewController {

  /// Reactive wrapper for `delegate`.
  ///
  /// For more information take a look at `DelegateProxyType` protocol documentation.
  public var delegate: DelegateProxy<CNContactPickerViewController, CNContactPickerDelegate> {
    return RxCNContactPickerDelegateProxy.proxy(for: base)
  }

  /// Reactive wrapper for delegate method `contactPicker(_:didSelect:)`.
  var didSelectContact: ControlEvent<CNContact> {
    let sel = #selector((CNContactPickerDelegate.contactPicker(_:didSelect:)! as (CNContactPickerDelegate) ->  (CNContactPickerViewController, CNContact) -> Void))
    let source: Observable<CNContact> = delegate.methodInvoked(sel)
      .map { arg in
        let contact = arg[1] as! CNContact
        return contact
    }
    return ControlEvent(events: source)
  }

  /// Reactive wrapper for delegate method `contactPicker(_:didSelect:)`.
  var didSelectContactProperty: ControlEvent<CNContactProperty> {
    let sel = #selector((CNContactPickerDelegate.contactPicker(_:didSelect:)! as (CNContactPickerDelegate) ->  (CNContactPickerViewController, CNContactProperty) -> Void))
    let source: Observable<CNContactProperty> = delegate.methodInvoked(sel)
      .map { arg in
        let contact = arg[1] as! CNContactProperty
        return contact
    }
    return ControlEvent(events: source)
  }
}
```

Придется немного повозиться с селекторами:

```swift
let sel = #selector((CNContactPickerDelegate.contactPicker(_:didSelect:)! as (CNContactPickerDelegate) -> (CNContactPickerViewController, CNContact) -> Void))

let sel = #selector((CNContactPickerDelegate.contactPicker(_:didSelect:)! as (CNContactPickerDelegate) -> (CNContactPickerViewController, CNContactProperty) -> Void))
```

Т.к. из objc оба метода выглядят идентично, мы сразу получаем ошибку `Ambiguous use of 'contactPicker(_:didSelect:)'`. Решить ее можно приведением к правильному типу, который не так легко записать с первой попытки:

```swift
(CNContactPickerDelegate) -> (CNContactPickerViewController, CNContact) -> Void)
(CNContactPickerDelegate) -> (CNContactPickerViewController, CNContactProperty) -> Void)
``` 

Мне хоть и не нравятся приведения типов, как в кложуре `map`, но с этим ничего не поделать, так уж реализован `methodInvoked(_:)` в `DelegateProxy`:

```swift
.map { arg in
        let contact = arg[1] as! CNContactProperty
        return contact
    }
```

И, возможно, мы захотим использовать известный нам метод:

```swift
extension Reactive where Base: CNContactPickerViewController {
  func setDelegate(_ delegate: CNContactPickerDelegate)
    -> Disposable {
      return RxCNContactPickerDelegateProxy
        .installForwardDelegate(delegate,
                                retainDelegate: false,
                                onProxyForObject: self.base)
  }
}
```

Пример проекта можно посмотреть в репозиторие [github](https://github.com/SergeyDukhovich/DelegateRxSample).