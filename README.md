在这之前首先感谢我的朋友：[netero](https://github.com/0x54164)，是他给了我很多帮助完成了这份代码。

在我们编写代码的时候会经常对字符串进行处理。比如：获取字符串长度，获取某个索引所在的字符。


```CS
string strText = "abc";
Console.WriteLine(strText.Length) // output is: 3

//但是...当有一些特殊字符的时候...比如emoji表情的时候。。

string strText = "👩‍🦰👩‍👩‍👦‍👦🏳️‍🌈";
Console.WriteLine(strText.Length) // output is: 22
```

可是我们希望的结果是3，但是结果却输出了22。。这是为什么？

# 字素簇

注意这里是[字素]而不是[字数]

字素簇 (grapheme cluster) 指人直觉和认知上所认为是单个字符的文本元素。字素簇可能就是一个抽象字，也可能由多个抽象字组成。字素簇应当是文本操作的基本单位。

之所以会出现这样的情况是：在众多的编译器，或者内存中。字符都是以Unicode编码的。所以在统计长度的时候是统计的Unicode编码个数
众所周知，一个Unicode是两字节，即便全部用来做字符编码区间也仅仅是0x0000-0xFFFF也就是65536个字符，这个区间可能中国的汉字都装不下。

# 编码区间

所以Unicoe组织想到了一个办法，那就是代理。Unicode组织并不打算把0x0000-0xFFFF全部作为字符区间

所以这个时候Unicode组织决定拿出2048个字符区间作为代理字符。

分别是 0xD800-0xDBFF 是高代理字符。。0xDC00-0xDFFF 是低代理字符。

高代理字符之后通常跟随低代理字符，它们的编码分别取出最后10位组合后再加0x10000成新的编码，这样就可以有更多的字符组合，多达1048576种。

所以这样的字符需要两个Unicode字符组成。

```CS
private static int GetCodePoint(string strText, int nIndex) {
    if (!char.IsHighSurrogate(strText, nIndex)) {
        return strText[nIndex];
    }
    if (nIndex + 1 >= strText.Length) {
        return 0;
    }
    return ((strText[nIndex] & 0x03FF) << 10) + (strText[nIndex + 1] & 0x03FF) + 0x10000;
}
```

上面说到【高代理】之后就是【低代理】，那么一个字符最多就是两个Unicode也就是四字节？

不不不。。。并不是这么算的。。因为不同区间的字符编码有着不同的特性。。而Unicode根据这些特性来确定字素簇。

以最常见的字符举例，比如：【\r\n】

在大对数的编程语言中认为这里是两个字符。。是的。。。他确实是两个字符

但是对于人类感官来说无论是【\r\n】或者【\n】他都是一个字符，那就是【换行】

所以【\r\n】在人类的意识中是一个字符，而不是两个。

如果不这么做那么就会出现这样的情况

```CS
string strA = "A\r\nB";
var strB =  strA.Reverse(); // "B\n\rA";
```

这显然是我们不希望的结果。我们希望的结果是"B\r\nA"
而Unicode也确实是这样定义的[GB3]:

[https://www.unicode.org/reports/tr29/#GB3](https://www.unicode.org/reports/tr29/#GB3)

```
Do not break between a CR and LF. Otherwise, break before and after controls.
GB3                     CR   ×   LF
GB4    (Control | CR | LF)   ÷ 	 
GB5                          ÷   (Control | CR | LF)
```

字符还有组合属性，比如：[ā]

看上去是一个字符，其实是两个字符的组合[a + ̄ = ā] -> "a\u0304"

在Unicode中是这样定义0x0300-0x036F区间的

```
0300..036F    ; Extend # Mn [112] COMBINING GRAVE ACCENT..COMBINING LATIN SMALL LETTER X
```

也就是说"\u0304"具有[Extend]特性，而关于[Extend]在分割规则中这样定义：

```
Do not break before extending characters or ZWJ.
GB9                          ×    (Extend | ZWJ) 
```

Unicode定义了很多特性，而用来确定分割的特性有：

```
CR, LF, Control, L, V, LV, LVT, T, Extend, ZWJ, SpacingMark, Prepend, Extended_Pictographic, RI
```

这些特性分布区间也是Unicode定义的：

[https://www.unicode.org/Public/14.0.0/ucd/auxiliary/GraphemeBreakProperty.txt](https://www.unicode.org/Public/14.0.0/ucd/auxiliary/GraphemeBreakProperty.txt)

而决定这些字符是否应当组合在一起的标准在这里:

[https://www.unicode.org/reports/tr29/#Grapheme_Cluster_Boundary_Rules](https://www.unicode.org/reports/tr29/#Grapheme_Cluster_Boundary_Rules)

而此代码全部按照Unicode最新标准编写，就算以后Unicode更新，代码中也提供了代码生成函数，可以根据Unicode最新标准生成最新的代码 比如：

```CS
/// <summary>
/// Build the [GetGraphemeBreakProperty] function and [m_lst_code_range]
/// Current [GetGraphemeBreakProperty] and [m_lst_code_range] create by:
/// https://www.unicode.org/Public/14.0.0/ucd/auxiliary/GraphemeBreakProperty.txt
/// https://www.unicode.org/Public/14.0.0/ucd/emoji/emoji-data.txt
/// [Extended_Pictographic] type was not in [GraphemeBreakProperty.txt(14.0.0)]
/// So append [emoji-data.txt] to [GraphemeBreakProperty.txt] to create code
/// </summary>
/// <param name="strText">The text of [GraphemeBreakProperty.txt]</param>
/// <returns>Code</returns>
public static string CreateBreakPropertyCodeFromText(string strText);
```

# Demo

```CS
string strText = "汉字👩‍🦰👩‍👩‍👦‍👦🏳️‍🌈Abc";
List<string> lst = STGraphemeSplitter.Split(strText);
Console.WriteLine(string.Join(",", lst.ToArray())); //Output: 汉,字,👩‍🦰,👩‍👩‍👦‍👦,🏳️‍🌈,A,b,c

int nLen = STGraphemeSplitter.GetLength(strText);   //仅仅获取长度.

foreach (var v in STGraphemeSplitter.GetEnumerator(strText)) {
    Console.WriteLine(v);
}

STGraphemeSplitter.Each(strText, (str, nStart, nLen) => { //速度最快
    Console.WriteLine(str.Substring(nStart, nLen));
});

//如果上面的速度还不够快？那就在使用之前创建缓存
STGraphemeSplitter.CreateArrayCache();          //创建缓存到数组，速度相对快，占用空间高
STGraphemeSplitter.CreateDictionaryCache();     //创建缓存到字典，速度相对慢，暂用空间少
STGraphemeSplitter.ClearCache();                //清除所有缓存
```


# 关于作者
* Github: [DebugST](https://github.com/DebugST/)
* Blog: [Crystal_lz](http://st233.com)
* Mail: (2212233137@qq.com)
