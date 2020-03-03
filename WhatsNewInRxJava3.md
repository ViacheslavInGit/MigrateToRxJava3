# What new in RxJava3

## info
в rxJava2 больше не будет получать обновление операторов / доки

RxJava3 лежит в отдельном пакете, т.ч можно юзать вместе со 2й версией

## affects behavior

полностью пофиксить undeliverable errors во 2й	версии не вышло, зато вышло в 3й версии.
Теперь они все должны попадать в RxJavaPlugins.onError() и ловится через RxJavaPlugins.setErrorHandler()

---

**Connectable Sources**

во 2й версии если такой источник прерывался вместо дисонекта, новые подписчики не получали источаемые объекты

в 3й версии появился метод reset(), который вернет источник в прежнее состояние, если он был прерван. Иначе не возымеет накакого эффекта

---

**Flowable.publish**

во 2й версии мог "потерять" элемент если от него на время отписались.

в 3й версии потребитель получит элементы даже если подписался чуть позже чем они были излучены

---
**Handle null**

В 3й версии, если у  PublishProcessor | BehaviorProcessor | MulticastProcessor вызвать offer(null)
вместо вызова onError будет бросаться NullPointerException. Так что теперь можно обработать ошибку и не убить цепочку

пример:
```kotlin
val v: String?
val processor = PublishProcessor.create();
try {
   processor.offer(v);
} catch (NullPointerException expected) {
   Timber.d("v not inited!")
}
// we can still use processor
```

---

**MulticastProcessor offer** и on...

 во 2й версии можно было вызывать свободно, даже когда происходило слияние, что могло приводить к проблемам.


В 3й версии если неправильно использовать этот процессор то рх просто бросит IllegalStateException (т.е сохранить ссылку на него и вызвать offer() когда цепочка запущена)

---


**groubBy** у Flowable/Observable возвращает Flowable/Observable<GroupedObservable>

во 2й версии можно подписываться на каждый GroupedObservable отдельно, что плохо(группы будут игнорироватся, отписка остановит источник, могут быть утечки ресурсов)

в 3й версии можно подписаться только синхронно, иначе цепочка будет считатся "abandoned" (заброшенной) и будет terminated

У .window() теперь такое же поведение как и .groupBy()

---

**Эти ператоры в 3й версии вместо IndexOutOfBoundsException бросают IllegalArgumentException**:

skip

skipLast

takeLast

takeLastTimed


---
**fromRunnable & fromAction**

В 2й версии fromRunnable & fromAction вели себя не так как остальные from... . Т.е если отменить последовательность, runnable|action все-равно выполнятся

В 3й версии они не выполнятся.

пример:
```kotlin
import io.reactivex.Completable as Completable2
import io.reactivex.rxjava3.core.Completable as Completable3


fun main() {

    val completable2 = Completable2.fromRunnable { println("completable 2 done") }
    val completable3 = Completable3.fromRunnable { println("completable 3 done") }

    completable2.test(true)  		 // cancelled = true
    completable3.test(true)  		 // dispose   = true
}

//output:
//completable 2 done
```
---

**Using operator**

имеет перегрузку где есть параметр eager, если он true, то ресурс освободится перед terminal event, иначе после

Но во 2й версии, вне зависимости от его значения ресурс освобождался перед terminal event

В 3й версии значение eager влияет на время освобождения ресурса

---

## does not affect behavior

---

**Functions**

rxjava2 юзала свои функциональные интерфейсы из io.reactivex.functions

rxjava3 тоже юзает свои кастомные функциональные интерфейсы, несмотря на возможность использовать их из java 8. Однако к ним добавлена аннотация @FunctionalInterface

Еще благодаря 8й джаве типам аргументов можно добавлять nullability annotations


Function в rx2 бросает **Exception**

Function в rx3 бросает **Throwable**

---


В rx3 новый тип: **io.reactivex.rxjava3.functions.Supplier**,

который многие операторы теперь используют вместо java.util.concurrent.Callable. Их отличие в том что Supplier возвращает **@NonNull** параметр, и **бросает Throwable вместо Exception**


---

**Конвертеры**

Поменялись конвертеры
если в rx2 Flowable.to(Function<Flowable<T>, R>)

то   в rx3 Flowable.to(FlowableConverter<T, R>)

Лямбды выглядат одинаково, отличие в том что, например, для FlowableConverter аргумент Flowable<T>, параметры @NonNull и он не бросает исключения

---


Метод **empty()**, для создания нового диспосабла перемещен из Disposables в Disposable. Disposables в 3й версии нет

---

**DisposableContainer** (родитель CompositeDisposable)  в 3й версии попал в public API

---

**Эксперементальные операторы из из rx 2.2.x есть в апи rx3**.

 Среди них:

flowable    .dematerialize

observable  .dematerialize

maybe       .materialize    .doOnTerminate

single      .dematerialize  .materialize

completable .materialize    .delaySubscription

---

### api additions

У у Flowable & Observable появились

concatMap       with Scheduler

blockingForEach with buffer size

весь список https://github.com/ReactiveX/RxJava/wiki/What's-different-in-3.0#api-additions


### api renames

некоторые startWith -> startWithArray, startWithIterable and startWithItem

onErrorResumeNext -> onErrorResumeWith

single.equeals -> single.sequenceEqual

zipIterable -> zip

Maybe.flatMapSingleElement ->  Maybe.flatMapSingle

весь список https://github.com/ReactiveX/RxJava/wiki/What's-different-in-3.0#api-renames

### api removals

Maybe.toSingle                                          // use .defaultIfEmpty(T)

Flowable|Observable .subscribe(4 arguments)		// use .doOnSubscribe()

Single.toCompletable()					// use .ignoreElement()

Completable.blockingGet()                               // use .blockingAwait()


весь список https://github.com/ReactiveX/RxJava/wiki/What's-different-in-3.0#api-removals


## migrate
Классы из разных версий можно кастить друг к другу используя библиотеку
https://github.com/akarnokd/RxJavaBridge#rxjavabridge
