@[toc]
## RecyclerView绘制过程
RecyclerView绘制的源码还是比较复杂的，我这里就大概梳理一下主要的流程。

1. RecyclerView.requestLayout开始发生绘制。
2. Layout过程会通过LayoutManager.fill去将RecyclerView填满
3. LayoutManager.fill会调用LayoutManager.layoutChunk去生成一个的ViewHolder
4. 然后LayoutManager会调用Recycler.getViewForPosition向Recycler去要ViewHolder
5. Recycler缓存里面查找，如果找到就直接返回，流程如下。
	- 首先会去一级缓存里面查找，。
	- 如果一级缓存没有就会去二级缓存找，二级缓存是开发者自定义的，mViewCatchExtension。
	- 若二级没找到则去三级缓存找，如果找到就会调用Adapter.bindViewHolder来绑定内容，然后返回。如果没有找到就调用Adapter.createViewHolder创建一个ViewHolder，然后绑定内容。
7. 重复3-5，直到创建的ViewHolder填满整个 RecyclerView。、

实际的流程中一级缓存找完会去mViewCacheExtension中自定义获取缓存

上述过程可用下图表示
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190920195709201.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
## RecyclerView的缓存原理
RecyclerView的缓存实现在Recycler里面。

其中包含以下5个的对象：

```java
	// 未与RecyclerView分离的ViewHolder列表
	final ArrayList<RecyclerView.ViewHolder> mAttachedScrap = new ArrayList();
	// 数据已经改变的列表
	ArrayList<RecyclerView.ViewHolder> mChangedScrap = null;
	// 缓存ViewHolder
	final ArrayList<RecyclerView.ViewHolder> mCachedViews = new ArrayList();
	// ViewHolder缓存池
	RecyclerView.RecycledViewPool mRecyclerPool;
	// 用户自定义的一层缓存 空实现
	private RecyclerView.ViewCacheExtension mViewCacheExtension;
```
代码中已经注释了其对应的作用。更详细的内容请查看文章最后的[关于mAttachedScrap]()。

上面绘制流程第三点说过，会调用LayoutManager.layoutChunk()方法去生成一个ViewHolder。我们来具体看一下相应的源码。

```java
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
                     LayoutState layoutState, LayoutChunkResult result) {
        View view = layoutState.next(recycler);
}
View next(RecyclerView.Recycler recycler) { 
            if (mScrapList != null) {
                return nextViewFromScrapList();
            } 
            final View view = recycler.getViewForPosition(mCurrentPosition);
            mCurrentPosition += mItemDirection;
            return view;
        }
View getViewForPosition(int position, boolean dryRun) {
    return this.tryGetViewHolderForPositionByDeadline(position, dryRun, 9223372036854775807L).itemView;
}
```
实际就是调用了LayoutState.next()方法，next()再调用Recycler.getViewForPosition()方法，然后getViewForPosition()再调用tryGetViewHolderForPositionByDeadline()方法去获得ViewHolder。

下来看一下具体源码。

