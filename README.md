NOTE: This README is also available [in Italian](/Translations/it/README.md).  
NOTA: Questo README è disponibile [in italiano](/Translations/it/README.md).

# Immuni's High-Level Description

Immuni is the Italian Government’s exposure notification solution, realised by the Special Commissioner for the COVID-19 emergency (Presidency of the Council of Ministers), in collaboration with the Ministry of Health and the Ministry for Technological Innovation and Digitalisation. It only uses public infrastructures located within the national borders. It is exclusively managed by the public company Sogei S.p.A. The source code prior to October 13, 2020 has been developed for the Presidency of the Council of Ministers by Bending Spoons S.p.A., and it is released under a GNU Affero General Public License version 3.

## Table of contents

- [Executive summary](#executive-summary)
- [Introduction](#introduction)
- [Vision and goals](#vision-and-goals)
- [Principles](#principles)
- [Product](#product)
  - [How it works](#how-it-works)
- [Analytics](#analytics)
  - [Epidemiological information](#epidemiological-information)
  - [Operational information](#operational-information)
- [Privacy](#privacy)

## Executive summary

- Immuni is a technological solution that centres on **an iOS and Android smartphone app.** It helps us to fight the COVID-19 epidemic by notifying users at risk of carrying the virus as early as possible—**even when they are asymptomatic.** These users can then isolate themselves to avoid infecting others, and seek medical advice.
- Immuni’s design and development are based on six main principles: **utility, accessibility, accuracy, privacy, scalability,** and **transparency.**
- It features an **exposure notification** system that leverages **Bluetooth Low Energy:**
  - When two users come sufficiently close to each other for long enough, their devices record each other’s _rolling proximity identifier_ in local memory. Rolling proximity identifiers are generated from _temporary exposure keys_ and change multiple times each hour. Temporary exposure keys are **generated randomly** and change once per day.
  - When a user tests positive for SARS-CoV-2, the virus causing COVID-19, they have the option to upload to a server their recent temporary exposure keys. This operation can only happen with the validation of a **healthcare operator.**
  - The app periodically downloads the new temporary exposure keys and uses them to derive the infected users’ rolling proximity identifiers for the recent past. It then matches the identifiers against those stored in the device’s memory and **notifies the user** if a risky exposure has occurred.
  - The system uses **no geolocation data** whatsoever, including GPS data. So, the app cannot tell where the contact with a potentially contagious user took place, nor the identities of those involved.
- To implement its exposure notification functionality, Immuni leverages **the Apple, Google and HMS Core Exposure Notification framework.** This allows Immuni to be more reliable than otherwise would be possible.
- Besides the temporary exposure keys, the Immuni app also sends to the server some analytics data. These include **epidemiological and operational information,** and are sent for the purpose of helping the National Healthcare Service (_Servizio Sanitario Nazionale_) to provide effective assistance to users.
- Immuni is being developed while paying a lot of attention to user privacy and a number of measures have been taken to protect it. For example, the app collects **no personal data that would disclose the user’s identity,** such as the user’s name, age, address, email, or phone number.

## Introduction

The whole world is united by a determination to stop the spread of COVID-19, the disease caused by SARS-CoV-2. The pandemic is threatening people’s health and severely damaging economies on a global scale.

Many experts agree that, in the future, new pandemics are a distinct possibility. Some may become even more dangerous to humanity than the one we are currently battling.

In this challenging context, the contribution of technological innovation can be decisive. Immuni is one of a number of tools deployed and initiatives taken by the Italian government to help slow down the spread of the disease and accelerate the return to everyday life.

This document provides a high-level description of Immuni—it is a good idea to read it first. More detailed information can be found in the following documents:

- [Product](/Product.md)
- [Technology](/Technology.md)
- [Application Security](/Application%20Security.md)
- [Privacy-Preserving Analytics](/Privacy-Preserving%20Analytics.md)
- [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md)

We have open-sourced Immuni’s software under the [GNU Affero General Public License version 3](https://www.gnu.org/licenses/agpl-3.0.en.html).

## Vision and goals

Immuni is a technological solution that centres on a smartphone app.

It helps us to fight epidemics—starting with COVID-19:

- The app aims to notify users at risk of carrying the virus as early as possible—even when they are asymptomatic.
- These users can then isolate themselves to avoid infecting others. This minimises the spread of the virus while speeding up a return to normal life for most people.
- By being notified early, they can also seek medical advice and lower the risk of serious health consequences.

Immuni is designed to address the current crisis, but the vision behind it is for the tools that are being developed to make us all better prepared in addressing similar threats that may arise in the future.

## Principles

The main principles guiding the Immuni project follow:

- **Utility.** The app needs to be useful in fulfilling the project's vision and goals, as outlined above. The key here is to be able to notify as high a percentage of the people who are substantially at risk as possible, and to do so as early as is practical. This is the most important principle.
- **Accessibility.** For fairness and to facilitate the widest adoption, Immuni should be accessible to the maximum possible number of people who may want to use it. This principle impacts decision-making across the board, including in user experience, design, localisation, and technology.
- **Accuracy.** Immuni aims to notify only those users who have a substantial risk of having contracted the virus. This is important because of the psychological toll that goes with being notified about a potential transmission of the disease, and because too high a rate of false positives would result in users losing trust in the app—and stopping their use of it. Also, the more accurate the app is, the more efficiently the National Healthcare Service (_Servizio Sanitario Nazionale_) will be able to take care of users, making sure to attend to higher-risk ones first.
- **Privacy.** Immuni must protect user privacy while remaining effective. Earning and maintaining user trust is critical—failing to do so reduces the likelihood of widespread adoption.
- **Scalability.** Immuni needs to be widely adopted throughout the country. This requires the system to scale well technologically and the operational burden it places on the National Healthcare Service to be manageable.
- **Transparency.** Everyone should be provided with access to documentation describing Immuni in all its parts and the rationale behind the most important design decisions. Also, the app will be open source. This allows the users to verify that the app works as documented and the expert community to help improve it.

## Product

Immuni is a technological solution that centres on a smartphone app available for iOS and Android.

It features an exposure notification system to help alert potentially SARS-CoV-2-positive users at an early stage. This system keeps track of contact between Immuni users, even when they are total strangers. When a user tests positive for SARS-CoV-2, the app uses this system to notify other at-risk users. The system is based on Bluetooth Low Energy and does not use any geolocation data whatsoever, including GPS data. So, while the app knows that the contact with an infected user took place, how long it lasted, and can estimate the distance that separated the two users, it cannot tell where the contact took place, nor the identities of those involved.

The app then recommends to at-risk users what to do. Recommendations may include self-isolation (which helps minimise the spread of the disease) and contacting their general practitioner (so that the user can receive the most appropriate care and reduce the likelihood of developing severe complications).

As stated, Immuni’s exposure notification system leverages Bluetooth Low Energy. This has some advantages compared to a solution based on location tracking:

- **The system is more accurate.** Geolocalisation, in many contexts, has an accuracy in the order of tens of metres. Meanwhile, Bluetooth Low Energy signals allow for capturing contact occurring within a radius of a few metres of the user. These are the cases of contact relevant to the transmission of SARS-CoV-2.
- **The battery is used more efficiently.** Bluetooth Low Energy excels when it comes to energy efficiency. This is important because it is reasonable to expect that the app’s uninstall rate will be correlated with battery consumption.
- **No geolocation data are required.** Thanks to Bluetooth Low Energy, contact is traced without tracking the users’ location. This may result in the app being more welcomed by the public and could facilitate wider adoption, increasing Immuni’s utility.

To implement its exposure notification functionality, Immuni leverages the Apple, Google and HMS Core Exposure Notification framework (see [Apple’s documentation](https://www.apple.com/covid19/contacttracing) and [Google’s documentation](https://www.google.com/covid19/exposurenotifications/)). This allows Immuni to overcome certain technical limitations, thus being more reliable than otherwise would be possible.

### How it works

Below, a high-level, simplified description of the system is provided. For more details, please study the rest of the Immuni documentation.

Once installed and set up on a device (_device_A_), the app generates a _temporary exposure key._ This key is generated randomly and changes daily. The app also starts transmitting a Bluetooth Low Energy signal. The signal contains a _rolling proximity identifier_ (_ID_A1,_ assumed fixed in this example, for simplicity). This is generated from the current temporary exposure key. When another device (_device_B_) running the app receives this signal, it records _ID_A1_ locally, in its memory. At the same time, _device_A_ records _device_B_’s identifier (_ID_B1,_ also assumed fixed in this example).

If the user of _device_A_ later tests positive for SARS-CoV-2, following the protocol defined by the National Healthcare Service, they have the option to upload to the Immuni server the temporary exposure keys from which the Immuni app can derive the rolling proximity identifiers recently broadcast by _device_A_ (including _ID_A1_). Periodically, _device_B_ checks the new keys uploaded to the server against its local list of identifiers. _ID_A1_ will be a match. The app notifies the user of _device_B_ that they may be at risk and provides advice on what to do next (for example, isolating themselves and calling their general practitioner).

In practice, when it comes to determining whether the user of _device_B_ is at risk, finding that they had been in the proximity of the user of _device_A_ is not enough. Immuni assesses this risk based on the duration of the exposure and the distance between the two devices. This is estimated from the attenuation of the Bluetooth Low Energy signal as received by _device_B._ The longer the exposure and the closer the contact, the higher the risk that a transmission of the virus occurred. Contact lasting only a couple of minutes and happening at several metres of distance will generally be considered to be low risk. The risk model may evolve with time as more information about SARS-CoV-2 becomes available.

It should be noted that the estimation of distance is error-prone. In fact, the attenuation of a Bluetooth Low Energy signal depends on factors such as the orientation of the two devices relative to each other and the obstacles (including human bodies) that lie in between. While leveraging this information is likely useful in increasing the accuracy of Immuni’s assessments of the risk of contagion, wrong assessments will happen with some frequency.

The rolling proximity identifier that is broadcast by the app is generated from random temporary exposure keys and does not contain any information about the device, let alone the user. Moreover, it is _rolling,_ meaning that it changes multiple times each hour, further protecting the privacy of Immuni’s users.

To ensure that only users who actually tested positive for SARS-CoV-2 upload their keys to the server, the upload procedure can only be performed with the cooperation of an authenticated healthcare operator. The operator asks the user to provide a code generated by the app and inputs it into a back-office tool. The upload can succeed only if the code used by the app to authenticate the data corresponds to that entered in the system by the healthcare operator.

## Analytics

Besides the temporary exposure keys, some additional information is sent to the server and analysed to ensure the proper functioning of the system:

- Epidemiological information
- Operational information

These data are important for the National Healthcare Service’s effective management of the system, including maximising the effectiveness of the exposure notifications and providing optimal healthcare assistance to users.

The optimisations enabled by the epidemiological and operational information may be most effective if carried out at the local level. The various Italian regions differ in healthcare policies, resources, and capabilities. Moreover, the epidemic might be at different stages in different locations. Therefore, when sending these data to the server, the app attaches the user’s province of domicile as provided by the user during the onboarding process.

### Epidemiological information

The only epidemiological data Immuni collects relate to the user’s exposure to infected users. The data include:

- The day the exposure occurred
- The duration of the exposure
- Signal attenuation information used for estimating the distance between the two users’ devices during the exposure

The app may send epidemiological information to the server only upon uploading temporary exposure keys. When a healthcare operator communicates to the user their positivity to a SARS-CoV-2 test, any available epidemiological information from the previous 14 days will be uploaded too. The data upload needs to be initiated by the user and approved by the healthcare operator.

To protect the user’s privacy, the data uploaded relating to their exposure to potentially contagious users have certain limitations. For example, the duration of the exposure is measured in five-minute increments and capped at 30 minutes. Moreover, Immuni has no way to determine that exposures occurring on different days may have involved the same infected user. The app also performs periodic dummy uploads to mitigate the risk of someone gaining sensitive information about the user through traffic analysis.

Collecting these data helps the National Healthcare Service to optimise the app’s risk model. By learning how the epidemiological information (e.g., the duration of the exposure) correlates with the user ultimately testing positive for SARS-CoV-2, it may be possible to improve the app's risk model and, therefore, increase its accuracy. This would, in turn, increase the effectiveness of the exposure notification system. Note that the assessment of risk always takes place on the user's device, while the latest model can be fetched from the server.

### Operational information

In addition to the above, some data on device activity and exposure notifications may be collected and uploaded. These data include:

- Whether the device runs iOS or Android
- Whether permission to leverage the Apple, Google and HMS Core Exposure Notification framework is granted
- Whether the device’s Bluetooth is enabled
- Whether permission to send local notifications is granted
- Whether the user was notified of a risky exposure after the last exposure detection (i.e., after the app has downloaded new temporary exposure keys from the server and detected if the user has been exposed to SARS-CoV-2-positive users)
- The date on which the last risky exposure took place, if any

The upload may take place after an exposure detection has been completed. The operational information is uploaded automatically.

To protect user privacy, the data are uploaded without requiring the user to authenticate in any way (e.g., no phone number or email verification). Moreover, traffic analysis is obstructed by dummy uploads.

Thanks to these data, it is possible to estimate the level of the app’s adoption across the country, not just measured by number of downloads—a largely meaningless metric—but by devices that are actually working properly. This information is very helpful, as the utility of Immuni depends heavily on its uptake within the population. Supported by these data, the National Healthcare Service can make better decisions in a number of areas critical to maximising the effectiveness of exposure notifications and providing optimal patient care. Such areas include product development, engineering, and communications.

The data would also help with optimising the allocation of resources. By notifying at-risk users, the app will increase the volume of interactions with the National Healthcare Service. Therefore, estimating the number of users that the system will notify may help the National Healthcare Service to allocate its resources accordingly and, thus, efficiently. Knowing the date on which the last risky exposure took place helps with estimating when symptoms may appear. Such factors enable the National Healthcare Service to adjust its response even more accurately.

## Privacy

Immuni has been and continues to be designed and developed while paying a lot of attention to user privacy. It is a fundamental right that we must do everything we can to protect. We also think that outstanding privacy protection is critical to making the app acceptable to the greatest number of people, thereby maximising Immuni’s utility.

Below, we provide a list of some of the measures by which Immuni protects the user’s privacy:

- The app does not collect any personal data that would disclose the user’s identity. For example, it does not collect the user’s name, date of birth, address, email, or phone number.
- The app does not collect any geolocation data, including GPS data. The user’s movements are not tracked in any shape or form.
- The rolling proximity identifier that is broadcast by the app is generated from random temporary exposure keys and does not contain any information about the device, let alone the user. Moreover, it changes multiple times each hour.
- The epidemiological information uploaded about the user’s exposure to potentially contagious users has certain limitations. For example, the duration of the exposure is measured in five-minute increments and capped at 30 minutes. Moreover, Immuni has no way to determine that exposures occurring on different days may have involved the same infected user.
- The operational information is uploaded without requiring the user to authenticate in any way (including verifying a phone number or email).
- The app performs periodic dummy uploads to mitigate the risk of someone gaining sensitive information about the user through traffic analysis.
- The data stored on the device are encrypted.
- All connections between the device and the server are encrypted.
- All data, whether stored on the device or on the server, are deleted when no longer needed, and in any case no later than December 31, 2020.
- The Ministry of Health will be the data controller. The data will be used solely with the aim of containing the COVID-19 epidemic or for scientific research.
- The data will be stored on servers located in Italy and managed by publicly controlled bodies.