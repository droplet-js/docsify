# 海外

##### Google Play

* [Google Play Console](https://play.google.com/apps/publish)
* 配置商品信息（APP名称/副标题/推荐语/简介）
* 商家账号/付款方式
* 设置 -> 开发者账号 -> API 权限 -> Google Play 新账号的话，要先关联下，必须在添加内购前关联 Google Play Android Developer
* 设置 -> 开发者账号 -> API 权限 -> 创建服务账号 -> 为 Billing（账单和财务） 创建服务账号并创建密钥（自动生成并下载 json 格式密钥）
* 设置 -> 开发者账号 -> 用户和权限 -> Billing（账单和财务）的服务账号添加访问权限（Google Play Android Developer项目上，这个必须要早于配置内购产品，否则需要重现保存一下内购产品）
* 设置 -> 开发者账号 -> 关联的账号 -> 先到 Firebase 上邀请 Google Play 账号所有者为管理人员后，Google Play 上才能关联 Firebase 项目

##### Firebase

* [Firebase](https://console.firebase.google.com/)
* 设置 - 云消息传递 - 配置 iOS APNs 身份验证密钥
* 设置 - 服务账号 - 配置 Firebase Admin SDK 密钥（自动生成并下载 json 格式密钥）
* 应用配置

    |模块|备注|
    |:---:|:---:|
    |后端|服务器密钥|
    |Android|google-services.json|
    |iOS|GoogleService-Info.plist|

##### Google Cloud - Google Play 创建服务账号 / Firebase 服务账号

* [Google Cloud Platform](https://console.cloud.google.com/)

##### Google APIs

* [Google APIs](https://console.developers.google.com/)
* 应用配置

    |Platform|Client ID|
    |:---:|:---:|
    |Android|111-xxx.apps.googleusercontent.com|
    |iOS|111-yyy.apps.googleusercontent.com|
    |Web|111-zzz.apps.googleusercontent.com|

##### Google Ads

* [Android Quick Start](https://developers.google.com/admob/android/quick-start)
* [iOS Quick Start](https://developers.google.com/admob/ios/quick-start)
* [Admob 控制台](https://admob.google.com/home/?utm_source=firebase&utm_medium=et&utm_campaign=firebase-docs&utm_content=2017Q1)
* 结算币种 -> 美元
* 广告格式
    * 横幅
        * 广告单元名称 -> banner
        * 高级设置 -> 去掉视频
    * 激励视频
        * 广告单元名称 -> reward
        * 高级设置 -> 服务器端验证 -> TODO
* 请注意，你创建的项目需要等待一段时间才能测试使用，所以尽管你的代码和配置都是正确的，但还是收到错误。等待一段时间就好了
* 激励视频并不总是会有的

##### Facebook

* [facebook for developers](https://developers.facebook.com/apps/)
* 配置隐私协议
* 配置 Android/iOS 平台
* 配置登录/深度链接
* 配置管理人员和开发人员
* 应用配置

    ```
    App ID: xxx
    App Secret: yyy
    ```

##### Facebook Ads

* xxx

##### LINE

* [LINE Developers](https://developers.line.biz/)
* 企业账号登录
* 配置隐私协议/用户协议
* 发布
* 应用配置

    ```
    Channel ID: xxx
    Channel secret: yyy
    ```

##### Google Play上架App设置隐私政策声明问题

* 在[App Privacy Policy Generator](https://app-privacy-policy-generator.firebaseapp.com/)生成隐私内容，在[Google 协作平台](https://sites.google.com/site?pli=1)上托管privacy-policy页面
* [参考文档](http://javaexception.com/archives/33)
