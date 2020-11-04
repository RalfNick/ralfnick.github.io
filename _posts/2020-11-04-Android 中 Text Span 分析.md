---
layout: post
title: "Android 中 Text Span 分析"
date: 2020-11-04
description: "Android 中 Text Span 分析"
tag: Text
---
### 1.为什么需要 Span？

在文本展示时，如果不需要设置样式，包括颜色，大小，对齐方式等属性时，可以利用 View 的属性来控制，但是很多时候我们希望控制颜色，字体大小，对齐方式，段落，超链接点击，甚至是可编辑等特性，这时候就需要能够对文本的一个或者多个字符进行属性控制，所以就有了 span。 span 译为“跨度”，可以理解为就是一个或者多个字符的意思。一个文本可以有多个 span，这些 span 就是用来设置文本的样式，每个 span 标识文本的一个字符或者段落级。这些 span 是附属在文本上的，通过它们来改变文本的一些属性，添加文本局部的文本颜色，使文本可点击，缩放文本字体大小，并且支持自定义绘制文本。同时也能够改变 TextPaint 属性，以及能够改变文本的 layout。

### 2. Span

#### 2.1 SpannedString、SpannableString、SpannableStringBuilder

设置 span，可以通过 3 个实现 spaned 接口的类，SpannedString、SpannableString、SpannableStringBuilder，下面是这单个类的区别：

