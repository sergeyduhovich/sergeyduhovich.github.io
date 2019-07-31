---
layout: single
title: "Улучшаем таблицы с RxDataSources"
date: 2019-06-21 12:00:00 +0300
categories: rxswift
---

В RxCocoa есть несколько стандартных методов биндинга Observable в UITableView или UICollectionView.

## RxCocoa

Допустим, перед нами стоит задача отобразить массив `Message` в UITableView:

```swift
enum Message {
  case text(String)
  case attributedText(NSAttributedString)
  case photo(UIImage)
  case location(lat: Float, lon: Float)
}
```

Для `text` и `attributedText` я буду использовать стандартные ячейки, а для `photo` и `location` я создам кастомные xib. А вот массив сообщений, который я хочу отобразить в таблице:

```swift
[
    .attributedText(NSAttributedString(string: "Blue text",
                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.blue])),
    .text("On the other hand, we denounce with righteous indignation and dislike men who are so beguiled and demoralized by the charms of pleasure of the moment"),
    .photo(UIImage(named: "rx-wide")!),
    .attributedText(NSAttributedString(string: "Red text",
                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.red])),
    .text("Another Message"),
    .location(lat: 37.334722, lon: -122.008889),
    .text("Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s"),
    .photo(UIImage(named: "rx-logo")!),
    .attributedText(NSAttributedString(string: "Green text",
                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.green])),
    .location(lat: 53.9, lon: 27.56667),
    .text("There are many variations of passages of Lorem Ipsum available, but the majority have suffered alteration in some form, by injected humour, or randomised words which don't look even slightly believable. If you are going to use a passage of Lorem Ipsum, you need to be sure there isn't anything embarrassing hidden in the middle of text."),
    .attributedText(NSAttributedString(string: "Yellow text",
                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.yellow])),
]
```

### Биндим массив к таблице

В RxCocoa к медодам для работы с таблицами, объявленным в файле  `UITableView+Rx.swift`, есть комментарии-примеры их использования:

#### 1.

```swift
 let items = Observable.just([
     "First Item",
     "Second Item",
     "Third Item"
 ])

 items
     .bind(to: tableView.rx.items(cellIdentifier: "Cell", cellType: UITableViewCell.self)) { (row, element, cell) in
        cell.textLabel?.text = "\(element) @ row \(row)"
     }
     .disposed(by: disposeBag)
```

Этот метод подойдет в случаях, когда мы используем одинаковые ячейки для всех элементов массива. Ни идентификатор ячейки, ни класс/xib ячейки не может быть изменен в замыкании. В нем мы только донастраиваем конечный вид ячейки перед ее появлением в таблице.

#### 2.

```swift
 let items = Observable.just([
     "First Item",
     "Second Item",
     "Third Item"
 ])

 items
 .bind(to: tableView.rx.items) { (tableView, row, element) in
     let cell = tableView.dequeueReusableCell(withIdentifier: "Cell")!
     cell.textLabel?.text = "\(element) @ row \(row)"
     return cell
 }
 .disposed(by: disposeBag)
```

Этот метод более гибкий, т.к. ответственность за создание ячейки лежит полностью на мне. Используя этот метод, в таблице можно использовать ячейки с разными идентификаторами и классами/xib ячеек.

Задача, которую я описал выше, может быть реализована как в примере ниже. Статические методы конфигурируют ячейки.

```swift
messages
  .bind(to: tableView.rx.items) { (table: UITableView, index: Int, message: Message) in
    guard let cell = table.dequeueReusableCell(withIdentifier: message.identifier.rawValue) else { return UITableViewCell() }
    switch message {
    case let .text(message):
      return ViewController.configure(text: message, cell: cell)
    case let .attributedText(attributed):
      return ViewController.configure(attributed: attributed, cell: cell)
    case let .photo(photo):
      return ViewController.configure(photo: photo, cell: cell)
    case let .location(lat: lat, lon: lon):
      return ViewController.configure(lat: lat, lon: lon, cell: cell)
    }
  }
.disposed(by: disposeBag)
```

И сама таблица, отображающая разные типы ячеек:

