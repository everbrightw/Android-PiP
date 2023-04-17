# Android PiP

Created: April 12, 2023 3:24 PM
Status: In Progress

# Android PiP

picture-in-picture (PiP) 从Android 8上开始出现, 是一种特殊的multi-window mode

和自由窗口最大的区别是画中画窗口更多的用于展示内容如视频, 表面会覆盖一层菜单界面用来控制media, 并且用户不能和存在于pip mode的activity的界面交互。

# PiP Window的交互

**从Android 12开始:**

- 单击: 展示操控界面(最大化按钮, 设置按钮, 关闭按钮,...)
- 双击: 最大化/最小化当前PiP window
- 拖动: 在屏幕上任意移动; stash window到屏幕边缘, if stashed, 单击或拖动窗口还原
- pinch-to-zoom(两指缩放): 改变PiP window 大小
- 拖动四角缩放(onDragCornerResize)

---

# App端的应用

Google官方的pip 应用例子: https://github.com/android/media-samples/tree/main/PictureInPicture/#readme

官方文档:https://developer.android.com/develop/ui/views/picture-in-picture

- 在AndroidManifest.xml中配置, actvity允许进入PiP

```jsx
<**activity** android:name=".MainActivity"
    android:configChanges="screenSize|smallestScreenSize|screenLayout|orientation"
    android:supportsPictureInPicture="true">
```

- App端主动进入pip模式

```java
*@Deprecated*
public void enterPictureInPictureMode() {
    enterPictureInPictureMode(new PictureInPictureParams.Builder().build());
}
enterPictureInPictureMode(@NonNull PictureInPictureParams *params*);
```

## PictureInPictureParams

App用来自定义一些pip的样式，动画过渡等等功能

几个常用params的例子:

- `.setAutoEnterEnabled(true)`: 退出activit时自动进入pip
    - 一些常见的视频app都会有类似的功能, 视频界面时退到home自动弹出小窗
    - 在老一些的版本中没有这个接口。实现这个效果是通过Activity中override `onUserLeavHint()`， 里面主动调用`enterPictureInPictureMode`
- `.setSourceRectHint(Rect)`： 提过提供期望截图hint rect, 进入pip的动画会更加丝滑
    - 默认过渡是颜色遮罩
    - 提供hint rect时会用截图做遮罩, 所以动画效果更好
- `.setAspectRatio()`: 设置pip window的窗口比例
    - 默认比例是16:9, landscape

```html
<!-- The default aspect ratio for picture-in-picture windows. -->
<item name="config_pictureInPictureDefaultAspectRatio" format="float" type="dimen">
    1.777778
</item>
```

- 也可以自己自定义成竖屏的pip window: .setAspectRatio(new Rational(9, 16))

<aside>
💡 app自己配置sourceRectHint和aspectRatio需要相匹配才可以, 截图动画可以完成丝滑的过渡没有拉伸是因为做了**等比缩放,** 所以sourceRectHint的ratio不能和进入的pip aspectRatio差太多.

</aside>

---

# 总体的结构Overview

通过**App**端调用`enterPictureInPictureMode`通知**ATMS**和**WM core**开始对task结构和configuration做相应的变化。在变化过程中**WM Shell**通过`TaskOrganizer`感知到window mode的变化(WINDOWING_MODE_PINNED)再通过`TaskListener callback`告诉`PipTaskOrganizer`完成对应的独立动画，和input相关事件的注册等。

![Untitled](Android%20PiP%20b3661494880e403ba99872148754e846/Untitled.png)

## WM Core中的相关处理

关键代码:

- **RootWindowContainer#moveActivityToPinnedRootTask:**
    - `mService.deferWindowLayout()`
    - `rootTask.setWindowingMode(WINDOWING_MODE_PINNED)` -> 有一些关于WINDOWING_MODE_PINNED变化的特殊处理
    - `rootTask.setDeferTaskAppear(false)` -> onTaskAppeared PendingTaskEvent
    - `mService.continueWindowLayout()` -> dispatchPendingEvents
    - `notifyActivityPipModeChanged(*r*.getTask(), *r*)` -> PipController onActivityPinned listeners
    
    ![Untitled](Android%20PiP%20b3661494880e403ba99872148754e846/Untitled%201.png)
    
