# Android自定义View须知

**本文参考自《Android开发艺术探索》**

自定义View的过程中有一些需要注意的地方，处理不好可能会导致设置的属性不生效等问题，影响View的显示和使用。

## 1.支持wrap_content

自定义View需要重写`onMeasure()`方法来支持`wrap_content`属性，否则View的显示效果会和`match_parent`一样，即占满父布局，原因可以查看View的`onMeasure()`方法。

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    // 获取宽-测量规则的模式和大小
    int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);
    // 获取高-测量规则的模式和大小
    int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
    int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);

    // 设置wrap_content的默认宽/高值，可以根据需要设置
    if (widthSpecMode == MeasureSpec.AT_MOST && heightSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(200, 200);
    } else if (widthSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(200, heightSpecSize);
    } else if (heightSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(widthSpecSize, 200);
    }
}
```

## 2.支持padding

如果是直接继承自View需要在`onDraw()`方法中处理`padding`的值，否则不会生效。如果是继承自ViewGroup需要在`onMeasure()`和`onLayout()`方法中考虑`padding`和子View的`margin`属性影响，否则`padding`和子View的`margin`不会生效。

```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    // 获取设置的padding值
    final int paddingLeft = getPaddingLeft();
    final int paddingRight = getPaddingRight();
    final int paddingTop = getPaddingTop();
    final int paddingBottom = getPaddingBottom();

    // 获取绘制内容实际的高度和宽度，考虑四个方向的padding值
    int width = getWidth() - paddingLeft - paddingRight;
    int height = getHeight() - paddingTop - paddingBottom;

    int radius = Math.min(width / 2, height / 2);
    canvas.drawCircle(paddingLeft + width / 2, paddingTop + height / 2, radius, mPaint);
}
```

## 3.避免使用Handler

View的内部本身提供了post系列的方法，完全可以替代Handler的作用，除非很明确地需要Handler来发送消息。

## 4. 防止内存泄漏

当View退出或不可见时需要及时停止线程和动画，否则可能会导致内存泄漏。可以在`onDetachedFromWindow()`方法中来处理，当包含此View的Activity退出或当前View被remove时会被调用。与此方法对应的是`onAttachedToWindow()` ，该方法会在包含此View的Activity启动时被调用。

## 5.处理好滑动冲突

当自定义View涉及到嵌套滑动时需要处理好滑动冲突。