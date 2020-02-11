 https://www.jianshu.com/p/a406b94f3188 

 https://www.jianshu.com/p/cd984dd5aae8 

## RxJava2.0的基本使用

- 第一步 :创建连载小说（被观察者） 

```
//被观察者
        Observable novel=Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("连载1");
                emitter.onNext("连载2");
                emitter.onNext("连载3");
                emitter.onComplete();
            }
        });
```

Observable中文意思就是被观察者，通过create方法生成对象，里面放的参数ObservableOnSubscribe<T>，可以理解为一个计划表，泛型T是要操作对象的类型，重写subscribe方法，里面写具体的计划，本文的例子就是推送连载1、连载2和连载3，在subscribe中的ObservableEmitter<String>对象的Emitter是发射器的意思。ObservableEmitter有三种发射的方法，分别是void onNext(T value)、void onError(Throwable error)、void onComplete()，onNext方法可以无限调用，Observer（观察者）所有的都能接收到，onError和onComplete是互斥的，Observer（观察者）只能接收到一个，OnComplete可以重复调用，但是Observer（观察者）只会接收一次，而onError不可以重复调用，第二次调用就会报异常。

-  第二步：创建读者（观察者） 

```
//观察者
        Observer<String> reader=new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                mDisposable=d;
                Log.e(TAG,"onSubscribe");
            }

            @Override
            public void onNext(String value) {
                if ("2".equals(value)){
                    mDisposable.dispose();
                    return;
                }
                Log.e(TAG,"onNext:"+value);
            }

            @Override
            public void onError(Throwable e) {
                Log.e(TAG,"onError="+e.getMessage());
            }

            @Override
            public void onComplete() {
                Log.e(TAG,"onComplete()");
            }
        };
```

通过new创建接口，并实现其内部的方法，看方法其实就应该差不多知道干嘛的，onNext、onError、onComplete都是跟被观察者发射的方法一一对应的，这里就相当于接收了。onSubscribe（Disposable d）里面的Disposable对象要说一下，Disposable英文意思是可随意使用的，这里就相当于读者和连载小说的订阅关系，如果读者不想再订阅该小说了，可以调用 mDisposable.dispose()取消订阅，此时连载小说更新的时候就不会再推送给读者了。

-  第三步：读者和连载小说建立订阅关系 

```
novel.subscribe(reader);//一行代码搞定
```

## RxJava2.0的异步和链式编程

前言里面有提到，RxJava是支持异步的，但是RxJava是如何做到的呢？这里就需要Scheduler。Scheduler，英文名调度器，它是RxJava用来控制线程。当我们没有设置的时候，RxJava遵循哪个线程产生就在哪个线程消费的原则，也就是说线程不会产生变化，始终在同一个。然后我们一般使用RxJava都是后台执行，前台调用，本着这个原则，我们需要调用observeOn(AndroidSchedulers.mainThread())，observeOn是事件回调的线程，AndroidSchedulers.mainThread()一看就知道是主线程，subscribeOn(Schedulers.io())，subscribeOn是事件执行的线程，Schedulers.io()是子线程，这里也可以用Schedulers.newThread()，只不过io线程可以重用空闲的线程，因此多数情况下 io() 比 newThread() 更有效率。前面的代码根据异步和链式编程的原则，我们可以写成

```
 Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("连载1");
                emitter.onNext("连载2");
                emitter.onNext("连载3");
                emitter.onComplete();
            }
        })
                .observeOn(AndroidSchedulers.mainThread())//回调在主线程
                .subscribeOn(Schedulers.io())//执行在io线程
                .subscribe(new Observer<String>() {
                    @Override
                    public void onSubscribe(Disposable d) {
                        Log.e(TAG,"onSubscribe");
                    }

                    @Override
                    public void onNext(String value) {
                        Log.e(TAG,"onNext:"+value);
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.e(TAG,"onError="+e.getMessage());
                    }

                    @Override
                    public void onComplete() {
                        Log.e(TAG,"onComplete()");
                    }
                });
```