```java
/**
 * Attempts to get the ViewHolder for the given position, either from the Recycler scrap,
 * cache, the RecycledViewPool, or creating it directly.
 **/
 @Nullable
 ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
     if (position < 0 || position >= mState.getItemCount()) {
         throw new IndexOutOfBoundsException("Invalid item position " + position
                 + "(" + position + "). Item count:" + mState.getItemCount()
                 + exceptionLabel());
     }
     boolean fromScrapOrHiddenOrCache = false;
     ViewHolder holder = null;
     // 0) If there is a changed scrap, try to find from there
     if (mState.isPreLayout()) {
         //preLayout默认是false，只有有动画的时候才为true
         holder = getChangedScrapViewForPosition(position);
         fromScrapOrHiddenOrCache = holder != null;
     }
     // 1) Find by position from scrap/hidden list/cache
     if (holder == null) {
         holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
         if (holder != null) {
             if (!validateViewHolderForOffsetPosition(holder)) {
                 //如果检查发现这个holder不是当前position的
                 // recycle holder (and unscrap if relevant) since it can't be used
                 if (!dryRun) {
                     // we would like to recycle this but need to make sure it is not used by
                     // animation logic etc.
                     holder.addFlags(ViewHolder.FLAG_INVALID);
                     //从scrap中移除
                     if (holder.isScrap()) {
                         removeDetachedView(holder.itemView, false);
                         holder.unScrap();
                     } else if (holder.wasReturnedFromScrap()) {
                         holder.clearReturnedFromScrapFlag();
                     }
                     //放到ViewCache或者Pool中
                     recycleViewHolderInternal(holder);
                 }
                 //至空继续寻找
                 holder = null;
             } else {
                 fromScrapOrHiddenOrCache = true;
             }
         }
     }
     if (holder == null) {
         final int offsetPosition = mAdapterHelper.findPositionOffset(position);
         if (offsetPosition < 0 || offsetPosition >= mAdapter.getItemCount()) {
             throw new IndexOutOfBoundsException("Inconsistency detected. Invalid item "
                     + "position " + position + "(offset:" + offsetPosition + ")."
                     + "state:" + mState.getItemCount() + exceptionLabel());
         }

         final int type = mAdapter.getItemViewType(offsetPosition);
         // 2) Find from scrap/cache via stable ids, if exists
         if (mAdapter.hasStableIds()) {
             holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                     type, dryRun);
             if (holder != null) {
                 // update position
                 holder.mPosition = offsetPosition;
                 fromScrapOrHiddenOrCache = true;
             }
         }
         //自定义缓存
         if (holder == null && mViewCacheExtension != null) {
             // We are NOT sending the offsetPosition because LayoutManager does not
             // know it.
             final View view = mViewCacheExtension
                     .getViewForPositionAndType(this, position, type);
             if (view != null) {
                 holder = getChildViewHolder(view);
                 if (holder == null) {
                     throw new IllegalArgumentException("getViewForPositionAndType returned"
                             + " a view which does not have a ViewHolder"
                             + exceptionLabel());
                 } else if (holder.shouldIgnore()) {
                     throw new IllegalArgumentException("getViewForPositionAndType returned"
                             + " a view that is ignored. You must call stopIgnoring before"
                             + " returning this view." + exceptionLabel());
                 }
             }
         }
         //pool
         if (holder == null) { // fallback to pool
             if (DEBUG) {
                 Log.d(TAG, "tryGetViewHolderForPositionByDeadline("
                         + position + ") fetching from shared pool");
             }
             holder = getRecycledViewPool().getRecycledView(type);
             if (holder != null) {
                 holder.resetInternal();
                 if (FORCE_INVALIDATE_DISPLAY_LIST) {
                     invalidateDisplayListInt(holder);
                 }
             }
         }
         //create
         if (holder == null) {
             long start = getNanoTime();
             if (deadlineNs != FOREVER_NS
                     && !mRecyclerPool.willCreateInTime(type, start, deadlineNs)) {
                 // abort - we have a deadline we can't meet
                 return null;
             }
             holder = mAdapter.createViewHolder(RecyclerView.this, type);
             if (ALLOW_THREAD_GAP_WORK) {
                 // only bother finding nested RV if prefetching
                 RecyclerView innerView = findNestedRecyclerView(holder.itemView);
                 if (innerView != null) {
                     holder.mNestedRecyclerView = new WeakReference<>(innerView);
                 }
             }

             long end = getNanoTime();
             mRecyclerPool.factorInCreateTime(type, end - start);
         }
     }
     ....
     return holder;
 }
```
从代码的注释中可以清楚的看出其三级缓存和最后一次的create。

### 第一次获取（mAttachedScrap和mCacheView）

```java
if (holder == null) {
	holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
	 if (holder != null) {
	     if (!validateViewHolderForOffsetPosition(holder)) {
	         //如果检查发现这个holder不是当前position的
	 ...
	             //从scrap中移除
	             if (holder.isScrap()) {
	                 removeDetachedView(holder.itemView, false);
	                 holder.unScrap();
	             } else if (holder.wasReturnedFromScrap()) {
	                 ...
	             }
	             //放到ViewCache或者Pool中
	             recycleViewHolderInternal(holder);
	         }
	         //至空继续寻找
	         holder = null;
	     } else {
	         fromScrapOrHiddenOrCache = true;
	     }
	 }
}
```
从源码中可以看到，一级缓存首先会通过position得到ViewHolder，，并检验其有效性，如果无效就移除并加入MCacheViews或者Pool中，然后将其置位空进行下一级寻找。

然后进入getScrapOrHiddenOrCachedHolderForPosition()方法看一下具体内容。

