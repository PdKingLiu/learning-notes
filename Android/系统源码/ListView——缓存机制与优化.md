@[toc]
# ListView 缓存机制
## 单类型Item
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190919144924838.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
上图B代表View的类型，1/2/3代表在ListView中第几个位置。

ListView的缓存池是ListView的父类AbsListView的内部类RecycleBin里面的一个ArrayList的数组。

	ArrayList<View>[] scrapViews = new ArrayList[viewTypeCount];


看上图的List列表，当状态1变为状态2时，1B滑出6B滑入，此时scrapViews[0]中还没有缓存的View，所以6B创建，1B进入缓存。

状态2变为状态3时。缓存中存在B类型的元素，所以直接拿出来复用。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190919151119923.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
## 多类型Item
多类型机制和单类型区别不大，如下图示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190919152221898.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
状态1到状态2，1A进入scrapViews[0]的缓存。

状态2到状态3，1A复用，2B进入scrapViews[1]缓存。

# 缓存剖析

```java
 	/**
     * 根据数据列表的position返回需要展示的layout的对应的type
     * type的值必须从0开始 
     */
    @Override
    public int getItemViewType(int position) { 
    	return 0;
    }

    /**
     * 该方法返回多少个不同的布局
     */
    @Override
    public int getViewTypeCount() {
        return 1;
    }
 
```

这两个方法是BaseAdapter里面的，用于设置多Item的。

getItemViewType() 根据列表的position加载指定的Item。

getViewTypeCount() 返回Item的个数。这决定了scrapViews 数组的长度。

ListView缓存原理由它的父类AbsListView的内部类RecycleBin负责。

下面源码是对缓冲池的初始化。

```java
        public void setViewTypeCount(int viewTypeCount) {
            if (viewTypeCount < 1) {
                throw new IllegalArgumentException("Can't have a viewTypeCount < 1");
            }
            //noinspection unchecked
            ArrayList<View>[] scrapViews = new ArrayList[viewTypeCount];
            for (int i = 0; i < viewTypeCount; i++) {
                scrapViews[i] = new ArrayList<View>();
            }
            mViewTypeCount = viewTypeCount;
            mCurrentScrap = scrapViews[0];
            mScrapViews = scrapViews;
        }
```

可以看出，缓存大小取决于BaseAdapter getViewTypeCount()的返回值。

下面这个方法是每次初始化都会调用的。

```java
    private void setItemViewLayoutParams(View child, int position) {
        final ViewGroup.LayoutParams vlp = child.getLayoutParams();
        LayoutParams lp;
        if (vlp == null) {
            lp = (LayoutParams) generateDefaultLayoutParams();
        } else if (!checkLayoutParams(vlp)) {
            lp = (LayoutParams) generateLayoutParams(vlp);
        } else {
            lp = (LayoutParams) vlp;
        }

        if (mAdapterHasStableIds) {
            lp.itemId = mAdapter.getItemId(position);
        }
        lp.viewType = mAdapter.getItemViewType(position);
        lp.isEnabled = mAdapter.isEnabled(position);
        if (lp != vlp) {
          child.setLayoutParams(lp);
        }
    }
```

注意到

	lp.viewType = mAdapter.getItemViewType(position);

这句的意思是给view的参数设置上它的类型，但是这个类型是调用getItemViewType()得到的，所以我们需要重写getItemViewType()方法，让相同类型的View返回同一个值。

下面是AbsListView.obtainView()函数，该函数的作用是绘制每一行的View

