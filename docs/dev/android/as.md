# Android Studio

##### 安装

##### 记录异常解决

* Android Studio自带JDK与系统已安装的JDK的冲突

    问题：
    ```shell
    FAILURE: Build failed with an exception.
    
    * What went wrong:
    A problem occurred configuring project ':app'.
    > Failed to notify project evaluation listener.
       > javax/xml/bind/annotation/XmlSchema
    ```
    
    解决方案：
    
    ```shell
    export JAVA_HOME="/Applications/Android Studio.app/Contents/jre/jdk/Contents/Home"
    ```