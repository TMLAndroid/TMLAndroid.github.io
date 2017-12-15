---
title: Android 基于TextView实现填空题
tags: 自定义View
description: 本文也可以说是TextView的扩展自定义样式，可以发挥想象实现你能想到的任何效果
categories: Android
---
# 前言
关于填空题，每人都有自己实现的特点，比如[这篇文章](http://www.jianshu.com/p/3c8a1c987cea)这种基于PopupWindow或者是Dialog中转来把文字填入，当然也有选词填入的，但个人觉得这都不是真正意义上的填空题。真正意义上的填空题应该可以能在文本下划线里输入文字，而且能满足比较好的扩展。这一点我觉得`猿题库`做的比较好，所以这篇文章可以说是模仿`猿题库`填空题的效果，但是也有不同之处，效果的实现思路很大程度上借鉴了[z56402344](https://github.com/z56402344)的[AnswerDemo](https://github.com/z56402344/AnswerDemo),在此对作者表示感谢。
# 高清无码图

![***的无码图.gif](http://upload-images.jianshu.io/upload_images/1263922-40366151da46aeab.gif?imageMogr2/auto-orient/strip)
怎么样，效果还是还可以吧，当然，你要选中加其它各种效果还是很容易扩展的，下面我将一步一步讲解实现的思路，心急的童鞋想直接看代码可以戳这里
 👉[特殊通道](https://github.com/TMLAndroid/FillBlankDemo)

# 臆想
如果你的设计师要求你实现这种效果，你大概想采用TextView +EditText的组合来实现，然后根据文本出现的下划线拆分，然后一个一个拼接，咦，怎么布局呢？突然流畅的思路被卡住了！又水平又垂直的。可能你会想到了流式布局，阴郁的天气瞬间晴朗了起来！BUT,光性能先不说，比如我以后要加一个试题的复制功能，就不好扩展了，代码整洁性以及扩展性是所不能够忍受的！

那怎么搞？
 
![怎么搞.jpg](http://upload-images.jianshu.io/upload_images/1263922-36e1499c25fc4070.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面将从两方面来讲
* 需要实现的效果
* 实现思路

> PS:需要具备Spanned系列Api使用的基础，具体学习可以网上搜索博客
![Spanned继承关系图](http://upload-images.jianshu.io/upload_images/1263922-3493430a610975ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 需要的效果

 1.  自动识别特定带填空题的标识，并能够自定义样式
 2.  填空题部分能够相应点击效果
3.   填空题部分能够输入文字
4.  输入完成之后可以把文本显示到下划线上面

需求写完一时爽，程序猿实现起来苦叫天。没办法，毕竟拿人钱财替人消灾，不行也得行，硬着头皮上！

# 实现思路
有点Android基础的都晓得TextView可以展示富文本信息，其实这些富文本信息的实现都是由Spanned系列拼接成的的SpannableStringBuilder,然后通过```TextView.setText(CharSequence text)```方法设置进去（说到富文本展示，还有一个方式是```Html.fromHtml(String source)```及其重载方法，翻看源码，本质上的实现还是和Spanned系列拼接而成的，我这里采用的就是这种方式），这里我把填空题设置为自定义的*<edit>*标签，然后在```Html. fromHtml(String source, int flags, Html.ImageGetter imageGetter, Html.TagHandler tagHandler)```重载方法的第三个参数```Html.TagHandler```来处理我的自定义标签*<edit>*,替换成我想要的内容，这里替换成我的自定义ReplacementSpan，这里可能很多人没有听说过这个类，本篇文章讲解实现思路，所以不打算深入讲解这个类的使用，这里简单介绍一下它的类的继承关系

![ReplacementSpan继承关系](http://upload-images.jianshu.io/upload_images/1263922-7d4b6765ac03421b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到ReplacementSpan类继承自MetricAffectingSpan，而MetricAffectingSpan又继承自CharacterStyle，而我们熟知的许多下划线UnderlineSpan 背景 BackgroundColorSpan 都是继承该类。

![CharacterStyle继承关系](http://upload-images.jianshu.io/upload_images/1263922-6dcb5023e16a6a8a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里我自定义ReplacementSpan，就可以实现下划线以及在下划线上显示用户输入文字的效果（这里在思路上已经实现了第1个和第4个效果）

TextView有个点击触摸事件方法```setMovementMethod(MovementMethod movement)```，这里我们可以自定义MovementMethod来实现特定字符的点击效果（这里我们就实现了第2个效果），在点击的时候计算好填空题的位置和长宽，然后把EditText放置上去就可以输入文字了（这里实现第3个效果）
大体上解决思路就讲完了，具体到代码可能需要很多细节。

咦？搞什么，没有代码？？没有贴代码吹啥牛逼！

 

![吹牛逼](http://upload-images.jianshu.io/upload_images/1263922-496830e3fada0b0b.gif?imageMogr2/auto-orient/strip)



  这里我放一下关键类自定义ReplacementSpan，更多的细节我的代码已经上传gitHub，前面已经给出链接。
 Show you the code
```java
/**
 * Created by tangminglong on 17/10/19.
 * 自定义Span，用来绘制填空题
 */

public class ReplaceSpan extends ReplacementSpan {

    private final Context context;
    public String mText = "";//保存的String

    private final Paint mPaint;

    public Object mObject;//回调中的任意对象
    private int textWidth = 80;//单词的宽度

    public OnClickListener mOnClick;
    public int id = 0;//回调中的对应Span的ID


    public ReplaceSpan(Context context,Paint paint) {
        this.context = context;
        mPaint = paint;
        textWidth = DensityUtils.dp2px(context,textWidth);

    }

    public void setDrawTextColor(int res){
        mPaint.setColor(context.getResources().getColor(res));
    }


    @Override
    public int getSize(@NonNull Paint paint, CharSequence text, int start, int end, Paint.FontMetricsInt fm) {
        //返回自定义Span宽度

        return textWidth;
    }

    @Override
    public void draw(@NonNull Canvas canvas, CharSequence text, int start, int end, float x, int top, int y, int bottom, @NonNull Paint paint) {

        float bottom1 =  paint.getFontMetrics().bottom;
        float y1 = y+bottom1;

        CharSequence ellipsize = TextUtils.ellipsize(mText, (TextPaint) paint, textWidth, TextUtils.TruncateAt.END);
        int width = (int)paint.measureText(ellipsize,0,ellipsize.length());

        width = (textWidth-width)/2;

        canvas.drawText(ellipsize,0,ellipsize.length(),  x+width, (float) y,mPaint);

        //需要填写的单词下方画线
        //这里bottom-1，是为解决有时候下划线超出canvas
        Paint linePaint = new Paint();

        linePaint.setColor(mPaint.getColor());
        linePaint.setStrokeWidth(2);
        canvas.drawLine(x, y1, x + textWidth, y1, linePaint);
    }

    public void onClick(TextView v, Spannable buffer, boolean isDown, int x, int y, int line, int off){
        if (mOnClick != null){
            mOnClick.OnClick(v,id,this);
        }
    }

    public  interface OnClickListener{
        void OnClick(TextView v, int id, ReplaceSpan span);
    }
    
}
```
# 最后
有时间的话我会整理出其他题型的适配，欢迎关注。
最近学习自定义View的进阶，扔物线的系列教程很不错，好东西和大家一起分享。
👉[特殊通道](http://hencoder.com/overview/)
 


