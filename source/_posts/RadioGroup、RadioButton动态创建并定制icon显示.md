---
title: RadioGroup、RadioButton动态创建并定制icon显示
date: 2016-10-25 10:24
categories: Android
tags: Radiobutton
---
由于原生的Radiobutton不能满足业务需求，所以需要自己定制icon图片，和默认选中某一项。需要自己代码动态实现。废话不多说，上代码：
```
private void setViewData() {
        radioGroup.removeAllViews();

        int margin = ScreenUtil.dip2px(this, 14.0f);
        int marginLeft = ScreenUtil.dip2px(this, 20.0f);
        int paddingLeft = ScreenUtil.dip2px(this, 3.0f);

        RadioGroup.LayoutParams layoutParams = 
            new RadioGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT);
        RadioGroup.LayoutParams layoutParams1 = 
            new RadioGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, 1);
        layoutParams.setMargins(marginLeft, margin, margin, margin);
        layoutParams1.setMargins(margin, 0, margin, 0);

        int defaultId = -1;
        for (int i = 0; i < messageList.size(); i++) {
            RadioButton rb = new RadioButton(this);
            rb.setMaxLines(2);
            rb.setPadding(paddingLeft, 0, 0, 0);
            rb.setText(messageList.get(i).relayMessage);
            rb.setButtonDrawable(android.R.color.transparent);
            rb.setTextColor(Color.parseColor("#555555"));
            rb.setButtonDrawable(null);// 去掉左边默认图标
            rb.setCompoundDrawablePadding(margin);
            rb.setEllipsize(TextUtils.TruncateAt.END);// 结尾处打点显示
            Drawable drawable = getResources().getDrawable(R.drawable.relay_message_radio_selector);
            drawable.setBounds(0, 0, drawable.getMinimumWidth(), drawable.getMinimumHeight());
            rb.setCompoundDrawables(drawable, null, null, null);
            // 手动生成id
            int generateId = generateViewId();
            messages.put(generateId, messageList.get(i).relayMessage);
            rb.setId(generateId);
            if (i == 0) {
                defaultId = generateId;
            }

            TextView view = new TextView(this);
            view.setBackgroundColor(Color.parseColor("#dddddd"));

            radioGroup.addView(rb, layoutParams);
            radioGroup.addView(view, layoutParams1);
        }
        // 此处：默认选中第一个
        radioGroup.check(defaultId);
    }
```
R.drawable.relay_message_radio_selector 代码如下：

```
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <!--设置自己的图片icon-->
    <item android:drawable="@drawable/relay_radio_unchecked" 
        android:state_checked="false" />
    <item android:drawable="@drawable/relay_radio_checked" 
        android:state_checked="true" />
</selector>
```

效果如下图：

![](http://7xr1vo.com1.z0.glb.clouddn.com/%E9%80%89%E6%8B%A9.png)