- 提供进入pip接口, 客户端最终调用到`enterPictureInPictureMode`通知ATMS当前Activity进入pip模式
    - `ActivityTaskManagerService#enterPictureInPictureMode`
- ATMS持有mRootWindowContainer, 作为WindowConfiguration树形结构的root节点(全局单例), 开始"指挥"当前task完成相对应的操作
    - `RootWindowContainer#moveActivityToPinnedRootTask(...)`
    - RootWindowContainer完成了对即将要进入pip task的window mode的改变, 但是此时, 改变的仅仅是windowmode, task surface的大小在等待Pip模块完成动画后由`WindowContainerTransaction`通知更新
- `ActivityRecord#finishing`

<aside>
💡 RootWindowContainer根据当前要进入pip的`activityRecord.getTask().getNonFinishingActivityCount()`来判断是否需要Build新的task作为进入pip activity的rootTask

见下面视频, 两种不同的情况

</aside>

- 在不同场景下, WM Core会让Shell感知到Task的变化, PiP模块就可以根据container的不同变化场景完成不同的独立动画
    - TaskOrganizerController
        - onTaskAppeared
        - onTaskInfoChaned
        - onTaskVanished
        - DispatchPendingEvents

## Shell感知Task的变化

- 通过`ShellTaskOrganizer`感知task的变化
    - onTaskAppeared
    - onTaskInfoChanged
    - onTaskVanished
- 不同模块注册ShellTaskOrganizer.TaskListener

`TaskListener`是shell端的calback, 可以理解为`ShellTaskOrganizer`作为shell的对接人在接受wm core发来的task变化的情况, 再通过`TaskListener`告诉shell的对应模块发生了什么

e.g.

1. PipTaskOrganizer implements ShellTaskOrganizer.TaskListener
2. StageTaskOrganizer implements ShellTaskOrganizer.TaskListener

需要注意的细节是onTaskInfoChanged接口不一定对应TaskListener的onTaskInfoChanged.

e.g.

WM core告诉`ShellTaskOrganizer` onTaskInfoChanged, 对应pip可能感知到的是 onTaskAppeared

比如在"WM Core中的相关处理"的结构图中，如果RootWindowContainer没有新建task, 把当前**唯一**ActivityRecord的Task变成了PINNED task, 这时候ShellTaskOrganizer感知到的是task info的变化。**但是对于`PipTaskOrganizer` 应该理解成onTaskAppeared (on PINNED task appeared),** 所以源码中区分了这一点, 在ShellTaskOrganizer的onTaskInfoChanged中尝试更新callback `ShellTaskOrganizer#updateTaskListenerIfNeeded`，这样做确保了Pip可以在处理task变化的逻辑时做到统一

- 通过TaskStackListener:

`PipController`

- onActivityPinned 注册4个listeners
    - `PipResizeGestureHandler`: 处理pinch-resize, onDragCornerResize, dismiss-target
    - `PipInputConsumer`: 处理移动Pip, pip menu touch 事件的接受
    - `PipMediaController`: Pip menu对于视频media的控制
    - `PipAppOpsListener`: 监听app设置相关变化, runtime permissions access
- onActivityUnPinned 销毁listeners

<aside>
❓ //TODO

我理解`TaskStackListener`和`TaskOrganizer`都是为了感知WM core中container的变化(可能一个针对task一个针对activityrecord), 看代码的区别是感知的时机不同, 还有其他区别吗? 为什么不能只用其中一个在pip中做事情?

</aside>

---

# WM Shell pip

Pip 作为一个单独的模块在shell 中独立处理了动画实现, 包括手势缩放动画, 窗口变化的动画。完成了两种input消费的处理, task surface大小的input consumer和屏幕大小的gesture monitor。在对task做变化(移动, 缩放, 进入退出pip等)的过程和完成时, 通过SurfaceControl.Transaction 和 WindowContainerTransaction通知WM core相关变化

## 动画系统

