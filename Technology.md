# Immuni's Technology

## Table of contents

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core flow](#core-flow)
- [Architecture](#architecture)
- [Mobile Client](#mobile-client)
  - [Features](#features)
    - [Onboarding](#onboarding)
    - [Exposure Detection](#exposure-detection)
    - [TEKs upload](#teks-upload)
    - [Analytics](#analytics)
  - [A/G Framework](#ag-framework)
  - [iOS App](#ios-app)
    - [iOS App technologies](#ios-app-technologies)
    - [iOS App security](#ios-app-security)
  - [Android App](#android-app)
    - [Android App technologies](#android-app-technologies)
    - [Android App security](#android-app-security)
  - [App build verification](#app-build-verification)
    - [Open build process](#open-build-process)
    - [Reproducible builds](#reproducible-builds)
      - [iOS](#ios)
      - [Android](#android)
- [Backend Services](#backend-services)
  - [Backend Services technologies](#backend-services-technologies)
  - [App Configuration Service](#app-configuration-service)
    - [API - Fetch Configuration Settings](#api-fetch-configuration-settings)
  - [OTP Service](#otp-service)
    - [OTP format](#otp-format)
    - [API - Authorise OTP](#api-authorise-otp)
  - [Exposure Ingestion Service](#exposure-ingestion-service)
    - [API - Validate OTP](#api-validate-otp)
    - [API - Upload TEKs](#api-upload-teks)
    - [Data model - Uploads](#data-model-uploads)
    - [Data model - TEKChunks](#data-model-tekchunks)
  - [Exposure Reporting Service](#exposure-reporting-service)
    - [API - Fetch TEK Chunk indexes](#api-fetch-tek-chunk-indexes)
    - [API - Download TEKs](#api-download-teks)
  - [Analytics Service](#analytics-service)
    - [API - Authorise analytics token (iOS)](#api---authorise-analytics-token-ios)
    - [API - Upload Operational Info (iOS)](#api---upload-operational-info-ios)
    - [API - Upload Operational Info (Android)](#api---upload-operational-info-android)
    - [Data model - Exposures](#data-model---exposures)
    - [Data model - Operational Info](#data-model---operational-info)

## Introduction

This document provides a low-level design description of the technology developed for Immuni. It assumes that you have already read the following documents:

- [High-Level Description](/README.md)
- [Product](/Product.md)

More information can be found in the documentation linked throughout. For those interested in the technology behind Immuni, the following documents are especially relevant:

- [Application Security](/Application%20Security.md)
- [Privacy-Preserving Analytics](/Privacy-Preserving%20Analytics.md)
- [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md)

## Glossary

For ease of reference, the following list defines some of the terms used throughout this document.

**Analytics Service.** See [Architecture](#architecture).

**App (iOS App and Android App).** See [Architecture](#architecture).

**App Configuration Service.** See [Architecture](#archicture).

**Apple and Google Framework (A/G Framework).** See [Architecture](#architecture).

**Attenuation.** The difference between the power of the BLE transmission and the RSSI. The Attenuation is affected by factors such as the distance between transmitter and receiver (higher distance implies higher Attenuation), the presence of obstacles between devices (obstacles imply higher Attenuation), and the presence of walls around the devices (closed spaces imply lower Attenuation). The Attenuation can be used as a proxy for the riskiness of an Exposure (higher Attenuation generally implies lower risk).

**Backend Services.** See [Architecture](#architecture).

**Bluetooth Low Energy (BLE).** A short-range radio communication standard that uses little transmission power to minimise its impact on battery consumption. Immuni uses BLE for the exchange of RPIs across Mobile Clients.

**Configuration Settings.** Data returned by the App Configuration Service to the Mobile Client to make it possible to dynamically change the way the App is configured. For example, the Configuration Settings are used to communicate the parameters to be used by the Mobile Client in the calculation of the Total Risk Score.

**Epidemiological Info.** Any set of Exposure Detection Summaries and Exposure Info.

**Exposure.** The event of two Mobile Clients coming into sufficient proximity of each other for long enough (generally a few metres for at least a few minutes) that they will receive and locally store each other’s RPIs. We will say that these Mobile Clients were ‘Exposed’ to each other or we will use similar expressions. The duration of the Exposure is defined as that estimated by the Mobile Client, independently of the actual length of the period of time during which the Mobile Clients were in proximity of each other. Considering that the detection of the relevant BLE signals is performed periodically and that these can be obstructed by various obstacles, the estimation of duration will necessarily be imperfect. When the TEK of one of the two Mobile Clients changes (this occurs once a day), the Exposure is considered to have ended. A new Exposure may start if the two Mobile Clients continue to be in sufficient proximity of each other.

**Exposure Detection.** The process through which the Mobile Client periodically checks whether Exposure to a SARS-CoV-2-positive user has taken place and computes an assessment of the risk that a transmission ensued. More detailed information can be found in [Exposure Detection](#exposure-detection).

**Exposure Detection Summary.** A summary of aggregate information related to the Exposures of a Mobile Client for a set of TEKs. When the A/G Framework on the Mobile Client is provided with a set of TEKs, it can check whether they Match any of the locally stored RPIs. After completing this check, the Mobile Client generates an Exposure Detection Summary. Calculated for the given set of TEKs and their respective Exposures, the Exposure Detection Summary includes the following:

- The number of TEKs Matching any of the locally stored RPIs
- The number of days since the most recent Exposure to any Matching TEK of the given set
- The sum of the durations of those Exposures, grouped by three ranges of Attenuation (each measured in five-minute increments and capped at 30 minutes)
- The highest Total Risk Score among those of the various Exposures

Note that the Exposure Detection Summary does not contain either the TEKs that Matched a locally stored RPI, or the Matched RPIs.

**Exposure Info.** Information related to a specific Exposure. For each Exposure for a set of TEKs Matching locally stored RPIs, the A/G Framework on the Mobile Client can generate an object containing the following information:

- The day the Exposure occurred
- The duration of the Exposure (measured in five-minute increments and capped at 30 minutes)
- The average Attenuation during the Exposure
- The duration of the Exposure, broken down by three ranges of Attenuation (each measured in five-minute increments and capped at 30 minutes)
- The Transmission Risk for the relevant TEK
- The Total Risk Score for the Exposure

Note that the Exposure Info does not contain either the TEK for which it was computed, or the RPIs that Matched the TEK.

**Exposure Ingestion Service.** See [Architecture](#architecture).

**Exposure Reporting Service.** See [Architecture](#architecture).

**Health Information System (HIS).** A web interface available to authorised and authenticated Healthcare Operators that is used to authorise an OTP. Once the OTP has been authorised, the Mobile Client is allowed to use it to send TEKs to the Exposure Ingestion Service.

**Healthcare Operator.** An employee of the National Healthcare Service who is authorised to facilitate the upload of the user’s TEKs by authorising an OTP generated on the Mobile Client.

**Immuni.** The sum of all the parts underlying the exposure detection and notification system described in this document, including Mobile Clients, Backend Services, Healthcare Operators, and HIS.

**Match.** The Mobile Client can determine whether an RPI which was stored locally after an Exposure has been generated from a given TEK. When this is the case, for simplicity, we will say that 'the TEK Matches the RPI', 'the RPI Matches the TEK', 'there is a Match for the TEK', or we will use similar expressions.

**Mobile Client.** See [Architecture](#architecture).

**National Healthcare Service.** The Italian national healthcare system, known as the _Servizio Sanitario Nazionale._

**One Time Password (OTP).** A random, ten-character code used to authenticate calls between the Mobile Client and the Exposure Ingestion Service while uploading TEKs, Epidemiological Info, and the user’s Province of Domicile. The code is generated on the Mobile Client and then authorised by a Healthcare Operator via the OTP Service. It expires after a defined interval.

**Operational Info.** Data on the Mobile Client’s working condition and Exposure notifications. The data include:

- Whether the device runs iOS or Android
- Whether permission to leverage the A/G Framework is granted
- Whether the device’s Bluetooth is enabled
- Whether permission to send local notifications is granted
- Whether the user was notified of a Risky Exposure after the last Exposure Detection
- The date on which the last Risky Exposure took place, if any

The data are periodically uploaded to the Analytics Service without requiring the user to authenticate in any way (including verifying a phone number or email address). This helps protect user privacy.

**OTP Service.** See [Architecture](#architecture).

**Province of Domicile.** The Epidemiological Info and the Operational Info are more informative when broken down by geographical regions. To protect the user’s privacy, the App only asks for the province in which they live. This happens during the onboarding process.

**Received Signal Strength Indicator (RSSI).** The power of a radio signal as measured at the receiving end of the transmission.

**Risky Exposure.** An Exposure for which the Total Risk Score as determined through the Exposure Detection is high enough that Immuni elects to notify the user that they may be at risk.

**Rolling Proximity Identifier (RPI).** A 16-byte identifier generated from a TEK, broadcast via BLE by a Mobile Client, and stored locally by the nearby Mobile Clients. The RPI being broadcast changes multiple times an hour to prevent wireless tracking of the Mobile Client.

**RPI Database.** The local database where the RPIs that the Mobile Client has collected are stored.

**TEK Chunk.** A set of TEKs, created by the Exposure Ingestion Service by grouping TEKs generated on and uploaded by the Mobile Clients of users who tested positive for SARS-CoV-2 in a specific time window. The TEK Chunks are then made available for download to the Mobile Client by the Exposure Reporting Service. To avoid duplicate downloads, the TEK Chunks are immutable and assigned a unique incremental index. They are generated periodically as new TEKs are uploaded by the Mobile Clients and deleted after 14 days.

**Temporary Exposure Key (TEK).** A 16-byte, cryptographic, random key generated by a Mobile Client. Every day, the Mobile Client generates a different TEK and uses it to derive a set of RPIs. The TEKs leave the Mobile Client only after receiving user approval and only when the user has tested positive for SARS-CoV-2. The Backend Services delete TEKs older than 14 days.

**Total Risk Score.** For a specific Exposure, the assessed risk that a SARS-CoV-2 transmission may have occurred. This is calculated through a risk model. The model accounts for contributions from the following factors:

- The number of days since the Exposure occurred
- The duration of the Exposure
- The average Attenuation
- The Transmission Risk associated to that specific TEK

Currently, Immuni leverages only the duration of the Exposure and the average Attenuation to calculate the Total Risk Score. More detailed information on how the Total Risk Score is calculated can be found [here](https://developer.apple.com/documentation/exposurenotification/enexposureconfiguration).

**Transmission Risk.** People who tested positive for SARS-CoV-2 may be more or less infectious according to a number of factors, such as the time elapsed since the appearance of symptoms. The Transmission Risk is a 3-bit value that can be associated with a TEK to indicate how contagious the user was while a specific TEK was in use. It allows for a more sophisticated calculation of the Total Risk Score. Currently, Immuni does not leverage the Transmission Risk.

## Core flow

**TEK and RPI generation.** Every 24 hours, the Mobile Client generates a new TEK and stores it locally. The Mobile Client derives RPIs from the TEK via a cryptographic hash function. Every RPI is associated with a subinterval of the 24 hours of a TEK’s validity. Using a cryptographic hash function makes it practically impossible to derive a TEK from an RPI.

**RPI exchange.** Mobile Clients use BLE to continuously broadcast the RPI associated with the current subinterval of the 24 hours, and collect RPIs from Mobile Clients they were Exposed to. The received RPIs are stored locally, together with the timestamp of the event and the Attenuation.

**TEK upload.** When a user tests positive for SARS-CoV-2, a Healthcare Operator will ask them whether they wish to dictate the OTP they can locate in the App. If so, the Healthcare Operator inserts the OTP provided by the user into the HIS, and the HIS forwards the OTP to the OTP Service. This operation authorises the App to upload to the Exposure Ingestion Service the TEKs generated over the previous 14 days.

**TEK Chunk publishing.** On a regular basis, the Exposure Ingestion Service creates TEK Chunks containing the TEKs that have been uploaded since the last TEK Chunk creation. The Exposure Reporting Service makes the TEK Chunk available publicly.

**Exposure Detection and notification.** On a regular basis, the Mobile Clients download the new TEK Chunks, verify whether any TEK Matches the RPIs received and stored locally by the Mobile Client over the previous 14 days, and compute an Exposure Detection Summary. This includes summary information on Exposures to the TEKs within the TEK Chunk. It also includes the maximum Total Risk Score across these Exposures, if any. If at least one Exposure has occurred and the maximum Total Risk Score is above a certain threshold, Exposure Info is computed for each Exposure and the user is notified that they may be at risk, then provided with a recommendation of what to do.

## Architecture

Immuni’s architecture includes Mobile Client and Backend Services.

**Mobile Client.** The user’s mobile device and all the software running on it. Within the context of Immuni, its core functions include the following:

- Generating the TEKs on a daily basis
- Generating the RPIs from the daily TEK
- Broadcasting the RPIs through BLE signals
- Collecting RPIs broadcast by other Mobile Clients
- Periodically downloading the TEKs of users who tested positive for SARS-CoV-2 and decided to make them available for download
- Computing Exposure Detection Summaries and Exposure Info
- Notifying the user if relevant
- Allowing the user to upload their recent TEKs (together with their Province of Domicile and any Epidemiological Info from the previous 14 days) in the case that they test positive for the virus
- Periodically collecting and uploading Operational Info, together with the user’s Province of Domicile

The Mobile Client includes the following parts:

- **A/G Framework.** The Exposure Notification framework made available by Apple and Google to help developers implement Exposure notification apps (see [Apple’s documentation](https://www.apple.com/covid19/contacttracing) and [Google’s documentation](https://www.google.com/covid19/exposurenotifications/)). Immuni leverages this framework. The A/G Framework is responsible for generating and storing the TEKs and RPIs, controlling the BLE interface, and providing Exposure Detection Summaries and Exposure Infos.
- **App (iOS App and Android App).** The Immuni iOS or Android apps that users download from the App Store and Google Play respectively. The App is responsible for handling user interactions (including notifying users that they may be at risk) and for implementing the business logic (including all communications with the Backend Services).

**Backend Services.** Immuni’s backend software and infrastructure, which include the following services:

- **App Configuration Service.** This serves the Configuration Settings used by the App.
- **OTP Service.** This facilitates the authorisation of the OTP used by the Mobile Client for authenticating with the Exposure Ingestion Service. The Healthcare Operator will collect the OTP from a consenting user who tested positive for SARS-CoV-2, and enter it into the HIS. The HIS will then record the OTP in a database, authorising it and making it accessible to the Exposure Ingestion Service.
- **Exposure Ingestion Service.** When a Healthcare Operator informs a user that they tested positive for SARS-CoV-2, this service allows the user’s Mobile Client to start an upload of the TEKs it generated over the previous 14 days and their Province of Domicile. If any Epidemiological Info from the previous 14 days is available, the Mobile Client will upload that too.
- **Exposure Reporting Service.** This provides a scalable system to serve the TEKs of SARS-CoV-2-positive users to the Mobile Clients, after they were uploaded through the Exposure Ingestion Service.
- **Analytics Service.** The National Healthcare Service needs to have some visibility as to the number of devices on which the app is correctly functioning, and the number of users Immuni is notifying. This helps it to operate the system effectively. The Analytics Service allows the upload of Operational Info (together with the user’s Province of Domicile) by the Mobile Client. To help protect user privacy, the data are uploaded without identifying the user, who is not asked to authenticate in any way.

## Mobile Client

### Features

Below, we describe the main functionality offered by the Mobile Client.

#### Onboarding

New users undertake the onboarding process, in which they are asked for some information and permissions that are essential for the App to work.

The onboarding process has been designed to limit the information asked of the user to their Province of Domicile, and nothing else.

The onboarding process guides the user through the following steps:

- **Terms of use, privacy notice, and minimum age.** If they wish to use the App, the user must read and accept the terms of use, read the privacy notice, and confirm they are 14 or older.
- **A/G Framework.** The A/G Framework requires the user’s permission before it can be used.
- **Location (Android only).** On Android devices, _Location_ needs to be on at the operating system level to detect nearby devices, although the A/G Framework’s documentation explicitly states that it does not actually use location data. This may be quite confusing for the user, but unfortunately it is outside of the Android App’s control. Please note that the Android App will not request the Location permission, nor will it have access to location data. However, Location needs to be activated at the operating system level for the A/G Framework to work properly.
- **Bluetooth.** Understandably, the A/G Framework requires the device’s Bluetooth to be turned on.
- **Push notifications (iOS only).** The user is asked to give permission to the iOS App for it to show a notification whenever it detects that the user is at risk of having been infected with SARS-CoV-2. On Android this permission is on by default, so the Android App does not need to ask for it during the onboarding. Note that these notifications are generated locally, not sent from a server.

#### Exposure Detection

The Mobile Client periodically checks whether Exposure to a SARS-CoV-2-positive user has taken place and computes an assessment of the risk that a transmission ensued by executing the following steps:

1. The App downloads the new TEK Chunks made available by the Exposure Reporting Service.
2. The A/G Framework verifies whether there is a Match between one or more TEKs in the TEK Chunks and the locally stored RPIs.
3. For each Exposure for these TEKs, the A/G Framework computes a Total Risk Score using a simple, publicly available risk model with parameters set by the App.
4. The A/G Framework generates an Exposure Detection Summary, which includes the maximum Total Risk Score across all Exposures for the TEKs.
5. If the maximum Total Risk Score exceeds a certain threshold, the App has the A/G Framework provide the Exposure Info for each of these Exposures (this triggers an operating system alert to the user) and notifies the user that they may be at risk.

#### TEKs upload

If a user is found to be SARS-CoV-2-positive, they can decide to upload the following information to the Exposure Ingestion Service:

- The TEKs generated by user’s Mobile Client in the previous 14 days
- The user’s Province of Domicile
- The Epidemiological Info computed in the previous 14 days, if any

A Healthcare Operator will assist the user with this operation. The upload process can be summarised as follows:

1. The user enters a dedicated section of the App. The App generates an OTP and displays it to the user, who is invited to communicate it to the Healthcare Operator.
2. The Healthcare Operator inserts the OTP into the HIS, which authorises it by sending it to the Exposure Ingestion Service.
3. The Healthcare Operator invites the user to validate the OTP. Upon user action, the Mobile Client validates the OTP with the Exposure Ingestion Service. If the validation is successful, the process continues as described below.
4. The operating system shows a native confirmation pop-up and the user must provide their permission before the App can access the TEKs. Then, the App proceeds to query the A/G Framework to retrieve the 14 most recent TEKs. Note that the App cannot bypass this authorisation process.
5. After receiving the user’s permission, the App uploads to the Exposure Ingestion Service the 14 most recent TEKs, the user’s Province of Domicile, and the Epidemiological Info for the previous 14 days (if any). The upload is authenticated with the OTP.

The Mobile Client periodically generates dummy traffic to mitigate the risk that information about the user (chiefly whether they tested positive for SARS-CoV-2) may be inferred by an attacker through traffic analysis. Details can be found in [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md).

#### Analytics

Periodically, the Mobile Client sends Operational Info (together with the user’s Province of Domicile) to the Analytics Service. This information is critical for the National Healthcare Service to operate Immuni effectively, thereby maximising the containment of the epidemic and patient care, as explained in the [High-Level Description](/README.md).

We devised a system to allow the Mobile Client to upload such data without requiring the user to authenticate in any way (including through the verification of a phone number or email address). At the same time, the system protects the integrity of the data from large-scale pollution caused by attackers sending fake data. The system does not work for all devices—the server will discard untrustworthy data. Details can be found in [Privacy-Preserving Analytics](/Privacy-Preserving%20Analytics.md).

The Mobile Client periodically generates dummy traffic to mitigate the risk that information about the user (chiefly whether a Risky Exposure was detected) may be inferred by an attacker through traffic analysis. Details can be found in [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md).

### A/G Framework <a name="ag-framework" />

For the implementation of the [core flow](#core-flow), Immuni leverages the A/G Framework (see [Apple’s documentation](https://www.apple.com/covid19/contacttracing) and [Google’s documentation](https://www.google.com/covid19/exposurenotifications/)).

Part of the functionality is provided directly by the operating system, which results in enhanced user privacy:

- The RPI Database is handled by the operating system. As such, the App does not have access to information such as which specific TEK triggered a Match, if any. Without notifying the user, the App can only access the Exposure Detection Summary for a set of TEKs. This offers high-level information on the Exposures for those TEKs. To access the Exposure Info—which offers a bit more detail—the App has to trigger a notification to the user that is handled by the operating system and cannot be bypassed. Moreover, the operating system limits the App’s access to this information in various ways, including through aggregations and rate limits on the relevant API.
- The App must obtain the user’s permission to get their TEKs so that it can upload them to the Exposure Ingestion Service. The entire authorisation flow is triggered by the App and handled by the operating system. The App has no way to bypass this security measure.

On top of the privacy enhancements mentioned above, electing to leverage the A/G Framework unlocks a number of other advantages that we do not believe would be achievable in a custom implementation. These include:

- Allowing reliable background broadcasting and detection of RPIs, especially between two iOS Mobile Clients, which is severely constrained by iOS’s [Core Bluetooth API](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html). There are several workarounds to mitigate this limitation. In our view, however, most of them are ineffective, energy inefficient, not compliant with Apple’s [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/), or all of the above.
- Synchronising the RPI rotation with the Bluetooth MAC address rotation, so these two pieces of information cannot be cross-referenced to identify a user.
- Ensuring the iOS App automatically resumes its normal operation after an operating system reboot.
- Minimising battery consumption.

### iOS App

This section discusses the iOS-specific aspects of the App.

The iOS App is available for devices running iOS 13. However, iOS 13.5 is required for the A/G Framework to work and, therefore, Immuni’s core flow to function properly. To date, Apple has not announced the intention to release it for earlier iOS versions. Hence, the iOS App prevents the user from completing the onboarding if they have not updated to iOS 13.5.

#### iOS App technologies

The App is written using [Swift](https://swift.org/) 5.2 and [Xcode](https://developer.apple.com/xcode/) 11.5.

The development process leverages the following technologies:

- **[Brew](https://brew.sh/).** A package manager for macOS, Brew is released as an open-source project under the BSD 2-Clause 'Simplified' licence.
- **[CocoaPods](https://cocoapods.org/).** The most widespread dependency manager in the iOS community, CocoaPods is released as an open-source project under the MIT licence.
- **[CommitLint](https://commitlint.js.org/#/).** A linter for commits, CommitLint is released as an open-source project under the MIT licence.
- **[Danger](https://danger.systems/js/).** A system to enforce code quality and contribution rules in pull requests, Danger is released as an open-source project under the MIT licence.
- **[SwiftFormat](https://github.com/nicklockwood/SwiftFormat/).** A formatter for Swift code, SwiftFormat is released as an open-source project under the MIT licence.
- **[SwiftGen](https://github.com/SwiftGen/SwiftGen).** A tool that generates type-safe Swift code for accessing project resources, SwiftGen is released as an open-source project under the MIT licence.
- **[SwiftLint](https://github.com/realm/SwiftLint).** A linter for Swift code, SwiftLint is released as an open-source project under the MIT licence.
- **[XcodeGen](https://github.com/yonaskolb/XcodeGen).** A tool that generates Xcode projects starting from a YAML definition, XcodeGen is released as an open-source project under the MIT licence.

The App uses a number of native technologies provided by Apple's iOS SDK (e.g., [Foundation](https://developer.apple.com/documentation/foundation) and [UIKit](https://developer.apple.com/documentation/uikit)).

When it comes to the iOS App’s architecture, Immuni combines a unidirectional data flow for the business logic with a MVVM architecture for the user interface. This structure is the result of combining two libraries:

- **[Katana](https://github.com/BendingSpoons/katana-swift).** A library that provides structure to the flow of information concerning the business logic of a mobile app, Katana is developed and maintained by Bending Spoons and released as an open-source project under the MIT licence.
- **[Tempura](https://github.com/BendingSpoons/tempura-swift/).** Tempura is a library for organising the user interface of a mobile app. Building on top of Katana, Tempura reduces the boilerplate needed to render the current state of the app. Tempura is developed and maintained by Bending Spoons and released as an open-source project under the MIT licence.

Finally, Immuni leverages other open-source libraries, including:

- **[Alamofire](https://github.com/Alamofire/Alamofire).** The de-facto standard for performing HTTP(S) requests on iOS, Alamofire is released as an open-source project under the MIT licence.
- **[BonMot](https://github.com/Rightpoint/BonMot).** A library that abstracts the complexity of creating and styling textual content within an iOS app, BonMot is released as an open-source project under the MIT licence.
- **[Hydra](https://github.com/malcommac/Hydra/).** A promise-pattern implementation that also features constructs such as _async/await,_ Hydra is used to implement the app’s business logic. It is released as an open-source project under the MIT licence.
- **[Lottie](https://github.com/airbnb/lottie-ios).** A library for natively rendering vector-based animations and art in realtime, Lottie is developed and maintained by Airbnb and released as an open-source project under the Apache 2.0 licence.
- **[PinLayout](https://github.com/layoutBox/PinLayout).** A library that permits writing user interface layout code leveraging a series of chainable function calls, PinLayout is released as an open-source project under the MIT licence.
- **[SwiftLog](https://github.com/apple/swift-log/).** A logging package for Swift, SwiftLog is released by Apple under the Swift open-source effort. It is important to note that logs are never stored or sent outside the Mobile Client. Moreover, logs are disabled in production builds. SwiftLog is released as an open-source project under the Apache 2.0 licence.
- **[ZipFoundation](https://github.com/weichsel/ZIPFoundation).** A utility class to zip and unzip files on iOS. It is used to manage the archives containing TEK Chunks downloaded from the Exposure Reporting Service, as specified in Apple's [Setting Up an Exposure Notification Server](https://developer.apple.com/documentation/exposurenotification/setting_up_an_exposure_notification_server). ZipFoundation is released as an open-source project under the MIT licence.

#### iOS App security

Security is one of the most critical topics when it comes to Immuni. What follows is a discussion of some of the measures taken to protect user data, both while stored on the iOS Mobile Client and when sent to any of the Backend Services.

Following Katana's architecture, most of the data that the iOS App uses are retained in a single atom called the _Store._ The Store’s content is persisted in a file stored on the iOS Mobile Client after being encrypted using [AES](https://developer.apple.com/documentation/cryptokit/aes/gcm). The encryption key is generated on the iOS Mobile Client and stored in its [keychain](https://developer.apple.com/documentation/security/keychain_services). Moreover, the file in which the Store is persisted is further protected using the [encryption API](https://developer.apple.com/documentation/uikit/protecting_the_user_s_privacy/encrypting_your_app_s_files) available on iOS.

Some data are also stored in the iOS Mobile Client’s [UserDefaults](https://developer.apple.com/documentation/foundation/userdefaults). The App provides a layer that encrypts this information as well. As with the Store’s content, the encryption key is generated on the iOS Mobile Client and stored in its keychain.

The cryptographic operations that the App performs are implemented using the [Apple CryptoKit](https://developer.apple.com/documentation/cryptokit) framework.

Finally, every communication with the server is established using HTTPS. The Mobile Client only accepts certificates signed by the chosen certificate authority.

### Android App

This section discusses the Android-specific aspects of the App.

The Android App is available for devices running Android 6 (Marshmallow, API 23) or above. For the A/G Framework to work, the user needs to have updated Google Play Services to version 20.18.13 or above. Therefore, the Android App prevents the user from completing the onboarding if they do not have a sufficiently recent version of Google Play Services.

#### Android App technologies

The Android App is written using [Kotlin](https://kotlinlang.org/) 1.3 and [Android Studio](https://developer.android.com/studio) 3.6.

Both the build process and the dependency management are handled by [Gradle](https://gradle.org/), an open-source project released under the Apache 2.0 licence that has seen widespread adoption by the Android community.

The development process leverages also the following technologies:

- **[CommitLint](https://commitlint.js.org/#/).** A linter for commits, CommitLint is released as an open-source project under the MIT licence.
- **[Danger](https://danger.systems/js/).** A system to enforce code quality and contribution rules in pull requests, Danger is released as an open-source project under the MIT licence.
- **[ktlint](https://ktlint.github.io/).** A linter for Kotlin code, ktlint is maintained by Pinterest and released as an open-source project under the MIT licence.

The source code is implemented leveraging native technologies offered by the Android SDK, as well as third-party libraries which we outline below.

The architecture is built on [Android Architecture Components](https://developer.android.com/topic/libraries/architecture/). When it comes to the project’s structure, Immuni is implemented using a data-driven MVVM architecture that follows the recommendations laid out in the [Android guide to app architecture](https://developer.android.com/jetpack/docs/guide). To achieve this, the Android App is based on the following technologies:

- **[Kotlin Coroutines](https://github.com/Kotlin/kotlinx.coroutines), [Channels](https://kotlinlang.org/docs/reference/coroutines/channels.html), and [Flows](https://kotlinlang.org/docs/reference/coroutines/flow.html).** Kotlin Coroutines, Channels, and Flows are used for asynchronous programming, reactive streams of data, and [structured concurrency](https://en.wikipedia.org/wiki/Structured*concurrency). Support for these functionalities is supplied by the _kotlinx.coroutines_ library, which is released as an open-source project under the Apache 2.0 licence.
- **[LiveData](https://developer.android.com/topic/libraries/architecture/livedata) and [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel).** LiveData are observable data holders, while ViewModels are designed to store and manage user interface related data in a lifecycle-conscious way. They both are part of the Android Jetpack set of libraries, and are released as open-source projects under the Apache 2.0 licence.
- **[Navigation Component](https://developer.android.com/guide/navigation).** Navigation Component is used to create single activity apps, reducing complexity and ensuring a consistent and predictable user experience by adhering to an established set of [principles of navigation](https://developer.android.com/guide/navigation/navigation-principles). It is part of the Android Jetpack set of libraries, and is released as an open-source project under the Apache 2.0 licence.
- **[Work Manager](https://developer.android.com/topic/libraries/architecture/workmanager).** The Work Manager API makes it easy to schedule deferrable, asynchronous tasks that are expected to run even if the Android App exits or the device restarts. It is part of the Android Jetpack set of libraries, and is released as an open-source project under the Apache 2.0 licence.

Additionally, the Android App uses the following libraries:

- **[Glide](https://github.com/bumptech/glide).** An image loading and caching library for Android focused on smooth scrolling, Glide is developed and maintained by Google and released as an open-source project under the BSD, MIT, and Apache 2.0 licences.
- **[Koin](https://github.com/InsertKoinIO/koin).** A dependency injection framework for Kotlin, Koin is released as an open-source project under the Apache 2.0 licence.
- **[Lottie](https://github.com/airbnb/lottie-android).** A library for natively rendering vector-based animations and art in realtime, Lottie is developed and maintained by Airbnb and is released as an open-source project under the Apache 2.0 licence.
- **[Moshi](https://github.com/square/moshi).** A modern JSON library for Kotlin and Java, Moshi is developed and maintained by Square Inc. and released as an open-source project under the Apache 2.0 licence.
- **[MockK](https://github.com/mockk/mockk).** A mocking library for Kotlin, MockK is released as an open-source project under the Apache 2.0 licence.
- **[NumberPicker](https://github.com/ShawnLin013/NumberPicker).** An Android library that provides a simple and customisable number picker, NumberPicker is released as an open-source project under the MIT licence.
- **[OkHttp](https://github.com/square/okhttp/).** An efficient, low-level HTTP(S) client for Java and Kotlin, OkHttp offers a robust foundation to high-level HTTP(S) clients like Retrofit. It is developed and maintained by Square Inc. and released as an open-source project under the Apache 2.0 licence.
- **[Retrofit](https://github.com/square/retrofit).** A high-level, type-safe HTTP(S) client for Android and Java, Retrofit is developed and maintained by Square Inc. and released as an open-source project under the Apache 2.0 licence.

#### Android App security

Security is one of the most critical topics when it comes to Immuni. What follows is a discussion of some of the measures taken to protect user data, both while stored on the Android Mobile Client and when sent to any of the Backend Services.

The Android App stores data inside [EncryptedSharedPreferences](https://developer.android.com/reference/androidx/security/crypto/EncryptedSharedPreferences), encrypting them using AES256. The encryption key is generated on the Android Mobile Client and stored in its [keystore](https://developer.android.com/training/articles/keystore). 

To generate the encryption keys on the Android Mobile Client, we use [Jetpack](https://developer.android.com/topic/security/data).

On devices that support them, [Full-Disk Encryption](https://source.android.com/security/encryption/full-disk) and [File-Based Encryption](https://source.android.com/security/encryption/file-based) add a further level of security to the stored data.

Finally, every communication with the server is established using HTTPS. The Mobile Client only accepts certificates signed by the chosen certificate authority.

### App build verification

An important part of building trust around the project is enabling the community to verify that the source code published in the open-source repositories matches that from which the apps made available on the production environments—the App Store and Google Play—are built.

To reach this goal, we must ensure that:

- The build process is publicly available
- The builds are reproducible

Before digging into the details of these two topics, it is important to mention that this is an ongoing effort. We look forward to collaborating with security experts and researchers to further improve the system.

#### Open build process

An _open build process_ is a process that is publicly available for building and uploading Immuni builds to App Store Connect and Google Play Console. This means both making the implementation of the system accessible to the community, and enabling people to inspect logs to verify what the build process actually does. Moreover, the result of the build process should be available for download in the public repository so that it can be used for further analysis by anyone interested in the topic.

Building and uploading builds in such a way that all the tools are publicly verifiable (e.g., open source or provided by Apple and Google), and impossible for the developers to alter, ensures that the production builds are created starting from the public repository. Once a build with a specific combination of application identifier, build number, and release version is uploaded to App Store Connect or Google Play Console, no other build may have the same combination.

On iOS, this triple is determined by the _bundle identifier_ (_CFBundleIdentifier_), _bundle version_ (_CFBundleVersion_), and _release version_ (_CFBundleShortVersionString_). On Android, the same three fields are called _application identifier_ (_applicationId_), _version number_ (_versionCode_), and _release version_ (_versionName_).

On both platforms, the application identifier and the release version are publicly available, while the build number is not. However, this can be easily extracted from the App binaries downloaded from the respective app stores, as it is contained in an unencrypted file (the _Info.plist_ file on iOS, and the _AndroidManifest.xml_ file on Android).

For both platforms, the build process leverages the following technologies:

- **[Fastlane](https://fastlane.tools/).** A platform to orchestrate the build and release process of iOS and Android apps, Fastlane is released as an open-source project under the MIT licence.
- **[GHR](https://deeeet.com/ghr/).** A command to create GitHub releases, GHR is released as an open-source project under the MIT licence.

As a final note, having an open build process does not mean that everyone will be able to access the build system secrets. There are some pieces of information that must be kept secret. For example, we can not share the App Store Connect credentials, otherwise anyone would be able to publish an update of the App on the App Store. However, the commands launched in the build process and, most importantly, the effects and artifacts generated by these commands, are publicly available. Therefore, they confirm the integrity and consistency of the published version.

#### Reproducible builds

As the [Reproducible Builds website](https://reproducible-builds.org/tools/) states, _reproducible builds_ are a set of software development practices that create an independently verifiable path from source to binary code.

In the context of Immuni, reproducibility would offer security researchers an additional way to verify that builds have been made from commits of the publicly available repositories. In fact, reproducibility would exceed the assurance offered by the open build process by letting security researchers compare the App Store and Google Play binaries with binaries generated on their own computers.

Below, we discuss the efforts we have made towards achieving reproducibility of builds on both platforms.

##### iOS  
Reproducible builds in an iOS context are complex. Even with the same environment (e.g., operating system and compiler), builds may differ from run to run. Moreover, Apple requires that App Store builds be signed using a sophisticated  [code signing system](https://developer.apple.com/support/code-signing/). Finally, iOS App builds pass through a process called  [app thinning](https://developer.apple.com/documentation/xcode/reducing_your_app_s_size) that further modifies the uploaded artifacts before the final builds are made available on the App Store for download.

A detailed description of the work done so far can be found [here](https://github.com/immuni-app/immuni-app-ios/issues/217).

##### Android  
The reproducibility of Android builds is affected by similar problems as on iOS, as builds may differ from run to run even with the same building environment.

These are the main aspects that may limit reproducibility:

- The building environment must be exactly the same (e.g., Android Studio, Gradle version, and plugins version)
- The _resources.arsc_ file can be different across builds of the same commit due to a known [issue](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=901956) regarding file ordering
- Production builds are signed with certificates that are kept secret, for security reasons
- The _AndroidManifest.xml_ file of production builds could be slightly different from that of independent builds made from the same commit, due to changes performed by Google Play during app publishing

However, some of these limitations can be addressed:

- APKs can be compared with a [Python script](https://github.com/DrKLO/Telegram/blob/master/apkdiff.py) maintained by Telegram as part of their effort on reproducible builds, or with tools such as [apkanalyzer](https://developer.android.com/studio/command-line/apkanalyzer).
- The APK signature does not modify existing files inside the APK, but only adds three new files which can be skipped during the comparison (_META-INF/MANIFEST.MF, META-INF/CERT.RSA,_ and _META-INF/CERT.SF_).
- Device-specific APKs can be generated from the published Android App bundle using [bundletool](https://developer.android.com/studio/command-line/bundletool), the underlying tool used by Gradle, Android Studio, and Google Play to create and deploy app bundles. It should be noted that APKs generated this way may include bundletool-specific files that can differ between builds (e.g., _META-INF/BNDLTOOL.SF_ and _META-INF/BNDLTOOL.RSA_) and that may not be present in the production device-specific APKs obtained from Google Play. If present, these files can be safely skipped during the comparison.

To increase transparency, the code is neither shrunk nor obfuscated, making it easier for security experts to decompile the bytecode and compare it with the published source code.

At present, we have managed to match different builds of the same commit successfully, but for a few files. However, we recognise that more work needs to be done to achieve a complete solution, and we are eager to collaborate with experts in relevant fields to improve the process.

## Backend Services

Below, we describe the technologies with which the Backend Services are built. Then, we provide a more detailed description of each Backend Service, including the documentation of the APIs and data models.

### Backend Services technologies

All Backend Services are implemented in [Python](https://www.python.org/) 3.8, with [Poetry](https://python-poetry.org/) as dependency manager. They use [Redis](https://redis.io/), an in-memory data store, as a message broker for communication between them. The persistence layer for the relevant data is provided by [MongoDB](https://www.mongodb.com/), a non-relational database.

The backend services follow a micro-service architecture, where each critical functionality is deployed as its own component. Components are distributed in dedicated [Docker](https://www.docker.com/) images, Docker being an industry standard platform for the containerization and virtualization of software.

To monitor the performance and stability of all backend services and therefore guarantee a better availability of such a critical project, the code is instrumented with [Prometheus](https://prometheus.io/), an open-source software monitoring solution.

The following dependencies are used to implement the business logic:

- **[Aioredis](https://aioredis.readthedocs.io/).** An async client for Redis interaction, Aioredis is released as an open-source project under the MIT licence.
- **[Celery](http://www.celeryproject.org/).** An asynchronous task framework to execute delayed or scheduled tasks, Celery is released as an open-source project under the BSD-3 licence.
- **[Decouple](https://github.com/henriquebastos/python-decouple/).** A library to easily inject configuration via static files or environment variables, Decouple is released as an open-source project under the MIT licence.
- **[Marshmallow](https://pypi.org/project/marshmallow/).** A library to easily manage and validate endpoints schemas, Marshmallow is released as an open-source project under the MIT licence.
- **[MongoEngine](http://mongoengine.org/).** A document-object mapper for working with MongoDB from Python, MongoEngine is released as an open-source project under the MIT licence.
- **[Pymongo](https://docs.mongodb.com/drivers/pymongo).** A low-level library to interact with MongoDB, Pymongo is released as an open-source project under the Apache 2.0 licence.
- **[Sanic](https://github.com/huge-success/sanic).** A fast, async web framework to implement web APIs, Sanic is released as an open-source project under the MIT licence.
- **[Uvicorn](https://www.uvicorn.org).** An async web server, Uvicorn is released as an open-source project under the BSD-3 licence.

To increase the security, robustness, and quality of the code, we integrated several established tools into our development pipeline:

- **[Bandit](https://github.com/PyCQA/bandit).** A tool designed to find common security vulnerabilities in Python code, it is maintained by the Python Code Quality Authority and released as an open-source project under the Apache 2.0 licence.
- **[Black](https://github.com/psf/black).** A code formatter for Python, Black is released as an open-source project under the MIT licence.
- **[Flake8](https://flake8.pycqa.org/en/latest/).** A tool to enforce consistent coding style rules, Flake8 is maintained by the Python Code Quality Authority and released as an open-source project under the MIT licence.
- **[Isort](https://isort.readthedocs.io/en/latest/).** A utility to sort Python imports alphabetically, isort is released as an open-source project under the MIT licence.
- **[Mypy](https://mypy.readthedocs.io/en/stable/).** A static type checker for Python, mypy is released as an open-source project under the MIT licence.
- **[Pylint](https://www.pylint.org/).** A linter and static code analysis tool for Python, pylint is maintained by the Python Code Quality Authority and released as an open-source project under the GNU General Public Licence v2.
- **[Pytest](https://docs.pytest.org/en/latest/).** A framework to write tests in Python, pytest is released as an open-source project under the MIT licence.

### App Configuration Service

The App can be partially customised through the Configuration Settings downloaded at the App startup and updated every time the App starts a background session. For example, these include information such as the weights to be used by the Mobile Client in the calculation of the Total Risk Score.

#### API - Fetch Configuration Settings <a name="api-fetch-configuration-settings" />

**Caller**  
Mobile Client

**Description**  
The Mobile Client fetches the Configuration Settings. Different Configuration Settings may be made available for different platforms (i.e., iOS and Android) and App build numbers.

**Resource hostname**  
`get.immuni.gov.it`

**Resource path**  
`GET /v1/settings`

**Request parameters**  
`platform=<ios|android>`  
`build=<build number>`

**Response headers**  
`Cache-Control: public, max-age=3600`  
`Content-Type: application/json; charset=utf-8`

### OTP Service

The OTP Service provides an API to the National Healthcare Service for authorising OTPs that can be used to upload data from the Mobile Client via the Exposure Ingestion Service. The OTP is generated by the App and communicated by the user to the Healthcare Operator. The Healthcare Operator inserts the OTP into the HIS, which then registers the OTP on the OTP Service.

The OTP automatically expires after a defined interval. If the data have not been uploaded by the time the OTP expires, the user will have to restart the process with a new OTP.

#### OTP format

The OTP is composed of 10 uppercase alphanumeric characters, with the last character being a check digit.

To prevent misunderstandings when the user communicates the OTP to the Healthcare Operator, the characters used in the OTP are limited to the following subset: _A, E, F, H, I, J, K, L, Q, R, S, U, W, X, Y, Z,_ 1, 2, 3, 4, 5, 6, 7, 8, 9. This leads to 25^9 possible valid combinations (since the last character is computed deterministically, based on the previous 9 characters).

#### API - Authorise OTP <a name="api-authorise-otp" />

**Caller**  
HIS

**Description**  
The provided OTP authorises the upload of the Mobile Client’s TEKs. This API will not be publicly exposed, to prevent unauthorised users from reaching it. Authentication for having the OTP Service and the HIS trust each other is configured at the infrastructure level. The payload also contains the start date of the symptoms. The Exposure Ingestion Service can use this date to determine the likelihood of the user having been contagious when a specific TEK was current. Those with the lowest probability can then be disregarded.

**Resource hostname**  
The API is only accessible by the HIS

**Resource path**  
`POST /v1/otp`

**Request headers**  
`Content-Type: application/json; charset=utf-8`

**Request body**

```
{
  “otp”: string,
  “symptoms_started_on”: date
}
```

### Exposure Ingestion Service

The Exposure Ingestion Service provides an API for the Mobile Client to upload its TEKs for the previous 14 days, in the case that the user tests positive for SARS-CoV-2 and decides to share them. Contextually, the Mobile Client uploads the user’s Province of Domicile and the Epidemiological Info from the previous 14 days, if any. The upload can only take place with an authorised OTP.

The Exposure Ingestion Service is also responsible for periodically generating the TEK Chunks to be published by the Exposure Reporting Service. The TEK Chunks are assigned a unique incremental index and are immutable. They are generated periodically as the Mobile Clients upload new TEKs.

TEK Chunks older than 14 days are automatically deleted from the database by an async cleanup job.

Province of Domicile and Epidemiological Info are collected while protecting user privacy, as described in [Privacy-Preserving Analytics](/Privacy-Preserving%20Analytics.md) and [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md). The Exposure Ingestion Service forwards them to the Analytics Service.

#### API - Validate OTP <a name="api-validate-otp" />

**Caller**  
Mobile Client

**Description**  
The Mobile Client validates the OTP prior to uploading data. The request is authenticated with the OTP to be validated. Using the dedicated request header, the Mobile Client can indicate to the server that the call it is making is a dummy one. The server will ignore the content of such calls. Padding bytes are added as described in [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md).

**Resource hostname**  
`upload.immuni.gov.it`

**Resource path**  
`POST /v1/ingestion/check-otp`

**Request headers**  
`Authorization: Bearer <SHA256(OTP)>`  
`Content-Type: application/json; charset=utf-8`  
`Immuni-Dummy-Data: <0|1>`

**Request body**

```
{
  “padding”: string
}
```

#### API - Upload TEKs <a name="api-upload-teks" />

**Caller**  
Mobile Client

**Description**  
Once it has validated the OTP, the Mobile Client uploads its TEKs for the past 14 days, together with the user’s Province of Domicile. If any Epidemiological Info from the previous 14 days is available, the Mobile Client uploads that too. The timestamp that accompanies each TEK is referred to the Mobile Client’s system time. The Mobile Client informs the Exposure Ingestion Service about its system time so that a potential skew can be corrected. Using the dedicated request header, the Mobile Client can indicate to the server that the call it is making is a dummy one. The server will ignore the content of such calls. Padding bytes are added as described in [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md).

**Resource hostname**  
`upload.immuni.gov.it`

**Resource path**  
`POST /v1/ingestion/upload`

**Request headers**  
`Authorization: Bearer <SHA256(OTP)>`  
`Content-Type: application/json; charset=utf-8`  
`Immuni-Client-Clock: <UNIX epoch time>`  
`Immuni-Dummy-Data: <0|1>`

**Request body**

```
{
  “teks”: [ 
    {
      “key_data”: string,
      “rolling_start_number”: int,
      “rolling_period”: int
    }, ...
  ],
  “province”: string, 
  “exposure_detection_summaries”: [
    {
      “date”: date
      “matched_key_count”: int,
      “days_since_last_exposure”: int,
      “attenuation_durations”: array[int],
      “maximum_risk_score”: int,
      “exposure_info”: [
        {
          “date”: date,
          “duration”: int,
          “attenuation_value”: int,
          “attenuation_durations”: array[int],
          “transmission_risk_level”: int,
          “total_risk_score”: int
        }, ...
      ]
    }, ...
  ],
  “padding”: string
}
```

#### Data model - Uploads <a name="data-model-uploads" />

**Description**  
The _Uploads_ collection stores the TEKs uploaded by a Mobile Client with an authorised OTP, together with information on the day of symptoms onset, if any. _Uploads_ can be automatically deleted after 14 days by filtering on the generation time of _ObjectId._ _to_publish_ indicates whether the data still needs to be processed to be included in a new TEK Chunk. _teks_ is an array of TEKs. _key_data_ is the base64-encoding of the 16-byte TEK. Any Epidemiological Info is forwarded to the Analytics Service.

**Schema**

```
{
  _id: ObjectId,
  to_publish: bool,
  symptoms_started_on: date,
  teks: [
    {
      key_data: string,
      rolling_start_number: int,
      rolling_period : int
    }, ...
  ]
}
```

#### Data model - TEKChunks <a name="data-model-tekchunks" />

**Description**  
The _TEKChunks_ collection holds the TEK Chunks generated by the Exposure Ingestion Service, and it is also read-accessible by the Exposure Reporting Service so that it can return the available TEK Chunks to the Mobile Clients. _TEKChunks_ can be automatically deleted after 14 days by filtering on the generation time of _ObjectId._ _index_ is the incremental TEK Chunk index. _teks_ is an array of TEKs. _key_data_ is the base64-encoding of the 16-byte TEK.

**Schema**

```
{
  _id: ObjectId,
  index: int,
  teks: [
    {
      key_data: string,
      rolling_start_number: int,
      transmission_risk_level: int
    }, ...
  ]
}
```

### Exposure Reporting Service

The Exposure Reporting Service makes the TEK Chunks created by the Exposure Ingestion Service available to the Mobile Client. Only TEK Chunks for the previous 14 days are made available.

To avoid downloading the same TEKs multiple times, the Mobile Clients first fetch the indexes of the TEK Chunks available to download. Then, they download TEK Chunks with indexes for which TEK Chunks have not previously been downloaded.

#### API - Fetch TEK Chunk indexes <a name="api-fetch-tek-chunk-indexes" />

**Caller**  
Mobile Client

**Description**  
Return the index of the oldest relevant TEK Chunk (no older than 14 days) and the index of the newest available TEK Chunk. It is up to the Mobile Client to not download the same TEK Chunk more than once.

**Resource hostname**  
`get.immuni.gov.it`

**Resource path**  
`GET /v1/keys/index`

**Response headers**  
`Cache-Control: public, max-age=1800`  
`Content-Type: application/json; charset=utf-8`

**Response body**

```
{
  “oldest”: int,
  “newest”: int
}
```

#### API - Download TEKs <a name="api-download-teks" />

**Caller**  
Mobile Client

**Description**  
Given a specific TEK Chunk index, the Mobile Client downloads the associated TEK Chunk from the Exposure Reporting Service.

**Resource hostname**  
`get.immuni.gov.it`

**Resource path**  
`GET /v1/keys/<TEKChunkIndex>`

**Response headers**  
`Cache-Control: public, max-age=1296000`  
`Content-Type: application/zip`

### Analytics Service

The Analytics Service exposes an API that makes it possible for the Mobile Client to upload Operational Info, together with the user’s Province of Domicile. [Privacy-Preserving Analytics](/Privacy-Preserving%20Analytics.md) and [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md) describe how these data are collected while protecting user privacy.

The data are uploaded without requiring the user to authenticate in any way (e.g., no phone number or email verification). Without sensible countermeasures, it would be easy for an attacker to pollute the data at scale, making them unreliable and, therefore, useless. The mitigations we have put in place are described in [Privacy-Preserving Analytics](/Privacy-Preserving%20Analytics.md). They differ for iOS and Android, as the relevant technologies available for the two operating systems—namely Apple’s [DeviceCheck](https://developer.apple.com/documentation/devicecheck) and Google’s [SafetyNet Attestation](https://developer.android.com/training/safetynet/attestation)—are distinct.

Finally, the Analytics Service also receives Province of Domicile and Epidemiological Info from the Exposure Ingestion Service.

#### API - Authorise analytics token (iOS)

**Caller**  
Mobile Client

**Description**  
The Mobile Client requests the authorisation of an analytics token. Such authorisation is carried out by the Analytics Service asynchronously. The Mobile Client subsequently calls the same API to verify whether its current analytics token has been authorised and is therefore valid for the upload of Operational Info. A [DeviceCheck](https://developer.apple.com/documentation/devicecheck) token is provided by the Mobile Client to the Analytics Service. The analytics token and the DeviceCheck token are used by the Analytics Service to mitigate the risk of large-scale pollution of the analytics data, as described in [Privacy-Preserving Analytics](/Privacy-Preserving%20Analytics.md).

**Resource hostname**  
`analytics.immuni.gov.it`

**Resource path**  
`POST /v1/analytics/apple/token`

**Request Headers**  
`Content-Type: application/json; charset=utf-8`

**Request Body**
```
{
  “analytics_token”: <128-byte-string>
  “device_token”: string  
}
```

**Response status codes**  
201: authorised token  
202: authorisation in progress

#### API - Upload Operational Info (iOS)

**Caller**  
Mobile Client

**Description**  
Right after an Exposure Detection completes, the Mobile Client may compute and upload the Operational Info, together with the user’s Province of Domicile. A valid analytics token is needed to complete the operation successfully. The Operational Info fields use integers instead of booleans to ensure that the size of the payloads does not depend, among other things, on sensitive information such as whether the user was notified of a Risky Exposure. Using the dedicated request header, the Mobile Client can indicate to the server that the call it is making is a dummy one. The server will ignore the content of such calls. Padding bytes are added as described in [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md).

**Resource hostname**  
`analytics.immuni.gov.it`

**Resource path**  
`POST /v1/analytics/apple/operational-info`

**Request headers**  
`Authorization: Bearer <analytics_token>`   
`Content-Type: application/json; charset=utf-8`  
`Immuni-Dummy-Data: <0|1>`

**Request body**
```
{
  “province”: string
  “exposure_permission”: int
  “bluetooth_active”: int
  “notification_permission”: int
  “exposure_notification”: int
  “last_risky_exposure_on”: date,
  “padding”: string
}
```

#### API - Upload Operational Info (Android)
*This section is under construction.*

#### Data model - Exposures

**Description**  
The _Exposures_ collection stores the Province of Domicile and the Epidemiological Info forwarded by the Exposure Ingestion Service to the Analytics Service. _Exposures_ can be automatically deleted after a configurable number of days by filtering on the generation time of _ObjectId_.

**Schema**
```
{
  _id: ObjectId,
  province: string,
  exposure_detection_summaries: [
    {
      date: date
      matched_key_count: int,
      days_since_last_exposure: int,
      attenuation_durations: array[int],
      maximum_risk_score: int,
      exposure_info: [
        {
          date: date,
          duration: int,
          attenuation_value: int,
          attenuation_durations: array[int],
          transmission_risk_level: int,
          total_risk_score: int
        }, ...
      ]
    }, ...
  ]
}
```

#### Data model - Operational Info

**Description**  
The _OperationalInfo_ collection holds the Operational Info sent by the Mobile Clients to the Analytics Service. _platform_ indicates whether the data is coming from an iOS or Android Mobile Client. _OperationalInfo_ documents can be automatically deleted after a configurable number of days by filtering on the generation time of _ObjectId_.

**Schema**
```
{
  _id: ObjectId,
  province: string,
  platform: string,
  exposure_permission: boolean,
  bluetooth_active: boolean,
  notification_permission: boolean,
  exposure_notification: boolean,
  last_risky_exposure_on: date
}
```
