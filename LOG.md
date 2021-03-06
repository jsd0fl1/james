# Concept
I start with the basic concept. This contains the problem, a non-technical solution and a technical solution. At this point I plan to use the NodeMCU firmware and send the push notifications directly from the ESP8266. Maybe this could be problematic because I am not sure that it is possible to authenticate with the Apple Push Notification (APN) servers. I think of using something like a raspberry pi instead. But for now I write a NodeMCU because this is the better solution since normally micro-controller are more reliable than mini computers or servers.
# Implementing the iOS App
I implement the iOS app using Swift and SwiftUI for the UI. The app is not designed pretty. But this is not important for now. I will make a note that designing the app should be a further step. The app is sending a UDP packet with the device token when the users registers his device. Receiving a push notification, the app checks that the device is connected to the given WiFi. Since iOS 13.2 Apple decided for accessing WiFi information the app requires location permission. In iOS 13 the authorization system for the location access "always" has changed fundamentally. It is not possible just requesting that permission. To solve that, the user is advised to set the permission manually. Sometimes iOS asks randomly whether the user wants to upgrade the permission. This should be fixed in the future, but is not essentially.
# Write the NodeMCU firmware
I use a NodeMCU v3 board for testing.
## JWT token
To authorize to the APN servers a self generated JWT token is required as described in the [Apple Documentation](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/establishing_a_token-based_connection_to_apns). This token is not valid for more than one hour. So it is not possible to generate the token on setup. The JWT token must be signed using the ES256 algorithm. The `crypto` module in the NodeMCU firmware only supports AES256 and basic hash algorithms. The SHA256 hash is supported natively by the firmware but the ECDSA algorithm is not. I found a [fork of the ArduinoJWT library](https://github.com/Barmaley13/ArduinoJWT) that supports signing JWT tokens with ES256 on GitHub. Maybe I could use this library, without using the NodeMCU firmware. On the documentation of the library, the library supports the ESP8266 processor. The problem with this library is that the header is hardcoded. I need to set the `kid` field in the header. But I could use parts of the library. I only need creating a JWT token not validating one, and only the one singing algorithm. 
## HTTP/2
According to the [Apple Documentation](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/sending_notification_requests_to_apns) it is required to use the HTTP/2 protocol to communicate with their servers. The NodeMCU firmware has a http module, but when I took a look at the [source code](https://github.com/nodemcu/nodemcu-firmware/blob/master/app/http/httpclient.c#L244) of the module I found that only HTTP/1.1 are supported. So I need to implement a HTTP/2 client on my own. 
## Move logic to Amazon AWS
But all this is very complicated and it is easier to move that logic into an external server on which just libraries can be used. So I move the communication to the APN servers into a AWS Lambda. Another reason to use this options is that in the apple documentation there is a notice that the token has not to be changed more often than every 20 minutes. But when every NodeMCU has its own token handling that cannot be guaranteed. And when this system is shipped to different customers, the private key for signing the JWT token should not be given to every customer. When the key is only stored in the AWS lambda the customer does not have access to it.
# AWS Lambda
There are two AWS lambdas. One is called when somebody rings at the door, the other when a device is registered. The second one has an additional parameter which indicates whether or not the device token was already in the list. That lambdas can be called using a HTTPS request containing the WiFi name in which the notification should be delivered (see at [Implementing the iOS App](#implementing-the-ios-app)) and a list of device tokens.
## Modules
The AWS lambda code is separated into four modules. Two are containing the entrypoints for the lambdas. One is sending the APN message and renews the token (which is stored in a secret manager) if the old one expired. The fourth is a simple module which collects all necessary files and packs them into a ZIP file which then can be uploaded to the lambda function.
## Problems
The NodeMCU does not seem to be able to establish a TLS connection to the Amazon API gateway properly because the certificate chains are too big. This means that it is not able to trigger the lambdas via HTTPS from the NodeMCU. The only other possibilities to trigger a lambda are via the AWS IoT or via SQS. AWS IoT is much overpowered for this simple use case and SQS is not natively supported by the NodeMCU and will properly have the same TLS problem as the API gateway.
# Use a simple Node.js server
So I finally decided that Amazon AWS is not the best option for that. I'll rather use a Node.js server. That server has to be hosted either in the local network (e.g on a raspberry pi or something similar), or on a public server which is accessible from the network the device is connected to (no requirements of a VPN or proxy).
## Usage of UDP
It makes most sense to use UDP for the communication between the Node.js server and the NodeMCU, because it is faster and lighter than TCP and the NodeMCU just needs to send a simple event without any response.
## Security
To secure the connection each packet has to be signed. The NodeMCU supports HMAC-SHA256 natively (besides a few others). This algorithm is appropriate to sign a message. Each device gets a unique security key and a message counter. In every message that counter has to be increased so that a potential attacker cannot send a message multiple times.
## Protocol specifications
Each notification is wrapped into the following packet as binary data and sent to port `3294` to the defined server IP address or hostname:
 - 4 bytes: magic number: 0x1A0E4DE7 (big endian)
 - 2 bytes: 1 bit type of message (`0` for doorbell notification, `1` for register device notification, `2` for already registered notification), 14 bits the identifier of the device
 - 2 bytes: ssid length
 - n bytes: ssid
 - 4 bytes: message counter (unsigned 32bit integer, big endian)
 - 1 byte: the number of devices (only if type is `0`)
 - 32 bytes: the device token (as many as defined in number of devices)
 - 32 bytes: the signature using a HMAC-SHA256 algorithm
## Registration of device to the NodeMCU
When a user clicks on the register device button, the iPhone sends a UDP packet to the network broadcaster on port `32425`. That message has the following structure:
 - 4 bytes: magic number: 0x2A0E4DE9 (big endian)
 - 32 bytes: the device token
