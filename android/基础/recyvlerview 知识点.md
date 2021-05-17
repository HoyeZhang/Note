# 和list区别
- layoutmanage 支持多种布局
- item 动画
-  强制实行viewholder
- 没有默认item点击事件
- 没有添加头部尾部

# viewholder解决什么
## converView绑定 settag 存入viewholder 
## viewHoler 跟复用无关，主要是解决findviewbyId （深度优先）
## 复用是converView

# 确定某种item出现在屏幕的次数（比如计算广告）
- 不能使用onCracreateViewHolder和onBindViewHolder，因为缓存复用
- 使用onViewAttachedToWindow()
# 缓存
## 有四重缓存
- mAttachedScrap 
- mCachedViews
- ViewCacheExtension
- RecycledViewPool 

# mAttachedScrap和mCachedViews
- 屏幕内，和屏幕外两个的缓存，根据position获取，找到直接返回使用
# ViewCacheExtension
- ViewCacheExtension用于开发者自定义表项缓存，且这层缓存的访问顺序位于mAttachedScrap和mCachedViews之后，RecycledViewPool之前。这和Recycler. tryGetViewHolderForPositionByDeadline()中的代码逻辑一致，那接下来的第五次尝试，应该是从 RecycledViewPool 中获取 ViewHolder
- 返回的是item不是viewholder
# RecycledViewPool
- RecycledViewPool中的ViewHolder存储在SparseArray中，并且按viewType分类存储（即是Adapter.getItemViewType()的返回值），同一类型的ViewHolder存放在ArrayList中，且默认最多存储5个。
- 需要重新走 onBindViewHolder方法

# 优化

- onBindViewHolder 里不做点击事件 缓存复用可能会走
- 共用RecyclerViewPool。如果多个 RecyclerView 的 Adapter 是一样的，比如嵌套的 RecyclerView 中存在一样的 Adapter，可以通过设置 RecyclerView.setRecycledViewPool(pool); 来共用一个 RecycledViewPool。

# diffutil

# 分割线