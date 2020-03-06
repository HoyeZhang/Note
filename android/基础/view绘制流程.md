# 整体流程
- View 绘制中主要流程分为measure，layout， draw 三个阶段。

- measure ：根据父 view 传递的 MeasureSpec 进行计算大小。

- layout ：根据 measure 子 View 所得到的布局大小和布局参数，将子View放在合适的位置上。

- draw ：把 View 对象绘制到屏幕上。
- 那么发起绘制的入口在哪里呢？

- 在介绍发起绘制的入口之前，我们需要先了解Window，ViewRootImpl，DecorView之间的联系。

- 一个 Activity 包含一个Window，Window是一个抽象基类，是 Activity 和整个 View 系统交互的接口，只有一个子类实现类PhoneWindow，提供了一系列窗口的方法，比如设置背景，标题等。一个PhoneWindow 对应一个 DecorView 跟 一个 ViewRootImpl，DecorView 是ViewTree 里面的顶层布局，是继承于FrameLayout，包含两个子View，一个id=statusBarBackground 的 View 和 LineaLayout，LineaLayout 里面包含 title 跟 content，title就是平时用的TitleBar或者ActionBar，contenty也是 FrameLayout，activity通过 setContent（）加载布局的时候加载到这个View上。ViewRootImpl 就是建立 DecorView 和 Window 之间的联系。

这三个阶段的核心入口是在 ViewRootImpl 类的 performTraversals() 方法中。
<pre>
private void performTraversals() {
    ......
    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
    ......
    mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    ......
    mView.layout(0, 0, mView.getMeasuredWidth(), mView.getMeasuredHeight());
    ......
    mView.draw(canvas);
    ......
 }
</pre>
- mView就是decorview，在activity创建时建立两者的联系
# 理解MeasureSpec
- MeasureSpec很大程度上决定一个View的尺寸规格，测量过程中，系统会将View的layoutParams根据父容器所施加的规则转换成对应的MeasureSpec，再根据这个measureSpec来测量出View的宽/高。即对于一般view来说它的measureSpec由父容器和自身的layoutParams来决定
- MeasureSpec代表一个32位的int值，高2位为SpecMode，低30位为SpecSize，SpecMode是指测量模式，SpecSize是指在某种测量模式下的规格大小。
- MpecMode有三类；
	- 1.UNSPECIFIED 父容器不对View进行任何限制，要多大给多大，一般用于系统内部
	- 2.EXACTLY 父容器检测到View所需要的精确大小，这时候View的最终大小就是SpecSize所指定的值，对应LayoutParams中的match_parent和具体数值这两种模式。
	- 3.AT_MOST 父容器指定了一个可用大小即SpecSize，View的大小不能大于这个值，不同View实现不同，对应LayoutParams中的wrap_content。
- 当View采用固定宽/高的时候，不管父容器的MeasureSpec的是什么，View的MeasureSpec都是精确模式兵其大小遵循Layoutparams的大小。 当View的宽/高是match_parent时，如果他的父容器的模式是精确模式，那View也是精确模式并且大小是父容器的剩余空间；如果父容器是最大模式，那么View也是最大模式并且起大小不会超过父容器的剩余空间。 当View的宽/高是wrap_content时，不管父容器的模式是精确还是最大化，View的模式总是最大化并且不能超过父容器的剩余空间。
# view的工作流程
## measure
<pre>
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {          
     setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(),widthMeasureSpec),     
     getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
