# Logging

```groovy
//
//使用方法
//
//apply from: 'logging.gradle'
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

apply plugin: LoggingPlugin

class LoggingPlugin implements Plugin<Project> {

    @Override
    void apply(Project target) {
        target.plugins.withId('com.android.application') {
            target.android.applicationVariants.whenObjectAdded { variant ->
                variant.assemble.doLast {
                    environment(target, variant)
                }

                target.tasks.findByPath("bundle${variant.name.capitalize()}").doLast {
                    environment(target, variant)
                }
            }
        }
        target.afterEvaluate {
            if (!target.plugins.hasPlugin('com.android.application')) {
                target.logger.warn("The Android Gradle Plugin was not applied. Gradle Logging will not be configured.")
            }
        }
    }

    void environment(Project target, variant) {
        System.out.println('--- environment ---')

        System.out.println("compile sdk version:: ${target.android.compileSdkVersion}")
        System.out.println("build tools version: ${target.android.buildToolsVersion}")
        System.out.println("flavor: ${variant.flavorName}")
        System.out.println("applicationId: ${variant.applicationId}")
        System.out.println("version: ${variant.versionName}(${variant.versionCode})")
        System.out.println("signing: ${variant.signingConfig.name}(${variant.signingConfig.storeFile.name})")

        System.out.println('--- environment ---')
    }
}
```