1. 在RootWindowContainer对task进行WindowMode的改变后, `rootTask.setDeferTaskAppeared(false)` 会让`TaskOrganizerController`在根据不同的场景添加`PendingTaskEvent`, 这个event会在这之后的`continueLayout` 流程中会被`dispatchPendingEvents`发送给`TaskOrganizer`
2. `PipTaskOraganizer`在感知到Task的变化并且拿到taskInfo和leash后, 就会通过`PipAnimationController`触发pip独立的动画
3. `PipAnimationController.PipTransitionAnimator` 本身是一种ValueAnimator, 换句话说, 通过ValueAnimator作为驱动, 去不断的更新task的SurfaceControl, 达到了动画的效果。
4. 在动画结束时, Pip再通过WindowContainerTransaction向WM core更新bounds, activityWindowingMode等

![Untitled](Android%20PiP%20b3661494880e403ba99872148754e846/Untitled%202.png)

**动画驱动-PipAnimationController**

- *AnimationType*
    - ANIM_TYPE_BOUNDS
    - ANIM_TYPE_ALPHA
- `PipTansitionAnimator` abstract
    - 作为PipAnimationController的内部类, 是一种ValueAnimator, 驱动整个pip的动画系统, 对task的SurfaceControl做操作
    - 在PipAnimationController中静态实现了两种concrete class:
        - ofBounds
        - ofAlpha
    - pip模块会根据不同的场景通过controller拿到上面两种不同的animator实现, `PipAnimationController#getAnimator`
    - PipTransitionAnimator作为一个抽象类，封装好了pip在做不同动画时的通用逻辑, 如应用`SurfaceControlTransaction`对leash的操作, 添加/删除`PipContentOverlay`, 插入通用的`PipAnimationCallback`逻辑等。 并且规定好了generic type的变量已经需要定制的操作:
        - T mBaseValue;
        - T mCurrentValue;
        - T mStartValue;
        - T mEndValue;
        - appySurfaceTransaction()
        - ...
        - 两个不同的具体实现ofBounds, ofAlpha分别对应Rect 和 float

**动画遮罩-PipContentOverlay**

pip在做动画的时候会根据app的提供的不同的PipParameters使用不同的遮罩:

- PipColorOverlay
- PipSnapShotOverlay

![Untitled](Android%20PiP%20b3661494880e403ba99872148754e846/Untitled%203.png)

PipContentOverlay被reparent到Task leash, Integer.MAX_VALUE确保overlay在所有sibilings中z-order是最高的

<aside>
💡 做动画时的SurfaceControl操作还是在task的leash上完成, see `SurfaceControl#reparent`

Re-parents a given layer to a new parent. **Children inherit transform (position, scaling) crop, visibility, and Z-ordering from their parents**, as if the children were pixels within the parent Surface.

</aside>

App端如果提供了**有效**的sourceRectHint就会使用PipSnapShotOverlay; 如果没有提供sourceRectHint或者是无效的, 就会使用默认的PipColorOverlay

PipContentOverlay提供了3种callback来定制不同overlay的表现:

- attach
- onAnimationUpdate
- onAnimationEnd

比如, SnapshotOverlay在onAnimationUpdate的时候不需要做任何事情, 跟着parentLeash动就可以; ColorOverlay的逻辑则是需要用适当的方式在做动画时更新纯色遮罩的透明度, 来达到相对好的效果。

两种不同的遮罩样式:

暂时无法在文档外展示此内容

**动画代码示例**

关键代码

- PipAnimationController
- PipSurfaceTransactionHelper: 里面封装了对surface在pip不同场景下的组合操作

PipAnimationController在配置好之后就会启动PipTransitionAnimator,这时属性动画开始根据设置好的动画曲线对常量值做改变, 在不同的`ValueAnimator.AnimatorUpdateListener`, 回调中做不同的事情, 关键的操作就是通过PipSurfaceTransactionHelper来对leash做各种变化来达到动画的效果

