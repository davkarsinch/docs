---
title: "The verification process"
excerpt: ""
---
Verification of a phone number is performed in two steps: requesting a verification code and verifying the received code. If all of the permissions are provided (see [Permissions](#permissions)), the Verification SDK attempts to automatically verify any matching code that is received through SMS or flash call during the verification process. This means that during a flash call verification, the Verification SDK will automatically hang up the incoming call if (and only if) it matches the expected pattern. During an SMS verification, if enabled, any received messages text will be matched against the template, and if it matches, the code will be extracted and automatically sent to the Sinch backend. The Verification SDK will callback to the `VerificationListener` during the process, see [VerificationListener](#verificationlistener) for more information.

If the permissions needed for automatic verification are not provided, the `verify` method on the `Verification` object should be invoked with a user-supplied verification code to verify the phone number.
[block:callout]
{
  "type": "info",
  "title": "Note",
  "body": "The SDK will abort verification process and trigger the \\`\\`onVerificationFailed\\`\\` callback after a certain time, typically around 20 seconds. However, the SDK will continue to listen for calls for an additional time after the verification has failed, in order to intercept and report any possible late calls."
}
[/block]
## Permissions

The `android.permission.INTERNET` permission is required for the Verification SDK to work. To handle flash call verification automatically, use `READ_PHONE_STATE`, and `CALL_PHONE`. `CALL_PHONE` is used to automatically hang up the call.

The full list of permissions:
[block:code]
{
  "codes": [
    {
      "code": "<uses-permission android:name=\"android.permission.INTERNET\" />\n<uses-permission android:name=\"android.permission.ACCESS_NETWORK_STATE\" />\n<uses-permission android:name=\"android.permission.READ_PHONE_STATE\" />\n<uses-permission android:name=\"android.permission.CALL_PHONE\" />",
      "language": "xml"
    }
  ]
}
[/block]
### Permissions handling on API levels 23 or later.

If your application targets API 23 or later the permissions have to be requested at runtime, see [Runtime Permissions](https://developer.android.com/training/permissions/requesting.html).   It is recommended to ask the user for corresponding permissions just before initiating verification. See the sample application for an example.   If the verification SDK fails to intercept the code automatically due to missing permissions, the `onVerificationFailed` callback will be executed with an instance of `CodeInterceptionException`. In this case it is still possible to proceed with verification by asking the user to enter the code manually.

## Flash Call Verification

To initiate a flash call verification, start by creating a `Verification` object through `SinchVerification.createFlashCallVerification`, then start the verification process by invoking `initiate()` on the `Verification` object. This method can be called multiple times, for example if another call should be placed.
[block:code]
{
  "codes": [
    {
      "code": "Config config = SinchVerification.config().applicationKey(APPLICATION_KEY).context(getApplicationContext()).build();\nVerificationListener listener = new MyVerificationListener();\nString defaultRegion = PhoneNumberUtils.getDefaultCountryIso();\nString phoneNumberInE164 = PhoneNumberUtils.formatNumberToE164(phoneNumberString, defaultRegion);\nVerification verification = SinchVerification.createFlashCallVerification(config, phoneNumberInE164, listener);\nverification.initiate();",
      "language": "java"
    }
  ]
}
[/block]

[block:callout]
{
  "type": "warning",
  "title": "Important",
  "body": "It is important to pass `ApplicationContext` to the verification config builder, as the verification object might outlive the activity or fragment it is instantiated from."
}
[/block]
## SMS Verification

To initiate an SMS verification, start by creating a `Verification` object through `SinchVerification.createSmsVerification`, then start the verification process by invoking `initiate()` on the `Verification` object. This method can be called multiple times, for example, if another SMS should be sent.
[block:code]
{
  "codes": [
    {
      "code": "Config config = SinchVerification.config().applicationKey(APPLICATION_KEY).context(getApplicationContext()).build();\nVerificationListener listener = new MyVerificationListener();\nString defaultRegion = PhoneNumberUtils.getDefaultCountryIso();\nString phoneNumberInE164 = PhoneNumberUtils.formatNumberToE164(phoneNumberString, defaultRegion);\nVerification verification = SinchVerification.createSmsVerification(config, phoneNumberInE164, listener);\nverification.initiate();",
      "language": "java"
    }
  ]
}
[/block]
A `VerificationListener` object must be provided.

### Automatic code extraction from SMS

Due to newly introduced [restrictions for accessing SMS logs](https://support.google.com/googleplay/android-developer/answer/9047303?hl=en), automatic code extraction now relies on Google Play Services API. To enable it in your project:

1.  Calculate application hash from your release key: ``keytool -exportcert -alias KEY_ALIAS -keystore MY_KEY.jks | xxd -p | tr -d "[:space:]" | echo -n MY.APPLICATION.PACKAGE `cat` | sha256sum | tr -d "[:space:]-" | xxd -r -p | base64 | cut -c1-11``
Example result: `XtNB5qNssI/`

2.  Include generated application hash in SinchVerification config: `Config config = SinchVerification.config() .applicationKey(APPLICATION_KEY) .appHash(APPLICATION_HASH) .context(getApplicationContext()) .build();`

3.  In your project’s `build.gradle`, add dependency for `play-services-auth-api-phone`: `dependencies{ implementation "com.google.android.gms:play-services-auth-api-phone:11.0.0" }`
    Minimal supported version is `11.0.0`.

That’s it\! When this configuration is applied, SDK will be able to automatically parse incoming verification messages across all Android versions without the need of declaring SMS permissions in app’s manifest.

### Set the language of an SMS verification

It is possible to specify the content language for SMS verification. This is specified via a list of [IETF](https://tools.ietf.org/html/rfc3282) language tags in order of priority. If the first language is not available, the next one will be selected and so forth. The default is ‘en-US’. The actual language selected can be retrieved by calling `selectedLanguage()` method of the `InitiationResult` object passed to `onInitiated()` callback. For example:
[block:code]
{
  "codes": [
    {
      "code": "List<String> languages = new ArrayList();\nlanguages.add(\"es-ES\");\nVerification verification = SinchVerification.createSmsVerification(config, phoneNumberInE164, null, languages, new VerificationListener() {\n    @Override\n    public void onInitiated(InitiationResult result) {\n        // Verification initiated\n        Log.i(\"Selected sms language: \" + result.selectedLanguage());\n    }\n    ...\n});",
      "language": "java"
    }
  ]
}
[/block]

[block:callout]
{
  "type": "info",
  "title": "Note",
  "body": "The content language specified can be overridden by carrier provider specific templates, due to compliance and legal requirements, such as [US shortcode requirements (pdf)](https://www.usshortcodes.com/info/static/docs/CTIA%20Short%20Code%20Monitoring%20Handbook%20v1.5.2.pdf)."
}
[/block]
## Verification listener

The `VerificationListener` provides callbacks during the verification process. If initiation is successful, the `onInitiated()` callback runs and the verification code interceptor is started automatically. If initiation fails, the `onInitiationError()` callback runs and the exception describing the error is passed. If code verification is successful, the `onVerified()` callback is called. If verification fails, `onVerificationError()` callback runs and the exception describing the error is passed.
[block:code]
{
  "codes": [
    {
      "code": "VerificationListener listener = new VerificationListener() {\n    @Override\n    public void onInitiated(InitiationResult result) {\n        // Verification initiated\n    }\n    @Override\n    public void onInitiationFailed(Exception e) {\n        if (e instanceof InvalidInputException) {\n            // Incorrect number provided\n        } else if (e instanceof ServiceErrorException) {\n            // Verification initiation aborted due to early reject feature,\n            // client callback denial, or some other Sinch service error.\n            // Fallback to other verification method here.\n        } else {\n            // Other system error, such as UnknownHostException in case of network error\n        }\n    }\n    @Override\n    public void onVerified() {\n        // Verification successful\n    }\n    @Override\n    public void onVerificationFailed(Exception e) {\n        if (e instanceof InvalidInputException) {\n            // Incorrect number or code provided\n        } else if (e instanceof CodeInterceptionException) {\n            // Intercepting the verification code automatically failed, input the code manually with verify()\n        } else if (e instanceof IncorrectCodeException) {\n            // The verification code provided was incorrect\n        } else if (e instanceof ServiceErrorException) {\n            // Sinch service error\n        } else {\n            // Other system error, such as UnknownHostException in case of network error\n        }\n    }\n}",
      "language": "java"
    }
  ]
}
[/block]
### Phone numbers
[block:callout]
{
  "type": "warning",
  "title": "Important",
  "body": "The phone number should be specified according to the E.164 number formatting (<http://en.wikipedia.org/wiki/E.164>) recommendation and should be prefixed with a <span class=\"title-ref\">+</span>. See the section `Phone numbers <phonenumbers>` for details."
}
[/block]
## Validate code manually

To complete the verification of the phone number, the code should be passed to `verify`. For SMS verification, the code is in the message body. For flash calls, the caller id is the code. Example:
[block:code]
{
  "codes": [
    {
      "code": "verification.verify(code); ",
      "language": "java"
    }
  ]
}
[/block]
The method `verify` may be invoked multiple times. For example, if the verification listener is invoked with an `IncorrectCodeException`, the application may hint to the user that the code was incorrect, let the user adjust it, and call `verify` again on the same verification instance.

## Early reject.

If Sinch knows that verification is most likely to fail, an application can be configured to catch this condition and provide means to fallback fast to other verification methods. In this case the verification listener `onInitiationFailed()` callback will be executed with an instance of `ServiceErrorException`. To enable this feature contact us at <dev@sinch.com>

## Network connectivity errors

The Sinch Verification SDK will try to resend HTTP requests to the Sinch backend if a request failed due to a network related error. The SDK schedules a number of retries for approximately 30 seconds. If sending the HTTP request is still unsuccessful, it eventually invokes the verification listener `onInitiationError` or `onVerificationError` with an exception that indicates the problem, for example, `java.net.UnknownHostException`.

## Pass data to your backend

For each call to `Verification.initiate()`, the Sinch backend can perform a callback to the application backend to allow or disallow an SMS or flashcall being initiated. By using the optional parameter `custom` on `SinchVerification.createFlashCallVerification` and `createSmsVerification`, any unique identifier can be passed from the application to the application backend. The data will be passed as a string. If there is a need for a more complex datatype, it needs to be stringified or encoded before being sent.