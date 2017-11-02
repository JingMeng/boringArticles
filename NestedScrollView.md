NestedScrollView 源码解析
=========
### 1. 序言
既然主题是NestedScrollView，你是不是应该知道它的名字中为什么含有Nested呢？”Nested”翻译成中文是”嵌套的、内装的”。NestedScrollView既然是被内嵌的View，则它应该会跟其它的view勾勾搭搭，扯不清关系吧。同时，名字中含有ScrollView这么熟悉的单词，则，它应该能跟我们熟悉的ScrollView一样，实现滑动和Fling效果吧。<br><br>
用过NestedScrollView的同学都已知道，NestedScrollView和CoordinatorLayout若是勾搭在一起，能实现视差滑动的效果。如果NestedScrollView独立成行，则它能实现跟ScrollView一样的功能。<br><br>
NestesdScrollView是什么？<br>
* 它是一个view;<br>
* 它是一个viewGroup;<br>
* 它是一个FrameLayout；<br>
* 它能跟ScrollView一样实现滑动效果；<br>
* 它能跟CoordinatorLayout勾搭在一起.<br>

本篇文章以第四点和第五点作为主线，以一种学习的心态分析一下Google的工程师们写下的这个滑动工具类。因本人才疏学浅，文中如有纰漏，或有不妥之处，请一一指正，也请一定联系我修改，感激不尽。<br>
那咱们继续切入正文-<br><br>

一般情况下，layout文件中使用NestedScrollView应该会这么写：<br>
```xml
<android.support.v4.widget.NestedScrollView
    android:id="@+id/scroll_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
      android:layout_width="match_parent"
      android:layout_height="wrap_content"
      android:orientation="vertical">
      
	    <!-- 若干子View -->
	    
    </LinearLayout>
</android.support.v4.widget.NestedScrollView>
```

