---
title: RxJava2源码分析
date: 2018-09-13 15:00:19
tags: RxJava2
categories: [RxJava2]
---
RxJava2学习笔记，基于`io.reactivex.rxjava2:rxjava:2.1.10`
<!--more-->

### 基础
RxJava简单的使用如下：


	Observable.create(new ObservableOnSubscribe<String>() {
	            @Override
	            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
	                emitter.onNext("aaa");
	                emitter.onNext("zzz");
	                emitter.onComplete();
	            }
	        }).subscribe(new Observer<String>() {
	            @Override
	            public void onSubscribe(Disposable d) {
	                
	            }
	
	            @Override
	            public void onNext(String s) {
	                Log.d("xcx", "onNext: "+s);
	            }
	
	            @Override
	            public void onError(Throwable e) {
	
	            }
	
	            @Override
	            public void onComplete() {
	                Log.d("xcx", "onComplete: ");
	            }
	        });

### Observable.create
我们看一下create()方法做了什么：


	public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
			 //判断空值
	        ObjectHelper.requireNonNull(source, "source is null");
	        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
	    }
调用onAssembly()方法，传入一个ObservableCreate类，并且将我们在create传入的ObservableOnSubscribe作为构造参数传入。

#### 看一下onAssembly：

	public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
	        Function<? super Observable, ? extends Observable> f = onObservableAssembly;
	        if (f != null) {
	            return apply(f, source);
	        }
	        return source;
	    }
如果onObservableAssembly为空，则返回source。onObservableAssembly在哪里赋值的呢？在这里：

	setOnObservableAssembly(null);
	public static void setOnObservableAssembly(@Nullable Function<? super Observable, ? extends Observable> onObservableAssembly) {
        if (lockdown) {
            throw new IllegalStateException("Plugins can't be changed anymore");
        }
        RxJavaPlugins.onObservableAssembly = onObservableAssembly;
    }
可以看出，这一步只是将Observable做了封装，传入什么就返回了什么。RxJava中还有很多这样的方法，碰到我们再说。此时转入ObservableCreate类中。

#### ObservableCreate类
这一步只是将我们传入的source保存了起来，此时完成了observable的创建任务，是何时订阅的呢？没错，就是在subscribe()方法中订阅的。接下来看subscribe(Observer)方法

	public final class ObservableCreate<T> extends Observable<T> {
	    final ObservableOnSubscribe<T> source;
	
	    public ObservableCreate(ObservableOnSubscribe<T> source) {
	        this.source = source;
	    }
	
	    @Override
	    protected void subscribeActual(Observer<? super T> observer) {
	        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
	        observer.onSubscribe(parent);
	
	        try {
	            source.subscribe(parent);
	        } catch (Throwable ex) {
	            Exceptions.throwIfFatal(ex);
	            parent.onError(ex);
	        }
	    }
	
	    static final class CreateEmitter<T>
	    extends AtomicReference<Disposable>
	    implements ObservableEmitter<T>, Disposable { ... }
	}

### Observable#subscribe(Observer<T>)

	public final void subscribe(Observer<? super T> observer) {
	        ObjectHelper.requireNonNull(observer, "observer is null");
	        try {
	        	  //调用onSubscribe，跟onAssembly一样，返回了传入的observer
	            observer = RxJavaPlugins.onSubscribe(this, observer);
	
	            ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");
	            //核心代码，传入observer
	            subscribeActual(observer);
	        } catch (NullPointerException e) { // NOPMD
	            throw e;
	        } catch (Throwable e) {
	            Exceptions.throwIfFatal(e);
	            // can't call onError because no way to know if a Disposable has been set or not
	            // can't call onSubscribe because the call might have set a Subscription already
	            RxJavaPlugins.onError(e);
	
	            NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
	            npe.initCause(e);
	            throw npe;
	        }
	    }
细心的你看到了么？调用了`subscribeActual(observer);`这个方法，上面ObservableCreate类也有相同的方法。是巧合么？肯定不是啦。这个方法在Observable中：


	protected abstract void subscribeActual(Observer<? super T> observer);
上面提到过ObservableCreate类，它继承了了Observable，实现了`subscribeActual(observer);`这个方法。当我们调用subscribe时，其实就是调用了ObservableCreate实现方法。


	ObservableCreate类的subscribeActual
	
	@Override
    protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);
	
        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
调用source就是我们传入的ObservableOnSubscribe的onSubscribe开始传输数据。

### 总结
整个过程其实分三步：

1. 创建ObservableCreate类该类继承了Observable，实现了`subscribeActual()`方法。
2. 调用ObservableCreate的`subscribe()`方法，内部调用了第一步的`subscribeActual()`
3. `subscribeActual()`内部调用了ObservableOnSubscribe的subscribe()方法，开始传输数据。


把observer的引用传给了Observable，从而可以调用observer的方法。