```java
`// PipAnimationController
*@Override*
public void onAnimationUpdate(ValueAnimator *animation*) {
    // customized by concrete classes
    applySurfaceControlTransaction(mLeash, newSurfaceControlTransaction(),
            *animation*.getAnimatedFraction());
}`

`// e.g. 
// PipAnimationController#ofBounds
*@Override*
void applySurfaceControlTransaction(SurfaceControl *leash*,
        SurfaceControl.Transaction *tx*, float *fraction*) {
    final Rect base = getBaseValue();
    final Rect start = getStartValue();
    final Rect end = getEndValue();
    if (mContentOverlay != null) {
        mContentOverlay.onAnimationUpdate(*tx*, *fraction*);
    }
    ...
    Rect bounds = mRectEvaluator.evaluate(*fraction*, start, end);
    float angle = (1.0f - *fraction*) * *startingAngle*;
    setCurrentValue(bounds);
    if (inScaleTransition() || *sourceHintRect* == null) {
        //做不等比的scale
        ...
    } else {
        // fraction是现在动画常量变化完成的比例, 根据这个fraction计算出来一个预期的temp的切割rect, 用于后面的crop
        final Rect insets = computeInsets(*fraction*);
        getSurfaceTransactionHelper().scaleAndCrop(*tx*, *leash*,
                *sourceHintRect*, initialSourceValue, bounds, insets,
                isInPipDirection);
        ...
    }
}
**see: PipSurfaceTransactionHelper#scaleAndCrop**// scale, crop, position(左上坐标位移)
*tx*.setMatrix(*leash*, mTmpTransform, mTmpFloat9)
        // 这个mTmpDestinationRect就是上面computeInsets计算出来的这次动画update需要crop到的地方
        .setCrop(*leash*, mTmpDestinationRect)
        .setPosition(*leash*, left, top);`
```

## 手势系统

![Untitled](Android%20PiP%20b3661494880e403ba99872148754e846/Untitled%204.png)

- 注册InputConsumer接收移动, touch事件
- 注册"pip-resize"gesture monitor监听全局gesture
    - pinch resize 双指缩放
        - 开关配置: `PipResizeGestureHandler#mEnablePinchResize`
    - onDragCornerResize 拖拽四角缩放
        - 开关配置: `PipResieGestureHandler#mEnableDragCornerResize`

<aside>
💡 拖拽四角缩放需要监听task surface之外的触摸事件, 所以用InputConsumer注册一个和PiP window 一样大小的surface做不到这一点。Gesutre Monitor的touchableRegion是整个屏幕, 所以PiP可以感知到task surface之外的触摸事件

</aside>

**移动单双击-InputConsumer**

注册`PipInputConusmer`

```java
`PipController#init() onActivityPinned()`

`mPipInputConsumer.registerInputConsumer();`

`public void registerInputConsumer() {
    if (mInputEventReceiver != null) {
        return;
    }
    final InputChannel inputChannel = new InputChannel();
    try {
        *// TODO(b/113087003): Support Picture-in-picture in multi-display.*        mWindowManager.destroyInputConsumer(mName, DEFAULT_DISPLAY);
        **mWindowManager.createInputConsumer(mToken, mName, DEFAULT_DISPLAY, inputChannel);**    } catch (RemoteException e) {
        ProtoLog.e(ShellProtoLogGroup.WM_SHELL_PICTURE_IN_PICTURE,
                "%s: Failed to create input consumer, %s", TAG, e);
    }
    mMainExecutor.execute(() -> {
        *// Choreographer.getSfInstance() must be called on the thread that the input event        // receiver should be receiving events        // TODO(b/222697646): remove getSfInstance usage and use vsyncId for transactions        // YW_PIP_NOTE        // TODO:*        mInputEventReceiver = new InputEventReceiver(inputChannel,
            Looper.myLooper(), Choreographer.getSfInstance());
        if (mRegistrationListener != null) {
            mRegistrationListener.onRegistrationChanged(true */* isRegistered */*);
        }
    });
}`
```

在上面注册InputConsumer的时候会发现并没有设置成task surface的大小, 这个逻辑是在InputMonitor中特殊处理了`mPipInputConsumer`

`InputMonitor#UpdateInputForAllWindowsConsumer`

// 特殊记录

`mPipInputConsumer = getInputConsumer(INPUT_CONSUMER_PIP);`

