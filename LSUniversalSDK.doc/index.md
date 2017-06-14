## Integration

### Dependencies

To add this SDK to your project, clone the framework from Github using a tag:

```
git clone https://github.com/sightcall/iOS-UniversalSDK.git -t <YOUR TAG>
```

Then add `LSUniversalSDK.framework` in your Xcode project (See [Apple Documentation](https://developer.apple.com/library/ios/recipes/xcode_help-project_editor/Articles/AddingaLibrarytoaTarget.html)).

### Common link errors

##### Bitcode

The Framework is not compiled with bitcode. That means you may see an error like:
```
ld: 'LSUniversalSDK.framework/LSUniversalSDK' does not contain bitcode. You must rebuild it with bitcode enabled (Xcode setting ENABLE_BITCODE), obtain an updated library from the vendor, or disable bitcode for this target.
```

In order to fix that, you must disable Bitcode generation in your app. See your project Build Settings (in the `Build Options`, the `Enable Bitcode` should be `No`).


##### Architecture

The Framework is not compiled for `x86`/`x86_64` architectures. That means you may see errors like:

```
ld: warning: ignoring file LSUniversalSDK.framework/LSUniversalSDK, missing required architecture i386 in file LSUniversalSDK.framework/LSUniversalSDK (2 slices)
```

In order to fix that, you must compile for a device target. Either build for `Generic iOS Device` or connect a device and compile for it.

This means the USDK can't run on a `simulator`.

### Usage

To start a connection, create a `LSUniversal` variable and retain it. Then call `startWithString:` with your start URL in string form (see `-[NSURL absoluteString]` for more info).


For example, let's say your application received a URL through your UIApplication delegate's `application:openURL:sourceApplication:annotation:` method:

```objc
- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url sourceApplication:(NSString *)sourceApplication annotation:(id)annotation
{
    //yourSDKPointer is a pointer to the LSUniversal instance, that was instantiated somewhere
    //to start the connection, you would simply call:
    [yourSDKPointer startWithString: [url absoluteString]];
}
```

### iOS Privacy

The SDK requires the application to declare a few Privacy keys in its plist"


| Key                                      | Description |
| --- | --- |
| `Privacy - Photo Library Usage Description`|  During the call, the user can share picture and video from the devices library |
| `Privacy - Camera usage decription`      | What the app does with the camera |
| `Privacy - Microphone Usage Description` | What the app does with the microphone |



### Background

The app integrating the SDK should declare the `Audio, AirPlay and Picture and Picture` background mode.

### Delegations

#### Connection and Calls

The SDK offers a delegate with three callbacks. Those callbacks informs the application that the connection status changed, that a call ended and that an error occured.

To register a delegate, simply set the LSUniversal variable [mySDKPointer delegate] after intializing it:

```objc
id<LSUniversalDelegate> yourDelegatePointer = [[YourDelegateType alloc] init];
mySDKPointer.delegate = yourDelegatePointer;
```

The delegate is notified through those 3 methods:


```objc
[yourDelegatePointer connectionEvent: ] //Connection Status update event
[yourDelegatePointer connectionError: ] //Connection Errors
[yourDelegatePointer callReport: ] //Calls end report

```

For example, your delegate can be declared as such:

```objc
@interface YouDelegateType: NSObject <LSUniversalDelegate>

@end


@implementation YourDelegateType
- (void)connectionEvent:(lsConnectionStatus_t)status
{
    switch (status) {
        case lsConnectionStatus_idle: break;
        case lsConnectionStatus_connecting: break;
        case lsConnectionStatus_active: break;
        case lsConnectionStatus_calling: break;
        case lsConnectionStatus_callActive: break;
        case lsConnectionStatus_disconnecting: break;
    }
}

- (void)connectionError:(lsConnectionError_t)error
{

}

- (void)callReport:(lsCallReport_s)callEnd
{

}
@end

```

In addition to those mandatory callbacks, optional callbacks are offered to display informations about the platform used, your queue position if you are connected to an ACD system and if a survey is to be displayed after a call.

Please note that the callbacks may not be on the main thread.

### Call UI

A ViewController is available at `mySDKPointer.callViewController`. This VC is the controller of the call UI. An exemple to display it would be something like:

```objc
- (void)connectionEvent:(lsConnectionStatus_t)status
{
    switch (status) {
        case lsConnectionStatus_idle: break;
        case lsConnectionStatus_connecting: break;
        case lsConnectionStatus_active: break;
        case lsConnectionStatus_calling: break;
        case lsConnectionStatus_callActive: 
        {
            dispatch_async(dispatch_get_main_queue(), ^{
                [yourPresentationController presentViewController:mySDKPointer.callViewController animated:YES completion:nil];
            });
        }   break;
        case lsConnectionStatus_disconnecting: break;
    }
}

```

On call end, simply dismiss the ViewController.

#### Customization

The call controls buttons appearances can be customized through the `LSCustomizationDelegate`. This delegate is entirely optional. Each method allows you to customize the related button.

Let's say you want to customize the hangup button:

```objc
//declare the customization object
@interface MyCustomizationDelegate: NSObject <LSCustomizationDelegate>
@end

@implementation MyCustomizationDelegate
@end

//somewhere in your code, after the initialization of mySDKPointer
id <LSCustomizationDelegate>myCustomizationDelegate = [[MyCustomizationDelegate alloc] init];
mySDKPointer.customizationDelegate = myCustomizationDelegate;
```

At this point, the SDK will try to inform you every time a button is resized that you can customize it. You will only receive this message if you implement the related button callback.

To customize the hangup button, implement the `customizeHangup:` method in myCustomizationDelegate

```objc
//Somewhere in MyCustomizationDelegate.m

- (void)customizeHangup:(UIButton *)b
{
    dispatch_async(dispatch_get_main_queue(), ^{
        b.backgroundColor = [UIColor colorWithRed:0. green:1. blue:0. alpha:0.3];
        [b setImage:[UIImage imageNamed:@"hangup_image"] forState:UIControlStateNormal]; 
        //hangup_image is an image available in your project
    });
}

```
Each button has an image for state. See the USDK documentation to see the meaning of each button state.

When the call control menu appears on screen, the hangup button will appear customized.


**Note:** the callbacks will only be called if the related buttons should appear.


## Universal SDK ACD information

ACD informations are sent through two optional delegation methods.


```
- (void)acdStatusUpdate:(LSACDQueue_s)update;
- (void)acdAcceptedEvent:(NSString *)agentUID;
```

`update` contains informations pertaining to the ACD current status. If the `update.status` of the request is `ongoing`, the update will inform you about the position in the queue or an ETA. Otherwise, the request is cancelled or invalid and the `update.status` informs you of the reason (service closed, agent unavailable, etc.)

Note that `acdAcceptedEvent:` **may not** be called before the SDK `status` moves to `callActive`.

### Survey
If you have configured a survey for the call end, your LSUniversalDelegate will be notified on call end through the `callSurvey:` method. This optional method is called with a `LSSurveyInfo` object. 

This parameter will tell you if you need to display a popup (if `infos.displayPopup` is equal to YES) and the text in this popup (`infos.popupLabel` and `infos.popupButton`), and give you the URL to the survey (`infos.url`).

```objc
- (void)callSurvey:(LSSurveyInfos *)infos
{
    if (!infos.displayPopup) {
        //open infos.url in a webbrowser
    } else {
        //ask the user if she wants to participate in a survey
    }
}
```

### Callflows
The standard callflow is as such:

![Standard iOS Callflow](./connection_callflow.png "iOS Callflow")

<!--

https://www.websequencediagrams.com/
title Connection callflow
participant BROWSER
participant APP
participant SDK

BROWSER->APP: URL scheme trigger
    APP->SDK: Universal.start(scheme)
    SDK->SDK: parse

alt Invalid URL scheme
    SDK->APP: UniversalConfigurationErrorEvent
else Valid URL scheme
    SDK->APP: connectionEvent: lsConnectionStatus_connecting
    SDK->SDK: connect
    SDK->APP: connectionEvent: lsConnectionStatus_active
    SDK->APP: connectionEvent:lsConnectionStatus_calling
    SDK->SDK: call
    SDK->APP: connectionEvent:lsConnectionStatus_callActive
    Note over APP,SDK:
        The call is now active
        Display the callViewController here
    end note
    SDK->SDK: hangup
    SDK->APP: callReport:
    SDK->APP: connectionEvent:lsConnectionStatus_disconnecting
    SDK->SDK: disconnect
    SDK->APP: connectionEvent:lsConnectionStatus_idle
end
!-->

## Advanced
### Abort

To abort an ongoing connection attempt, call `[mySDKPointer abort]`.

`connectionError:` and a `connectionEvent:` callbacks will then be called.

### URL Scheme Trigger

To start your application using an URL scheme, declare the scheme in the Xcode project of your app. See [Apple Documentation](https://developer.apple.com/library/ios/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html#//apple_ref/doc/uid/TP40007072-CH6-SW10)


### Customization

Several fields can be translated using the `Strings.localizable` file of your project (See [Apple Documentation](https://developer.apple.com/library/ios/documentation/MacOSX/Conceptual/BPInternational/InternationalizingYourUserInterface/InternationalizingYourUserInterface.html#//apple_ref/doc/uid/10000171i-CH3-SW4)):

See [Languages.md](./Languages.md) for the key/value entries and descriptions.