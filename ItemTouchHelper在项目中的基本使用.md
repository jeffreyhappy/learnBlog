---
title: ItemTouchHelper在项目中的基本使用
---

看到汽车之家出了个新界面，我们需要抄一下。那就开始抄吧，界面如下：

![demo_car_home_20170726161156.gif](http://upload-images.jianshu.io/upload_images/2120696-e8b2564d1e56b27f.gif?imageMogr2/auto-orient/strip)
### 功能点
* 长按后可以拖拽调整位置，最后一个是添加按钮，不需要拖拽
* 滑动后，需要调整首个item的位置，不能出现首个item有部分不显示的情况

### 实现简述
其实系统已经将大部分工作做好了，就是基于RecyclerView，通过系统提供的ItemTouchHelper来实现.[官方文档点击查看](https://developer.android.google.cn/reference/android/support/v7/widget/helper/ItemTouchHelper.html)

最简单的来说，就像下面的代码一样，实例化后绑定。然后我们根据需求来调整回调
`````
     /**
     * 将ItemTouchHelper与RecyclerView绑定
     * @param recyclerView
     */
    private void bindTouchHelper(RecyclerView recyclerView){
        ItemTouchHelper itemTouchHelper = new ItemTouchHelper(new ItemTouchHelper.Callback() {

            /**
             * 这个是用来设置用户可以对 viewHolder进行什么操作，推荐用makeMovementFlags(int dragFlags, int swipeFlags)来处理
             * 例如 makeMovementFlags(UP | DOWN, LEFT);就是可以上下拖拽，向左滑动
             * @param recyclerView
             * @param viewHolder
             * @return
             */
            @Override
            public int getMovementFlags(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder) {
                return 0;
            }

            /**
             * 这个是拖拽的回调
             * @param recyclerView
             * @param viewHolder  这个是我们拖拽中的ViewHolder
             * @param target      这个是离我们拖拽ViewHolder最近的ViewHolder，也就是我们松手后需要替换的ViewHolder
             * @return  返回true的话 我们已经做好相关操作了，false就我们没做啥操作
             */
            @Override
            public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
                return false;
            }

            @Override
            public void onSwiped(RecyclerView.ViewHolder viewHolder, int direction) {
                //这个是滑动的回调，这次关注拖拽
            }
        });
        itemTouchHelper.attachToRecyclerView(recyclerView);
    }
`````
我们先来准备下RecyclerView，然后开始调整。RecyclerView的准备如下
`````
/**
    * 初始化RecyclerView，绑定LinearLayoutManager和Adapter
    * @return
    */
   private RecyclerView initRecyclerView(){
       //横向的LinearLayoutManager
       LinearLayoutManager linearLayoutManager = new LinearLayoutManager(this,LinearLayoutManager.HORIZONTAL,false);
       RecyclerView rv  = (RecyclerView) findViewById(R.id.rv);
       rv.setLayoutManager(linearLayoutManager);
       rv.setAdapter(new Adapter());

       //这个是为每个item之间增加一个20px的右边距
       rv.addItemDecoration(new RecyclerView.ItemDecoration() {
           @Override
           public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
               super.getItemOffsets(outRect, view, parent, state);
               outRect.right = 20;
           }
       });
       //在初始化的时候添加滑动监听，是为了实现RecyclerView自动回位的，实现在后面给出
       rv.addOnScrollListener(new ScrollListener());
       return rv;
   }



   /**
    * 测试Adapter，里面一个ImageView显示不同颜色的背景，一个TextView显示原始的position
    */
   public static class Adapter extends RecyclerView.Adapter<VH>{
       ArrayList<Bean> list = new ArrayList<>();

       public Adapter(){
           for( int i = 0 ; i < 20 ; i++){
               int color = R.color.colorPrimary;

               switch (i%3){
                   case 0:
                       color = R.color.colorPrimary;
                       break;
                   case 1:
                       color = R.color.colorAccent;
                       break;
                   case 2:
                       color = R.color.colorBlack;
                       break;
               }
               Bean bean = new Bean();
               bean.pos = i;
               bean.color = color;
               list.add(bean);
           }
       }

       @Override
       public VH onCreateViewHolder(ViewGroup parent, int viewType) {
           View itemView = LayoutInflater.from(parent.getContext()).inflate(R.layout.item,parent,false);
           return new VH(itemView);
       }

       @Override
       public void onBindViewHolder(VH holder, int position) {
           Bean  bean = list.get(position);
           holder.iv.setImageResource(bean.color);
           holder.tv.setText(String.valueOf(bean.pos));
       }


       public ArrayList<Bean> getList(){
           return list;
       }

       @Override
       public int getItemCount() {
           return list.size();
       }
   }

   public static class VH extends RecyclerView.ViewHolder{
       public        ImageView  iv;
       public        TextView   tv;
       public VH(View itemView) {
           super(itemView);
           iv = (ImageView) itemView.findViewById(R.id.iv);
           tv = (TextView) itemView.findViewById(R.id.tv);
       }
   }

   public static class Bean{
       public int pos;
       public int color;
   }

``````
item.xml
``````
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="170dp"
    android:clipChildren="false"
    android:scaleX="0.8"
    android:scaleY="0.8"
    android:layout_height="match_parent">
    <ImageView
        android:id="@+id/iv"
        android:src="@color/colorAccent"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
    <TextView
        android:id="@+id/tv"
        android:textColor="@android:color/white"
        android:textSize="16sp"
        android:layout_centerInParent="true"
        android:layout_gravity="center"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />
</RelativeLayout>
``````

好了，RecyclerView 准备好了，为了实现拖拽，我们现在处理下ItemTouchHelper的回调,我都注释了

`````
public static class DragCallBack extends ItemTouchHelper.Callback{
       RecyclerView rv;
       RecyclerView.ViewHolder lastDragViewHolder;
       public DragCallBack(RecyclerView rv){
           this.rv = rv;
       }

       /**
        * 当用户长按后，会触发拖拽的选中效果，viewHolder就是当前的选中
        * @param viewHolder
        * @param actionState 取下面中的值
        *                    {@link ItemTouchHelper#ACTION_STATE_IDLE},
        *                    {@link ItemTouchHelper#ACTION_STATE_SWIPE},
        *                    {@link ItemTouchHelper#ACTION_STATE_DRAG}.
        */
       @Override
       public void onSelectedChanged(RecyclerView.ViewHolder viewHolder, int actionState) {
           super.onSelectedChanged(viewHolder, actionState);
           //如果状态为拖拽，说明选中了
           //我在xml里面写的scale都为0.8 我们需要把当前的视图放大一下，所以设置为1就可以了
           if (viewHolder != null && actionState ==  ACTION_STATE_DRAG){
               lastDragViewHolder = viewHolder;
               viewHolder.itemView.setScaleX(1);
               viewHolder.itemView.setScaleY(1);
           }

           //ACTION_STATE_IDLE就是松开了，把大小改为原状
           if (lastDragViewHolder != null && actionState == ACTION_STATE_IDLE){
               lastDragViewHolder.itemView.setScaleX(0.8F);
               lastDragViewHolder.itemView.setScaleY(0.8F);
               lastDragViewHolder = null;
               //是为了实现RecyclerView自动回位的，实现在后面给出
               ensurePositionV1(rv);
           }
       }

       /**
        * 我们不需要滑动删除，所以返回false
        * @return
        */
       @Override
       public boolean isItemViewSwipeEnabled() {
           return false;
       }

       /**
        * 这个是用来设置用户可以对 viewHolder进行什么操作，推荐用makeMovementFlags(int dragFlags, int swipeFlags)来处理
        * 例如 makeMovementFlags(UP | DOWN, LEFT);就是可以上下拖拽，向左滑动
        * @param recyclerView
        * @param viewHolder
        * @return
        */
       @Override
       public int getMovementFlags(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder) {
           int position = viewHolder.getAdapterPosition();
           Adapter adapter = (Adapter) recyclerView.getAdapter();
           if (position == adapter.getItemCount() - 1){
               return 0;
           }
           return makeMovementFlags(ItemTouchHelper.LEFT|ItemTouchHelper.RIGHT,-1);
       }

       /**
        * 这个是拖拽的回调
        * @param recyclerView
        * @param viewHolder  这个是我们拖拽中的ViewHolder
        * @param target      这个是离我们拖拽ViewHolder最近的ViewHolder，也就是我们松手后需要替换的ViewHolder
        * @return  返回true的话 我们已经做好相关操作了，false就我们没做啥操作
        */
       @Override
       public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
           //把当前的和目标viewHoder提取出来换位置
           Adapter adapter = (Adapter) recyclerView.getAdapter();
           ArrayList<Bean> list = adapter.getList();
           int currentPos = viewHolder.getAdapterPosition();
           int targetPos  = target.getAdapterPosition();
           Bean currentBean = list.get(currentPos);
           Bean targetBean  = list.get(targetPos);
           list.set(currentPos,targetBean);
           list.set(targetPos,currentBean);
           //最后还要通知下adapter
           adapter.notifyItemMoved(currentPos,targetPos);
           return true;
       }

       @Override
       public void onSwiped(RecyclerView.ViewHolder viewHolder, int direction) {
           //这个是滑动的回调，这次关注拖拽
       }

       /**
        * 用来判断 target是否可以被替换
        * @param recyclerView
        * @param current
        * @param target
        * @return  true :target可以被current替换
        *          false：不可以
        */
       @Override
       public boolean canDropOver(RecyclerView recyclerView, RecyclerView.ViewHolder current, RecyclerView.ViewHolder target) {
           //汽车之家最后一个为加号，所以不支持拖拽
           if (target.getAdapterPosition() == recyclerView.getAdapter().getItemCount() -1){
               return false;
           }
           return super.canDropOver(recyclerView, current, target);
       }
   }
`````
现在拖拽替换就实现了 ，最后还要确保第一个item的边界可以自动回位。我们要在两个地方做处理，一个是用户正常滑动，一个是用户拖拽完成后


正常滑动处理: 添加RecyclerView的滑动监听，在上面的初始化RecyclerView里有个ScrollListener没给，ScrollListener只是做了在停下来的时候处理，
```````
    public static class ScrollListener extends RecyclerView.OnScrollListener{
        @Override
        public void onScrollStateChanged(final RecyclerView recyclerView, int newState) {
            super.onScrollStateChanged(recyclerView, newState);
            if (newState == RecyclerView.SCROLL_STATE_IDLE){
                ensurePositionV1(recyclerView);
            }
        }

        @Override
        public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
            super.onScrolled(recyclerView, dx, dy);
        }
    }


