# Android 5.1 Webview内存泄漏新场景

来源:[https://coolpers.github.io/webview/memory/leak/2015/07/16/android-5.1-webview-memory-leak.html](https://coolpers.github.io/webview/memory/leak/2015/07/16/android-5.1-webview-memory-leak.html)

## 问题现象

今天发现App存在 WebView 泄漏情况，还比较严重。并且只是发生在 Android 5.1 系统。

GC roots 如下:

![](5/1.JPG)

每新打开一次这个WebViewActivity，就会发生就会发生一次改Webview实例无法释放，新增一个对象。

上图中的两个 AppSearchWebView实例，就是由于打开了两次导致。

## 问题分析

出现了这个问题分析起来还是比较简单的，根据这个引用关系，我们可以直观的看到是由于 `Appsearch（extends Application）`的 `mComponentCallbacks` 一直在强引用`AWComponentCallbacks`，导致无法释放。然后`AWComponentCallbacks -> AWContents > AppSearchWebView`。

通过分析代码发现关键在于 AwContents 这里的 AwComponentsCallbacks 为什么没有释放。

```
@Override
public void onAttachedToWindow() {
    if (isDestroyed()) return;
    if (mIsAttachedToWindow) {
        Log.w(TAG, "onAttachedToWindow called when already attached. Ignoring");
        return;
    }

    ......

    mComponentCallbacks = new AwComponentCallbacks();
    mContext.registerComponentCallbacks(mComponentCallbacks);
}

@Override
public void onDetachedFromWindow() {
    if (isDestroyed()) return;
    if (!mIsAttachedToWindow) {
        Log.w(TAG, "onDetachedFromWindow called when already detached. Ignoring");
        return;
    }
    ......

    if (mComponentCallbacks != null) {
        mContext.unregisterComponentCallbacks(mComponentCallbacks);
        mComponentCallbacks = null;
    }

    ......
}
```

看这段代码看不出来什么问题，onAttach的时候 register，detach的时候 unregister, 不会存在问题。

但是为什么呢？

难道是由于 if (isDestroyed()) return 这条return引起的？

> 当调用 Webview.destroy() 后 这个判断 返回true。

我们看下哪里调用了 webview.destroy()

```
// source from WebViewActivity

@Override
protected void onDestroy() {
    super.onDestroy();
    
    if (mWebView != null) {
        mWebView.destroy();
    }
}
```

很多应用应该都是这么做的，包括系统浏览器，在Activity destroy的时候，调用 webview的destroy。并且一直工作的很好。

通过调试发现，确实是由于此调用导致的。onDestroy 发生在 onDetach 之前。

那为什么 android 5.1 之前的代码没有问题呢？

看下代码：

```
// AwContents.java

@Override
public void onDetachedFromWindow() {
    if (!mIsAttachedToWindow) {
        Log.w(TAG, "onDetachedFromWindow called when already detached. Ignoring");
        return;
    }

    ......

    if (mComponentCallbacks != null) {
        mContext.unregisterComponentCallbacks(mComponentCallbacks);
        mComponentCallbacks = null;
    }
    ......
}
```

相对于 5.1 的代码少了那句`if (isDestroyed()) return;`

## 规避方法
三水哥的解决方案可以：在destroy之前，把webview 从 parent 中 remove 掉，同样可以提前detach。

```
protected void onDestroy() {
	if (mWebView != null) {
		((ViewGroup) mWebView.getParent()).removeView(mWebView);
		mWebView.destroy();
		mWebView = null;
	}
	super.onDestroy();
}
```