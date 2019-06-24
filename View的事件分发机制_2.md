# View的事件分发机制_2
三个重要方法：
`public boolean dispatchTouchEvent(MotionEvent ev)`
事件分发，如果事件传递到 View ， 该方法就一定会执行。 返回值表示是否消费事件。
`public boolean onInterceptTouchEvent(MotionEvent ev)`
ViewGroup 中的方法， 在 dispatchTouchEvent 内部调用， 判断是否拦截某个方法， 如果当前 View 拦截某个事件， 那么在**同一事件序列**中， 该方法不再被调用。 返回值表示是否拦截事件。
`public boolean onTouchEvent(MotionEvent event)`
在 dispatchTouchEvent 中调用， 处理点击事件。 返回结果表示是否消费当前事件， 如果不消费， 在同一个事件序列中， 当前 View 无法再接收到事件， 而是交给父 View 的ouTouchEvent 处理。

当一个事件发生后， 其传递过程为： **Activity -> Window -> View**
**事件分发的源码解析：**
```
Activity------>
 public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            // 事件开始分发时首先会调用，默认为空实现
            onUserInteraction();
        }
         // 将事件分发到 Window，Window 进行分发，如果返回 true ，事件就结束， 如果都没人处理， Activity 的 onTouchEvent 会被调用
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }


Activity------>
public Window getWindow() {
        // 返回 Window 对象
        return mWindow;
    }


Activity------>mWindow被赋值， PhoneWindow 是 Window 唯一的实现类
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);
        //  mWindow 赋值， PhoneWindow 是 Window 的唯一子类
        mWindow = new PhoneWindow(this, window, activityConfigCallback);



PhoneWindow------> PhoneWindow 将事件分发到mDector, 也就是DecorView， 即 Activity 的根 View
public class PhoneWindow extends Window implements MenuBuilder.Callback {
......
private DecorView mDecor;
......
......
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDector.superDispatchTouchEvent(event);
  }
}



DecorView------> DecorView 调用super dispatchTouchEvent， DecorView 继承至 FrameLayout， 最终事件将会传递到 View， 在 Activity 中通过 setContentView 设置的 View 是 DecorView 的子 View，而这里的 View 一般都是 ViewGroup， 所以接下来关注 ViewGroup 的 disPatchTouchEvent
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
......
......
......
public boolean superDispatchTouchEvent(MotionEvent event) {
            // 
            return super.dispatchTouchEvent(event);
        }
}




ViewGroup(dispatchTouchEvent)---------> ViewGroup 的 dispatchTouchEvent 开始 进行事件分发
......
            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                /***
                 * 1. DOWN 动作触发时，将 TouchTarget 对象清空， 也就是事件接收到的对象。mFirstTouchTarget 被重置为 null，
                 * mFirstTouchTarget 可以理解为当事件被 ViewGroup 的子 View 处理时，mFirstTouchTarget 被赋值并指向子 View
                 * 2. 重置触摸状态准备一个新的事件序列，resetTouchState 中将标志位 FLAG_DISALLOW_INTERCEPT 重置
                 */
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                /**
                 * 1. DOWN 事件下或者 mFirstTouchTarget != null（也就是有子 View 成功处理事件）才会调用 onInterceptTouchEvent 来判断是否拦截事件，
                 *    如果 mFirstTouchTarget 为 null， 也就是拦截当前事件，MOVE UP 事件都不再调用 onInterceptTouchEvent，而且当前事件序列中的其他
                 *    事件均由当前 View 的 onTouchEvent 处理。
                 * 2. onInterceptTouchEvent 的调用还被标志位 FLAG_DISALLOW_INTERCEPT 影响， 子 View 可通过 requestDisallowInterceptTouchEvent
                 *    方法来请求父类不进行拦截， 但是这个只是在除 DOWN 事件之外的事件序列中，因为上面在 DOWN 方法时 resetTouchState 方法将                   FLAG_DISALLOW_INTERCEPT
                 *    标志位重置了。当子 View 调用 requestDisallowInterceptTouchEvent 方法返回 true 时，也就是请求父类不进行拦截，onInterceptTouchEvent
                 *    将不被调用，父 View 直接不进行拦截。
                 * 3. 由此可见， 如果想提前处理所有的点击事件， 应该在 dispatchTouchEvent 方法中处理，onInterceptTouchEvent 并不是每次都会调用
                 *    FLAG_DISALLOW_INTERCEPT 标记位可用于处理滑动冲突。
                 */
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

ViewGroup(dispatchTouchEvent)------> 当 ViewGroup 不拦截事件时， 即上面的 intercepted = true 事件会向下分发由它的子 View 处理
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            /**
                             * 遍历子 View， 顺序为自上而下
                             */
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

                            /**
                             *  判断当前子 View ： 1. 可见状态并且没有动画执行
                             *                    2. 点击区域在当前子 View 区域内
                             *  如果满足这两个条件中的一个， 事件就会传递给它处理
                             */
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
                            /**
                             * dispatchTransformedTouchEvent 方法中，如果 child 不为空， 那么将调用 child.dispatchTouchEvent(event) 方法将事件传递给了 child
                             * 如果 child.dispatchTouchEvent(event) 返回true， 事件将在子元素内部进行分发，跳出循环。 如果是 false， 将事件分发给下一个子元素
                             */
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
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
                                /**
                                 *  在 addTouchTarget 中 mFirstTouchTarget 被赋值， 其实就是事件传递给 child 处理时，mFirstTouchTarget 就会指向该 child
                                 */
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
                            .........
                            .........
                             // Dispatch to touch targets.
                            if (mFirstTouchTarget == null) {
                                // No touch targets so treat this as an ordinary view.
                                /**
                                 *  如果遍历完子 View 都没有对事件做处理，可能是该 ViewGroup 没有子 View，或者是子 View 处理了事件但是在 dispatchTouchEvent 中返回了 false
                                 *  也就是这里的 child 为 null， 那么就将调用 super.dispatchTouchEvent(event); 这里就到了 View 的 dispatchTouchEvent
                                 */
                                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                                        TouchTarget.ALL_POINTER_IDS);
                            }


View(dispatchTouchEvent)-------> 开始分析 View 的 dispatchTouchEvent
......
......
 /**
         * 判断窗口是否被遮挡， 返回 true 则继续向下分发
         */
        if (onFilterTouchEventForSecurity(event)) {
            // 当前 View 是否被激活， 并且有滚动事件
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            /**
             *  首先判断是否有设置 onTouchListener， 如果有设置并且其中的 onTouch 返回 true， 那么 onTouchEvent 将不会执行
             *  如果 onTouch 返回 false 或者其他条件不满足， 将执行 View 的 onTouchEvent 进行对事件的处理
             *  由此可见 onTouchListener 优先级高于 onTouchEvent
             */
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
        ......
        ......
        return result;




View(onTouchEvent)-------> 最后是由 View 的 onTouchEvent 来处理
        /**
         *  如果当前 View 是 disable 状态， 仍然是 return 消耗事件
         */
        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return clickable;
        }
        if (mTouchDelegate != null) {
            // 如果该 View 有设置代理，那么该事件就由 setTouchDelegate 的代理处理
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        /**
         * 只要是 clickable 为 true（ CLICKABLE，LONG_CLICKABLE，CONTEXT_CLICKABLE 只要有一个为 true ）， 那么就将消费事件，直接返回true
         */
        if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
            switch (action) {
                case MotionEvent.ACTION_UP:
                ......
                ......
                            // Only perform take click actions if we were in the pressed state
                            /**
                             * 在 ACTION_UP 中会触发 performClick， 如果有设置 onClickListener， 就会触发 onClick 方法
                             */
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClickInternal();
                                }
                            }
                        }
                        ......
                        ......
                        }




View(setOnclickListener/setOnLongClickListener)------>View 的LONG_CLICKABLE 属性都默认为 false，而 CLICKABLE 属性的值根据 View 是否可点击决定

    /**
     * setOnclickListener 会自动将 View 的 CLICKABLE 属性设置为 true
     * setOnLongClickListener 会自动将 View 的LONG_CLICKABLE 属性设置成 true
     */
public void setOnClickListener(@Nullable OnClickListener l) {
        if (!isClickable()) {
            setClickable(true);
        }
        getListenerInfo().mOnClickListener = l;
    }

public void setOnLongClickListener(@Nullable OnLongClickListener l) {
        if (!isLongClickable()) {
            setLongClickable(true);
        }
        getListenerInfo().mOnLongClickListener = l;
    }

```