```````
拖拽完成处理：就是在ItemTouchHelper的onSelectedChanged回调里，松手后处理

其实就是等滑动/拖拽停下来后，获取第一个item的位置，然后确定是向左还是向右滚动一下，确保第一个可视view的left为0
```````
    /**
     * 确保第一个item自动回位，第一版
     * @param recyclerView
     */
    public static void ensurePositionV1(RecyclerView recyclerView){
        LinearLayoutManager layoutManager = (LinearLayoutManager) recyclerView.getLayoutManager();
        int firstVisible = layoutManager.findFirstVisibleItemPosition();
        View firstVisibleView = layoutManager.findViewByPosition(firstVisible);
        int left  = firstVisibleView.getLeft();
        int width = firstVisibleView.getWidth();
        //left就是这个item被RecyclerView裁剪掉了多少
        //如果第一个可视item的左边界在RecyclerView内左边线的右边则 left是正值，
        //如果在在RecyclerView内左边线的左边则为负值
        //如果滑动超过一定距离，就滚动到下个item去，我这里取一半多一点点，
        if (Math.abs(left) > width * 0.6){
            recyclerView.getScrollY();
            layoutManager.scrollToPositionWithOffset(firstVisible+1,0);
        }else {
            layoutManager.scrollToPositionWithOffset(firstVisible,0);
        }
    }