</pre>
- setMeasuredDimension方法会设置View的宽/高的测量值
- 0getDefaultSize方法返回的大小就是measureSpec中的specSize，也就是View测量后的大小，绝大部分情况和View的最终大小(layout阶段确定)相同。
- getSuggestedMinimumWidth方法，作为getDefaultSize的第一个参数（建议宽度)
直接继承View的自定义控件，需要重写onMeasure方法并且设置
- wrap_content时的自身大小，否则在布局中使用了wrap_content相当于使用了match_parent。解决方法：在onMeasure时，给View指定一个内部宽/高，并在wrap_content时设置即可，其他情况沿用系统的测量值即可。
## ViewGroup的measure过程
- 对于ViewGroup来说，除了完成自己的measure过程之外，还会遍历去调用所有子元素的measure方法，个个子元素再递归去执行这个过程，和View不同的是，ViewGroup是一个抽象类，没有重写View的onMeasure方法，提供了measureChildren方法。
- measureChildren方法，遍历获取子元素，子元素调用measureChild方法
- measureChild方法，取出子元素的LayoutParams，再通过getChildMeasureSpec方法来创建子元素的MeasureSpec，接着将MeasureSpec传递给View的measure方法进行测量。
- ViewGroup没有定义其测量的具体过程，因为不同的ViewGroup子类有不同的布局特征，所以其测量过程的onMeasure方法需要各个子类去具体实现。
- measure完成之后，通过getMeasureWidth/Height方法就可以获取View的测量宽/高，需要注意的是，在某些极端情况下，系统可能要多次 measure才能确定最终的测量宽/高，比较好的习惯是在onLayout方法中去获取测量宽/高或者最终宽/高。
- 在setMeasuredDimension()方法调用之后，我们才能使用getMeasuredWidth()和getMeasuredHeight()来获取视图测量出的宽高，以此之前调用这两个方法得到的值都会是0。
- getMeasuredWidth是measure阶段获得的View的原始宽度，getWidth是layout阶段完成后，其在父容器中所占的最终宽度
## Layout
- measure过程结束后，视图的大小就已经测量好了，接下来就是layout的过程了。正如其名字所描述的一样，这个方法是用于给视图进行布局的，也就是确定视图的位置。ViewRoot的performTraversals()方法会在measure结束后继续执行，并调用View的layout()方法来执行此过程，如下所示：
<pre>
mView.layout(0, 0, host.mMeasuredWidth, host.mMeasuredHeight);
</pre>
- layout()方法接收四个参数，分别代表着左、上、右、下的坐标，当然这个坐标是相对于当前视图的父视图而言的。可以看到，这里还把刚才测量出的宽度和高度传到了layout()方法中。那么我们来看下layout()方法中的代码是什么样的吧，如下所示：
<pre>

public void layout(int l, int t, int r, int b) {
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
    boolean changed = setFrame(l, t, r, b);
    if (changed || (mPrivateFlags & LAYOUT_REQUIRED) == LAYOUT_REQUIRED) {
        if (ViewDebug.TRACE_HIERARCHY) {
            ViewDebug.trace(this, ViewDebug.HierarchyTraceType.ON_LAYOUT);
        }
        onLayout(changed, l, t, r, b);
        mPrivateFlags &= ~LAYOUT_REQUIRED;
        if (mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>) mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }
    mPrivateFlags &= ~FORCE_LAYOUT;
}

</pre>
- 在layout()方法中，首先会调用setFrame()方法来判断视图的大小是否发生过变化，以确定有没有必要对当前的视图进行重绘，同时还会在这里把传递过来的四个参数分别赋值给mLeft、mTop、mRight和mBottom这几个变量。
- View中的onLayout()方法就是一个空方法，因为onLayout()过程是为了确定视图在布局中所在的位置，而这个操作应该是由布局来完成的，即父视图决定子视图的显示位置。既然如此，我们来看下ViewGroup中的onLayout()方法是怎么写的吧，代码如下：
<pre>
@Override
protected abstract void onLayout(boolean changed, int l, int t, int r, int b);
</pre>
- 可以看到，ViewGroup中的onLayout()方法竟然是一个抽象方法，这就意味着所有ViewGroup的子类都必须重写这个方法。没错，像LinearLayout、RelativeLayout等布局，都是重写了这个方法，然后在内部按照各自的规则对子视图进行布局的。
- onLayout中通常会遍历每个子view调用它们的layout方法来摆放它们
<pre>
	@Override
	protected void onLayout(boolean changed, int l, int t, int r, int b) {
		if (getChildCount() > 0) {
			View childView = getChildAt(0);
			childView.layout(0, 0, childView.getMeasuredWidth(), childView.getMeasuredHeight());
		}
	}

</pre>
# Draw
- measure和layout的过程都结束后，接下来就进入到draw的过程了。同样，根据名字你就能够判断出，在这里才真正地开始对视图进行绘制。ViewRoot中的代码会继续执行并创建出一个Canvas对象，然后调用View的draw()方法来执行具体的绘制工作。draw()方法内部的绘制过程总共可以分为六步，其中第二步和第五步在一般情况下很少用到，因此这里我们只分析简化后的绘制过程。代码如下所示：

