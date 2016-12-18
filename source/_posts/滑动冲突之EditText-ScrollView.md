---
title: 滑动冲突之EditText-ScrollView
date: 2016-08-16 17:14
categories: Android
tags: 滑动冲突
---
### 问题一
	EditText的内容过多时，EditText的内容并不能滚动，而是ScrollView的滚动
### 解决
		重写EditText的onTouch事件，将触摸事件交给EditText来处理

```
et.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
            	// 设置ScrollView不拦截事件
                scrollView.requestDisallowInterceptTouchEvent(true);
            	switch (event.getAction() & MotionEvent.ACTION_MASK){
                	case MotionEvent.ACTION_UP:
                	// 手指离开时：重置ScrollView事件拦截的状态
                    slContainer.requestDisallowInterceptTouchEvent(false);
                    	break;
            	}
            	return false;
            }
        });
    }
```
### 问题二
		若有ScrollView内容比较多，比较长时，编辑EditText里的内容时会出现ScrollView滑到底部的现象，使得当前编辑的EditText看不到了
### 解决
		修改AndroidManifest.xml中Activity的windowSoftInputMode属性
```
<activity
	android:name=".me.MineResumeProjectExperienceActivity"
	android:screenOrientation="portrait
	android:windowSoftInputMode="stateHidden|adjustPan"/>
```