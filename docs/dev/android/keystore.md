# 签名

##### 创建签名

* 创建签名

    ```shell
    flavorName=dev
    keytool -genkeypair -alias ${flavorName} -keyalg RSA -keypass 123456 -keystore android/app/infos/${flavorName}.jks -storepass 123456 -validity 36500
    ```

* 创建好签名后，执行以下命令，并提交生成文件

    ```shell
    flavorName=dev
    echo "--- 签名基本信息 ---" >> android/app/infos/${flavorName}_info.txt
    echo "" >> android/app/infos/${flavorName}_info.txt
    keytool -list -v -keystore android/app/infos/${flavorName}.jks -storepass 123456 2>/dev/null >> android/app/infos/${flavorName}_info.txt
    echo "" >> android/app/infos/${flavorName}_info.txt
    ```

* 微信/微博

    ```shell
    flavorName=dev
    echo "--- 微信/微博签名 ---" >> android/app/infos/${flavorName}_info.txt
    echo "" >> android/app/infos/${flavorName}_info.txt
    keytool -list -v -keystore android/app/infos/${flavorName}.jks -storepass 123456 2>/dev/null | grep -p 'MD5:.*' -o | sed 's/MD5://' | sed 's/ //g' | sed 's/://g' | awk '{print tolower($0)}' >> android/app/infos/${flavorName}_info.txt
    echo "" >> android/app/infos/${flavorName}_info.txt
    ```

* Google

    调试签名证书 SHA-1
    ```shell
    flavorName=dev
    echo "--- Google 调试签名证书 SHA-1 ---" >> android/app/infos/${flavorName}_info.txt
    echo "" >> android/app/infos/${flavorName}_info.txt
    keytool -exportcert -list -v -alias ${flavorName} -keystore android/app/infos/${flavorName}.jks -storepass 123456 2>/dev/null >> android/app/infos/${flavorName}_info.txt
    echo "" >> android/app/infos/${flavorName}_info.txt
    ```

    Google 管理并保护应用签名密钥
    ```shell
    flavorName=dev
    encryptionkey=xxx # 从 Java 密钥存储区导出并上传密钥和证书
    java -jar android/app/infos/pepk.jar --keystore=android/app/infos/${flavorName}.jks --alias=${flavorName} --output=android/app/infos/${flavorName}_google.zip --encryptionkey=${encryptionkey} --include-cert
    ```

* Facebook

    开发和发布密钥散列
    ```shell
    flavorName=dev
    echo "--- Facebook 开发和发布密钥散列 ---" >> android/app/infos/${flavorName}_info.txt
    echo "" >> android/app/infos/${flavorName}_info.txt
    keytool -exportcert -alias ${flavorName} -keystore android/app/infos/${flavorName}.jks -storepass 123456 2>/dev/null | openssl sha1 -binary | openssl base64 >> android/app/infos/${flavorName}_info.txt
    echo "" >> android/app/infos/${flavorName}_info.txt
    ```

* Line

    ```shell
    flavorName=dev
    echo "--- Line SHA hash ---" >> android/app/infos/${flavorName}_info.txt
    echo "" >> android/app/infos/${flavorName}_info.txt
    keytool -list -v -keystore android/app/infos/${flavorName}.jks -storepass 123456 2>/dev/null | grep -p 'SHA1:.*' -o | sed 's/SHA1://' | sed 's/ //g' | sed 's/://g' | awk '{print tolower($0)}' >> android/app/infos/${flavorName}_info.txt
    echo "" >> android/app/infos/${flavorName}_info.txt
    ```