<pre>
public void draw(Canvas canvas) {
	if (ViewDebug.TRACE_HIERARCHY) {
	    ViewDebug.trace(this, ViewDebug.HierarchyTraceType.DRAW);
	}
	final int privateFlags = mPrivateFlags;
	final boolean dirtyOpaque = (privateFlags & DIRTY_MASK) == DIRTY_OPAQUE &&
	        (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
	mPrivateFlags = (privateFlags & ~DIRTY_MASK) | DRAWN;
	// Step 1, draw the background, if needed
	int saveCount;
	if (!dirtyOpaque) {
	    final Drawable background = mBGDrawable;
	    if (background != null) {
	        final int scrollX = mScrollX;
	        final int scrollY = mScrollY;
	        if (mBackgroundSizeChanged) {
	            background.setBounds(0, 0,  mRight - mLeft, mBottom - mTop);
	            mBackgroundSizeChanged = false;
	        }
	        if ((scrollX | scrollY) == 0) {
	            background.draw(canvas);
	        } else {
	            canvas.translate(scrollX, scrollY);
	            background.draw(canvas);
	            canvas.translate(-scrollX, -scrollY);
	        }
	    }
	}
	final int viewFlags = mViewFlags;
	boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
	boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
	if (!verticalEdges && !horizontalEdges) {
	    // Step 3, draw the content
	    if (!dirtyOpaque) onDraw(canvas);
	    // Step 4, draw the children
	    dispatchDraw(canvas);
	    // Step 6, draw decorations (scrollbars)
	    onDrawScrollBars(canvas);
	    // we're done...
	    return;
	}
}

</pre>
- 可以看到，第一步是从第9行代码开始的，这一步的作用是对视图的背景进行绘制。这里会先得到一个mBGDrawable对象，然后根据layout过程确定的视图位置来设置背景的绘制区域，之后再调用Drawable的draw()方法来完成背景的绘制工作。那么这个mBGDrawable对象是从哪里来的呢？其实就是在XML中通过android:background属性设置的图片或颜色。当然你也可以在代码中通过setBackgroundColor()、setBackgroundResource()等方法进行赋值。

- 接下来的第三步是在第34行执行的，这一步的作用是对视图的内容进行绘制。可以看到，这里去调用了一下onDraw()方法，那么onDraw()方法里又写了什么代码呢？进去一看你会发现，原来又是个空方法啊。其实也可以理解，因为每个视图的内容部分肯定都是各不相同的，这部分的功能交给子类来去实现也是理所当然的。

- 第三步完成之后紧接着会执行第四步，这一步的作用是对当前视图的所有子视图进行绘制。但如果当前的视图没有子视图，那么也就不需要进行绘制了。因此你会发现View中的dispatchDraw()方法又是一个空方法，而ViewGroup的dispatchDraw()方法中就会有具体的绘制代码。

- 以上都执行完后就会进入到第六步，也是最后一步，这一步的作用是对视图的滚动条进行绘制。那么你可能会奇怪，当前的视图又不一定是ListView或者ScrollView，为什么要绘制滚动条呢？其实不管是Button也好，TextView也好，任何一个视图都是有滚动条的，只是一般情况下我们都没有让它显示出来而已。绘制滚动条的代码逻辑也比较复杂，这里就不再贴出来了，因为我们的重点是第三步过程。

- 通过以上流程分析，相信大家已经知道，View是不会帮我们绘制内容部分的，因此需要每个视图根据想要展示的内容来自行绘制。如果你去观察TextView、ImageView等类的源码，你会发现它们都有重写onDraw()这个方法，并且在里面执行了相当不少的绘制逻辑。绘制的方式主要是借助Canvas这个类，它会作为参数传入到onDraw()方法中，供给每个视图使用。Canvas这个类的用法非常丰富，基本可以把它当成一块画布，在上面绘制任意的东西，
- 将View绘制到屏幕上，大概的几个步骤：
	- .绘制背景background.draw(canvas)
	- .绘制自己（onDraw）
	- .绘制children(dispatchDraw)
	- .绘制装饰（onDrawScrollBars）
- View的绘制过程是通过dispatchDraw来实现的，它会遍历所有子元素的draw方法。
- 如果一个View不需要绘制任何内容，那么设置setWillNotDraw为true后，系统会进行相应的优化；ViewGroup默认为true，如果我们的自定义ViewGroup需要通过onDraw来绘制内容的时候，需要显示的关闭它。
