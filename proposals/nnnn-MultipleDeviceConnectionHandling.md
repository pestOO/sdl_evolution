# Handling multiple connections to the same iOS device

* Proposal: [SDL-NNNN](nnnn-MultipleDeviceConnectionHandling.md)
* Author: [Elisey Zamakhov](https://github.com/pestOO)
* Status: **Awaiting review**
* Impacted Platforms: Mobile, <TBD after option select>

## Introduction

***TBD***

Proposal contains 4 solutions:

1. iAP driver changes without SDL changes
2. SDL Code Transport layer changes, which requires iAP driver notification about the same device connection
3. SDL Core Protocol layer changes, which requires Protocol specification extension
4. SDL Core Application layer changes, which requires long (1-2) connection procedure

## Motivation

***TBD***
The biggest concern for this feature is the need to minimize the impact to the user when an iOS device connected via BT is also connected via USB. Since Apple requires that only one transport should be connected (priority should be for wired transport) to the same device and since an iAP session over both the transports are needed to determine if the same device is connected over both the transports, the head unit should be able to disconnect the app connection over the wireless transport and restore the connection back over the wired transport within a very short period of time – to provide a seamless experience to the user.



# Solution #1 - iAP Driver Layer
## Proposed solution

Target head-unit provides an iAP (iPod Accessory Protocol) driver, which handles BT and USB iOS mobile device connections and encapsulates SDL-to-IAP connection throw existing named EAP (External Accessory Protocol Strings).
iAP driver switches BT and USB connections.
This switching is to be hidden for SDL Core and HMI.

***Pro:***

* No SDL changes
* No Application disconnection for HMI side
* No Application disconnection on Application layer of SDL Core
* No Application disconnection on Protocol layer of SDL Core

***Cons:***

* Requires iAP driver update for each integrator
* Covers only one device type connection (iAP)

## High-level design

![iAP Driver Solution](assets/driver.png)

Step 1 (precondition): BT connection

* Mobile connects with BT
* iAP driver connects to device and notify SDL Core with a EAP
* SDL Core connects to Device with a EAP
* App is working as usual

Step 2: USB connection

* The same mobile connects with USB
* iAP driver catches new USB connection and detect the same device
* iAP driver stops processing SDL-to-Mobile data transferring
* (optional) iAP driver disconnects BT connection
* Mobile app starts streaming all new data throw USB transport
* iAP driver start forwarding new Mob-to-SDL data from USB to the same EAP
* iAP driver start forwarding all SDL-to-MOb data for EAP to the USB connection
* Note: SDL Core works without any changes and notifications

Case 2: USB dis-connection

* The same mobile disconnects with USB
* iAP driver catches USB disconnect and detect the same device BT connection
* iAP driver stops processing SDL-to-Mobile data transferring
* (optional) iAP driver connects with BT transport
* Mobile app starts streaming all new data throw BT transport
* iAP driver start forwarding new Mob-to-SDL data from BT to the same EAP
* iAP driver start forwarding all SDL-to-MOb data for EAP to the BT connection
* Note: SDL Core works without any changes and notifications

## Impact on existing code

* iOS Mobile SDK
	* ***TBD***
* HaedUnit integration driver
	* Add logic for using the same EAP for BT and USB connection from the same device



# Solution #2 - SDL Core Transport layer changes

## Proposed solution

SDL can switch between transport connection due to integration system (transport drivers) events.
iAP drive still handles the same device connection and provide this information to the SDL Core.
SDL core Transport layer switches BT and USB connections.
This switching is to be hidden for upper (protocol and application) SDL Core levels and HMI.

***Pro:***

* No Application disconnection for HMI side
* No Application disconnection on Protocol layer of SDL Core
* Could cover any type of device
* Could be delivered to Open-source as a template for further integration
* Could be integrated for Android, Windows Phone devices
* Could be integrated for new transport type (TCP/WiFi)

***Cons:***

* Require iAP driver notification about device re-connection
* Require SDL Core Transport Manager changes
* Bigger timeout, which more affects User reconnection experience
* Dropping SDL-to-Mob data during waiting timeouts
* Risk of data loose during Connection/disconnection (when data is already send to driver, but new connection is not established yet)

## High-level design

![SDL Core Transport Solution](assets/sdl_transport.png)

Step 1 (precondition): BT connection

* Mobile connects with BT
* iAP driver connects to device and notifies SDL Core with a EAP_1
* SDL Core connects to Device with a EAP_1
* App is working as usual

Step 2: USB connection

* The same mobile connects with USB
* iAP driver connects to device and notifies SDL Core with a EAP_2
* Mobile app starts streaming all new data throw USB transport
* (Sub-case A) iAP Driver notifies SDL about the same device connection
* (Sub-case B) SDL identify new connection as the same device
* SDL Core connects to the EAP_2
* (optional) SDL Core disconnects from the EAP_1 (BT)
* SDL Core starts processing all data from EAP_2 as for a EAP_1 connection
* Note: Protocol and Application layers don't involve in transport change procedure

Case 2: USB dis-connection

* The same mobile disconnects with USB
* iAP driver notifies SDL Core with a EAP_2 disconnect
* (optional) SDL starts timer for waiting EAP_1 (BT) reconnection
* SDL connects to EAP_1 (BT)
* SDL Core starts processing all data from EAP_1 as for a EAP_2 connection
* Note: Protocol and Application layers don't involve in transport change procedure

## Impact on existing code

* iOS Mobile SDK
	* Implementation reconnection flow (without Session establish and App registration)
	* ***TBD***
* HaedUnit integration driver
	* Add BT establishing 
* SDL Core - Transport layer
	* Adding logic for the same device determination
	* Updating existing connections identifier schema
	* Adding timeouts for waiting device re-connection
	* Adding limitation for the outcome data during waiting timeout
	* Updating DeviceUpdate and ApplicationListUpdate notifications
* SDL Core - Protocol Handler
	* Update Heartbeat restricts (for avoid disconnect due to HB timeout)
	* Update Waiting Consecutive Frames timeout restricts (for avoid disconnect due to HB timeout)
* SDL Core - Application Manager
	* Update Waiting timeout for reconnecting procedure
* SDL Core - Policy
	* Updating <transportType> info



# Solution #3 - SDL Core Protocol layer changes
## Proposed solution

Each re-connecting application could send a request for fast reconnection to SDL with a new type of transport.
It will required extending Ford Protocol specification with a new Control packets.
SDL core Protocol layer switches BT and USB connections due to Mobile request.
This switching is to be hidden for SDL Core application level and HMI.

***Pro:***

* Following SDL Protocol way with an Application responsible for (re)connection to HU
* Works for Android and other devices "Out-Of-Box"
* No additional iAP changes
* No Application disconnection on Application layer of SDL Core
* No Application disconnection on Protocol layer of SDL Core
* No Application disconnection for HMI side

***Cons:***

* Requires Ford Protocol Specification extension
* A lot of SDL Core changes


## High-level design

![SDL Core Protocol Solution](assets/sdl_protocol.png)

## Impact on existing code

* iOS Mobile SDK
	* Implementation reconnection flow
	* Implementation extension of Protocol
	* ***TBD***
* SDL Core - Transport layer
	* Adding timeouts for waiting device re-connection
	* Adding limitation for the outcome data during waiting timeout
	* Updating DeviceUpdate and ApplicationListUpdate notifications
* SDL Core - Protocol Handler
	* Implementation extension of Protocol with new Frame
	* Implement connection switching mechanism
	* Secure session reestablishing for new session
	* Video and audio services reestablishing for new session
	* Update Heartbeat restricts (for avoid disconnect due to HB timeout)
	* Update Waiting Consecutive Frames timeout restricts (for avoid disconnect due to HB timeout)
* SDL Core - Application Manager
	* Update Waiting timeout for reconnecting procedure
* SDL Core - Policy
	* Updating <transportType> info



# Solution #4 - SDL Core Application layer changes
## Proposed solution

This solution provides ability to switch connection during new Application registration.
In case of Registering the same application from the same device it could be determined as the same application.
SDL core Application layer switches BT and USB connections due to Mobile Register Application Interface Request.
This switching is to be hidden for HMI.

***Pro:***

* Following SDL Protocol way with an Application responsible for (re)connection to HU
* No additional iAP changes
* Could cover any type of device
* Works for Android and other devices "Out-Of-Box"
* No Application disconnection for HMI side

***Cons:***

* The longest reconnection procedure, which mostly affects User reconnection experience
* A lot of SDL Core changes


## High-level design

![SDL Core Protocol Solution](assets/sdl_application.png)

## Impact on existing code

* iOS Mobile SDK
	* Implementation reconnection flow
	* ***TBD***
* SDL Core - Transport layer
	* Adding timeouts for waiting device re-connection
	* Adding limitation for the outcome data during waiting timeout
	* Updating DeviceUpdate and ApplicationListUpdate notifications
* SDL Core - Protocol Handler
	* Secure session reestablishing for new session
	* Video and audio services reestablishing for new session
	* Update Heartbeat restricts (for avoid disconnect due to HB timeout)
	* Update Waiting Consecutive Frames timeout restricts (for avoid disconnect due to HB timeout)
* SDL Core - Application Manager
	* Implement connection switching mechanism
	* Update Waiting timeout for reconnecting procedure
* SDL Core - Policy
	* Updating <transportType> info
* SDL Core - Resumption
	* Updating device type information
* SDL Core - Security
	* Secure session reestablishing for new sessions


## Alternatives considered

All 4 alternatives are provided. See pro and cons bellow.