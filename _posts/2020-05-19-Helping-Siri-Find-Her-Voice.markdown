---
layout: post
title:  "Helping Siri find her voice with UE Booms"
date:   2020-05-19 21:04:38 -0400
categories: Reverse Engineering
---

### Introduction
In 2015 I purchased a [UE Boom 2][ue-boom] speaker, and much to my delight the speaker came along with an app which among other things allowed you to power the speaker on and off from the app, initially this piqued my interest as to how it was done efficiently. Fast forward a few years later and Apple releases Siri Shortcuts, allowing app developers to integrate their app with Siri and even allow users to automate certain tasks. Now, I have rather patiently been waiting for UE to add this feature to their app; to no avail. So this blog will focus on my efforts to reverse engineer the UE Boom speaker and create an app which will allow Siri to “find her voice” when talking to the UE Boom speaker. Please note that this research was conducted solely on a UE Boom 2 so support for other speakers may vary.

### Required Tools
To reverse engineer the speaker protocol we will need a few tools.
These include:
* [IDA Disassembler][IDA]
* A jailbroken iPhone
* [Frida][frida]
* Patience

### Step 1: Dumping the App
Apple encrypts each app with their [FairPlay DRM][fairplay], so rather than painstakingly reverse engineering the decryption protocol only to have it changed, we just dump the decrypted app from memory. My preferred way of doing this is [installing Frida on a jailbroken iPhone][frida-install] and using a tool called [Bagbak][bagbak] to dump the app’s contents.

A list of apps can be found using: `bagbak -l`

The UE Boom app can be dumped using: `bagbak -o "UEBoom" com.logitech.ue.ueboom`

![Dumping the app binaries](/blog/assets/asciinema/dumping.svg)


### Aside: Bluetooth Low Energy

As mentioned above, how does the speaker provide a power on function without using up an incredibly large amount of power monitoring for connections, the answer is the Bluetooth Low Energy protocol (or BLE from here out). Essentially BLE allows for devices to provide limited Bluetooth functionality using relatively low power. BLE is composed of 4 types of roles, however we will be focusing on the main two types used which are peripherals and centrals. As their names suggests a central is a device which listens for advertising devices and can connect to them, in the case of the speaker this would be the phone. On the opposite side, a peripheral is a device that advertises itself and accepts connections from central devices, the speaker. Peripherals will send out advertisement packets containing information which centrals can parse and use the data accordingly. 

Once a central and peripheral are connected they can now communicate information, the two main focuses for understanding how the UE Boom speaker uses BLE are Services and Characteristics. A Service is a group of one or more attributes, or Characteristics, that satisfy a certain functionality. A Characteristic is a part of a service that represents a specific type of information such as the device name or battery level which has a specific UUID to address it.

![Peripheral General Diagram](/blog/assets/images/peripheral-structure.png)

This is an incredibly brief overview of BLE designed only to give the basic understanding required for reverse engineering the protocol of a UE Boom speaker. A more in-depth explanation can be found [here][ble-indepth].


### Step 2: Poking Around the Bush
Now that we have a general idea of how BLE works, we can start analysis of the speaker protocol. Using an app called [Light Blue][lightblue-app] we can see that our speaker (the device with the lowest RSSI, as it is the closest to the phone) only has one service. 

After examining the service there are some interesting characteristics such as "Device Name", and "Serial Number". We will come back to these later.

![LightBlue Service Lister](/assets/images/lightbluedevices.png)|![LightBlue Characteristics](/assets/images/lightbluecharacteristics.png)

Now, loading the app binary, `UEBoom/com.logitech.ue.ueboom/Payload/menhum.app/menhum`, into IDA we can start poking around in the internal functions.

Once IDA has loaded the binary and analyzed all the Objective C function names we can search for all classes containing "BLE"

![IDA BLE Listing](/assets/images/idablelisting.png)

As we can see there is one particular class that refers specifically to BLE and after looking through some functions, we can see a large number of references to a class called "Samurai" which is not in the binary, a few minutes of googling yields no results, so our next step is to look at the frameworks

![UEBoom Frameworks](/assets/images/ueboomframeworks.png)

The `UEComm` framework looks interesting, analyzing the binary contained yields our Samurai class!

![IDA Samurai Listing](/assets/images/idasamurailisting.png)

### Step 3: Analyzing the BLE Protocol
Poking around the Samurai class we can see it implements the `CBCentralManagerDelegate` which, from Apple's documentation calls the `didDiscoverPeripheral` when a peripheral is discovered, searching for this function yields two classes `Samurai` and `NouveauSamurai`, based on my tenuous knowledge of French, I'm guessing `NouveauSamurai` is the class to examine

![IDA didDiscover](/assets/images/idadiscoverperipherallisting.png)

