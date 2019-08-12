---
title: "Совмещение Observables"
date: 2019-08-10 12:00:00 +0300
categories: rxswift
toc: true
toc_sticky: true
---

Довольно часто у нас есть два и более Observable, которые нужно объединить в один, используя то или иное поведение. В этой статье я попробую описать некоторые операторы, которые используются для этих целей.

## Merge

Оператор позволяет совместить все ивенты из разных источников Observable в один. Тип должен быть одинаковый у всех Observable, он будет использован в финальном Observable.

* Реакция на `completed`: Если 2 из 3 Observable завершатся нормально, т.е. отправкой `completed` ивента, финальный Observable продолжит работу.
* Реакция на `error`: Если любой из Observable завершится с ошибкой, т.е. отправкой `error` ивента, финальный Observable завершится тоже. 

Всякий раз, когда финальный Observable эмитит ивент, мы не можем определить, из какого источника он был отправлен.

![merge marble diagram](http://dukhovich.by/assets/images/articles/16/merge.png)

Пример использования:

```swift
let arrayOfObservable: [Observable<String>]
  = [first, second, third]

Observable.merge(arrayOfObservable)
  .subscribe(onNext: { str in
    print(str)
  })
  .disposed(by: disposeBag)
```

Потенциальная проблема может возникнуть при совмещении большого количества источников Observable, каждый из которых представляет из себя один или неколько URL запросов. Все они будут выполнены одновременно, с большой вероятностью, часть из них будет завершена по тайм-ауту.

## CombineLatest

Всякий раз, когда какой-нибудь из источников эмитит `next` ивент, финальный Observable проверяет наличие хотя бы одного ивента у каждого источника и, если условие выполняется, забирает последний ивент от каждого источника и использует их в качестве своего `next` ивента. Observable могут иметь разные типы в случае использования метода, в котором не используется массив.

* Реакция на `completed`: Если 2 из 3 Observable завершатся нормально, т.е. отправкой `completed` ивента, финальный Observable продолжит работу. Но, если завершившийся Observable не отправил ни одного `next` ивента, то финальный Observable будет вести себя как `never` оператор.
* Реакция на `error`: Если любой из Observable завершится с ошибкой, т.е. отправкой `error` ивента, финальный Observable завершится тоже. 

Всякий раз, когда финальный Observable эмитит ивент, мы можем соотнести источник и его последний ивент по индексу в массиве или tuple.

![combine latest marble diagram](http://dukhovich.by/assets/images/articles/16/combinelatest.png)

```swift
Observable
  .combineLatest(arrayOfObservable) {
    return $0.reduce("", { $0 + " \($1)" })
  }
  .subscribe(onNext: { str in
    print(str)
  })
  .disposed(by: disposeBag)

Observable
  .combineLatest(first, second, third)
  .subscribe(onNext: { (s1, s2, s3) in
    print("\(s1) \(s2) \(s3)")
  })
  .disposed(by: disposeBag)

Observable.combineLatest(arrayOfObservable)
  .subscribe(onNext: { array in
    print(array.reduce("", { $0 + " \($1)" }))
  })
  .disposed(by: disposeBag)
```

Есть 2 варианта создания оператора. В одном из них можно использовать от 2 до 8 Observable, во втором используется массив Observable.

Дополнительно можно сразу трансформировать массив/tuple, передав `resultSelector` в метод, иначе next ивент будет содержать массив/tuple. 

Очень распространенная проблема, когда один из Observable не эмитит ничего, тогда финальный Observable работает не так как от него ожидается.

## WithLatestFrom

Этот оператор работает только с 2 Observable. Когда первый источник эмитит ивент, в том случае, когда второй источник отправил хотя бы один ивент, финальный Observable повторяет второй ивент, или их комбинацию, в противном случае финальный Observable не эмитит ничего.

* Реакция на `error`: Если любой из Observable завершится с ошибкой, т.е. отправкой `error` ивента, финальный Observable завершится тоже. 

* Реакция на `completed`: Если 1й Observable завершается нормально, т.е. отправкой `completed` ивента, финальный Observable тоже завершается. 

![with latest from marble diagram](http://dukhovich.by/assets/images/articles/16/withlatestfrom1.png)

* Реакция на `completed`: Если 2й Observable завершается нормально, т.е. отправкой `completed` ивента, финальный Observable продолжает работу, но если 2й не отправил ни одного ивента, то финальный Observable будет работать как never оператор.

![with latest from marble diagram](http://dukhovich.by/assets/images/articles/16/withlatestfrom2.png)

Довольно часто возникает проблема с этим оператором по той же причине, что и у `combineLatest` оператора. Если 2й Observable не отправил ни одного ивента до того момента, как 1й начал что-то отправлять, то эти ивенты будут проигнорированы. На обеих диаграммах нет `1*` ивента.

## StartWith

Проблемы, упомянутые в `combineLatest` и `withLatestFrom`, решает следующий оператор, который эмитит элементы из последовательности,  до того как основной Observable начнет что-нибудь эмитить.

![start with marble diagram](http://dukhovich.by/assets/images/articles/16/startwith.png)

## Zip

Комбинирует `next` ивенты из всех источников по индексам, первый `next` одного источника с первым `next` второго источника, второй ивент со вторым и т.д.

![zip marble diagram](http://dukhovich.by/assets/images/articles/16/zip.png)

## Concat

Выстраивает выполнение всех Observable друг за другом. Как только нормально заканчивается предыдущая (`completed` ивентом), запускается следующая Observable. Если какая-нибудь из Observable не заканчивается `completed` ивентом, следующая не запускается.

![concat marble diagram](http://dukhovich.by/assets/images/articles/16/concat.png)

Может пригодится в том случае, когда нужно отправить очень много URL запросов друг за другом.

## FlatMap

`flatMap` оператор обычно используется немного в другом контексте, но он также может быть использован, когда нужно выполнить цепочку из Observable друг за дргуом, передавая результат выполнения предыдущего в следующий. Один из возможных сценариев - зависимые друг от друга API запросы:

* Сначала получаем список ресторанов;
* Потом загружаем отзывы о первом ресторане из списка;
* Загружаем профиль пользователя из первого отзыва.

```swift
api
  .restaurantList()
  .flatMap { [weak self] restaurants -> Observable<[Review]> in
    guard let self = self,
      let id = restaurants.first?.id
      else { return .empty() }
    return self.api.restaurantReviews(by: id)
  }
  .flatMap { [weak self] reviews -> Observable<User> in
    guard let self = self,
      let id = reviews.first?.user.id
      else { return .empty() }
    return self.api.user(by: id)
  }
  .subscribe(onNext: { user in
    print(user)
  })
  .disposed(by: disposeBag)
```

## Amb

Оператор работает с двумя и более Observable источниками. Полностью повторит тот источник, который первым начнет эмитить ивенты. Остальные источники будут проигнорированы.

Не встречал применение этого оператора на правктике.

![amb marble diagram](http://dukhovich.by/assets/images/articles/16/amb.png)