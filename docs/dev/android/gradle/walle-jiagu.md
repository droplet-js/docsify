# walle & 加固

##### walle

* [Meituan-Dianping/walle](https://github.com/Meituan-Dianping/walle)
* [v7lin/walle-docker](https://github.com/v7lin/walle-docker)

##### 360加固

* [360加固助手](https://jiagu.360.cn/#/global/download)
* [v7lin/qihoo360-jiagu-docker](https://github.com/v7lin/qihoo360-jiagu-docker)

##### 腾讯乐固

* 腾讯云
    * [访问管理](https://console.cloud.tencent.com/cam/capi)
    * [移动应用安全](https://console.cloud.tencent.com/ms/reinforce/list)
* [v7lin/tencentcloud-legu](https://github.com/v7lin/tencentcloud-legu)

##### 混合

walle 官方已经半放弃维护了，想要继续用用，自能自己维护 gradle 插件。相比而言维护 walle-cli 更简单，而且还能定制360加固、腾讯乐固、以及支持falvor

walle.gradle

```groovy
//
//使用方法
//
//apply from: 'walle.gradle'
//
//android {
//    productFlavors {
//        prod {...}
//    }
//
//    walleConfigs {
//        prod {
//            enabled = true
//
////            // https://github.com/v7lin/walle-docker
////            jarFile = file('script/walle-cli-all.jar') // 默认：file('script/walle-cli-all.jar')
//            channelFile = file('channel')
//
//            qihoo360 {
////                // https://github.com/v7lin/qihoo360-jiagu-docker
////                jiaguJarFile = file('script/jiagu/jiagu.jar') // 默认：file('script/jiagu/jiagu.jar')
//
//                account = 'xxx'
//                password = 'xxx'
//                channelId = 'qihu360'
//            }
//
//            tencent {
////                // https://github.com/v7lin/tencentcloud-legu
////                leguJarFile = file('script/legu-all.jar') // 默认：file('script/legu-all.jar')
//
//                secretId = 'xxx'
//                secretKey = 'xxx'
////                region = 'ap-guangzhou' // 可选：'ap-guangzhou'、'ap-shanghai'，默认：'ap-guangzhou'
//                channelId = 'tencent'
//            }
//
//            outputDir = file("${project.buildDir}/walle") // 默认：file("${project.buildDir}/outputs/apk/${flavorName}/${buildType}/walle")
//        }
//    }
//}
//
//walle {
//    enabled = false
//}
//

buildscript {
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.2'
    }
}

android {
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}

class Qihoo360 {
    File jiaguJarFile
    String account
    String password
    String channelId

    void validate(String variant) {
        if (account == null || account.empty) {
            throw new IllegalStateException("Qihoo360 account is empty for variant '$variant'")
        }
        if (password == null || password.empty) {
            throw new IllegalStateException("Qihoo360 password is empty for variant '$variant'")
        }
        if (channelId == null || channelId.empty) {
            throw new IllegalStateException("Qihoo360 channelId is empty for variant '$variant'")
        }
    }
}

class Tencent {
    File leguJarFile
    String secretId
    String secretKey
    String region
    String channelId

    void validate(String variant) {
        if (secretId == null || secretId.empty) {
            throw new IllegalStateException("Tencent secretId is empty for variant '$variant'")
        }
        if (secretKey == null || secretKey.empty) {
            throw new IllegalStateException("Tencent secretKey is empty for variant '$variant'")
        }
        if (channelId == null || channelId.empty) {
            throw new IllegalStateException("Tencent channelId is empty for variant '$variant'")
        }
    }
}

class Walle {
    final String name
    Boolean enabled
    File jarFile
    File channelFile
    Qihoo360 qihoo360
    Tencent tencent
    File outputDir

    Walle(String name = 'default') {
        this.name = name
    }

    void qihoo360(Closure closure){
        qihoo360 = new Qihoo360()
        closure.delegate = qihoo360
        closure()
    }

    void tencent(Closure closure){
        tencent = new Tencent()
        closure.delegate = tencent
        closure()
    }

    // ---

    void validate() {
        if (enabled == null || !enabled.booleanValue()) {
            return
        }
        if (channelFile == null) {
            throw new IllegalStateException("walle channel file is null for variant '$name'")
        }
        qihoo360?.validate(name)
        tencent?.validate(name)
    }

    Walle mergeWith(Walle other) {
        if (other == null) {
            return this
        }
        def mergeWalle = new Walle(name == 'default' ? other.name : (other.name == 'default' ? name : "$name${other.name.capitalize()}"))
        mergeWalle.enabled = other.enabled != null ? other.enabled : enabled
        mergeWalle.jarFile = other.jarFile ?: jarFile
        mergeWalle.channelFile = other.channelFile ?: channelFile
        mergeWalle.qihoo360 = other.qihoo360 ?: qihoo360
        mergeWalle.tencent = other.tencent ?: tencent
        mergeWalle.outputDir = other.outputDir ?: outputDir
        return mergeWalle
    }
}

apply plugin: WallePlugin

class WallePlugin implements Plugin<Project> {

    @Override
    void apply(Project target) {
        target.extensions.create('walle', Walle.class)
        target.plugins.withId('com.android.application') {
            def baseWalle = target.walle
            def walleConfigs = target.container(Walle.class)
            target.android.extensions.walleConfigs = walleConfigs
            target.android.applicationVariants.whenObjectAdded { variant ->
                Walle mergeWalle = null
                def flavorWalles = variant.productFlavors?.stream().map{flavor -> walleConfigs.findByName(flavor.name)}.collect().toList() ?: Collections.emptyList()
                def buildTypeWalle = walleConfigs.findByName(variant.buildType.name)
                if (buildTypeWalle == null && (variant.buildType.name == 'debug' || variant.buildType.name == 'profile')) {
                    buildTypeWalle = new Walle(variant.buildType.name)
                    buildTypeWalle.enabled = false
                }
                // buildType > flavor > base
                List<Walle> walles = new ArrayList<>()
                walles.add(baseWalle)
                walles.addAll(flavorWalles)
                walles.add(buildTypeWalle)
                for (def walle in walles) {
                    if (mergeWalle == null) {
                        mergeWalle = walle
                    } else {
                        mergeWalle = mergeWalle.mergeWith(walle)
                    }
                }

                mergeWalle?.validate()

                variant.assemble.doLast {
                    if (mergeWalle == null || mergeWalle.enabled == null || !mergeWalle.enabled.booleanValue()) {
                        target.logger.info("Gradle Walle is disabled for variant '${variant.name}'.")
                        return
                    }

                    if (!variant.signingReady && !variant.outputsAreSigned) {
                        target.logger.error("Signing not ready for Gradle Walle. Be sure to specify a signingConfig for variant '${variant.name}'.")
                        return
                    }

                    if (!org.gradle.internal.os.OperatingSystem.current().isMacOsX() && !org.gradle.internal.os.OperatingSystem.current().isLinux()) {
                        target.logger.info("Gradle Walle 仅能运行于 MacOS 或 Linux，不能运行于 ${org.gradle.internal.os.OperatingSystem.current().osName}。")
                        return
                    }

                    File apkFile = variant.outputs.first().outputFile as File

                    // android.buildToolsVersion，walle 官方暂未作 Apk Signature Scheme v3 支持
                    def buildToolsVersionForApkSigner = target.android.buildToolsVersion
                    if (v3SigningEnabled(target, apkFile)) {
                        buildToolsVersionForApkSigner = '27.0.3'
                        if (mergeWalle.qihoo360 != null || mergeWalle.tencent != null) {
                            if (!new File("${target.android.sdkDirectory.path}/build-tools/${buildToolsVersionForApkSigner}/apksigner").exists()) {
                                target.exec {
                                    commandLine 'bash', '-lc', "mkdir -p ~/.android/ && " +
                                            "touch ~/.android/repositories.cfg && " +
                                            "echo y | ${target.android.sdkDirectory.path}/tools/bin/sdkmanager --install \"build-tools;${buildToolsVersionForApkSigner}\""
                                }
                            }
                        }
                    }

                    System.out.println('--- walle ---')
                    System.out.println("name: ${mergeWalle.name}")
                    System.out.println("channel file: ${mergeWalle.channelFile}")

                    File jarFile = mergeWalle.jarFile
                    if (jarFile == null) {
                        jarFile = target.file('script/walle-cli-all.jar')
                    }
                    if (!jarFile.exists()) {
                        jarFile.parentFile.mkdirs()
                        System.out.println('download walle-cli-all.jar')
                        def downloadUrl = 'https://github.com/Meituan-Dianping/walle/releases/download/v1.1.6/walle-cli-all.jar'
                        if (org.gradle.internal.os.OperatingSystem.current().isMacOsX()) {
                            target.exec {
                                commandLine 'bash', '-lc', "curl -iL --max-redirs 5 -o ${jarFile.path} $downloadUrl"
                            }
                        }
                        if (org.gradle.internal.os.OperatingSystem.current().isLinux()) {
                            target.exec {
                                commandLine 'bash', '-lc', "wget -O ${jarFile.path} $downloadUrl"
                            }
                        }
                        if (!jarFile.exists()) {
                            throw new IllegalStateException('下载 walle-cli-all.jar 失败')
                        }
                    }
                    if (!mergeWalle.channelFile.exists()) {
                        throw new IllegalStateException('walle channel file 不存在')
                    }

                    System.out.println("apk file path: ${apkFile.path}")

                    File outputDir = mergeWalle.outputDir
                    if (outputDir == null) {
                        outputDir = new File(apkFile.parentFile, 'walle')
                    }
                    File channelsDir = new File(outputDir, 'channels')
                    if (!channelsDir.exists()) {
                        channelsDir.mkdirs()
                    }

                    File destApkFile = new File(outputDir, apkFile.name.replace('app-', '').replace("${variant.buildType.name}", "${variant.versionName}"))

                    System.out.println("v3SigningEnabled: ${v3SigningEnabled(target, apkFile)}")

                    if (v3SigningEnabled(target, apkFile)) {
                        // 重新签名
                        signApk(target, variant, buildToolsVersionForApkSigner, apkFile, destApkFile)
                    } else {
                        target.exec {
                            commandLine 'bash', '-lc', "cp ${apkFile.path} ${destApkFile.path}"
                        }
                    }

                    System.out.println('write channel')
                    System.out.println("package name: ${variant.applicationId}")
                    writeChannels(target, jarFile, destApkFile, mergeWalle.channelFile, channelsDir)

                    if (mergeWalle.qihoo360 != null) {
                        checkChannel(mergeWalle.channelFile, mergeWalle.qihoo360.channelId, 'qihoo360 渠道配置错误')

                        System.out.println('qihoo360 jiagu & channel')

                        File jiaguJarFile = mergeWalle.qihoo360.jiaguJarFile
                        if (jiaguJarFile == null) {
                            jiaguJarFile = target.file('script/jiagu/jiagu.jar')
                        }
                        if (!jiaguJarFile.exists()) {
                            jiaguJarFile.parentFile.parentFile.mkdirs()
                            System.out.println('download jiagu/jiagu.jar')
                            if (org.gradle.internal.os.OperatingSystem.current().isMacOsX()) {
                                File jiaguZipFile = target.file('script/qihoo360-jiagu.zip')
                                if (!jiaguZipFile.exists()) {
                                    def downloadUrl = 'https://down.360safe.com/360Jiagu/360jiagubao_mac.zip'
                                    target.exec {
                                        commandLine 'bash', '-lc', "curl -iL --max-redirs 5 -o ${jiaguZipFile.path} $downloadUrl"
                                    }
                                }
                                try {
                                    target.exec {
                                        commandLine 'bash', '-lc', "unzip -o ${jiaguZipFile.path} 'jiagu/*' -d ${jiaguZipFile.parentFile.path}" // 坑爹啊，普通 shell 下 unzip 是正常的！！！
                                    }
                                } catch (e) {
                                }
                            }
                            if (org.gradle.internal.os.OperatingSystem.current().isLinux()) {
                                File jiaguZipFile = target.file('script/qihoo360-jiagu.zip')
                                if (!jiaguZipFile.exists()) {
                                    def downloadUrl = 'https://down.360safe.com/360Jiagu/360jiagubao_linux_64.zip'
                                    target.exec {
                                        commandLine 'bash', '-lc', "wget -O ${jiaguZipFile.path} $downloadUrl"
                                    }
                                }
                                target.exec {
                                    commandLine 'bash', '-lc', "unzip ${jiaguZipFile.path}"
                                }
                            }
                            if (!jiaguJarFile.exists()) {
                                throw new IllegalStateException('下载 jiagu/jiagu.jar 失败')
                            }
                        }

                        System.out.println('jiagu')
                        File jiaguDir = new File(outputDir, 'jiagu')
                        jiaguDir.mkdirs()
                        if (org.gradle.internal.os.OperatingSystem.current().isMacOsX()) {
                            target.exec {
                                commandLine 'bash', '-lc', "java -jar ${jiaguJarFile.path} -login ${mergeWalle.qihoo360.account} ${mergeWalle.qihoo360.password}"
                            }
                            target.exec {
                                commandLine 'bash', '-lc', "java -jar ${jiaguJarFile.path} -jiagu ${destApkFile.path} ${jiaguDir.path}"
                            }
                        }
                        if (org.gradle.internal.os.OperatingSystem.current().isLinux()) {
                            target.exec {
                                commandLine 'bash', '-lc', "${jiaguJarFile.parentFile.path}/java/bin/java -jar ${jiaguJarFile.path} -login ${mergeWalle.qihoo360.account} ${mergeWalle.qihoo360.password}"
                            }
                            target.exec {
                                commandLine 'bash', '-lc', "${jiaguJarFile.parentFile.path}/java/bin/java -jar ${jiaguJarFile.path} -jiagu ${destApkFile.path} ${jiaguDir.path}"
                            }
                        }

                        File jiaguApk = jiaguDir.listFiles(new FilenameFilter() {
                            @Override
                            boolean accept(File dir, String name) {
                                return name.toLowerCase().endsWith('.apk')
                            }
                        }).first()

                        System.out.println('jiagu sign')
                        File jiaguApkSigned = new File(jiaguDir.path, destApkFile.name.replace('.apk', '_jiagu_signed.apk'))
                        signApk(target, variant, buildToolsVersionForApkSigner, jiaguApk, jiaguApkSigned)

                        System.out.println('jiagu write channel')
                        File jiaguApkChannel = new File(channelsDir.path, destApkFile.name.replace('.apk', "_${mergeWalle.qihoo360.channelId}.apk"))
                        writeChannel(target, jarFile, jiaguApkSigned, mergeWalle.qihoo360.channelId, jiaguApkChannel)
                    }

                    if (mergeWalle.tencent != null) {
                        checkChannel(mergeWalle.channelFile, mergeWalle.tencent.channelId, 'tencent 渠道配置错误')

                        System.out.println('tencent legu & channel')

                        File leguJarFile = mergeWalle.tencent.leguJarFile
                        if (leguJarFile == null) {
                            leguJarFile = target.file('script/legu-all.jar')
                        }
                        if (!leguJarFile.exists()) {
                            leguJarFile.parentFile.mkdirs()
                            System.out.println('download legu-all.jar')
                            def downloadUrl = 'https://github.com/v7lin/tencentcloud-legu/releases/download/3.0.60/legu-all.jar'
                            if (org.gradle.internal.os.OperatingSystem.current().isMacOsX()) {
                                target.exec {
                                    commandLine 'bash', '-lc', "curl -iL --max-redirs 5 -o ${leguJarFile.path} $downloadUrl"
                                }
                            }
                            if (org.gradle.internal.os.OperatingSystem.current().isLinux()) {
                                target.exec {
                                    commandLine 'bash', '-lc', "wget -O ${leguJarFile.path} $downloadUrl"
                                }
                            }
                            if (!leguJarFile.exists()) {
                                throw new IllegalStateException('下载 legu-all.jar 失败')
                            }
                        }

                        System.out.println('legu')
                        File leguDir = new File(outputDir, 'legu')
                        leguDir.mkdirs()
                        target.exec {
                            commandLine 'bash', '-lc', "java -jar ${leguJarFile.path} configure " +
                                    "-secretId ${mergeWalle.tencent.secretId} " +
                                    "-secretKey ${mergeWalle.tencent.secretKey} " +
                                    "-region ${mergeWalle.tencent.region ?: 'ap-guangzhou'}"
                        }
                        target.exec {
                            commandLine 'bash', '-lc', "java -jar ${leguJarFile.path} legu " +
                                    "-in ${destApkFile.path} " +
                                    "-out ${leguDir.path}"
                        }

                        File leguApk = leguDir.listFiles(new FilenameFilter() {
                            @Override
                            boolean accept(File dir, String name) {
                                return name.toLowerCase().endsWith('.apk')
                            }
                        }).first()

                        System.out.println('legu sign')
                        File leguApkSigned = new File(leguDir.path, destApkFile.name.replace('.apk', '_legu_signed.apk'))
                        signApk(target, variant, buildToolsVersionForApkSigner, leguApk, leguApkSigned)

                        System.out.println('legu write channel')
                        File leguApkChannel = new File(channelsDir.path, destApkFile.name.replace('.apk', "_${mergeWalle.tencent.channelId}.apk"))
                        writeChannel(target, jarFile, leguApkSigned, mergeWalle.tencent.channelId, leguApkChannel)
                    }

                    System.out.println('--- walle ---')
                }
            }
        }
        target.afterEvaluate {
            if (!target.plugins.hasPlugin('com.android.application')) {
                target.logger.warn("The Android Gradle Plugin was not applied. Gradle Walle will not be configured.")
            }
        }
    }

    boolean v3SigningEnabled(Project target, File apkFile) {
        // android.buildToolsVersion 从 29.0.0 才开始支持 v3 signing scheme
        File verifyLogFile = new File(apkFile.parentFile, 'verify.log')
        if (verifyLogFile.exists()) {
            verifyLogFile.delete()
        }
        target.exec {
            commandLine 'bash', '-lc', "${target.android.sdkDirectory.path}/build-tools/${target.android.buildToolsVersion}/apksigner verify --verbose ${apkFile.path} >> ${verifyLogFile.path}"
        }
        return verifyLogFile.readLines().contains('Verified using v3 scheme (APK Signature Scheme v3): true')
    }

    void signApk(Project target, variant, String buildToolsVersion, File apkFile, File destApkFile) {
        target.exec {
            commandLine 'bash', '-lc', "${target.android.sdkDirectory.path}/build-tools/${buildToolsVersion}/apksigner sign " +
                    "-ks ${variant.signingConfig.storeFile.path} " +
                    "-ks-pass pass:${variant.signingConfig.storePassword} " +
                    "-ks-key-alias ${variant.signingConfig.keyAlias} " +
                    "--key-pass pass:${variant.signingConfig.keyPassword} " +
                    "--out ${destApkFile.path} ${apkFile.path}"
        }
    }

    void writeChannels(Project target, File jarFile, File apkFile, File channelFile, File outputDir) {
        target.exec {
            commandLine 'bash', '-lc', "java -jar ${jarFile.path} batch -f ${channelFile.path} ${apkFile.path} ${outputDir.path}"
        }
    }

    void writeChannel(Project target, File jarFile, File apkFile, String channelId, File outputApkFile) {
        target.exec {
            commandLine 'bash', '-lc', "java -jar ${jarFile.path} put -c $channelId ${apkFile.path} ${outputApkFile.path}"
        }
    }

    void checkChannel(File channelFile, String channelId, String errorMsg) {
        List<String> channelLines = channelFile.readLines()
        if (!channelLines.contains(channelId)) {
            throw new IllegalStateException(errorMsg)
        }
    }
}
```