**关于事件传递机制的一些结论（ Android 开发艺术探索 ）**
1. 同一个事件序列是指从手指接触屏幕的那一刻起， 到手指离开屏幕的那一刻结束， 在这个过程中所产生的的一系列事件， 这个事件序列以 down 事件开始， 中间含有数量不定的 move 事件， 最终以 up 事件结束。
2. 正常情况下， 一个事件序列只能被一个 View 拦截消耗， 因为一旦一个元素拦截了某事件， 那么同一个事件序列内的所有事情都会直接交给它处理， 因此同一个事件序列中的事件不能发别由两个 View 同时处理， 但是通过特殊手段可以做到， 比如一个 View 将本该自己处理的事件通过 onTouchEvent 强行传递给其他 View 处理。
3. 某个 View 一旦决定拦截， 那么这一个事件序列都只能由它来处理（ 如果事件序列能传递给它的话 ），并且它的 onInterceptTouchEvent 不会再被调用。 就是说当一个 View 决定拦截一个事件后， 那么系统会把同一个事件序列内的其他方法都交给它来处理， 因此就不用再调用 onInterceptTouchEvent 来询问是否拦截。
4. 某个 View 一旦开始处理事件， 如果它不消耗 ACTION_DOWN 事件 （ onTouchEvent 返回了 false），那么同一事件序列中的其他事情都不会再交给它来处理， 并且事件将重新交由它的父元素去处理， 即父元素的 onTouchEvent 会被调用。 意思就是事件一旦交给一个 View 处理， 那么它就必须消耗掉， 否者同一事件序列中其他事件就不会再交给它处理。
5. 如果 View 不消耗除 ACTION_DOWN 以外的其他事件，那么这个点击事件会消失， 此时父元素的 onTouchEvent 并不会被调用， 并且当前 View 可以持续收到后续的事件， 最终这些消失的点击事件会传递给 Activity 处理。
6.  ViewGroup 默认不拦截任何事件。 ViewGroup 的onInterceptTouchEvent 默认返回 false。
7.  View 没有 onInterceptTouchEvent 方法， 一旦有事件传递给它的， 那么 onTouchEvent 就会被调用。（ 对这个结论感觉有点问题， View 设置了 onTouchListener 并且返回为 true 的时候 onTouchEvent 就不会被调用）。
8. View 的onTouchEvent 默认都会消耗事件， 除非是不可点击的 （clickable， longClickable，contextClickable 同时为false）。 View 的 longClickable 默认都为 false， clickable 分情况， 比如 Button 的clickable 属性为 true， TextView 为 false。
9. View 的 enable 属性不影响 onTouchEvent 的默认返回值。 哪怕一个 View 是disable 状态的， 只要它的 clickable 或者 longClickable，contextClickable 有一个为 true ，那么它的 onTouchEvent 就返回 true。
10. onClick 会发生的前提是当前 View 是可点击的，并且它收到了 down 和 up 事件。
11. 事件传递过程是由外向内的， 即事件总是先传递给父元素， 然后再由父元素分发给子 View， 通过 requestDisallowInterceptTouchEvent 方法可以在子元素中干预父元素的事件分发过程， 但是 ACTION_DOWN 事件除外。

最后推荐一个很赞的流程图（ 来自 https://www.jianshu.com/p/238d1b753e64）
![事件分发机制流程图](_v_images/20190624162812449_17456.png)