这里就是RxJava最常用的写法，异步+链式编程，还要再说一下，subscribe的方法重载，subscribe（）方法里什么参数也不放是空实现，也就是说连载小说无论出什么连载，读者都不关心，推送过来了也不读，如果读者只关心onNext方法里的内容，可以直接重载subscribe(Consumer<? spuer T> onNext)这个方法，会减少代码，当然如果是初学者还是建议创建Observer对象。





### IO scheduler的弊端

RxJavaSchedulersHook类 里会生成IO Scheduler，默认调用CachedThreadScheduler。
 里面的CachedWorkerPool维护了一个线程管理的队列expiringWorkerQueue，
 默认是每隔60s就去通过evictor清除已经过期的线程，线程池没有上限。因此如果短时间内有大量任务要执行，会导致不停地创建新线程，所以存在出现pthread_create、OOM、耗费大量系统资源造成卡顿等问题。



## [Rxjava线程池]( https://www.cnblogs.com/baiqiantao/p/9650968.html )

- RxJava线程池

RxJava中的多线程操作主要是由强大的`Scheduler`集合提供的。在RxJava中，我们无法直接访问或操作线程。如果想要使用线程的话，必须要通过内置的Scheduler来实现。如果你需要在特定的线程中执行任务的话，我们就需要此选择恰当的Scheduler，Scheduler接下来会从它的池中获取一个可用的线程，并基于该线程执行任务。

在RxJava框架中有多种类型的Scheduler，但是这里比较有技巧的一点就是`为合适的工作选择恰当的Scheduler`。如果你没有选择恰当的Scheduler的话，那么任务就无法最优地运行，所以接下来，我们尝试理解每一个Scheduler。

-  Schedulers.io() 

```
这是由无边界线程池作为支撑的一个Scheduler，它适用于非CPU密集的I/O工作，比如访问文件系统、执行网络调用、访问数据库等等。
这个Scheduler是没有限制的，它的线程池可以按需一直增长。
注意：在使用无边界线程池支撑的Scheduler时，我们要特别小心，因为它有可能会导致线程池无限增长，使系统中出现大量的线程。
```

-  Schedulers.computation() 

```
这个Scheduler用于执行CPU密集的工作，比如处理大规模的数据集、图像处理等等。它由一个有界的线程池作为支撑，线程的最大数量就是可用的处理器数量。
因为这个Scheduler只适用于CPU密集的任务，我们希望限制线程的数量，这样的话，它们不会彼此抢占CPU时间或出现线程饿死的现象。
```

-  Schedulers.newThread() 

```
这个Scheduler 每次都会创建一个全新的线程来完成一组工作。它不会从任何线程池中受益，线程的创建和销毁都是很昂贵的，所以你需要非常小心，不要衍生出太多的线程，导致服务器系统变慢或出现内存溢出的错误。
理想情况下，你应该很少使用这个Scheduler，它大多用于在一个完全分离的线程中开始一项长时间运行、隔离的一组任务。
```

-  Schedulers.single() 

```
这个Scheduler是RxJava 2新引入的，它的背后只有一个线程作为支撑，只能按照有序的方式执行任务。如果你有一组后台任务要在App的不同地方执行，但是同时只能承受一个任务执行的话，那么这个Scheduler就可以派上用场了。
```

-  Schedulers.from(Executor executor) 

```
我们可以使用它创建自定义的Scheduler，它是由我们自己的Executor作为支撑的。在有些场景下，我们希望创建自定义的Scheduler为App执行特定的任务，这些任务可能需要自定义的线程逻辑。
假设，我们想要限制App中并行网络请求的数量，那么我们就可以创建一个自定义的Scheduler，使其具有一个固定线程池大小的Executor：Scheduler.from(Executors.newFixedThreadPool(n))，然后将其应用到代码中所有网络相关的Observable上。
```

-  AndroidSchedulers.mainThread() 

```
这是一个特殊的Scheduler，它无法在核心RxJava库中使用，要使用它，必须要借助RxAndroid扩展库。这个Scheduler对Android App特别有用，它能够在应用的主线程中执行基于UI的任务。
默认情况下，它会在应用主线程关联的looper中进行任务排队，但是它有一个其他的变种，允许我们以API的形式使用任意的Looper：AndroidSchedulers.from(Looper looper)。
```

