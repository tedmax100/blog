---
title: PixiJS-簡介
date: 2017-06-05 23:59:50
tags:
    - JavaSCript
    - Pixi
---
# PixiJS - PixiJs 簡介
`Pixi.js is a rendering library that will allow you to create rich, interactive graphics, cross platform applications, and games without having to dive into the WebGL API or deal with browser and device compatibility.`
<!--more-->
PixiJS官網上的說明這是一個套件，能讓玩家創造出豐富的、互動的、跨平台的應用/遊戲。開發者不必深入理解WebGL的API或是處理瀏覽器的相容性問題的一個容易上手的套件。

利用texture(紋理)來準備遊戲圖形，使用Proton粒子引擎做到particle effect(粒子效果)，以及如何將Pixi整合到自己做的遊戲引擎中。當然它不只適用於遊戲，也能用來創建任何交互式的應用程式。

## 安裝
小弟安裝學習的版本是V4.0.0
直接使用官網提供的CDN載點
[Pixi Link](https://github.com/pixijs/pixi.js/wiki/FAQs#where-can-i-get-a-build)
```javascript
<head>
  <meta charset="utf-8">
  <title>Hello World</title>
</head>
  <script src="pixi.min.js"></script>
<body>
  <script type="text/javascript">
    var type = "WebGL"
    if(!PIXI.utils.isWebGLSupported()){
      type = "canvas"
    }
    PIXI.utils.sayHello(type)
  </script>
</body>
```