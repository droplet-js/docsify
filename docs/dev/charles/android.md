# Android

### Android 7及以上

* 新增 res/xml/network_security_config.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true">
    </base-config>

    <debug-overrides>
        <trust-anchors>
            <!--<certificates src="system" />-->
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

* 修改 AndroidManifest.xml

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application
        android:networkSecurityConfig="@xml/network_security_config">
    </application>
</manifest>
```