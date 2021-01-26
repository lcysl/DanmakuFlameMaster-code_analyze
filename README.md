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
DanmakuView 的 drawDanmakus 方法中通过同步时间并且更新计时，弹幕的滑动就是靠时间来计算位置并更新。调用 lockCanvas 方法并执行 postInvalidateCompat 方法触发 DanmakuView 自身 onDraw 方法的调用。
```java
protected void lockCanvas() {
    if(mDanmakuVisible == false) {
        return;
    }
    postInvalidateCompat();
    synchronized (mDrawMonitor) {
        while ((!mDrawFinished) && (handler != null)) {
            try {
                mDrawMonitor.wait(200);
            } catch (InterruptedException e) {
                if (mDanmakuVisible == false || handler == null || handler.isStop()) {
                    break;
                } else {
                    Thread.currentThread().interrupt();
                }
            }
        }
        mDrawFinished = false;
    }
}

private void postInvalidateCompat() {
    mRequestRender = true;
    if(Build.VERSION.SDK_INT >= 16) {
        this.postInvalidateOnAnimation();
    } else {
        this.postInvalidate();
    }
}
```
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
DrawTask 的 draw 方法中调用自身的 drawDanmakus，继续跟踪到了 DrawTask 的 drawDanmakus 方法，代码太多只看关键
```java
protected RenderingState drawDanmakus(AbsDisplayer disp, DanmakuTimer timer) {
    //...
    if (danmakuList != null) {
        // ...
        IDanmakus screenDanmakus = danmakus;
        if(mLastBeginMills > beginMills || timer.currMillisecond > mLastEndMills) {
            screenDanmakus = danmakuList.sub(beginMills, endMills);
            if (screenDanmakus != null) {
                danmakus = screenDanmakus;
            }
            mLastBeginMills = beginMills;
            mLastEndMills = endMills;
        } else {
            beginMills = mLastBeginMills;
            endMills = mLastEndMills;
        }

        // prepare runningDanmakus to draw (in sync-mode)
        IDanmakus runningDanmakus = mRunningDanmakus;
        beginTracing(renderingState, runningDanmakus, screenDanmakus);
        if (runningDanmakus != null && !runningDanmakus.isEmpty()) {
            mRenderingState.isRunningDanmakus = true;
            mRenderer.draw(disp, runningDanmakus, 0, mRenderingState);
        }

        // draw screenDanmakus
        mRenderingState.isRunningDanmakus = false;
        if (screenDanmakus != null && !screenDanmakus.isEmpty()) {
            mRenderer.draw(mDisp, screenDanmakus, mStartRenderTime, renderingState);
            //...
        } else {
            // ...
        }
    }
    return null;
}
```
首先截取要显示的弹幕，通过 DanmakuRenderer 的 draw 方法开始绘制。danmakuList是一个弹幕Collection，addDanmaku 和 removeDanmaku 方法就是在操作此集合。

跟踪到DanmakuRenderer的draw方法，在该方法中调用了Danmakus中的foreach，去遍历所有被添加进去的弹幕实例
```java
 public void forEach(Consumer<? super BaseDanmaku, ?> consumer) {
    consumer.before();
    Iterator<BaseDanmaku> it = items.iterator();
    while (it.hasNext()) {
        BaseDanmaku next = it.next();
        if (next == null) {
            continue;
        }
        int action = consumer.accept(next);
        if (action == DefaultConsumer.ACTION_BREAK) {
            break;
        } else if (action == DefaultConsumer.ACTION_REMOVE) {
            it.remove();
            mSize.decrementAndGet();
        } else if (action == DefaultConsumer.ACTION_REMOVE_AND_BREAK) {
            it.remove();
            mSize.decrementAndGet();
            break;
        }
    }
    consumer.after();
}
```
foreach 方法中将每个弹幕实例交给 consumer 的 accept 方法执行。Consumer 是 DanmakuRender 类的私有内部类，accept 核心代码如下
```java
public int accept(BaseDanmaku drawItem) {
    // ...

    // measure
    if (!drawItem.isMeasured()) {
        drawItem.measure(disp, false);
    }

    // notify prepare drawing
    if (!drawItem.isPrepared()) {
        drawItem.prepare(disp, false);
    }

    // layout
    mDanmakusRetainer.fix(drawItem, disp, mVerifier);

    // draw
    if (drawItem.isShown()) {
        if (drawItem.lines == null && drawItem.getBottom() > disp.getHeight()) {
            return ACTION_CONTINUE;    // skip bottom outside danmaku
        }
        int renderingType = drawItem.draw(disp);
        if (renderingType == IRenderer.CACHE_RENDERING) {
            renderingState.cacheHitCount++;
        } else if (renderingType == IRenderer.TEXT_RENDERING) {
            renderingState.cacheMissCount++;
            if (mCacheManager != null) {
                mCacheManager.addDanmaku(drawItem);
            }
        }
        renderingState.addCount(drawItem.getType(), 1);
        renderingState.addTotalCount(1);
        renderingState.appendToRunningDanmakus(drawItem);

        if (mOnDanmakuShownListener != null
                && drawItem.firstShownFlag != mContext.mGlobalFlagValues.FIRST_SHOWN_RESET_FLAG) {
            drawItem.firstShownFlag = mContext.mGlobalFlagValues.FIRST_SHOWN_RESET_FLAG;
            mOnDanmakuShownListener.onDanmakuShown(drawItem);
        }
    }
    return ACTION_CONTINUE;
}
```
在从弹幕 Collection 集合取出的 DanmakuView 传给 accept，在 accept 中见到了我们熟悉的自定义view三大步，测量(measure)、布局(layout)、绘制(draw)。