```````
layoutManager.scrollToPositionWithOffset(int position, int offset)滑动太快了，嗖的一下就回去，很不友好。我们需要来给滑动搞个动画
通过ValueAnimator结合RecyclerView.scrollBy(int x, int y)来实现


```````

  public static void ensurePosition(RecyclerView recyclerView){
        LinearLayoutManager layoutManager = (LinearLayoutManager) recyclerView.getLayoutManager();
        int firstVisible = layoutManager.findFirstVisibleItemPosition();
        View firstVisibleView = layoutManager.findViewByPosition(firstVisible);
        int left  = firstVisibleView.getLeft();
        int width = firstVisibleView.getWidth();
        ValueAnimator mAnimator ;
        //left就是这个item被RecyclerView裁剪掉了多少
        //如果第一个可视item的左边界在RecyclerView内左边线的右边则 left是正值，
        //如果在在RecyclerView内左边线的左边则为负值
        //如果滑动超过一定距离，就滚动到下个item去，我这里取一半多一点点，
        if (Math.abs(left) > width * 0.6){
            recyclerView.getScrollY();
            mAnimator = new ValueAnimator().ofInt(0, width - Math.abs(left));
        }else {
            mAnimator = new ValueAnimator().ofInt(0, left);
        }
        mAnimator.setDuration(500);
        mAnimator.addUpdateListener(new SimpleAnimatorListener(recyclerView));
        mAnimator.start();
    }


    public static class SimpleAnimatorListener implements ValueAnimator.AnimatorUpdateListener{
            private RecyclerView recyclerView;
            private int lastValue = 0;
            public SimpleAnimatorListener(RecyclerView view){
                this.recyclerView = view;
            }
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                int now = (int) animation.getAnimatedValue();
                int dx = now - lastValue;
                recyclerView.scrollBy(dx ,0);
                lastValue = now;
            }
    }
``````  

![demo_car_my_20170726173336.gif](https://upload-images.jianshu.io/upload_images/2120696-26977324d6132ad7.gif?imageMogr2/auto-orient/strip)



好了，好了 大功告成
