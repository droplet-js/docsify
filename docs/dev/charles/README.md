# Charles

官网：[https://www.charlesproxy.com/](https://www.charlesproxy.com/)
下载地址：[https://www.charlesproxy.com/download/](https://www.charlesproxy.com/download/)

### 安装

* Help -> SSL Proxying -> Install Charles Root Certificate
* 钥匙串访问 -> Charles Proxy SSL Proxying -> 双击 -> 展开"信任" -> "使用此证书时"："始终信任" -> 输入Mac密码
* Proxy -> Proxy Settings -> HTTP Proxy -> 配置端口（默认：8888，一般不作修改）
* Proxy -> SSL Proxying Settings -> SSL Proxying -> Enable SSL Proxying -> Include -> Add "*:*", "*:443"
* Help -> SSL Proxying -> Install Charles Root Certificate on a Mobile Device or Remote Browser
* 手机网络和电脑网络需在同一局域网内，并配置手机网络代理，代理IP为电脑IP，代理端口时上文中的端口