![span_string](https://github.com/RalfNick/PicRepository/raw/master/span/span_string.png)

一般情况下，我们采用 SpannableStringBuilder，实际上也可以采用另外两个类，那么如果选择呢？

>* SpannedString：创建后不再改变文本和 span
>
>* SpannableString：创建后文本仅仅可读，不能改变，span 可以增删，但 span 数量较少 
>
>* SpannableStringBuilder：创建后文本和 span 数量
>
>* SpannableStringBuilder：文本的 span 数量比较多

所谓文本不可改变，是指能否对原始文本进行拼接，span 数量的改变是指能否再对文本添加或删除 span。上面三个类都实现 Spanned 接口，SpannableString 和 SpannableStringBuilder 同时实现了 Spannable 接口，意味着可以对 span 进行增删，通过 setSpan() 方法和 removeSpan() 方法。

#### 2.2 简单使用

下面看下使用 SpannableStringBuilder 的简单操作，展示颜色 span、字体大小 span、段落 span、点击链接 span：

```java
private fun textSpan() {
    // 颜色
    val ssb = SpannableStringBuilder("Text is spantastic!")
    ssb.setSpan(ForegroundColorSpan(Color.RED), 8, 12, Spannable.SPAN_EXCLUSIVE_INCLUSIVE)
    text1.text = ssb

    // 字体大小
    val text = "Text with relative size span"
    val sbs2 = SpannableString(text)
    sbs2.setSpan(RelativeSizeSpan(1.5f), 10, 24, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE)
    text2.text = sbs2

    // 段落
    val str = "Hello World!"
    val sbs3 = SpannableStringBuilder(str)
    sbs3.setSpan(
        BulletSpan(20, Color.BLUE, 10), 0, str.length,
        Spannable.SPAN_EXCLUSIVE_EXCLUSIVE
    )
    sbs3.append("\n")
    val len1 = sbs3.length
    sbs3.append(str)
    sbs3.setSpan(
        BulletSpan(20, Color.BLUE, 10),
        len1,
        sbs3.length,
        Spannable.SPAN_EXCLUSIVE_EXCLUSIVE
    )
    text3.text = sbs3

    // 链接
    val sbs4 = SpannableString(text4.text)
    sbs4.setSpan(
        URLSpan("https://www.baidu.com/"),
        0,
        text4.text.length,
        Spannable.SPAN_EXCLUSIVE_EXCLUSIVE
    )
    text4.movementMethod = LinkMovementMethod()
    text4.setLinkTextColor(Color.parseColor("#004182"))
    text4.highlightColor = Color.TRANSPARENT
    text4.text = sbs4
}
```

![span_demo](https://github.com/RalfNick/PicRepository/raw/master/span/span_demo.png)

flag 参数说明：

>- Spannable.SPAN_EXCLUSIVE_EXCLUSIVE，已设置 span 不包括在前面和结尾插入字符串
>- Spannable.SPAN_EXCLUSIVE_INCLUSIVE，已设置 span 不包括在前面插入的字符串，包括结尾插入字符串
>- Spannable.SPAN_INCLUSIVE_INCLUSIVE，已设置 span 包括在前面插入的字符串和结尾插入字符串
>- Spannable.SPAN_INCLUSIVE_EXCLUSIVE，已设置 span 包括在前面插入的字符串，不包括结尾插入字符串

#### 2.3 span 分类

span 主要有两大类：字符处理类型和段落类型。

>* 字符处理类型：其中字符处理类型指 span 作用的对象是一个或者多个字符,该类型又可以分为外观类型和尺寸类型，从字面上可以知道外观类型指字体颜色，背景色，下划线等属性，尺寸类型指字体的大小，缩放等属性
>* 段落处理类型：段落处理类型的作用对象是一个段落，在 Android 中文字以 \n 结尾的视为一个段落。

![span_type](https://github.com/RalfNick/PicRepository/raw/master/span/span_type.png)

不同类型的 span 实现不同的标记接口来代表不同的类别：

> - CharacterStyle 字符样式接口，代表字符处理类型 span
> - ParagraphStyle 段落样式接口，代表段落类型 span
> - UpdateAppearance  外观接口，代表属于外观类型 span
> - UpdateLayout layout 接口，代表尺寸类型 span

常用的 span 有：ClickableSpan、DrawableMarginSpan、ImageSpan、RelativeSizeSpan、URLSpan、UnderlineSpan。除了 API 提供的 span，也可以自定义 span 来满足特定的需求，如设置文字 margin 的 span，设置图标的 span 等。

TextMarginSpan - 设置文字 margin

```java

class TextMarginSpan(
    @field:ColorInt private val mTextColor: Int,
    private val mLeftMargin: Int,
    private val mTopMargin: Int) : ReplacementSpan() {

    override fun getSize(
        paint: Paint,
        charSequence: CharSequence?,
        start: Int,
        end: Int,
        fm: FontMetricsInt?): Int {
        val subString = charSequence?.subSequence(start, end).toString()
        return (mLeftMargin + paint.measureText(subString)).toInt()
    }

    override fun draw(
        canvas: Canvas, text: CharSequence, start: Int, end: Int, x: Float,
        top: Int, y: Int, bottom: Int, paint: Paint) {
        paint.color = mTextColor
        canvas.drawText(text, start, end, x + mLeftMargin, y.toFloat() - mTopMargin, paint)
    }
}

```


CenterMarginImageSpan - 设置图标居中

```java
class CenterMarginImageSpan @JvmOverloads constructor(drawable: Drawable, source: String,
    private val mLeftMargin: Int = 0, private val mRightMargin: Int = 0
) : ImageSpan(drawable, source, ALIGN_BASELINE) {

  private var mDrawableRef: WeakReference<Drawable?>? = null

  override fun draw(canvas: Canvas, text: CharSequence, start: Int, end: Int, x: Float,
      top: Int, y: Int, bottom: Int, paint: Paint) {
    val drawable = cachedDrawable
    val fm = paint.fontMetricsInt
    val offset = (fm.descent - fm.ascent - drawable!!.bounds.bottom) / 2
    canvas.save()
    canvas.translate(mLeftMargin + x, y + fm.ascent + offset.toFloat())
    drawable.draw(canvas)
    canvas.restore()
  }

  override fun getSize(paint: Paint, text: CharSequence, start: Int, end: Int,
      fm: FontMetricsInt?): Int {
    return mLeftMargin + super.getSize(paint, text, start, end, fm) + mRightMargin
  }

  private val cachedDrawable: Drawable?
    get() {
      val wr = mDrawableRef
      var d: Drawable? = null
      if (wr != null) {
        d = wr.get()
      }
      if (d == null) {
        d = drawable
        mDrawableRef = WeakReference(d)
      }
      return d
    }
}
```

![span_custom](https://github.com/RalfNick/PicRepository/raw/master/span/span_custom.png)

#### 2.3 span 原理分析

span 是附属在文本上的，那么 span 是怎么对应相应的字符串上呢？设置字符索引 star 和 end，将 span 对象设置到对应的字符串上。在设置 span 时，还有一个 flag，除了存储 span，start 和 end，还有一个 flag，源码中用两个数组来记录。其中一个数组用来记录 span 对象，另一个数组用来记录 star、end 和flag，每三个一组，当前位置index 的 star 和 end、flag，会和 span 数组中的 span  对应上。

![span_data_struct](https://github.com/RalfNick/PicRepository/raw/master/span/span_data_struct.png)

以上就是 span 存储的基本原理，具体在代码实现上，对于 SpannedString、SpannableString，是通过继承 SpannableStringInternal 来实现的，增删 span 通过实现 Spanned 和 Spannable 来区分，即实现 Spanned 接口，代表 span 不能增删，实现 Spannable 代表 span 可以增删。

SpannableStringInternal 同时实现了 Spanned，Spannable，CharSequence 对应的功能，但是没有直接实现对应的接口，可以看成是一种委托， SpanedString  和 SpannableString 复用同一份代码，均将 Spanned，Spannable，CharSequence 接口实现委托给 SpannableStringInternal。看下主要的类图结构。

![span_main_class](https://github.com/RalfNick/PicRepository/raw/master/span/span_main_class.png)

SpannableStringInternal 内部主要的几个方法，构造方法、copySpans、setSpan、getSpans、removeSpan。

（1）首选看构造方法，构造方法个人认为最主要的目的是给 SpannedString 使用，因为 SpannedString 不能增删 span，所以需要构造一个 SpannedString 对象，需要通过一个已有的 CharSequence 来构造。SpannableStringInternal 构造方法主要就是看 CharSequence 中国是否包含 span，如果包含就将所有的 span 对象拷贝到新的 SpannableStringInternal 对象中，构造方法中调用的 copySpans 方法，copySpans 方法有两个重载方法，其主要区别就是判断 CharSequence 是否是 SpannableStringInternal 类型，如果不是 SpannableStringInternal 类型，那么承装 span、和 star、end、flag 的两数组 mSpans 和 mSpanData 需要创建，否则直接利用已经创建好的 mSpans 和 mSpanData，然后将 CharSequence 中的 span 和 对应 star、end、flag 拷贝到 mSpans 和 mSpanData 中。对于一个确定位置的 span，和 start、end、flag 的对应关系：

```java
    Object span = mSpans[i] = srcSpans[i];
    int start = mSpanData[i * COLUMNS + START];
    int end =mSpanData[i * COLUMNS + END];
    int flag = mSpanData[i * COLUMNS + FLAGS];
```

（2）getSpans 方法是查询属于当前 xx.class 的 span 对象

```java

public <T> T[] getSpans(int queryStart, int queryEnd, Class<T> kind) {
int count = 0;

int spanCount = mSpanCount;
Object[] spans = mSpans;
int[] data = mSpanData;
Object[] ret = null;
Object ret1 = null;

 for (int i = 0; i < spanCount; i++) {
    int spanStart = data[i * COLUMNS + START];
    int spanEnd = data[i * COLUMNS + END];
    
    // 省略边界判断代码等
    ....
    // 找到一个 span 时记录到 ret1，暂时不用创建结果数组
    if (count == 0) {
        ret1 = spans[i];
        count++;
    } else {
        // 找到第二个 span 时，需要创建数组，该数组最多有 spanCount - i + 1 个
        if (count == 1) {
            ret = (Object[]) Array.newInstance(kind, spanCount - i + 1);
            ret[0] = ret1;
        }
        // span 优先级处理，先获取优先级较大的，按照顺序放在数组中
        int prio = data[i * COLUMNS + FLAGS] & Spanned.SPAN_PRIORITY;
        if (prio != 0) {
            int j;
    
            for (j = 0; j < count; j++) {
                int p = getSpanFlags(ret[j]) & Spanned.SPAN_PRIORITY;
    
                if (prio > p) {
                    break;
                }
            }
    
            System.arraycopy(ret, j, ret, j + 1, count - j);
            ret[j] = spans[i];
            count++;
        } else {
            // 相同优先级的，直接放在数组中
            ret[count++] = spans[i];
        }
    }
}
    // 返回空数组
    if (count == 0) {
        return (T[]) ArrayUtils.emptyArray(kind);
    }
    // 返回只有一个元素的数组
    if (count == 1) {
        ret = (Object[]) Array.newInstance(kind, 1);
        ret[0] = ret1;
        return (T[]) ret;
    }
    // 上面 spanCount - i + 1 个元素装满数组，直接返回
    if (count == ret.length) {
        return (T[]) ret;
    }
    // 未装满需要截取数组
    Object[] nret = (Object[]) Array.newInstance(kind, count);
    System.arraycopy(ret, 0, nret, 0, count);
    return (T[]) nret;
}
```

（3）setSpan 方法

```java
private void setSpan(Object what, int start, int end, int flags, boolean enforceParagraph) {
    int nstart = start;
    int nend = end;

    // 省略代码范围检测、段落检测
    ...

    int count = mSpanCount;
    Object[] spans = mSpans;
    int[] data = mSpanData;

    // 如果span已经存在，重新复制
    for (int i = 0; i < count; i++) {
        if (spans[i] == what) {
            int ostart = data[i * COLUMNS + START];
            int oend = data[i * COLUMNS + END];

            data[i * COLUMNS + START] = start;
            data[i * COLUMNS + END] = end;
            data[i * COLUMNS + FLAGS] = flags;
            // 通知 span 改变，回调所有实现 SpanWatcher 接口的 span
            sendSpanChanged(what, ostart, oend, nstart, nend);
            return;
        }
    }
    
    // 如果 mSpans 容量不够，需要扩容
    if (mSpanCount + 1 >= mSpans.length) {
        Object[] newtags = ArrayUtils.newUnpaddedObjectArray(
                GrowingArrayUtils.growSize(mSpanCount));
        int[] newdata = new int[newtags.length * 3];

        System.arraycopy(mSpans, 0, newtags, 0, mSpanCount);
        System.arraycopy(mSpanData, 0, newdata, 0, mSpanCount * 3);

        mSpans = newtags;
        mSpanData = newdata;
    }

    // 将新添加的 span 放到 mSpans 数组中，start、end、flag 放到 mSpanData 中
    mSpans[mSpanCount] = what;
    mSpanData[mSpanCount * COLUMNS + START] = start;
    mSpanData[mSpanCount * COLUMNS + END] = end;
    mSpanData[mSpanCount * COLUMNS + FLAGS] = flags;
    mSpanCount++;

    if (this instanceof Spannable)
        sendSpanAdded(what, nstart, nend);
}
```

（4）removeSpan 方法

```java
public void removeSpan(Object what, int flags) {
    int count = mSpanCount;
    Object[] spans = mSpans;
    int[] data = mSpanData;

    for (int i = count - 1; i >= 0; i--) {
        if (spans[i] == what) {
            int ostart = data[i * COLUMNS + START];
            int oend = data[i * COLUMNS + END];

            int c = count - (i + 1);

            System.arraycopy(spans, i + 1, spans, i, c);
            System.arraycopy(data, (i + 1) * COLUMNS,
                    data, i * COLUMNS, c * COLUMNS);

            mSpanCount--;

            if ((flags & Spanned.SPAN_INTERMEDIATE) == 0) {
                sendSpanRemoved(what, ostart, oend);
            }
            return;
        }
    }
}
```
removeSpan 方法同理，也是找到对应的 span，如果找到，删除对应的 span，即把后面的 span 往前移动，同时也需要移动 start、end、flag。前面已经提到 SpannableString 适合数量不多的 span 的情况，由此可以知道原因，因为每次删除 span，都要移动数组，添加时，容量不够需要对数组进行扩容，如果 span 数量较多时，导致效率不高，所以在数量较少时是可以接受的。SpannableStringBuilder 内部使用的是线段树，实现会更为复杂一些，适合 span 数量较多的情况。

### 3. Html 转 span

#### 3.1 Html 简介

Html 是一种标记性语言，通过设置标签来表示不同的属性，而且 html 使用固有的标记，一般都是预定义的，一般前端会使用 Html + CSS 来设置页面样式，所以采用 Html 格式的字符串可以在不同的平台之间来解析设置文字样式，可以具有较好的通用性，不同的平台按照同样的规则解析得到统一的样式。由于 html 中的样式属性不多，所以会采用 CSS 在 html 的标签中嵌入样式属性。

![span_html](https://github.com/RalfNick/PicRepository/raw/master/span/span_html.png)

```html
<p style="color:blue;margin-left:20px;">这是一个段落。</p>
```

一般在移动端根据 html 解析的标签和 css 样式不会很多，一般都是常见的一些样式。

![span_css_text](https://github.com/RalfNick/PicRepository/raw/master/span/span_css_text.png)

由于 html 属于标记性语言，都是尖括号开头和结尾的标签，那么解析可以采用 xml 解析的形式来完成。在 android 中解析 xml 有三种方式：SAX、DOM、PULL。

![span_sax](https://github.com/RalfNick/PicRepository/raw/master/span/span_sax.png)

通过上述比较所以在 Android 中采用的是 SAX 解析的方式。

![span_xml_parse](https://github.com/RalfNick/PicRepository/raw/master/span/span_xml_parse.png)

下面一段代码是解析出字符串中的 URL，来设置 URL 的点击和颜色等，通过调用  Html.fromHtml() 得到 Spaned 字符串，从上面的介绍中 Spanned 接口不具有 getSpans 方法，所以通过 SpannableStringBuilder 重新构造，SpannableStringBuilder 实现了 Spannable 接口，可以获取所有 span。当然可以使用 SpannableString 来构造，同样有 getSpans 方法。

```java
    final String content = text.replace("\n", "<br>");
    final Spanned html = Html.fromHtml(content, null, tagHandler);
    final SpannableStringBuilder ssb = new SpannableStringBuilder(html);
    final URLSpan[] spans = ssb.getSpans(0, html.length(), URLSpan.class);
```

#### 3.2 Html 解析

Html.fromHtml() 的解析过程:

```java
public static Spanned fromHtml(String source, int flags, ImageGetter imageGetter,TagHandler tagHandler) {
    // 构造 parse
    Parser parser = new Parser();
    try {
        parser.setProperty(Parser.schemaProperty, HtmlParser.schema);
    } catch (org.xml.sax.SAXNotRecognizedException e) {
        // Should not happen.
        throw new RuntimeException(e);
    } catch (org.xml.sax.SAXNotSupportedException e) {
        // Should not happen.
        throw new RuntimeException(e);
    }
    // 通过 HtmlToSpannedConverter 解析 html，
    HtmlToSpannedConverter converter =
            new HtmlToSpannedConverter(source, imageGetter, tagHandler, parser, flags);
    return converter.convert();
}
```
fromHtml 方法解析过程中首先构造 parse，然后将 parse 传给 HtmlToSpannedConverter，通过 HtmlToSpannedConverter 来解析 html 的标签并存储到一个 SpannableStringBuilder 中。

```java
public Spanned convert() {
    mReader.setContentHandler(this);
    try {
        mReader.parse(new InputSource(new StringReader(mSource)));
    } catch (IOException e) {
        // We are reading from a string. There should not be IO problems.
        throw new RuntimeException(e);
    } catch (SAXException e) {
        // TagSoup doesn't throw parse exceptions.
        throw new RuntimeException(e);
    }
    
    ...

    return mSpannableStringBuilder;
}
```
上面构造的 parse 需要设置一个 ContentHandler 接口，HtmlToSpannedConverter 实现了 ContentHandler 接口，所以在解析过程中主要会回调 startElement 和 endElement，来解析出 html 标签中属性。


```java
public interface ContentHandler {
    
    public void startDocument () throws SAXException;
    
    public void endDocument() throws SAXException;
    
    public void startElement (String uri, String localName,String qName, Attributes atts) throws SAXException;
    
    public void endElement (String uri, String localName,String qName) throws SAXException;
    
    public void characters (char ch[], int start, int length) throws SAXException;
}
```

在 startElement 方法中会调用 handleStartTag 方法来解析开始标签，在 endElement 方法中会调用 handleEndTag 方法

```java
// 在 startElement 中调用，处理 html 开始标签
private void handleStartTag(String tag, Attributes attributes) {
    if (tag.equalsIgnoreCase("br")) {
        // We don't need to handle this. TagSoup will ensure that there's a </br> for each <br>
        // so we can safely emit the linebreaks when we handle the close tag.
    } else if (tag.equalsIgnoreCase("p")) {
        startBlockElement(mSpannableStringBuilder, attributes, getMarginParagraph());
        startCssStyle(mSpannableStringBuilder, attributes);
    } 
    
    ...
    
}

// 在 endElement 中调用，处理 html 结束标签
private void handleEndTag(String tag) {
    if (tag.equalsIgnoreCase("br")) {
        handleBr(mSpannableStringBuilder);
    } else if (tag.equalsIgnoreCase("p")) {
        endCssStyle(mSpannableStringBuilder);
        endBlockElement(mSpannableStringBuilder);
    } else if (tag.equalsIgnoreCase("ul")) {
        endBlockElement(mSpannableStringBuilder);
    }
    
}
```

解析过程中会通过正则匹配标签中的 css 属性，从而设置对应的 span，设置 span 的过程中有一个技巧，开始时设置一个占位 span，当遍历到结束标签时，找到前面最近的一个标签，然后移除 span，再设置 span，此时该 span 对应的 start 和 end 才是对应相应字符串的起始位置。

```java
// 开始标签设置一个 span，setSpan 开始和结束位置是 start = end = len
private static void start(Editable text, Object mark) {
    int len = text.length();
    text.setSpan(mark, len, len, Spannable.SPAN_INCLUSIVE_EXCLUSIVE);
}

// 结束标签处理时找到最近的一个 span，然后移除重新设置span，此时才能更正 start 和 end
private static void end(Editable text, Class kind, Object repl) {
    int len = text.length();
    Object obj = getLast(text, kind);
    if (obj != null) {
        setSpanFromMark(text, obj, repl);
    }
}
```

#### 3.3 Span 转 html

span 转 html 相对简单一些，找出 text 中的 span，根据 span 类型拼接 html 格式的字符串.

```java
public static String toHtml(Spanned text, int option) {
    StringBuilder out = new StringBuilder();
    withinHtml(out, text, option);
    return out.toString();
}
```

#### 3.4 其他 tag 解析

html 解析除了提供 html 中预定义的一些标签，还可以对图片以及自定义的一些标签进行处理，解析过程中提供了回调接口。

(1)自定义 tag 解析回调接口

```java
public static interface TagHandler {

    public void handleTag(boolean opening, String tag,Editable output, XMLReader xmlReader);
}
```

(2) 图片解析回调接口，解析 html 中 img 标签

```java
public static interface ImageGetter {
    public Drawable getDrawable(String source);
}
```

### 4 处理技巧和优化

（1）由于 html 解析涉及 IO 处理，如，在使用 SAX 解析 html 标签过程就有 IO 处理，所以该过程属于耗时操作。特别是在有 RecyclerView 在给 TextView 设置数据时，如果在主线程处理后端返回的 html 字符串，可能导致页面卡顿，滑动不够流畅。对于这种情况，有两种时机可以处理：

> - 一种是在 setText 之前，利用异步处理，在子线程中处理字符串，处理完在主线程 setText
> - 另一种是在接口返回数据之后直接处理，也就是说数据模型中直接保存处理好的字符串，在显示视图时直接 setText 即可

（2）测量文字宽度的几种方式

**Paint.measureText**

```java
  Paint paint = new Paint();
  paint.setTextSize(size);
  float strWidth = paint.measureText(str);
```

**Paint.getTextBounds (获得文字所在矩形区域，可以得到宽高)**

 ```java
   Paint paint = new Paint();
   Rect rect = new Rect();  
   paint.getTextBounds(str, 0, str.length(), rect);  
   int w = rect.width();  
   int h = rect.height();
 ```

**使用 StaticLayout 或者 DynamicLayout**
 
StaticLayout 或者 DynamicLayout 对于多行文字计算比较方便，如果视图中的文字设置完之后不改变，选择 StaticLayout 测量，效率较 DynamicLayout 高。如果是 EditView 中设置文字，并且文字可改变，那么选择使用 DynamicLayout 来测量。

 ```java
  DynamicLayout dynamicLayout = new DynamicLayout(charSequence, mTextView.getPaint(),lineBreakWidth, DynamicLayout.Alignment.ALIGN_NORMAL, 0f, 0,false);
  // 获取文字行数
  final int lineCount = dynamicLayout.getLineCount();
  // 获取某一行宽度
  dynamicLayout.getLineWidth(lineCount - 1)
 ```

（3）setText 优化

如果使用 setText(CharSequence text)，TextView 会对传入的 Spannable 拷贝一份，生成 SpannedString 保存在内存中，并且其类型是 CharSequence。这样也就是说文本和 span 是不可改变的。所以当想要更新文字和 span 时，需要再创建一个 Spannable 对象，然后调用 setText()，这样会触发 TextView 重新测量和绘制。如果文字不变，span 会改变，可以使用 setText(CharSequence text, TextView.BufferType type) 方法，传入 BufferType.SPANNABLE 参数，在内存中保存的是 Spannable，这样就可以对这个 Spannable 进行 span 的添加和删除操作，同时 TextView 会自动更新。有一种情况，如果是不是增删 span，而是修改已有的 span，需要调用 invalidate() 或 requestLayout() 来刷新，如果是外观类的 span，选择 invalidate() 方法，如果是 尺寸类 span，选择 requestLayout()。

```java
textView.setText(spannable, BufferType.SPANNABLE);
Spannable spannableText = (Spannable) textView.getText();
spannableText.setSpan(new ForegroundColorSpan(color),8, spannableText.getLength(),SPAN_INCLUSIVE_INCLUSIVE);
```

（4）使用 Spannable.Factory 优化

在 RecyclerView 中使用的 TextView，由于 TextView 是复用的，所以 setText 重复调用不可避免，每次调用 setText 方法，extView 都会新创建一个 SpannableString 对象，因为在 setText 方法中会调用 mSpannableFactory.newSpannable(text) 方法。为避免重复创建 SpannableString 对象，可以给 TextView 设置一个自定义的 Spannable.Factory，同时设置 BufferType.SPANNABLE，而且也要求我们掺入的 text 类型也是一个我们创建好的 Spannable 类型的字符串。这样每次调用 setText 方法时，TextView 就不会额外创建一个 SpannableString 对象，减少内存占用。

```java
// 1
Spannable.Factory spannableFactory = new Spannable.Factory(){
    @Override
    public Spannable newSpannable(CharSequence source) {
        return (Spannable) source;
    }
};

// 2
textView.setSpannableFactory(spannableFactory);

// 3
textView.setText(spannableObject, BufferType.SPANNABLE);

```

（5）LinkMovementMethod 使用注意事项

 **代码设置：**

由于设置 setMovementMethod 后，会使得 TextView 拥有焦点，并且可以点击 setFocusable、setClickable、setLongClickable 均设置为 true，此时需要根据场景来确定是否将这些开关设置为 true。setFocusable 设置为 true 在 RecyclerView 中可能会引起滚动定位问题，所以如果不需要获取焦点，最好设置为 false。

```java
    mTitleTextView.setMovementMethod(LinkMovementMethod.getInstance());
    mTitleTextView.setHighlightColor(Color.TRANSPARENT);
    mTitleTextView.setFocusable(false);
    mTitleTextView.setClickable(false);
    mTitleTextView.setLongClickable(false);
```

**xml 中设置**

```
android:clickable="false"
android:longClickable="false"
android:focusable="false"
android:linksClickable="true"
android:autoLink="all"

```

### 5 参考

[Spans](https://developer.android.com/guide/topics/text/spans#kotlin)
