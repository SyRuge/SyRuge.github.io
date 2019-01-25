---
title: View事件分发机制
date: 2019-01-25 17:51:09
tags: Android开发艺术探索笔记
categories: [Android开发艺术探索笔记]
---

### 相关方法

点击事件相关的三个方法：**dispatchTouchEvent**、**onInterceptTouchEvent**、**onTouchEvent**。

**public boolean dispatchTouchEvent(MotionEvent ev)**

用来进行事件分发，如果事件能够传递给当前View则一定会调用，返回值受当前View的onTouchEvent和下级View的dispatchTouchEvent影响。

<!--more-->

**public boolean onInterceptTouchEvent(MotionEvent ev)**

只存在于ViewGroup内，在dispatchTouchEvent内部调用，用来判断是否拦截某个事件。若当前ViewGroup拦截了某个事件，在同一个事件序列中，此方法不会再次调用，返回结果表示是否拦截当前事件。子View可以调用**requestDisallowInterceptTouchEvent(boolean)**不让父ViewGroup拦截某个事件，ACTION_DOWN事件除外，因为每次分发新的事件序列会重置此标记位。

**public boolean onTouchEvent(MotionEvent event)**

在dispatchTouchEvent内部调用，返回结果表示是否消费当前事件，如果不消耗，同一个时间序列中，无法再次接受到本次事件的其他事件。

伪代码：

```kotlin
override fun dispatchTouchEvent(ev: MotionEvent?): Boolean {
    var consume = false
    if (onInterceptTouchEvent(ev)) {
        consume = onTouchEvent(ev)
    } else {
        consume = child.dispatchTouchEvent(ev)
    }
    return consume
}
```

### 源码分析

事件分发经过Activity的dispatchTouchEvent会传递到window，window是以一个抽象类，PhoneWindow是它的唯一实现类，最终会传递到DecorView。DecorView继承与FrameLayout，最终事件进入了ViewGroup的dispatchTouchEvent中：

