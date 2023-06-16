
```java
public RecyclerView(@NonNull Context context, @Nullable AttributeSet attrs, int defStyle) {
    super(context, attrs, defStyle);
    if (attrs != null) {
        TypedArray a = context.obtainStyledAttributes(attrs, CLIP_TO_PADDING_ATTR, defStyle, 0);
        mClipToPadding = a.getBoolean(0, true);
        a.recycle();
    } else {
        mClipToPadding = true;
    }
    setScrollContainer(true);
    setFocusableInTouchMode(true);

    final ViewConfiguration vc = ViewConfiguration.get(context);
    mTouchSlop = vc.getScaledTouchSlop();
    mScaledHorizontalScrollFactor =
        ViewConfigurationCompat.getScaledHorizontalScrollFactor(vc, context);
    mScaledVerticalScrollFactor =
        ViewConfigurationCompat.getScaledVerticalScrollFactor(vc, context);
    mMinFlingVelocity = vc.getScaledMinimumFlingVelocity();
    mMaxFlingVelocity = vc.getScaledMaximumFlingVelocity();
    setWillNotDraw(getOverScrollMode() == View.OVER_SCROLL_NEVER);

    mItemAnimator.setListener(mItemAnimatorListener);
    initAdapterManager();
    initChildrenHelper();
    initAutofill();
    // If not explicitly specified this view is important for accessibility.
    if (ViewCompat.getImportantForAccessibility(this)
        == ViewCompat.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
        ViewCompat.setImportantForAccessibility(this,
                                                ViewCompat.IMPORTANT_FOR_ACCESSIBILITY_YES);
    }
    mAccessibilityManager = (AccessibilityManager) getContext()
        .getSystemService(Context.ACCESSIBILITY_SERVICE);
    setAccessibilityDelegateCompat(new RecyclerViewAccessibilityDelegate(this));
    // Create the layoutManager if specified.

    boolean nestedScrollingEnabled = true;

    if (attrs != null) {
        int defStyleRes = 0;
        TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.RecyclerView,
                                                      defStyle, defStyleRes);
        String layoutManagerName = a.getString(R.styleable.RecyclerView_layoutManager);
        int descendantFocusability = a.getInt(
            R.styleable.RecyclerView_android_descendantFocusability, -1);
        if (descendantFocusability == -1) {
            setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        }
        mEnableFastScroller = a.getBoolean(R.styleable.RecyclerView_fastScrollEnabled, false);
        if (mEnableFastScroller) {
            StateListDrawable verticalThumbDrawable = (StateListDrawable) a
                .getDrawable(R.styleable.RecyclerView_fastScrollVerticalThumbDrawable);
            Drawable verticalTrackDrawable = a
                .getDrawable(R.styleable.RecyclerView_fastScrollVerticalTrackDrawable);
            StateListDrawable horizontalThumbDrawable = (StateListDrawable) a
                .getDrawable(R.styleable.RecyclerView_fastScrollHorizontalThumbDrawable);
            Drawable horizontalTrackDrawable = a
                .getDrawable(R.styleable.RecyclerView_fastScrollHorizontalTrackDrawable);
            initFastScroller(verticalThumbDrawable, verticalTrackDrawable,
                             horizontalThumbDrawable, horizontalTrackDrawable);
        }
        a.recycle();
        createLayoutManager(context, layoutManagerName, attrs, defStyle, defStyleRes);

        if (Build.VERSION.SDK_INT >= 21) {
            a = context.obtainStyledAttributes(attrs, NESTED_SCROLLING_ATTRS,
                                               defStyle, defStyleRes);
            nestedScrollingEnabled = a.getBoolean(0, true);
            a.recycle();
        }
    } else {
        setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
    }

    // Re-set whether nested scrolling is enabled so that it is set on all API levels
    setNestedScrollingEnabled(nestedScrollingEnabled);
}
```

**第一段：**

```java
if (attrs != null) {
    TypedArray a = context.obtainStyledAttributes(attrs, CLIP_TO_PADDING_ATTR, defStyle, 0);
    mClipToPadding = a.getBoolean(0, true);
    a.recycle();
} else {
    mClipToPadding = true;
}
```

配置 ClipToPadding 属性，可从 xml 文件配置，默认 tue。

**第二段：**

```java
setScrollContainer(true);
setFocusableInTouchMode(true);
```

这两个都是 View 的属性配置。
第一个方法是**设置该组件是否作为可滚动容器使用，设为 true 后，展开软键盘 View 会被压缩。**
第二个方法是**设置该组件在触摸模式下是否可以得到焦点。**
**
**第三段：**

```java
final ViewConfiguration vc = ViewConfiguration.get(context);
mTouchSlop = vc.getScaledTouchSlop();
mScaledHorizontalScrollFactor =
    ViewConfigurationCompat.getScaledHorizontalScrollFactor(vc, context);
mScaledVerticalScrollFactor =
    ViewConfigurationCompat.getScaledVerticalScrollFactor(vc, context);
mMinFlingVelocity = vc.getScaledMinimumFlingVelocity();
mMaxFlingVelocity = vc.getScaledMaximumFlingVelocity();
setWillNotDraw(getOverScrollMode() == View.OVER_SCROLL_NEVER);

mItemAnimator.setListener(mItemAnimatorListener);
```

ViewConfiguration，熟悉自定义 View 的都会知道。

1. mTouchSlop 是一个距离，表示滑动的时候，手的移动要大于这个距离才开始移动控件。
2. mScaledHorizontalScrollFactor 响应水平{@link MotionEventCompat＃ACTION_SCROLL}事件而滚动的数量。 将此乘以事件的轴值即可获得要滚动的像素数。
3. mScaledVerticalScrollFactor 相对于 mScaledHorizontalScrollFactor。
4. mMinFlingVelocity 发起快速滑动的最小速度，以每秒像素数为单位。
5. mMaxFlingVelocity 发起快速滑动的最大速度，以每秒像素数为单位。
6. setWillNotDraw ViewGroup 是否执行 onDraw 方法，默认情况下，出于性能考虑，会被设置成WILL_NOT_DROW，这样，onDraw 就不会被执行了。如果我们想重写一个viewgroup的ondraw方法，有两种方法：1.构造函数中，给viewgroup设置一个颜色。2.构造函数中，调用setWillNotDraw（false），去掉其WILL_NOT_DRAW flag。这里的意思是 如果用户过度滚动此视图，就不执行 onDraw 方法。
7. 最后一行为 RecycleView.ItemAnimator 设置监听器，ItemAnimator 默认实现是 DefaultItemAnimator。

其他代码也是一些配置，其中有一个方法** createLayoutManager，它的作用是如果在 xml 里面设置了 LayoutManager ，就会通过反射去创建并设置它。**
**
**