```java
View obtainView(int position, boolean[] isScrap) {  
    isScrap[0] = false;  
    View scrapView; 
    scrapView = mRecycler.getScrapView(position);  
    // 根据position获取缓存的View
    View child;  
    if (scrapView != null) {  
    	// 如果不为null便可复用convertView
        child = mAdapter.getView(position, scrapView, this);  
        if (child != scrapView) {
        	// 和缓存中的不同的话，就重新存进去。
            mRecycler.addScrapView(scrapView);  
            if (mCacheColorHint != 0) {  
                child.setDrawingCacheBackgroundColor(mCacheColorHint);  
            }  
        } else {  
            isScrap[0] = true;  
            dispatchFinishTemporaryDetach(child);  
        }  
    } else {  
    	// 缓存为null的话，传递convertView为null
        child = mAdapter.getView(position, null, this);  
        if (mCacheColorHint != 0) {  
            child.setDrawingCacheBackgroundColor(mCacheColorHint);  
        }  
    }  
    return child;  
}  
```

关于如何获得缓存的View，代码里面的注释已经写的很清楚了。

> 参考
> [https://blog.csdn.net/geekerhw/article/details/52176914](https://blog.csdn.net/geekerhw/article/details/52176914)
> [https://hit-alibaba.github.io/interview/Android/basic/ListView-Optimize.html](https://hit-alibaba.github.io/interview/Android/basic/ListView-Optimize.html)
> [https://www.jianshu.com/p/2791bc155b65](https://www.jianshu.com/p/2791bc155b65)

# ListView优化
## 缓存优化
使用ListView自带的缓存优化。
上面也分析了，BaseAdapter自身的缓存原理。
ListView的Adapter的作用如下所示。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190919165350755.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvZGVGYXJtZXJfXw==,size_16,color_FFFFFF,t_70)
### 1. 复用convertView
一情况下getView()是是这样写的

```java
   @Override
   public View getView(int position, View convertView, ViewGroup parent) { 
       View view = View.inflate(MainActivity.this, R.layout.listview_item, null); 
       TextView tv_item = (TextView) view.findViewById(R.id.tv_item);
       tv_item.setText(list.get(position));
       return view;
   } 
```
在上面你的缓存原理可以知道，在得到Item时会首先在缓存中找。

	   child = mAdapter.getView(position, scrapView, this);  

当找到后会调用getView()并传入找到的缓存scrapView，此时我们应该在getView中对convertView进行复用。所以如果每次都进行`View.inflate(MainActivity.this, R.layout.listview_item, null); `缓存就失去了意义。

所以在getView中如果convertView不为null就要对其复用。更改为如下代码。

```java 
  public View getView(int position, View convertView, ViewGroup parent) {
      View view;
      // 判断convertView的状态，来达到复用效果
      if(null == convertView){
          //如果convertView为空，则表示第一次显示该条目，需要创建一个view
          view = View.inflate(Main1Activity.this,R.layout.listview_item,null);
      }else{
          //否则表示可以复用convertView
          view = convertView;
      }
      TextView textView =(TextView) view.findViewById(R.id.tv_item);
      textView.setText(list.get(position));
      return view;
  } 
```
### 2. 缓存Item条目
findViewById()这个方法是比较耗性能的操作，因为这个方法要找到指定的布局文件，进行不断地解析每个节点：从最顶端的节点进行一层一层的解析查询，找到后在一层一层的返回，如果在左边没找到，就会接着解析右边，并进行相应的查询，直到找到位置。因此可以对findViewById进行优化处理。 

解决：**定义控件引用的类ViewHolder**

优化后的代码：

```java
   public class MyAdapter extends BaseAdapter{ 
	······		
       @Override
       public View getView(int position, View convertView, ViewGroup parent) {
           ViewHolder  viewHolder;
           if(null == convertView){
               convertView = View.inflate(MainActivity.this,R.layout.listview_item,null);
               viewHolder = new ViewHolder();
               viewHolder.textView = (TextView) convertView.findViewById(R.id.tv_item);
               convertView.setTag(viewHolder);
           }else{
               viewHolder = (ViewHolder) convertView.getTag();
           }

           viewHolder.textView.setText(list.get(position));
           return convertView;
       }
   }
   static class ViewHolder{
       TextView textView;
   }
```

> 参考
> [https://blog.csdn.net/u013278940/article/details/52923807](https://blog.csdn.net/u013278940/article/details/52923807)

## 其他优化 
······