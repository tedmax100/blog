---
title: JS30 - JavaScript Drum Kit
date: 2017-06-04 23:49:37
tags:
    - JavaSCript
    - JS30Day
---
#核心需求
* 根據鍵盤的KeyCode，來撥放對應的聲音
* 改變觸發的物件樣式

# 實現思維
* 在元素上綁定keydown event
* 對應事件的處理流程
    + 給每個div元素綁定transitioned event
        1. 綁定事件
        2. 獲取所有classname為key的元素
* 去除樣式的事件處理流程

{% iframe https://codepen.io/tedmax100/pen/GERaWN %}

```javascript
windows.addEventListener('keydown', function(e){
    console.log(e);
});
```