---
title: View 测量相关总结
date: 2017-08-11 13:18
categories: Android
tags: [View, Measure]
---

## ViewGroup的测量过程
**ViewGroup 没有重写 View 的 measure()  和 onMeasure()  方法，沿用的还是View的**

然后在 View 的源码中，我们找到 measure()，它内部调用到 onMeasure()。

```java
/**
 * This is called to find out how big a view should be. The parent
 * supplies constraint information in the width and height parameters.
 * The actual measurement work of a view is performed in
 * {@link #onMeasure(int, int)}, called by this method. Therefore, only
 * {@link #onMeasure(int, int)} can and must be overridden by subclasses.
 *
 * @param widthMeasureSpec Horizontal space requirements as imposed by the
 *        parent
 * @param heightMeasureSpec Vertical space requirements as imposed by the
 *        parent
 */
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {

    ......

    long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
    if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

    if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
            widthMeasureSpec != mOldWidthMeasureSpec ||
            heightMeasureSpec != mOldHeightMeasureSpec) {

        // first clears the measured dimension flag
        mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

        // 解析 layout、padding、drawable 相关参数
        resolveRtlPropertiesIfNeeded();

        int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                mMeasureCache.indexOfKey(key);
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } else {
            long value = mMeasureCache.valueAt(cacheIndex);
            // Casting a long to int drops the high 32 bits, no mask needed
            setMeasuredDimensionRaw((int) (value >> 32), (int) value);
            mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        // flag not set, setMeasuredDimension() was not invoked, we raise
        // an exception to warn the developer
        if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
            throw new IllegalStateException("View with id " + getId() + ": "
                    + getClass().getName() + "#onMeasure() did not set the"
                    + " measured dimension by calling"
                    + " setMeasuredDimension()");
        }

        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }

    mOldWidthMeasureSpec = widthMeasureSpec;
    mOldHeightMeasureSpec = heightMeasureSpec;

    mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
            (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
}
```
其中，方法内传入的2个参数：widthMeasureSpec、heightMeasureSpec 也原封不动的传给 onMeasure() 方法中。 那 widthMeasureSpec，heightMeasureSpec 又是什么东西呢，我们从注释中看到：是 父View提供给子View(当前的view)宽高的约束信息，即：*父View根据自己的宽高、内边距信息计算合成的值，提供给子View自己测量时候使用，作为一个建议值*，**不是父View的宽高信息**
从哪里能看出呢？我们找到ViewGroup的源码：
```java
protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
那么，widthMeasureSpec 和 heightMeasureSpec 具体又代表测量信息的什么呢？我们再看看 MeasureSpec(View 中的静态类)，发现，里面有 mode 和 size，mode 是测量模式（3中，不再细说，大家都知道），size 是测量大小，并且提供了 makeMeasureSpec() 和 getMode() ，getSize() 方法。 
**measureSpec** 这个东西是由 mode 和 size 合成的一个值，合成方法就是依靠 makeMeasureSpec() 来做位运算生成的。然后View和ViewGroup 会拿着它作为参数传递的。

	measureSpec 是携带了测量模式和测量大小信息的一个值

getChildMeasureSpec() 是个具有Android View 测量方法论的一个方法，必须要知道。
```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);// 父View的测量信息
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

然后这个方法内，最后一步，也是将 mode 和 size 合成 measureSpec，并返回给 childWidthMeasureSpec。
所以，我们在自定义View中可能会重写onMeasure方法时，要想知道 测量模式 mode 和大小，则必须通过getMode()、getSize()来获取。
我们再看看View的 onMeasure() ：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
        getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```
这里的 getDefaultSize() 方法，他也是这么做的:
```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;

    // 数值：AT_MOST(-2147483648)  EXACTLY(1073741824)  UNSPECIFIED(0)
}
```
## onMeasure() 被重写时，一定要调用 setMeasuredDimension() 方法
为什么一定要这么做呢，我们先来看看 这个方法干了什么？
```java
private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    // 赋值给测量宽高变量，并设置标志位的值
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;

    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}
```
再回到View的measure()方法，发现：如果mPrivateFlags没有被设置的话，会抛出异常。
那么，后面的mMeasureCache的put操作也就更新不了里面的值。这个缓存是个比较关键的东西，间接存储了测量的宽高信息。我们知道，一个View的测量操作是需要进行多次的(正常情况下3次measure，2次layout，1次draw)，所以这个存储就记录了当前测量结果的信息，下一次进行测量的时候，也是需要读取上次的信息，再判断如何进行下次测量操作的。
从measure()方法中能看到，需要读取到缓存中的值。在重新请求测量View时，也会清空缓存，到测量时自己再创建新的缓存对象。如：requestLayout()方法
这个LongSparseLongArray数据结构，使用形式上和map类似，但是它更省内存，也避免了装箱拆箱的操作，也不需要额外的entry。
Google 也提倡使用此类的 array 来代替 HashMap 在Android中的使用，其他类似的array大家也都知道

```java
/**
 * LongSparseLongArray:
 *
 * Map of {@code long} to {@code long}. Unlike a normal array of longs, there
 * can be gaps in the indices. It is intended to be more memory efficient than using a
 * {@code HashMap}, both because it avoids
 * auto-boxing keys and values and its data structure doesn't rely on an extra entry object
 * for each mapping.
 */
```
## onMeasure()被重写时，wrap_content 的功能需要自己实现
如果不单独对 AT_MOST 做处理，在使用的时候设置 wrap_content,效果也就是和 match_parent 等价的。
为什么呢? 还是分析 getChildMeasureSpec() 方法:
此时 childDimension 为 wrap_content，其测量结果 resultSize = size(specSize - padding, 父View的测量尺寸大小 或 父View剩余可用空间)
所以，我们可能按需求要对自定义的View，测量时，AT_MOST情况做处理了，我们可以对其设置一个默认的宽高值。这样写：
```java
private int defaultWidth, defaultHeight;

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int widthSpec = MeasureSpec.getMode(widthMeasureSpec);
    int heightSpec = MeasureSpec.getMode(heightMeasureSpec);
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSize = MeasureSpec.getSize(heightMeasureSpec);

    if (widthSpec == MeasureSpec.AT_MOST && heightSpec == MeasureSpec.AT_MOST) {
        setMeasuredDimension(defaultWidth, defaultHeight);
    } else if (widthSpec == MeasureSpec.AT_MOST) {
        setMeasuredDimension(defaultWidth, widthSize);
    } else if (heightSpec == MeasureSpec.AT_MOST) {
        setMeasuredDimension(widthSize, defaultHeight);
    }
}
```


## View 的测量方法是由它的父View发起的
View 自己并不能主动发起测量，都是由父View或外界调用requestLayout()来调View的测量。
ViewGroup会对自己所有的子View调起来测量的过程，父View对所有的子View 测量完毕后，再测量自己


## Android 测量起始由ViewRoot执行开始
从 ViewRootImpl 的performTraversals 开始，首先调用测量相关方法，ViewRootImpl的performMeasure方法，方法中会调用 ViewGroup measure方法，ViewGroup再对子View进行测量，如此递归下去，知道完成测量


附：
https://developer.android.com/reference/android/view/ViewGroup.html
http://blog.csdn.net/u012422440/article/details/52972825