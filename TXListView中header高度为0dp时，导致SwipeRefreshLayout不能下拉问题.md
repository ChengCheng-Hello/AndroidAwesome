## TXListView中header高度为0dp时，导致SwipeRefreshLayout不能下拉问题

#### TXListView内部是RecyclerView实现的，Header也是一个Item。

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.v4.widget.SwipeRefreshLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/swipeRefreshLayout"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <android.support.v7.widget.RecyclerView
        android:id="@+id/rv_list"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:overScrollMode="always"
        android:scrollbarDefaultDelayBeforeFade="500"
        android:scrollbarStyle="outsideOverlay"
        android:scrollbarThumbVertical="@drawable/tx_shape_vertical_scrollbar"
        android:scrollbars="vertical"/>
</android.support.v4.widget.SwipeRefreshLayout>
```

#### 滚动原理分析

- 当发生滚动的时候，先判断 `child` 是否可以滚动，如果不可滚动，才自己处理滚动。

  ```java
  @Override
  public void onNestedScroll(final View target, final int dxConsumed, final int dyConsumed, final int dxUnconsumed, final int dyUnconsumed) {
  	// Dispatch up to the nested parent first
  	dispatchNestedScroll(dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed, 	     		   mParentOffsetInWindow);

  	// This is a bit of a hack. Nested scrolling works from the bottom up, and as we are
  	// sometimes between two nested scrolling views, we need a way to be able to know when any
  	// nested scrolling parent has stopped handling events. We do that by using the
  	// 'offset in window 'functionality to see if we have been moved from the event.
  	// This is a decent indication of whether we should take over the event stream or not.
  	final int dy = dyUnconsumed + mParentOffsetInWindow[1];
      // 先判断 child 是否可以滚动，如果不可滚动，才自己处理滚动。
  	if (dy < 0 && !canChildScrollUp()) {
  		mTotalUnconsumed += Math.abs(dy);
  		moveSpinner(mTotalUnconsumed);
  	}
  }
  ```

- child 使用的是 `RecyclerView`，滚动的计算通过 `LinearLayoutManager` 的 `computeScrollOffset` 方法。

  ```java
  private int computeScrollOffset(RecyclerView.State state) {
  	if (getChildCount() == 0) {
  		return 0;
  	}
  	ensureLayoutState();
      // 由于header的高度为0，导致 findFirstVisibleChildClosestToStart 找到的第一个可见的 child 
      // 是第二个Item。
  	return ScrollbarHelper.computeScrollOffset(state, mOrientationHelper,   		findFirstVisibleChildClosestToStart(!mSmoothScrollbarEnabled, true), 			findFirstVisibleChildClosestToEnd(!mSmoothScrollbarEnabled, true), this, 			mSmoothScrollbarEnabled, mShouldReverseLayout);
  }
  ```

- 调用 `ScrollbarHelper` 的 `computeScrollOffset` 方法。

  ```java
  /**
   * A helper class to do scroll offset calculations.
   */
  class ScrollbarHelper {

      /**
       * @param startChild View closest to start of the list. (top or left)
       * @param endChild   View closest to end of the list (bottom or right)
       */
      static int computeScrollOffset(RecyclerView.State state, OrientationHelper orientation,
              View startChild, View endChild, RecyclerView.LayoutManager lm,
              boolean smoothScrollbarEnabled, boolean reverseLayout) {
          if (lm.getChildCount() == 0 || state.getItemCount() == 0 || startChild == null ||
                  endChild == null) {
              return 0;
          }
          // 由于高度为0，导致这里的 startChild 就是第二个，所以 minPosition 为 1。
          final int minPosition = Math.min(lm.getPosition(startChild),
                  lm.getPosition(endChild));
          final int maxPosition = Math.max(lm.getPosition(startChild),
                  lm.getPosition(endChild));
          // 所以这的 itemsBefore 为 1。
          final int itemsBefore = reverseLayout
                  ? Math.max(0, state.getItemCount() - maxPosition - 1)
                  : Math.max(0, minPosition);
          if (!smoothScrollbarEnabled) {
              return itemsBefore;
          }
          final int laidOutArea = Math.abs(orientation.getDecoratedEnd(endChild) -
                  orientation.getDecoratedStart(startChild));
          final int itemRange = Math.abs(lm.getPosition(startChild) -
                  lm.getPosition(endChild)) + 1;
          final float avgSizePerRow = (float) laidOutArea / itemRange;

          // 因为 itemsBefore 为 1， 导致计算的距离永远大于 0，相当于 RecyclerView 永远滚动不到顶部
          // 也就是说 child 还可以滚动，所以 SwipeRefreshLayout 就不能处理滚动事件。
          return Math.round(itemsBefore * avgSizePerRow + (orientation.getStartAfterPadding()
                  - orientation.getDecoratedStart(startChild)));
      }
  	// ........
  }
  ```

  ​

#### 结论

所以当 `Header` 高度为 `0dp` 时，相当于不可见，找到的第一个可见 `View` 实际上是第二个 `View`，导致计算结果永远大于 0，也就是说 `child` 永远存在滚动的空间，导致 `SwipeRefreshLayout` 不会处理滚动，也就显示不出来。