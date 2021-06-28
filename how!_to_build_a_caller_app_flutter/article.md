# How !to make a calling app in Flutter for Android devices

*Disclaimer- That `!` in the title is not a typo, read it as `not`. This article is not a tutorial about how to make a whatsapp or skype type calling app, rather it is an article about how I fell short of making such an app using flutter.*

I have been trying to make a voice calling app using Flutter. You will find many articles and videos about how easy peasy it is to integrate Twilio or Agora into a flutter app, but I could not find one resource which dived deep into making a true calling app in Flutter. What do I mean by ***true calling app*** ?

## What a true calling app should have

![On receiving data notification for call](https://i.ibb.co/jVwkGQk/mermaid-diagram-20210614214920.png)

<p style="text-align: center;"> On receiving data notification for call </p>

- The app should work even when it is in background, or terminated even.
- The app should work over a keyguard(i.e- the lockscreen)
- If the user is using the phone, a [time sensitive notification](https://developer.android.com/training/notify-user/time-sensitive) should pop up.
- When accepted, this should launch into its own UI, where you see the call details like caller id, duration etc.
- This notification should be a full screen notificaion so that we can see the call interface when the device is locked and not in use.
- During a call, when the user presses the back button, there should be a notification of a foreground service, which shows the info about ongoing call, and has options for disconnecting.
- We can use either the inbuilt calling app, or we can implement our own screen for incoming/outgoing calls and for call in progress.
- I wanted to build it without writing any platform specific code, or just the least amount of code.

## Building the app

I am using the [agora_rtc_engine](https://pub.dev/packages/agora_rtc_engine) plugin for making voice calls. You can find a tutorial on their [official docs here](https://docs.agora.io/en/Voice/start_call_audio_flutter?platform=Flutter). If we implement just what the tutorial says, we will be joining a voice channel as soon as we launch the app, and anyone having the same app can speak into and listen to this voice call.

So the voice call part is taken care of pretty easily. Now for how to get the behaviour of getting an actual phone call(getting a Caller ID screen, an accept/reject screen).

### Take #1

I set up a FCM in flutter to receive data messages. You can read about difference between notification messages and data messages in my [stackoverflow answer here](https://stackoverflow.com/a/67968481/14371894). I went with data messages because I wanted to be in control of the notification I generate.

My next plan was to use [flutter_local_notifications](https://pub.dev/packages/flutter_local_notifications#custom-notification-icons-and-sounds) plugin and generate a notification. This plugin also has a full screen intent option, which when set to true can show your app when when phone is locked. But there are certain properties the notification should have.So I would listen to a data message, in both foreground and background/terminated state. Upon receiving one, I will generate a notification with full screen intent as true.

#### Properties of the notification

- They should be non dismissable by swipe action. Only way it is dismissed if the ring is over or the user rejects the call.

- The notification should have accept and reject buttons. Depending on the response, we should open either the ongoing call UI or reject the call.

- There should be a non dismissable notification during the call, which will show some options like mute, disconnect, etc, along with caller id, duration etc.

#### The fault in our notifications

- `flutter_local_notifications` does not have any solution for time sensitive notifications.

- It also does not have any solution for having action buttons in the notification

- `awesome_notifications` has a solution for action button, but no solution for time sensitive notification.

- It also does not support full screen notifications.

![Always has been eh](https://i.imgflip.com/5d8f0n.jpg)
<p style="text-align: center;"> Always has been eh? </p>


Navigating to a particular screen upon tapping the notification is rather easy, as there are callbacks available for times when the app has been opened due to a notification. For this though you need to use a notificaication message, in which the notification will be handled directly by the plugin. If you want to implement it in data messages, you will have to write some platform dependent code as well.

### Take #2

I tried handling the incoming calls like a gentleman!
Upon further digging I found out that apps like Whatsapp, Skype, Insta, Zoom etc handle incoming calls and outgoing calls using something called [callkit in iOS](https://developer.apple.com/documentation/callkit) and [ConnectionService in Android](https://developer.android.com/reference/android/telecom/ConnectionService). Plugins already exist for this in flutter, namely [callkeep](https://pub.dev/packages/callkeep) and [flutter_callkeep](https://pub.dev/packages/callkeep). 

These plugins, when asked to show a call, mimic the exact behavior as a normal telecom call would, using the same notifications, showing the same UI. Each interaction, like accepting a call, rejecting a call, toggling loudspeaker, holding a call etc all have their callbacks. Seems pretty straightforward, right?

#### Callkeep couldn't keep up

There is a bug in both the packages.

- When you start a call, and reject a call then everything works fine.

- When you accept the call though, the Call UI hangs all of a sudden. Fron the logs it seems like it is starting a lot of calls internally, and they all clash with each other.

The plugins are still a bit usable in Android Pie, but completely unusable in Android 10.

![Why are we here, just to suffer?](https://i.imgflip.com/5d8fkt.jpg)
<p style="text-align: center;"> Why are we here, just to suffer? </p>

### Take #3

A hacky solution (platform side code included).

Our pain problem with notification is that we can't have action buttons + time sensitive + full screen notifications together. We can all compensate for this by generating our own notification using [platform channel](https://flutter.dev/docs/development/platform-integration/platform-channels).

#### The plan

- Write a function that can generate a notification of our requirements, a good example is [time sensitive notification](https://developer.android.com/training/notify-user/time-sensitive) example here.

- You can assign a pending intent to this intent for full screen notification. Flutter apps have only one activity. To get the reference to this activity, you can follow this snippet.

``` java
    /// This code snippet is taken from flutter_local_notification plugin
    private static Intent getLaunchIntent(Context context) {
        String packageName = context.getPackageName();
        PackageManager packageManager = context.getPackageManager();
        return packageManager.getLaunchIntentForPackage(packageName);
    }
```

- Open `MainActivity` as soon as the user either taps on the notification or accepts the call.

- Check whether the app was called because of a notification tap by invoking some MethodChannel right as the app starts. Depending on the result, you can either navigate to the app as usual or to the Call UI screen.

- If you find it difficult to determine if the app was started from a notification or not, using something like Shared Preferences. Store a key-value pair signifying that app was started in notification inside the BroadcastListener. Once the app has launched and a MethodChannel is involed asking whether app was launched from notification, reset it. P.S- This is a hacky solution, better solutions exist.

- As soon as the app is launched, spawn another service, this time a [foreground service](https://developer.android.com/guide/components/foreground-services). Every foreground service **MUST** have a notification associated with it. We can use this notification to signify an ongoing call, regardless of whether is in the app or outside it. This service runs until the user has disconnected the call.

![Same energy as our solution](https://i.imgflip.com/5d8aqg.jpg)
<p style="text-align: center;"> Same energy as our solution</p>

### Take #4

What I feel is the legit solution!

Previously in the article I mentioned something about [ConnectionService](https://developer.android.com/reference/android/telecom/ConnectionService). This is the recommended way for any app that either

- Can make phone calls (VoIP or otherwise) and want those calls to be integrated into the built-in phone app. Referred to as a *system managed* ConnectionService.

- Are a standalone calling app and don't want their calls to be integrated into the built-in phone app. Referred to as a *self managed* ConnectionService.

What both `callkeep` and `flutter_callkeep` plugins offered is system managed ConnectionService. What we need is self managed. The main issue here is resources, or lack thereof. The only resource I could find is [this guide](https://developer.android.com/guide/topics/connectivity/telecom/selfManaged#no-active). There are not any tutorials/blogs about how to implement this. We can learn how to implement this by looking at some existing projects though, a good point is [AgoraIO-Usercase/Video-Calling](https://github.com/AgoraIO-Usecase/Video-Calling) repository.

Getting this functionality all wrapped in a flutter plugin is the best solution we can manage with flutter in my opinion. What I believe personally though is that implementing it in native is the easiest way out. That way we do not have to worry about handling communication between native android and flutter.

> *Flutter is a cross-platform UI **toolkit** that is designed to allow code > reuse across operating systems such as iOS and Android, while also 
> allowing applications to interface directly with underlying platform 
> services*. - [Flutter architectural overview](https://flutter.dev/docs/resources/architectural-overview)

Google considers Flutter as a UI toolkit, and personally so do I. Sometimes we may have to implement some framework level functionalities(like handling telephony in this case) just so that we can show a call accept/reject screen in flutter. This can be pretty straightforward in native android. So the question lies with the developer, *is it worth the effort?*

!['tis complicated](https://i.imgflip.com/5d8e1w.jpg)

P.S - This article is everything I could find and make sense of while researching about this for 3 days straight. I decided to write this article because there are a lot of people on StackOverflow and Github Issues who do not have an answer to this question. While I have not been able to provide an answer, yet, I hope to make the question clearer.

I am not a pro developer, so if you find any discrepancies, feel free to comment, the whole point of this article to bring whatever little exists about this on the internet at a single place!