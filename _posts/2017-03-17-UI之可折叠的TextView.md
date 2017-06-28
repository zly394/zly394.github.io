### 先上效果
![](https://ww3.sinaimg.cn/large/006tNbRwgy1fdmk51rek3g30900g0ndq.gif)

### 一、思路

#### 1. 计算text的行数

实现可折叠的TextView最重要的一点是在setText()前计算出text所需的行数   
计算行数需要分为两种情况

##### 1.1 没有换行符的text

```
    行数等于text的宽度除于TextView的宽度
    再判断text的宽度对TextView的宽度取余是否为0，如果不等于0则加1
    lines = textWidth / TextViewWidth + textWidth % TextViewWidth == 0 ? 0 : 1
```

##### 1.2 含有换行符的text

```
    1. 先用换行符拆分 
    2. 对于拆分后的文本
        如果不为空，则然后再按照没有换行符的方式计算   
        如果为空，则行数为1
    3. 累加所有的拆分文本行数
```

#### 2. 截取text

计算出text的行数之后，需要对text进行截取，截取到text能在指定的行数内显示完的位置，

1. 首先用换行符对text进行拆分，将text分为若干段落
2. 对拆分后的文本段落循环计算行数累加，并累加字符数
3. 累加的行数小于指定行数，继续循环，直到累加的行数大于指定行数或循环完成；如果在循环完成之前累加的行数大于指定行数，则截取该次循环的段落
4. 调用[TextUtils](https://developer.android.com/reference/android/text/TextUtils.html)的ellipsize()方法对指定的段落进行截取，ellipsize()方法中的avail参数，传入剩余的可显示宽度   
   
   因为在文本的最后要拼接上“...提示文本”，所以可显宽度的计算方式如下：

    ```
    TextViewWidth * (指定行数 - 累加行数) - (... + 提示文本)Width
    ```
 5. 把截取后的文本设置给TextView

### 二、实现

实现可折叠的TextView需要继承TextView并重写setText(CharSequence text, BufferType type)方法

> 因为setText(CharSequence text)方法是final的，并且setText(CharSequence text)最终调用的也是setText(CharSequence text, BufferType type)方法，所以重写后者即可。

核心代码

```
/**
 * 末尾省略号
 */
private static final String ELLIPSE = "...";
/**
 * 默认的折叠行数
 */
public static final int COLLAPSED_LINES = 4;
/**
 * 折叠时的默认文本
 */
private static final String EXPANDED_TEXT = "展开全文";
/**
 * 展开时的默认文本
 */
private static final String COLLAPSED_TEXT = "收起全文";
/**
 * 在文本末尾
 */
public static final int END = 0;
/**
 * 在文本下方
 */
public static final int BOTTOM = 1;
/**
 * 提示文字展示的位置
 */
@IntDef({END, BOTTOM})
@Retention(RetentionPolicy.SOURCE)
public @interface TipsGravityMode {}
/**
 * 折叠的行数
 */
private int mCollapsedLines;
/**
 * 折叠时的文本
 */
private String mExpandedText;
/**
 * 展开时的文本
 */
private String mCollapsedText;
/**
 * 折叠时的图片资源
 */
private Drawable mExpandedDrawabl
/**
 * 展开时的图片资源
 */
private Drawable mCollapsedDrawab
/**
 * 原始的文本
 */
private CharSequence mOriginalTex
/**
 * TextView中文字可显示的宽度
 */
private int mShowWidth;
/**
 * 是否是展开的
 */
private boolean mIsExpanded;
/**
 * 提示文字位置
 */
private int mTipsGravity;
/**
 * 提示文字颜色
 */
private int mTipsColor;
/**
 * 提示文字是否显示下划线
 */
private boolean mTipsUnderline;
/**
 * 提示是否可点击
 */
private boolean mTipsClickable;

... 

@Override
public void setText(CharSequence text, final BufferType type) {
    // 如果text为空或mCollapsedLines为0则直接显示
    if (TextUtils.isEmpty(text) || mCollapsedLines == 0) {
        super.setText(text, type);
    } else if (mIsExpanded) {
        // 保存原始文本，去掉文本末尾的空字符
        this.mOriginalText = CharUtil.trimFrom(text);
        formatExpandedText(type);
    } else {
        // 保存原始文本，去掉文本末尾的空字符
        this.mOriginalText = CharUtil.trimFrom(text);
        // 获取TextView中文字显示的宽度，需要在layout之后才能获取到，避免在列表中重复获取
        if (mCollapsedLines > 0 && mShowWidth == 0) {
            getViewTreeObserver().addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
                @Override
                public void onGlobalLayout() {
                    getViewTreeObserver().removeOnGlobalLayoutListener(this);
                    mShowWidth = getWidth() - getPaddingLeft() - getPaddingRight();
                    formatCollapsedText(type);
                }
            });
        } else {
            formatCollapsedText(type);
        }
    }
}

/**
 * 格式化折叠时的文本
 *
 * @param type ref android.R.styleable#TextView_bufferType
 */
private void formatCollapsedText(BufferType type) {
    // 将原始文本按换行符拆分成段落
    String[] paragraphs = mOriginalText.toString().split("\\n");
    // 获取paint，用于计算文字宽度
    TextPaint paint = getPaint();
    // 文字宽度
    float textWidth;
    // 字符数，用于最后截取字符串
    int charCount = 0;
    // 剩余行数
    int lastLines = mCollapsedLines;
    for (int i = 0; i < paragraphs.length; i++) {
        // 每个段落
        String paragraph = paragraphs[i];
        // 每个段落文本的宽度
        textWidth = paint.measureText(paragraph);
        // 计算每段的行数
        int paragraphLines = (int) (textWidth / mShowWidth);
        // 如果该段为空（表示空行）或还有余，多加一行
        if (TextUtils.isEmpty(paragraph) || textWidth % mShowWidth != 0) {
            paragraphLines++;
        }
        if (paragraphLines < lastLines) {
            // 如果该段落行数小于等于剩余的行数，则减少lastLines，并增加字符数
            // 这里只计算字符数，并不拼接字符
            charCount += paragraph.length() + 1;
            lastLines -= paragraphLines;
            if (i == paragraphs.length - 1) {
                super.setText(mOriginalText, type);
                break;
            }
        } else if (paragraphLines == lastLines && i == paragraphs.length - 1) {
            // 如果该段落行数等于剩余行数，并且是最后一个段落，表示刚好能够显示完全
            super.setText(mOriginalText, type);
            break;
        } else {
            // 如果该段落的行数大于等于剩余的行数，则格式化文本
            // 因设置的文本可能是带有样式的文本，如SpannableStringBuilder，所以根据计算的字符数从原始文本中截取
            SpannableStringBuilder spannable = new SpannableStringBuilder(mOriginalText, 0, charCount);
            // 计算后缀的宽度，因样式的问题对后缀的宽度乘2
            int expandedTextWidth = 2 * (int) (paint.measureText(ELLIPSE + mExpandedText));
            // 获取最后一段的文本，还是因为原始文本的样式原因不能直接使用paragraphs中的文本
            CharSequence lastParagraph = mOriginalText.subSequence(charCount, charCount + paragraph.length());
            // 对最后一段文本进行截取
            CharSequence ellipsizeText = TextUtils.ellipsize(lastParagraph, paint,
                    mShowWidth * lastLines - expandedTextWidth, TextUtils.TruncateAt.END);
            spannable.append(ellipsizeText);
            // 如果lastParagraph == ellipsizeText表示最后一段文本在可显示范围内，此时需要手动加上"..."
            // 如果lastParagraph != ellipsizeText表示进行了截取TextUtils.ellipsize()方法会自动加上"..."
            if (lastParagraph == ellipsizeText) {
                spannable.append(ELLIPSE);
            }
            // 设置样式
            setSpan(spannable);
            // 使点击有效
            setMovementMethod(LinkMovementMethod.getInstance());
            super.setText(spannable, type);
            break;
        }
    }
}

/**
 * 格式化展开式的文本，直接在后面拼接即可
 *
 * @param type
 */
private void formatExpandedText(BufferType type) {
    SpannableStringBuilder spannable = new SpannableStringBuilder(mOriginalText);
    setSpan(spannable);
    super.setText(spannable, type);
}

/**
 * 设置提示的样式
 *
 * @param spannable 需修改样式的文本
 */
private void setSpan(SpannableStringBuilder spannable) {
    Drawable drawable;
    // 根据提示文本需要展示的文字拼接不同的字符
    if (mTipsGravity == END) {
        spannable.append(" ");
    } else {
        spannable.append("\n");
    }
    int tipsLen;
    // 判断是展开还是收起
    if (mIsExpanded) {
        spannable.append(mCollapsedText);
        drawable = mCollapsedDrawable;
        tipsLen = mCollapsedText.length();
    } else {
        spannable.append(mExpandedText);
        drawable = mExpandedDrawable;
        tipsLen = mExpandedText.length();
    }
    // 设置点击事件
    spannable.setSpan(new ExpandedClickableSpan(), spannable.length() - tipsLen,
            spannable.length(), Spanned.SPAN_INCLUSIVE_EXCLUSIVE);
    // 如果提示的图片资源不为空，则使用图片代替提示文本
    if (drawable != null) {
        spannable.setSpan(new ImageSpan(drawable, ImageSpan.ALIGN_BASELINE),
                spannable.length() - tipsLen, spannable.length(), Spanned.SPAN_INCLUSIVE_EXCLUSIVE);
    }
}

/**
 * 提示的点击事件
 */
private class ExpandedClickableSpan extends ClickableSpan {
    @Override
    public void onClick(View widget) {
        // 是否可点击
        if (mTipsClickable) {
            mIsExpanded = !mIsExpanded;
            setText(mOriginalText);
        }
    }
    @Override
    public void updateDrawState(TextPaint ds) {
        // 设置提示文本的颜色和是否需要下划线
        ds.setColor(mTipsColor == 0 ? ds.linkColor : mTipsColor);
        ds.setUnderlineText(mTipsUnderline);
    }
}
```

##### 因为用户设置给TextView的文本可能是含有样式的文本，即实现了Spannable接口的文本，所以在拆分并拼接文本的时候不能直接使用拆分后的字符串，会丢失原有样式，需要重新在原始文本中截取

**[可以从这里获取代码](https://github.com/zly394/CollapsedTextView)**