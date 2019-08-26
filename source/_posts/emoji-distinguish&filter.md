---
title: Emoji 识别与过滤
date: 2018-07-15 2:50:33
categories:
- 技术向
tags:
- Java
---

后台提了一个需求，要求用户输入上传的内容中不能带 Emoji。网上有一些资料，都提到了过滤 Emoji 的方法，但都存在多过滤或少过滤的情况。我从官方的标准资料入手，希望能解决掉这个问题。

那我们先看看 Emoji 有什么特征。

## Emoji 是什么

Emoji 就是可以在文字中输入的表情符。想必大家都用过：

😄😊😃😍😉

看到这些图标，不用多说了吧。当前正式标准为 11.0 版本，Emoji 是 Unicode 的一部分，它在 Unicode 中有对应的码点（ CodePoint），也就是说，Emoji 符号就是一个文字。

根据 [Emoji](https://en.wikipedia.org/wiki/Emoji) 维基百科说明，当前版本中共有 1212 个 Emoji ，实际上这指的是单码点的 Emoji，而还有一些 Emoji 是通过多个码点组合而成。

例如"零宽度连接符"（ZERO WIDTH JOINER，缩写 ZWJ）`U+200D`。将`U+1F468：男人` `U+1F469：女人`  `U+1F467：女孩`这三个码点使用`U+200D`连接起来，`U+1F468 U+200D U+1F469 U+200D U+1F467`，就会显示为一个 Emoji 👨‍👩‍👧，表示他们组成的家庭。如果用户的系统不支持这种方法，就还是显示为三个独立的 Emoji 👨👩👧。

例如如代表肤色的(U+1F3FB–U+1F3FF): 🏻 🏼 🏽 🏾 🏿 ，发色的（U+1F9B0-U+1F9B3）,组合起来后得到同一个表情的不同肤色版本，这一特性在国际大厂的输入法上可以看到，例如 Apple、Google、Samsung 的输入法上都可以输入。

例如 `U+1F1E8 ` `U+1F1F3` 组合起来成了中国国旗。

由于多码点组合的存在，可以显示的 Emoji 实际上数量多于 1212 个。

<!--more-->

## Emoji 的识别

一个字符串是否包含了 Emoji？通过上面的描述，我们可以想到，如果字符串中包含了 Emoji 的码点，那不就说明该字符串包含了 Emoji 吗？因此，我们先获取一份完整的码点集合用来判断。

### Emoji 所有的码点

那 Emoji 的码点有哪些呢，Unicode 组织的 [Unicode® Emoji Charts v11.0](https://www.unicode.org/emoji/charts/index.html) 页面中可以找到完整的 Emoji 码点数据：[emoji-data.txt](https://unicode.org/Public/emoji/11.0/) ，这个表的内容的解读可见参考资料。

数据经过以下脚本`getEmojiData.sh` 处理，可以得到一个完整的、**有重复** 的码表。

```shell
#! /bin/bash
cat "$1" |
grep -v ^# |
grep -v ^$ |
while read line
do
	echo $line | cut -d \; -f 1
done
```

```
$ ./getEmojiData.sh emoji-data.txt > emoji-all-data.txt
```

得到的格式如下，数量数百行，列出来的码点不重复的有 3000 多个：

```
0023
002A
0030..0039
00A9
00AE
...
```

由于这个码点表有重复元素，我们选择将所有码点添加到 Set 集合中。代码如下（使用列编辑模式可快速编辑完成）：

```java
public class EmojiUtils {

    private static final String TAG = EmojiUtils.class.getSimpleName();
    private static Set<Character> emojiSignatureSet = new HashSet<>();

    private EmojiUtils() {}

    static {
        // 省略……
        addUnicodeToSet(emojiSignatureSet, 0x2122);
        addUnicodeToSet(emojiSignatureSet, 0x2139);
        addUnicodeToSet(emojiSignatureSet, 0x2194, 0x2199);
        addUnicodeToSet(emojiSignatureSet, 0x21A9, 0x21AA);
        // 省略……
    }

    private static void addUnicodeToSet(Set<Character> set, int code) {
        if (set == null) {
            return;
        }
        set.add((char) code);
    }

    private static void addUnicodeToSet(Set<Character> set, int codeStart, int codeEnd) {
        if (set == null) {
            return;
        }
        for (int i = codeStart; i <= codeEnd; i++) {
            addUnicodeToSet(set, i);
        }
    }
}
```

### 初试 Emoji 识别

我们有了码表，识别方法就好说了，将字符串拆成单个字符，逐一判断是否是 Emoji 特征码点。

```java
public static boolean isContainEmoji(String s) {
    char[] chars = s.toCharArray();
    int charsLength = chars.length;

    for (int i = 0; i < charsLength; i++) {
        char c = chars[i];
        if (emojiSignatureSet.contains(c)) {
            return true;
        }
    }
    return false;
}
```

有了识别的方法 `EmojiUtils.isContainEmoji(String s)` 后，我们来实践一下，过滤掉输入的 Emoji：

```java
etTest.setFilters(new InputFilter[]{new InputFilter() {
    @Override
    public CharSequence filter(CharSequence source, int start, int end, Spanned dest, int dstart, int dend) {
        if (EmojiUtils.isContainEmoji(source.toString())) {
            return "";
        }
        return source;
    }
}});
```

运行起来，你会发现**并……没……有……用……** ，想怎么输入就怎么输入。

难道这些码点不对吗？当然，这是 Unicode 标准提供的码点表，不可能不对，那一定是判断时出了问题，我们查看一下输入 Emoji 时输入的是什么字符。

### Emoji 的表现形式

通过断点，可以看到，输入一个微笑的 Emoji，其内容实际上是 '\uD83D''\uDE03' ，好像离码点 0x1F603 有点远。实际上，这个输入的编码是特殊的。

Unicode 中计划使用 17 个平面，常用的编码都在第 0 平面中（关于 Unicode 更多知识可以从参考资料进行了解），而在第 0 平面中，有一个特殊的代理区域，不表示任何字符，只用于指向第 1 到第 16 个平面中的字符，这段区域是：D800—DFFF.。其中 0xD800—0xDBFF 是前导代理(lead surrogates)，0xDC00—0xDFFF 是后尾代理(trail surrogates)。

它们的代理关系如下图所示：

![](https://img-blog.csdn.net/20140831182025921)

因此具体的公式是：`0x10000 + (前导-0xD800) * 0x400 + (后导-0xDC00) = utf-16编码`

我们将微笑 Emoji 的字符串代入计算，结果是：`0x10000+(0xD83D - 0xD800)*0x400 + (0xDE03-0xDC00) = 0x1F603` ，与码点正好对应上了！

因此，我们需要修改一下判断方法：

```java
public static boolean isContainEmoji(String s) {
    char[] chars = s.toCharArray();
    int charsLength = chars.length;

    for (int i = 0; i < charsLength; i++) {
        char c = chars[i];
        char realChar = c;
        if (c >= 0xD800 && c <= 0xDBFF && ++i < charsLength) {
            char nextChar = chars[i];
            realChar = (char) (0x10000 + (c - 0xD800) * 0x400 + (nextChar - 0xDC00));
        }
        if (emojiSignatureSet.contains(realChar)) {
            return true;
        }
    }
    return false;
}
```

修改过后，使用不同的输入法都尝试一下输入 Emoji，果然，全都被过滤了，无法输入。搞定收工！给自己输入一个`666`!

嗯？我的 666 呢？这时候，你会发现，无法输入：数字、英文、@、# 等符号。果然没这么简单！

### 码点表中的奸细

显然，被多过滤掉了字符一定是因为码点集合太多了。仔细查看，终于发现了问题所在。

#### 数字

0023          ; Emoji_Component      #  1.1  [1] (#️)       number sign             
002A          ; Emoji_Component      #  1.1  [1] (\*️)       asterisk  
0030..0039    ; Emoji_Component      #  1.1 [10] (0️..9️)    digit zero..digit nine

这些 # * 0~9 这些字符本身是正常的字符，但是它们搭配其他的特征码则变成了 Emoji。       

因此，这些字符不能加入特征集合中。数字的问题解决了，还有字母的问题。

#### 字母

根据 [ Tags (Unicode block) ]( https://en.wikipedia.org/wiki/Tags_(Unicode_block) ) 和 [emoji-sequences.txt](http://unicode.org/Public/emoji/5.0/emoji-sequences.txt)

E0020 ~ E007F 的使用仅有

1F3F4 E0067 E0062 E0065 E006E E0067 E007F; Emoji_Tag_Sequence; England  #  7.0  [1] (🏴󠁧󠁢󠁥󠁮󠁧󠁿)  
1F3F4 E0067 E0062 E0073 E0063 E0074 E007F; Emoji_Tag_Sequence; Scotland  #  7.0  [1] (🏴󠁧󠁢󠁳󠁣󠁴󠁿)
1F3F4 E0067 E0062 E0077 E006C E0073 E007F; Emoji_Tag_Sequence; Wales  #  7.0  [1] (🏴󠁧󠁢󠁷󠁬󠁳󠁿)    

而这个范围覆盖了 a~z 及一些符号，如果添加了反而会误判，或者仅添加 E007F 作为 Emoji 特征码

去掉这些奸细，终于可以愉快地输入了……吗？并没有。

还会发现不能输入中文的一些标点符号。再看看还有什么内容没必要添加的。

#### 保留区域

在 emoji-data 中可以看到有一部分码点标记为 reserved，即当前保留着不用，例如`<reserved-1F02C>..<reserved-1F02F>`，那么把这些保留区域去除是否就可以了呢，经过实验，去除之后确实就没问题了，特别是最后一行，保留区域`1FA6E..1FFFD`个数达 1424 个，去除这个就可以正常输入中文字符了。

## Emoji 的过滤

有了上面提到的特征码点集合，过滤和识别其实是一样的。

```java
public static String filterEmoji(String s) {
    StringBuilder sb = new StringBuilder();
    char[] chars = s.toCharArray();
    int charsLength = chars.length;

    for (int i = 0; i < charsLength; i++) {
        char c = chars[i];
        char realChar = c;
        if (c >= 0xD800 && c <= 0xDBFF && ++i < charsLength) {
            char nextChar = chars[i];
            realChar = (char) (0x10000 + (c - 0xD800) * 0x400 + (nextChar - 0xDC00));
        }
        if (!emojiSignatureSet.contains(realChar)) {
            sb.append(c);
        }
    }
    return sb.toString();
}
```

## 参考链接

通过一番探索，虽然对于字符编码相关的知识还不是特别清晰，但至少比以前了解得更多了。在实现过程中参考了诸多的网上的资料，如果我写得你觉得看得不甚了了，可以看看下面这些资料。

* [字符编码的奥秘utf-8, Unicode](https://blog.csdn.net/hherima/article/details/8655200)：Unicode 多平面的理解
* [iPhone emoji问题牵出的Unicode代理区的思考](https://blog.csdn.net/hherima/article/details/38961575)：代理区的转化识别
* [Android 准确过滤(禁止) Emoji表情](https://www.jianshu.com/p/1c04c3617469)：实现方案的参考来源
* [Emoji](https://en.wikipedia.org/wiki/Emoji)
* [Unicode® Emoji Charts v11.0](https://www.unicode.org/emoji/charts/index.html)
* [Full Emoji List, v11.0](http://www.unicode.org/emoji/charts/full-emoji-list.html)
* [Unicode Emoji Data Files](http://www.unicode.org/Public/emoji/latest/) ：对数据文件的解读可看

  * [Unicode® Technical Standard #51 UNICODE EMOJI](https://www.unicode.org/reports/tr51/index.html)

  * [Emoji， 没想到你是这样的...](https://www.jianshu.com/p/9682f8ce1260)
* [ Tags (Unicode block) ](https://en.wikipedia.org/wiki/Tags_(Unicode_block))
* [Emoji与unicode特殊字符的处理](https://www.cnblogs.com/hahahjx/p/4522913.html)
* [百行代码集成Emoji并转成iOS、后台可识别字符](https://blog.csdn.net/myatlantis/article/details/60770246)

## 附：完整代码

以下是完整的代码，大家可以自行测试是否有问题，发现问题的话也麻烦反馈给我。

```java
public class EmojiUtils {

    private static final String TAG = EmojiUtils.class.getSimpleName();
    private static Set<Character> emojiSignatureSet = new HashSet<>(1801);

    private EmojiUtils() {}

    public static boolean isContainEmoji(String s) {
        char[] chars = s.toCharArray();
        int charsLength = chars.length;

        for (int i = 0; i < charsLength; i++) {
            char c = chars[i];
            char realChar = c;
            if (c >= 0xD800 && c <= 0xDBFF && ++i < charsLength) {
                char nextChar = chars[i];
                realChar = (char) (0x10000 + (c - 0xD800) * 0x400 + (nextChar - 0xDC00));
            }
            if (emojiSignatureSet.contains(realChar)) {
                return true;
            }
        }
        return false;
    }

    public static String filterEmoji(String s) {
        StringBuilder sb = new StringBuilder();
        char[] chars = s.toCharArray();
        int charsLength = chars.length;

        for (int i = 0; i < charsLength; i++) {
            char c = chars[i];
            char realChar = c;
            if (c >= 0xD800 && c <= 0xDBFF && ++i < charsLength) {
                char nextChar = chars[i];
                realChar = (char) (0x10000 + (c - 0xD800) * 0x400 + (nextChar - 0xDC00));
            }
            if (!emojiSignatureSet.contains(realChar)) {
                sb.append(c);
            }
        }
        return sb.toString();
    }

    private static void addUnicodeToSet(Set<Character> set, int code) {
        if (set == null) {
            return;
        }
        set.add((char) code);
    }

    private static void addUnicodeToSet(Set<Character> set, int codeStart, int codeEnd) {
        if (set == null) {
            return;
        }
        for (int i = codeStart; i <= codeEnd; i++) {
            addUnicodeToSet(set, i);
        }
    }

    static {
        Log.d(TAG, "initSignatureSet start");
        // -------------------------------------------------------------------
        // Part 1： 个数 1212 + 26 = 1238
        // -------------------------------------------------------------------
        // 根据 https://en.wikipedia.org/wiki/Emoji
        // Emoji 11.0 列有 1212 个
        // 而从 https://unicode.org/Public/emoji/11.0/emoji-data.txt 可知实际上远不止这个数量
        // 即使其列出的 # All omitted code points have Emoji=No 总共也有 1250 个
        // 多出来的分别是：
        // 0023         #
        // 002A         *
        // 0030..0039   0..9
        // 1F1E6..1F1FF A..Z
        // 从 http://unicode.org/Public/emoji/5.0/emoji-sequences.txt
        // 可以看到这些多出来的码是什么样的 Emoji
        // A..Z 之间相互组合成 AC 等，由于它们不是正常字符，所以我们后面加入特征码表中
        // 1F1E6 1F1E8   ; Emoji_Flag_Sequence       ; Ascension Island #  6.0  [1] (🇦🇨)
        // 而 # * 0~9 这些字符本身是正常的字符，但是它们搭配其他的特征码则变成了 Emoji
        // # Emoji Keycap Sequence
        // 0023 FE0F 20E3; Emoji_Keycap_Sequence     ; keycap: \x{23} #  3.0  [1] (#️⃣)
        // 002A FE0F 20E3; Emoji_Keycap_Sequence     ; keycap: *      #  3.0  [1] (*️⃣)
        // 0030 FE0F 20E3; Emoji_Keycap_Sequence     ; keycap: 0      #  3.0  [1] (0️⃣)
        // 0031 FE0F 20E3; Emoji_Keycap_Sequence     ; keycap: 1      #  3.0  [1] (1️⃣)
        // 0032 FE0F 20E3; Emoji_Keycap_Sequence     ; keycap: 2      #  3.0  [1] (2️⃣)
        // 0033 FE0F 20E3; Emoji_Keycap_Sequence     ; keycap: 3      #  3.0  [1] (3️⃣)
        // 0034 FE0F 20E3; Emoji_Keycap_Sequence     ; keycap: 4      #  3.0  [1] (4️⃣)
        // 0035 FE0F 20E3; Emoji_Keycap_Sequence     ; keycap: 5      #  3.0  [1] (5️⃣)
        // 0036 FE0F 20E3; Emoji_Keycap_Sequence     ; keycap: 6      #  3.0  [1] (6️⃣)
        // 0037 FE0F 20E3; Emoji_Keycap_Sequence     ; keycap: 7      #  3.0  [1] (7️⃣)
        // 0038 FE0F 20E3; Emoji_Keycap_Sequence     ; keycap: 8      #  3.0  [1] (8️⃣)
        // 0039 FE0F 20E3; Emoji_Keycap_Sequence     ; keycap: 9      #  3.0  [1] (9️⃣)
        // addUnicodeToSet(emojiSignatureSet, 0x0023);
        // addUnicodeToSet(emojiSignatureSet, 0x002A);
        // addUnicodeToSet(emojiSignatureSet, 0x0030, 0x0039);
        addUnicodeToSet(emojiSignatureSet, 0x00A9);
        addUnicodeToSet(emojiSignatureSet, 0x00AE);
        addUnicodeToSet(emojiSignatureSet, 0x203C);
        addUnicodeToSet(emojiSignatureSet, 0x2049);
        addUnicodeToSet(emojiSignatureSet, 0x2122);
        addUnicodeToSet(emojiSignatureSet, 0x2139);
        addUnicodeToSet(emojiSignatureSet, 0x2194, 0x2199);
        addUnicodeToSet(emojiSignatureSet, 0x21A9, 0x21AA);
        addUnicodeToSet(emojiSignatureSet, 0x231A, 0x231B);
        addUnicodeToSet(emojiSignatureSet, 0x2328);
        addUnicodeToSet(emojiSignatureSet, 0x23CF);
        addUnicodeToSet(emojiSignatureSet, 0x23E9, 0x23F3);
        addUnicodeToSet(emojiSignatureSet, 0x23F8, 0x23FA);
        addUnicodeToSet(emojiSignatureSet, 0x24C2);
        addUnicodeToSet(emojiSignatureSet, 0x25AA, 0x25AB);
        addUnicodeToSet(emojiSignatureSet, 0x25B6);
        addUnicodeToSet(emojiSignatureSet, 0x25C0);
        addUnicodeToSet(emojiSignatureSet, 0x25FB, 0x25FE);
        addUnicodeToSet(emojiSignatureSet, 0x2600, 0x2604);
        addUnicodeToSet(emojiSignatureSet, 0x260E);
        addUnicodeToSet(emojiSignatureSet, 0x2611);
        addUnicodeToSet(emojiSignatureSet, 0x2614, 0x2615);
        addUnicodeToSet(emojiSignatureSet, 0x2618);
        addUnicodeToSet(emojiSignatureSet, 0x261D);
        addUnicodeToSet(emojiSignatureSet, 0x2620);
        addUnicodeToSet(emojiSignatureSet, 0x2622, 0x2623);
        addUnicodeToSet(emojiSignatureSet, 0x2626);
        addUnicodeToSet(emojiSignatureSet, 0x262A);
        addUnicodeToSet(emojiSignatureSet, 0x262E, 0x262F);
        addUnicodeToSet(emojiSignatureSet, 0x2638, 0x263A);
        addUnicodeToSet(emojiSignatureSet, 0x2640);
        addUnicodeToSet(emojiSignatureSet, 0x2642);
        addUnicodeToSet(emojiSignatureSet, 0x2648, 0x2653);
        addUnicodeToSet(emojiSignatureSet, 0x265F, 0x2660);
        addUnicodeToSet(emojiSignatureSet, 0x2663);
        addUnicodeToSet(emojiSignatureSet, 0x2665, 0x2666);
        addUnicodeToSet(emojiSignatureSet, 0x2668);
        addUnicodeToSet(emojiSignatureSet, 0x267B);
        addUnicodeToSet(emojiSignatureSet, 0x267E, 0x267F);
        addUnicodeToSet(emojiSignatureSet, 0x2692, 0x2697);
        addUnicodeToSet(emojiSignatureSet, 0x2699);
        addUnicodeToSet(emojiSignatureSet, 0x269B, 0x269C);
        addUnicodeToSet(emojiSignatureSet, 0x26A0, 0x26A1);
        addUnicodeToSet(emojiSignatureSet, 0x26AA, 0x26AB);
        addUnicodeToSet(emojiSignatureSet, 0x26B0, 0x26B1);
        addUnicodeToSet(emojiSignatureSet, 0x26BD, 0x26BE);
        addUnicodeToSet(emojiSignatureSet, 0x26C4, 0x26C5);
        addUnicodeToSet(emojiSignatureSet, 0x26C8);
        addUnicodeToSet(emojiSignatureSet, 0x26CE);
        addUnicodeToSet(emojiSignatureSet, 0x26CF);
        addUnicodeToSet(emojiSignatureSet, 0x26D1);
        addUnicodeToSet(emojiSignatureSet, 0x26D3, 0x26D4);
        addUnicodeToSet(emojiSignatureSet, 0x26E9, 0x26EA);
        addUnicodeToSet(emojiSignatureSet, 0x26F0, 0x26F5);
        addUnicodeToSet(emojiSignatureSet, 0x26F7, 0x26FA);
        addUnicodeToSet(emojiSignatureSet, 0x26FD);
        addUnicodeToSet(emojiSignatureSet, 0x2702);
        addUnicodeToSet(emojiSignatureSet, 0x2705);
        addUnicodeToSet(emojiSignatureSet, 0x2708, 0x2709);
        addUnicodeToSet(emojiSignatureSet, 0x270A, 0x270B);
        addUnicodeToSet(emojiSignatureSet, 0x270C, 0x270D);
        addUnicodeToSet(emojiSignatureSet, 0x270F);
        addUnicodeToSet(emojiSignatureSet, 0x2712);
        addUnicodeToSet(emojiSignatureSet, 0x2714);
        addUnicodeToSet(emojiSignatureSet, 0x2716);
        addUnicodeToSet(emojiSignatureSet, 0x271D);
        addUnicodeToSet(emojiSignatureSet, 0x2721);
        addUnicodeToSet(emojiSignatureSet, 0x2728);
        addUnicodeToSet(emojiSignatureSet, 0x2733, 0x2734);
        addUnicodeToSet(emojiSignatureSet, 0x2744);
        addUnicodeToSet(emojiSignatureSet, 0x2747);
        addUnicodeToSet(emojiSignatureSet, 0x274C);
        addUnicodeToSet(emojiSignatureSet, 0x274E);
        addUnicodeToSet(emojiSignatureSet, 0x2753, 0x2755);
        addUnicodeToSet(emojiSignatureSet, 0x2757);
        addUnicodeToSet(emojiSignatureSet, 0x2763, 0x2764);
        addUnicodeToSet(emojiSignatureSet, 0x2795, 0x2797);
        addUnicodeToSet(emojiSignatureSet, 0x27A1);
        addUnicodeToSet(emojiSignatureSet, 0x27B0);
        addUnicodeToSet(emojiSignatureSet, 0x27BF);
        addUnicodeToSet(emojiSignatureSet, 0x2934, 0x2935);
        addUnicodeToSet(emojiSignatureSet, 0x2B05, 0x2B07);
        addUnicodeToSet(emojiSignatureSet, 0x2B1B, 0x2B1C);
        addUnicodeToSet(emojiSignatureSet, 0x2B50);
        addUnicodeToSet(emojiSignatureSet, 0x2B55);
        addUnicodeToSet(emojiSignatureSet, 0x3030);
        addUnicodeToSet(emojiSignatureSet, 0x303D);
        addUnicodeToSet(emojiSignatureSet, 0x3297);
        addUnicodeToSet(emojiSignatureSet, 0x3299);
        addUnicodeToSet(emojiSignatureSet, 0x1F004);
        addUnicodeToSet(emojiSignatureSet, 0x1F0CF);
        addUnicodeToSet(emojiSignatureSet, 0x1F170, 0x1F171);
        addUnicodeToSet(emojiSignatureSet, 0x1F17E);
        addUnicodeToSet(emojiSignatureSet, 0x1F17F);
        addUnicodeToSet(emojiSignatureSet, 0x1F18E);
        addUnicodeToSet(emojiSignatureSet, 0x1F191, 0x1F19A);
        // 1F1E6 1F1E8   ; Emoji_Flag_Sequence       ; Ascension Island #  6.0  [1] (🇦🇨)
        // addUnicodeToSet(emojiSignatureSet, 0x1F1E6, 0x1F1FF);
        addUnicodeToSet(emojiSignatureSet, 0x1F201, 0x1F202);
        addUnicodeToSet(emojiSignatureSet, 0x1F21A);
        addUnicodeToSet(emojiSignatureSet, 0x1F22F);
        addUnicodeToSet(emojiSignatureSet, 0x1F232, 0x1F23A);
        addUnicodeToSet(emojiSignatureSet, 0x1F250, 0x1F251);
        addUnicodeToSet(emojiSignatureSet, 0x1F300, 0x1F320);
        addUnicodeToSet(emojiSignatureSet, 0x1F321);
        addUnicodeToSet(emojiSignatureSet, 0x1F324, 0x1F32C);
        addUnicodeToSet(emojiSignatureSet, 0x1F32D, 0x1F32F);
        addUnicodeToSet(emojiSignatureSet, 0x1F330, 0x1F335);
        addUnicodeToSet(emojiSignatureSet, 0x1F336);
        addUnicodeToSet(emojiSignatureSet, 0x1F337, 0x1F37C);
        addUnicodeToSet(emojiSignatureSet, 0x1F37D);
        addUnicodeToSet(emojiSignatureSet, 0x1F37E, 0x1F37F);
        addUnicodeToSet(emojiSignatureSet, 0x1F380, 0x1F393);
        addUnicodeToSet(emojiSignatureSet, 0x1F396, 0x1F397);
        addUnicodeToSet(emojiSignatureSet, 0x1F399, 0x1F39B);
        addUnicodeToSet(emojiSignatureSet, 0x1F39E, 0x1F39F);
        addUnicodeToSet(emojiSignatureSet, 0x1F3A0, 0x1F3C4);
        addUnicodeToSet(emojiSignatureSet, 0x1F3C5);
        addUnicodeToSet(emojiSignatureSet, 0x1F3C6, 0x1F3CA);
        addUnicodeToSet(emojiSignatureSet, 0x1F3CB, 0x1F3CE);
        addUnicodeToSet(emojiSignatureSet, 0x1F3CF, 0x1F3D3);
        addUnicodeToSet(emojiSignatureSet, 0x1F3D4, 0x1F3DF);
        addUnicodeToSet(emojiSignatureSet, 0x1F3E0, 0x1F3F0);
        addUnicodeToSet(emojiSignatureSet, 0x1F3F3, 0x1F3F5);
        addUnicodeToSet(emojiSignatureSet, 0x1F3F7);
        addUnicodeToSet(emojiSignatureSet, 0x1F3F8, 0x1F3FF);
        addUnicodeToSet(emojiSignatureSet, 0x1F400, 0x1F43E);
        addUnicodeToSet(emojiSignatureSet, 0x1F43F);
        addUnicodeToSet(emojiSignatureSet, 0x1F440);
        addUnicodeToSet(emojiSignatureSet, 0x1F441);
        addUnicodeToSet(emojiSignatureSet, 0x1F442, 0x1F4F7);
        addUnicodeToSet(emojiSignatureSet, 0x1F4F8);
        addUnicodeToSet(emojiSignatureSet, 0x1F4F9, 0x1F4FC);
        addUnicodeToSet(emojiSignatureSet, 0x1F4FD);
        addUnicodeToSet(emojiSignatureSet, 0x1F4FF);
        addUnicodeToSet(emojiSignatureSet, 0x1F500, 0x1F53D);
        addUnicodeToSet(emojiSignatureSet, 0x1F549, 0x1F54A);
        addUnicodeToSet(emojiSignatureSet, 0x1F54B, 0x1F54E);
        addUnicodeToSet(emojiSignatureSet, 0x1F550, 0x1F567);
        addUnicodeToSet(emojiSignatureSet, 0x1F56F, 0x1F570);
        addUnicodeToSet(emojiSignatureSet, 0x1F573, 0x1F579);
        addUnicodeToSet(emojiSignatureSet, 0x1F57A);
        addUnicodeToSet(emojiSignatureSet, 0x1F587);
        addUnicodeToSet(emojiSignatureSet, 0x1F58A, 0x1F58D);
        addUnicodeToSet(emojiSignatureSet, 0x1F590);
        addUnicodeToSet(emojiSignatureSet, 0x1F595, 0x1F596);
        addUnicodeToSet(emojiSignatureSet, 0x1F5A4);
        addUnicodeToSet(emojiSignatureSet, 0x1F5A5);
        addUnicodeToSet(emojiSignatureSet, 0x1F5A8);
        addUnicodeToSet(emojiSignatureSet, 0x1F5B1, 0x1F5B2);
        addUnicodeToSet(emojiSignatureSet, 0x1F5BC);
        addUnicodeToSet(emojiSignatureSet, 0x1F5C2, 0x1F5C4);
        addUnicodeToSet(emojiSignatureSet, 0x1F5D1, 0x1F5D3);
        addUnicodeToSet(emojiSignatureSet, 0x1F5DC, 0x1F5DE);
        addUnicodeToSet(emojiSignatureSet, 0x1F5E1);
        addUnicodeToSet(emojiSignatureSet, 0x1F5E3);
        addUnicodeToSet(emojiSignatureSet, 0x1F5E8);
        addUnicodeToSet(emojiSignatureSet, 0x1F5EF);
        addUnicodeToSet(emojiSignatureSet, 0x1F5F3);
        addUnicodeToSet(emojiSignatureSet, 0x1F5FA);
        addUnicodeToSet(emojiSignatureSet, 0x1F5FB, 0x1F5FF);
        addUnicodeToSet(emojiSignatureSet, 0x1F600);
        addUnicodeToSet(emojiSignatureSet, 0x1F601, 0x1F610);
        addUnicodeToSet(emojiSignatureSet, 0x1F611);
        addUnicodeToSet(emojiSignatureSet, 0x1F612, 0x1F614);
        addUnicodeToSet(emojiSignatureSet, 0x1F615);
        addUnicodeToSet(emojiSignatureSet, 0x1F616);
        addUnicodeToSet(emojiSignatureSet, 0x1F617);
        addUnicodeToSet(emojiSignatureSet, 0x1F618);
        addUnicodeToSet(emojiSignatureSet, 0x1F619);
        addUnicodeToSet(emojiSignatureSet, 0x1F61A);
        addUnicodeToSet(emojiSignatureSet, 0x1F61B);
        addUnicodeToSet(emojiSignatureSet, 0x1F61C, 0x1F61E);
        addUnicodeToSet(emojiSignatureSet, 0x1F61F);
        addUnicodeToSet(emojiSignatureSet, 0x1F620, 0x1F625);
        addUnicodeToSet(emojiSignatureSet, 0x1F626, 0x1F627);
        addUnicodeToSet(emojiSignatureSet, 0x1F628, 0x1F62B);
        addUnicodeToSet(emojiSignatureSet, 0x1F62C);
        addUnicodeToSet(emojiSignatureSet, 0x1F62D);
        addUnicodeToSet(emojiSignatureSet, 0x1F62E, 0x1F62F);
        addUnicodeToSet(emojiSignatureSet, 0x1F630, 0x1F633);
        addUnicodeToSet(emojiSignatureSet, 0x1F634);
        addUnicodeToSet(emojiSignatureSet, 0x1F635, 0x1F640);
        addUnicodeToSet(emojiSignatureSet, 0x1F641, 0x1F642);
        addUnicodeToSet(emojiSignatureSet, 0x1F643, 0x1F644);
        addUnicodeToSet(emojiSignatureSet, 0x1F645, 0x1F64F);
        addUnicodeToSet(emojiSignatureSet, 0x1F680, 0x1F6C5);
        addUnicodeToSet(emojiSignatureSet, 0x1F6CB, 0x1F6CF);
        addUnicodeToSet(emojiSignatureSet, 0x1F6D0);
        addUnicodeToSet(emojiSignatureSet, 0x1F6D1, 0x1F6D2);
        addUnicodeToSet(emojiSignatureSet, 0x1F6E0, 0x1F6E5);
        addUnicodeToSet(emojiSignatureSet, 0x1F6E9);
        addUnicodeToSet(emojiSignatureSet, 0x1F6EB, 0x1F6EC);
        addUnicodeToSet(emojiSignatureSet, 0x1F6F0);
        addUnicodeToSet(emojiSignatureSet, 0x1F6F3);
        addUnicodeToSet(emojiSignatureSet, 0x1F6F4, 0x1F6F6);
        addUnicodeToSet(emojiSignatureSet, 0x1F6F7, 0x1F6F8);
        addUnicodeToSet(emojiSignatureSet, 0x1F6F9);
        addUnicodeToSet(emojiSignatureSet, 0x1F910, 0x1F918);
        addUnicodeToSet(emojiSignatureSet, 0x1F919, 0x1F91E);
        addUnicodeToSet(emojiSignatureSet, 0x1F91F);
        addUnicodeToSet(emojiSignatureSet, 0x1F920, 0x1F927);
        addUnicodeToSet(emojiSignatureSet, 0x1F928, 0x1F92F);
        addUnicodeToSet(emojiSignatureSet, 0x1F930);
        addUnicodeToSet(emojiSignatureSet, 0x1F931, 0x1F932);
        addUnicodeToSet(emojiSignatureSet, 0x1F933, 0x1F93A);
        addUnicodeToSet(emojiSignatureSet, 0x1F93C, 0x1F93E);
        addUnicodeToSet(emojiSignatureSet, 0x1F940, 0x1F945);
        addUnicodeToSet(emojiSignatureSet, 0x1F947, 0x1F94B);
        addUnicodeToSet(emojiSignatureSet, 0x1F94C);
        addUnicodeToSet(emojiSignatureSet, 0x1F94D, 0x1F94F);
        addUnicodeToSet(emojiSignatureSet, 0x1F950, 0x1F95E);
        addUnicodeToSet(emojiSignatureSet, 0x1F95F, 0x1F96B);
        addUnicodeToSet(emojiSignatureSet, 0x1F96C, 0x1F970);
        addUnicodeToSet(emojiSignatureSet, 0x1F973, 0x1F976);
        addUnicodeToSet(emojiSignatureSet, 0x1F97A);
        addUnicodeToSet(emojiSignatureSet, 0x1F97C, 0x1F97F);
        addUnicodeToSet(emojiSignatureSet, 0x1F980, 0x1F984);
        addUnicodeToSet(emojiSignatureSet, 0x1F985, 0x1F991);
        addUnicodeToSet(emojiSignatureSet, 0x1F992, 0x1F997);
        addUnicodeToSet(emojiSignatureSet, 0x1F998, 0x1F9A2);
        addUnicodeToSet(emojiSignatureSet, 0x1F9B0, 0x1F9B9);
        addUnicodeToSet(emojiSignatureSet, 0x1F9C0);
        addUnicodeToSet(emojiSignatureSet, 0x1F9C1, 0x1F9C2);
        addUnicodeToSet(emojiSignatureSet, 0x1F9D0, 0x1F9E6);
        addUnicodeToSet(emojiSignatureSet, 0x1F9E7, 0x1F9FF);
        // 以上为 https://en.wikipedia.org/wiki/Emoji 列出的 1212 个 Emoji
        addUnicodeToSet(emojiSignatureSet, 0x1F1E6, 0x1F1FF);

        // -------------------------------------------------------------------
        // 以下这部分则是 emoji-data.txt 完整的部分。
        // 与上面列出的相比，多了一些杂项符号，zwj、肤色、发色的组合符号，去除了保留字符
        // -------------------------------------------------------------------
        addUnicodeToSet(emojiSignatureSet, 0x231A, 0x231B);
        addUnicodeToSet(emojiSignatureSet, 0x23E9, 0x23EC);
        addUnicodeToSet(emojiSignatureSet, 0x23F0);
        addUnicodeToSet(emojiSignatureSet, 0x23F3);
        addUnicodeToSet(emojiSignatureSet, 0x25FD, 0x25FE);
        addUnicodeToSet(emojiSignatureSet, 0x2614, 0x2615);
        addUnicodeToSet(emojiSignatureSet, 0x2648, 0x2653);
        addUnicodeToSet(emojiSignatureSet, 0x267F);
        addUnicodeToSet(emojiSignatureSet, 0x2693);
        addUnicodeToSet(emojiSignatureSet, 0x26A1);
        addUnicodeToSet(emojiSignatureSet, 0x26AA, 0x26AB);
        addUnicodeToSet(emojiSignatureSet, 0x26BD, 0x26BE);
        addUnicodeToSet(emojiSignatureSet, 0x26C4, 0x26C5);
        addUnicodeToSet(emojiSignatureSet, 0x26CE);
        addUnicodeToSet(emojiSignatureSet, 0x26D4);
        addUnicodeToSet(emojiSignatureSet, 0x26EA);
        addUnicodeToSet(emojiSignatureSet, 0x26F2, 0x26F3);
        addUnicodeToSet(emojiSignatureSet, 0x26F5);
        addUnicodeToSet(emojiSignatureSet, 0x26FA);
        addUnicodeToSet(emojiSignatureSet, 0x26FD);
        addUnicodeToSet(emojiSignatureSet, 0x2705);
        addUnicodeToSet(emojiSignatureSet, 0x270A, 0x270B);
        addUnicodeToSet(emojiSignatureSet, 0x2728);
        addUnicodeToSet(emojiSignatureSet, 0x274C);
        addUnicodeToSet(emojiSignatureSet, 0x274E);
        addUnicodeToSet(emojiSignatureSet, 0x2753, 0x2755);
        addUnicodeToSet(emojiSignatureSet, 0x2757);
        addUnicodeToSet(emojiSignatureSet, 0x2795, 0x2797);
        addUnicodeToSet(emojiSignatureSet, 0x27B0);
        addUnicodeToSet(emojiSignatureSet, 0x27BF);
        addUnicodeToSet(emojiSignatureSet, 0x2B1B, 0x2B1C);
        addUnicodeToSet(emojiSignatureSet, 0x2B50);
        addUnicodeToSet(emojiSignatureSet, 0x2B55);
        addUnicodeToSet(emojiSignatureSet, 0x1F004);
        addUnicodeToSet(emojiSignatureSet, 0x1F0CF);
        addUnicodeToSet(emojiSignatureSet, 0x1F18E);
        addUnicodeToSet(emojiSignatureSet, 0x1F191, 0x1F19A);
        addUnicodeToSet(emojiSignatureSet, 0x1F1E6, 0x1F1FF);
        addUnicodeToSet(emojiSignatureSet, 0x1F201);
        addUnicodeToSet(emojiSignatureSet, 0x1F21A);
        addUnicodeToSet(emojiSignatureSet, 0x1F22F);
        addUnicodeToSet(emojiSignatureSet, 0x1F232, 0x1F236);
        addUnicodeToSet(emojiSignatureSet, 0x1F238, 0x1F23A);
        addUnicodeToSet(emojiSignatureSet, 0x1F250, 0x1F251);
        addUnicodeToSet(emojiSignatureSet, 0x1F300, 0x1F320);
        addUnicodeToSet(emojiSignatureSet, 0x1F32D, 0x1F32F);
        addUnicodeToSet(emojiSignatureSet, 0x1F330, 0x1F335);
        addUnicodeToSet(emojiSignatureSet, 0x1F337, 0x1F37C);
        addUnicodeToSet(emojiSignatureSet, 0x1F37E, 0x1F37F);
        addUnicodeToSet(emojiSignatureSet, 0x1F380, 0x1F393);
        addUnicodeToSet(emojiSignatureSet, 0x1F3A0, 0x1F3C4);
        addUnicodeToSet(emojiSignatureSet, 0x1F3C5);
        addUnicodeToSet(emojiSignatureSet, 0x1F3C6, 0x1F3CA);
        addUnicodeToSet(emojiSignatureSet, 0x1F3CF, 0x1F3D3);
        addUnicodeToSet(emojiSignatureSet, 0x1F3E0, 0x1F3F0);
        addUnicodeToSet(emojiSignatureSet, 0x1F3F4);
        addUnicodeToSet(emojiSignatureSet, 0x1F3F8, 0x1F3FF);
        addUnicodeToSet(emojiSignatureSet, 0x1F400, 0x1F43E);
        addUnicodeToSet(emojiSignatureSet, 0x1F440);
        addUnicodeToSet(emojiSignatureSet, 0x1F442, 0x1F4F7);
        addUnicodeToSet(emojiSignatureSet, 0x1F4F8);
        addUnicodeToSet(emojiSignatureSet, 0x1F4F9, 0x1F4FC);
        addUnicodeToSet(emojiSignatureSet, 0x1F4FF);
        addUnicodeToSet(emojiSignatureSet, 0x1F500, 0x1F53D);
        addUnicodeToSet(emojiSignatureSet, 0x1F54B, 0x1F54E);
        addUnicodeToSet(emojiSignatureSet, 0x1F550, 0x1F567);
        addUnicodeToSet(emojiSignatureSet, 0x1F57A);
        addUnicodeToSet(emojiSignatureSet, 0x1F595, 0x1F596);
        addUnicodeToSet(emojiSignatureSet, 0x1F5A4);
        addUnicodeToSet(emojiSignatureSet, 0x1F5FB, 0x1F5FF);
        addUnicodeToSet(emojiSignatureSet, 0x1F600);
        addUnicodeToSet(emojiSignatureSet, 0x1F601, 0x1F610);
        addUnicodeToSet(emojiSignatureSet, 0x1F611);
        addUnicodeToSet(emojiSignatureSet, 0x1F612, 0x1F614);
        addUnicodeToSet(emojiSignatureSet, 0x1F615);
        addUnicodeToSet(emojiSignatureSet, 0x1F616);
        addUnicodeToSet(emojiSignatureSet, 0x1F617);
        addUnicodeToSet(emojiSignatureSet, 0x1F618);
        addUnicodeToSet(emojiSignatureSet, 0x1F619);
        addUnicodeToSet(emojiSignatureSet, 0x1F61A);
        addUnicodeToSet(emojiSignatureSet, 0x1F61B);
        addUnicodeToSet(emojiSignatureSet, 0x1F61C, 0x1F61E);
        addUnicodeToSet(emojiSignatureSet, 0x1F61F);
        addUnicodeToSet(emojiSignatureSet, 0x1F620, 0x1F625);
        addUnicodeToSet(emojiSignatureSet, 0x1F626, 0x1F627);
        addUnicodeToSet(emojiSignatureSet, 0x1F628, 0x1F62B);
        addUnicodeToSet(emojiSignatureSet, 0x1F62C);
        addUnicodeToSet(emojiSignatureSet, 0x1F62D);
        addUnicodeToSet(emojiSignatureSet, 0x1F62E, 0x1F62F);
        addUnicodeToSet(emojiSignatureSet, 0x1F630, 0x1F633);
        addUnicodeToSet(emojiSignatureSet, 0x1F634);
        addUnicodeToSet(emojiSignatureSet, 0x1F635, 0x1F640);
        addUnicodeToSet(emojiSignatureSet, 0x1F641, 0x1F642);
        addUnicodeToSet(emojiSignatureSet, 0x1F643, 0x1F644);
        addUnicodeToSet(emojiSignatureSet, 0x1F645, 0x1F64F);
        addUnicodeToSet(emojiSignatureSet, 0x1F680, 0x1F6C5);
        addUnicodeToSet(emojiSignatureSet, 0x1F6CC);
        addUnicodeToSet(emojiSignatureSet, 0x1F6D0);
        addUnicodeToSet(emojiSignatureSet, 0x1F6D1, 0x1F6D2);
        addUnicodeToSet(emojiSignatureSet, 0x1F6EB, 0x1F6EC);
        addUnicodeToSet(emojiSignatureSet, 0x1F6F4, 0x1F6F6);
        addUnicodeToSet(emojiSignatureSet, 0x1F6F7, 0x1F6F8);
        addUnicodeToSet(emojiSignatureSet, 0x1F6F9);
        addUnicodeToSet(emojiSignatureSet, 0x1F910, 0x1F918);
        addUnicodeToSet(emojiSignatureSet, 0x1F919, 0x1F91E);
        addUnicodeToSet(emojiSignatureSet, 0x1F91F);
        addUnicodeToSet(emojiSignatureSet, 0x1F920, 0x1F927);
        addUnicodeToSet(emojiSignatureSet, 0x1F928, 0x1F92F);
        addUnicodeToSet(emojiSignatureSet, 0x1F930);
        addUnicodeToSet(emojiSignatureSet, 0x1F931, 0x1F932);
        addUnicodeToSet(emojiSignatureSet, 0x1F933, 0x1F93A);
        addUnicodeToSet(emojiSignatureSet, 0x1F93C, 0x1F93E);
        addUnicodeToSet(emojiSignatureSet, 0x1F940, 0x1F945);
        addUnicodeToSet(emojiSignatureSet, 0x1F947, 0x1F94B);
        addUnicodeToSet(emojiSignatureSet, 0x1F94C);
        addUnicodeToSet(emojiSignatureSet, 0x1F94D, 0x1F94F);
        addUnicodeToSet(emojiSignatureSet, 0x1F950, 0x1F95E);
        addUnicodeToSet(emojiSignatureSet, 0x1F95F, 0x1F96B);
        addUnicodeToSet(emojiSignatureSet, 0x1F96C, 0x1F970);
        addUnicodeToSet(emojiSignatureSet, 0x1F973, 0x1F976);
        addUnicodeToSet(emojiSignatureSet, 0x1F97A);
        addUnicodeToSet(emojiSignatureSet, 0x1F97C, 0x1F97F);
        addUnicodeToSet(emojiSignatureSet, 0x1F980, 0x1F984);
        addUnicodeToSet(emojiSignatureSet, 0x1F985, 0x1F991);
        addUnicodeToSet(emojiSignatureSet, 0x1F992, 0x1F997);
        addUnicodeToSet(emojiSignatureSet, 0x1F998, 0x1F9A2);
        addUnicodeToSet(emojiSignatureSet, 0x1F9B0, 0x1F9B9);
        addUnicodeToSet(emojiSignatureSet, 0x1F9C0);
        addUnicodeToSet(emojiSignatureSet, 0x1F9C1, 0x1F9C2);
        addUnicodeToSet(emojiSignatureSet, 0x1F9D0, 0x1F9E6);
        addUnicodeToSet(emojiSignatureSet, 0x1F9E7, 0x1F9FF);
        addUnicodeToSet(emojiSignatureSet, 0x1F3FB, 0x1F3FF);
        addUnicodeToSet(emojiSignatureSet, 0x261D);
        addUnicodeToSet(emojiSignatureSet, 0x26F9);
        addUnicodeToSet(emojiSignatureSet, 0x270A, 0x270B);
        addUnicodeToSet(emojiSignatureSet, 0x270C, 0x270D);
        addUnicodeToSet(emojiSignatureSet, 0x1F385);
        addUnicodeToSet(emojiSignatureSet, 0x1F3C2, 0x1F3C4);
        addUnicodeToSet(emojiSignatureSet, 0x1F3C7);
        addUnicodeToSet(emojiSignatureSet, 0x1F3CA);
        addUnicodeToSet(emojiSignatureSet, 0x1F3CB, 0x1F3CC);
        addUnicodeToSet(emojiSignatureSet, 0x1F442, 0x1F443);
        addUnicodeToSet(emojiSignatureSet, 0x1F446, 0x1F450);
        addUnicodeToSet(emojiSignatureSet, 0x1F466, 0x1F469);
        addUnicodeToSet(emojiSignatureSet, 0x1F46E);
        addUnicodeToSet(emojiSignatureSet, 0x1F470, 0x1F478);
        addUnicodeToSet(emojiSignatureSet, 0x1F47C);
        addUnicodeToSet(emojiSignatureSet, 0x1F481, 0x1F483);
        addUnicodeToSet(emojiSignatureSet, 0x1F485, 0x1F487);
        addUnicodeToSet(emojiSignatureSet, 0x1F4AA);
        addUnicodeToSet(emojiSignatureSet, 0x1F574, 0x1F575);
        addUnicodeToSet(emojiSignatureSet, 0x1F57A);
        addUnicodeToSet(emojiSignatureSet, 0x1F590);
        addUnicodeToSet(emojiSignatureSet, 0x1F595, 0x1F596);
        addUnicodeToSet(emojiSignatureSet, 0x1F645, 0x1F647);
        addUnicodeToSet(emojiSignatureSet, 0x1F64B, 0x1F64F);
        addUnicodeToSet(emojiSignatureSet, 0x1F6A3);
        addUnicodeToSet(emojiSignatureSet, 0x1F6B4, 0x1F6B6);
        addUnicodeToSet(emojiSignatureSet, 0x1F6C0);
        addUnicodeToSet(emojiSignatureSet, 0x1F6CC);
        addUnicodeToSet(emojiSignatureSet, 0x1F918);
        addUnicodeToSet(emojiSignatureSet, 0x1F919, 0x1F91C);
        addUnicodeToSet(emojiSignatureSet, 0x1F91E);
        addUnicodeToSet(emojiSignatureSet, 0x1F91F);
        addUnicodeToSet(emojiSignatureSet, 0x1F926);
        addUnicodeToSet(emojiSignatureSet, 0x1F930);
        addUnicodeToSet(emojiSignatureSet, 0x1F931, 0x1F932);
        addUnicodeToSet(emojiSignatureSet, 0x1F933, 0x1F939);
        addUnicodeToSet(emojiSignatureSet, 0x1F93D, 0x1F93E);
        addUnicodeToSet(emojiSignatureSet, 0x1F9B5, 0x1F9B6);
        addUnicodeToSet(emojiSignatureSet, 0x1F9B8, 0x1F9B9);
        addUnicodeToSet(emojiSignatureSet, 0x1F9D1, 0x1F9DD);

        // 0023          ; Emoji_Component      #  1.1  [1] (#️)       number sign
        // 002A          ; Emoji_Component      #  1.1  [1] (*️)       asterisk
        // 0030..0039    ; Emoji_Component      #  1.1 [10] (0️..9️)    digit zero..digit nine
        // 上面说了这 #*0~9都是正常字符，只是和 20E3 FE0F组合起来变成了 Emoji， 所以把以下两个加入特征码
        // 20E3          ; Emoji_Component      #  3.0  [1] (⃣)       combining enclosing keycap
        // FE0F          ; Emoji_Component      #  3.2  [1] ()        VARIATION SELECTOR-16
        addUnicodeToSet(emojiSignatureSet, 0x20E3);
        addUnicodeToSet(emojiSignatureSet, 0XFE0F);
        // 与之相似的 VARIATION SELECTOR-15
        addUnicodeToSet(emojiSignatureSet, 0XFE0E);

        // 200D          ; Emoji_Component      #  1.1  [1] (‍)        zero width joiner
        // 这个是将多个Emoji拼接成一个Emoji，在支持的机器上可以显示。zwj
        addUnicodeToSet(emojiSignatureSet, 0x200D);

        // 1F1E6..1F1FF  ; Emoji                #  6.0 [26] (🇦..🇿)
        // regional indicator symbol letter a..regional indicator symbol letter z
        // A..Z 之间相互组合成 AC 等
        // 1F1E6 1F1E8   ; Emoji_Flag_Sequence       ; Ascension Island #  6.0  [1] (🇦🇨)
        addUnicodeToSet(emojiSignatureSet, 0x1F1E6, 0x1F1FF);
        // 1F3FB..1F3FF  ; Emoji_Modifier       #  8.0  [5] (🏻..🏿)    light skin tone..dark skin tone
        // 用来表示肤色，这里是重复添加，第一部分已经添加过了
        addUnicodeToSet(emojiSignatureSet, 0x1F3FB, 0x1F3FF);
        // 1F9B0..1F9B3  ; Emoji_Component      # 11.0  [4] (🦰..🦳)    red-haired..white-haired
        // 表示发色， 这里也是重复添加了，第一部分已经添加过了
        addUnicodeToSet(emojiSignatureSet, 0x1F9B0, 0x1F9B3);
        // E0020..E007F  ; Emoji_Component      #  3.1 [96] (󠀠..󠁿)      tag space..cancel tag
        // 根据 https://en.wikipedia.org/wiki/Tags_(Unicode_block)
        // http://unicode.org/Public/emoji/5.0/emoji-sequences.txt
        // E0020 ~ E007F 的使用仅有
        // # Emoji Tag Sequence: See Annex C of TR51 for more information.
        // 1F3F4 E0067 E0062 E0065 E006E E0067 E007F; Emoji_Tag_Sequence; England  #  7.0  [1] (🏴󠁧󠁢󠁥󠁮󠁧󠁿)
        // 1F3F4 E0067 E0062 E0073 E0063 E0074 E007F; Emoji_Tag_Sequence; Scotland  #  7.0  [1] (🏴󠁧󠁢󠁳󠁣󠁴󠁿)
        // 1F3F4 E0067 E0062 E0077 E006C E0073 E007F; Emoji_Tag_Sequence; Wales  #  7.0  [1] (🏴󠁧󠁢󠁷󠁬󠁳󠁿)
        // 1F3F4 已经是特征Emoji了，所以就不再添加，且这些覆盖了a~z及一些符号，如果添加了反而会误判
        // 或者仅添加 E007F 作为 Emoji 特征码
        addUnicodeToSet(emojiSignatureSet, 0xE007F);

        addUnicodeToSet(emojiSignatureSet, 0x00A9);
        addUnicodeToSet(emojiSignatureSet, 0x00AE);
        addUnicodeToSet(emojiSignatureSet, 0x203C);
        addUnicodeToSet(emojiSignatureSet, 0x2049);
        addUnicodeToSet(emojiSignatureSet, 0x2122);
        addUnicodeToSet(emojiSignatureSet, 0x2139);
        addUnicodeToSet(emojiSignatureSet, 0x2194, 0x2199);
        addUnicodeToSet(emojiSignatureSet, 0x21A9, 0x21AA);
        addUnicodeToSet(emojiSignatureSet, 0x231A, 0x231B);
        addUnicodeToSet(emojiSignatureSet, 0x2328);
        addUnicodeToSet(emojiSignatureSet, 0x2388);
        addUnicodeToSet(emojiSignatureSet, 0x23CF);
        addUnicodeToSet(emojiSignatureSet, 0x23E9, 0x23F3);
        addUnicodeToSet(emojiSignatureSet, 0x23F8, 0x23FA);
        addUnicodeToSet(emojiSignatureSet, 0x24C2);
        addUnicodeToSet(emojiSignatureSet, 0x25AA, 0x25AB);
        addUnicodeToSet(emojiSignatureSet, 0x25B6);
        addUnicodeToSet(emojiSignatureSet, 0x25C0);
        addUnicodeToSet(emojiSignatureSet, 0x25FB, 0x25FE);
        addUnicodeToSet(emojiSignatureSet, 0x2600, 0x2605);
        addUnicodeToSet(emojiSignatureSet, 0x2607, 0x2612);
        addUnicodeToSet(emojiSignatureSet, 0x2614, 0x2615);
        addUnicodeToSet(emojiSignatureSet, 0x2616, 0x2617);
        addUnicodeToSet(emojiSignatureSet, 0x2618);
        addUnicodeToSet(emojiSignatureSet, 0x2619);
        addUnicodeToSet(emojiSignatureSet, 0x261A, 0x266F);
        addUnicodeToSet(emojiSignatureSet, 0x2670, 0x2671);
        addUnicodeToSet(emojiSignatureSet, 0x2672, 0x267D);
        addUnicodeToSet(emojiSignatureSet, 0x267E, 0x267F);
        addUnicodeToSet(emojiSignatureSet, 0x2680, 0x2685);
        addUnicodeToSet(emojiSignatureSet, 0x2690, 0x2691);
        addUnicodeToSet(emojiSignatureSet, 0x2692, 0x269C);
        addUnicodeToSet(emojiSignatureSet, 0x269D);
        addUnicodeToSet(emojiSignatureSet, 0x269E, 0x269F);
        addUnicodeToSet(emojiSignatureSet, 0x26A0, 0x26A1);
        addUnicodeToSet(emojiSignatureSet, 0x26A2, 0x26B1);
        addUnicodeToSet(emojiSignatureSet, 0x26B2);
        addUnicodeToSet(emojiSignatureSet, 0x26B3, 0x26BC);
        addUnicodeToSet(emojiSignatureSet, 0x26BD, 0x26BF);
        addUnicodeToSet(emojiSignatureSet, 0x26C0, 0x26C3);
        addUnicodeToSet(emojiSignatureSet, 0x26C4, 0x26CD);
        addUnicodeToSet(emojiSignatureSet, 0x26CE);
        addUnicodeToSet(emojiSignatureSet, 0x26CF, 0x26E1);
        addUnicodeToSet(emojiSignatureSet, 0x26E2);
        addUnicodeToSet(emojiSignatureSet, 0x26E3);
        addUnicodeToSet(emojiSignatureSet, 0x26E4, 0x26E7);
        addUnicodeToSet(emojiSignatureSet, 0x26E8, 0x26FF);
        addUnicodeToSet(emojiSignatureSet, 0x2700);
        addUnicodeToSet(emojiSignatureSet, 0x2701, 0x2704);
        addUnicodeToSet(emojiSignatureSet, 0x2705);
        addUnicodeToSet(emojiSignatureSet, 0x2708, 0x2709);
        addUnicodeToSet(emojiSignatureSet, 0x270A, 0x270B);
        addUnicodeToSet(emojiSignatureSet, 0x270C, 0x2712);
        addUnicodeToSet(emojiSignatureSet, 0x2714);
        addUnicodeToSet(emojiSignatureSet, 0x2716);
        addUnicodeToSet(emojiSignatureSet, 0x271D);
        addUnicodeToSet(emojiSignatureSet, 0x2721);
        addUnicodeToSet(emojiSignatureSet, 0x2728);
        addUnicodeToSet(emojiSignatureSet, 0x2733, 0x2734);
        addUnicodeToSet(emojiSignatureSet, 0x2744);
        addUnicodeToSet(emojiSignatureSet, 0x2747);
        addUnicodeToSet(emojiSignatureSet, 0x274C);
        addUnicodeToSet(emojiSignatureSet, 0x274E);
        addUnicodeToSet(emojiSignatureSet, 0x2753, 0x2755);
        addUnicodeToSet(emojiSignatureSet, 0x2757);
        addUnicodeToSet(emojiSignatureSet, 0x2763, 0x2767);
        addUnicodeToSet(emojiSignatureSet, 0x2795, 0x2797);
        addUnicodeToSet(emojiSignatureSet, 0x27A1);
        addUnicodeToSet(emojiSignatureSet, 0x27B0);
        addUnicodeToSet(emojiSignatureSet, 0x27BF);
        addUnicodeToSet(emojiSignatureSet, 0x2934, 0x2935);
        addUnicodeToSet(emojiSignatureSet, 0x2B05, 0x2B07);
        addUnicodeToSet(emojiSignatureSet, 0x2B1B, 0x2B1C);
        addUnicodeToSet(emojiSignatureSet, 0x2B50);
        addUnicodeToSet(emojiSignatureSet, 0x2B55);
        addUnicodeToSet(emojiSignatureSet, 0x3030);
        addUnicodeToSet(emojiSignatureSet, 0x303D);
        addUnicodeToSet(emojiSignatureSet, 0x3297);
        addUnicodeToSet(emojiSignatureSet, 0x3299);
        addUnicodeToSet(emojiSignatureSet, 0x1F000, 0x1F02B);
        // 1F02C..1F02F  ; Extended_Pictographic#   NA  [4] (🀬️..🀯️)    <reserved-1F02C>..<reserved-1F02F>
        // addUnicodeToSet(emojiSignatureSet, 0x1F02C, 0x1F02F);
        addUnicodeToSet(emojiSignatureSet, 0x1F030, 0x1F093);
        // addUnicodeToSet(emojiSignatureSet, 0x1F094, 0x1F09F);
        addUnicodeToSet(emojiSignatureSet, 0x1F0A0, 0x1F0AE);
        // addUnicodeToSet(emojiSignatureSet, 0x1F0AF, 0x1F0B0);
        addUnicodeToSet(emojiSignatureSet, 0x1F0B1, 0x1F0BE);
        addUnicodeToSet(emojiSignatureSet, 0x1F0BF);
        // addUnicodeToSet(emojiSignatureSet, 0x1F0C0);
        addUnicodeToSet(emojiSignatureSet, 0x1F0C1, 0x1F0CF);
        // addUnicodeToSet(emojiSignatureSet, 0x1F0D0);
        addUnicodeToSet(emojiSignatureSet, 0x1F0D1, 0x1F0DF);
        addUnicodeToSet(emojiSignatureSet, 0x1F0E0, 0x1F0F5);
        // addUnicodeToSet(emojiSignatureSet, 0x1F0F6, 0x1F0FF);
        // addUnicodeToSet(emojiSignatureSet, 0x1F10D, 0x1F10F);
        addUnicodeToSet(emojiSignatureSet, 0x1F12F);
        // addUnicodeToSet(emojiSignatureSet, 0x1F16C, 0x1F16F);
        addUnicodeToSet(emojiSignatureSet, 0x1F170, 0x1F171);
        addUnicodeToSet(emojiSignatureSet, 0x1F17E);
        addUnicodeToSet(emojiSignatureSet, 0x1F17F);
        addUnicodeToSet(emojiSignatureSet, 0x1F18E);
        addUnicodeToSet(emojiSignatureSet, 0x1F191, 0x1F19A);
        // addUnicodeToSet(emojiSignatureSet, 0x1F1AD, 0x1F1E5);
        addUnicodeToSet(emojiSignatureSet, 0x1F201, 0x1F202);
        // addUnicodeToSet(emojiSignatureSet, 0x1F203, 0x1F20F);
        addUnicodeToSet(emojiSignatureSet, 0x1F21A);
        addUnicodeToSet(emojiSignatureSet, 0x1F22F);
        addUnicodeToSet(emojiSignatureSet, 0x1F232, 0x1F23A);
        // addUnicodeToSet(emojiSignatureSet, 0x1F23C, 0x1F23F);
        // addUnicodeToSet(emojiSignatureSet, 0x1F249, 0x1F24F);
        addUnicodeToSet(emojiSignatureSet, 0x1F250, 0x1F251);
        // addUnicodeToSet(emojiSignatureSet, 0x1F252, 0x1F25F);
        addUnicodeToSet(emojiSignatureSet, 0x1F260, 0x1F265);
        // addUnicodeToSet(emojiSignatureSet, 0x1F266, 0x1F2FF);
        addUnicodeToSet(emojiSignatureSet, 0x1F300, 0x1F320);
        addUnicodeToSet(emojiSignatureSet, 0x1F321, 0x1F32C);
        addUnicodeToSet(emojiSignatureSet, 0x1F32D, 0x1F32F);
        addUnicodeToSet(emojiSignatureSet, 0x1F330, 0x1F335);
        addUnicodeToSet(emojiSignatureSet, 0x1F336);
        addUnicodeToSet(emojiSignatureSet, 0x1F337, 0x1F37C);
        addUnicodeToSet(emojiSignatureSet, 0x1F37D);
        addUnicodeToSet(emojiSignatureSet, 0x1F37E, 0x1F37F);
        addUnicodeToSet(emojiSignatureSet, 0x1F380, 0x1F393);
        addUnicodeToSet(emojiSignatureSet, 0x1F394, 0x1F39F);
        addUnicodeToSet(emojiSignatureSet, 0x1F3A0, 0x1F3C4);
        addUnicodeToSet(emojiSignatureSet, 0x1F3C5);
        addUnicodeToSet(emojiSignatureSet, 0x1F3C6, 0x1F3CA);
        addUnicodeToSet(emojiSignatureSet, 0x1F3CB, 0x1F3CE);
        addUnicodeToSet(emojiSignatureSet, 0x1F3CF, 0x1F3D3);
        addUnicodeToSet(emojiSignatureSet, 0x1F3D4, 0x1F3DF);
        addUnicodeToSet(emojiSignatureSet, 0x1F3E0, 0x1F3F0);
        addUnicodeToSet(emojiSignatureSet, 0x1F3F1, 0x1F3F7);
        addUnicodeToSet(emojiSignatureSet, 0x1F3F8, 0x1F3FA);
        addUnicodeToSet(emojiSignatureSet, 0x1F400, 0x1F43E);
        addUnicodeToSet(emojiSignatureSet, 0x1F43F);
        addUnicodeToSet(emojiSignatureSet, 0x1F440);
        addUnicodeToSet(emojiSignatureSet, 0x1F441);
        addUnicodeToSet(emojiSignatureSet, 0x1F442, 0x1F4F7);
        addUnicodeToSet(emojiSignatureSet, 0x1F4F8);
        addUnicodeToSet(emojiSignatureSet, 0x1F4F9, 0x1F4FC);
        addUnicodeToSet(emojiSignatureSet, 0x1F4FD, 0x1F4FE);
        addUnicodeToSet(emojiSignatureSet, 0x1F4FF);
        addUnicodeToSet(emojiSignatureSet, 0x1F500, 0x1F53D);
        addUnicodeToSet(emojiSignatureSet, 0x1F546, 0x1F54A);
        addUnicodeToSet(emojiSignatureSet, 0x1F54B, 0x1F54F);
        addUnicodeToSet(emojiSignatureSet, 0x1F550, 0x1F567);
        addUnicodeToSet(emojiSignatureSet, 0x1F568, 0x1F579);
        addUnicodeToSet(emojiSignatureSet, 0x1F57A);
        addUnicodeToSet(emojiSignatureSet, 0x1F57B, 0x1F5A3);
        addUnicodeToSet(emojiSignatureSet, 0x1F5A4);
        addUnicodeToSet(emojiSignatureSet, 0x1F5A5, 0x1F5FA);
        addUnicodeToSet(emojiSignatureSet, 0x1F5FB, 0x1F5FF);
        addUnicodeToSet(emojiSignatureSet, 0x1F600);
        addUnicodeToSet(emojiSignatureSet, 0x1F601, 0x1F610);
        addUnicodeToSet(emojiSignatureSet, 0x1F611);
        addUnicodeToSet(emojiSignatureSet, 0x1F612, 0x1F614);
        addUnicodeToSet(emojiSignatureSet, 0x1F615);
        addUnicodeToSet(emojiSignatureSet, 0x1F616);
        addUnicodeToSet(emojiSignatureSet, 0x1F617);
        addUnicodeToSet(emojiSignatureSet, 0x1F618);
        addUnicodeToSet(emojiSignatureSet, 0x1F619);
        addUnicodeToSet(emojiSignatureSet, 0x1F61A);
        addUnicodeToSet(emojiSignatureSet, 0x1F61B);
        addUnicodeToSet(emojiSignatureSet, 0x1F61C, 0x1F61E);
        addUnicodeToSet(emojiSignatureSet, 0x1F61F);
        addUnicodeToSet(emojiSignatureSet, 0x1F620, 0x1F625);
        addUnicodeToSet(emojiSignatureSet, 0x1F626, 0x1F627);
        addUnicodeToSet(emojiSignatureSet, 0x1F628, 0x1F62B);
        addUnicodeToSet(emojiSignatureSet, 0x1F62C);
        addUnicodeToSet(emojiSignatureSet, 0x1F62D);
        addUnicodeToSet(emojiSignatureSet, 0x1F62E, 0x1F62F);
        addUnicodeToSet(emojiSignatureSet, 0x1F630, 0x1F633);
        addUnicodeToSet(emojiSignatureSet, 0x1F634);
        addUnicodeToSet(emojiSignatureSet, 0x1F635, 0x1F640);
        addUnicodeToSet(emojiSignatureSet, 0x1F641, 0x1F642);
        addUnicodeToSet(emojiSignatureSet, 0x1F643, 0x1F644);
        addUnicodeToSet(emojiSignatureSet, 0x1F645, 0x1F64F);
        addUnicodeToSet(emojiSignatureSet, 0x1F680, 0x1F6C5);
        addUnicodeToSet(emojiSignatureSet, 0x1F6C6, 0x1F6CF);
        addUnicodeToSet(emojiSignatureSet, 0x1F6D0);
        addUnicodeToSet(emojiSignatureSet, 0x1F6D1, 0x1F6D2);
        addUnicodeToSet(emojiSignatureSet, 0x1F6D3, 0x1F6D4);
        // addUnicodeToSet(emojiSignatureSet, 0x1F6D5, 0x1F6DF);
        addUnicodeToSet(emojiSignatureSet, 0x1F6E0, 0x1F6EC);
        // addUnicodeToSet(emojiSignatureSet, 0x1F6ED, 0x1F6EF);
        addUnicodeToSet(emojiSignatureSet, 0x1F6F0, 0x1F6F3);
        addUnicodeToSet(emojiSignatureSet, 0x1F6F4, 0x1F6F6);
        addUnicodeToSet(emojiSignatureSet, 0x1F6F7, 0x1F6F8);
        addUnicodeToSet(emojiSignatureSet, 0x1F6F9);
        // addUnicodeToSet(emojiSignatureSet, 0x1F6FA, 0x1F6FF);
        // addUnicodeToSet(emojiSignatureSet, 0x1F774, 0x1F77F);
        addUnicodeToSet(emojiSignatureSet, 0x1F7D5, 0x1F7D8);
        // addUnicodeToSet(emojiSignatureSet, 0x1F7D9, 0x1F7FF);
        // addUnicodeToSet(emojiSignatureSet, 0x1F80C, 0x1F80F);
        // addUnicodeToSet(emojiSignatureSet, 0x1F848, 0x1F84F);
        // addUnicodeToSet(emojiSignatureSet, 0x1F85A, 0x1F85F);
        // addUnicodeToSet(emojiSignatureSet, 0x1F888, 0x1F88F);
        // addUnicodeToSet(emojiSignatureSet, 0x1F8AE, 0x1F8FF);
        // addUnicodeToSet(emojiSignatureSet, 0x1F90C, 0x1F90F);
        addUnicodeToSet(emojiSignatureSet, 0x1F910, 0x1F918);
        addUnicodeToSet(emojiSignatureSet, 0x1F919, 0x1F91E);
        addUnicodeToSet(emojiSignatureSet, 0x1F91F);
        addUnicodeToSet(emojiSignatureSet, 0x1F920, 0x1F927);
        addUnicodeToSet(emojiSignatureSet, 0x1F928, 0x1F92F);
        addUnicodeToSet(emojiSignatureSet, 0x1F930);
        addUnicodeToSet(emojiSignatureSet, 0x1F931, 0x1F932);
        addUnicodeToSet(emojiSignatureSet, 0x1F933, 0x1F93A);
        addUnicodeToSet(emojiSignatureSet, 0x1F93C, 0x1F93E);
        // addUnicodeToSet(emojiSignatureSet, 0x1F93F);
        addUnicodeToSet(emojiSignatureSet, 0x1F940, 0x1F945);
        addUnicodeToSet(emojiSignatureSet, 0x1F947, 0x1F94B);
        addUnicodeToSet(emojiSignatureSet, 0x1F94C);
        addUnicodeToSet(emojiSignatureSet, 0x1F94D, 0x1F94F);
        addUnicodeToSet(emojiSignatureSet, 0x1F950, 0x1F95E);
        addUnicodeToSet(emojiSignatureSet, 0x1F95F, 0x1F96B);
        addUnicodeToSet(emojiSignatureSet, 0x1F96C, 0x1F970);
        // addUnicodeToSet(emojiSignatureSet, 0x1F971, 0x1F972);
        addUnicodeToSet(emojiSignatureSet, 0x1F973, 0x1F976);
        // addUnicodeToSet(emojiSignatureSet, 0x1F977, 0x1F979);
        addUnicodeToSet(emojiSignatureSet, 0x1F97A);
        // addUnicodeToSet(emojiSignatureSet, 0x1F97B);
        addUnicodeToSet(emojiSignatureSet, 0x1F97C, 0x1F97F);
        addUnicodeToSet(emojiSignatureSet, 0x1F980, 0x1F984);
        addUnicodeToSet(emojiSignatureSet, 0x1F985, 0x1F991);
        addUnicodeToSet(emojiSignatureSet, 0x1F992, 0x1F997);
        addUnicodeToSet(emojiSignatureSet, 0x1F998, 0x1F9A2);
        // addUnicodeToSet(emojiSignatureSet, 0x1F9A3, 0x1F9AF);
        addUnicodeToSet(emojiSignatureSet, 0x1F9B0, 0x1F9B9);
        // addUnicodeToSet(emojiSignatureSet, 0x1F9BA, 0x1F9BF);
        addUnicodeToSet(emojiSignatureSet, 0x1F9C0);
        addUnicodeToSet(emojiSignatureSet, 0x1F9C1, 0x1F9C2);
        // addUnicodeToSet(emojiSignatureSet, 0x1F9C3, 0x1F9CF);
        addUnicodeToSet(emojiSignatureSet, 0x1F9D0, 0x1F9E6);
        addUnicodeToSet(emojiSignatureSet, 0x1F9E7, 0x1F9FF);
        // addUnicodeToSet(emojiSignatureSet, 0x1FA00, 0x1FA5F);
        addUnicodeToSet(emojiSignatureSet, 0x1FA60, 0x1FA6D);
        // addUnicodeToSet(emojiSignatureSet, 0x1FA6E, 0x1FFFD);
        Log.d(TAG, "initSignatureSet end: " + emojiSignatureSet.size());
    }
}
```
