---
title: fragment嵌套viewpagr+fragment会出现白屏
date: 2019-09-12 14:34:03
tags:
---

## getChildFragmentManager()
有时候，我们会在 Fragment 里面继续嵌套二级甚至三级 Fragment，即 Activity 嵌套多级 Fragment。此时在 Fragment 里管理子 Fragment 时，也需要使用到 FragmentManager。但是一定要使用 v4.fragment.getChildFragmentManager() 方法获取 FragmentManager 对象！
#### 1.Fragment嵌套Fragment要用getChildFragmentManager。

本来里面的fragment用的还是getFragmentManager,Fragment嵌套Fragment时，里面要用getChildFragmentManager。

``` java 
FragmentManager childFragmentManager = getChildFragmentManager();
  ViewPager_Adapter viewPager_adapter = new ViewPager_Adapter(childFragmentManager, fragments);    //FragmentPagerAdapter
```
#### 2.常见错误  java.lang.IllegalStateException: Activity has been destroyed

onCreate中缺少super.onCreate(savedInstanceState) ，把它放在所有语句之前，应该解决这个问题。