![rxcocoa-table-sample](http://dukhovich.by/assets/images/articles/10/rxcocoa-table-sample.png)

[Ссылка на коммит с текущей реализацией.](https://github.com/SergeyDukhovich/RxDataSourcesSample/tree/326ca6e6a182d54a184edbec1a1fd9bcf288a733)

Но если RxCocoa справляется с задачей, зачем тогда нужен RxDataSources?

## RxDataSources

### Секции

Во-первых, RxDataSources предоставляет возможность отображать данные в таблице с несколькими секциями, в то время как RxCocoa отображает весь массив только в первой секции по-умолчанию. Давайте модифицируем немного исходную задачу, и разобьем 12 сообщений на 3 секции, по 4 сообщения в каждом.

Фреймворк предоставляет 2 типа dataSource для таблиц: `RxTableViewSectionedReloadDataSource` и `RxTableViewSectionedAnimatedDataSource`, и аналогичные для коллекций. Эти типы требуют типов-секций, которые реализуют `SectionModelType` протокол. В Differentiator, который является зависимостью к RxDataSources, есть готовая `SectionModel` структура, в которой нужно указать тип секции и тип элементов.

`Observable<[Message]>` изменится на `Observable<[SectionModel<String, Message>]>`.

У вью контроллера появится свойство `dataSource`:

```swift
 let dataSource = RxTableViewSectionedReloadDataSource<SectionModel<String, Message>>(configureCell: { dataSource, table, indexPath, message in
    guard let cell = table.dequeueReusableCell(withIdentifier: message.identifier.rawValue) else { return UITableViewCell() }
    switch message {
    case let .text(message):
      return ViewController.configure(text: message, cell: cell)
    case let .attributedText(attributed):
      return ViewController.configure(attributed: attributed, cell: cell)
    case let .photo(photo):
      return ViewController.configure(photo: photo, cell: cell)
    case let .location(lat: lat, lon: lon):
      return ViewController.configure(lat: lat, lon: lon, cell: cell)
    }
  })
```

Обновленное свойство `messages`: 

```swift
private var messages = Observable<[SectionModel<String, Message>]>.just([
    SectionModel<String, Message>(model: "1",
                                  items: [
                                    .attributedText(NSAttributedString(string: "Blue text",
                                                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.blue])),
                                    .text("On the other hand, we denounce with righteous indignation and dislike men who are so beguiled and demoralized by the charms of pleasure of the moment"),
                                    .photo(UIImage(named: "rx-wide")!),
                                    .attributedText(NSAttributedString(string: "Red text",
                                                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.red]))
      ]),
    SectionModel<String, Message>(model: "2",
                                  items: [
                                    .text("Another Message"),
                                    .location(lat: 37.334722, lon: -122.008889),
                                    .text("Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s"),
                                    .photo(UIImage(named: "rx-logo")!),
                                    .attributedText(NSAttributedString(string: "Green text",
                                                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.green]))
      ]),
    SectionModel<String, Message>(model: "3",
                                  items: [
                                    .location(lat: 53.9, lon: 27.56667),
                                    .text("There are many variations of passages of Lorem Ipsum available, but the majority have suffered alteration in some form, by injected humour, or randomised words which don't look even slightly believable. If you are going to use a passage of Lorem Ipsum, you need to be sure there isn't anything embarrassing hidden in the middle of text."),
                                    .attributedText(NSAttributedString(string: "Yellow text",
                                                                       attributes: [NSAttributedString.Key.foregroundColor : UIColor.yellow]))
      ])
    ])
```

Несмотря на то, что мы используем секции, таблица внешне ничем не отличается от таблицы из первого примера. Что бы отобразить заголовки секций, нужно добавить следующий код (перед биндингом таблицы):

```swift
    dataSource.titleForHeaderInSection = { dataSource, index in
      return dataSource.sectionModels[index].model
    }
```

И сама таблица, отображающая разные типы ячеек, помещенные в секции:

![rxdatasources-table-sample](http://dukhovich.by/assets/images/articles/10/rxdatasources-table-sample.png)

[Ссылка на коммит с текущей реализацией.](https://github.com/SergeyDukhovich/RxDataSourcesSample/tree/c3c55b3a7e484d2b067f182ffee492f99e22a67b)

### Animation 

Второй особенностью RxDataSources является возможность анимировано обновлять ячейки, каждый раз, когда Observable эмитит `next` ивент. В предыдущей реализации таблица использовала метод `reloadData` на каждый `next` ивент.

Для анимаций нужно реализовать еще парочку протоколов нашим типам. `IdentifiableType` для `Message` может выглядеть следующим образом:

```swift
extension Message: IdentifiableType {
  var identity : String {
    switch self {
    case let .text(text):
      return "text_\(text)"
    case let .attributedText(text):
      return "attributed_\(text)"
    case let .location(lat: lat, lon: lon):
      return "\(lat)_\(lon)"
    case let .photo(image):
      guard let data = image.pngData() else { return "image" }
      return String(data.hashValue)
    }
  }
}
```

Но тут нужно взять паузу и хорошенько подумать, над каким типом задачи мы работаем, и что именно мы хотим реализовать. Т.к. например для чата такая реализация не подойдет. Т.к. текстовая ячейка с "привет" от Пети будет конфликтовать с "привет"-ячейкой от Васи. Несмотря на то, что ошибок компиляции в коде нет, на лицо логическая ошибка. А в случае, когда на экране нужно будет отобразить эти 2 ячейки, то приложение крешнется.

Итак, что нужно обновить в проекте для подключения анимаций?

* Изменить `RxTableViewSectionedReloadDataSource` на`RxTableViewSectionedAnimatedDataSource`;
* Изменить `SectionModel` на `AnimatableSectionModel`;
* Реализовать `IdentifiableType` протокол у `Message`;
* Реализовать `Equatable` протокол у `Message`;

[Ссылка на коммит с текущей реализацией.](https://github.com/SergeyDukhovich/RxDataSourcesSample/tree/1cd8dd53296739df4871978dea7bd54a2be3840a)

### NSObject

Время от времени, в качестве источника у нас будет массив объектов класса, унаследованного от `NSObject`, нам придется реализовывать у него `IdentifiableType`.

В качестве примера, реализация `MessageObject`:

```swift
class MessageObject: NSObject {
  let message: Message
  let messageId: String

  init(message: Message, messageId: String) {
    self.message = message
    self.messageId = messageId
  }

  override func isEqual(_ object: Any?) -> Bool {
    guard let obj = object as? MessageObject else { return false }
    return messageId == obj.messageId
  }
}
```

`Message` все тот же enum, который использовался выше.

![rxdatasources-table-sample](http://dukhovich.by/assets/images/articles/10/rxdatasources-animated-implementation.gif)

[Ссылка на коммит с текущей реализацией.](https://github.com/SergeyDukhovich/RxDataSourcesSample/tree/115397f2bbc86986e7454a01a00e45999228b807)
