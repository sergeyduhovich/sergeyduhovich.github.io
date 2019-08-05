---
title: "Пишем реактивное расширение"
date: 2019-08-05 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

## Что понимать под реактивным расширением?

Технически - это просто `extension`, который позволяет, не внося изменений в текущую реализацию класса, дописать Observable или Observer функционал. Например для сущности класса `SomeClass` дописав `.rx.`, мы можем получить доступ к методу `someObservable()`, который объявлен следующим образом:

```swift
extension Reactive where Base: SomeClass {
  func someObservable() -> Observable<Int> {
    return Observable<Int>...
  }
}
```

Правда для того, чтобы строчка `someClass.rx.someObservable()` скомпилилась (в случае, когда `SomeClass` не является потомком `NSObject`), нужно реализовать следующий протокол:

```swift
extension SomeClass: ReactiveCompatible {}
```

Обычно в `extension Reactive` помещаются переменные и методы, которые добавляют реактивную функциональность. Несмотря на то, что в swift нет ограничений на возвращаемый тип в рамках extension, в добавлении нереактивных методов и переменных мало смысла. Более того, это может запутать пользователя данного расширения. Каждый раз, когда я пишу `.rx.` после какой-нибудь переменной, я ожидаю, что все подсказки из автокомплита реактивные. Поэтому хорошая практика - держать в этом расширении только то, что обладает поведением Observable, Observer или сразу обеим.   

### UITextField

RxCocoa  содержит большое разнообразие готовых расширений для классов из UIKit. Возьмем для примера расширение написанное в `UITextField+Rx.swift`, которое добавляет `text` Observable & Observer поведение для филдов. Использовать его можно следующим образом:

```swift
//as Observable
loginField.rx.text.orEmpty
  .bind(to: viewModel.loginRelay)
  .disposed(by: disposeBag)
	
//as Observer
viewModel.loginRelay
  .bind(to: loginField.rx.text)
  .disposed(by: disposeBag)
```

Чтобы посмотреть список доступных реактивных свойств для сущности какого-нибудь типа, просто добавляем после нее `.rx.` и выбираем из списка, предложенного в автокомплите.

Наличие расширений помогает писать однотипный, легко узнаваемый rx-код, что само по себе является одним из [плюсов RxSwift]({% post_url 2019-04-26-rxswift-introduction %}).

## Расширения не обязательны

Использование и написание расширений не является чем-то обязательным при работе с RxSwift. Однако я рекомендую хотя бы ознакомиться со списком доступных методов и переменных для UIKit классов в RxCocoa. Как минимум - это поднимет Ваш уровень владения этим инструментом.

Допустим, я не хочу, или не знаю, как пользоваться RxCocoa, но при этом уже пишу на RxSwift. Посмотрим, как поменяется пример выше с `loginField`:

```swift
override func viewDidLoad() {
  super.viewDidLoad()
  loginField.addTarget(self,
	                   action: #selector(textChanged(sender:)),
	                   for: [.allEditingEvents, .valueChanged])
	
  //as Observer
  viewModel.loginRelay
    .subscribe(onNext: { [weak self] text in
      self?.loginField.text = text
    })
    .disposed(by: disposeBag)
}

//as Observable
@objc private func textChanged(sender: Any) {
  if let textField = sender as? UITextField,
    textField == loginField,
    let text = loginField.text {
      viewModel.loginRelay.accept(text)
  }
}
```

Кода стало немного больше, и код, отвечающий за Observable часть, перестал быть декларативным, коротким, узнаваемым, в rx-стиле, потому что Observable часть разделилась на 2 блока кода.

## RxCocoa расширения

Если задаться целью узнать, каким образом реализованы RxCocoa расширения, то можно узнать о 3 типах:

* Binder;
* [ControlEvent]({% post_url 2019-05-17-traits %});
* [ControlProperty]({% post_url 2019-05-17-traits %});

Понимание этих типов поможет определиться, какой из них лучше подойдет в случае, когда понадобится написать расширение.

### Binder (только Observer)

Пожалуй, самый простой способ добавить поведение Observer. Посмотрите, как просто было дописано свойство `isAnimating` для класса `UIActivityIndicatorView`:

```swift
extension Reactive where Base: UIActivityIndicatorView {
  public var isAnimating: Binder<Bool> {
    return Binder(self.base) { activityIndicator, active in
      if active {
        activityIndicator.startAnimating()
      } else {
        activityIndicator.stopAnimating()
      }
    }
  }
}
```

Или `isEnabled` для `UIAlertAction`:

```swift
extension Reactive where Base: UIAlertAction {
  public var isEnabled: Binder<Bool> {
    return Binder(self.base) { alertAction, value in
      alertAction.isEnabled = value
    }
  }
}
```

Довольно просто, не правда ли? 

И само описание структуры:

* Кложура выполняется только для `next` ивентов, `completed` игнорируются, `error` в дебаге крешится с `fatalError`, в релизе логируется;
* Можно указать шедулер (по-умолчанию `MainScheduler.instance`) на котором будет выполняться кложура;
* Сама структура `Binder` не держит объект, с которым она работает, сильной ссылкой.

### ControlEvent (только Observable)

Еще один тип, доступный в RxCocoa, обладает следующими свойствами:

* Никогда не отправляет ошибку;
* Не отправляет последнее значение в момент подписки;
* Отправляет `completed` ивент, когда UI элемент, на котором он базируется, удаляется из памяти;
* Ивенты доставляются на `MainScheduler.instance`.

