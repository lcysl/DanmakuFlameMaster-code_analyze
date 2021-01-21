# DanmakuFlameMaster-code_analyze
Android 平台直播业务中互动弹幕使用了第三方 B 站的 DanmakuFlameMaster 依赖库，DanmakuFlameMaster 与业界其他弹幕实现库相比，功能性能较为完善和稳定。弹幕的显示处理方式主要有两种：
- 显示用户实时发送的弹幕
- 显示长连接收到的或服务端下发的弹幕

DanmakuFlameMaster 库涉及的逻辑多而复杂，下面的内容会从使用方式切入，主要分析一条弹幕是如何绘制到手机屏幕上的。

关于 DanmakuFlameMaster 库的集成就不列举了，DanmakuFlameMaster 使用多种方式(View/SurfaceView/TextureView) 实现高效绘制，其中分别对应(DanmakuView/DanmakuSurfaceView/DanmakuTextureView)等相关的View，由于项目中集成的是最简单的弹幕功能，所有就没有使用关于SurfaceView类的弹幕控件.首先是在 xml 中引入 DanmakuView
```xml
<master.flame.danmaku.ui.widget.DanmakuView
    android:id="@+id/danmakuView"
    android:layout_width="match_parent"
    android:layout_height="match_parent"/>
```
有了 DanmakuView，一个弹幕显示的前提首先需要初始化弹幕上下文环境
```java
val danmakuContext = DanmakuContext.create()
```
DanmakuContext 是一个实现了 Cloneable 的类，提供了静态的 create 方法用来创建默认的初始化配置。类中提供了大量的 set 方法，定义关于弹幕字体、样式、显示方式、显示速度等属性的基础配置。
```java
// 设置一些相关的配置
danmakuContext.setDuplicateMergingEnabled(false)
    .setScrollSpeedFactor(1.2f)
    // 设置显示最大行数
    .setMaximumLines(mapOf(BaseDanmaku.TYPE_SCROLL_RL to MAX_LINES))
    .setCacheStuffer(BackgroundCacheStuffer(view.context), null)
    // 设置防重叠，null代表可以重叠
    .preventOverlapping(mapOf(
        BaseDanmaku.TYPE_SCROLL_RL to true
    ))
```
DanmakuView 中提供了 prepare 方法，prepare 方法接收两个参数，分别是 DanmakupParser 和 DanmakuContext，parser 即弹幕解析器，源码中提供了 BaseDanmakuParser 类。如果没有需要定制处理特殊的解析流程，自定义 Parser 类继承 BaseDanmakuParser，使用默认实现即可。
```java
// 设置解析器
val defaultDanmakuParser = getDefaultDanmakuParser()
// 弹幕准备
view.danmakuView.prepare(defaultDanmakuParser, danmakuContext)
```
接下来看看 DanmakuView 的 prepare 方法
```java
public void prepare(BaseDanmakuParser parser, DanmakuContext config) {
    prepare();
    handler.setConfig(config);
    handler.setParser(parser);
    handler.setCallback(mCallback);
    handler.prepare();
}

private void prepare() {
    if (handler == null)
        handler = new DrawHandler(getLooper(mDrawingThreadType), this, mDanmakuVisible);
}
```
首先调用内部的 prepare 方法初始化 DrawHandler，可以发现我们传入的解析器和弹幕配置都传递了 handler 对应的set方法。handler是DrawHandler的实例，将创建的 HandlerThread 的 Looper 传递给 DrawHandler，handleMessage 的业务处理都将会在子线程执行。
```java
public void handleMessage(Message msg) {
    int what = msg.what;
    switch (what) {
        case PREPARE:
            // ...
            break;
        case SHOW_DANMAKUS:
            // ...
        case START:
            // ...
        case SEEK_POS:
            // ...
        case RESUME:
            // ...
            break;
        case UPDATE:
            // ...
            break;
        case NOTIFY_DISP_SIZE_CHANGED:
            // ...
            break;
        case HIDE_DANMAKUS:
            // ...
        case PAUSE:
            // ...
        case QUIT:
            // ...
            break;
        case NOTIFY_RENDERING:
            // ...
            break;
        case UPDATE_WHEN_PAUSED:
            // ...
            break;
        case CLEAR_DANMAKUS_ON_SCREEN:
            // ...
            break;
        case FORCE_RENDER:
            // ...
            break;
    }
}
```
从 handleMessage 方法可以看到，DrawHandler 类似一个中间使者一样，负责协调弹幕多个状态之间的同步。
```java
case PREPARE:
    mTimeBase = SystemClock.uptimeMillis();
    if (mParser == null || !mDanmakuView.isViewReady()) {
        sendEmptyMessageDelayed(PREPARE, 100);
    } else {
        prepare(new Runnable() {
            @Override
            public void run() {
                pausedPosition = 0;
                mReady = true;
                if (mCallback != null) {
                    mCallback.prepared();
                }
            }
        });
    }
    break;
```
PREPARE 状态下会调用 DanmakuView 的 callback 的 prepare 方法，DanmakuView 中 setCallback 提供了用来监听弹幕从创建到显示的回调方法。
```java
 view.danmakuView.setCallback(object : DrawHandler.Callback {
    override fun updateTimer(timer: DanmakuTimer) {
        // 定时器更新的时候回调
    }

    override fun drawingFinished() {
        // 弹幕绘制完成时回调
    }

    override fun danmakuShown(danmaku: BaseDanmaku) {
        // 弹幕展示的时候回调
    }

    override fun prepared() {
        // 弹幕准备好的时候回调，这里启动弹幕
        view.danmakuView.start()
    }
})
```
在 prepared 回调方法中，调用 DanmakuView 的 start 方法开启弹幕播放。由于上面我们说到，DrawHandler 的 Looper 运行在新的子线程，所以不能在 prepare 的回调里更新 UI。
```java
@Override
public void start(long position) {
    // ...
    if (handler != null) {
        handler.obtainMessage(DrawHandler.START, position).sendToTarget();
    }
}
```
DanmakuView 的 start 方法中通过 obtainMessage 通知到 DrawHandler 的 START 状态，由于 START 状态执行没有 break，最终的执行会经过 START、SEEK_TO、RESUME。
```java
sendEmptyMessage(UPDATE);
drawTask.onPlayStateChanged(IDrawTask.PLAY_STATE_PLAYING);
```
在 RESUME 状态下发出 UPDATE 通知，在 prepare 的时候就初始化了 DrawTask 用来处理弹幕绘制。
```java
private IDrawTask createDrawTask(boolean useDrwaingCache, DanmakuTimer timer,
                                 Context context,
                                 int width, int height,
                                 boolean isHardwareAccelerated,
                                 IDrawTask.TaskListener taskListener) {
    //...
    IDrawTask task = useDrwaingCache ?
            new CacheManagingDrawTask(timer, mContext, taskListener)
            : new DrawTask(timer, mContext, taskListener);
    task.setParser(mParser);
    task.prepare();
    obtainMessage(NOTIFY_DISP_SIZE_CHANGED, false).sendToTarget();
    return task;
}
```
这里会通过 DanmakuContext 初始化配置来决定是否使用 DrawTask 缓存管理类。跟进到 DrawTask 类，在构造函数中，初始化了 DanmakuRender，通过监听 DanmakuRender 的 Shown 方法来通知到DanmakuView 的 danmakuShown Callback。
```java
public DrawTask(DanmakuTimer timer, DanmakuContext context,
        TaskListener taskListener) {
    // ...
    mRenderer = new DanmakuRenderer(context);
    mRenderer.setOnDanmakuShownListener(new IRenderer.OnDanmakuShownListener() {

        @Override
        public void onDanmakuShown(BaseDanmaku danmaku) {
            if (mTaskListener != null) {
                mTaskListener.onDanmakuShown(danmaku);
            }
        }
    });
    // ...
}
```
继续回来看 UPDATE 状态，在 UPDATE 状态下，通过判断 updateMethod 来决定选择线程更新方式
```java
case UPDATE:
    if (mContext.updateMethod == 0) {
        updateInChoreographer();
    } else if (mContext.updateMethod == 1) {
        updateInNewThread();
    } else if (mContext.updateMethod == 2) {
        updateInCurrentThread();
    }
    break;
```
updateMethod 表示当前系统线程数，默认为 0，即使用 Choreographer 驱动 DrawHandler 线程刷新。
```java
/**
 * 0 默认 Choreographer驱动DrawHandler线程刷新 <br />
 * 1 "DFM Update"单独线程刷新 <br />
 * 2 DrawHandler线程自驱动刷新
 *
 * Note: 在系统{@link android.os.Build.VERSION_CODES#JELLY_BEAN}以下, 0方式会被2方式代替
 */
public byte updateMethod = 0;
```
DanmakuContext 类中没有提供设置 updateMethod 方法，所以默认不能自定义 updateMethod，交由系统来决定。三种实现区别只是在线程更新方式上，Choreographer、NewThread、CurrentThread，但最终都会调用到 DanmakuView 的 drawDanmakus 和 waitingRendering 方法
```java
d = mDanmakuView.drawDanmakus();
removeMessages(UPDATE);
if (d > mCordonTime2) {  // this situation may be cuased by ui-thread waiting of DanmakuView, so we sync-timer at once
    timer.add(d);
    mDrawTimes.clear();
}
if (!mDanmakusVisible) {
    waitRendering(INDEFINITE_TIME);
    return;
} else if (mRenderingState.nothingRendered && mIdleSleep) {
    long dTime = mRenderingState.endTime - timer.currMillisecond;
    if (dTime > 500) {
        waitRendering(dTime - 10);
        return;
    }
}
```
DanmakuView 的 drawDanmakus 方法中通过同步时间并且更新计时，并控制最多16ms一帧，弹幕的滑动就是靠时间来计算位置并更新。调用 lockCanvas 方法并执行 postInvalidateCompat 方法触发 DanmakuView 自身 onDraw 方法的调用。
```java
@Override
protected void onDraw(Canvas canvas) {
    if ((!mDanmakuVisible) && (!mRequestRender)) {
        super.onDraw(canvas);
        return;
    }
    if (mClearFlag) {
        DrawHelper.clearCanvas(canvas);
        mClearFlag = false;
    } else {
        if (handler != null) {
            RenderingState rs = handler.draw(canvas);
            if (mShowFps) {
                if (mDrawTimes == null)
                    mDrawTimes = new LinkedList<Long>();
                String fps = String.format(Locale.getDefault(),
                        "fps %.2f,time:%d s,cache:%d,miss:%d", fps(), getCurrentTime() / 1000,
                        rs.cacheHitCount, rs.cacheMissCount);
                DrawHelper.drawFPS(canvas, fps);
            }
        }
    }
    mRequestRender = false;
    unlockCanvasAndPost();
}
```
在 onDraw 方法中，可以看到调用了 handler.draw(canvas)，将 DanmakuView 的 canvas 传递给 draw 方法并返回 RenderingState 实例，DrawHandler 中的 draw 方法调用 DrawTask 的 draw 方法
```java
public RenderingState draw(Canvas canvas) {
    // ...
    mDisp.setExtraData(canvas);
    mRenderingState.set(drawTask.draw(mDisp));
    recordRenderingTime();
    return mRenderingState;
}
```