测量、布局、绘制都使用到了同一个参数：disp。measure 和 draw最终都交给了 AbsDisplayer 的实现 AndroidDisplayer 去做。AndroidDisplayer 中提供了一个静态内部类 DisplayerConfig。查看DisplayerConfig 中提供的属性，不难看出，我们最初在 DanmakuContext 时提供的初始化属性，最终都被设置到了 DisplayerConfig 上。
### 测量
```java
@Override
public void measure(BaseDanmaku danmaku, boolean fromWorkerThread) {
    TextPaint paint = getPaint(danmaku, fromWorkerThread);
    if (mDisplayConfig.HAS_STROKE) {
        mDisplayConfig.applyPaintConfig(danmaku, paint, true);
    }
    calcPaintWH(danmaku, paint, fromWorkerThread);
    if (mDisplayConfig.HAS_STROKE) {
        mDisplayConfig.applyPaintConfig(danmaku, paint, false);
    }
}
```
measure 方法中拿到初始化的 TextPaint，调用 calcPaintWH 方法计算当前弹幕宽高。
```java
private void calcPaintWH(BaseDanmaku danmaku, TextPaint paint, boolean fromWorkerThread) {
    sStuffer.measure(danmaku, paint, fromWorkerThread);
    setDanmakuPaintWidthAndHeight(danmaku, danmaku.paintWidth, danmaku.paintHeight);
}
```
在测量过程中，还调用了 sStuffer 的 measure 方法，sStuffer 是 BaseCacheStuffer 的实现类 SimpleTextCacheStuffer（BaseCacheStuffer 有多种类型的实现，SimpleTextCacheStuffer 是常用的文字类型的弹幕实现） 的实例，在该 measure 方法中对弹幕中的文本宽高进行了详细计算。
### 布局
布局过程调用了 DanmakusRetainer 的 fix 方法，通过设置的弹幕 Type，即显示的方向，来决定使用哪种方式布局
```java
public void fix(BaseDanmaku danmaku, IDisplayer disp, Verifier verifier) {
    int type = danmaku.getType();
    switch (type) {
        case BaseDanmaku.TYPE_SCROLL_RL:
            rldrInstance.fix(danmaku, disp, verifier);
            break;
        case BaseDanmaku.TYPE_SCROLL_LR:
            lrdrInstance.fix(danmaku, disp, verifier);
            break;
        case BaseDanmaku.TYPE_FIX_TOP:
            ftdrInstance.fix(danmaku, disp, verifier);
            break;
        case BaseDanmaku.TYPE_FIX_BOTTOM:
            fbdrInstance.fix(danmaku, disp, verifier);
            break;
        case BaseDanmaku.TYPE_SPECIAL:
            danmaku.layout(disp, 0, 0);
            break;
    }
}
```
跟踪代码到
```java
drawItem.layout(disp, drawItem.getLeft(), topPos);
```
我们使用的是弹幕从右向左布局显示，所以直接看 drawItem 的实现类 R2LDanmaku 的 layout 方法
```java
 public void layout(IDisplayer displayer, float x, float y) {
    if (mTimer != null) {
        long currMS = mTimer.currMillisecond;
        long deltaDuration = currMS - getActualTime();
        if (deltaDuration > 0 && deltaDuration < duration.value) {
            this.x = getAccurateLeft(displayer, currMS);
            if (!this.isShown()) {
                this.y = y;
                this.setVisibility(true);
            }
            mLastTime = currMS;
            return;
        }
        mLastTime = currMS;
    }
    this.setVisibility(false);
}
```
通过 getAccurateLeft 获取 X 坐标位置
```java
protected float getAccurateLeft(IDisplayer displayer, long currTime) {
    long elapsedTime = currTime - getActualTime();
    if (elapsedTime >= duration.value) {
        return -paintWidth;
    }

    return displayer.getWidth() - elapsedTime * mStepX;
}
```
getActualTime 返回的是弹幕显示的时间，现在的时间减去弹幕的时间就是已经过去的时间，如果过去的时间超过弹幕的持续时间，说明弹幕已经显示完毕。否则返回
displayer.getWidth() - elapsedTime * mStepX; 屏幕宽度-时间*速度，即距离左边的距离

