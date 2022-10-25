## .NET MAUI Push Notification for iOS
Follow manual below to enable **Firebase** Cloud Messaging for .NET **MAUI** project on **iOS** platform.
### Preparation
I consider you have set up the Apple Provisioning profile properly. This means you set up Certificate, Identifier and Profile (especially Distribution - App Store) on [Apple Developer Site](developer.apple.com/account/resources). Do not forget to enable "Push Notification" on Identifier page of your application and use the same Bundle ID as you are using in your app. On Keys page [configure the APN Key](https://www.kodeco.com/20201639-firebase-cloud-messaging-for-ios-push-notifications#toc-anchor-003) - check Apple Push Notifications service (APNs) - download it and remember Key ID. You will need it in the [Firebase Console](https://console.firebase.google.com).

I also consider all preparation work on Firebase Console is done according to [manual](https://support.google.com/firebase/answer/7015592#ios) and you have the GoogleService-Info.plist file ready (but not added to your project). Do not forget to [register APNs Authentication Key](https://firebase.google.com/docs/cloud-messaging/ios/client#upload_your_apns_authentication_key) on page Project setting, tab "Cloud Messaging", part "Apple app configuration".
### Project changes
Having all this done let's open solution with MAUI project in **Visual Studio 2022**. In my case version 17.3.6, .NET SDK 6.0.402, MAUI workload 6.0.541, build server macOS Monterey 12.6 with Xcode 14.0.1.

Check target iOS Framework is "net6.0-ios15.4" and then add nuget package "Xamarin.Firebase.iOS.CloudMessaging" version 8.10.0.2 in traditional way using Nuget Package Manager.

You can run into an issue with a long path. Package installation might fail with exception 
"System.IO.DirectoryNotFoundException: Could not find a part of the path 'C:\Users\\...\\.nuget\packages\xamarin.firebase.ios.installations\\...\\FirebaseInstallations-umbrella.h'."
But you can solve it by adding Nuget.config file into solution folder with this content:
```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <config>
    <clear />
    <add key="globalPackagesFolder" value="C:\Nuget" />
  </config>
</configuration>
```

Add Entitlements.plist file into Platforms\iOS folder with this content:
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>aps-environment</key>
	<string>development</string>
</dict>
</plist>
```
Add into Info.plist file these rows:
```
<key>UIBackgroundModes</key>
<array>
	<string>remote-notification</string>
</array>
```
You can add into project file these lines:
```
<ItemGroup Condition="'$(TargetFramework)'=='net6.0-ios15.4'">
	<BundleResource Include="Platforms\iOS\GoogleService-Info.plist" />
</ItemGroup>
```
... and let the Firebase package to read settings from generated GoogleService-Info.plist but let's keep it simpler and do not add this file to project.

Instead, change file Platforms\iOS\\**AppDelegate.cs** like this:
```
using Firebase.CloudMessaging;
using Foundation;
using UIKit;
using UserNotifications;

namespace ...

[Register("AppDelegate")]
public class AppDelegate : MauiUIApplicationDelegate, IUNUserNotificationCenterDelegate, IMessagingDelegate
{
    protected override MauiApp CreateMauiApp() => MauiProgram.CreateMauiApp();

	public override bool FinishedLaunching(UIApplication application, NSDictionary launchOptions)
	{
		...

        var result = base.FinishedLaunching(application, launchOptions);

        var authOptions = UNAuthorizationOptions.Alert | UNAuthorizationOptions.Badge | UNAuthorizationOptions.Sound;
        UNUserNotificationCenter.Current.RequestAuthorization(authOptions, (granted, error) =>
        {
            this.Log($"RequestAuthorization: {granted}" + (error != null ? $" with error: {error.LocalizedDescription}" : string.Empty));

            if (granted && error == null)
            {
                this.InvokeOnMainThread(() =>
                {
                    UIApplication.SharedApplication.RegisterForRemoteNotifications();
                    this.InitFirebase();
                });
            }
        });

        return result;
    }

    private void InitFirebase()
    {
        this.Log($"{nameof(this.InitFirebase)}");

        try
        {
            var options = new Firebase.Core.Options("[GOOGLE_APP_ID]", "[GCM_SENDER_ID]");
            options.ApiKey = "[API_KEY]";
            options.ProjectId = "[PROJECT_ID]";
            options.BundleId = "[BUNDLE_ID]";
            options.ClientId = "[CLIENT_ID]";

            Firebase.Core.App.Configure(options);
        }
        catch (Exception x)
        {
            this.Log("Firebase-configure Exception: " + x.Message);
        }

        UNUserNotificationCenter.Current.Delegate = this;

        if (Messaging.SharedInstance != null)
        {
            Messaging.SharedInstance.Delegate = this;
            this.Log("Messaging.SharedInstance SET");
        }
        else
        {
            this.Log("Messaging.SharedInstance IS NULL");
        }
    }

    // indicates that a call to RegisterForRemoteNotifications() failed
    // see developer.apple.com/documentation/uikit/uiapplicationdelegate/1622962-application
    [Export("application:didFailToRegisterForRemoteNotificationsWithError:")]
    public void FailedToRegisterForRemoteNotifications(UIApplication application, NSError error)
    {
        this.Log($"{nameof(FailedToRegisterForRemoteNotifications)}: {error?.LocalizedDescription}");
    }

    // this callback is called at each app startup
    // it can be called two times:
    //   1. with old token
    //   2. with new token
    // this callback is called whenever a new token is generated during app run
    [Export("messaging:didReceiveRegistrationToken:")]
    public void DidReceiveRegistrationToken(Messaging messaging, string fcmToken)
    {
        this.Log($"{nameof(DidReceiveRegistrationToken)} - Firebase token: {fcmToken}");

        //Utils.RefreshCloudMessagingToken(fcmToken);
    }

    // the message just arrived and will be presented to user
    [Export("userNotificationCenter:willPresentNotification:withCompletionHandler:")]
    public void WillPresentNotification(UNUserNotificationCenter center, UNNotification notification, Action<UNNotificationPresentationOptions> completionHandler)
    {
        var userInfo = notification.Request.Content.UserInfo;

        this.Log($"{nameof(WillPresentNotification)}: " + userInfo);

        // tell the system to display the notification in a standard way
        // or use None to say app handled the notification locally
        completionHandler(UNNotificationPresentationOptions.Alert);
    }

    // user clicked at presented notification
    [Export("userNotificationCenter:didReceiveNotificationResponse:withCompletionHandler:")]
    public void DidReceiveNotificationResponse(UNUserNotificationCenter center, UNNotificationResponse response, Action completionHandler)
    {
        this.Log($"{nameof(DidReceiveNotificationResponse)}: " + response.Notification.Request.Content.UserInfo);
        completionHandler();
    }

    void Log(string msg)
    {
        ...
    }
}
```
... and change Options values to correct ones from your GoogleService-Info.plist file...
### Build
Now, we can build the project!

Beware, Firebase iOS CloudMessaging works **only in Release mode**. In Debug mode Configure method fails with message "Could not create an native instance of the type 'Firebase.Core.Options': the native class hasn't been loaded", no matter you test on physical device with properly set manual Provisioning profile. So, we can test messaging using TestFlight or AdHoc distribution only.
### Test
The easiest way for sending test notification messages is to use Firebase Console. Just open Cloud Messaging in Engage menu and tap "Send your first message" button. Fill in "Notification text" field and tap "Send test message", fill in token in field "Add an FCM registration token" (you got it from log file of your iOS app) and tap "Test" button.
### Recommended links
- [FirebasePushNotificationPlugin - Firebase Setup](https://github.com/CrossGeeks/FirebasePushNotificationPlugin/blob/master/docs/FirebaseSetup.md)
- [GoogleApisForiOSComponents - Firebase Cloud Messaging on iOS](https://github.com/xamarin/GoogleApisForiOSComponents/blob/main/docs/Firebase/CloudMessaging/GettingStarted.md)
- [Plugin.Firebase - Read me](https://github.com/TobiasBuchholz/Plugin.Firebase/blob/master/README.md)
- [FirebasePushNotificationPlugin - Read me](https://github.com/CrossGeeks/FirebasePushNotificationPlugin#readme)
