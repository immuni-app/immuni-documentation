# Immuni's Technology Description

## Table of contents

* [Context](#context)
* [Glossary](#glossary)
* [Exposure detection and notification flow](#exposure-detection-and-notification-flow)
* [Architecture](#architecture)
* [Mobile Client](#mobile-client)
	* [Features](#features)
		* [Onboarding](#onboarding)
		* [Transmission Risk Assessment](#transmission-risk-assessment)
		* [TEKs upload](#teks-upload)
		* [Analytics](#analytics)
	* [A/G Framework](#ag-framework)
	* [iOS App](#ios-app)
		* [iOS App technologies](#ios-app-technologies)
		* [iOS App security](#ios-app-security)
	* [Android App](#android-app)
		* [Android App technologies](#android-app-technologies)
		* [Android App security](#android-app-security)
	* [App build verification](#app-build-verification)
		* [Open build process](#open-build-process)
		* [Reproducible builds](#reproducible-builds)
* [Backend Services](#backend-services)
	* [Backend Services technologies](#backend-services-technologies)
	* [App Configuration Service](#app-configuration-service)
		* [API - Fetch Configuration Settings](#api-fetch-configuration-settings)
	* [OTP Service](#otp-service)
		* [OTP format](#otp-format)
		* [API - Authorise OTP](#api-authorise-otp)
	* [Exposure Ingestion Service](#exposure-ingestion-service)
		* [API - Validate OTP](#api-validate-otp)
		* [API - Upload TEKs](#api-upload-teks)
		* [Data model - Uploads](#data-model-uploads)
		* [Data model - TEKChunks](#data-model-tekchunks)
	* [Exposure Reporting Service](#exposure-reporting-service)
		* [API - Fetch TEK Chunk indexes](#api-fetch-tek-chunk-indexes)
		* [API - Download TEKs](#api-download-teks)
	* [Analytics Service](#analytics-service)
* [Open points](#open-points)

## Context

This document provides a low-level design description of the technology developed for Immuni. It assumes that you have already read the [High-Level Description](https://github.com/immuni-app/immuni-documentation/blob/master/README.md).

More information can be found in the documentation linked throughout. Please note that a Product Description and Security Description are in the works and will be published soon, together with the results of some penetration tests. Also, the code will soon be open-sourced.

## Glossary

For ease of reference, the following list defines some of the terms used throughout this document.

**Analytics Service.** See [Architecture](#architecture).

**App (iOS App and Android App).** See [Architecture](#architecture).

**App Configuration Service.** See [Architecture](#archicture).

**Apple and Google Framework (A/G Framework).** See [Architecture](#architecture).

**Attenuation.** The difference between the power of the BLE transmission and the RSSI. The Attenuation is affected by factors such as the distance between transmitter and receiver (higher distance implies higher Attenuation), the presence of obstacles between devices (obstacles imply higher Attenuation), and the presence of walls around the devices (closed spaces imply lower Attenuation). The Attenuation can be used as a proxy for the riskiness of an Exposure (higher Attenuation generally implies lower risk).

**Backend Services.** See [Architecture](#architecture).

**Bluetooth Low Energy (BLE).** A short-range radio communication standard that uses little transmission power to minimise its impact on battery consumption. Immuni uses BLE for the exchange of RPIs across Mobile Clients.

**Configuration Settings.** Data returned by the App Configuration Service to the Mobile Client to make it possible to dynamically change the way the App is configured. For example, the Configuration Settings are used to communicate the weights to be used by the Mobile Client in the calculation of the Total Risk Score.

**Contact.** The event of two Mobile Clients coming into sufficient proximity of each other for long enough (generally a few metres for at least five minutes) that they will receive and locally store each other’s RPIs. We will say that these Mobile Clients ‘came into Contact’ with each other or we will use similar expressions. The duration of the Contact is defined as that estimated by the Mobile Client, independently of the actual length of the period of time during which the Mobile Clients were in proximity of each other. Considering that the detection of the relevant BLE signals is performed periodically and that these can be obstructed by various obstacles, the estimation of duration will necessarily be imperfect.

**Epidemiological Info.** Any set of Exposure Detection Summaries and Exposure Infos.

**Exposure.** For a specific TEK, the set of all the Contacts occurring between the user’s Mobile Client and another Mobile Client for as long as the latter was broadcasting RPIs generated from that TEK (TEKs change daily). For Contacts during which said TEK was generated or retired in favour of a new one, only the portion of the Contact occurring for the relevant TEK is considered part of the Exposure. When the Exposure of a Mobile Client to a certain TEK is not an empty set, we will say that ‘the Mobile Client was Exposed to that TEK’, or we will use similar expressions. The duration of an Exposure is defined as the sum of the durations of all its Contacts.

**Exposure Detection Summary.** A summary of aggregate information related to the Exposures of a Mobile Client for a set of TEKs. When the A/G Framework on the Mobile Client is provided with these TEKs, it can check whether they Match any of the locally stored RPIs. After completing this check, the Mobile Client will generate an Exposure Detection Summary. Calculated for the given set of TEKs and their respective Exposures, the Exposure Detection Summary includes the following:

* The number of TEKs Matching any of the locally stored RPIs, which is also the number of Exposures for that set of TEKs
* The number of days since the most recent Exposure to any Matching TEK of the given set
* The sum of the durations of those Exposures, grouped by two ranges of minimum Attenuation (each measured in five-minute increments and capped at 30 minutes)
* The highest Total Risk Score among those of the various Exposures

Note that the Exposure Detection Summary does not contain either the TEKs that Matched a locally stored RPI, or the Matched RPIs.

**Exposure Info.** Information related to a specific Exposure. For each TEK Matching at least one locally stored RPI—which is to say, for each Exposure—the A/G Framework on the Mobile Client can generate an Exposure Info containing the following information:

* The day the Exposure occurred
* The duration of the Exposure (measured in five-minute increments and capped at 30 minutes)
* The minimum Attenuation during the Exposure
* The sum of the durations of the Exposure’s Contacts, grouped by two ranges of Attenuation (each measured in five-minute increments and capped at 30 minutes)
* The Transmission Risk for the relevant TEK
* The Total Risk Score for the Exposure

Note that the Exposure Info does not contain either the TEK for which it was computed, or the RPI that Matched the TEK.

**Exposure Ingestion Service.** See [Architecture](#architecture).

**Exposure Reporting Service.** See [Architecture](#architecture).

**Health Information System (HIS).** A web interface available to authorised and authenticated Healthcare Operators that is used to authorise an OTP. Once the OTP has been authorised, the Mobile Client is allowed to use it to send TEKs to the Exposure Ingestion Service.

**Healthcare Operator.** An employee of the National Healthcare Service who is authorised to facilitate the upload of the user’s TEKs by authorising an OTP generated on the Mobile Client.

**Immuni.** The sum of all the parts underlying the exposure detection and notification system described in this document, including Mobile Clients, Backend Services, Healthcare Operators, and HIS.

**Match.** The Mobile Client can determine whether an RPI which was stored locally after a Contact has been generated from a given TEK. When this is the case, for simplicity, we will say that 'the TEK Matches the RPI', 'the RPI Matches the TEK', 'there is a Match for the TEK', or we will use similar expressions.

**Mobile Client.** See [Architecture](#architecture).

**National Healthcare Service.** The Italian national healthcare system, known as the *Servizio Sanitario Nazionale.*

**One Time Password (OTP).** A random, ten-character code used to authenticate calls between the Mobile Client and the Exposure Ingestion Service while uploading TEKs, Exposure Detection Summaries, Exposure Infos, and the user’s Province of Domicile. The code is generated on the Mobile Client and then authorised by a Healthcare Operator via the OTP Service. It expires after a defined interval.

**OTP Service.** See [Architecture](#architecture).

**Province of Domicile.** Different areas in Italy are managing the emergency differently, and the advice users receive may vary based on where they are located. To protect the user’s privacy, the App only asks for the province in which they live. This happens during the onboarding process.

**Received Signal Strength Indicator (RSSI).** The power of a radio signal as measured at the receiving end of the transmission.

**Rolling Proximity Identifier (RPI).** A 16-byte identifier generated from a TEK, broadcast via BLE by a Mobile Client, and stored locally by the nearby Mobile Clients. The RPI being broadcast changes multiple times an hour to prevent wireless tracking of the Mobile Client.

**RPI Database.** The local database where the RPIs that the Mobile Client has collected are stored.

**Technical Info.** Data on the Mobile Client’s working condition, useful to estimate the adoption of the App among the population and inform the decision making process of the National Healthcare Service in a number of areas, including product development, engineering, and communications. The data include:

* Whether the device runs iOS or Android
* Whether permission to leverage the A/G Framework is on
* Whether the device’s Bluetooth is on
* Whether permission to send local notifications is on

The data are periodically uploaded to the Analytics Service without asking the user to authenticate or leveraging any user or device identifier, which helps protect user privacy.

**TEK Chunk.** A set of TEKs, created by the Exposure Ingestion Service by grouping TEKs generated on and uploaded by the Mobile Clients of users who tested positive for SARS-CoV-2 in a specific time window. The TEK Chunks are then made available for download to the Mobile Client by the Exposure Reporting Service. To avoid duplicate downloads, the TEK Chunks are immutable and assigned a unique incremental index. They are generated periodically as new TEKs are uploaded by the Mobile Clients and deleted after 14 days.

**Temporary Exposure Key (TEK).** A 16-byte, cryptographic, random key generated by a Mobile Client. Every day, the Mobile Client generates a different TEK and uses it to derive a set of RPIs. The TEKs leave the Mobile Client only after receiving user approval and only when the user has tested positive for SARS-CoV-2. The server deletes TEKs older than 14 days.

**Total Risk Score.** For a specific Exposure, the assessed risk that a SARS-CoV-2 transmission may have occurred. This is expressed as a value from 1 to 8 and calculated through a risk model. The model is a weighted average of contributions from different factors:

* The number of days since the Exposure occurred
* The duration of the Exposure
* The minimum Attenuation
* The Transmission Risk associated to that specific TEK

More detailed information on how the Total Risk Score is calculated can be found [here](https://developer.apple.com/documentation/exposurenotification/enexposureconfiguration).

**Transmission Risk.** People who tested positive for SARS-CoV-2 can be more or less infectious according to the time elapsed since the appearance of symptoms. The Transmission Risk is a 3-bit value that can be associated with a TEK to indicate how contagious the user was while a specific TEK was in use. It allows for a more sophisticated calculation of the Total Risk Score.

**Transmission Risk Assessment.** The process through which the Mobile Client periodically checks whether a Contact with a SARS-CoV-2 positive person has taken place and computes an assessment of the risk that a transmission ensued. More detailed information can be found in [Transmission Risk Assessment](#transmission-risk-assessment).

## Exposure detection and notification flow

**TEK and RPI generation.** Every 24 hours, the Mobile Client generates a new TEK and stores it locally. The Mobile Client derives RPIs from the TEK via a cryptographic hash function. Every RPI is associated with a subinterval of the 24 hours of a TEK’s validity. Using a cryptographic hash function makes it practically impossible to derive a TEK from an RPI.

**RPI exchange.** Mobile Clients use BLE to continuously broadcast the RPI associated with the current subinterval of the 24 hours, and collect RPIs from Mobile Clients they come into Contact with. The received RPIs are stored locally, together with the timestamp of the event and the Attenuation.

**TEK upload.** When a user tests positive for SARS-CoV-2, a Healthcare Operator will ask them whether they wish to dictate the OTP they can locate in the App. If so, the Healthcare Operator inserts the OTP provided by the user into the HIS, and the HIS forwards the OTP to the OTP Service. This operation authorises the App to upload to the Exposure Ingestion Service the TEKs generated over the previous 14 days.

**TEK Chunk publishing.** On a regular basis, the Exposure Ingestion Service creates TEK Chunks containing the TEKs that have been uploaded since the last TEK Chunk creation. The Exposure Reporting Service makes the TEK Chunk available publicly.

**Exposure detection and notification.** On a regular basis, the Mobile Clients download the new TEK Chunks, verify whether any TEK Matches the RPIs received and stored locally by the Mobile Client over the previous 14 days, and compute an Exposure Detection Summary. This includes summary information on Exposures to the TEKs within the TEK Chunk. It also includes the maximum Total Risk Score across these Exposures, if any. If at least one Exposure has occurred and the maximum Total Risk Score is above a certain threshold, an Exposure Info is computed for each Exposure and the user is notified that they may be at risk, then provided with a recommendation of what to do.

## Architecture

The Immuni system includes Mobile Client and Backend Services.

**Mobile Client.** The user’s mobile device and all the software running on it. Within the context of Immuni, its core functions include:

* Generating the TEKs on a daily basis
* Generating the RPIs from the daily TEK
* Broadcasting the RPIs through BLE signals
* Collecting RPIs broadcast by other Mobile Clients
* Periodically downloading the TEKs of users who tested positive for SARS-CoV-2 and decided to make them available for download
* Computing Exposure Detection Summaries and Exposure Infos
* Notifying the user if relevant
* Allowing the user to upload their recent TEKs in the case that they test positive for the virus
* Collecting and uploading the user’s Province of Domicile, Epidemiological Infos (if any), and Technical Infos

The Mobile Client includes the following parts:

* **A/G Framework.** The Exposure Notification framework made available by Apple and Google to help developers implement contact tracing apps (see [Apple’s documentation](https://www.apple.com/covid19/contacttracing) and [Google’s documentation](https://www.google.com/covid19/exposurenotifications/)). Immuni leverages this framework. The A/G Framework is responsible for generating and storing the TEKs and RPIs, controlling the BLE interface, and providing Exposure Detection Summaries and Exposure Infos.
* **App (iOS App and Android App).** The Immuni iOS or Android apps that users download from the App Store and Google Play respectively. The App is responsible for handling user interactions (including notifying users that they may be at risk) and for implementing the business logic (including all communications with the Backend Services).

**Backend Services.** Immuni’s backend software and infrastructure, which include the following services:

* **App Configuration Service.** This serves the Configuration Settings used by the App.
* **OTP Service.** This facilitates the authorisation of the OTP used by the Mobile Client for authenticating with the Exposure Ingestion Service. The Healthcare Operator will collect the OTP from a consenting user who tested positive for SARS-CoV-2, and enter it into the HIS. The HIS will then record the OTP in a database, authorising it and making it accessible to the Exposure Ingestion Service.
* **Exposure Ingestion Service.** When a Healthcare Operator informs a user that they tested positive for SARS-CoV-2, this service allows the user’s Mobile Client to start an upload of the TEKs it generated over the previous 14 days and their Province of Domicile. If any Epidemiological Infos from the previous 14 days are available, the Mobile Client will upload them too.
* **Exposure Reporting Service.** This provides a scalable system to serve the TEKs of SARS-CoV-2 positive users to the Mobile Clients, after they were uploaded through the Exposure Ingestion Service.
* **Analytics Service.** The National Healthcare Service needs to have some visibility into the number of devices the app is correctly functioning on, and the number of users Immuni is notifying. This helps it operate the system effectively, thereby minimising the epidemic’s health toll. The Analytics Service allows the upload of Province of Domicile, Epidemiological Infos (if any), and Technical Infos by the Mobile Client without identifying users. The data have certain limitations enforced by the A/G Framework to protect user privacy. Further protection to user privacy is achieved by uploading such data without asking the user to authenticate, and without leveraging a user identifier or device identifier.

## Mobile Client

### Features

Below, we describe the main functionality offered by the Mobile Client.

#### Onboarding

New users undertake the onboarding process, in which they are asked for some information and permissions that are essential for the App to work.

The onboarding process has been designed to limit the information asked of the user to their Province of Domicile, and nothing else. This is used to show locally relevant recommendations to the user if a risky Exposure is detected.

The onboarding process guides the user through the following steps:

* **Privacy Policy, terms of service, and minimum age.** The user must accept the Privacy Policy and terms of service and confirm they are 14 or older if they wish to use the App.
* **A/G Framework.** The A/G Framework requires the user’s permission before it can be used.
* **Location (Android only).** On Android devices, *Location* needs to be on to detect nearby devices, although the A/G Framework’s documentation explicitly states that it does not actually use location data. This may be quite confusing for the user, but unfortunately it is outside of the Android App’s control. Please note that this does not mean that the Android App will request the Location permission, but only that Location needs to be activated at system level in order for the A/G Framework to work properly.
* **Bluetooth.** Understandably, the A/G Framework requires the device’s Bluetooth to be turned on.
* **Local notifications (iOS only).** The user is asked to give permission to the iOS App for it to show a notification whenever it detects that the user is at risk of having been infected with SARS-CoV-2. On Android this permission is on by default, so the Android App does not need to ask for it during the onboarding. Note that these notifications are generated locally, not sent from a server.

#### Transmission Risk Assessment

The Mobile Client periodically checks whether a Contact with a SARS-CoV-2 positive person has taken place and computes an assessment of the risk that a transmission ensued by executing the following steps:

1. The App downloads the new TEK Chunks made available by the Exposure Reporting Service.
2. The A/G Framework verifies whether there is a Match between one or more TEKs in the TEK Chunks and the locally stored RPIs.
3. For each Matching TEK—and, therefore, each Exposure—the A/G Framework computes a Total Risk Score using a simple, publicly available risk model with parameters set by the App.
4. The A/G Framework generates an Exposure Detection Summary, which includes the maximum Total Risk Score across all Exposures for the TEK Chunks.
5. If the maximum Total Risk Score exceeds a certain threshold, the App has the A/G Framework provide an Exposure Info for each Exposure for the TEK Chunks (this triggers an operating system alert to the user) and notifies the user that they may be at risk. The exact recommendations offered by the App depend on the maximum Total Risk Score.

#### TEKs upload

If a user is found to be SARS-CoV-2 positive, they can decide to upload the following information to the Exposure Ingestion Service:

* The TEKs generated by user’s Mobile Client in the previous 14 days
* The user’s Province of Domicile
* The Epidemiological Infos computed in the previous 14 days, if any

A Healthcare Operator will assist the user with this operation. The upload process can be summarised as follows:

1. The user enters a dedicated section of the App. The App generates an OTP and displays it to the user, who is invited to communicate it to the Healthcare Operator.
2. The Healthcare Operator inserts the OTP into the HIS, which authorises it by sending it to the Exposure Ingestion Service.
3. The Healthcare Operator invites the user to validate the OTP. Upon user action, the Mobile Client validates the OTP with the Exposure Ingestion Service. If the validation is successful, the process continues as described below.
4. The operating system shows a native confirmation pop-up and the user must provide their permission before the App can access the TEKs. Then, the App proceeds to query the A/G Framework to retrieve the 14 most recent TEKs. Note that the App cannot bypass this authorisation process.
5. After receiving the user’s permission, the App uploads to the Exposure Ingestion Service the 14 most recent TEKs, the user’s Province of Domicile, and the Epidemiological Infos for the previous 14 days (if any). The upload is authenticated with the OTP.

The Mobile Client periodically generates dummy traffic to prevent information about the user (chiefly whether they tested positive for SARS-CoV-2) to be gained through traffic analysis. Details can be found in Security Description.

#### Analytics

Periodically, the Mobile Client will send the user’s Province of Domicile and some Epidemiological Infos (if any are available) and Technical Infos to the Analytics Service. This information is critical for the National Healthcare Service to operate Immuni effectively, thereby maximising the containment of the epidemic and patient care, as explained in [High-Level Description](https://github.com/immuni-app/immuni-documentation/blob/master/README.md).

To protect user privacy, the Epidemiological Infos have a number of limitations, which are enforced by the A/G Framework. For example, the operating system alerts the user every time the App requests the Exposure Infos for a given Exposure Detection Summary. Also, the duration of each Exposure is measured in five-minute increments and capped at 30 minutes. Moreover, Immuni has no way to determine that multiple Exposures on different days may have occurred with the same infected user.

Furthermore, we are working on a system to allow the Mobile Client to upload such data without requiring any sort of user authentication (including phone number or email address verification), user identifiers, or device identifiers. At the same time, the system protects the integrity of the data from large-scale pollution caused by attackers sending fake data. The system will not work for all devices—the server will discard untrustworthy data. We will share the relevant documentation soon.

### A/G Framework <a name="ag-framework" />

For the implementation of the [Exposure detection and notification flow](#exposure-detection-and-notification-flow), Immuni leverages the A/G Framework (see [Apple’s documentation](https://www.apple.com/covid19/contacttracing) and [Google’s documentation](https://www.google.com/covid19/exposurenotifications/)).

Part of the functionality is provided directly by the operating system, which results in enhanced user privacy:

* The RPI Database is handled by the operating system. As such, the App does not have access to information such as which specific TEK triggered a Match, if any. Without notifying the user, the App can only access the Exposure Detection Summary for a set of TEKs. This offers high-level information on the Exposures for those TEKs. To access the Exposure Infos—which offer a bit more detail—the App has to trigger a notification to the user that is handled by the operating system and cannot be bypassed. Moreover, the operating system limits the App’s access to this information in various ways, including through aggregations and rate limits on the relevant API.
* The App must obtain the user’s permission to get their TEKs so that it can upload them to the Exposure Ingestion Service. The entire authorisation flow is triggered by the App and handled by the operating system. The App has no way to bypass this security measure.

On top of the privacy enhancements mentioned above, electing to leverage the A/G Framework unlocks a number of other advantages that we do not believe would be achievable in a custom implementation. These include:

* Allowing reliable background broadcasting and detection of RPIs, especially between two iOS Mobile Clients, which is severely constrained by iOS’s [Core Bluetooth API](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html). There are several workarounds to mitigate this limitation. In our view, however, most of them are ineffective, energy inefficient, not compliant with Apple’s [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/), or all of the above.
* Synchronising the RPI rotation with the Bluetooth MAC address rotation, so these two pieces of information cannot be cross-referenced to identify a user.
* Ensuring the iOS App automatically resumes its normal operation after an operating system reboot.
* Minimising battery consumption.

### iOS App

This section discusses the iOS-specific aspects of the App.

The iOS App will be available for devices running iOS 13. However, iOS 13.5 is required for the A/G Framework to work and therefore the Exposure detection and notification flow to function properly. To date, Apple has not announced the intention to release it for earlier iOS versions. Hence, the iOS App will prevent the user from completing the onboarding if they have not updated to iOS 13.5.

#### iOS App technologies

The App is written using Swift 5.2 and XCode 11.5.

The development process leverages the following technologies:

* **[Brew](https://brew.sh/).** A package manager for macOS, Brew is released as an open-source project under the BSD 2-Clause 'Simplified' licence.
* **[CocoaPods](http://cocoapods.org/).** The most widespread dependency manager in the iOS community, CocoaPods is released as an open-source project under the MIT licence.
* **[CommitLint](https://commitlint.js.org/#/).** A linter for commits, CommitLint is released as an open-source project under the MIT licence.
* **[Danger](https://danger.systems/js/).** A system to enforce code quality and contribution rules in pull requests, Danger is released as an open-source project under the MIT licence.
* **[SwiftFormat](https://github.com/nicklockwood/SwiftFormat/).** A formatter for Swift code, SwiftFormat is released as an open-source project under the MIT licence.
* **[SwiftGen](https://github.com/SwiftGen/SwiftGen).** A tool that generates type-safe Swift code for accessing project resources, SwiftGen is released as an open-source project under the MIT licence.
* **[SwiftLint](https://github.com/realm/SwiftLint).** A linter for Swift code, SwiftLint is released as an open-source project under the MIT licence.
* **[XcodeGen](https://github.com/yonaskolb/XcodeGen).** A tool that generates Xcode projects starting from a YAML definition, XcodeGen is released as an open-source project under the MIT licence.

The App uses a number of native technologies provided by Apple's iOS SDK (e.g., [Foundation](https://developer.apple.com/documentation/foundation) and [UIKit](https://developer.apple.com/documentation/uikit)).

When it comes to the iOS App’s architecture, Immuni combines a unidirectional data flow for the business logic with a MVVM architecture for the user interface. This structure is the result of combining two libraries:

* **[Katana](https://github.com/BendingSpoons/katana-swift).** A library that provides structure to the flow of information concerning the business logic of a mobile app, Katana is developed and maintained by Bending Spoons and released as an open-source project under the MIT licence.
* **[Tempura](https://github.com/BendingSpoons/tempura-swift/).** Tempura is a library for organising the user interface of a mobile app. Building on top of Katana, Tempura reduces the boilerplate needed to render the current state of the app. Tempura is developed and maintained by Bending Spoons and released as an open-source project under the MIT licence.

Finally, Immuni leverages other open-source libraries, including:

* **[Alamofire](https://github.com/Alamofire/Alamofire).** The de-facto standard for performing HTTP(S) requests on iOS, Alamofire is released as an open-source project under the MIT licence.
* **[BonMot](https://github.com/Rightpoint/BonMot).** A library that abstracts the complexity of creating and styling textual content within an iOS app, BonMot is released as an open-source project under the MIT licence.
* **[Hydra](https://github.com/malcommac/Hydra/).** A promise-pattern implementation that also features constructs such as *async/await,* Hydra is used to implement the app’s business logic. It is released as an open-source project under the MIT licence.
* **[Lottie](https://github.com/airbnb/lottie-ios).** A library for natively rendering vector-based animations and art in realtime, Lottie is developed and maintained by Airbnb and released as an open-source project under the Apache 2.0 licence.
* **[PinLayout](https://github.com/layoutBox/PinLayout).** A library that permits writing user interface layout code leveraging a series of chainable function calls, PinLayout is released as an open-source project under the MIT licence.
* **[SwiftLog](https://github.com/apple/swift-log/).** A logging package for Swift, SwiftLog is released by Apple under the Swift open-source effort. It is important to note that logs are never stored or sent outside the Mobile Client. Moreover, logs are disabled in production builds. SwiftLog is released as an open-source project under the Apache 2.0 licence.

#### iOS App security

Security is one of the most critical topics when it comes to Immuni. What follows is a discussion of some of the measures taken to protect user data, both while stored on the iOS Mobile Client and when sent to any of the Backend Services.

Following Katana's architecture, most of the data that the iOS App uses are retained in a single atom called the *Store.* The Store’s content is persisted in a file stored on the iOS Mobile Client after being encrypted using [AES](https://developer.apple.com/documentation/cryptokit/aes/gcm). The encryption key is generated on the iOS Mobile Client and stored in its [keychain](https://developer.apple.com/documentation/security/keychain_services). Moreover, the file in which the Store is persisted is further protected using the [encryption API](https://developer.apple.com/documentation/uikit/protecting_the_user_s_privacy/encrypting_your_app_s_files) available on iOS.

Some data are also stored in the iOS Mobile Client’s [UserDefaults](https://developer.apple.com/documentation/foundation/userdefaults). The App provides a layer that encrypts this information as well. As with the Store’s content, the encryption key is generated on the iOS Mobile Client and stored in its keychain.

The cryptographic operations that the App performs are implemented using the [Apple CryptoKit](https://developer.apple.com/documentation/cryptokit) framework.

Finally, every communication with the server is established using HTTPS. The Mobile Client only accepts certificates signed by the chosen certificate authority.

### Android App

This section discusses the Android-specific aspects of the App.

The Android App will be available for devices running Android 6 (Marshmallow, API 23) or above. For the A/G Framework to work, the user needs to have updated Google Play Services to version 20.18.13 or above. Therefore, the Android App will prevent the user from completing the onboarding if they do not have a sufficiently recent version of Google Play Services.

#### Android App technologies

The Android App is written using Kotlin 1.3 and Android Studio 3.6.

Both the build process and the dependency management are handled by [Gradle](https://gradle.org/), an open-source project released under the Apache 2.0 licence that has seen widespread adoption by the Android community.

The development process leverages also the following technologies:

* **[CommitLint](https://commitlint.js.org/#/).** A linter for commits, CommitLint is released as an open-source project under the MIT licence.
* **[Danger](https://danger.systems/js/).** A system to enforce code quality and contribution rules in pull requests, Danger is released as an open-source project under the MIT licence.
* **[ktlint](https://ktlint.github.io/).** A linter for Kotlin code, ktlint is maintained by Pinterest and released as an open-source project under the MIT licence.

The source code is implemented leveraging native technologies offered by the Android SDK, as well as third-party libraries which we outline below.

The architecture is built on [Android Architecture Components](https://developer.android.com/topic/libraries/architecture/). When it comes to the project’s structure, Immuni is implemented using a data-driven MVVM architecture that follows the recommendations laid out in the [Android guide to app architecture](https://developer.android.com/jetpack/docs/guide). To achieve this, the Android App is based on the following technologies:

* **[Kotlin Coroutines](https://github.com/Kotlin/kotlinx.coroutines), [Channels](https://kotlinlang.org/docs/reference/coroutines/channels.html), and [Flows](https://kotlinlang.org/docs/reference/coroutines/flow.html).** Kotlin Coroutines, Channels, and Flows are used for asynchronous programming, reactive streams of data, and [structured concurrency](https://en.wikipedia.org/wiki/Structured_concurrency). Support for these functionalities is supplied by the *kotlinx.coroutines* library, which is released as an open-source project under the Apache 2.0 licence.
* **[LiveData](https://developer.android.com/topic/libraries/architecture/livedata) and [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel).** LiveData are observable data holders, while ViewModels are designed to store and manage user interface related data in a lifecycle-conscious way. They both are part of the Android Jetpack set of libraries, and are released as open-source projects under the Apache 2.0 licence.
* **[Navigation Component](https://developer.android.com/guide/navigation).** Navigation Component is used to create single activity apps, reducing complexity and ensuring a consistent and predictable user experience by adhering to an established set of [principles of navigation](https://developer.android.com/guide/navigation/navigation-principles). It is part of the Android Jetpack set of libraries, and is released as an open-source project under the Apache 2.0 licence.
* **[Room](https://developer.android.com/topic/libraries/architecture/room).** The Room persistence library provides an abstraction layer over SQLite to allow for more robust database access. It can also expose data as LiveData/Flow, allowing for reactive-style programming.
* **[Work Manager](https://developer.android.com/topic/libraries/architecture/workmanager).** The Work Manager API makes it easy to schedule deferrable, asynchronous tasks that are expected to run even if the Android App exits or the device restarts. It is part of the Android Jetpack set of libraries, and is released as an open-source project under the Apache 2.0 licence.

Additionally, the Android App uses the following libraries:

* **[Glide](https://github.com/bumptech/glide).** An image loading and caching library for Android focused on smooth scrolling, Glide is developed and maintained by Google and released as an open-source project under the BSD, MIT, and Apache 2.0 licences.
* **[Koin](https://github.com/InsertKoinIO/koin).** A dependency injection framework for Kotlin, Koin is released as an open-source project under the Apache 2.0 licence.
* **[Lottie](https://github.com/airbnb/lottie-android).** A library for natively rendering vector-based animations and art in realtime, Lottie is developed and maintained by Airbnb and is released as an open-source project under the Apache 2.0 licence.
* **[Moshi](https://github.com/square/moshi).** A modern JSON library for Kotlin and Java, Moshi is developed and maintained by Square Inc. and released as an open-source project under the Apache 2.0 licence.
* **[MockK](https://github.com/mockk/mockk).** A mocking library for Kotlin, MockK is released as an open-source project under the Apache 2.0 licence.
* **[NumberPicker](https://github.com/ShawnLin013/NumberPicker).** An Android library that provides a simple and customisable number picker, NumberPicker is released as an open-source project under the MIT licence.
* **[OkHttp](https://github.com/square/okhttp/).** An efficient, low-level HTTP(S) client for Java and Kotlin, OkHttp offers a robust foundation to high-level HTTP(S) clients like Retrofit. It is developed and maintained by Square Inc. and released as an open-source project under the Apache 2.0 licence.
* **[Retrofit](https://github.com/square/retrofit).** A high-level, type-safe HTTP(S) client for Android and Java, Retrofit is developed and maintained by Square Inc. and released as an open-source project under the Apache 2.0 licence.
* **[SQLCipher](https://github.com/sqlcipher/android-database-sqlcipher).** An Android SQLite API, SQLCipher allows the storage of encrypted data. The SQLCipher code itself is licensed under a BSD-style licence by Zetetic LLC.

#### Android App security

Security is one of the most critical topics when it comes to Immuni. What follows is a discussion of some of the measures taken to protect user data, both while stored on the Android Mobile Client and when sent to any of the Backend Services.

The Android App stores data inside a SQLite local database. The SQLCipher library is used to encrypt the database using AES256. The encryption key is generated on the Android Mobile Client and stored in its [keystore](https://developer.android.com/training/articles/keystore).

Some data are also stored on the Android Mobile Client’s [SharedPreferences](https://developer.android.com/training/data-storage/shared-preferences). The Android Mobile Client provides a layer that encrypts this information using AES256. Like the database content, the encryption key is generated on the Android Mobile Client and stored in its keystore.

To generate the encryption keys on the Android Mobile Client, we use [Jetpack](https://developer.android.com/topic/security/data).

On devices that support them, [Full-Disk Encryption](https://source.android.com/security/encryption/full-disk) and [File-Based Encryption](https://source.android.com/security/encryption/file-based) add a further level of security to the stored data.

Finally, every communication with the server is established using HTTPS. The Mobile Client only accepts certificates signed by the chosen certificate authority.

### App build verification

An important part of building trust around the project is enabling the community to verify that the source code published in the open-source repositories matches that from which the apps made available on the production environments—the App Store and Google Play—are built.

To reach this goal, we must ensure that:

* The build process is publicly available
* The builds are reproducible

Before digging into the details of these two topics, it is important to mention that this is an ongoing effort. We look forward to collaborating with security experts and researchers to further improve the system.

#### Open build process

An *open build process* is a process that is publicly available for building and uploading Immuni builds to App Store Connect and Google Play Console. This means both making the implementation of the system accessible to the community, and enabling people to inspect logs to verify what the build process actually does. Moreover, the result of the build process should be available for download in the public repository so that it can be used for further analysis by anyone interested in the topic.

Building and uploading builds in such a way that all the tools are publicly verifiable (e.g., open source or provided by Apple and Google), and impossible for the developers to alter, ensures that the production builds are created starting from the public repository. Once a build with a specific combination of application identifier, build number, and release version is uploaded to App Store Connect or Google Play Console, no other build may have the same combination.

On iOS, this triple is determined by the *bundle identifier* (*CFBundleIdentifier*), *bundle version* (*CFBundleVersion*), and *release version* (*CFBundleShortVersionString*). On Android, the same three fields are called *application identifier* (*applicationId*), *version number* (*versionCode*), and *release version* (*versionName*).

On both platforms, the application identifier and the release version are publicly available, while the build number is not. However, this can be easily extracted from the App binaries downloaded from the respective app stores, as it is contained in an unencrypted file (the *Info.plist* file on iOS, and the *AndroidManifest.xml* file on Android).

For both platforms, the build process leverages the following technologies:

* **[Fastlane](https://fastlane.tools/).** A platform to orchestrate the build and release process of iOS and Android apps, Fastlane is released as an open-source project under the MIT licence.
* **[GHR](https://deeeet.com/ghr/).** A command to create GitHub releases, GHR is released as an open-source project under the MIT licence.

As a final note, having an open build process does not mean that everyone will be able to access the build system secrets. There are some pieces of information that must be kept secret. For example, we can not share the App Store Connect credentials, otherwise anyone would be able to publish an update of the App on the App Store. However, the commands launched in the build process and, most importantly, the effects and artifacts generated by these commands, are publicly available. Therefore, they confirm the integrity and consistency of the published version.

#### Reproducible builds

As the [Reproducible Builds website](https://reproducible-builds.org/tools/) states, *reproducible builds* are a set of software development practices that create an independently verifiable path from source to binary code.

In the context of Immuni, reproducibility would offer security researchers an additional way to verify that builds have been made from commits of the publicly available repositories. In fact, reproducibility would exceed the assurance offered by the open build process by letting security researchers compare the App Store and Google Play binaries with binaries generated on their own computers.

Below, we discuss the efforts we have made towards achieving reproducibility of builds on both platforms.

**iOS**  
Reproducible builds in an iOS context are complex. Even with the same environment (e.g., operating system and compiler), builds may differ from run to run. Moreover, Apple requires that App Store builds be signed using a sophisticated [code signing system](https://developer.apple.com/support/code-signing/). Finally, iOS App builds pass through a process called [app thinning](https://developer.apple.com/documentation/xcode/reducing_your_app_s_size) that further modifies the uploaded artifacts before the final builds are made available on the App Store for download.

We have been working on this problem, also leveraging [Telegram’s great work](https://core.telegram.org/reproducible-builds#reproducible-builds-for-ios), but we have not yet found a satisfactory solution. At the current stage, we have managed to successfully compare an App Store build with the *xcarchive* that generated it. However, the comparison starts from an already compiled artifact and requires the iOS App code signing certificates. For security reasons, these are not publicly available.

We expect more work to be done in this area, and look forward to improving the process in the future.

**Android**  
The reproducibility of Android builds is affected by similar problems as on iOS, as builds may differ from run to run even with the same building environment.

These are the main aspects that may limit reproducibility:

* The building environment must be exactly the same (e.g., Android Studio, Gradle version, and plugins version)
* The *resources.arsc* file can be different across builds of the same commit due to a known [issue](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=901956) regarding file ordering
* Production builds are signed with certificates that are kept secret, for security reasons
* The Android App bundle format generates different APKs for different devices and device configurations, while also introducing some bundletool-specific files that are different for each build (e.g., *META-INF/BNDLTOOL.SF* and *META-INF/BNDLTOOL.RSA*)
* The *AndroidManifest.xml* file of production builds could be slightly different from that of independent builds made from the same commit, due to changes performed by Google Play during app publishing

However, some of these limitations can be addressed:

* APKs can be compared with a [Python script](https://github.com/DrKLO/Telegram/blob/master/apkdiff.py) maintained by Telegram as part of their effort on reproducible builds, or with tools such as [apkanalyzer](https://developer.android.com/studio/command-line/apkanalyzer)
* The APK signature does not modify existing files inside the APK, but only adds three new files which can be skipped during the comparison (*META-INF/MANIFEST.MF, META-INF/CERT.RSA,* and *META-INF/CERT.SF*)
* Device-specific APKs can be generated from the published Android App bundle using [bundletool](https://developer.android.com/studio/command-line/bundletool)

To increase transparency, the code is neither shrunk nor obfuscated, making it easier for security experts to decompile the bytecode and compare it with the published source code.

At present, we have managed to successfully match different builds of the same commit but for a few files. However, we recognise that more work needs to be done to achieve a complete solution, and we are eager to collaborate with experts in relevant fields to improve the process.

## Backend Services

Below, we describe the technologies with which the Backend Services are built. Then, we provide a more detailed description of each Backend Service, including the documentation of the APIs and data models.

### Backend Services technologies

All Backend Services are implemented in Python 3.8, with [Poetry](https://python-poetry.org/) as dependency manager. They use [Redis](https://redis.io/), an in-memory data store, as a message broker for communication between them. The persistence layer for the relevant data is provided by [MongoDB](https://www.mongodb.com/), a non-relational database.

The backend services follow a micro-service architecture, where each critical functionality is deployed as its own component. Components are distributed in dedicated [Docker](https://www.docker.com/) images, Docker being an industry standard platform for the containerization and virtualization of software.

To monitor the performance and stability of all backend services and therefore guarantee a better availability of such a critical project, the code is instrumented with [Prometheus](https://prometheus.io/), an open-source software monitoring solution.

The following dependencies are used to implement the business logic:

* **[Aioredis](https://aioredis.readthedocs.io/).** An async client for Redis interaction, Aioredis is released as an open-source project under the MIT licence.
* **[Celery](http://www.celeryproject.org/).** An asynchronous task framework to execute delayed or scheduled tasks, Celery is released as an open-source project under the BSD-3 licence.
* **[Decouple](https://github.com/henriquebastos/python-decouple/).** A library to easily inject configuration via static files or environment variables, Decouple is released as an open-source project under the MIT licence.
* **[Marshmallow](https://pypi.org/project/marshmallow/).** A library to easily manage and validate endpoints schemas, Marshmallow is released as an open-source project under the MIT licence.
* **[MongoEngine](http://mongoengine.org/).** A document-object mapper for working with MongoDB from Python, MongoEngine is released as an open-source project under the MIT licence.
* **[Pymongo](https://docs.mongodb.com/drivers/pymongo).** A low-level library to interact with MongoDB, Pymongo is released as an open-source project under the Apache 2.0 licence.
* **[Sanic](https://github.com/huge-success/sanic).** A fast, async web framework to implement web APIs, Sanic is released as an open-source project under the MIT licence.
* **[Uvicorn](https://www.uvicorn.org).** An async web server, Uvicorn is released as an open-source project under the BSD-3 licence.

To increase the security, robustness, and quality of the code, we integrated several established tools into our development pipeline:

* **[Bandit](https://github.com/PyCQA/bandit).** A tool designed to find common security vulnerabilities in Python code, it is maintained by the Python Code Quality Authority and released as an open-source project under the Apache 2.0 licence.
* **[Black](https://github.com/psf/black).** A code formatter for Python, Black is released as an open-source project under the MIT licence.
* **[Flake8](https://flake8.pycqa.org/en/latest/).** A tool to enforce consistent coding style rules, Flake8 is maintained by the Python Code Quality Authority and released as an open-source project under the MIT licence.
* **[Isort](https://isort.readthedocs.io/en/latest/).** A utility to sort Python imports alphabetically, isort is released as an open-source project under the MIT licence.
* **[Mypy](https://mypy.readthedocs.io/en/stable/).** A static type checker for Python, mypy is released as an open-source project under the MIT licence.
* **[Pylint](https://www.pylint.org/).** A linter and static code analysis tool for Python, pylint is maintained by the Python Code Quality Authority and released as an open-source project under the GNU General Public Licence v2.
* **[Pytest](https://docs.pytest.org/en/latest/).** A framework to write tests in Python, pytest is released as an open-source project under the MIT licence.

### App Configuration Service

The App can be partially customised through the Configuration Settings downloaded at the App startup and updated every time the App starts a session, whether in the foreground or background. For example, these include information such as the weights to be used by the Mobile Client in the calculation of the Total Risk Score.

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
`Content-Type: application/json`

**Response body**  
The response body is a JSON-formatted dictionary. We are still finalising the relevant specifications.

### OTP Service

The OTP Service provides an API to the National Healthcare Service for authorising OTPs that can be used to upload data from the Mobile Client via the Exposure Ingestion Service. The OTP is generated by the App and communicated by the user to the Healthcare Operator (e.g., during a phone call). The Healthcare Operator inserts the OTP into the HIS, which then registers the OTP on the OTP Service.

The OTP automatically expires after a defined interval. If the data have not been uploaded by the time the OTP expires, the user will have to restart the process with a new OTP.

#### OTP format

The OTP is composed of 10 uppercase alphanumeric characters, with the last character being a check digit.

To prevent misunderstandings when the user communicates the OTP to the Healthcare Operator, the characters used in the OTP are limited to the following subset: *A, E, F, H, I, J, K, L, Q, R, S, U, W, X, Y, Z, 1, 2, 3, 4, 5, 6, 7, 8, 9.* This leads to 25^9 possible valid combinations (since the last character is computed deterministically, based on the previous 9 characters).

#### API - Authorise OTP <a name="api-authorise-otp" />

**Caller**  
HIS

**Description**  
The provided OTP authorises the upload of the Mobile Client’s TEKs. This API will not be publicly exposed, to prevent unauthorised users from reaching it. Authentication for having the OTP Service and the HIS trust each other is configured at the infrastructure level. The payload also contains the start date of the symptoms, so that the Exposure Ingestion Service can compute the Transmission Risk for each uploaded TEK.

**Resource hostname**  
The API is only accessible by the HIS

**Resource path**  
`POST /v1/authorise-otp`

**Request body**  

```
{
	"otp": string,
	"symptoms_started_on": string
}
```

### Exposure Ingestion Service

The Exposure Ingestion Service provides an API for the Mobile Client to upload its TEKs for the previous 14 days, in the case that the user tests positive for SARS-CoV-2 and decides to share them. Contextually, the Mobile Client uploads the Epidemiological Infos from the previous 14 days, if any. If some Epidemiological Infos are indeed uploaded, the user’s Province of Domicile is uploaded too. The upload can only take place with an authorised OTP.

The Exposure Ingestion Service is also responsible for periodically generating the TEK Chunks to be published by the Exposure Reporting Service. The TEK Chunks are assigned a unique incremental index and are immutable. They are generated periodically as the Mobile Clients upload new TEKs.

TEK Chunks older than 14 days are automatically deleted from the database by an async cleanup job.

Province of Domicile and Epidemiological Infos are forwarded to the Analytics Service.

#### API - Validate OTP <a name="api-validate-otp" />

**Caller**  
Mobile Client

**Description**  
The Mobile Client validates the OTP prior to uploading data. The request is authenticated with the OTP to be validated. Using the dedicated request header, the Mobile Client can indicate to the server that the call it is making is a dummy one. The server will ignore the content of such calls.

**Resource hostname**  
`upload.immuni.gov.it`

**Resource path**  
`POST /v1/ingestion/validate-otp`

**Request headers**  
`Authorization: Bearer <SHA256(OTP)>`  
`Content-Type: application/json`  
`Exp-Dummy-Data: <true|false>`

#### API - Upload TEKs <a name="api-upload-teks" />

**Caller**  
Mobile Client

**Description**  
Once it has validated the OTP, the Mobile Client uploads its TEKs for the past 14 days, together with the user’s Province of Domicile. If any Epidemiological Infos from the previous 14 days are available, the Mobile Client uploads those too. The timestamp that accompanies each TEK is referred to the Mobile Client’s system time. For this reason, the Mobile Client informs the Exposure Ingestion Service about its system time so that a skew can be corrected.

**Resource hostname**  
`upload.immuni.gov.it`

**Resource path**  
`POST /v1/ingestion/upload`

**Request headers**  
`Authorization: Bearer <SHA256(OTP)>`  
`Content-Type: application/json`  
`Exp-Client-Clock: <UNIX epoch time>`  
`Exp-Dummy-Data: <true|false>`

**Request body**  

```
{
	"teks": [ 
	{
		"key_data": string,
		"rolling_start_number": int,
		"rolling_period" : int
	}, ...
	],
	"province": string, 
	"exposure_detection_summaries": [
	{
    	"matched_key_count": int,
    	"days_since_last_exposure": int,
    	"attenuation_durations": array[int],
    	"maximum_risk_score": int
	}, ...
	],
	"exposure_infos": [
	{
    	"date": date,
		"duration": int,
		"attenuation_value": int,
		"attenuation_durations": array[int],
  		"transmission_risk_level": int,
		"total_risk_score": int
    }, ...
	]
}
```

#### Data model - Uploads <a name="data-model-uploads" />

**Description**  
The *uploads* collection stores the TEKs uploaded by a Mobile Client with an authorised OTP, together with information on the day of symptoms onset, if any. *Uploads* can be automatically deleted after 14 days by filtering on the generation time of *ObjectId.* *to_publish* indicates whether the data still needs to be processed to be included in a new TEK Chunk. *teks* is an array of TEKs. *key_data* is the base64-encoding of the 16-byte TEK. Any Epidemiological Infos are forwarded to the Analytics Service.

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
The *TEKChunks* collection holds the TEK Chunks generated by the Exposure Ingestion Service, and it is also read-accessible by the Exposure Reporting Service so that it can return the available TEK Chunks to the Mobile Clients. TEK Chunks can be automatically deleted after 14 days by filtering on the generation time of *ObjectId.* *index* is the incremental TEK Chunk index. *teks* is an array of TEKs. *key_data* is the base64-encoding of the 16-byte TEK.

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

The Exposure Reporting Service makes the TEK Chunks created by the Exposure Ingestion Service available to the Mobile Client. Only TEK Chunks for the last 14 days are made available.

To avoid downloading the same TEKs multiple times, the Mobile Clients fetch the indexes of the TEK Chunks available to download first. Then, they only actually download TEK Chunks with indexes for which TEK Chunks have not been downloaded before.

#### API - Fetch TEK Chunk indexes <a name="api-fetch-tek-chunk-indexes" />

**Caller**  
Mobile Client

**Description**  
Return the index of the oldest relevant TEK Chunk (no older than 14 days) and the index of the newest available TEK Chunk. It is up to the Mobile Client not to download the same TEK Chunk twice.

**Resource hostname**  
`get.immuni.gov.it`

**Resource path**  
`GET /v1/reporting/indexes`

**Response headers**  
`Content-Type: application/json`  
`Cache-Control: public, max-age=1800`

**Response body**  

```
{
	"oldest": int,
	"newest": int
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
`GET /v1/reporting/<TEKChunkIndex>`

**Response headers**  
`Content-Type: application/x-protobuf`  
`Cache-Control: public, max-age=1296000`

**Response body**  
We have elected not to provide an example of a response body here, as it would hardly be readable or informative in this context.

### Analytics Service

*Section under construction.*

## Open points

These are some of the most pressing points on which we are working:

* **Dummy traffic.** We would like to minimise the information that an attacker could gain by analysing network traffic. We are finalising decisions on dummy uploads.
* **Analytics integrity.** We are trying to collect the necessary Epidemiological Infos and Technical Infos while preserving user privacy to the maximum possible extent. However, performing such collection without asking the user to authenticate (e.g., by verifying a phone number or email address) and without using any user or device identifier makes it harder to prevent attackers from polluting the data with fake uploads. We have a promising solution under development that we expect to work for a significant portion of devices.
* **Data retention, retrieval, and erasure.** We are finalising the specifics of these important policies.