```java
ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
        final int scrapCount = mAttachedScrap.size();

        // Try first for an exact, non-invalid match from scrap.
        //先从scrap中寻找
        int cacheSize;
        RecyclerView.ViewHolder vh;
        for(cacheSize = 0; cacheSize < scrapCount; ++cacheSize) {
            vh = (RecyclerView.ViewHolder)this.mAttachedScrap.get(cacheSize);
            if (!vh.wasReturnedFromScrap() && vh.getLayoutPosition() == position && !vh.isInvalid() && (RecyclerView.this.mState.mInPreLayout || !vh.isRemoved())) {
                vh.addFlags(32);
                return vh;
            }
        }
        //dryRun为false
        if (!dryRun) {
            //从HiddenView中获得，这里获得是View
            View view = mChildHelper.findHiddenNonRemovedView(position);
            if (view != null) {
                // This View is good to be used. We just need to unhide, detach and move to the
                // scrap list.
                //通过View的LayoutParam获得ViewHolder
                final ViewHolder vh = getChildViewHolderInt(view);
                //从HiddenView中移除
                mChildHelper.unhide(view);
                ....
                mChildHelper.detachViewFromParent(layoutIndex);
                //添加到Scrap中，其实这里既然已经拿到了ViewHolder，可以直接传vh进去
                scrapView(view);
                vh.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP
                        | ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
                return vh;
            }
        }

        // Search in our first-level recycled view cache.
        //从CacheView中拿
        cacheSize = mCachedViews.size();
        for (int i = 0; i < cacheSize; i++) {
            final ViewHolder holder = mCachedViews.get(i);
            // invalid view holders may be in cache if adapter has stable ids as they can be
            // retrieved via getScrapOrCachedViewForId
            //holder是有效的，并且position相同
            if (!holder.isInvalid() && holder.getLayoutPosition() == position) {
                if (!dryRun) {
                    mCachedViews.remove(i);
                }
                return holder;
            }
        }
        return null;
    }
```

可以清晰的看到寻找的次序，首先从mAttachedScrap中获取，然后从HiddenView中获取，接下来从CacheView中获取。
### 第二次获取（type）
然后看一下接下来的代码。
```java
    int offsetPosition;
    int type;
    if (holder == null) {
        offsetPosition = RecyclerView.this.mAdapterHelper.findPositionOffset(position);
        if (offsetPosition < 0 || offsetPosition >= RecyclerView.this.mAdapter.getItemCount()) {
            throw new IndexOutOfBoundsException("Inconsistency detected. Invalid item position " + position + "(offset:" + offsetPosition + ")." + "state:" + RecyclerView.this.mState.getItemCount() + RecyclerView.this.exceptionLabel());
        }

        type = RecyclerView.this.mAdapter.getItemViewType(offsetPosition);
        if (RecyclerView.this.mAdapter.hasStableIds()) {
            holder = this.getScrapOrCachedViewForId(RecyclerView.this.mAdapter.getItemId(offsetPosition), type, dryRun);
            if (holder != null) {
                holder.mPosition = offsetPosition;
                fromScrapOrHiddenOrCache = true;
            }
        }
```

	offsetPosition = RecyclerView.this.mAdapterHelper.findPositionOffset(position);

这里调用了我们平常使用的RecyclerView进行多样式Item的方法，也就是说，前面对于一级缓存，mAttachedScrap和mCacheViews是不区分type的，从现在开始寻找区分type的缓存。

然后再跟进一下代码
	
	 holder = this.getScrapOrCachedViewForId(RecyclerView.this.mAdapter.getItemId(offsetPosition), type, dryRun);
对应getScrapOrCachedViewForId方法

