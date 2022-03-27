---
layout: post
title: "BottomSheet 设计分析与踩坑"
date: 2022-03-27
description: "BottomSheet 设计分析与踩坑"
tag: BottomSheet
---
### 1. BottomSheet

底部弹窗是一个很常见的一个功能，取消确认面板、分享面板、评论面板等，都是底部弹出的场景，那么想实现这样一个面板，应该怎么思考去设计一个面板满足需求呢？

![bottomsheet2](https://github.com/RalfNick/PicRepository/raw/master/bottomsheet/bottom_2.png)![bottomsheet3](https://github.com/RalfNick/PicRepository/raw/master/bottomsheet/bottom_3.png)![bottomsheet4](https://github.com/RalfNick/PicRepository/raw/master/bottomsheet/bottom_4.png)

对于开发来说，完成一个功能大致会完成这几步，首先是需求分析，然后方案调研与分析、方案设计、功能编码实现，当然最后一步复盘总结可有可无，但对于我们掌握知识来说还是很重要的！

### 2. 需求分析

![bottomsheet](https://github.com/RalfNick/PicRepository/raw/master/bottomsheet/bottomsheet.gif)

对于一个需求主要也是从几个方面出发：

>* UI：宽高、主题、背景、内部是否有多个布局或者 Fragment
>* 交互：拖动、动画、与外部联动、操作关联等
>* 业务场景：业务埋点、关联链路等
>* 功能扩展：应对可能的变化等

（1）首先 UI  部分：根据设计要求，需要看下面板的宽高，这里主要看宽高是否会有变化，如半屏面板会变成全屏面板，如果有高度变化在设计时就要考虑到。

（2）交互：交互这部分主要考虑面板的动态变化，如果把 UI 看成是静态的，那么交互可以看成是动态的，如打开和关闭动画、手势拖动、以及与外部界面的联动等

（3）业务场景：在进行设计时还要考虑到业务场景，如埋点，一些相关的链路记录等，由于不同的 app 采用不同的框架和一些基础类，这就有可能一些通用的实现可能并不满足业务需求，这就需要前期多加考虑，避免后期大改动

（4）上面几点主要针对当前需求的考虑，同时也需要针对面板后期的变化，给出一定考虑，尽可能在后期修改或者扩展时降低开发成本

### 3. 方案调研与方案设计

#### 3.1 方案调研

其实这一步主要就是考虑怎么实现功能，一般从功能实现角度来说，基本都会在满足需求的前提下，选择成本最小的方案。比如这样的面板考虑采用 BottomSheetDialogFragment，如果现有的组件能够满足需求，就会降低开发成本，也就是所谓的避免重复『造轮子』。

这里就以 BottomSheetDialogFragment 为例分析一下，使用它能否满足需求。

（1）BottomSheetDialogFragment 是一个 DialogFragment，它会借助一个 Dialog 的 Window 来展示，也就是它能设置主题、背景，对于宽高改变也能支持，BottomSheetDialogFragment 中使用 CoordinatorLayout 布局，通过 BottomSheetBehavior 控制收起和展开的高度。对于作为容器方面，由于 BottomSheetDialogFragment 也是一个 Fragment，内部添加多个子 Fragment 也支持。


（2）交互这块，如果有 BottomSheetBehavior 的展开和收起效果，那么 BottomSheetDialogFragment 可以很好的满足，如果没有这个效果，是一个固定高度，BottomSheetDialogFragment 也能设置，将它设置成跳过收起态，skipCollapsed = true，即每次都是展开状态。动画这块有几种方式，但是不同的方式并不一定满足，可能需要写个 Demo 看下效果。

>* BottomSheetBehavior 展开和关闭效果
>* 利用 Dialog 的动画效果
>* 自己操作 Fragment 的 View，利用属性动画

利用 BottomSheetBehavior 展开关闭效果也有版本限制，在最新版本上才有 setDismissWithAnimation 设置，如果 material design 的版本过低是没有这个设置的。

利用 Dialog 的动画效果也看下是否是设计想要的效果，Dialog 的动画效果就是视图动画，都是一些简单的动画。

如果面板外部和外面没有交互，点击外部就关闭面板，那没什么可说的。但是如果面板的状态和外部有联动，如上下拖动面板，外部的视图也跟着变化，就需要考虑 BottomSheetBehavior 给出的回调 BottomSheetCallback 能否满足

（3）业务场景，是要根据实际的业务情况来分析了，比如需要埋点上报，可能页面 Fragment 继承一些基类，或者实现特定接口。继承情况就没办法使用 BottomSheetDialogFragment，因为也需要继承 BottomSheetDialogFragment，实现特定接口没什么问题；还有，弹出面板的前一个页面是否需要暂停，是否需要被背景色覆盖

（4）扩展这块是根据当前业务来做出一定的预测，当然也不能过度设计，毕竟业务变化是很快的，很难设计一个完全通用的面板。可以考虑，面板内部是否会增加多个子 Fragment，是否会有 ViewPager 之类的

#### 3.2 方案设计

方案设计这块除了有公司自己的流程之外，开发者也可以给自己设置一点流程，比如，先快速跑个 Demo，写个文档，这个过程一方面是积累，另一方面更重要的是摸底，防止以为方案不合理给后面的开发过程造成过大压力。

通过上面的初步调研分析，就要根据情况来判断是否能够使用已有的组件，如 material design 版本，BottomSheetDialogFragment 动画效果，能否作为多个子 Fragment 的容器等等，调研过程中最好是跑 demo，实测一下。

比如使用 BottomSheetDialogFragment 跑个 Demo

```kotlin

class DialogBottomSheetFragment : BottomSheetDialogFragment(), IBottomSheetOperator {

    companion object {

        private const val TAG = "BottomSheetFragment"
        private const val RADIO_DEFAULT = 0.618f

        fun newInstance(args: Bundle?): DialogBottomSheetFragment {
            return DialogBottomSheetFragment().apply {
                this.arguments = args ?: Bundle()
            }
        }
    }

    private var mRadio = RADIO_DEFAULT
    private var mBottomSheet: FrameLayout? = null
    private var mBehavior: BottomSheetBehavior<FrameLayout>? = null
    var mStateChangedCallback: BottomSheetBehavior.BottomSheetCallback? = null

    override fun getTheme(): Int {
        return if (showsDialog) R.style.MyDialog_Transparent else 0
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return inflater.inflate(R.layout.dialog_bottom_sheet_fragment, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        initBottomSheetLayout()
        view.layoutParams?.height =
            (mRadio * ViewUtil.getScreenHeight(requireActivity())).toInt()
        view.addOnLayoutChangeListener { _, _, _, _, _, _, _, _, _ ->
            view.post {
                if (childFragmentManager.fragments.size > 0) {
                    fixBehaviorNestedScroll(view, getBehavior())
                }
            }
        }
        // 自己修改弹出动画
        BottomSheetAnimationUtil.overrideDialogEnterAnimFromBottom(view)
        getBehavior()?.apply {
            skipCollapsed = true
            state = BottomSheetBehavior.STATE_EXPANDED
            mBottomSheet?.let {
                mStateChangedCallback?.onStateChanged(
                    it,
                    BottomSheetBehavior.STATE_EXPANDED
                )
            }
        }
    }

    override fun onCreateDialog(savedInstanceState: Bundle?): Dialog {
        return super.onCreateDialog(savedInstanceState).apply {
            // 高版本 material design
            (this as? BottomSheetDialog)?.dismissWithAnimation = true
        }
    }

    override fun closePanel() {
        mBehavior?.state = BottomSheetBehavior.STATE_HIDDEN
    }

    private fun initBottomSheetLayout() {
        if (mBottomSheet == null) {
            mBottomSheet = view?.findViewById(R.id.design_bottom_sheet)
        }
        getBehavior()?.apply {
            mBehavior = (dialog as? BottomSheetDialog)?.behavior
            mBehavior?.addBottomSheetCallback(object : BottomSheetBehavior.BottomSheetCallback() {
                override fun onStateChanged(bottomSheet: View, newState: Int) {
                    if (newState == BottomSheetBehavior.STATE_HIDDEN) {
                        dismissInternal()
                    }
                }

                override fun onSlide(bottomSheet: View, slideOffset: Float) {
                    // LEFT-DO-NOTHING
                }

            })
            mStateChangedCallback?.let {
                mBehavior?.addBottomSheetCallback(it)
            }
        }
    }

    private fun getBehavior() = (dialog as? BottomSheetDialog)?.behavior

    private fun dismissInternal() {
        parentFragmentManager
            .beginTransaction()
            .remove(this)
            .commitAllowingStateLoss()
        if (showsDialog) {
            // 前一个页面需要有 onPause，需要新开启一个 Activity 承接 面板
            (activity as? BottomSheetActivity)?.finish()
        }
    }

}

```

对于这种情况使用 BottomSheetDialogFragment，对我们的开发成本最小。

但是如果不能继承 BottomSheetDialogFragment，只能使用工程中的一些 Fragment 基类，这时就要考虑怎么设计。BottomSheetDialogFragment 的 view 是借助于 BottomSheetDialog 来展示的，那就可以仿照 BottomSheetDialogFragment 新建一个类似的 Fragment，而 BottomSheetDialogFragment 代码本身就很少，更多的实现都是在 BottomSheetDialog 中。

此外，如果面板不需要有一个蒙层背景，也不需要一个新的 Window，它仅仅是前一个页面的一个子 View 或者子Fragment,而且还需要和它的父容器有很多交互，类似于抖音上的评论面板，拖动时也要改变视频区域的大小，这种情况下，可以仿照 BottomSheetDialog 的布局来实现一个 Fragment，核心目的是使用 CoordinatorLayout 布局和 BottomSheetBehavior。

```java
class BottomSheetFragment : Fragment(), IBottomSheetOperator {

    companion object {

        private const val TAG = "BottomSheetFragment"
        private const val RADIO_DEFAULT = 0.618f

        fun newInstance(args: Bundle?): BottomSheetFragment {
            return BottomSheetFragment().apply {
                this.arguments = args ?: Bundle()
            }
        }
    }

    private var mRadio = RADIO_DEFAULT
    private var mBottomSheet: FrameLayout? = null
    private var mBehavior: BottomSheetBehavior<FrameLayout>? = null
    var mStateChangedCallback: BottomSheetBehavior.BottomSheetCallback? = null

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return inflater.inflate(R.layout.bottom_sheet_fragment, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        initBottomSheetLayout()
        mBottomSheet?.layoutParams?.height =
            (mRadio * ViewUtil.getScreenHeight(requireActivity())).toInt()
        mBottomSheet?.requestLayout()
        BottomSheetAnimationUtil.overrideDialogEnterAnimFromBottom(view)
        mBehavior?.apply {
            skipCollapsed = true
            state = BottomSheetBehavior.STATE_EXPANDED
            mBottomSheet?.let {
                mStateChangedCallback?.onStateChanged(
                    it,
                    BottomSheetBehavior.STATE_EXPANDED
                )
            }
        }
    }

    override fun closePanel() {
        mBehavior?.state = BottomSheetBehavior.STATE_HIDDEN
    }

    @SuppressLint("ClickableViewAccessibility")
    private fun initBottomSheetLayout() {
        if (mBottomSheet == null) {
            mBottomSheet = view?.findViewById(R.id.design_bottom_sheet)
        }
        view?.findViewById<View>(R.id.touch_outside)?.apply {
            setOnClickListener {
                closePanel()
            }
        }
        mBottomSheet?.apply {
            setOnTouchListener { _: View?, _: MotionEvent? ->
                // Consume the event and prevent it from falling through
                true
            }
            mBehavior = BottomSheetBehavior.from(this)
            mBehavior?.isHideable = true
            mBehavior?.addBottomSheetCallback(object : BottomSheetBehavior.BottomSheetCallback() {
                override fun onStateChanged(bottomSheet: View, newState: Int) {
                    if (newState == BottomSheetBehavior.STATE_HIDDEN) {
                        dismissInternal()
                    }
                }

                override fun onSlide(bottomSheet: View, slideOffset: Float) {
                    // LEFT-DO-NOTHING
                }

            })
            mStateChangedCallback?.let {
                mBehavior?.addBottomSheetCallback(it)
            }
        }

    }

    private fun dismissInternal() {
        parentFragmentManager
            .beginTransaction()
            .remove(this)
            .commitAllowingStateLoss()
    }

}
```

到此，看似没啥大问题，基本满足需求，只不过一些细节需要根据需求不断完善。对于扩展部分，可以考虑下如果是作为多个子 Fragment 的情况是否能够满足，子 Fragment 中也有上下滑动手势，是否会和 CoordinatorLayout 有冲突，如果有 ViewPager 和 多个 tab 的水平滑动组件是否有冲突，这类问题也是需要考虑的。

假如作为多个子 Fragment 容器的情况，每个子 Fragment 中有 Recyclerview，可以实际验证一下。实际结果是对于内部有多个子 Fragment 时，每个Fragment 内部有 Recyclerview，仅仅在第一个 Fragment 容器可以拖动变化高度，其他子页面时 CoordinatorLayout 是不能拖动的，这就是实际遇到的坑点。这类问题需要解决，否则是很影响后面的开发的，如果作为 bug，测试阶段改 bug 很可能改动非常大。这个问题怎么解后面会回答一下。

再有就是现有  CoordinatorLayout 布局和 BottomSheetBehavior 不能满足和外部的交互联动，那么就需要自己去定义一个布局，实现期望的交互，这种情况成本就会高很多，本身自定义一个布局需要考虑的情况就很多，除了满足当前页面的要求，还要设计合理的接口和回调，满足外部的联动需求。

```kotlin
class BottomSheetNestedLayout @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : FrameLayout(context, attrs, defStyleAttr), NestedScrollingParent3 {

    companion object {
        const val TAG = "BottomSheetNestedLayout"
    }

    var mOnTopChanged: ((top: Int) -> Unit)? = null
    var mOnDragOutEvent: (() -> Unit)? = null
    private val mMaxDragSlop: Int = ViewUtil.dip2px(getContext(), 30f)
    private val mParentHelper: NestedScrollingParentHelper by lazy {
        NestedScrollingParentHelper(this)
    }
    private val mTouchSlop by lazy {
        ViewConfiguration.get(context).scaledTouchSlop
    }
    private var mInitPosition = 0f
    private var mLastY: Int = 0

    override fun onStartNestedScroll(child: View, target: View, axes: Int, type: Int): Boolean {
        return isEnabled
    }

    override fun onNestedScrollAccepted(child: View, target: View, axes: Int, type: Int) {
        mParentHelper.onNestedScrollAccepted(child, target, axes, type)
        onTopChanged()
    }

    override fun onNestedFling(
        target: View,
        velocityX: Float,
        velocityY: Float,
        consumed: Boolean
    ): Boolean {
        val nestedFling = !isUnderLollipop()
                && super.onNestedFling(target, velocityX, velocityY, consumed)
        onTopChanged()
        return nestedFling
    }

    override fun onNestedPreFling(target: View, velocityX: Float, velocityY: Float): Boolean {
        val nestedPreFling = !isUnderLollipop()
                && super.onNestedPreFling(target, velocityX, velocityY)
        onTopChanged()
        return nestedPreFling
    }

    override fun onStopNestedScroll(target: View, type: Int) {
        if (!isEnabled) {
            return
        }
        mParentHelper.onStopNestedScroll(target, type)
        onStopScroll()
    }

    private fun onStopScroll() {
        if (getOffsetFromInitPosition() > height / 2) {
            // 关闭
            val anim =
                ValueAnimator.ofFloat(getOffsetFromInitPosition().toFloat(), height.toFloat())
            anim.duration = 150
            anim.addUpdateListener { animation: ValueAnimator ->
                val top = (animation.animatedValue as Float).toInt()
                setOffsetFromInitPosition(top)
                onTopChanged(top)
            }
            anim.addListener(object : AnimatorListenerAdapter() {
                override fun onAnimationEnd(animation: Animator) {
                    super.onAnimationEnd(animation)
                    isEnabled = true
                    mOnDragOutEvent?.invoke()
                }

                override fun onAnimationStart(animation: Animator) {
                    super.onAnimationStart(animation)
                    isEnabled = false
                }
            })
            anim.start()
        } else if (getOffsetFromInitPosition() != 0 && getOffsetFromInitPosition() < height / 2) {
            // 回弹
            val anim = ValueAnimator.ofFloat(getOffsetFromInitPosition().toFloat(), 0f)
            anim.duration = 150
            anim.addUpdateListener { animation: ValueAnimator ->
                val top = (animation.animatedValue as Float).toInt()
                setOffsetFromInitPosition(top)
                onTopChanged(top)
            }
            anim.addListener(object : AnimatorListenerAdapter() {
                override fun onAnimationEnd(animation: Animator) {
                    super.onAnimationEnd(animation)
                    isEnabled = true
                }

                override fun onAnimationStart(animation: Animator) {
                    super.onAnimationStart(animation)
                    isEnabled = false
                }
            })
            anim.start()
        }
    }

    override fun onNestedScroll(
        target: View,
        dxConsumed: Int,
        dyConsumed: Int,
        dxUnconsumed: Int,
        dyUnconsumed: Int,
        type: Int,
        consumed: IntArray
    ) {
        this.onNestedScroll(target, dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed, type)
    }

    override fun onNestedScroll(
        target: View,
        dxConsumed: Int,
        dyConsumed: Int,
        dxUnconsumed: Int,
        dyUnconsumed: Int,
        type: Int
    ) {
        if (!isUnderLollipop()) {
            super.onNestedScroll(target, dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed)
        }
        onTopChanged()
    }

    override fun onNestedPreScroll(target: View, dx: Int, dy: Int, consumed: IntArray, type: Int) {
        if (!isEnabled) {
            return
        }
        if (!target.canScrollHorizontally(dx)) {
            consumed[0] += dx
        }
        if (dy == 0) {
            return
        }
        // 向下滑动
        if (dy < 0) {
            // 在顶部
            if (!target.canScrollVertically(-1)) {
                scrollByOffset(-dy)
                consumed[1] += dy
            }
        }
        // 向上滑动
        else {
            // 未达到顶部
            if (getOffsetFromInitPosition() - dy > 0) {
                scrollByOffset(-dy.toFloat())
                consumed[1] += dy
            }
            // 到达顶部
            else if (getOffsetFromInitPosition() != 0 && getOffsetFromInitPosition() - dy < 0) {
                val consumedY = dy - getOffsetFromInitPosition()
                scrollByOffset(-getOffsetFromInitPosition())
                consumed[1] += consumedY
            }
        }
        onTopChanged()
    }

    @SuppressLint("ClickableViewAccessibility")
    override fun onTouchEvent(event: MotionEvent): Boolean {
        if (!isEnabled) {
            return super.onTouchEvent(event)
        }
        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                mLastY= (event.y + 0.5f).toInt()
            }
            MotionEvent.ACTION_MOVE -> {
                val cy = (event.y + 0.5f).toInt()
                var dy: Int = mLastY - cy
                if (dy > 0) {
                    dy = Math.max(0, dy - mTouchSlop)
                } else {
                    dy = Math.min(0, dy + mTouchSlop)
                }
                Log.d(TAG, "onTouchEvent: dy "  + dy)
                handleSelfVerticalScroll(dy.toInt())
                onTopChanged()
                mLastY = cy
            }
            MotionEvent.ACTION_UP -> {
                onStopScroll()
            }
        }
        return super.onTouchEvent(event)
    }

    private fun handleSelfVerticalScroll(dy: Int) {
        if (dy == 0) {
            return
        }
        // 向下滑动
        if (dy < 0) {
            Log.d(TAG, "handleSelfVerticalScroll: 11 dy "  +dy)
            scrollByOffset(-dy)
        }
        // 向上滑动
        else {
            // 未达到顶部
            if (getOffsetFromInitPosition() - dy > 0) {
                scrollByOffset(-dy.toFloat())
            }
            // 到达顶部
            else if (getOffsetFromInitPosition() != 0 && getOffsetFromInitPosition() - dy < 0) {
                scrollByOffset(-getOffsetFromInitPosition())
            }
        }
    }

    private fun scrollByOffset(offset: Float) {
        translationY += offset
    }

    private fun isUnderLollipop(): Boolean {
        return Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP
    }

    fun getInitPosition(): Float {
        return mInitPosition
    }

    fun setInitPosition(initPosition: Float) {
        mInitPosition = initPosition
        translationY = mInitPosition
    }

    /**
     * 正数往下，负数往上
     */
    private fun getOffsetFromInitPosition(): Int {
        return (translationY - mInitPosition).toInt()
    }

    /**
     * 正数往下，负数往上
     */
    fun setOffsetFromInitPosition(f: Int) {
        translationY = f + mInitPosition
    }

    private fun scrollByOffset(offset: Int) {
        translationY += offset
    }

    private fun onTopChanged(top: Int = getOffsetFromInitPosition()) {
        mOnTopChanged?.invoke(top)
    }

    private fun onOnDragOut() {
        mOnDragOutEvent?.invoke()
    }
}
```

上面就是自己写一个布局的实例，显然成本会高很多，一般情况当然是能用现有的就用现有的，毕竟自己写一个的话后期还要去维护。

### 4 复盘总结

总结这块一方面是积累，另一方面是记录踩坑过程，避免以后重复踩坑。这里简单说下实现需求时遇到的一些问题：

1、是否是要新开启一个 Activity，以及 Activity 和 Dialog 的主题设置

不同的 app，不同的页面都有特定的主题样式，有沉浸式和非沉浸式的。对于 Activity 需要设置透明和沉浸式的适配，Dialog 需要设置背景的透明度，动画效果等

2、如果面板的高度有设置，触发重新 layout，那么会影响进入的动画效果，那么可能就需要自己处理一下动画

```kotlin
internal fun overrideDialogEnterAnimFromBottom(
        translateView: View,
        enterDuration: Long = 300L,
        listener: Animator.AnimatorListener? = null
    ) {
        translateView.viewTreeObserver.addOnPreDrawListener(object :
            ViewTreeObserver.OnPreDrawListener {
            override fun onPreDraw(): Boolean {
                translateView.viewTreeObserver?.let {
                    if (it.isAlive) {
                        it.removeOnPreDrawListener(this)
                    }
                }
                val translateTo = 0f
                val translateFrom = translateView.measuredHeight.toFloat()
                translateView.translationY = translateFrom
                ValueAnimator.ofFloat(translateFrom, translateTo).apply {
                    interpolator = PathInterpolatorCompat.create(0.645f, 0.045f, 0.355f, 1f)
                    duration = enterDuration
                    addUpdateListener { animation ->
                        val animatedValue = animation.animatedValue as Float
                        translateView.translationY = animatedValue
                    }
                    if (listener != null) {
                        addListener(listener)
                    }
                    start()
                }
                return false
            }
        })
    }
```

3、容器中有多个子 Fragment 时，其他子 Fragment 进入时也有动画时,有两种简单的方式：

(1)使用 setCustomAnimations 设置

```kotlin
childFragmentManager.beginTransaction().run {
    setCustomAnimations(enterAnim, exitAnim)
    add(R.id.design_bottom_sheet, fragment, fragmentTag)
    commitAllowingStateLoss()
}
```

(2)使用 setCustomAnimations 设置

```kotlin
// 容器中
beginTransaction().setTransition(FragmentTransaction.TRANSIT_FRAGMENT_OPEN)
    .add(R.id.design_bottom_sheet, fragment, tag)
    .commitAllowingStateLoss()


// 子 Fragment 中, 加 onAnimationEnd 监听是防止动画过程中加载数据卡顿
  @Nullable
  override fun onCreateAnimation(transit: Int, enter: Boolean, nextAnim: Int): Animation? {
    return when {
      transit == FragmentTransaction.TRANSIT_FRAGMENT_OPEN && enter -> {
        AnimationUtils.loadAnimation(context, R.anim.slide_in_from_right).apply {
          setAnimationListener(object : SimpleAnimationListener(){
            override fun onAnimationEnd(animation: Animation?) {
              super.onAnimationEnd(animation)
              mIsReadyLoad = true
              refresh()
            }
          })
        }
      }
      transit == FragmentTransaction.TRANSIT_FRAGMENT_CLOSE && !enter -> {
        AnimationUtils.loadAnimation(context, R.anim.slide_out_to_right)
      }
      else -> null
    }
  }
```
  android:clickable="true"
  android:focusable="true"

4、CoordinatorLayout 和子 Fragment 中 RecyclerView 滑动冲突

当容器中只有一个子 Fragment 时是没问题的，当多余一个 Fragment 时，后面的子 Fragment 中 RecyclerView 会和 CoordinatorLayout 出现滑动冲突，在 RecyclerView 在顶部向下滑动时，期望通过 CoordinatorLayout 能够拖动面板，但是拖不动，事件被 RecyclerView 拦截了。

(1)方式一

最直接的解决方式就是在顶部时，如果是向下滑动，则让 CoordinatorLayout 拦截事件。CoordinatorLayout 的事件处理是在 BottomSheetBehavior 处理的。而如果使用 BottomSheetDialogFragment 的话，是无法修改 BottomSheetBehavior 的，因为在 BottomSheetDialog 的布局中已经设置了 BottomSheetBehavior。方式一这里仅看下没有继承 BottomSheetDialogFragment 的情况。

自定义 CustomBottomSheetBehavior，继承 BottomSheetBehavior，重写 onInterceptTouchEvent，在里面判断是否向下滑动时，是否有子 View 在滑动，没有的话，则自己拦截，否则不拦截

还要设置是否需要开启拦截，因为第一个子 Fragment 不需要拦截处理

```kotlin
class CustomBottomSheetBehavior<V : View> : BottomSheetBehavior<V> {

    companion object {
        private const val TAG = "CustomBehavior"
    }

    interface ScrollDownInterceptor {
        fun canScrollDown(): Boolean
    }

    constructor() : super()
    constructor(context: Context, attrs: AttributeSet) : super(context, attrs)

    var mEnableIntercept = true
    private var mTouchSlop = 0
    private var mLastX = 0
    private var mLaseY = 0
    private val mChildLocationArray = IntArray(2)

    override fun onInterceptTouchEvent(
        parent: CoordinatorLayout,
        child: V,
        event: MotionEvent
    ): Boolean {
        if (event.actionMasked == MotionEvent.ACTION_DOWN) {
            mLastX = event.x.toInt()
            mLaseY = event.y.toInt()
        } else if (event.actionMasked == MotionEvent.ACTION_MOVE) {
            if (mTouchSlop == 0) {
                mTouchSlop = ViewConfiguration.get(parent.context).scaledTouchSlop
            }
            val dx = abs(mLastX - event.x)
            val dy = abs(mLaseY - event.y)
            if (mEnableIntercept
                && (mLaseY - event.y) < 0
                && dx < dy
                && dy > mTouchSlop
                && !hasScrollDownChild(parent, event)
            ) {
                return true
            }
        }
        return super.onInterceptTouchEvent(parent, child, event)
    }

    private fun hasScrollDownChild(view: View, event: MotionEvent): Boolean {
        view.getLocationOnScreen(mChildLocationArray)
        val rawX = event.rawX
        val rawY = event.rawY
        if (rawX < mChildLocationArray[0]
            || rawX > mChildLocationArray[0] + view.width
            || rawY < mChildLocationArray[1]
            || rawY > mChildLocationArray[1] + view.height
        ) {
            return false
        }

        if (view is ScrollDownInterceptor) {
            if (view.canScrollDown()) {
                return true
            }
        }
        if (view is ViewGroup) {
            (0 until view.childCount).forEach { index ->
                if (hasScrollDownChild(view.getChildAt(index), event)) {
                    return true
                }
            }
        }
        return false
    }
}

```

需要处理的子 Fragment 中的 RecyclerView 需要实现上面的接口，并在当前页面被选中时，开启允许拦截

```java
class BottomSheetRecyclerView : RecyclerView, CustomBottomSheetBehavior.ScrollDownInterceptor {

    constructor(context: Context) : super(context)
    constructor(context: Context, attrs: AttributeSet?) : super(context, attrs)
    constructor(context: Context, attrs: AttributeSet?, defStyleAttr: Int) : super(
        context,
        attrs,
        defStyleAttr
    )

    private val mHelper: RecyclerViewPositionHelper by lazy {
        RecyclerViewPositionHelper(this)
    }

    override fun canScrollDown(): Boolean {
        if (childCount <= 0) {
            return false
        }

        val firstChildPosition = mHelper.findFirstVisibleItemPosition()
        if (firstChildPosition > 0) {
            return true
        }
        val child = getChildAt(0)
        val insets = Rect(0, 0, 0, 0)
        getDecoratedBoundsWithMargins(child, insets)
        return insets.top >= 0
    }

}
```

这种方式有点麻烦，而且最终效果还有瑕疵的，向下滑动时，第一次仍旧可能不起作用。

(2)方式二

思考为什么只有第一个子 Fragment 是没问题的，BottomSheetBehavior 中又是如何处理的导致冲突的。

```java
  @Override
  public boolean onLayoutChild(){
      ...

    nestedScrollingChildRef = new WeakReference<>(findScrollingChild(child));
  }

  // 找到第一个可以滑动子 View
  @Nullable
  @VisibleForTesting
  View findScrollingChild(View view) {
    if (ViewCompat.isNestedScrollingEnabled(view)) {
      return view;
    }
    if (view instanceof ViewGroup) {
      ViewGroup group = (ViewGroup) view;
      for (int i = 0, count = group.getChildCount(); i < count; i++) {
        View scrollingChild = findScrollingChild(group.getChildAt(i));
        if (scrollingChild != null) {
          return scrollingChild;
        }
      }
    }
    return null;
  }

```

通过上述代码看到，BottomSheetBehavior 在发生布局时会找到第一个可滑动的子布局,而 CoordinatorLayout 的滑动是基于 NestedScroll 机制的,实现了 NestedScrollingParent3 和 NestedScrollingParent2 接口，也就是说子 RecyclerView 将滑动值传给 CoordinatorLayout，告诉 CoordinatorLayout 滑动，BottomSheetBehavior 只找到第一个可滑动的子布局，也就是第一个子 Fragment 的 RecyclerView，后面的 子 RecyclerView 是无法将滑动值传给 CoordinatorLayout，导致 CoordinatorLayout 无法拖动。

```java
  @Override
  public boolean onNestedPreFling(
      @NonNull CoordinatorLayout coordinatorLayout,
      @NonNull V child,
      @NonNull View target,
      float velocityX,
      float velocityY) {
    if (nestedScrollingChildRef != null) {
      return target == nestedScrollingChildRef.get()
          && (state != STATE_EXPANDED
              || super.onNestedPreFling(coordinatorLayout, child, target, velocityX, velocityY));
    } else {
      return false;
    }
  }
```

那么找到原因之后该怎么修改？也要分情况处理：

（1）对于直接使用 BottomSheetDialogFragment 的话，由于无法修改 BottomSheetBehavior，所以就无法通过继承修改，可以通过反射来修改。修改的时机就是在 layout 发生改变时，所以可以通过监听 OnLayoutChangeListener 来处理，但注意调用频繁度，毕竟是通过反射处理的，可能会影响效率。

```java

override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)
    ...
    view.addOnLayoutChangeListener { _, _, _, _, _, _, _, _, _ ->
        view.post {
            if (childFragmentManager.fragments.size > 0) {
                fixBehaviorNestedScroll(view, getBehavior())
            }
        }
    }
 }


 /** BottomSheetBehavior 在多个子 View 情况下滑动冲突处理 */
object BottomSheetBehaviorFixExt {

    @JvmStatic
    internal fun fixBehaviorNestedScroll(containerView: View, behavior: BottomSheetBehavior<*>?) {
        try {
            val field = BottomSheetBehavior::class.java.getDeclaredField("nestedScrollingChildRef")
            field.isAccessible = true
            behavior?.let {
                field.set(it, WeakReference<View>(findScrollingChild(containerView)))
            }
        } catch (e: NoSuchFieldException) {
            e.printStackTrace()
        } catch (e: IllegalAccessException) {
            e.printStackTrace()
        }
    }

    @JvmStatic
    private fun findScrollingChild(view: View): View? {
        if (ViewCompat.isNestedScrollingEnabled(view)) {
            return view
        }
        if (view is ViewGroup) {
            val count = view.childCount
            for (i in count - 1 downTo 0) {
                val scrollingChild = findScrollingChild(view.getChildAt(i))
                if (scrollingChild != null) {
                    return scrollingChild
                }
            }
        }
        return null
    }
}

```

（2）如果不使用 BottomSheetDialog，可以自己定义修改 BottomSheetBehavior

```java
class CustomBottomSheetBehaviorV2<V : View> : BottomSheetBehavior<V> {

    constructor() : super()
    constructor(context: Context, attrs: AttributeSet) : super(context, attrs)

    @Nullable
    override fun findScrollingChild(view: View): View? {
        if (ViewCompat.isNestedScrollingEnabled(view)) {
            return view
        }
        if (view is ViewGroup) {
            val count = view.childCount
            for (i in count - 1 downTo 0) {
                val scrollingChild = findScrollingChild(view.getChildAt(i))
                if (scrollingChild != null) {
                    return scrollingChild
                }
            }
        }
        return null
    }

    override fun shouldHide(child: View, yvel: Float): Boolean {
        return child.top.toFloat() + yvel * 0.2f >= fitToContentsOffset + child.height / 2.0f
    }
}
```

在布局上设置自定义的 BottomSheetBehavior

```
<FrameLayout
    android:id="@+id/design_bottom_sheet"
    style="?attr/bottomSheetStyle"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_gravity="center_horizontal|top"
    app:layout_behavior="@string/custom_bottom_sheet_behavior_v2" />
```

```
<resources>
    <string name="app_name">BottomSheet</string>
    <string name="custom_bottom_sheet_behavior" translatable="false">com.google.android.material.bottomsheet.CustomBottomSheetBehavior</string>
    <string name="custom_bottom_sheet_behavior_v2" translatable="false">com.google.android.material.bottomsheet.CustomBottomSheetBehaviorV2</string>
</resources>
```

5、多个子 Fragment 之间的事件透传问题

（1）可以是不可见的 Fragment hide 处理

（2）在当前的 Fragment 的布局上设置事件拦截，防止透传

```
  android:clickable="true"
  android:focusable="true"
```
