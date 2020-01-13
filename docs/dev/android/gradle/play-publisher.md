# Play Publisher

##### Android

[Triple-T/gradle-play-publisher](https://github.com/Triple-T/gradle-play-publisher)

这家伙弄的工具很牛逼，不过有点复杂，而且不适合接入 Flutter

##### Flutter

没办法，只能自己定制一个了

```groovy
//
//android {
//    productFlavors {
//        prod {...}
//    }
//
//    playConfigs {
//        prod {
//            enabled = true
//
//            // 服务账号：主账号创建的才能使用，关联账号创建的不能使用
//            // 设置 -> 开发者账号 -> 用户和权限 -> 邀请新用户 -> 版本管理 -> 管理测试版 & 管理正式版(production)
//            serviceAccountCredentials = file('play/play-publisher.json')
//
//            track = 'internal' // 可选：'internal'、'alpha'、'beta'、'production'，默认：'internal'
////            releaseName = "custom release name" // 默认：variant.versionName
////            releaseNotes = [
////                    'en-US': 'xxx', // 英文
////                    'zh-CN': 'xxx', // 简体中文
////                    'zh-TW': 'xxx', // 繁体中文
////                    'vi': 'xxx', // 越南
////            ] // 默认取上个版本填写的内容
//            releaseStatus = 'draft' // 可选：'draft'、'completed'，默认：'draft'
//        }
//    }
//}
//
//play {
//    enabled = false
//}
//

import com.google.api.client.googleapis.auth.oauth2.GoogleCredential
import com.google.api.client.googleapis.javanet.GoogleNetHttpTransport
import com.google.api.client.googleapis.json.GoogleJsonResponseException
import com.google.api.client.http.FileContent
import com.google.api.client.http.HttpRequest
import com.google.api.client.http.HttpRequestInitializer
import com.google.api.client.json.jackson2.JacksonFactory
import com.google.api.services.androidpublisher.AndroidPublisher
import com.google.api.services.androidpublisher.AndroidPublisherScopes
import com.google.api.services.androidpublisher.model.Apk
import com.google.api.services.androidpublisher.model.Bundle
import com.google.api.services.androidpublisher.model.LocalizedText
import com.google.api.services.androidpublisher.model.Track
import com.google.api.services.androidpublisher.model.TrackRelease

buildscript {
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.2'
        classpath 'com.google.apis:google-api-services-androidpublisher:v3-rev20190910-1.30.1'
    }
}

android {
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}

class Play {
    final String name
    Boolean enabled
    File serviceAccountCredentials
    String track
    String releaseName
    Map<String, String> releaseNotes
    String releaseStatus

    Play(String name = 'default') {
        this.name = name
    }

    void validate() {
        if (enabled == null || !enabled.booleanValue()) {
            return
        }
        if (serviceAccountCredentials == null) {
            throw new IllegalStateException("Play Publisher service account credentials is null for variant '$name'")
        }
        if (track != 'internal' && track != 'alpha' && track != 'beta' && track != 'production') {
            throw new IllegalStateException("Play Publisher track is invalid for variant '$name'")
        }
        if (releaseStatus != 'draft' && releaseStatus != 'completed') {
            throw new IllegalStateException("Play Publisher release status is invalid for variant '$name'")
        }
    }

    // ---

    Play mergeWith(Play other) {
        if (other == null) {
            return this
        }
        def mergePlay = new Play(name == 'default' ? other.name : (other.name == 'default' ? name : "$name${other.name.capitalize()}"))
        mergePlay.enabled = other.enabled != null ? other.enabled : enabled
        mergePlay.serviceAccountCredentials = other.serviceAccountCredentials ?: serviceAccountCredentials
        mergePlay.track = other.track ?: track
        mergePlay.releaseName = other.releaseName ?: releaseName
        mergePlay.releaseNotes = other.releaseNotes ?: releaseNotes
        mergePlay.releaseStatus = other.releaseStatus ?: releaseStatus
        return mergePlay
    }
}

apply plugin: PlayPublisherPlugin

class PlayPublisherPlugin implements Plugin<Project> {

    @Override
    void apply(Project target) {
        target.extensions.create('play', Play.class)
        target.plugins.withId('com.android.application') {
            def basePlay = target.play
            def playConfigs = target.container(Play.class)
            target.android.extensions.playConfigs = playConfigs
            target.android.applicationVariants.whenObjectAdded { variant ->
                def mergePlay = null
                def flavorPlays = variant.productFlavors?.stream().map{flavor -> playConfigs.findByName(flavor.name)}.collect().toList() ?: Collections.emptyList()
                def buildTypePlay = playConfigs.findByName(variant.buildType.name)
                if (buildTypePlay == null && (variant.buildType.name == 'debug' || variant.buildType.name == 'profile')) {
                    buildTypePlay = new Play(variant.buildType.name)
                    buildTypePlay.enabled = false
                }
                // buildType > flavor > base
                List<Play> plays = new ArrayList<>()
                plays.add(basePlay)
                plays.addAll(flavorPlays)
                plays.add(buildTypePlay)
                for (def play in plays) {
                    if (mergePlay == null) {
                        mergePlay = play
                    } else {
                        mergePlay = mergePlay.mergeWith(play)
                    }
                }

                mergePlay?.validate()

                variant.assemble.doLast {
                    publishToPlay(target, variant, mergePlay, false)
                }

                target.tasks.findByPath("bundle${variant.name.capitalize()}").doLast {
                    publishToPlay(target, variant, mergePlay, true)
                }
            }
        }
        target.afterEvaluate {
            if (!target.plugins.hasPlugin('com.android.application')) {
                target.logger.warn("The Android Gradle Plugin was not applied. Gradle Play Publisher will not be configured.")
            }
        }
    }

    void publishToPlay(Project target, variant, Play play, boolean publishAab) {
        if (play == null || play.enabled == null || !play.enabled.booleanValue()) {
            target.logger.info("Gradle Play Publisher is disabled for variant '${variant.name}'.")
            return
        }

        if (!variant.signingReady && !variant.outputsAreSigned) {
            target.logger.error("Signing not ready for Gradle Play Publisher. Be sure to specify a signingConfig for variant '${variant.name}'.")
            return
        }

        def publisher = createPublisher(play)
        def editId = publisher.edits().insert(variant.applicationId, null).execute().id

        System.out.println('--- play publisher ---')
        System.out.println("name: ${play.name}")
        System.out.println("channel file: ${play.serviceAccountCredentials}")
        System.out.println("track: ${play.track ?: 'internal'}")
        System.out.println("release name: ${play.releaseName ?: variant.versionName}")
        System.out.println("release status: ${play.releaseStatus ?: 'draft'}")
        def releaseNotes = play.releaseNotes?.collect{note -> new LocalizedText().setLanguage(note.key).setText(note.value)}.collect().toList()
        if (releaseNotes == null || releaseNotes.empty) {
            List<Track> tracks = publisher.edits().tracks().list(variant.applicationId, editId).execute().tracks
            if (tracks != null && !tracks.empty) {
                releaseNotes = tracks.first().releases?.first().releaseNotes
            }
        }
        if (releaseNotes != null && !releaseNotes.empty) {
            System.out.println("release notes: \n${releaseNotes.collect{note -> "\t${note.language}: ${note.text}"}.join('\n')}")
        }
        System.out.println("Publish ${publishAab ? 'App Bundle' : 'APK'} to Google Play Console")
        System.out.println("package name: ${variant.applicationId}")
        if (publishAab) {
            File aabFile = null
            if (variant.productFlavors != null && !variant.productFlavors.empty) {
                aabFile = new File("${target.buildDir}/outputs/bundle/${variant.name}/app-${variant.productFlavors.stream().map{flavor -> flavor.name}.collect().join('-')}-${variant.buildType.name}.aab")
            }
            if (aabFile == null || !aabFile.exists()) {
                aabFile = new File("${target.buildDir}/outputs/bundle/${variant.name}/app.aab")
            }
            System.out.println("aab file path: ${aabFile.path}")

            try {
                // upload
                def uploadAab = publisher
                        .edits()
                        .bundles()
                        .upload(variant.applicationId, editId, new FileContent("application/octet-stream", aabFile))
                Bundle bundle = uploadAab.execute()
                System.out.println('upload aab success')

                // assign
                Track track = new Track()
                track.track = play.track ?: 'internal'
                TrackRelease release = new TrackRelease()
                release.name = play.releaseName ?: variant.versionName
                release.versionCodes = Arrays.asList(bundle.versionCode.toLong()) // Arrays.asList(variant.versionCode)
                release.releaseNotes = releaseNotes
                release.status = play.releaseStatus ?: 'draft'
                track.releases = Arrays.asList(release)
                publisher.edits()
                        .tracks()
                        .update(variant.applicationId, editId, track.track, track)
                        .execute()
                System.out.println("assign aab to ${track.track}-${release.name}(${bundle.versionCode})")

                // commit
                publisher.edits()
                        .commit(variant.applicationId, editId)
                        .execute()
                System.out.println('commit success')
            } catch (GoogleJsonResponseException e) {
                throw e
            }
        } else {
            File apkFile = variant.outputs.first().outputFile as File
            System.out.println("apk file path: ${apkFile.path}")
            try {
                // upload
                def uploadApk = publisher
                        .edits()
                        .apks()
                        .upload(variant.applicationId, editId, new FileContent("application/vnd.android.package-archive", apkFile))
                Apk apk = uploadApk.execute()
                System.out.println('upload apk success')

                // assign
                Track track = new Track()
                track.track = play.track ?: 'internal'
                TrackRelease release = new TrackRelease()
                release.name = play.releaseName ?: variant.versionName
                release.versionCodes = Arrays.asList(apk.versionCode.toLong()) // Arrays.asList(variant.versionCode)
                release.releaseNotes = releaseNotes
                release.status = play.releaseStatus ?: 'draft'
                track.releases = Arrays.asList(release)
                publisher.edits()
                        .tracks()
                        .update(variant.applicationId, editId, track.track, track)
                        .execute()
                System.out.println("assign aab to ${track.track}-${release.name}(${apk.versionCode})")

                // commit
                publisher.edits()
                        .commit(variant.applicationId, editId)
                        .execute()
                System.out.println('commit success')
            } catch (GoogleJsonResponseException e) {
                throw e
            }
        }

        System.out.println('--- play publisher ---')
    }

    AndroidPublisher createPublisher(Play play) {
        def transport = GoogleNetHttpTransport.newTrustedTransport()
        def factory = JacksonFactory.getDefaultInstance()
        def credential = GoogleCredential
                .fromStream(new FileInputStream(play.serviceAccountCredentials), transport, factory)
                .createScoped(Arrays.asList(AndroidPublisherScopes.ANDROIDPUBLISHER))
        return new AndroidPublisher.Builder(transport, factory, new HttpRequestInitializer() {
                    void initialize(HttpRequest request) throws IOException {
                        credential.initialize(request.setReadTimeout(0))
                    }
                })
                .setApplicationName('Play Publisher')
                .build()
    }
}
```