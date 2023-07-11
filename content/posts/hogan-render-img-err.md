---
title: 使用Hogan渲染html模板加载图片出错
date: 2019-01-08 18:22:04
tags:
- hogan
categories:
- JavaScript
---
在从服务器加载数据并渲染商品列表的时候出现`Cannot find module "./{{imageHost}}{{mainImage}}"`错误，查看返回的json字段信息如下:
<!-- more -->

```json
{
    "status": 0,
    "data": {
        "pageNum": 1,
        "pageSize": 10,
        "size": 1,
        "orderBy": "price asc",
        "startRow": 1,
        "endRow": 1,
        "total": 1,
        "pages": 1,
        "list": [
            {
                "id": 26,
                "categoryId": 100002,
                "name": "Apple iPhone 7 Plus (A1661) 128G 玫瑰金色 移动联通电信4G手机",
                "subtitle": "iPhone 7，现更以红色呈现。",
                "mainImage": "241997c4-9e62-4824-b7f0-7425c3c28917.jpeg",
                "price": 6999,
                "status": 1,
                "imageHost": "//img.shashamall.com/"
            }
        ],
        "firstPage": 1,
        "prePage": 0,
        "nextPage": 0,
        "lastPage": 1,
        "isFirstPage": true,
        "isLastPage": true,
        "hasPreviousPage": false,
        "hasNextPage": false,
        "navigatePages": 8,
        "navigatepageNums": [
            1
        ]
    }
}
```
HTML模板的string文件如下：<br>
```html
{{#list}}
    <li class="p-item">
        <div class="p-img-con">
            <a href="./detail.html?productId={{id}}" target="_blank" class="link">
                <img src='{{imageHost}}{{mainImage}}' alt="{{name}}">
            </a>
        </div>
        <div class="p-price-con">
            <span class="p-price">￥{{price}}</span>
        </div>
        <div class="p-name-con">
            <a href="./detail.html?productId={{id}}" target="_blank" class="p-name">{{name}}</a>
        </div>
    </li>
{{/list}}
```
页面传参如下：<br>
```js
// 调用service层
_product.getProducts(listParam, function(res){
    listHtml = _util.renderHtml(templateIndex, {
        list : res.list
    });
    // 将渲染成功的html放入容器
    $('.p-list-con').html(listHtml);
    // 渲染分页信息
    _this.loadPagination(res.pageNum, res.pages);
}, function(errMsg){
    // 错误提示
    _util.errorTips(errMsg);
});
```
按道理`img`标签的`src`属性应该会解析成`//img.shashamall.com/241997c4-9e62-4824-b7f0-7425c3c28917.jpeg`，然后这里浏览器解析的时候应该会加上`http`从而获取图片资源的，可是这里却报错。多番尝试之后在模板src前添加"`/`"符号解决问题，目前原因并不知道，这里先记录这个错误，后续再查阅资料，也希望知道的大神告知，解我疑惑！
>解决方法：
改`<img src='{{imageHost}}{{mainImage}}' alt="{{name}}">`为`<img src='/{{imageHost}}{{mainImage}}' alt="{{name}}">`