#### ViewGroup#dispatchTouchEvent

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            intercepted = true;
        }
    }
    ...
    return handled;
}
```

ViewGroup中当事件为ACTION_DOWN或者mFirstTouchTarget不为null时才去判断拦截事件，ACTION_DOWN好理解，当ViewGroup的子View成功处理事件时，mFirstTouchTarget会指向这个子View，即当ViewGroup不拦截事件时，mFirstTouchTarget不为null。从代码中看，当事件为MOVE和UP时，mFirstTouchTarget不为null，则不再执行onInterceptTouchEvent方法。并且同一个事件序列的其他事件都会交给这个子View。

那么之前说的requestDisallowInterceptTouchEvent可以指定父ViewGroup不拦截本事件以及对DOWN事件无效呢？

是否拦截事件取决于mFirstTouchTarget和disallowIntercept。

在DOWN事件首次进入时，调用了clearTouchTargets和resetTouchState方法：

```java
private void clearTouchTargets() {
    TouchTarget target = mFirstTouchTarget;
    if (target != null) {
        do {
            TouchTarget next = target.next;
            target.recycle();
            target = next;
        } while (target != null);
        mFirstTouchTarget = null;
    }
}
private void resetTouchState() {
    clearTouchTargets();
    resetCancelNextUpFlag(this);
    mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    mNestedScrollAxes = SCROLL_AXIS_NONE;
}
```

mFirstTouchTarget在第一次进入时赋值为null，disallowIntercept又在这个if判断的内部，因此对于DOWN事件，设不设置requestDisallowInterceptTouchEvent都无所谓，首次调用时重置了mGroupFlags，disallowIntercept为false，默认调用onInterceptTouchEvent方法。

当调用requestDisallowInterceptTouchEvent方法：

```java
@Override
public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {

    if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
        // We're already in this state, assume our ancestors are too
        return;
    }

    if (disallowIntercept) {
        mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
    } else {
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    }

    // Pass it up to our parent
    if (mParent != null) {
        mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
    }
}
```

requestDisallowInterceptTouchEvent(true)实际上是设置mGroupFlags这个变量，dispatchTouchEvent的disallowIntercept就会为true，因此onInterceptTouchEvent就不会调用，同时intercepted=false，不拦截本次事件。

接下来，当不拦截事件时：

```java
final int childrenCount = mChildrenCount;
if (newTouchTarget == null && childrenCount != 0) {
    final float x = ev.getX(actionIndex);
    final float y = ev.getY(actionIndex);
    // Find a child that can receive the event.
    // Scan children from front to back.
    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
    final boolean customOrder = preorderedList == null
            && isChildrenDrawingOrderEnabled();
    final View[] children = mChildren;
    for (int i = childrenCount - 1; i >= 0; i--) {
        final int childIndex = getAndVerifyPreorderedIndex(
                childrenCount, i, customOrder);
        final View child = getAndVerifyPreorderedView(
                preorderedList, children, childIndex);

        if (!canViewReceivePointerEvents(child)
                || !isTransformedTouchPointInView(x, y, child, null)) {
            ev.setTargetAccessibilityFocus(false);
            continue;
        }
        newTouchTarget = getTouchTarget(child);
        ...
        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
            // Child wants to receive touch within its bounds.
            ...
            newTouchTarget = addTouchTarget(child, idBitsToAssign);
            break;
        }
        // Dispatch to touch targets.
        if (mFirstTouchTarget == null) {
            // No touch targets so treat this as an ordinary view.
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                                                    TouchTarget.ALL_POINTER_IDS);
        }
    }
}
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
    final boolean handled;

    // Perform any necessary transformations and dispatch.
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        ...
        handled = child.dispatchTouchEvent(transformedEvent);
    }
    // Done.
    transformedEvent.recycle();
    return handled;
}
```

判断View是否在点击范围内，然后调用dispatchTransformedTouchEvent方法，注意，第三个参数child不为null，则调用了child的dispatchTouchEvent，假定它返回了true，则会执行addTouchTarget方法：

```java
private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
    final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```

mFirstTouchTarget指向了这个子View，因此下一个MOVE或者UP事件mFirstTouchTarget不为null，因此不会执行拦截，其余事件都会交给这个View。

当View没有消费事件，dispatchTransformedTouchEvent返回了false，则会执行dispatchTransformedTouchEvent，注意，此时第三个参数child为null，因此会执行super.dispatchTouchEvent，事件转入View处理。

#### View#dispatchTouchEvent

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    ...
    boolean result = false;
    ...
    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
    ...
    return result;
}
```

由于View不包含子View，不能向下传递事件，只能自己处理。首先会判断是否设置了OnTouchListener并且为enable可用状态，当OnTouchListener返回了true消费事件，那么onTouchEvent就不会调用。

onTouchEvent事件中，重点关注UP事件：

```java
public boolean onTouchEvent(MotionEvent event) {
    ...
    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;
    ...
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }

    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                ...
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    ...
                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        if (!focusTaken) {
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClick();
                            }
                        }
                    }
                    ...
                }
                mIgnoreNextUpEvent = false;
                break;
                ...
        }
        return true;
    }
    return false;
}
```

可以看到，当CLICKABLE或LONG_CLICKABLE并且为可点击状态(CONTEXT_CLICKABLE是鼠标点击的)，就会消费这个事件，onTouchEvent返回true，不管它是不是DISABLE状态。当UP事件时，会触发performClick方法，如果View设置了OnClickListener，最终会调用它的onClick方法，另外，setOnClickListener默认会将不可点击设置为可点击状态，LONG_CLICKABLE也是一样：

```java
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }
    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
    notifyEnterOrExitForAutoFillIfNeeded(true);
    return result;
}

public void setOnClickListener(@Nullable OnClickListener l) {
    if (!isClickable()) {
        setClickable(true);
    }
    getListenerInfo().mOnClickListener = l;
}
```