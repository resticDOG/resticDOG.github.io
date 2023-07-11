---
title: 取非符号的妙用
date: 2018-12-28 16:55:56
tags:
- 小技巧
categories: 
- JavaScript
---

如下代码块：

```javascript
    // 字段的验证，支持是否非空、是否是手机号、是否是邮箱地址的判断
    validate : function(value, type){
        //需要把前后的空格去掉
        var value = $.trim(value);
        // 非空判断
        if ('require' === type){
            // 这里可以将value转换成boolean型
            return !!value;
        }
    }
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;经过测试在js中`!null`、`!undefined`、`!''`输出结果都为`true`，而任意非空字符串取非均为`false`。代码中巧妙的利用了这点，先用jquery把传入的参数去掉空格，同时转换成字符串，再进行二次取非操作，这样传入的空字符就会返回boolean型的`true`,这样函数的返回值也更容易理解，当传入的是空串时返回`true`。
