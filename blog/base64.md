# Base64编码
> 这是一种网络上最常见的用于传输8bit字节码的编码方式之一，Base64就是一种基于64个可打印字符来表示二进制数据的方法。摘自百度百科

[TOC]
# 原理
`Base64`的原理特别简单，首先有一张含有64个元素的表格。如下表：

|  下标 | 对应字符  |  下标 | 对应字符  |   下标 | 对应字符  |
| ------------ | ------------ | ------------ | ------------ | ------------ | ------------ |
| 0  | A  | 26 | a | 52 | 0 |
| 1  | B  | 27 | b | ... | ... |
| ... | ... | ... | ... | 61 | 9 |
| 24 | Y | 50 | y | 62 | + |
| 25 | Z | 51 | z | 63 | / |

紧接着，对于给定的字符（8bit），将`8bit`拆分为`6bit`。
假设字符`'A'`, `'B'`, `'C'`，`8bit`二进制如下:
- A => 01000001
- B => 01000010
- C => 01000011

拆分过程如下：
![拆分过程](http://aclrp.com/images/blog/3c8TGVMYNHN4a5ETfg3h5iz4az5gRU787DPKQYtK.png "拆分过程")
可以看出并没有新增任何位，也没有删除。把8bit排列好，从左到右，每6bit取为一个新的数字（作为下标），上图的下标依次是：`16`, `20`, `9`, `3`， 即为 `QUJD`。这就是Base64的编码过程。
# 用途
表面上看把`ABC`变成`QUJD`，显得毫无作用，编码解码还有浪费时间，编码后占用空间还变大了。我们知道`ASCII`的并不完全是可见字符，还有很多的不可见或者是控制字符，当我们传输文本的时候，问题不大，因为很多控制字符是不会，也不应该出现在文本中，但是传输音视频、多媒体的时候，就不一样了。
如果传输过程中，遇到的路由器对信息中控制字符进行了解释，可能会带来意想不到的效果。
所以Base64可以解决传输不可见字符带来的不方便。
# 算法
通过对原理里面的图的观察可以看出，算法也是很简单的，我们只需要使用位运算即可。
现在假设有3个`8bit`字符。
1. 把第一个字符右移2位即可
2. 把第一个字符除最后2位外，全部清零，并左移4位；将第二个字符右移4位；将两个结果做与运算。
3. 把第二个字符左移4位；将第三个字符右移6位；将两个结果做与运算
4. 把第三个字符除后6位外全部清零；

# 特殊情况
上述算法假设是3个字符，但是如果不是呢？
标准Base64规定，当字符个数不是3的倍数时，再原串后面补充1到2的全零`8Bit`。并且编码出来的`000000`为`=`而不是`A`。
例如字符`A` 结果为`QQ==`

![A字符编码过程](http://aclrp.com/images/blog/9UaryLXOWjWo3hm5esIDyzV8G47cKU7spanbdsJj.png "A字符编码过程")

例如字符`AB`, 结果为`QUI=`
![AB编码过程](http://aclrp.com/images/blog/9iXjkksokjUZ0xFgj32AmmiShiecDmTV9AxcpZSC.png "AB编码过程")

所以带有`=`的编码再解码的时候，就能区分到底是额外添加的字节 还是原始的`0000 0000`字节
# 代码
```cpp
char tables[] = {
    'A', 'B', 'C', 'D', 'E', 'F', 'G',
    'H', 'I', 'J', 'K', 'L', 'M', 'N',
    'O', 'P', 'Q', 'R', 'S', 'T',
    'U', 'V', 'W', 'X', 'Y', 'Z',
    'a', 'b', 'c', 'd', 'e', 'f', 'g',
    'h', 'i', 'j', 'k', 'l', 'm', 'n',
    'o', 'p', 'q', 'r', 's', 't',
    'u', 'v', 'w', 'x', 'y', 'z',
    '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
    '+', '/'
};
char src[100];
char des[200];

int main()
{
    cin >> src;
    int len1 = strlen(src);
    int len = len1;

    if(len1 % 3 == 1)
        len += 2;
    else if(len1 % 3 == 2)
        len += 1;

    int k = 0;
    for(int i = 0; i < len; i += 3)
    {
        int a = src[i], b = src[i + 1], c = src[i + 2];
        des[k++] = tables[a >> 2];
        des[k++] = tables[((a & 3) << 4) | (b >> 4)];
        des[k++] = tables[((b & 15) << 2) | (c >> 6)];
        des[k++] = tables[c & 63];
    }
    if(len1 % 3 == 1)
        des[k - 1] = '=', des[k - 2] = '=';
    else if(len1 % 3 == 2)
        des[k - 1] = '=';
    des[k] = '\0';
    cout << des << endl;
    return 0;
}
```

这份代码基本做法是，首先填空原始字符使得它为3的倍数，并且记录原始字符串长度，方便`=`的添加。由于这是控制台输入的字符串，且`src`和`des`已经提前被初始化为`\0`，所以仅仅简单的调整了字符串长度，即可添加空字节。

-----------------
循环过程中每3个字符一组，执行原理中描述的操作，并且在映射表找到对应的字符，放入`des`中，最后判断当前字符串末尾产生的`A`是否是添加的空字节，并转换为`=`.至此算法结束。
# 解码
Base64的解码和编码类似，这里就不在继续说了，需要注意的是，Base64是一种编码算法，不能用于加密。
