### 前言
Android 的富文本还是 HTML 标签的那一套东西，通过解析 HTML，拿到不同的标签，然后渲染成不同的样式。但和在 web 端还是有很大不同的，比如 Android 默认支持的标签很少，而且功能都比较简单，多数情况都需要自己定义；而且客户端在打开一个链接的时候又是并不像 web 那样打开个页面就行了，对于图片、视频等可能需要用本地的页面打开，这时候也需要自定义一下标签和对应的 `span`。对 `span` 还不是很了解的可以参考之前的文章：  

[Android TextView 富文本之 android.text.style.xxxSpan](https://juejin.im/post/5c7b40fee51d453ecd04a23c)  
[Android TextView 富文本之 ImageSpan](https://juejin.im/post/5c7d2267e51d45523b0f72a6)  
[Android TextView 富文本之 ClickableSpan](https://juejin.im/post/5c84902ce51d453ce668b750)

### Html 解析

Android 的 `android.text.Html` 是一个专门用来解析 html 的工具，使用也很简单，直接调用静态方法 `public static Spanned fromHtml(String source)` 就行。接下来大概看下解析的流程和一些需要关注的地方。  

1、`HtmlToSpannedConverter`   

``` java
public static Spanned fromHtml(String source, int flags, ImageGetter imageGetter, TagHandler tagHandler) {
    Parser parser = new Parser();
    // ...
    parser.setProperty(Parser.schemaProperty, HtmlParser.schema);
    // ...
    HtmlToSpannedConverter converter = new HtmlToSpannedConverter(source, imageGetter, tagHandler, parser, flags);
    return converter.convert();
}
```
`fromHtml` 先创建了一个解析器 `Parser` 指定 `HtmlParser.schema` 来解析 html，然后用 `HtmlToSpannedConverter` 实现了 html 标签到 span 的转换，`HtmlToSpannedConverter` 才是我们要关注的重点。简单看下它的构造方法：
``` java
public HtmlToSpannedConverter( String source, Html.ImageGetter imageGetter, Html.TagHandler tagHandler, Parser parser, int flags) {
    // 输入的文本
    mSource = source;
    // 要输出的带 Span 的文本
    mSpannableStringBuilder = new SpannableStringBuilder();
    // 自定义获取图片的工具，没深入研究过，不多阐述
    mImageGetter = imageGetter;
    // 自定义的标签处理器，本文关注的重点
    mTagHandler = tagHandler;
    // 刚才的 Parser，是一个 XMLReader
    mReader = parser;
    mFlags = flags;
}
```  
2、`HtmlToSpannedConverter.convert`  

``` java
public Spanned convert() {
    mReader.setContentHandler(this);
    // ...
    mReader.parse(new InputSource(new StringReader(mSource)));
    // ...
    return mSpannableStringBuilder;
}
```
这里先设置 `mReader` 的 `ContentHandler` 为自己，然后调用 `mReader` 的 `parse` 解析刚才传入的 `mSource`，解析过程中就会回调到 `ContentHandler`，在回调中处理各种标签的开始结束等，最后返回结果。  

3、`ContentHandler`  

大致来看下 `ContentHandler` 定义：
``` java
public interface ContentHandler {
    // ...
    // 处理标签的开始
    public void startElement (String uri, String localName, String qName, Attributes atts) throws SAXException;\
    // 处理标签的结束
    public void endElement (String uri, String localName, String qName) throws SAXException;
    // ch 中就是我们要处理的字符，start 和 length 表示当前要处理的那些字符的位置
    public void characters (char ch[], int start, int length) throws SAXException;
    // ...
}
```
我们只关心这三个方法，接下来看下 `HtmlToSpannedConverter` 是怎么实现者三个方法的。  
首先 `characters` 会把要处理的字符串分成快传给我们，这里只进行简单处理，注释写的也很明白，就是过滤掉连续的空格，换行也按空格处理。因为 html 有对应的换行标签 `<br>`，所以这里会把换行过滤掉，但是对于普通文本夹带部分 html 标签的情况，换行符过滤掉显示就会有问题，我们可以在解析之前把换行符替换成 `<br>`。  
``` java
public void characters(char ch[], int start, int length) throws SAXException {
    StringBuilder sb = new StringBuilder();

    /*
    * Ignore whitespace that immediately follows other whitespace;
    * newlines count as spaces.
    */
    for (int i = 0; i < length; i++) {
        char c = ch[i + start];

        if (c == ' ' || c == '\n') {
            char pred;
            int len = sb.length();

            if (len == 0) {
                len = mSpannableStringBuilder.length();

                if (len == 0) {
                    pred = '\n';
                } else {
                    pred = mSpannableStringBuilder.charAt(len - 1);
                }
            } else {
                pred = sb.charAt(len - 1);
            }

            if (pred != ' ' && pred != '\n') {
                sb.append(' ');
            }
        } else {
            sb.append(c);
        }
    }
    mSpannableStringBuilder.append(sb);
}
```
`startElement` 会调用到 `handleStartTag`，处理标签的开始部分，在这里我们也可以看到 `Html` 工具默认支持的那些标签。  
``` java
private void handleStartTag(String tag, Attributes attributes) {
    // ...
    else if (tag.equalsIgnoreCase("a")) {
        startA(mSpannableStringBuilder, attributes);
    }
    // ...
    else if (tag.equalsIgnoreCase("img")) {
        startImg(mSpannableStringBuilder, attributes, mImageGetter);
    } 
    // ...
    else if (mTagHandler != null) {
        mTagHandler.handleTag(true, tag, mSpannableStringBuilder, mReader);
    }
}
```
`endElement` 也对应调用 `handleEndTag` 处理标签的结束。这里有一点需要注意的是，我们自定义的 `mTagHandler` 是放在这么多 `if-else` 的最后的，也就是说我们不能处理 `Html` 工具已经支持的那些标签的，而为了保证兼容性，和后端同学重新定义一套标签规则也不太现实，这时候就需要先把这些标签替换成 `Html` 工具不支持的，我们自己可以随便定义，然后再解析替换后的标签。  
 
4、标签的处理  

这里我们简单以 `a` 标签为例，简单看下做了哪些事情。  
``` java
private static void startA(Editable text, Attributes attributes) {
    String href = attributes.getValue("", "href");
    start(text, new Href(href));
}

private static void start(Editable text, Object mark) {
    int len = text.length();
    text.setSpan(mark, len, len, Spannable.SPAN_INCLUSIVE_EXCLUSIVE);
}

private static void endA(Editable text) {
    Href h = getLast(text, Href.class);
    if (h != null) {
        if (h.mHref != null) {
            setSpanFromMark(text, h, new URLSpan((h.mHref)));
        }
    }
}
```
`a` 标签在开始的时候拿到 `href` 属性，然后存到 `Href` 中，并将 `Href` 以 `span` 的形式设置到 `text` 中保存下来，这时候我们只能知道标签的开始位置；然后在结束的时候我们可以得知标签的结束位置，再把 `Href` 拿出来，重新设置上 `URLSpan`，这个 `span` 会在点击的时候打开一个链接，默认是用浏览器打开。  
刚才说的 `<br>` 标签则是简单的在文本后面加上 `'\n'`。
``` java
private static void handleBr(Editable text) {
    text.append('\n');
}
```
其他的标签处理也类似，这里不再赘述。  

### 自定义 Html.TagHandler

日常工作中，使用最多的可能就是 `a` 标签了，打开图片、视频和页面等，都可能需要我们自己处理点击和跳转，这里就以 `a` 标签为例进行说明。  
之前说过 `Html` 工具支持 `a` 标签，而且我们自定义的 `mTagHandler` 优先级比较低，这里就需要坐下简单的预处理，把 `a` 标签替换成 `Html` 工具不认识的就行。  
``` java
private static final String A_PATTERN = "<\s*[aA] (.*?)>(.*?)<\s*/[aA]\s*>";
private static final String SELF_DEF_A_TAG = "zdl_a";
private static final String A_REPLACE = "<" + SELF_DEF_A_TAG +" $1>$2</" + SELF_DEF_A_TAG + ">";

public String preProcess(String text) {
    return text.replaceAll(A_PATTERN, A_REPLACE);
}
```
预处理完成后我们就可以愉快的解析标签了，我们可以参考 `Html` 工具的方式，在开始时把属性读出来，然后以 `span` 的形式存起来，在结束时拿出来，最后再根据属性生成对应样式的 `span`。  
`public void handleTag(boolean opening, String tag, Editable output, XMLReader xmlReader)` 方法给了 `XMLReader`，根据它获取属性的方法可以参考 [How to get an attribute from an XMLReader](https://stackoverflow.com/questions/6952243/how-to-get-an-attribute-from-an-xmlreader?rq=1)  
我们定义一个存储属性的 `span`：
``` java
class AttributeSpan {
    public HashMap<String, String> attrs = new HashMap<>();
    public int start;
    public int end;
}
```
在标签开始时读取并保存所有的属性，结束时再读出来使用。  
``` java
public void handleTag(boolean opening, String tag, Editable output, XMLReader xmlReader) {
    if(!SELF_DEF_A_TAG.equalsIgnoreCase(tag)) { return; }
    if (opening) {
        AttributeSpan span = new AttributeSpan();
        span.start = output.length();
        // 这里是把属性存到 HashMap 中
        TagHandler.getAttribute(span.attrs, xmlReader);
        output.setSpan(span, span.start, span.start, Spanned.SPAN_INCLUSIVE_EXCLUSIVE);
    } else {
        AttributeSpan span = TagHandler.getLast(output, AttributeSpan.class);
        if (span != null) {
            span.end = output.length();
            // 由于图片、视频和页面都是以 a 标签的形式存在的，因此还需要用 class 属性进行区分
            // 然后根据不同类型添加不同的 span，比如我们可以定义一个 ToastSpan 在点击时弹 toast 显示所有属性
            // 而默认的，我们就用 URLSpan，对于图片、视频和页面等的特殊处理，我们只需要新增 class 属性和对应的 span
            String clazz = span.attrs.get("class");
            if (clazz == null) { clazz = ""; }
            Object realSpan;
            switch (clazz) {
                case ToastSpan.CLASS:
                    realSpan = new ToastSpan(span);
                    break;
                default:
                    realSpan = new URLSpan(span.attrs.get("href"));
                    break;
            }
            output.setSpan(realSpan, span.start, span.end, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
        }
    }
}

public static class ToastSpan extends ClickableSpan {
    public static final String CLASS = "toast";
    private final AttributeSpan attributeSpan;

    public ToastSpan(AttributeSpan attributeSpan) {
        this.attributeSpan = attributeSpan;
    }

    @Override
    public void onClick(@NonNull View widget) {
        StringBuilder builder = new StringBuilder();
        for (String key: attributeSpan.attrs.keySet()) {
            if (builder.length() > 0) {
                builder.append(", ");
            }
            builder.append(key);
            builder.append(": ");
            builder.append(attributeSpan.attrs.get(key));
        }
        Toast.makeText(widget.getContext(), builder, Toast.LENGTH_SHORT).show();
    }
}
```
### 总结
Android TextView 的富文本相对简单，对应的 HTML 解析也不是很复杂，其实也就是标签到 `span` 的一个转换过程，这里贴上 demo 地址 [RichTextDemo](https://github.com/funnywolfdadada/RichTextDemo)，感兴趣的可以参考一下。 
