### add,remove的生命周期
使用FragmentTransaction的add和remove的生命周期
```
getSupportFragmentManager().beginTransaction()
                    .add(R.id.fl_container,emptyFragment)
                    .commit();

getSupportFragmentManager().beginTransaction()
                            .remove(emptyFragment)
                            .commit();
```
感觉和activity很相似
```
01-05 12:50:13.694 25984-25984  D/EmptyFragment: onCreate
01-05 12:50:13.707 25984-25984  D/EmptyFragment: onCreateView
01-05 12:50:13.708 25984-25984  D/EmptyFragment: onStart
01-05 12:50:13.708 25984-25984  D/EmptyFragment: onResume
01-05 12:50:18.896 25984-25984  D/EmptyFragment: onPause
01-05 12:50:18.896 25984-25984  D/EmptyFragment: onStop
01-05 12:50:18.896 25984-25984  D/EmptyFragment: onDestroyView
01-05 12:50:18.899 25984-25984  D/EmptyFragment: onDestroy
```

### onHiddenChanged何时调用
使用FragmentTransaction的show和hide就是会触发 onHiddenChanged
FragmentTransaction.add
```
01-05 12:50:13.694 25984-25984  D/EmptyFragment: onCreate
01-05 12:50:13.707 25984-25984  D/EmptyFragment: onCreateView
01-05 12:50:13.708 25984-25984  D/EmptyFragment: onStart
01-05 12:50:13.708 25984-25984  D/EmptyFragment: onResume
```
这时候Fragment已经显示出来了
FragmentTransaction.hide只会调用onHiddenChanged
```
01-05 13:02:46.384 26851-26851  D/EmptyFragment: onHiddenChanged  true
```
再调用FragmentTransaction.show
```
01-05 13:02:46.981 26851-26851  D/EmptyFragment: onHiddenChanged 0 false
```

### setUserVisibleHint何时调用

> An app may set this to false to indicate that the fragment's UI is scrolled out of visibility or is otherwise not directly visible to the user

之前的add,remove,show,hide,都不会触发这个.而这个是很常用的,百度了下就是使用FragmentPagerAdapter时候调用
搞个例子试下
```
vp.setAdapter(new FragmentPagerAdapter(getSupportFragmentManager()) {
            @Override
            public Fragment getItem(int i) {
                if (i==  1){
                    return EmptyFragment2.getInstance(i+"");
                }
                if (i == 2){
                    return EmptyFragment3.getInstance(i+"");
                }
                return EmptyFragment.getInstance(i+"");
            }

            @Override
            public int getCount() {
                return 3;
            }
        });
```
这里的EmptyFragment都是空的,只是打印了下各自的生命周期函数而已.
```
01-05 13:22:35.488 28123-28123  D/EmptyFragment: setUserVisibleHint null false
01-05 13:22:35.488 28123-28123  D/EmptyFragment1: setUserVisibleHint null false
01-05 13:22:35.488 28123-28123  D/EmptyFragment: setUserVisibleHint null true
01-05 13:22:35.488 28123-28123  D/EmptyFragment: onCreate 0
01-05 13:22:35.488 28123-28123  D/EmptyFragment1: onCreate 1
01-05 13:22:35.492 28123-28123  D/EmptyFragment: onCreateView 0
01-05 13:22:35.493 28123-28123  D/EmptyFragment: onStart 0
01-05 13:22:35.493 28123-28123  D/EmptyFragment: onResume 0
01-05 13:22:35.495 28123-28123  D/EmptyFragment1: onCreateView 1
01-05 13:22:35.496 28123-28123  D/EmptyFragment1: onStart 1
01-05 13:22:35.496 28123-28123  D/EmptyFragment1: onResume 1
```
这里有个坑,setUserVisibleHint早于onCreate 调用,我们传参数一般在onCreate里操作,所以setUserVisibleHint 中会有个null
还有就是setUserVisibleHint 会调用两次,第一次设为false,第二次设为true
```
  public static EmptyFragment getInstance(String position){
        EmptyFragment emptyFragment = new EmptyFragment();
        Bundle bundle = new Bundle();
        bundle.putString("position",position);
        emptyFragment.setArguments(bundle);
        return emptyFragment;
    }


    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        position = getArguments().getString("position");
        Log.d(TAG,"onCreate " + position);
    }

```
ViewPager是默认初始化两个Fragment的,所以我搞了三个Fragment,再滑动一下看看,当前滑动到第二页
```
01-05 13:27:38.474 28123-28123  D/EmptyFragment2: setUserVisibleHint null false
01-05 13:27:38.474 28123-28123  D/EmptyFragment: setUserVisibleHint 0 false
01-05 13:27:38.474 28123-28123  D/EmptyFragment1: setUserVisibleHint 1 true
01-05 13:27:38.474 28123-28123  D/EmptyFragment2: onCreate 2
01-05 13:27:38.479 28123-28123  D/EmptyFragment2: onCreateView 2
01-05 13:27:38.480 28123-28123  D/EmptyFragment2: onStart 2
01-05 13:27:38.480 28123-28123  D/EmptyFragment2: onResume 2
```
EmptyFragment的 setUserVisibleHint设为false了,且参数也拿到了, EmptyFragment1的setUserVisibleHint 设为true了,EmptyFragment2开始初始化了
结论:
setUserVisibleHint 是使用在Viewpager中的
1. setUserVisibleHint是早于onCreate调用的
1. Fragment初始化的时候先默认设为false,然后如果是需要显示再设为true
1. 综合1和2,setUserVisibleHint中不要做跟数据相关的操作,只设置下显示标志位就可以了

### 最后
Fragment的延迟加载貌似只能用在ViewPager上
1. 通过setUserVisibleHint 来判断当前是否是显示Fragment,一个标志位
1. 通过onResume中判断Fragment是否已经初始化完成,第二个标志位
1. 这两个标志位都为true的时候,加载数据,再设置一个标志位,数据加载完成的标志位.