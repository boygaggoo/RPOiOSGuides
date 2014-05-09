#iOS Push Notifications (APNS)

![img](http://core0.staticworld.net/images/article/2013/09/ios_7_notification_center-100054497-poster.jpg)

This guide will help you out to **start pushing iOS notifications** to real devices (since the simulator is not supported) taking advantage of the *Apple Push Notification Service*.

###What's APNS?

Apple Push Notification service (APNS for short) is the centerpiece of the push notifications feature. It is a robust and highly efficient service for propagating information to iOS and OS X devices. Each device establishes an accredited and encrypted IP connection with the service and receives notifications over this persistent connection. If a notification for an application arrives when that application is not running, the device alerts the user that the application has data waiting for it.

###How APNS works?

If the notification service is enabled in your app, iOS will generate a unique ID, called **device token**. This token is generated when the app is opened for the firt time and the user accepts the app to send him push notifications. We need to send this token to our server and store it on a database in order to send requests to the APNS server. Then, we will have to use a library to interact with the APNS server. We will see these libraries below.

###APNS server libraries

####PHP

* [**EasyAPNS**](http://www.easyapns.com/)
* [**ApnsPHP**](https://code.google.com/p/apns-php/)

####Node.js

* [**apnagent**](http://apnagent.qualiancy.com/) (*We'll use this one on this guide*)

###Setting up push notification support on our app

First of all, we have to be registered as [Apple Developer](http://developer.apple.com) because we need to create an App ID for our app that supports push notifications as well as a provisioning profile linked to that App ID and the device(s) we're going to test the notifications on. This provisioning profile has to be imported on Xcode.

Then, we have to tell iOS that our app wants to receive push notifications. So we should put this on ```application:didFinishLaunchingWithOptions:``` on the AppDelegate:

	[[UIApplication sharedApplication] registerForRemoteNotificationTypes: (UIRemoteNotificationTypeBadge | UIRemoteNotificationTypeSound | UIRemoteNotificationTypeAlert)];
	
This will make our app able to change its badge number, play notification sounds while closed and show notifications alerts from a server. If you don't need one of these options, feel free to remove it.

![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/registration_sequence_2x.png)

When we open our app and it detects that it should receive push notifications, it will call the APNS server requesting a **deviceToken**. The device token is a 32 bytes string with 64 Hexadecimal characters. We will use this unique ID to send notifications to this device. If we want to send a notification to a user, for example, that has multiple devices, we will have to store one token per device on our server and then send a notification for each device.

With ```-didRegisterForRemoteNotificationsWithDeviceToken:``` we will get the device token we're going to upload to our server.

	-(void)application:(UIApplication*)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData*)deviceToken {
	
        NSString *token = [deviceToken description];
	}
	
Then, we just have to upload it to our server with a simple request.

###Server side: setting up a APNS communicator

![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/remote_notif_simple_2x.png)

So the push notification will work like that: we have a provider, which is our server, that will send a request with a payload to the Apple Push Notification Service server. The device will get the notification and if the payload information matches with the device information and the notifications are enabled on the device, it will finally show up.

Ok, now we have our app ready to receive push notifications. But we need a server which will send to the Apple Push Notification Service the **payload**.

The payload is a JSON-defined property list that specifies how the user of an application on a device is to be alerted. This contains the message, badge count, sound alert file and custom keys if the developer want to send non-public information. I mean, data that will not be shown on the push notification but can be catched programmatically. Example:

	{
    	"aps" : {
        	"alert" : "Hi, I'm a push notification message!",
        	"badge" : 0,
        	"sound" : "chime"
    	},
    	"acme1" : "bar",
    	"acme2" : ["data1", "data2"]
	}

![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/token_trust_2x.png)

Before start building the API, we need to create some certificates.

####Certificates

1. Log in the iOS Provision Portal from the Apple Developer website. Select "App IDs" from the left side menu, then "configure" next to the application you wish generate certificates for.

2. Then we need to enable the APNS support for this app. If it is not already enabled, check the box for "Enable for Apple Push Notification server".

3. Select "Configure" for the environment you want to generate certificates for. Follow the wizard's instructions for generating a CSR file. Make sure to save the CSR file in a safe place so that you can reuse it for future certificates.

4. After you have uploaded your CSR it will generate a "Apple Push Notification service SSL Certificate". Download it to a safe place.

5. Once downloaded, locate the file with Finder and double-click to import the certificate into "Keychain Access". Use the filters on the left to locate your newly import certificate if it is not visible. It will be listed under the "login" keychain in the "Certificates" category. Once located, right-click and select "Export".

6. When the export dialog appears be sure that the "File Format" is set at *.p12*. Name the file according to the environment, such as **apnscert-dev.p12** or **apnscert-prod.p12** and save it to your project directory. We will then be prompted to enter a password to secure the exported item. This is optional, however if we do specify one we will need to configure the library we're going to use with that password.

7. apnagent's classes support *.p12* files through the pfx configuration options to authenticate connections. The last step is optional but instructs on how to turn a *.p12* file into a "key" and "cert" pair (*.pem* format).

Locate your ".p12" file in a terminal session and execute the following commands.

	openssl pkcs12 -clcerts -nokeys -out apns-dev-cert.pem -in apnscert-dev.p12
	openssl pkcs12 -nocerts -out apns-dev-key.pem -in apnscert-dev.p12

The first will generate a *.pem* file for use as the cert file. The second will generate a *.pem* file for use as the key file. The second requires a password be entered to secure the key. This can be removed by executing the following command:

	openssl rsa -in apnagent-dev-key.pem -out apnagent-dev-key.pem

####Building the API

So, we will have to build a simple API on our server to handle the device token reception and the remote notifications control.

In this guide we'll use [**apnagent**](http://apnagent.qualiancy.com/), a node.js module for APNS. First of all, we need to have node.js installed. Then, run on the terminal:

	npm install apnagent
	
In our server folder, let's create a **apns.js** (or whatever) file and a folder with all the certificates we generated, both development (sandbox) and production. On the **apns.js** we're going to write:

	var apnagent = require('apnagent'),
		agent = module.exports = new apnagent.Agent();
		
So the var agent will be our APNS manager. We have to tell apnagent the certficates we're going to use:

	agent
		.set('cert file', join(__dirname, 'certs/apns-dev-cert.pem'))
		.set('key file', join(__dirname, 'certs/apns-dev-key.pem'))
		.enable('sandbox');
		
If we are going to use our server on production, we will have to  change the certificates:

	agent
		.set('cert file', join(__dirname, 'certs/apns-prod-cert.pem'))
		.set('key file', join(__dirname, 'certs/apns-prod-key.pem'));
		
Once it's configured, we are ready to send our push notification to the APNS server.
		
	agent.createMessage()
    	.device("<a1b56d2c 08f621d8 7060da2b c3887246 f17bb200 89a9d44b fb91c7d0 97416b30>")
    	.alert("Hey! I'm a push notification! :D")
    	.send(function (err) {

      	if (err && err.toJSON) {
        	res.json(400, { error: err.toJSON(false) });
        	
        	// Handle the error
      	} 

      	else if (err) {
        	res.json(400, { error: err.message });
        	
        	// Handle anything else (not likely)
      	}

      	else {
        	res.json({ success: true });
        	
        	//Success!
      	}
    });
    
And if everything went fine, the notification will appear on the device.

##Author

Hi, I'm Álvaro Franco, you can reach me on:

* [**@alvarofr_** at Twitter](http://twitter.com/alvarofr_)
* [**@alvarofranco** at GitHub](http://github.com/alvarofranco)

If you have any doubt about the guide, feel free to send me an email at [alvarofrancoayala@gmail.com](mailto:alvarofrancoayala@gmail.com)