- 在`InputMonitor`中注册consumer的时候, 通过`private final ArrayMap<String, InputConsumerImpl> mInputConsumers = new ArrayMap();`
- 特殊记录了mPipInputConsumer, 并且在窗口`continueLayout`的时候更新InputConsumer的大小，**也就是当task变成Pip window的时候, InputConsumer的touchableRegion更新成和task surface一样的大小**

```java
//InputMonitor.UpdateInputForAllWindowsConsumer#accpet(WindowState w) 
if (w.inPinnedWindowingMode()) {
    if (mAddPipInputConsumerHandle) {
        *// YW_PIP_NOTE        // update the mPipInputConsumer to cropped by the task bounds*        final Task rootTask = w.getTask().getRootTask();
        **mPipInputConsumer.mWindowHandle.replaceTouchableRegionWithCrop(                rootTask.getSurfaceControl());**        final DisplayArea targetDA = rootTask.getDisplayArea();
        *// We set the layer to z=MAX-1 so that it's always on top.*        if (targetDA != null) {
            mPipInputConsumer.layout(mInputTransaction, rootTask.getBounds());
            mPipInputConsumer.reparent(mInputTransaction, targetDA);
            mPipInputConsumer.show(mInputTransaction, MAX_VALUE - 1);
            mAddPipInputConsumerHandle = false;
        }
    }
}
```

dumpsys input 和 dumpsys window的对比, consumer的touchableRegion和task bounds是一致的, 这也是PiP其中一个特性的原因, app进入pip模式是不能再和app自己的界面交互的

*adb shell dumpsys input* | vim -

/pip_input_consumer

*adb shell dumpsys window w | vim -*

*/mWindowingMode=pinned*

**缩放手势-Gesture Monitor**

`PipResizeGestureHandler`

- onActivityPinned()中注册 也就是wm structure被改变完成的时候, 此时currTask.windwMode == PINNED_MODE

```java
*// YW_PIP_NOTE// handle gestures and stuff// e.g. pinch gesture to resize, onDragCornerResize*mPipResizeGestureHandler.onActivityPinned();
if (mIsEnabled) {
    // Register input event receiver
    **mInputMonitor = InputManager.getInstance().monitorGestureInput(            "pip-resize", mDisplayId);**    try {
        mMainExecutor.executeBlocking(() -> {
            **mInputEventReceiver = new PipResizeInputEventReceiver(                    mInputMonitor.getInputChannel(), Looper.myLooper());**        });
    } catch (InterruptedException e) {
        throw new RuntimeException("Failed to create input event receiver", e);
    }
}
```

- 全局的手势监听

adb shell dumpsys input | vim -

/Gesture Monitor

- 拖拽四角缩放(onDragCornerResize)等手势实现

*InputManager.getInstance().monitorGestureInput(IBinder monitorToken, @NonNull String requestedName, int displayI);*

InputMonitor相关wiki: https://wiki.n.miui.com/display/~chuziqian/InputMonitor

**不同场景下的dumpsys对比**

**移动**

adb shell dumpsys input | vim -

/Input Dispatcher

**onDragCorner**

**两指缩放**

## PipMenu

**//TODO**

SystemWindows

SurfaceControlViewHost

## Pip转屏

---

# PiP dumpsys相关关键词

`adb shell dumpsys activity service com.android.systemui`

/PipController

//TODO

1. PipTochHandler
2. PipBoundsAlgorithm
- PipTaskOrganizer
    - mPictureInPictureParams: 可以用这个来区分app是自己addView悬浮窗还是用的pip.

`mPictureInPictureParams=PictureInPictureParams( 
    aspectRatio=null  #进入pip window 的 width / height ratio, null默认1.7
    expandedAspectRatio=null  
    sourceRectHint= Rect(0, 60 - 1600, 1060) #截图的提示rect
    hasSetActions=true 
    hasSetCloseAction=false 
    isAutoPipEnabled=false # onUserLeaveHint(), 返回桌面自动进入pip模式
    isSeamlessResizeEnabld=true title=null 
    subtitle=null 
    isLaunchIntoPip=false
)`

- PipBoundsState
- PipInputConsumer

# Reference

- https://developer.android.com/develop/ui/views/picture-in-picture