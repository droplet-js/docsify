# ~~Ngrok~~

Ngrok 2.x 商业化挺让人蛋疼的，而在 Docker 上搭建 Ngrok 1.x 服务器也让人非常恶心。

Docker 上搭建 Ngrok 服务器就不说了，说一下免费使用 Ngrok 内网穿透吧。没有私有服务器和域名的孩子可以耍耍。

```yaml
# 版本
version: "3.7"

# 服务
services:

  www:
    container_name: www
    image: nginx:1.15.6
    restart: always
    hostname: www
#    ports:
#      - 80
    volumes:
      - ../dev-ops-repo/www/html:/usr/share/nginx/html
    environment:
      - TZ=${TIME_ZONE}

  # 天下没有免费的午餐 - 访问慢，不稳定
  # 内网穿透
  ngrok-www:
    container_name: ngrok-www
    image: wernight/ngrok
    restart: always
    hostname: ngrok-www
    ports:
      - 4040:4040
    environment:
#      - TZ=${TIME_ZONE}
      - NGROK_AUTH={your ngrok auth}
      - NGROK_PROTOCOL=http
      - NGROK_PORT=www:80
    depends_on:
      - www
```