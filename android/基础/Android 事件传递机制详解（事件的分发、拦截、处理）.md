# 事件在Android中的传递顺序
- 事件在Android的传递顺序：
> Activity--> Window-->DecorView --> 布局View
# 事件的传递规则
- 一个点击事件，或者说触摸事件，被封装为了一个MotionEvent。事件的分发主要由三个重要的方法来完成：

	- 分发：dispatchTouchEvent;
	- 拦截：onInterceptTouchEvent；
	- 处理：onTouchEvent；
- 如果是ViewGroup容器类view，则以上三个方法都会用到。但是如果是View类型的，不能包含子view，那就没有第二个拦截方法，因为没有子view的话，拦截方法的就是多余的，只有ViewGroup才会有拦截。
##  public boolean dispatchTouchEvent(MotionEvent ev)
- 此方法用来处理事件的分发，当事件传递给当前view时，首先就是通过调用此方法来进行传递的，如果当前view锁包含的子view的dispatchTouchEvent方法或者当前view的onTouchEvent处理了事件， 通常返回true， 表示事件已消费。如果没有处理则返回false。
## public boolean onInterceptTouchEvent(MotionEvent ev)
- 此方法用来判断某个事件是否需要拦截，如果某个view拦截了此事件，那么同时个事件序列中，此方法不会被再次调用，因为会把当前view赋值给mFirstTouchTarget对象（原本为null），后续父必问判断mFirstTouchTarget != null时，就会去调用它的onTouchEvent方法，交给mFirstTouchTarget处理事件。
## public boolean onTouchEvent(MotionEvent ev)
- 用来处理事件，如果事件被消耗了，通常就返回true， 如果不做处理，则发挥false，并且在同一个时间序列中，当前view不会再接受到事件。

- 以上三个方法的关系可以用以下伪代码来表示：
<pre>
public boolean dispatchTouchEvent(MotionEvent ev) {
	boolean consume = false;
	if(onInterceptTouchEvent(ev)) {
		consume = onTouchEvent(ev);
	} else {
		consume = child.dispatchTouchEvent(ev);
	}
	
	return consume;
}

</pre>
![](../images/touchevent.png)
- 根据以上伪代码和图片展示的流程图，我们梳理一下从根ViewGroup（也就是DecorView）往下传递事件的过程：首先，事件产生后，通过调用根ViewGroup的dispatchTouchEvent方法，然后，如果这个ViewGroup要拦截事件， 则它的onInterceptTouchEvent返回true，然后事件交给它的onTouchEvent处理，不再进行传递。如果不拦截，则继续调用子view的dispatchTouchEvent方法，继续往下传递，往下递归，直到最终被处理。如果没有任何一个view处理事件， 则最终又会回调给Activity的onTouchEvent方法，如果Activity也不处理，则此事件结束，且没做任何处理。
# 源码分析
## Activity对事件的传递
- 前面已经讲到,APP层的事件传递是从Activity看是的，首先就是调用Activity的dispatchTouchEvent， 源码如下：
<pre>
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();//注释1
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;//注释2
    }
    return onTouchEvent(ev);//注释3
}

</pre>
- 注释1 是Activity的方法，如果当事件开始传递前，我们需要额外处理一些操作，可以咋onUserInteraction()中进行处理，这本身是一个空方法；

- 注释2 getWindow返回的就是DecorView对象，相当于最顶层的ViewGroup，然后就开始往下层的view传递， 如果事件在view的传递中被处理，则返回true，否则就调用注释3的代码；

- 注释3：如果事件在view的传递中未被处理，则调用Activity自己的onTouchEvent方法。
## View对事件的传递
- 在上面3.1小节讲到， 通过getWindow().superDispatchTouchEvent(ev)，把事件传递给了DecorView。其实在我们做APP开发时，有时就会用到(ViewGroup)getWindow().getDecorView().findViewById(android.R.id.content).getChildAt(0) 这种方式来获取我们给Activity的布局view。

- DecorView以前是PhoneWindow的一个内部类，不过现在已经独立成单独的一个类了，查看它的superDispatchTouchEvent方法代码
<pre>
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
</pre>


## 要点总结
- 对于一个根ViewGroup，点击事件产生后，首先会传递给它，这时他的dispatchTouchEvent会调用，
如果它的onInterceptTouchEvent返回true表示要拦截当前事件，接下来事件会交给这个ViewGroup处理，它的onTouchEvent就会被调用，如果这个ViewGroup的onInterceptTouchEvent返回false,则事件会继续传递给子元素，子元素的dispatchTouchEvent会调用，如此反复直到事件被处理。
- 当一个View需要处理事件时，如果设置了OnTouchListener,那么OnTouchListener的onTouch方法会回调，如果onTouch返回false,则当前View的onTouchEvent方法会被调用；如果返回true,那么onTouchEvent方法将不会调用。由此可见，OnTouchListener优先级高于onTouchEvent。OnClickListener优先级处在事件传递的尾端。
- 一个点击事件产生后，传递顺序：Activity->Window->View；如果一个View的onTouchEvent返回false,那么它的父容器的onTouchEvent会被调用，以此类推，所有元素都不处理该事件，最终将传递给Activity处理，即Activity的onTouchEvent会被调用。流程图如下

- 同一个事件序列是指从手指触摸屏幕那一刻开始，到手指离开屏幕那一刻（down->move...move->up)
- 一个事件序列只能被一个View拦截且消耗，同一个事件序列所有事件都会直接交给它处理，并且它的onInterceptTouchEvent不会再被调用。
- 某个View一旦开始处理事件，如果它不消耗ACTION_DOWN（onTouchEvent返回了false），那么同一事件序列中其他事件都不会再交给它来处理，事件将重新交给他的父元素处理，即父元素的onTouchEvent会被调用。
- 如果某个View不消耗除ACTION_DOWN以外的其他事件，那么这个点击事件会消失，此时父元素的onTouchEvent并不会被调用，并且当前View可以收到后续事件，最终这些消失的点击事件会传递给Activity处理
- ViewGroup默认不拦截任何事件，ViewGroup的onInterceptTouchEvent方法默认返回false。
- View没有onInterceptTouchEvent方法，一旦有事件传递给它，那么它的onTouchEvent方法就会被调用。
- View的onTouchEvent方法默认消耗事件（返回true）,除非他是不可点击的（clickable和longClickable同时为false）。View的longClickable属性默认都为false,clickable属性分情况，Button默认为true，TextView默认为false。
- onClick发生的前提是View可点击，并且它收到了down和up事件。
- 事件传递过程是由内而外，事件总是先传递给父元素，然后在由父元素分发给子View，通过requestDisallowInterceptTouchEvent方法可以在子元素干预父元素的事件分发过程，但ACTION_DOWN事件除外。


## ACTION_CANCEL 触发
- 我们知道如果某一个子View处理了Down事件，那么随之而来的Move和Up事件也会交给它处理。但是交给它处理之前，父View还是可以拦截事件的，如果拦截了事件，那么子View就会收到一个Cancel事件，并且不会收到后续的Move和Up事件。如果不拦截即使滑出子view范围，仍然可以收到Move和Up事件
- 
# 具体例子分析