```java
ViewHolder getScrapOrCachedViewForId(long id, int type, boolean dryRun) {
        // Look in our attached views first
        //
        final int count = mAttachedScrap.size();
        for (int i = count - 1; i >= 0; i--) {
            //在attachedScrap中寻找
            final ViewHolder holder = mAttachedScrap.get(i);
            if (holder.getItemId() == id && !holder.wasReturnedFromScrap()) {
                //id相同并且不是从scrap中返回的
                if (type == holder.getItemViewType()) {
                    holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
                    if (holder.isRemoved()) {
                        // this might be valid in two cases:
                        // > item is removed but we are in pre-layout pass
                        // >> do nothing. return as is. make sure we don't rebind
                        // > item is removed then added to another position and we are in
                        // post layout.
                        // >> remove removed and invalid flags, add update flag to rebind
                        // because item was invisible to us and we don't know what happened in
                        // between.
                        if (!mState.isPreLayout()) {
                            holder.setFlags(ViewHolder.FLAG_UPDATE, ViewHolder.FLAG_UPDATE
                                    | ViewHolder.FLAG_INVALID | ViewHolder.FLAG_REMOVED);
                        }
                    }
                    return holder;
                } else if (!dryRun) {
                    // if we are running animations, it is actually better to keep it in scrap
                    // but this would force layout manager to lay it out which would be bad.
                    // Recycle this scrap. Type mismatch.
                    //从scrap中移除
                    mAttachedScrap.remove(i);
                    removeDetachedView(holder.itemView, false);
                    //加入cacheView或者pool
                    quickRecycleScrapView(holder.itemView);
                }
            }
        }
        //从cacheView中找
        // Search the first-level cache
        final int cacheSize = mCachedViews.size();
        for (int i = cacheSize - 1; i >= 0; i--) {
            final ViewHolder holder = mCachedViews.get(i);
            if (holder.getItemId() == id) {
                if (type == holder.getItemViewType()) {
                    if (!dryRun) {
                        //从cache中移除
                        mCachedViews.remove(i);
                    }
                    return holder;
                } else if (!dryRun) {
                    //从cacheView中移除，但是放到pool中
                    recycleCachedViewAt(i);
                    return null;
                }
            }
        }
        return null;
    }

```
可以看到这里的判断其实和上面那一次差不多，需要注意的是多了对于id的判断和对于type的判断，也就是当我们将hasStableIds()设为true后需要重写holder.getItemId() 方法，来为每一个item设置一个单独的id。

### 第三次获取（ViewCacheExtension）
这一层的缓存由开发者自定义，具体的源码实现如下。
```java
    public abstract static class ViewCacheExtension {
        public ViewCacheExtension() {
        }

        @Nullable
        public abstract View getViewForPositionAndType(@NonNull RecyclerView.Recycler var1, int var2, int var3);
    }
```
### 第四次获取（Pool）

```java
   if (holder == null) {
       holder = this.getRecycledViewPool().getRecycledView(type);
       if (holder != null) {
           holder.resetInternal();
           if (RecyclerView.FORCE_INVALIDATE_DISPLAY_LIST) {
               this.invalidateDisplayListInt(holder);
           }
       }
   }
```

```java
    public static class RecycledViewPool {
        private static final int DEFAULT_MAX_SCRAP = 5;
        SparseArray<RecyclerView.RecycledViewPool.ScrapData> mScrap = new SparseArray();
        private int mAttachCount = 0;

        static class ScrapData {
            final ArrayList<RecyclerView.ViewHolder> mScrapHeap = new ArrayList();
            int mMaxScrap = 5;
            long mCreateRunningAverageNs = 0L;
            long mBindRunningAverageNs = 0L;
        }

        @Nullable
        public RecyclerView.ViewHolder getRecycledView(int viewType) {
            RecyclerView.RecycledViewPool.ScrapData scrapData = (RecyclerView.RecycledViewPool.ScrapData)this.mScrap.get(viewType);
            if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
                ArrayList<RecyclerView.ViewHolder> scrapHeap = scrapData.mScrapHeap;
                return (RecyclerView.ViewHolder)scrapHeap.remove(scrapHeap.size() - 1);
            } else {
                return null;
            }
        }
    }
```
RecyclerdViewPool中使用了SparseArray。

SparseArray比HashMap更省内存，在某些条件下性能更好，主要是因为它避免了对key的自动装箱（int转为Integer类型），它内部则是通过两个数组来进行数据存储的，一个存储key，另外一个存储value，为了优化性能，它内部对数据还采取了压缩的方式来表示稀疏数组的数据，从而节约内存空间。

```java
    private int[] mKeys;
    private Object[] mValues;
```
SparseArray 对应不同type有不同的ScrapData ，ScrapData 有对应的ArrayList，默认最大值为5。通过mScrap.get(viewType)得到ScrapData 然后从中得到对应的缓存。

### 重建（createViewHolder）

```java
  if (holder == null) {
      long start = RecyclerView.this.getNanoTime();
      if (deadlineNs != 9223372036854775807L && !this.mRecyclerPool.willCreateInTime(type, start, deadlineNs)) {
          return null;
      }

      holder = RecyclerView.this.mAdapter.createViewHolder(RecyclerView.this, type);
      if (RecyclerView.ALLOW_THREAD_GAP_WORK) {
          RecyclerView innerView = RecyclerView.findNestedRecyclerView(holder.itemView);
          if (innerView != null) {
              holder.mNestedRecyclerView = new WeakReference(innerView);
          }
      }

      long end = RecyclerView.this.getNanoTime();
      this.mRecyclerPool.factorInCreateTime(type, end - start);
  }
}
```
若在所有的缓存中都没获取到，那么最终就会调用createViewHolder进行重新创建。

