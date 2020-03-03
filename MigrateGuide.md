# Migrate to RxJava3 guide

### Постепенный переход со второй версии на третью успростит RxJavaBridge
https://github.com/akarnokd/RxJavaBridge#rxjamvabridge

### Если проект на MVP и BasePresenter наследуется от Execution, то для паралельного использования обеих версий:
функций **subscribe** & **subscribeWithProgress** можно проделать такие шаги:

#### 1) В Execution заменить все импорты 2й версии rxJava на 3ю

     паттерн такой:

```kotlin
import io.reactivex.X
import io.reactivex.rxjava3.core.X  
```

#### 2) Унаследовать BasePresenter от ExecutionRx2Adapter


```kotlin
import hu.akarnokd.rxjava3.bridge.RxJavaBridge
import io.reactivex.Completable as CompletableRx2
import io.reactivex.Observable as ObservableRx2
import io.reactivex.Scheduler as SchedulerRx2
import io.reactivex.Single as SingleRx2
import io.reactivex.disposables.Disposable as DisposableRx2
import io.reactivex.rxjava3.core.Scheduler as SchedulerRx3
import io.reactivex.rxjava3.disposables.Disposable as DisposableRx3

abstract class ExecutionRx2Adapter(
    backgroundScheduler: SchedulerRx3,
    foregroundScheduler: SchedulerRx3
) : Execution(backgroundScheduler, foregroundScheduler) {

    private val showProgressMapper = ShowProgressMapper()

    constructor(
        backgroundScheduler: SchedulerRx2,
        foregroundScheduler: SchedulerRx2
    ) : this(
        backgroundScheduler = RxJavaBridge.toV3Scheduler(backgroundScheduler),
        foregroundScheduler = RxJavaBridge.toV3Scheduler(foregroundScheduler)
    )

    protected fun <T : Any> subscribe(
        upstream: ObservableRx2<T>,
        onNext: (T) -> Unit
    ) {
        subscribe(
            upstream = RxJavaBridge.toV3Observable(upstream),
            onNext = onNext
        )
    }

    protected fun <T : Any> subscribe(
        upstream: SingleRx2<T>,
        onSuccess: (T) -> Unit
    ) {
        subscribe(
            upstream = RxJavaBridge.toV3Single(upstream),
            onSuccess = onSuccess
        )
    }

    protected fun subscribe(
        upstream: CompletableRx2,
        onComplete: () -> Unit
    ) {
        subscribe(
            upstream = RxJavaBridge.toV3Completable(upstream),
            onComplete = onComplete
        )
    }

    protected fun <T : Any> subscribeWithProgress(
        upstream: ObservableRx2<T>,
        onNext: (T) -> Unit,
        showProgress: (DisposableRx2) -> Unit = { showProgress() },
        hideProgress: () -> Unit = this::hideProgress
    ) {
        subscribeWithProgress(
            upstream = RxJavaBridge.toV3Observable(upstream),
            onNext = onNext,
            showProgress = showProgressMapper(showProgress),
            hideProgress = hideProgress
        )
    }

    protected fun <T : Any> subscribeWithProgress(
        upstream: SingleRx2<T>,
        onSuccess: (T) -> Unit,
        showProgress: (DisposableRx2) -> Unit = { showProgress() },
        hideProgress: () -> Unit = this::hideProgress
    ) {
        subscribeWithProgress(
            upstream = RxJavaBridge.toV3Single(upstream),
            onSuccess = onSuccess,
            showProgress = showProgressMapper(showProgress),
            hideProgress = hideProgress
        )
    }

    protected fun subscribeWithProgress(
        upstream: CompletableRx2,
        onComplete: () -> Unit,
        showProgress: (DisposableRx2) -> Unit = { showProgress() },
        hideProgress: () -> Unit = this::hideProgress
    ) {
        subscribeWithProgress(
            upstream = RxJavaBridge.toV3Completable(upstream),
            onComplete = onComplete,
            showProgress = showProgressMapper(showProgress),
            hideProgress = hideProgress
        )
    }
}

class ShowProgressMapper : ((DisposableRx2) -> Unit) -> ((DisposableRx3) -> Unit) {

    override fun invoke(showProgressRx2: (DisposableRx2) -> Unit) =
        object : (DisposableRx3) -> Unit {
            override fun invoke(disposableRx3: DisposableRx3) {
                RxJavaBridge
                    .toV2Disposable(disposableRx3)
                    .also(showProgressRx2)
            }
        }
}

```


#### 3) Добавить зависимости

```kotlin
import com.nixsolutions.uflowers.di.qualifier.execution.rx3.Background
import com.nixsolutions.uflowers.di.qualifier.execution.rx3.Foreground
import dagger.Module
import dagger.Provides
import io.reactivex.Scheduler as SchedulerRx2
import io.reactivex.android.schedulers.AndroidSchedulers as AndroidSchedulersRx2
import io.reactivex.rxjava3.android.schedulers.AndroidSchedulers as AndroidSchedulersRx3
import io.reactivex.rxjava3.core.Scheduler as SchedulerRx3
import io.reactivex.rxjava3.schedulers.Schedulers as SchedulersRx3
import io.reactivex.schedulers.Schedulers as SchedulersRx2

@Module object ExecutionModule {

    @Provides @JvmStatic @Background
    fun provideBackgroundSchedulerRx2(): SchedulerRx2 = SchedulersRx2.computation()

    @Provides @JvmStatic @Foreground
    fun provideForegroundSchedulerRx2(): SchedulerRx2 = AndroidSchedulersRx2.mainThread()

    @Provides @JvmStatic @Background
    fun provideBackgroundSchedulerRx3(): SchedulerRx3 = SchedulersRx3.computation()

    @Provides @JvmStatic @Foreground
    fun provideForegroundScheduleRx3(): SchedulerRx3 = AndroidSchedulersRx3.mainThread()
}

```
#### 4) В базовый презентер добавить 2й контрукор (что бы презентеры со старой rxJava не мешали проекту собраться)

```kotlin
import hu.akarnokd.rxjava3.bridge.RxJavaBridge
import io.reactivex.Scheduler as SchedulerRx2
import io.reactivex.rxjava3.core.Scheduler as SchedulerRx3

abstract class BasePresenter<V : BaseContract.View> protected constructor(
    backgroundScheduler: SchedulerRx3,
    foregroundScheduler: SchedulerRx3,
    val router: Router
) : ExecutionRx2Adapter(backgroundScheduler, foregroundScheduler),
    BaseContract.Presenter<V> {

    constructor(
        backgroundScheduler: SchedulerRx2,
        foregroundScheduler: SchedulerRx2,
        router: Router
    ) : this(
        backgroundScheduler = RxJavaBridge.toV3Scheduler(backgroundScheduler),
        foregroundScheduler = RxJavaBridge.toV3Scheduler(foregroundScheduler),
        router = router
    )

    ...
}
```

#### 5) Когда все будет на RxJava3 удаляем ExecutionRx2Adapter вместе с зависимостями от RxJava2
