# Setting up Push Notifications with React Native

Setting up the certifications for enabling Push Notifications in React Native can be a bit tricky! And now that Parse is being shut down, you're probably looking for alternative solutions for setting up a PN server for your app!

In enabling push notifications for our React Native app we decided against Parse and set up our own Push Notification server using the 'node-apn' module. This tutorial will walk you through the process of setting up PushNotificationIOS on your React Native Project, obtaining the necessary certificates and using the 'node-apn' module to set up the back end for sending push notifications.

To get started, I highly recommend reading an overview of the process of sending a Push Notification on the iOS Developer Library Guide: [The Path of a Push Notification](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/ApplePushService.html).

NB: Push notifications are not available in the iOS Simulator in Xcode. To test push notifications, you need an iOS device, as well as an Apple Developer license.

## Set up of the Push Notification server

### Step 1: Add PushNotificationIOS Library and Link

Copy the PushNotificationIOS.xcodeproj file (from the node_modules/react-native/Libraries/PushNotifications folder in your project) into the Libraries folder in Xcode. Follow the [step-by-step guide to linking libraries](https://facebook.github.io/react-native/docs/linking-libraries-ios.html#manual-linking) in the React Native docs on the website.

### Step 2: Add to AppDelegate.m

Add the following lines of code to the AppDelegate.m file in your project.

At the top of the file add:

```objective-c
#import "RCTPushNotificationManager.h"
```

At the bottom of the file before '@end':

```objective-c
// Required for the register event.
- (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken
{
 [RCTPushNotificationManager application:application didRegisterForRemoteNotificationsWithDeviceToken:deviceToken];
}
// Required for the notification event.
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)notification
{
 [RCTPushNotificationManager application:application didReceiveRemoteNotification:notification];
}
```

### Step 3: Get certificates

Push notifications are sent from Apple servers to the application on a users phone. This requires your app to have certain certificates to prove it is authorised to send push notifications. We recommend following the [step-by-step guide](https://parse.com/tutorials/ios-push-notifications) on the Parse website to obtain the necessary certificates. This is a quick overview of the process:

1. Create an App ID and the associated SSL certificate on the Apple Developer website associated with your Developer account. This certificate will allow the push notification server we will create to connect to the Apple Push Notification Service (APNs) send push notifications to the application identified by the App ID on a user's phone (identified by their specific device token). Each App ID is required to have its own client SSL certificate.

  * 1.1 Create a certificate signing request file ('CertificateSigningRequest.certSigningRequest'). This will authenticate the creation of the SSL certificate.
  * 1.2 Create an AppID on the Apple Developer Member Center  - every IOS application needs a unique AppID
  * 1.3 Configure your App ID for **development** push  notifications (NB a separate certificate is needed for production). This involves uploading the '.certSigningRequest' file, generating an SSL certificate of the form '**cert.cer**', downloading it and installing it (if you have a Mac, the certificate will be installed in your keychain when you double click on the download)
  * 1.4 Export the certificate from the keychain and name it '**key.p12**'.

2. Create a new Provisioning Profile for push notifications in the Apple Developer Member Centre and install it.
  * A Provisioning Profile authenticates your device to run the app
  * In Xcode > Preferences > Accounts click 'Download all' to download the Push notification provisioning profile

**NB: For Ad Hoc Production (beta testing) or Production (submission to the App store), new certificates and keys need to be generated so the whole of Step 3 needs to be repeated.**

### Step 4: Set up the APN server

Our app server might have an endpoint e.g. '/push' with a handler that sends push notifications to APNs which then sends it to the user (identified by their device token). We'll set up a push notification server using the [node-apn](https://github.com/argon/node-apn) module. To set up a connection with node-apn, you need a Certificate and Key files. These correspond to the __'cert.cer'__ and __'key.p12'__ files created in Step 3. These files need to be converted into '.pem' files. To do this, follow the tutorial on the [node-apn wiki](https://github.com/argon/node-apn/wiki/Preparing-Certificates). The key commands are:

```bash
$ openssl x509 -in cert.cer -inform DER -outform PEM -out cert.pem
$ openssl pkcs12 -in key.p12 -out key.pem -nodes
```
Run these commands in your terminal from the folder in which your cert.cer and key.p12 files are saved. Save the '.pem' files in the same folder as your APN server.

We'll not set up the basic boilerplate for an APN server:

```js
var apn  = require('apn');
var path = require('path');

var apnConnection = new apn.Connection({
  production: false,
  cert: path.join(__dirname, 'cert.pem'),  //path to cert.pem file
  key: path.join(__dirname, 'key.pem'),   // path to key.pem file
});

/**
 *	Create a apn instance.
 *	@param {String} - device_token
 *	@return {Object} - new instance object
**/

function getNewApnInstance (device_token) {

	return {
		device: new apn.Device(device_token),
		note: new apn.Notification()
	};
}

/**
 *	Send notification to a specific device identified by the device token
 *	@param (Function) - callback
 *
**/
function sendNotification (device_token) {

	var instance = getNewApnInstance(device_token);

	instance.note.alert = 'You have a new  push notification!';
	instance.note.payload = {text: 'Message from Joe Bloggs'};

	apnConnection.pushNotification(instance.note, instance.device);

	apnConnection.on('transmitted', function (notification, device){
		console.log('on.transmitted', arguments);
	});

	apnConnection.on('transmissionError', function (errorCode, notification, device) {
		console.log('on.transmissionError',arguments);
	});
};

module.exports = {
  sendNotification: sendNotification  
}

```

You can then use these functions in any of your server endpoint handlers. e.g the handler for the '/push' endpoint might look something like this:

```js
  var Apn = require('./apn.js');

  module.exports = (req, reply) => {
    const { device_token } = req.payload;
    Apn.sendNotification(device_token);

    return reply({
      status: 'success',
    });
  }
```
The device token is sent in the request payload and the  `sendNotification` method is used to send the notification to the specific user.

## Use PushNotificationIOS inside your React-Native component

Require in `PushNotificationIOS` at the top of your component. Event listeners can then be attached to the 'register' and 'notification' events. It is best to add these in the top level of your app inside `componentWillMount`.

```js
componentWillMount(){
  PushNotificationIOS.addEventListener('register', (token) => console.log('TOKEN', token))
  PushNotificationIOS.addEventListener('notification', (notification) => console.log('Notification', notification, "APP state", AppStateIOS.currentState))
  // you could check the app state to respond differently to push notifications depending on if the app is running in the background or is currently active.

  PushNotificationIOS.requestPermissions();
}
```
The callbacks for the event listeners could be replaced with Redux actions.

The device token is a hexadecimal string and is like a phone number. It enables the APNs to locate the device on which the app is installed. The callback for the 'register' event can save the device token associated with the particular user.

In this example the currentState of the app is also  being checked so that the app can respond to the push notification depending on if the app is running in the background or currently active.

When the app is opened with a push notification use the `PushNotificationIOS.popInitialNotification()` method to retrieve the data in the push notification payload.

### Trying it out!

Run the app on your phone and `console.log` your device token. Then send yourself a test push notification e.g. by sending a request to the '/push' endpoint on your server with your device token:

```sh
curl localhost:9009/push --data 'device_token=your_token_here'

```

## Credits

This tutorial was written while working on a React Native project with
@besartshyti, @izaakrogan and @jrans and @foundersandcoders.  

If you like what we've done, and want to get in touch tweet me @nikhilaravi.

You might also be interested in some of the React native modules we've created

* [react-native-multi-slider](https://github.com/JackDanielsAndCode/react-native-multi-slider)
* [react-native-smart-scroll-view](https://github.com/jrans/react-native-smart-scroll-view)
* [fetch-wrapper](https://github.com/nikhilaravi/fetch-wrapper)