Delving into these functions we can see the steps to identify if the peripheral is a valid speaker based on the advertisement data and if so returns the bluetooth address of the speaker. The follwing python snippet outlines the procedure:

{% gist 0eb63bc7e063a29b8c06d68e4e60e6ec %}

Our next steps are to identify what the characteristics found using LightBlue correspond to, all `CBCentralManagerDelegate` can implement `didDiscoverCharacteristicsForService` and examining this function for the `NouveauSamurai` class, gives us this pseudocode snippet.

![IDA Characteritics](/assets/images/idaidentifycharacteristics.png)

Although the code looks somewhat intimidating, the actual logic flow is quite simple, the function essentially checks the characteristic identifier and if it is equal to a defined characteristic it calls the function to set it for the device. The function continues the same template the entire way through to provide a list of BLE function, which are listed in table below. Comparing these identifiers to the characteristics found in the LightBlue app we can see they are the same!

| UUID                                 | Characteristic      |
|--------------------------------------|---------------------|
| 2A00                                 | Speaker Name        |
| 2A19                                 | Battery Level       |
| 2A24                                 | Speaker Color       |
| 2A25                                 | Serial Number       |
| 2A28                                 | Firmware Version    |
| c6d6dc0d-07f5-47ef-9b59-630622b01fd3 | Power On            |
| 16e005bb-3862-43c7-8f5c-6f654a4ffdd2 | Alarm               |
| 69c0f621-1354-4cf8-98a6-328b8faa1897 | Broadcaster Address |
| 2A1A                                 | Battery Power State |

#### Note: I have found that not all speakers implement all these characteristics.

### Powering On
Now that we have identified the characteristics provided by the speaker we can turn the speaker on, our best bet is probably whichever function uses the characteristic with the UUID `c6d6dc0d-07f5-47ef-9b59-630622b01fd3`, or the power on characteristic. Looking for references to this UUID we come across a function called `-[NouveauSamurai callPowerOnCommandWithForcedConection:returnBlock:]`

![Power On Function Pseudocode](/assets/images/idapoweronpseudocode.png)

From the pseudocode above we can see that to turn the speaker on the app sends the devices bluetooth MAC address with `01` appended to the power characteristic. We can test this using the LightBlue app... and success!

### Communicating with a Connected Speaker
Once the speaker is connected we have to figure out how to communicate with the speaker to perform actions such as changing the speaker name, and powering off. This will occur over bluetooth core as opposed to BLE. Sifting through the class names on particular class stands out, `EASessionManager`. While google yields no result, an EA session hints that UE is using Apple's ExternalAccessory framework to communicate with a connected speaker, this can be confirmed by looking through the `Info.plist` file for `UISupportedExternalAccessoryProtocols`.

![Message Building Procedure](/assets/images/idabuildmessage.png)

Now that we know the framework the devices communicate with we need to understand the protocol, examining the pseudocode above of the function `-[UEMessage buildRequestData]`, we know that each message will have the following structure

| Message Protocol                       |
|----------------------------------------|
| UInt8 - Message Size (2 + Data.length) |
| UInt16 - Message ID*                   |
| [UInt8] - Message Data                 |

##### * See the bottom of this post for all the message IDs

To check the messages being sent and received match our reversed protocol we can view the messages by hooking the responsible functions, `-[EASessionManager sendData: ]` and `-[EASessionManager - processIncomingData:]` using the below Frida script running it with the command `frida -U -f "com.logitech.ue.ueboom" -l dump_easession.js`

{% gist 02a61422e426273072b1de9f6c1b7da4 %}

![Frida Script Output](/assets/images/messagedump.png)

Success! Our reversed protocol matches the messages being sent.

### Conclusion
Now that we have reverse engineered the protocol we can implement our own app and integrate it with Siri... Well kind of, unfortunately Apple requires that you have a paid developer account to use Siri integration (real nice of them) so I will be providing my app source code for anyone with a paid developer account to fork and add Siri support if they so choose*. Ultimately this was a fun little project on understanding how bluetooth enabled devices interact, specifically BLE devices and devices in Apple's ecosystem and I learned a lot both reverse engineering the protocol and writing my own app to support it 

### Additional Info

#### [App Source Code][app-source-code]

#### Message IDs

