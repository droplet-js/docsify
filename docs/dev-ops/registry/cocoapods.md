# CocoaPods

##### 安装

```shell
不作赘述
```

##### 搭建私有仓库

```shell
```

##### 常用命令

更新 repo
```shell
pod repo update
```

搜索（"q"键退出搜索）
```shell
pod search xxx
```
```shell
[!] Unable to find a pod with name, author, summary, or description matching CYXCyclingPlayerView
rm ~/Library/Caches/CocoaPods/search_index.json
```

##### 注册

命令：
```shell
pod trunk register EMAIL [YOUR_NAME]
```

例子：
```shell
pod trunk register v7lin@qq.com v7lin
[!] Please verify the session by clicking the link in the verification email that has been sent to v7lin@qq.com
```

##### 查看

命令：
```shell
pod trunk me
```

例子：
```shell
pod trunk me
  - Name:     v7lin
  - Email:    v7lin@qq.com
  - Since:    February 22nd, 00:17
  - Pods:     None
  - Sessions:
    - February 22nd, 00:17 - June 30th, 00:17. IP: 112.111.40.24
```

##### Travis-CI

[示例](https://github.com/v7lin/FakeXinGePush)
```yaml
# references:
# * https://www.objc.io/issues/6-build-tools/travis-ci/
# * https://github.com/supermarin/xcpretty#usage

language: objective-c
osx_image: xcode10.1

cache: cocoapods

podfile: Example/Podfile

before_install:
- gem install cocoapods # Since Travis is not always on latest version

install:
- pod install --project-directory=Example

before_script:
- echo "orz"

script:
- set -o pipefail
- travis_retry xcodebuild test -enableCodeCoverage YES -workspace Example/FakeXinGePush.xcworkspace -scheme FakeXinGePush-Example -sdk iphonesimulator ONLY_ACTIVE_ARCH=NO | xcpretty
- pod lib lint

after_script:
- echo "orz"
```

##### Travis-CI 自动发布 Cocoapods

获取 Cocoapods Token
```shell
pod trunk register v7lin@qq.com --description='Travis CI'
[!] Please verify the session by clicking the link in the verification email that has been sent to v7lin@qq.com
```

打开邮箱，按提示验证
```shell
Hi v7lin,

Please confirm your CocoaPods session by clicking the following link:

https://trunk.cocoapods.org/sessions/verify/c3187166
If you did not request this you do not need to take any further action.

Kind regards, the CocoaPods team
```

验证完毕后，可以从 ~/.netrc 获取相关信息
```shell
grep -A2 'trunk.cocoapods.org' ~/.netrc
machine trunk.cocoapods.org
  login v7lin@qq.com
  password ${16进制字符串}
```

travis-ci repo 设置
```
COCOAPODS_TRUNK_TOKEN=${16进制字符串}
```

.travis.yml
```yaml
# references:
# * https://www.objc.io/issues/6-build-tools/travis-ci/
# * https://github.com/supermarin/xcpretty#usage

language: objective-c
osx_image: xcode10.1

cache: cocoapods

podfile: Example/Podfile

before_install:
- gem install cocoapods # Since Travis is not always on latest version

install:
- pod install --project-directory=Example

before_script:
- echo "orz"

jobs:
  include:
    - stage: check
      script:
      - pod lib lint
    - stage: test
      script:
      - travis_retry set -o pipefail && xcodebuild test -enableCodeCoverage YES -workspace Example/FakeXinGePush.xcworkspace -scheme FakeXinGePush-Example -sdk iphonesimulator ONLY_ACTIVE_ARCH=NO | xcpretty
    - stage: publish
      script:
      - pod trunk push FakeXinGePush.podspec
      if: tag IS present

after_script:
- echo "orz"
```