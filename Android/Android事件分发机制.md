# Android事件分发机制

[TOC]

> **前言**
>
> 

## 1.Android中的事件

### 1.1.MotionEvent简介

在Android中我们常说的事件其实就是指MotionEvent，它是对触摸事件的封装，包括事件类型、触摸点坐标等等信息。所谓事件分发，分发的就是MotionEvent对象。

常见的事件类型有以下四种：

**MotionEvent.ACTION_DOWN**：手指初次接触到屏幕时触发

**MotionEvent.ACTION_MOVE**：手指在屏幕上滑动时触发，会多次触发

**MotionEvent.ACTION_UP**：手指离开屏幕时触发

**MotionEvent.ACTION_CANCEL**：事件被上层拦截时触发

一般来说，一次正常的交互流程从**ACTION_DOWN** 开始，中间可能会有多次**ACTION_MOVE（手指滑动）**事件，最后以**ACTION_UP（手指离开）**事件结束，而**ACTION_CANCEL**事件不是人为原因产生的，是由于事件被上层ViewGroup拦截导致的，这里先不讨论。示意图如下所示（图片摘自[Carson_Ho博客](https://www.jianshu.com/p/38015afcdb58)）

![](https://upload-images.jianshu.io/upload_images/944365-79b1e86793514e99.png?imageMogr2/auto-orient/)



### 1.2.MotionEvent的常用方法

在开发中有时候会需要我们根据触摸事件（即MotionEvent对象）来进行相应的处理，最常用到的不外乎就是事件类型和触摸点坐标，下面我们就来看一下相关的API。

* **获取事件类型**

获取事件类型的方法有两个：`getAction() `和`getActionMasked()`，这两个方法有什么区别呢？首先我们需要知道Android中的触摸事件是用一个32位的int值表示的，以0x00000000为例，它的低8位（0x000000**00**）表示事件类型，再往前的8位（0x0000**00**00）表示事件编号（actionIndex）。单点触摸时，这个事件编号始终为00，但是当涉及到了多点触摸就不一样了，事件编号会随着按下的手指数量而增加，举个例子，第一个手指按下时，事件编号为00；第二个手指按下时，事件编号为01；第三个手指按下时，事件编号为02，依次类推。下面我们来看一下`getAction() `和`getActionMasked()`这两个方法的定义：

```java
public static final int ACTION_MASK = 0xff;

public final int getAction() {
    return nativeGetAction(mNativePtr);
}

public final int getActionMasked() {
    return nativeGetAction(mNativePtr) & ACTION_MASK;
}
```

可以看出，`getActionMasked()`和`getAction()`的不同之处就是进行了一次按位与的操作，使得获取到int值只包含了低8位即事件类型的信息，其余的位全部为0，这样才能保证我们获取到的值能够和MotionEvent中定义的标准事件类型（ACTION_DOWN、ACTION_MOVE、ACTION_UP等）进行比较，确定事件类型。

因此我们可以得出结论，当不涉及到多点触摸时，使用`getAction() `和`getActionMasked()`获取到的值是一样的，因为事件编号均为00；而当多点触摸时，必须使用`getActionMasked()`方法获取事件类型，因为`getAction() `方法获取的值包含了事件编号信息，无法和标准事件类型进行比较。

* **获取触摸点坐标**

获取触摸点坐标同样有两组方法：`getX()`和`getY()` 、`getRawX()`和`getRawY()`。这两组方法的区别直接上一张图就明白了：



可以看出，`getX()`和`getY()` 获取到的是相对于当前View的坐标，而`getRawX()`和`getRawY()`获取到的是相对于手机屏幕的坐标。

## 2.事件分发流程

事件分发涉及到了三个方法：

**dispatchTouchEvent(MotionEvent ev)**：负责事件的分发，Activity、ViewGroup和View都具有该方法

**onInterceptTouchEvent(MotionEvent ev)**：负责事件的拦截，只有ViewGroup具有该方法

**onTouchEvent(MotionEvent ev)**：负责事件的消费，Activity、ViewGroup和View都具有该方法

事件分发从Activity开始。

### 2.1.Activity

**Activity的dispatchTouchEvent方法**

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
      	// 空方法
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}

/**
 * 该方法是一个空方法，主要作用是实现屏保功能，并且当Activity在栈顶的时候，触屏点击Home、Back、Recent等键都会触发这个方法
 */
public void onUserInteraction() {
}
```



**PhoneWindow的superDispatchTouchEvent方法**

```java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```



**DecorView的superDispatchTouchEvent方法**

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```



### 2.2.ViewGroup

```java
// intercepted表示是否拦截事件
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN
        || mFirstTouchTarget != null) {
    // 这里判断的条件有两个
    // 1.actionMasked == MotionEvent.ACTION_DOWN，表示当前事件为ACTION_DOWN
    // 2.mFirstTouchTarget != null，当ViewGroup将事件交由子View处理时mFirstTouchTarget被赋值，因此mFirstTouchTarget != null表示ViewGroup不拦截事件
    
    // disallowIntercept表示是否禁用事件拦截的功能(默认是false)，可通过子View调用requestDisallowInterceptTouchEvent()方法修改
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action); // restore action in case it was changed
    } else {
    	// 禁用事件拦截
        intercepted = false;
    }
} else {
    // There are no touch targets and this action is not an initial down
    // so this view group continues to intercept touches.
    // 拦截事件
    intercepted = true;
}
```



```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
    if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
            && ev.getAction() == MotionEvent.ACTION_DOWN
            && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
            && isOnScrollbarThumb(ev.getX(), ev.getY())) {
        return true;
    }
    return false;
}
```



```java
final View[] children = mChildren;
// 遍历子View
for (int i = childrenCount - 1; i >= 0; i--) {
    final int childIndex = getAndVerifyPreorderedIndex(
            childrenCount, i, customOrder);
    final View child = getAndVerifyPreorderedView(
            preorderedList, children, childIndex);

    // If there is a view that has accessibility focus we want it
    // to get the event first and if not handled we will perform a
    // normal dispatch. We may do a double iteration but this is
    // safer given the timeframe.
    if (childWithAccessibilityFocus != null) {
        if (childWithAccessibilityFocus != child) {
            continue;
        }
        childWithAccessibilityFocus = null;
        i = childrenCount - 1;
    }

  	// 这里有两个判断条件，如果不满足则跳出当前循环
  	// canViewReceivePointerEvents()判断View是否能够接收事件，当前View处于可见状态或者正在执行动画才可以接收事件
  	// isTransformedTouchPointInView()判断触摸点坐标是否在View的区域内
    if (!canViewReceivePointerEvents(child)
            || !isTransformedTouchPointInView(x, y, child, null)) {
        ev.setTargetAccessibilityFocus(false);
        continue;
    }

    newTouchTarget = getTouchTarget(child);
    if (newTouchTarget != null) {
        // Child is already receiving touch within its bounds.
        // Give it the new pointer in addition to the ones it is handling.
        newTouchTarget.pointerIdBits |= idBitsToAssign;
        break;
    }

    resetCancelNextUpFlag(child);
    if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
        // Child wants to receive touch within its bounds.
      	// 子View的dispatchTouchEvent()方法返回true
        mLastTouchDownTime = ev.getDownTime();
        if (preorderedList != null) {
            // childIndex points into presorted list, find original index
            for (int j = 0; j < childrenCount; j++) {
                if (children[childIndex] == mChildren[j]) {
                    mLastTouchDownIndex = j;
                    break;
                }
            }
        } else {
            mLastTouchDownIndex = childIndex;
        }
        mLastTouchDownX = ev.getX();
        mLastTouchDownY = ev.getY();
      	// addTouchTarget方法完成对mFirstTouchTarget的赋值
        newTouchTarget = addTouchTarget(child, idBitsToAssign);
        alreadyDispatchedToNewTouchTarget = true;
        break;
    }

    // The accessibility focus didn't handle the event, so clear
    // the flag and do a normal dispatch to all children.
    ev.setTargetAccessibilityFocus(false);
}
```



```java
private static boolean canViewReceivePointerEvents(@NonNull View child) {
    return (child.mViewFlags & VISIBILITY_MASK) == VISIBLE
            || child.getAnimation() != null;
}
```



```java
protected boolean isTransformedTouchPointInView(float x, float y, View child,
                                                PointF outLocalPoint) {
    final float[] point = getTempPoint();
    point[0] = x;
    point[1] = y;
    transformPointToViewLocal(point, child);
    final boolean isInView = child.pointInView(point[0], point[1]);
    if (isInView && outLocalPoint != null) {
        outLocalPoint.set(point[0], point[1]);
    }
    return isInView;
}

public void transformPointToViewLocal(float[] point, View child) {
    point[0] += mScrollX - child.mLeft;
    point[1] += mScrollY - child.mTop;

    if (!child.hasIdentityMatrix()) {
        child.getInverseMatrix().mapPoints(point);
    }
}
```



**ViewGroup的dispatchTransformedTouchEvent方法**

```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
                                              View child, int desiredPointerIdBits) {
    final boolean handled;

    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
          	// 调用父类（即View）的dispatchTouchEvent()方法
            handled = super.dispatchTouchEvent(event);
        } else {
          	// 调用子View的dispatchTouchEvent()方法
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }

   	// 省略部分代码...
    return handled;
}
```



```java
if (mFirstTouchTarget == null) {
    // No touch targets so treat this as an ordinary view.
    handled = dispatchTransformedTouchEvent(ev, canceled, null,
            TouchTarget.ALL_POINTER_IDS);
}
```



### 2.3.View

**View的dispatchTouchEvent方法**

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    // If the event should be handled by accessibility focus first.
    if (event.isTargetAccessibilityFocus()) {
        // We don't have focus or no virtual descendant has it, do not handle the event.
        if (!isAccessibilityFocusedViewOrHost()) {
            return false;
        }
        // We have focus and got the event, then use normal event dispatch.
        event.setTargetAccessibilityFocus(false);
    }

    boolean result = false;

    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    }

    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        // Defensive cleanup for new gesture
        stopNestedScroll();
    }

    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
          	// 如果View设置了OnTouchListener，处于enable状态并且onTouch()方法返回true，则不会再调用onTouchEvent()方法
            result = true;
        }

        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }

    // Clean up after nested scrolls if this is the end of a gesture;
    // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
    // of the gesture.
    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }

    return result;
}
```