ControlEvent можно создать, передав в инициализатор другой Observable. Но нужно передавать такой Observable, который обладает всеми вышеперечисленными свойствами. В противном случае можно получить расширение, которое будет работать не так, как от него ожидают пользователи.

Но совсем не обязательно всегда продумывать и создавать Observable, сверяясь со списком свойств. Если расширение пишется для `UIControl` из стандартного фреймворка UIKit, то можно попросту воспользоваться хелпером для создания `ControlEvent` лишь указав массив `UIControlEvents` на которые должно реагировать расширение. Например так, как это было сделано для кнопки:

```swift
extension Reactive where Base: UIButton {
  public var tap: ControlEvent<Void> {
    return controlEvent(.touchUpInside)
  }
}
```

В случаях, когда нет UIControl-а, приходится создавать `source` Observable, учитывая список свойств выше. По такому принципу написано расширение для `UICollectionView` - `itemSelected`. Исходный Observable был получен с использованием [DelegateProxy]({% post_url 2019-07-26-delegates %}):

```swift
public var itemSelected: ControlEvent<IndexPath> {
  let source = delegate.methodInvoked(#selector(UICollectionViewDelegate.collectionView(_:didSelectItemAt:)))
    .map { a in
      return try castOrThrow(IndexPath.self, a[1])
    }  
  return ControlEvent(events: source)
}
```

### ControlProperty (Observer + Observable)

`ControlProperty` тип обладает свойствами как Observer-а, так и Observable. Тип написан таким образом, что у него есть ссылки на обе сущности, `values: Observable` и `valueSink: AnyObserver`. Обладает он следующими свойствами:

* Никогда не отправляет ошибку;
* Отправляет последнее значение в момент подписки (`shareReplay(1)`);
* Отправляет `completed` ивент, когда UI элемент, на котором он базируется, удаляется из памяти;
* Ивенты доставляются на `MainScheduler.instance`.

Observable последовательность **должна** отправить `next` ивент со значением контрола в момент инициализации и `next` ивент как реакция на пользовательский ввод. Программные изменения, например пришедшие из Observer **не должны** эмитить `next` ивент из Observable.

Как и для ControlEvent, есть хелпер, который помогает создавать ControlProperty для дочерних классов UIControl - `func controlProperty(editingEvents:getter:setter:)`

Вот так выглядит расширение `value` для `UISlider`:

```swift
extension Reactive where Base: UISlider {
  public var value: ControlProperty<Float> {
    return base.rx.controlPropertyWithDefaultEvents(
      getter: { slider in
        slider.value
      }, setter: { slider, value in
        slider.value = value
      }
    )
  }
}
```

В других случаях, без использования UIControl, нужно передать в инициализатор и Observable и Observer - `init(values:valueSink:)`.

## Расширение для кастомного UIControl

Разных реализаций [UIControl](https://developer.apple.com/documentation/uikit/uicontrol) достаточно много. Но как бы они не отличались внешне или стилем написания, их объединяют вызовы типа `sendActions(for: .valueChanged)`, или `sendActions(for: . touchUpInside)`, где-то после обработки жестов пользователя. Пару хороших примеров можно найти на сайте рэя - [крутелка](https://www.raywenderlich.com/5294-how-to-make-a-custom-control-tutorial-a-reusable-knob) и [кастомный слайдер](https://www.raywenderlich.com/2297-how-to-make-a-custom-control-tutorial-a-reusable-slider). Достаточно много разных контролов можно найти на GitHub.

### Используем ControlProperty

Если контрол построен вокруг некоторого значения, например громкость, и этот контрол использует ивент `.valueChanged`, то его будет относительно просто расширить свойством `ControlProperty`. 

Если по какой-то причине в реализации ивент `.valueChanged` не отправляется, что само по себе немного странно, то ControlProperty придется проинициализировать при помощи source Observable, соблюдая "контракт" ControlProperty.

### Используем ControlEvent

Если контрол не использует ивент `.valueChanged`, и не будет использоваться в качестве Observer-а, а только как Observable, например, `itemSelected: ControlEvent<IndexPath>`, `didBeginEditing: ControlEvent<()>`, тогда пишем расширение используя `ControlEvent`.

## Для остального UI

В случае, когда расширение пишется не для UIControl, также можно использовать перечисленные выше типы в зависимости от желаемого поведения - `ControlEvent`, `Binder` or `ControlProperty`.

Возможно будет достаточным использование `Driver` и `Signal`. 

## Расширения из комьюнити

Расширения не обязательно писать только на UI, и не обязательно ограничиваться только перечисленными типами выше. Можно использовать все доступные [traits]({% post_url 2019-05-17-traits %}), которые подходят по ситуации.

Еще одним плюсом RxSwift является его комьюнити, в котором можно найти довольно много всяких расширений нереактивных библиотек. Давайте посмотрим на типы, которые были использованы в некоторых из них.

### RxAlamofire

Здесь можно найти 3 класса, к которым был дописан функционал:

* URLSession;
* SessionManager;
* DataRequest;
* Request.

Расширения возвращают обычные Observable. Есть пару вхождений `Observable.create`, которые трансформируют асинхронные кложуры в реактивные последовательности.

### RxGesture

Это расширение тоже про UI, написано с использованием ControlEvent и ControlProperty.

### RxKeyboard

В RxKeyboard используется trait `Driver`. Он тоже подходит для UI.

Переменная `frame` `Driver<CGRect>` базируется на `BehaviorRelay`, а другие переменные просто трансформируют ее значение.

### RxRealm

Здесь присутствует базовый `ObserverType` - `RealmObserver<Element>` который используется в написании реактивного кода.