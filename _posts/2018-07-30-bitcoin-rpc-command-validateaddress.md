---
layout: post
title:  "比特币 RPC 命令剖析 \"validateaddress\""
date:   2018-07-30 09:12:36 +0800
author: mistydew
comments: true
categories: Blockchain Bitcoin
tags: CLI bitcoin-cli 区块链 比特币
excerpt: $ bitcoin-cli validateaddress "bitcoinaddress"
---
## 提示说明

{% highlight shell %}
validateaddress "bitcoinaddress" # 获取关于给定比特币地址的信息（不含余额）
{% endhighlight %}

参数：<br>
1.bitcoinaddress（字符串，必备）用于验证的比特币地址。

结果：<br>
{% highlight shell %}
{
  "isvalid" : true|false,       （布尔型）地址是否有效。若无效，则只返回该项
  "address" : "bitcoinaddress", （字符串）验证过的比特币地址
  "scriptPubKey" : "hex",       （字符串）通过地址生成的 16 进制编码的脚本公钥
  "ismine" : true|false,        （布尔型）地址是否属于我（钱包）
  "iswatchonly" : true|false,   （布尔型）地址是否为 watchonly 地址
  "isscript" : true|false,      （布尔型）密钥是否为一个脚本
  "pubkey" : "publickeyhex",    （字符串）原始公钥的 16 进制值
  "iscompressed" : true|false,  （布尔型）地址是否压缩过
  "account" : "account"         （字符串，已过时）该地址关联的账户，"" 为默认账户
}
{% endhighlight %}

## 用法示例

### 比特币核心客户端

用法一：验证本钱包中的地址。

{% highlight shell %}
$ bitcoin-cli getnewaddress
1C5RT9cGsDZwNyaEvUHPj7Qb1SgB5eePG
$ bitcoin-cli validateaddress 1C5RT9cGsDZwNyaEvUHPj7Qb1SgB5eePG
{
  "isvalid": true,
  "address": "1C5RT9cGsDZwNyaEvUHPj7Qb1SgB5eePG",
  "scriptPubKey": "76a9140218443a78a9ed0013efb2332506625dcfdf61b488ac",
  "ismine": true,
  "iswatchonly": false,
  "isscript": false,
  "pubkey": "028a82549a52b0e931d5dceadcf4509d93e45981683896e5c0d106dd802d59034f",
  "iscompressed": true,
  "account": ""
}
{% endhighlight %}

用法二：验证非本钱包中的地址。

{% highlight shell %}
$ bitcoin-cli validateaddress 1PSSGeFHDnKNxiEyFrD1wcEaHr9hrQDDWc
{
  "isvalid": true,
  "address": "1PSSGeFHDnKNxiEyFrD1wcEaHr9hrQDDWc",
  "scriptPubKey": "76a914f62242a747ec1cb02afd56aac978faf05b90462e88ac",
  "ismine": false,
  "iswatchonly": false,
  "isscript": false
}
{% endhighlight %}

用法三：验证错误地址，这里地址前缀错误。

{% highlight shell %}
$ bitcoin-cli validateaddress 2C5RT9cGsDZwNyaEvUHPj7Qb1SgB5eePG
{
  "isvalid": false
}
{% endhighlight %}

### cURL

{% highlight shell %}
$ curl --user myusername:mypassword --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "validateaddress", "params": ["1C5RT9cGsDZwNyaEvUHPj7Qb1SgB5eePG"] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/
{"result":{"isvalid":true,"address":"1C5RT9cGsDZwNyaEvUHPj7Qb1SgB5eePG","scriptPubKey":"76a9140218443a78a9ed0013efb2332506625dcfdf61b488ac","ismine":true,"iswatchonly":false,"isscript":false,"pubkey":"028a82549a52b0e931d5dceadcf4509d93e45981683896e5c0d106dd802d59034f","iscompressed":true,"account":""},"error":null,"id":"curltest"}
{% endhighlight %}

## 源码剖析
validateaddress 对应的函数在“rpcserver.h”文件中被引用。

{% highlight C++ %}
extern UniValue validateaddress(const UniValue& params, bool fHelp); // 验证地址
{% endhighlight %}

实现在“rpcmisc.cpp”文件中。