从这里可以看出，弹幕的显示就是根据一个计时器更新时间，并根据时间计算弹幕位置，实现弹幕的滑动效果。
### 绘制
上面提到绘制的过程和测量一样，也是交给了 AndroidDisplayer 去处理。AndroidDisplayer 的 draw 方法中拿到弹幕的 top、left，以及 Paint 实例。
接着调用 drawDanmaku 方法，把绘制工作又交给 sStuffer的drawDanmaku 方法，即 SimpleTextCacheStuffer 中的 drawDanmaku 方法，核心代码如下
```java
public int draw(BaseDanmaku danmaku) {
    float top = danmaku.getTop();
    float left = danmaku.getLeft();
    if (canvas != null) {

        Paint alphaPaint = null;
        boolean needRestore = false;
        if (danmaku.getType() == BaseDanmaku.TYPE_SPECIAL) {
            if (danmaku.getAlpha() == AlphaValue.TRANSPARENT) {
                return IRenderer.NOTHING_RENDERING;
            }
            if (danmaku.rotationZ != 0 || danmaku.rotationY != 0) {
                saveCanvas(danmaku, canvas, left, top);
                needRestore = true;
            }

            int alpha = danmaku.getAlpha();
            if (alpha != AlphaValue.MAX) {
                alphaPaint = mDisplayConfig.ALPHA_PAINT;
                alphaPaint.setAlpha(danmaku.getAlpha());
            }
        }

        // skip drawing when danmaku is transparent
        if (alphaPaint != null && alphaPaint.getAlpha() == AlphaValue.TRANSPARENT) {
            return IRenderer.NOTHING_RENDERING;
        }

        // drawing cache
        boolean cacheDrawn = sStuffer.drawCache(danmaku, canvas, left, top, alphaPaint, mDisplayConfig.PAINT);
        int result = IRenderer.CACHE_RENDERING;
        if (!cacheDrawn) {
            if (alphaPaint != null) {
                mDisplayConfig.PAINT.setAlpha(alphaPaint.getAlpha());
                mDisplayConfig.PAINT_DUPLICATE.setAlpha(alphaPaint.getAlpha());
            } else {
                resetPaintAlpha(mDisplayConfig.PAINT);
            }
            drawDanmaku(danmaku, canvas, left, top, false);
            result = IRenderer.TEXT_RENDERING;
        }

        if (needRestore) {
            restoreCanvas(canvas);
        }

        return result;
    }

    return IRenderer.NOTHING_RENDERING;
}
```
drawDanmaku 关键代码
```java
public void drawDanmaku(BaseDanmaku danmaku, Canvas canvas, float left, float top, boolean fromWorkerThread, AndroidDisplayer.DisplayerConfig displayerConfig) {
    // ...
    // 绘制背景
    drawBackground(danmaku, canvas, _left, _top);
    // draw stroke
    drawStroke(danmaku, lines[t], canvas, strokeLeft, strokeTop, paint);
    // draw text
    drawText(danmaku, lines[t], canvas, left, t * textHeight + top - paint.ascent(), paint, fromWorkerThread);


    // draw underline
    if (danmaku.underlineColor != 0) {
        Paint linePaint = displayerConfig.getUnderlinePaint(danmaku);
        float bottom = _top + danmaku.paintHeight - displayerConfig.UNDERLINE_HEIGHT;
        canvas.drawLine(_left, bottom, _left + danmaku.paintWidth, bottom, linePaint);
    }

    //draw border
    if (danmaku.borderColor != 0) {
        Paint borderPaint = displayerConfig.getBorderPaint(danmaku);
        canvas.drawRect(_left, _top, _left + danmaku.paintWidth, _top + danmaku.paintHeight,
                borderPaint);
    }

}
```
最终的绘制都交给了 Stuffer 去处理。在 DanmakuContext 中提供了 setCacheStuffer 方法，允许我们进行 Stuffer 的自定义，例如keep直播弹幕中涉及到圆角弹幕样式，就是利用自定义 Stuffer 来实现。
到此，弹幕的显示流程就全部执行结束了。整体来看，涉及了如下几个核心类：
* R2LDanmaku : 一个弹幕对象, 里面包含x、y坐标, 缓存的Bitmap等属性
* DanmakuView : 用来承载弹幕显示的ViewGroup, 除了它之外还有DanmakuSurfaceView、DanmakuTextureView
* DrawHandler : 一个绑定了异步HandlerThread的Handler, 控制整个弹幕的显示逻辑
* CacheManagingDrawTask : 维护需要绘制的弹幕列表, 控制弹幕缓存逻辑
* DanmakuRenderer : 对弹幕做一些过滤、碰撞检测、测量、布局、缓存等工作
* AndroidDisplayer : 持有Canvas画布, 绘制弹幕