### 总结
1. RecyclerView缓存主要分为三级缓存
		mAttachedScrap、mCatchView，ViewCacheExtension，Pool
2. mAttached，mCacheViews在第一次获取时只是对View的复用，并不区分type。其余几次获取区分了type，是对于ViewHolder的复用。
3. 如果缓存ViewHolder时超过了mCachedView的限制，会将最老的一个ViewHolder移到RecycledViewPool中。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9waGFudG9tdmsuZ2l0aHViLmlvL2ltZy9hbmRyb2lkL1JlY3ljbGVyVmlldy9SZWN5Y2xlclZpZXdfY2FjaGVfbGV2ZWwucG5n?x-oss-process=image/format,png)

### 关于mAttachedScrap 
在这里，其实我还有个疑问，对于mAttachedScrap 的作用我并不是特别清晰，不理解为什么一级缓存mAttachedScrap 和 mCachedView 同时工作，他们两个之间又存在什么关系？

因此我再次阅读了RecyclerView回收的相关内容。找到了以下内容，这下总算是彻底弄明白了他的缓存回收流程。

- ArrayList<ViewHolder> mAttachedScrap 和 mChangedScrap

	还绑定在父 RecyclerView 上的 view 。 当视图滑出屏幕外了，需要在滑入屏幕的地方绑定 view 的时候，就会触发报废 view 的机制；调用数据更新接口也能触发 RecyclerView 重新排列，会对绑定的数据先报废分离，重新绑定。这个时候，也触发了报废 view 的机制，mChangedScrap 中的change指的是item被标记为更新、有动画且动画支持变化(即实现了animateChange方法)， 这种情况下， ViewHolders 被添加到 mChangedScrap 中， 其他时候报废的 ViewHolders 会添加到 mAttachedScrap。

- ArrayList<ViewHolder> mCachedViews

	缓存滑出屏幕的 ViewHolder。
	ViewHolder 被回收的时候，如果已经缓存的 ViewHolder 数量小于最大缓存值（默认为2，可以通过 setItemViewCacheSize() 来修改最大缓存数量），会优先放到 mCachedViews 中。

- RecycledViewPool mRecyclerPool

	RecyclerViewPool可以在不同的 RecyclerView 里面复用 ViewHolder；如果没有为 RecyclerView 设置 RecyclerViewPool(通过 setRecycledViewPool 方法)， 默认会自己设置一个。这里简单分析一些RecyclerViewPool的实现：
	
	RecyclerViewPool内部持有一个二维的mScrap变量， mScrap 的 key 为 ViewHolder 的 type， value 为一个列表， 表示的是同一 type 的 ViewHolder 实例集合。内部的其他方法为外部提供回收和复用的接口。

- ViewCacheExtension mViewCacheExtension

	ViewCacheExtension 是一个抽象类，只包含一个抽象方法，这个抽象方法, 该方法为开发者自定义复用机制提供接口，由开发者自行决定 ViewHolder 缓存的实现，并根据 viewtype 和 adapter position 返回 view 给 RecyclerView。


**这下总算是清楚了。**

网上对这五个容器的解释很模糊。所以初次看缓存机制的话会有一些疑问。

**mAttachedScrap，** 实际上可以理解为还绑定在当前RecyclerView上的ViewHolder，既然都绑定着那么这个还有什么用呢，我猜测一下，可能是由于一些原因导致RecyclerView 需要重新 onLayout，在layout的话，RecyclerView回把所有的children先remove掉，然后再重新add上去，那么暂时remove掉的ViewHolder就会存放在mAttachedScrap中，然后再重新取出add进去。

**mChangedScrap，** 存放的是被标记为更新的item。

**mCachedViews，** 缓存滑出屏幕的ViewHolder。

> 参考
> [https://www.jianshu.com/p/4a28b4ed616e](https://www.jianshu.com/p/4a28b4ed616e)
> [https://www.jianshu.com/p/6720973e0809](https://www.jianshu.com/p/6720973e0809)
> [https://www.jianshu.com/p/e44961f8add5](https://www.jianshu.com/p/e44961f8add5)
> [https://juejin.im/post/5b79a0b851882542b13d204b](https://juejin.im/post/5b79a0b851882542b13d204b)
> [https://www.jianshu.com/p/9306b365da57](https://www.jianshu.com/p/9306b365da57)
>
> 