| Command Name                                      |   Command ID |
|---------------------------------------------------|--------------|
| UEAcknowledgeMessage                              |            0 |
| UEGetProtocolVersionMessage                       |            1 |
| UEReturnProtocolVersionMessage                    |            2 |
| UEGetFirmwareVersionMessage                       |            3 |
| UEReturnFirmwareVersionMessage                    |            4 |
| UEGetDeviceIdMessage                              |            6 |
| UEReturnDeviceIdMessage                           |            7 |
| UEGetSonificationMessage                          |          355 |
| UEReturnSonificationMessage                       |          356 |
| UESetSonificationMessage                          |          357 |
| UEGetDeviceStatusMessage                          |          358 |
| UEReturnDeviceStatusMessage                       |          359 |
| UEBeginDiscoveryMessage                           |          361 |
| UEBeginInquiryMessage                             |          360 |
| UEStopRestreamingModeMessage                      |          362 |
| UETriggerEventMessage                             |          364 |
| UEGetBluetoothNameMessage                         |          365 |
| UEReturnBluetoothNameMessage                      |          366 |
| UESetBluetoothNameMessage                         |          367 |
| UEGetHardwareIdMessage                            |          368 |
| UEReturnHardwareIdMessage                         |          369 |
| UEGetAudioRoutingMessage                          |          370 |
| UEReturnAudioRoutingMessage                       |          371 |
| UESetAudioRoutingMessage                          |          372 |
| UEGetChargingInfoMessage                          |          373 |
| UEReturnChargingInfoMessage                       |          374 |
| UEGetEQSettingsMessage                            |          375 |
| UEReturnEQSettingsMessage                         |          376 |
| UESetEQSettingsMessage                            |          377 |
| UEGetBatteryLevelMessage                          |          532 |
| UEReturnBatteryLevelMessage                       |          785 |
| UEGetTWSBalanceMessage                            |          411 |
| UEReturnTWSBalanceMessage                         |          412 |
| UESetTWSBalanceMessage                            |          413 |
| UEGetFeaturesMessage                              |          414 |
| UEReturnFeaturesMessage                           |          415 |
| UEGetTwoByteColorMessage                          |          446 |
| UEReturnTwoByteColorMessage                       |          447 |
| UEGetLanguageMessage                              |          352 |
| UEReturnLanguageMessage                           |          353 |
| UESetLanguageMessage                              |          354 |
| UEAnnounceBatteryLevel                            |          363 |
| UEGetChargerRegisterMessage                       |          385 |
| UEReturnChargerRegisterMessage                    |          386 |
| UEGetSoCInternalVariablesMessage                  |          387 |
| UEReturnSoCInternalVariablesMessage               |          388 |
| UEGetAlarmMessage                                 |          382 |
| UEReturnAlarmMessage                              |          383 |
| UESetAlarmMessage                                 |          384 |
| UESendAVRCPCommandMessage                         |          391 |
| UEGetCustomEqualizerMessage                       |          392 |
| UEReturnCustomEqualizerMessage                    |          393 |
| UESetCustomEqualizerMessage                       |          394 |
| UESet32StepVolumeMessage                          |          397 |
| UEGet32StepVolumeMessage                          |          398 |
| UEReturn32StepVolumeMessage                       |          399 |
| UESetVolumeMessage                                |          400 |
| UEGetVolumeMessage                                |          401 |
| UEReturnVolumeMessage                             |          402 |
| UEGetTWSSavePairFlagMessage                       |          403 |
| UEReturnTWSSavePairFlagMessage                    |          404 |
| UESetTWSSavePairFlagMessage                       |          405 |
| UEGetSupportedLanguagesMessage                    |          406 |
| UEReturnSupportedLanguagesMessage                 |          407 |
| UEGetCustomizationStateMessage                    |          408 |
| UEReturnCustomizationStateMessage                 |          409 |
| UESetCustomizationStateMessage                    |          410 |
| UEGetPlaybackMetadataMessage                      |          416 |
| UEReturnPlaybackMetadataMessage                   |          417 |
| UEGetSleepTimerMessage                            |          418 |
| UEReturnSleepTimerMessage                         |          419 |
| UESetSleepTimerMessage                            |          420 |
| UEGetSerialNumber                                 |          380 |
| UEReturnSerialNumberMessage                       |          381 |
| UEGetTWSSlaveBatteryLevelMessage                  |          426 |
| UEReturnTWSSlaveBatteryLevelMessage               |          427 |
| UEGetSourceBluetoothAddressMessage                |          428 |
| UEReturnSourceBluetoothAddressMessage             |          429 |
| UEGetDeviceBluetoothAddressMessage                |          430 |
| UEReturnDeviceBluetoothAddressMessage             |          431 |
| UEGetTWSSlaveBluetoothAddressMessage              |          432 |
| UEReturnTWSSlaveBluetoothAddressMessage           |          433 |
| UESetTWSSlaveOffMessage                           |          434 |
| UEGetAuxLevelMessage                              |          436 |
| UEReturnAuxLevelMessage                           |          437 |
| UESetDeviceOffMessage                             |          438 |
| UEGetOTAStateMessage                              |          424 |
| UEReturnOTAStateMessage                           |          425 |
| UESetOTAStateMessage                              |          423 |
| UEGetBLEStateMessage                              |          439 |
| UEReturnBLEStateMessage                           |          440 |
| UESetBLEStateMessage                              |          441 |
| UEPartitionSerialFlashMessage                     |          512 |
| UEMountPartitionMessage                           |          513 |
| UEGetPartitionStateMessage                        |          514 |
| UEReturnPartitionStateMessage                     |          515 |
| UESetPartitionStateMessage                        |          516 |
| UECheckPartitionStateMessage                      |          517 |
| UEReturnCheckPartitionStateMessage                |          518 |
| UEAdjustVolumeMessage                             |          443 |
| UEGetTWSSlaveVolumeMessage                        |          444 |
| UEReturnTWSSlaveVolumeMessage                     |          445 |
| UEGetPartyModeStateMessage                        |         1025 |
| UESetPartyModeStateMessage                        |         1024 |
| UEReturnPartyModeStateMessage                     |         1026 |
| UEGetPartyModeInfoMessage                         |         1027 |
| UEReturnPartyModeInfoMessage                      |         1028 |
| UEKickPartyModeMemberMessage                      |         1031 |
| UEPartyModeNotificationMessage                    |         1040 |
| UEAVRCPPlayStatusChangeNotificationMessage        |         1041 |
| UEGetConnectedDeviceNameMessage                   |          421 |
| UEReturnConnectedDeviceNameMessage                |          422 |
| UETrackLengthInfoNotificationMessage              |         1042 |
| UESetEnabledNotificationsMessage                  |          519 |
| UEGetEnabledNotificationsMessage                  |          520 |
| UEReturnEnabledNotificationsMessage               |          521 |
| UECheckPartitionFiveInfoMessage                   |          522 |
| UEReturnCheckPartitionFiveInfoMessage             |          523 |
| UEGetPartyModeCurrentStreamingDeviceMACMessage    |         1032 |
| UEReturnPartyModeCurrentStreamingDeviceMACMessage |         1033 |
| UEGetGestureControlStateMessage                   |          452 |
| UEReturnGestureControlStateMessage                |          453 |
| UESetGestureControlStateMessage                   |          454 |
| UEGetBroadcastModeMessage                         |         1044 |
| UESetBroadcastModeMessage                         |         1043 |
| UEReturnBroadcastModeMessage                      |         1045 |
| UEBroadcastReceiverStatusNotificationMessage      |         1046 |
| UEAddReceiverToBroadcastMessage                   |         1047 |
| UEAddReceiverToBroadcastResultMessage             |         1048 |
| UERemoveReceiverFromBroadcastMessage              |         1049 |
| UERemoveReceiverFromBroadcastResultMessage        |         1050 |
| UEGetFixedBroadcastReceiverAttributesMessage      |         1054 |
| UEReturnFixedBroadcastReceiverAttributesMessage   |         1055 |
| UESetBroadcastVolumeSyncMessage                   |         1051 |
| UESetReceiverXUpPromiscuousModeMessage            |         1062 |
| UEGetReceiverXUpPromiscuousModeMessage            |         1066 |
| UEReturnReceiverXUpPromiscuousModeMessage         |         1067 |
| UEBLEAvailableMessage                             |         1075 |
| UEVoiceNotificationMessage                        |         1068 |
| UEVoiceSetFlagMessage                             |         1072 |
| UEVoiceLedAndToneGenerationMessage                |         1069 |
| UEGetVoiceControlFlagMessage                      |         1070 |
| UEReturnVoiceControlFlagMessage                   |         1071 |
| UEGetVoiceCapabilitiesMessage                     |         1076 |
| UEReturnVoiceCapabilitiesMessage                  |         1077 |
| UEGetSonificationFileSizeMessage                  |          525 |
| UEReturnSonificationFileSizeMessage               |          526 |
| UEVoiceModeAckMessage                             |         1078 |
| UEDoubleTapDetectMessage                          |         1079 |
| UEVoicePingMessage                                |         1080 |
| UEVoiceExpectSpeechMessage                        |         1081 |
| UEMagicButtonPressedMessage                       |         1088 |
| UESetMagicLEDStateMessage                         |         1089 |
| UEReturnMagicButtonPowerUpStatusMessage           |         1093 |
| UEGetMagicButtonPowerUpStatusMessage              |         1092 |

##### This has only been tested with a UE Boom 2, use at your own risk

[ue-boom]: https://www.ultimateears.com/en-ca/wireless-speakers/boom-2.html

[IDA]: https://www.hex-rays.com/products/ida/
[frida]: https://frida.re/

[fairplay]: https://www.theiphonewiki.com/wiki/Copy_Protection_Overview
[frida-install]: https://frida.re/docs/ios/
[bagbak]: https://github.com/ChiChou/bagbak

[ble-indepth]: https://www.novelbits.io/basics-bluetooth-low-energy/

[lightblue-app]: https://apps.apple.com/ca/app/lightblue/id557428110
[app-source-code]: https://github.com/braydn-moore/SiriGoesBoom