---
title: Docker入門_01
date: 2017-11-13 00:01:59
tags:
    - Docker
---
# What is Docker?
<!--more-->
![](/images/Docker01/640.jpg)

Docker從廣義上是個服務容器(Application Container)，基本上跟一般系統中執行的Process並無不同，特別的是它負責操作鏡像檔案(images)。
所以Docker+構建昇成出來的image file == Docker Container。

## Windows上的Docker有何不同
Docker在Windows上安裝好時，會自動安裝一個Docker專用的Linux虛擬機，透過Hyper-V來管理。透過這個Linux虛擬基在背後提供和運行Container的方式來達成使用。

也正因為如此，所以目前找到的打包好的image file都是以Linux為主的原因了。
所以基本上，Docker上的容器都是跑在Linux內核中，只是單獨包成一隻應用程式檔，掛載進去Docker的進程內。

Docker images鏡像 ，就類似於VM中的快照，但容量卻小上許多，Docker透過ID或是容易識別的別名+tag來抓到唯一的目標鏡像檔。ImagesID是一個64bit長度的字串，但通常只要使用前4碼即可。

## Ubuntu上安裝Docker
[Link](https://docs.docker.com/install/linux/docker-ce/ubuntu/)