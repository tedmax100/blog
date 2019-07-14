---
title: AWS CloudFront
date: 2018-12-15 12:56:46
tags:
    - AWS
---
AWS CloudFront
* CloudFront
加快靜態和動態web內容分配給用戶的Web服務。 透過全球數據中心(edge location)來傳輸內容。當user向CloudFront請求提供內容時，user會被陸游到提供最低延遲的edge location，以最佳的速度傳送內容。
如果內容已經存在edge location十，則cloudfront將直接提供它。(但我們的情境不需要去快取資料)
如果請求的內容不再edge，則會對web server進行查找。

* 配置CloudFront
配置source server，CloudFront將從這些server獲取資料。
source server儲存的對象是當下最終的版本。
建立CloudFront distribution，user通過你的網站或api在請求資料時，告訴cloudfront從那些source server來獲取資料。還需要指定cloudfront是否記錄所有請求以及該分配建立後立即啟用。
將配置發送到所有edge location。
![](/images/AWS/how-you-configure-cf.png)

* Optional configuration:
可以配置source server對response添加header;header能設置希望資料再edge location保留的時限。默認是保留24hr。minimum expiration = 0s

* HTTP Methods
POST/PUT/PATCH/OPTIONS/DELETE，將直接從edge location站點直接會到連source server，並不會流過regional edge location做快取查找。

[指定對象在CloudFront edge location的expiration](https://docs.aws.amazon.com/zh_cn/AmazonCloudFront/latest/DeveloperGuide/Expiration.html)
[Values That You Specify When You Create or Update a Web Distribution](https://docs.aws.amazon.com/zh_cn/AmazonCloudFront/latest/DeveloperGuide/distribution-web-values-specify.html#DownloadDistValuesCacheBehavior)

-----------------------------------------------
**Lab**
* CloudFront +　S3 :
    * Staging config :
** Restrict Bucket Access： 選擇”Yes”，因為希望網站是向CloudFront存取，而不是向S3。
    * Query String Forwarding and Caching : only for API
Certifacate type : TLS11