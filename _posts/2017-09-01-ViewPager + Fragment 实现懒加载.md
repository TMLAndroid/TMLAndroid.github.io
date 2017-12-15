---
title: ViewPager + Fragment 实现懒加载
tags: 
- ViewPager
- Fragment
description: 懒加载
categories: Android
---
# 废话
`懒加载`已可以称延迟加载。在ViewPager + Fragment 实现左右切换的页面默认会默认加载当前Fragment相邻的Fragment数据，为了更好的用户体验，最好是切换到当前Fragment再加载数据，避免了加载对性能的损耗同时也节省了用户的流量。


我们发现ViewPager的setOffsetPageLimit(n)可以设置缓存页面，当前页面相邻的n个页面都会被缓存。话讲到这个份上，蓦然发现，“握草！”，不是直接把n设置为0不就行了麽，缓存页面为0，so easy！好了，解析完毕，下班回家收衣服！

当然肯定没有那么简单，不然就没必要在这里BB了

# No BB , Show me the Code

```java
public abstract class BaseFragment extends Fragment{

    private BaseActivity mActivity;
    private View mRootView; //缓存Fragment view
    private boolean mIsLoad = false;//是否加载数据

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        this.mActivity = (BaseActivity) getActivity();
    }

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        if (mRootView == null) {
            mRootView = inflater.inflate(attachLayoutRes(), null);
         mRootView.findViewById(R.id.empty_layout);
            initViews(mRootView, savedInstanceState);
        }


        ViewGroup parent = (ViewGroup) mRootView.getParent();
        if (parent != null){
            parent.removeView(mRootView);
        }

        return mRootView;
    }

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        if (getUserVisibleHint()  && mRootView != null && !mIsMulti) {
            mIsLoad = true;
            updateViews(true);
        }
    }

    @Override
    public void setUserVisibleHint(boolean isVisibleToUser) {

        if (isVisibleToUser && isVisible() && mRootView != null && !mIsLoad){
            mIsLoad = true;
            updateViews(true);
        }else {
            super.setUserVisibleHint(isVisibleToUser);
        }
    }

    /**
     * 更新布局 在这里加载数据
     * @param isShowLoading 是否显示加载中的布局
     */
    protected abstract void updateViews(boolean isShowLoading);

    /**
     * 加载布局
     * @return
     */
    public abstract int attachLayoutRes();

    /**
     * 初始化布局
     * @param view
     * @param savedInstanceState
     */
    public abstract void initViews(View view, Bundle savedInstanceState);
   
```
**想直接使用的朋友可以让Fragment继承该类就行了**

这里我就不通过打印的Log来说明，直接通过问答的方式简单介绍一下

* setUserVisibleHint方法调用时机
 >当Fragment可见状态发生变化的时候调用，在这里可以进行相应可见时进行相应的逻辑处理

* 为什么在setUserVisibleHint加个mRootView判空处理
>Fragment调用顺序 setUserVisibleHint——>onCreateView——>onActivityCreated(注意setUserVisibleHint调用是在onCreateView之后)
* 为什么要在setUserVisibleHint方法和onActivityCreated两处都调用获取数据进行界面更新的方法updateViews呢？
 >这里我直接给出表象，自己思考一下就知道为什么了：第一次进入界面，第一个Fragment获取数据是在onActivityCreated里面，剩余的Fragment获取数据是在setUserVisibleHint里面。

# END
就结束了？怎么会呢，文章还没有图呢，以一张帅气的哈士奇结束，祝大家周末愉快

![王之藐视](http://upload-images.jianshu.io/upload_images/1263922-0b97ca73d2013519.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 


