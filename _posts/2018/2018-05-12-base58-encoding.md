---
layout: post
title:  "Base58 编码"
date:   2018-05-12 16:02:28 +0800
author: mistydew
comments: true
categories: Blockchain Bitcoin
tags: 区块链 比特币 base58编码
---
Base58 是一组二进制文本编码方案，用于把大整数表示为含有字母和数字的文本。
它与 Base64 类似，但被修改过以避免打印输出时看起来有歧义的非字母数字和字母。
因此，被设计用于手动输入数据的人类用户，从一些可视源复制，还允许轻松复制和粘贴，因为双击通常会选择整个字符串。

对比 Base64 省略了以下外观类似的字母：
0（数字零），O（大写字母 o），I（大写字母 i）和 l（小写字母 L）以及非字母数字的字符 +（加号）和 /（斜杠）。
与 Base64 相反，编码数字不与原始数据字节边界对齐。因此，该方法适合编码大整数，但不适合编码较长的二进制数据。
字母表中字母真正的顺序基于应用程序，这就是为什么单独的“Base58”这一术语不足以完整描述格式。
一种变体 Base56，相比 Base58 不包含 1（数字一）和 o（小写字母 O）。

Base58Check 是一种 Base58 编码格式，可在前几个字符中明确编码的数据类型（即版本号），并在最后几个字符中包含一个错误检测代码（即校验和）。

## Base58 符号表

数值 | 字符 | 数值 | 字符 | 数值 | 字符 | 数值 | 字符
-----|------|------|------|------|------|------|------
0    | 1    | 1    | 2    | 2    | 3    | 3    | 4
4    | 5    | 5    | 6    | 6    | 7    | 7    | 8
8    | 9    | 9    | A    | 10   | B    | 11   | C
12   | D    | 13   | E    | 14   | F    | 15   | G
16   | H    | 17   | J    | 18   | K    | 19   | L
20   | M    | 21   | N    | 22   | P    | 23   | Q
24   | R    | 25   | S    | 26   | T    | 27   | U
28   | V    | 29   | W    | 30   | X    | 31   | Y
32   | Z    | 33   | a    | 34   | b    | 35   | c
36   | d    | 37   | e    | 38   | f    | 39   | g
40   | h    | 41   | i    | 42   | j    | 43   | k
44   | m    | 45   | n    | 46   | o    | 47   | p
48   | q    | 49   | r    | 50   | s    | 51   | t
52   | u    | 53   | v    | 54   | w    | 55   | x
56   | y    | 57   | z

## 源码剖析

比特币中的 Base58 编码使用的是 Base58Check，实现如下：

```cpp
/** All alphanumeric characters except for "0", "I", "O", and "l" */
static const char* pszBase58 = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz"; // 10 + 26 * 2 - 4 = 58
...
std::string EncodeBase58(const unsigned char* pbegin, const unsigned char* pend) // Base58 编码
{
    // Skip & count leading zeroes.
    int zeroes = 0; // 1.跳过并统计开头 0 的个数，最多为 1 byte 的 0x00
    while (pbegin != pend && *pbegin == 0) {
        pbegin++;
        zeroes++;
    }
    // Allocate enough space in big-endian base58 representation.
    std::vector<unsigned char> b58((pend - pbegin) * 138 / 100 + 1); // log(256) / log(58), rounded up. // 2.为大端 base58 表示，开辟足够的空间（34 或 35 bytes），向上取整
    // Process the bytes.
    while (pbegin != pend) { // 3.256 进制转 58 进制
        int carry = *pbegin;
        // Apply "b58 = b58 * 256 + ch".
        for (std::vector<unsigned char>::reverse_iterator it = b58.rbegin(); it != b58.rend(); it++) {
            carry += 256 * (*it);
            *it = carry % 58;
            carry /= 58;
        }
        assert(carry == 0);
        pbegin++;
    }
    // Skip leading zeroes in base58 result.
    std::vector<unsigned char>::iterator it = b58.begin();
    while (it != b58.end() && *it == 0) // 4.跳过开头的 0
        it++;
    // Translate the result into a string.
    std::string str;
    str.reserve(zeroes + (b58.end() - it)); // 1 + 33 or 0 + 34 = 34
    str.assign(zeroes, '1'); // 5.在字符串前面的位置指派 zeroes 个字符 1
    while (it != b58.end()) // 6.查 Base58 字符表转换结果为字符串
        str += pszBase58[*(it++)]; // append *it then ++it
    return str; // 前缀为 0x00 不参与 Base58 运算，地址长度固定为 34 bytes 且前缀位 '1'，其他不为 0x00 的前缀，均参与 Base58 运算，地址长度变换范围 33 - 35 bytes
}
```

对 25 bytes 的“前缀（Version） + 公钥（PubKeyID） + 校验和（Checksum）”进行 Base58 编码：
1. 跳过开头的 0 同时统计其个数。zeroes 的取值只有两种情况，当版本号即地址前缀取 “0x00” 时，zeroes 为 1，其它情况 zeroes 为 0。
2. 开辟足够大的空间，34 或 35 个字节，为大端表示的 Base58 编码做准备。
3. **进行 Base58 编码，把 256 进制转化为 58 进制。**
4. 再次跳过开头的 0，这次不统计。
5. 在字符串填充字符 '1'，个数为 1. 步中跳过 0 的个数 zeroes。
6. 根据 Base58 字符对照表把数字转换为对应字符，得到地址。

对于第 3 步 Base58 转换的核心算法，256 进制转换为 58 进制，公式 "b58 = b58 * 256 + ch"。

```cpp
carry += 256 * (*it);
*it = carry % 58;
carry /= 58;
```

不理解这里的进制转换？
请看该例：小端模式内存中 0x0102（256 进制）对应 10 进制的 258 转换为大端模式 58 进制的过程。

```cpp
DEC 258 HEX 0x0102
    +---------+              Step2: 2 + 1 * 256 = 258
MEM |0x01|0x02+-------------------+ 258 % 58 =[26]
   L+--+------+B                  | 258 / 58 = 4
       |   Step1: 1 + 0 * 256 = 1 |
       +--------+ 1 % 58 =[1]     | 4 + 0 * 256 = 4
                | 1 % 58 = 0      | 4 % 58 =[4]
b58 +---------+ |                 | 4 / 58 = 0
MEM |0x00|0x00| | 0 + 0 * 256 = 0 |
    +-------+-+ | 0 % 58 =[0]     |
            |   | 0 % 58 = 0      |
       +--------+                 |
       |    |                     |
b58 +--v----v-+                   |
MEM |0x00|0x01|                   |
    +-------+-+                   |
            |                     |
       +--------------------------+
       |    |
b58 +--v----v-+
MEM |0x04|0x1A|
    +---------+
```

## 参考链接

* [Base58 - Wikipedia](https://en.wikipedia.org/wiki/Base58){:target="_blank"}
* [Base64 - Wikipedia](https://en.wikipedia.org/wiki/Base64){:target="_blank"}
* [Base58Check encoding - Bitcoin Wiki](https://en.bitcoin.it/wiki/Base58Check_encoding){:target="_blank"}
* [bitcoin/bitcoin/src/base58.cpp](https://github.com/bitcoin/bitcoin/blob/master/src/base58.cpp){:target="_blank"}
* [Bitcoin Base58 Encoder, Decoder, and Validator](http://lenschulwitz.com/base58){:target="_blank"}