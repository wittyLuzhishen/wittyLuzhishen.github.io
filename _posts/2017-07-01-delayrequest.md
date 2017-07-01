---
layout: post
title: 延迟请求
date: 2017-07-01 15:59:56
categories: Java
---
# 背景
考虑以下场景：用户在搜索框里输入文本，我们希望在用户结束输入（用户在一段时间内没有再修改已输入的内容）后发起搜索请求。
如果用户在停止输入的特定时间内修改了先前输入的内容，则重新开始计时。
我们知道请求是耗时的，在请求已经发出但还没有得到响应之前，如果用户改变了先前输入的内容，则要取消掉对此次响应的处理。


# 代码清单

以下的代码片段适用于Android项目，它依赖了RxJava、RxAndroid以及support-annotations库。

每当有新请求到来（本例只用户输入变化）时，调用onNewRequest方法即可。

```java
/**
 * <pre>
 * 用于延时请求的类
 * 典型应用场景是等用户在搜索框里输入完成后，延迟一段时间再发起请求，该延迟和请求会因用户的再次输入而放弃
 * </pre>
 * @param <T> 请求用到的参数类型
 */
public abstract class BaseDelayRequest<T> {
    private int mDelayMillis = 1_000;
    private Subscription mDelaySub;
    private Subscription mRequestSub;

    /**
     *
     * @param delayMillis 发起请求的延迟，单位：毫秒，默认1000
     */
    protected BaseDelayRequest(int delayMillis) {
        if (delayMillis > 0) {
            mDelayMillis = delayMillis;
        }
        Log.d(TAG, "delay time: " + mDelayMillis + "ms");
    }

    /**
     * <pre>
     * 发起一个新的请求，这意味着要放弃先前的延迟和对已经发出去的请求响应的监听
     * 子类覆盖此方法时，应该首先调用父类的该方法
     * </pre>
     * @param req 请求用到的参数
     */
    public final void onNewRequest(@NonNull final T req) {
        Log.d(TAG, "onNewRequest, req:" + req);
        unsubscribe();
        if (!onPreDelay(req)) {
            return;
        }
        mDelaySub = Observable
                .timer(mDelayMillis, TimeUnit.MILLISECONDS)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<Long>() {
                    @Override
                    public void call(Long aLong) {
                        if (!onPreRequest(req)) {
                            return;
                        }
                        mRequestSub = getRequestSub(req);
                    }
                }, new Action1<Throwable>() {
                    @Override
                    public void call(Throwable e) {
                        Log.e(TAG, "unexpected error", e);
                    }
                });
    }

    /***
     * <pre>
     * 在开始延迟之前的处理
     * 通常用来设置界面和检查参数是否合法，如果不合法就不必发起请求了
     * 和{@link #onNewRequest(Object)}运行在同一线程上
     * </pre>
     * @param req
     * @return 是否有必要发起请求
     */
    protected abstract boolean onPreDelay(@NonNull T req);

    /**
     * <pre>
     * 在延迟到期，准备发起请求之前的处理
     * 通常用来设置界面
     * </pre>
     * @param req
     * @return 如果返回false，则请求不会发出去
     */
    @MainThread
    protected abstract boolean onPreRequest(@NonNull T req);

    /**
     * <pre>
     * 返回请求的Subscription
     * </pre>
     * @param req
     * @return
     */
    @MainThread
    protected abstract Subscription getRequestSub(@NonNull T req);

    public void unsubscribe() {
        if (mDelaySub != null && !mDelaySub.isUnsubscribed()) {
            mDelaySub.unsubscribe();
            mDelaySub = null;
        }
        if (mRequestSub != null && !mRequestSub.isUnsubscribed()) {
            mRequestSub.unsubscribe();
            mRequestSub = null;
        }
    }

}
```