{% highlight C++ %}
UniValue validateaddress(const UniValue& params, bool fHelp)
{
    if (fHelp || params.size() != 1) // 参数必须为 1 个
        throw runtime_error( // 命令帮助反馈
            "validateaddress \"bitcoinaddress\"\n"
            "\nReturn information about the given bitcoin address.\n"
            "\nArguments:\n"
            "1. \"bitcoinaddress\"     (string, required) The bitcoin address to validate\n"
            "\nResult:\n"
            "{\n"
            "  \"isvalid\" : true|false,       (boolean) If the address is valid or not. If not, this is the only property returned.\n"
            "  \"address\" : \"bitcoinaddress\", (string) The bitcoin address validated\n"
            "  \"scriptPubKey\" : \"hex\",       (string) The hex encoded scriptPubKey generated by the address\n"
            "  \"ismine\" : true|false,        (boolean) If the address is yours or not\n"
            "  \"iswatchonly\" : true|false,   (boolean) If the address is watchonly\n"
            "  \"isscript\" : true|false,      (boolean) If the key is a script\n"
            "  \"pubkey\" : \"publickeyhex\",    (string) The hex value of the raw public key\n"
            "  \"iscompressed\" : true|false,  (boolean) If the address is compressed\n"
            "  \"account\" : \"account\"         (string) DEPRECATED. The account associated with the address, \"\" is the default account\n"
            "}\n"
            "\nExamples:\n"
            + HelpExampleCli("validateaddress", "\"1PSSGeFHDnKNxiEyFrD1wcEaHr9hrQDDWc\"")
            + HelpExampleRpc("validateaddress", "\"1PSSGeFHDnKNxiEyFrD1wcEaHr9hrQDDWc\"")
        );

#ifdef ENABLE_WALLET
    LOCK2(cs_main, pwalletMain ? &pwalletMain->cs_wallet : NULL); // 钱包上锁
#else
    LOCK(cs_main);
#endif

    CBitcoinAddress address(params[0].get_str()); // 获取指定的比特币地址
    bool isValid = address.IsValid(); // 判断地址是否有效

    UniValue ret(UniValue::VOBJ); // 对象类型的返回结果
    ret.push_back(Pair("isvalid", isValid)); // 有效性
    if (isValid) // 若有效
    {
        CTxDestination dest = address.Get(); // 从比特币地址获取交易目的地址
        string currentAddress = address.ToString();
        ret.push_back(Pair("address", currentAddress)); // 当前比特币地址

        CScript scriptPubKey = GetScriptForDestination(dest); // 从交易目的地址获取脚本公钥
        ret.push_back(Pair("scriptPubKey", HexStr(scriptPubKey.begin(), scriptPubKey.end()))); // 脚本公钥

#ifdef ENABLE_WALLET
        isminetype mine = pwalletMain ? IsMine(*pwalletMain, dest) : ISMINE_NO;
        ret.push_back(Pair("ismine", (mine & ISMINE_SPENDABLE) ? true : false)); // 是否属于我的
        ret.push_back(Pair("iswatchonly", (mine & ISMINE_WATCH_ONLY) ? true: false)); // 是否为 watch-only 地址
        UniValue detail = boost::apply_visitor(DescribeAddressVisitor(), dest); // 排序
        ret.pushKVs(detail); // 地址细节
        if (pwalletMain && pwalletMain->mapAddressBook.count(dest)) // 若钱包可用 且 该目的地址在地址簿中
            ret.push_back(Pair("account", pwalletMain->mapAddressBook[dest].name)); // 获取该地址关联的账户名
#endif
    }
    return ret; // 返回结果
}
{% endhighlight %}

基本流程：<br>
1.处理命令帮助和参数个数。<br>
2.上锁，若开启钱包功能，钱包上锁。<br>
3.获取指定的比特币地址，并判断该地址是否有效。<br>
4.获取相关地址信息，判断该地址是否属于自己并添加到结果对象中。<br>
5.返回结果。

Thanks for your time.

## 参照
* [Developer Documentation - Bitcoin](https://bitcoin.org/en/developer-documentation)
* [Bitcoin Developer Reference - Bitcoin](https://bitcoin.org/en/developer-reference#validateaddress)
* [精通比特币（第二版） \| 巴比特图书](http://book.8btc.com/masterbitcoin2cn)
* [...](https://github.com/mistydew/blockchain)