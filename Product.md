# Immuni's Product

## Table of contents

- [Context](#context)
- [Initialisation](#initialisation)
  - [Language detection](#language-detection)
  - [Required update](#required-update)
- [Onboarding](#onboarding)
  - [Welcome](#welcome)
  - [Privacy](#privacy)
  - [Province selection](#province-selection)
  - [Permissions](#permissions)
- [Home](#home)
  - [App status](#app-status)
  - [Exposure notification and suggestions](#exposure-notification-and-suggestions)
- [Settings](#settings)
  - [Data upload](#data-upload)
  - [FAQ](#faq)

## Introduction

This document describes the features and user flows made available by the Immuni app. It assumes that you have already read [High-Level Description](/README.md).

Each section covers a specific part of the app. Features are described from the user’s point of view, using sentences in the first-person voice. When required, more details on _how_ the app delivers that particular feature are provided as sub-points. Note that this document does not attempt to define the specific details of the user experience or of the user interface. However, both are designed following the requirements laid out below.

In most of the cases, screenshots demonstrate how features are displayed on screen, based on the latest design of the app’s user interface.

## Initialisation

### Language detection

I use the app in my preferred system language. If the app does not offer that language, it defaults to British English.

- The app is available in Italian, British English, German, French, and Spanish.
- If the app detects a match between the user’s language preferences and one of the available languages, the app uses that language.
- If no such match is found, British English is used as the default.

### Required update

I am notified if my version of iOS or Google Play Services does not support the Apple and Google Exposure Notification framework. I am given the most straightforward way to update it, if necessary.

- Immediately upon being opened, if the app detects that an iOS or Google Play Services update is needed, a non-dismissible screen appears, inviting the user to perform the update.
- As soon as the user returns to the app after the update is complete, the screen disappears.

I am notified if I need to update the app. If that is the case, I am given the most straightforward way to do so.

- The app fetches the indication of the minimum version the server supports on a regular basis.
- If the version of the app is prior to the minimum version supported, when opening the app, a non-dismissible screen informs the user that they need to download the latest version.
- As soon as the user returns to the app after the update, that screen is no longer shown.

![Required Update](/Design/RequiredUpdate.png)

## Onboarding

### Welcome

I receive a brief explanation of why the app is useful and how it works.

- The app explains its benefits for both the individual user and the wider community.
- The app explains how it works, with a simplified description of the exchange of rolling proximity identifiers via Bluetooth and match detection.
- The app explains how privacy is preserved for all parties involved: Immuni does not know the identity of the user or of the people with whom the user comes into contact.

![Welcome](/Design/Welcome.png)  
![How It Works](/Design/HowItWorks.png)

### Privacy

I am informed of what data the app processes and, more generally, how my privacy is protected.

- The app lists data that it does not collect or process at all (e.g., geolocation, user identity, and contact information).
- The app describes some of the ways it protects the user’s privacy (e.g., by encrypting data and server connections).
- The app lists data that it collects and shares, and for what purposes it does so.

I read the app’s privacy notice, accept the terms of use, and confirm that I am at least 14 years old.

- The app provides links to access both its privacy notice and terms of use.
In order to continue, confirmation of being 14 or older, having read the privacy notice, and having read and accepted the terms of use is required.

![Privacy](/Design/Privacy.png)

### Province selection

I state the province in which I live, with full awareness of the purpose for which this information is used.

- The app shows a screen that allows the user to select from a list the region in which they live.
- Based on the region the user chooses, the app shows a screen that allows the user to select the province in which they live.
- Both screens inform the user about the purpose for which the region and province data are used.

![Province Selection](/Design/ProvinceSelection.png)

### Permissions

I provide all the permissions and I activate the device services required for the app to work correctly.

- The app asks for the user’s permission to use the Apple and Google Exposure Notification framework.
- If Bluetooth is inactive, the app notifies the user that it cannot work and provides simple instructions on how to reactivate it.
- The app asks for the user’s permission to send them push notifications.

![Permissions](/Design/Permissions.png)

## Home

### App status

I check if the app works as intended and, if that is not the case, I easily take action to restore its correct behaviour.

- In the Home screen, a card informs the user if everything works correctly.
- If the user withdrew any permission or disabled services that are necessary for the app to operate, the app displays a card informing them that the app is not working correctly. The card also provides guidance for the user to fix every issue detected, in a fashion similar to that which takes place during the onboarding process.

![App Status](/Design/AppStatus.png)

### Exposure notification and suggestions

I can receive generic advice on what to do to protect myself from the spread of COVID-19.

I am notified if I am at risk due to recent contact with a SARS-CoV-2-positive user, and I receive advice on what to do.

- The app advises the user to get in touch with the National Healthcare Service (_Servizio Sanitario Nazionale_) and provides them with methods of doing so.

![Exposure Notification and Suggestions](/Design/ExposureNotificationAndSuggestions.png)

## Settings

I can access the app’s privacy notice and terms of use.

I can contact Customer Support to solve issues I experience with the app or to clarify any doubts I have about it.

- The Customer Support screen shows some information that may help with troubleshooting, such as the version numbers of the app and of the operating system.

I can change my selection of the province where I live, if necessary.

![Settings](/Design/Settings.png)

### Data upload

I can upload my data when the outcome of a SARS-CoV-2 test is available and positive, with the guidance of a healthcare operator.

- The healthcare operator asks the user to tap ‘Report a positive result’ in the ‘Settings’ tab.
- The app shows a random, temporary 10-character alphanumeric code to the user.
- The user dictates the 10-character code to the healthcare operator, who inserts it into a web interface.
- Upon the healthcare operator’s request, the user taps ‘Continue’ to verify the code and proceed to the next step.
- The app shows an explanation of what data is about to be uploaded and why.
- When the user taps ‘Upload data’, the app uploads all the previously described data.
- Finally, the app shows a confirmation message saying that the upload was successful.

![Data Upload](/Design/DataUpload.png)

### FAQ

I can receive comprehensive answers to common questions that I may have about the app.

- Upon tapping ‘Frequently asked questions’, the user is provided with a list of questions.
- Questions are prioritised so that the app can ensure the most pressing and common ones are presented first.
- A search bar allows the user to find relevant questions more easily.
- If the user taps any of them, a separate screen will open to reveal the answer to that question.

![FAQ](/Design/FAQ.png)