就这么随意地在layout文件中写了这些，操作系统将依次调用：构造函数-->addView-->onMeasure-->onLayout.<br><br>
构造函数，无非就是将xml中定义的一些属性取出来，作为全局变量保存一下。然后对滑动相关的参数和工具类做一些初始化的工作，比如ViewConfiguration参数、ScrollerCompat、NestedScrollingParentHelper、NestedScrollingChildHelper等等。NestedXXXHelper看不懂没关系，你只需知道他们在构造函数中就已经存在就好了。构造函数之后，接下来的addView/onMeasure/onLayout咱们暂时抛开不谈，毕竟网上关于viewgroup绘制过程分析的文章已经满天飞了。<br><br>
本文的重心是NestedScrollView的滑动功能，NestedScrollView的滑动过程一定与DispatchTouchEvent、OnInterceptTouchEvent和OnTouchEvent息息相关。如果你不理解touch的分发、拦截、传递、消费机制，请先学习一下这个：[Android事件传递机制分析](http://www.jianshu.com/p/1528eb2ee54b)。否则，接下来的源码分析你可能会跌入云里雾里。<br>
### 2. 躲不掉的Touch分发和拦截
我们知道MotionEvent的dispatch和intercept是由上往下，由parent传递给child。一旦中间返回true，MotionEvent中断，便无法再向下传递。而MotionEvent的消费是由下往上，如果child的OnTouchEvent消费掉MotionEvent，则MotionEvent无法再向上传递，parent就再无机会消费这个Touch事件了。此处我有一个问题：一个ViewGroup的onTouchEvent总是返回true，那么这个ViewGroup的onInterceptTouchEvent能否收到连续的ACTION_MOVE事件?<br><br>
答案应该是<b>不能</b>。<br><br>
我在NestedScrollView里面填入了若干个TextView，NestedScrollView的onInterceptTouchEvent函数收不到任何一个ACTION_MOVE事件。<br><br>
我在NestedScrollView里面的第一个TextView加上了clickable="true"的属性，我手指按下第三个TextView开始拖动，onInterceptTouchEvent也收不到任何一个ACTION_MOVE时间。<br><br>
我按下了NestedScrollView里面第一个clickable的TextView，onInterceptTouchEvent收到了ACTION_MOVE事件。但是ACTION_MOVE事件第二次的时候，onInterceptTouchEvent返回了true，然后就再也收不到MotionEvent事件了。<br><br>
所以，这个例子包含了MotionEvent传递的细节，请各位观众仔细揣测。如果子View没有消费掉ACTION_DOWN事件，则其父ViewGroup在onInterceptTouchEvent收不到任何的ACTION_MOVE事件，所有的MotionEvent都会抛给ViewGroup的OnTouchEvent去处理。<br><br>
更近一步的理解应该是这样：一个view一旦不消费某个MotionEvent，后续的MotionEvent序列都不会再传递给这个view去处理了，直到下一次ACTION_DOWN事件，与此同时，上一级parentView的onInterceptTouchEvent就不会再调用了，因为子View不消费MotionEvent，parent还拦截个啥呢？<br>
### 3. NestedScrollView之ScrollView
言归正传，NestedScrollView具备滑动功能，此处你需要知道的是：NestedScrollView的父类是FrameLayout，FrameLayout对TouchEvent的处理没有任何定制，FrameLayout所有的TouchEvent处理都交给了它的父类ViewGroup。NestedScrollView对TouchEvent的两个入口做了定制：onInterceptTouchEvent和onTouchEvent。<br><br>
先看一下onInterceptTouchEvent，这个函数的字面意思是：Touch事件拦截。<br><br>
```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    // MotionEvent拦截，如果返回true，MotionEvent交给TouchEvent去处理
    // 如果返回false，MotionEvent传递给子View   

    final int action = ev.getAction();
    if ((action == MotionEvent.ACTION_MOVE) && (mIsBeingDragged)) {
        // 如果正在move而且被定性为正在拖拽中，直接返回true，将MotionEvent交给自己的onTouchEvent去处理
        return true;
    }

    switch (action & MotionEventCompat.ACTION_MASK) {
        case MotionEvent.ACTION_MOVE: {
            // 下面这么多代码大多是为了给mIsBeingDragged定性

	          /************ 若干代码略去 ************/

            final int y = (int) MotionEventCompat.getY(ev, pointerIndex);
            final int yDiff = Math.abs(y - mLastMotionY); // 垂直滑动的距离
            if (yDiff > mTouchSlop
                    && (getNestedScrollAxes() & ViewCompat.SCROLL_AXIS_VERTICAL) == 0) {
                // 如果垂直拖动距离大于mTouchSlop，就认定是正在scroll
                mIsBeingDragged = true;
  	            // 保存一些变量，速度跟踪初始化
                mLastMotionY = y;
                initVelocityTrackerIfNotExists();
                mVelocityTracker.addMovement(ev);
                mNestedYOffset = 0;
                final ViewParent parent = getParent();
                if (parent != null) {
                    // 如果认定了是scrollView滑动，则不让父类拦截，后续所有的MotionEvent都会有NestedScrollView去处理
                    parent.requestDisallowInterceptTouchEvent(true);
                }
            }
            break;
        }

        case MotionEvent.ACTION_DOWN: {
	  
            /************ 若干代码略去 ************/

            // 速度跟踪
            initOrResetVelocityTracker();
            mVelocityTracker.addMovement(ev);
            mScroller.computeScrollOffset();
            mIsBeingDragged = !mScroller.isFinished(); // mIsBeingDragged跟是否fling有关
            // 请格外关注下，因为startNestedScroll跟，因为它跟Behavior的一个成员函数重名
            startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL);
            break;
        }

        case MotionEvent.ACTION_CANCEL:
        case MotionEvent.ACTION_UP:
            // 手指松开
            mIsBeingDragged = false;
            mActivePointerId = INVALID_POINTER;
            recycleVelocityTracker();
            if (mScroller.springBack(getScrollX(), getScrollY(), 0, 0, 0, getScrollRange())) {
                ViewCompat.postInvalidateOnAnimation(this);
            }
            // stopNestedScroll跟Behavior的一个成员函数重名
            stopNestedScroll();
            break;
        case MotionEventCompat.ACTION_POINTER_UP:
            onSecondaryPointerUp(ev);
            break;
    }

    // 返回值也好理解：如果正在拖拽中，则返回true，告诉系统MotionEvent交给NestedScrollView的OnTouchEvent去处理
    // 如果没有拖拽，比如ACTION_DOWN、ACTION_UP内部Button点击的MotionEvent，返回false，MotionEvent传递给子View
    return mIsBeingDragged;
}
```
上面的onInterceptTouchEvent函数，贴上了关键的代码。这个函数也还比较容易理解。毕竟该函数负责拦截，不会将Scroll/Fling效果的功能代码写在这里。该函数主要是给mIsBeingDragged这个flag定性。一旦定性为上下拖动，就不再将MotionEvent传递给子View。然而，此处我们应该格外关注的是上面出现了startNestedScroll和stopNestedScroll这两个看起来比较敏感的函数调用。因为它们跟Behavior的两个函数重名。此处，我主观猜测它们会跟Behavior纠缠不清，以其中的startNestedScroll函数为例，贴上代码：<br><br>
```java
@Override
public boolean startNestedScroll(int axes) {
    return mChildHelper.startNestedScroll(axes);
}
```
居然就这一行代码！mChildHelper是NestedScrollingChildHelper对象，在最初的最初，我在构造函数中提到了它，还有印象么？我敢打包票，mChildHelper一定做了非常多的事情，否则Behavior怎么会跟它那么像。注意到NestedScrollView调用startNestedScroll的时候并没有关心返回值，此处我们也不关心返回的true还是false。下面载入mChildHelper的startNestedScroll函数：<br><br>
```java
public boolean startNestedScroll(int axes) {
    if (hasNestedScrollingParent()) {
        // 如果已经设置了NestedParent，啥都不用做了
        return true;
    }
    // 此处isNestedScrollingEnabled依赖于一个全局变量mIsNestedScrollingEnabled
    // 在NestedScrollView的构造函数中，这个flag被设置成了true，这个if分支一定能进得去
    if (isNestedScrollingEnabled()) {
        // mView就是NestedScrollView，构造函数中被初始化
        ViewParent p = mView.getParent();
        View child = mView; 
        while (p != null) {
            // 这个for循环，就是一直不断的寻找支持nested功能的ancestorView
            // 卧槽，如果外层View有一个CoordinatorLayout，则这个NestedScrollView就能勾搭上CoordinatorLayout了
            // 下面的函数onStartNestedScroll和onNestedScrollAccepted，应该和Behavior不远了
            if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes)) {
  	            // 找到了支持Nested功能的ancestorView，保存一下
                mNestedScrollingParent = p;
                ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes);
                return true;
            }
            if (p instanceof View) {
                child = (View) p;
            }
            p = p.getParent();
        }
    }
    return false;
}
```
这个函数的主要功能是找到祖先View中最近的mNestedScrollingParent，mNestedScrollingParent是一个支持Nested滑动的ancestorView。mNestedScrollingParent一旦找到，目测onStartNestedScroll和onNestedScrollAccepted已经跟Behavior不远了。此处我们先暂停，后面我们再回来。因为我们的第一个目标是看NestedScrollView怎么实现滑动的。况且，当layout文件中根View就是NestedScrollView时，startNestedScroll函数是找不到mNestedScrollingParent的。<br><br>
NestedScrollView实现滑动效果，当然要看OnTouchEvent:<br><br>
```java
@Override
public boolean onTouchEvent(MotionEvent ev) {
    initVelocityTrackerIfNotExists();
    MotionEvent vtev = MotionEvent.obtain(ev);
    final int actionMasked = MotionEventCompat.getActionMasked(ev);
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        // 这个是一个很重要的参数，视差值初始化为0
        mNestedYOffset = 0;
    }
    // 我们知道CoordinatorLayout和AppbarLayout视差滑动的时候，有悬停效果
    // mNestedYOffset记录的是悬停时候的scroll视差值
    vtev.offsetLocation(0, mNestedYOffset);

    switch (actionMasked) {
        case MotionEvent.ACTION_DOWN: {
            if (getChildCount() == 0) {
                return false;
            }
            if ((mIsBeingDragged = !mScroller.isFinished())) {
                final ViewParent parent = getParent();
                if (parent != null) {
                    // 如果按下的时候还在fling动画，就直接受理这个MotionEvent
 	                  // 告诉祖先view不用拦截了，后续的TouchEvent事件统一由NestedScrollView来消费 
                    parent.requestDisallowInterceptTouchEvent(true);
                }
            }

            // 手指按下，如果正在fling中，就停止fling
            if (!mScroller.isFinished()) {
                mScroller.abortAnimation();
            }

            mLastMotionY = (int) ev.getY();
            mActivePointerId = MotionEventCompat.getPointerId(ev, 0);
	          // 下面这个函数，在onInterceptTouchEvent中已经介绍过了，就是去勾搭支持Nested功能的祖先view
            startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL);
            break;
        }
        case MotionEvent.ACTION_MOVE:
            
	          /************ 若干代码略去 ************/

            final int y = (int) MotionEventCompat.getY(ev, activePointerIndex);
            int deltaY = mLastMotionY - y;
            // dispatchNestedPreScroll格外关注下
	          // 祖先view会根据deltaY和mScrollOffset来决定是否消费这个touch事件
            // 如果祖先view决定消费这个MotionEvent，会把结果写在mScrollConsumed和mScrollOffset中
            if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset)) {
                deltaY -= mScrollConsumed[1];
                // 祖先View消费了MotionEvent，引入视差值
 	              // 根据视差值，调整MotionEvent etev
                vtev.offsetLocation(0, mScrollOffset[1]);
                mNestedYOffset += mScrollOffset[1];
            }
            if (!mIsBeingDragged && Math.abs(deltaY) > mTouchSlop) {
	              // 给mIsBeingDragged 定性
                final ViewParent parent = getParent();
                if (parent != null) {
                    // 被定性为滑动了，就不让父View拦截了
                    parent.requestDisallowInterceptTouchEvent(true);
                }
                mIsBeingDragged = true;
                if (deltaY > 0) {
                    deltaY -= mTouchSlop;
                } else {
                    deltaY += mTouchSlop;
                }
            }
            if (mIsBeingDragged) {
                // 已被定性为拖拽
	              /************ 若干代码略去 ************/

	              // 根据当前的scrollY和deltaY，scroll到某一个特定的位置
                if (overScrollByCompat(0, deltaY, 0, getScrollY(), 0, range, 0,
                     0, true) && !hasNestedScrollingParent()) {
		              // 如果没有overScroll且没有支持nested功能的父View，速度追踪重置
    		          mVelocityTracker.clear();
	              }

                final int scrolledDeltaY = getScrollY() - oldY;
                final int unconsumedY = deltaY - scrolledDeltaY;
                // 每一次拖动都需要NestedParentView去计算是否视差了
                if (dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset)) {
	                  // 父View为了视差消费了这次MotionEvent
                    mLastMotionY -= mScrollOffset[1];
                    vtev.offsetLocation(0, mScrollOffset[1]);
                    mNestedYOffset += mScrollOffset[1];
                }
	              /************ 若干代码略去 ************/

            }
            break;
        case MotionEvent.ACTION_UP:
            if (mIsBeingDragged) {
                // 手指松开，根据fling的速度滑动下去
                final VelocityTracker velocityTracker = mVelocityTracker;
                velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
                int initialVelocity = (int) VelocityTrackerCompat.getYVelocity(velocityTracker,
                        mActivePointerId);

                if ((Math.abs(initialVelocity) > mMinimumVelocity)) {
                    // 该函数内部调用了dispatchNestedPreFling和dispatchNestedFling跟Behavior挂钩
                    // 同时也用mScroller实现了fling功能
                    flingWithNestedDispatch(-initialVelocity);
                } else if (mScroller.springBack(getScrollX(), getScrollY(), 0, 0, 0,
                        getScrollRange())) {
                    ViewCompat.postInvalidateOnAnimation(this);
                }
            }
            mActivePointerId = INVALID_POINTER;
            endDrag(); // 此处调用了stopNestedScroll函数
            break;
    }

    /************ 若干代码略去 ************/

    // 默认情况下，如果NestedScrollView有机会消费MotionEvent，就一定会消费掉的
    return true;
}
```

上述的代码中，ACTION_MOVE实现scroll滑动功能比较隐晦，在一个if语句中，一方面做了是否OverScroll的判断，另一方面又做了scrollTo的工作。在ACTION_UP的代码段中，NestedScrollView根据当前的滑动速度，使用mScroller将NestedScrollView的元素fling到目标位置。<br><br>
NestedScrollView的滑动功能，应该大致如此了。有些细节的知识点，限于篇幅问题，我并没有跟进去一探究竟。<br><br>
然而NestedScrollView，这个单词一分为二是Nested和ScrollView，上面的一坨分析是有关ScrollView的，却一直回避了这个更靠前的单词：Nested。不过，还好我们之前做了一个铺垫。<br>
### 4. NestedScrollView之Nested
还记得前面我们跟到了mChildHelper.startNestedScroll函数么，那个函数的主要工作就是要找到一个支持nested功能的mNestedScrollingParent。哦，其实应该是ancestorView。明眼人掐指一算，这个支持nested功能的view不就是我们熟悉的CoordinatorLayout么？此处，我们先建立一个大前提，layout文件中我们让NestedScrollView支持视差，使用CoordinatorLayout和AppbarLayout，xml如下：<br><br>
```xml
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/main_content"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <android.support.design.widget.AppBarLayout
        android:id="@+id/appbar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:elevation="6dp"
            app:layout_scrollFlags="scroll|enterAlways|snap"
            app:navigationIcon="?attr/homeAsUpIndicator" />

        <Button
            android:layout_width="fill_parent"
            android:layout_height="60dp"
            android:background="#111111"
            android:gravity="center"
            android:textColor="#ffffff"
            android:text="这个是悬停按钮" />

    </android.support.design.widget.AppBarLayout>

    <nested.stone.com.nestedscrolldemo.CoordinatorNestedScrollView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:scrollbars="vertical"
        app:layout_behavior="@string/appbar_scrolling_view_behavior">

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical">

	          <!--xxxxxxxxxxxxxxx若干子Viewxxxxxxxxxxxxxxxx-->
     </LinearLayout>
    </nested.stone.com.nestedscrolldemo.CoordinatorNestedScrollView>
</android.support.design.widget.CoordinatorLayout>
```

接下来我们回归java代码，mChildHelper.startNestedScroll可以找到支持nested功能的parentView，那一段代码是这样的：<br><br>
```java
if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes)) {
    mNestedScrollingParent = p;
    ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes);
    return true;
}
```
其中ViewParentCompat.onStartNestedScroll就是判断mView的某一个祖先视图p是否支持nested操作的。我们继续跟进去ViewParentCompat看看：<br><br>
```java
static final ViewParentCompatImpl IMPL;
static {
    final int version = Build.VERSION.SDK_INT;
    if (version >= 21) {
        IMPL = new ViewParentCompatLollipopImpl();
    } else if (version >= 19) {
        IMPL = new ViewParentCompatKitKatImpl();
    } else if (version >= 14) {
        IMPL = new ViewParentCompatICSImpl();
    } else {
        IMPL = new ViewParentCompatStubImpl();
    }
}
public static boolean onStartNestedScroll(ViewParent parent, View child, View target,
        int nestedScrollAxes) {
    return IMPL.onStartNestedScroll(parent, child, target, nestedScrollAxes);
}
```
由于操作系统版本的不同，IMPL有不同的实现类。我们看一下支持安卓5.0以后的这个兼容类：ViewParentCompatLollipopImpl<br><br>
```java
public static boolean onStartNestedScroll(ViewParent parent, View child, View target,
        int nestedScrollAxes) {
    try {
        return parent.onStartNestedScroll(child, target, nestedScrollAxes);
    } catch (AbstractMethodError e) {
        Log.e(TAG, "ViewParent " + parent + " does not implement interface " +
                "method onStartNestedScroll", e);
        return false;
    }
}
```
在上面的函数中，parent是NestedScrollView的某一个祖先视图。由于我们在xml中定义了CoordinatorLayout，所以这个parent就应该是CoordinatorLayout。然后呢，调用了这个parent.onStartNestedScroll函数，我们看一下CoordinatorLayout有没有这个函数。<br><br>
```java
public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
    boolean handled = false;

    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        // 遍历每一个子View
        final View view = getChildAt(i);
        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
            final boolean accepted = viewBehavior.onStartNestedScroll(this, view, child, target,
                    nestedScrollAxes);
            handled |= accepted;

            lp.acceptNestedScroll(accepted);
        } else {
            lp.acceptNestedScroll(false);
        }
    }
    return handled;
}
```
你看到了么，CoordinatorLayout有这个函数。在这个函数中，它遍历了CoordinatorLayout的所有子view，并找到了子View中对应的LayoutParams.Behavior。所以，AppbarLayout的Behavior就能排上用场了。它接下来会跳转到AppbarLayout.Behavior.onStartNestedScroll了。接下来我们就不跟了吧，这个只是nested功能开始的一个回调了。<br><br>
接下来我们关心视差吧，回到NestedScrollView.onTouchEvent里面的ACTION_MOVE代码段：<br><br>
```java
if (dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset)) {
    mLastMotionY -= mScrollOffset[1];
    vtev.offsetLocation(0, mScrollOffset[1]);
    mNestedYOffset += mScrollOffset[1];
}
```
之所以挑出来这一段，是因为dispatchNestedScroll比较眼熟。我们跟进去dispatchNestedScroll瞅瞅：<br><br>
```java
@Override
public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed,
        int dyUnconsumed, int[] offsetInWindow) {
    return mChildHelper.dispatchNestedScroll(dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed,
            offsetInWindow);
}
```
又是mChildHelper，我之前说过了它很强大，现在信了吧。此处我们跟进去这个mChildHelper.dispatchNestedScroll函数看看：<br><br>
```java
public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed,
        int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow) {
    if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
        if (dxConsumed != 0 || dyConsumed != 0 || dxUnconsumed != 0 || dyUnconsumed != 0) {
            int startX = 0;
            int startY = 0;
            if (offsetInWindow != null) {
                mView.getLocationInWindow(offsetInWindow);
                startX = offsetInWindow[0];
                startY = offsetInWindow[1];
            }

            ViewParentCompat.onNestedScroll(mNestedScrollingParent, mView, dxConsumed,
                    dyConsumed, dxUnconsumed, dyUnconsumed);

            if (offsetInWindow != null) {
                mView.getLocationInWindow(offsetInWindow);
                offsetInWindow[0] -= startX;
                offsetInWindow[1] -= startY;
            }
            return true;
        } else if (offsetInWindow != null) {
            // No motion, no dispatch. Keep offsetInWindow up to date.
            offsetInWindow[0] = 0;
            offsetInWindow[1] = 0;
        }
    }
    return false;
}
```
在这个函数中，支持nested功能的parentView如果愿意消费MotionEvent，就返回true，如果不愿意就返回false。此外还有两个参数dyUnconsumed和offsetInWindow，既是入参也是出参，意为子View愿意消费多少dy。ViewParentCompat.onNestedScroll就是真正把愿意消费的视差信息传递给CoordinatorLayout和Behavior了。后续的逻辑跟踪，你应该也大致看的懂了。<br><br>
读到这份源代码，我一直忽略了坐标值细节的一针一眼的计算。因为比较费时间，费精力。有兴趣的朋友私下去感受就好。<br><br>
当然，滑动时候，跟CoordinatorLayout和Behavior打交道还有其它一些函数，比如stopNestedScroll和dispatchNestedFling等。主要的一个逻辑，都是类似的，就是通过mChildHelper找到mNestedScrollingParent，然后再由mNestedScrollingParent找到对应子View的Behavior。限于篇幅问题，此处不再赘述了。<br>

### 5. 总结
一个View(或其子View)一旦不消费某一个MotionEvent事件，该View便再也捕获不到这个后续的MotionEvent序列。这样就使得“视差”这类的交互逻辑无法实现。<br><br>
试想，如果是我们自身去设计这个“视差”，以AppbarLayout的Behavior效果为例，常规思路应该是这样的：<br><br>
(1)当我们向上滑动视图的时候，MotionEvent由CoordinatorLayout去消费，这个时候NestedScrollView内部并没有滑动，只是CoordinatorLayout实现了位移。<br><br>
(2)当toolbar悬停的时候，我们再继续向上拖拽，MotionEvent由NestedScrollView去消费，CoordinatorLayout不去拦截，视图的滑动是NestedScrollView内部的滑动。<br><br>
这样的常规思路，对我们普通人来说，或许更容易理解。但是在安卓的Touch传递机制来看，这却是无解的。至于为何无解的原因，前面已经提及多次。Google的工程师们在最初设计这一套Touch机制的时候，应该是没有问题的。只是随着时间不断往前推移，出现了视差这样更复杂的交互逻辑。<br><br>
Behavior的出现，并没有改变Touch自身的机制，它只是钻了之前MotionEvent的一个空子。通过调用MotionEvent的offsetLocation方法，巧妙地解决了视差问题。在layout文件可视化的那个design页签上面，CoordinatorLayout的第二个子View是越过屏幕高度的。单从这个方面，我们也能猜测AppbarLayout的Behavior的工作机制了：MotionEvent事件一直由NestedScrollView去消费，只是在消费的同时，也通过Behavior影响了CoordinatorLayout的上下位移量。<br><br>
