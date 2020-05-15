# Immuni's High-Level Description

## Table of contents

- [Executive summary](#executive-summary)
- [Context](#context)
- [Vision and goals](#vision-and-goals)
- [Principles](#principles)
- [Product](#product)
  - [How it works](#how-it-works)
- [Analytics](#analytics)
  - [Epidemiological information](#epidemiological-information)
  - [Technical information](#technical-information)
- [Privacy](#privacy)
- [Open points](#open-points)

## Executive summary

- Immuni is a technological solution that centres on **an iOS and Android smartphone app.** It helps us to fight the COVID-19 pandemic by notifying users at risk of carrying the virus as early as possible—**even when they are asymptomatic.** These users can then isolate themselves to avoid infecting others, and seek medical advice.
- Immuni’s design and development are based on five main principles: **utility, accuracy, scalability, transparency,** and **privacy.**
- It features a **contact tracing** system based on **Bluetooth Low Energy:**
  - When two users come sufficiently close to each other for long enough, their devices record each other’s _rolling proximity identifier_ in their local memory. These identifiers are generated from _temporary exposure keys_ and change multiple times each hour. These keys are **generated randomly** and change once per day.
  - When a user tests positive for SARS-CoV-2, the virus causing COVID-19, they have the option to upload to a server their recent temporary exposure keys. This operation can only happen with the validation of a **healthcare operator.**
  - The app periodically downloads the new temporary exposure keys and uses them to derive the infected users’ rolling proximity identifiers for the recent past. It then matches them against those stored in the device’s memory and **notifies the user** if a risky contact has occurred.
  - The system uses **no geolocation data** whatsoever, including GPS data. So, the app cannot tell where the contact with a potentially contagious user took place, nor the identities of those involved.
- To implement its contact tracing functionality, Immuni leverages **the Apple and Google Exposure Notification framework.** This allows Immuni to be more resilient than otherwise would be possible.
- Besides the temporary exposure keys, the Immuni app also sends to the server some analytics data. These include **epidemiological and technical information,** and are sent for the purpose of helping the National Healthcare Service (Servizio Sanitario Nazionale) to provide effective assistance to users, in compliance with art. 6.2.b and 6.3 of the Law Decree 28/2020.
- Immuni is being developed while paying a lot of attention to user privacy and a number of measures have been taken to protect it. For example, the app collects **no personal data that would disclose the user’s identity,** such as the user’s name, age, address, email, or phone number.

## Context

The whole world is united by a determination to stop the spread of COVID-19, the disease caused by SARS-CoV-2. The pandemic is threatening people’s health and severely damaging economies on a global scale.

Many experts agree that, in the future, new pandemics are a distinct possibility. Some may become even more dangerous to humanity than the one we are currently battling.

In this challenging context, the contribution of technological innovation can be decisive. Immuni is one of a number of tools deployed and initiatives taken by the Italian government to help slow down the spread of the disease and accelerate the return to everyday life.

This document provides a high-level description of Immuni—it is a good idea to read it first. More detailed information can be found in the following documents (if you can not find them, check back in a few days—they are coming soon):

- Product Description
- [Technology Description](https://github.com/immuni-app/documentation/blob/master/Technology%20Description.md)
- Security Description

Additionally, we are going to open-source Immuni’s software under the [GNU Affero General Public License version 3](https://www.gnu.org/licenses/agpl-3.0.en.html). Finally, penetration tests are due to take place and we will share the resulting reports.

## Vision and goals

Immuni is a technological solution that centres on a smartphone app.

It helps us fight epidemics—starting with COVID-19:

1. The app aims to notify users at risk of carrying the virus as early as possible—even when they are asymptomatic.
2. These users can then isolate themselves to avoid infecting others. This minimises the spread of the virus while speeding up a return to normal life for most people.
3. By being notified early, they can also seek medical advice and lower the risk of serious health consequences.

Immuni is designed to address the current crisis, but the vision behind it is for the tools that are being developed to make us all better prepared in addressing similar threats that may arise in the future.

## Principles

The main principles that guide the design and development of Immuni follow:

- **Utility.** The app needs to be useful in fulfilling the vision and goals for the project as outlined above. The key here is to be able to notify as high a percentage of the people who are substantially at risk as possible, and to do so as early as is practical. This is the most important principle.
- **Accuracy.** Immuni aims to notify only those users who have a substantial risk of being positive to the virus. This is important because of the psychological toll that goes with being notified about a potential transmission of the disease, and because too high a rate of false positives would result in users losing trust in the app and stopping their use of it. Also, the more accurate the app is, the more efficiently the National Healthcare Service (_Servizio Sanitario Nazionale_) will be able to take care of users, making sure to attend to higher-risk ones first.
- **Scalability.** Immuni needs to be widely adopted throughout the country. This requires the system to scale well technologically and the operational burden it places on the National Healthcare Service to be manageable.
- **Transparency.** Everyone should be provided with access to documentation describing Immuni in all its parts and the rationale behind the most important design decisions. Also, all the relevant software will be open source. This allows the users to verify that the app works as documented and the expert community to help improve the system.
- **Privacy.** While keeping Immuni useful, user privacy must be protected as well as possible. Earning and maintaining user trust is critical to make sure the app can be widely adopted.

## Product

Immuni is a technological solution that centres on a smartphone app available for Android and iOS.

It features a contact tracing system to help notify potentially SARS-CoV-2 positive users at an early stage. This system keeps track of contact between Immuni users, even when they are total strangers. When a user tests positive for SARS-CoV-2, the app uses this system to notify other at-risk users. The system is based on Bluetooth Low Energy and does not use any geolocation data whatsoever, including GPS data. So, while the app knows that the contact with an infected user took place and how long it lasted, and can estimate the distance that separated the two users, it cannot tell where the contact took place, nor the identities of those involved.

The app then proceeds to recommend at-risk users what to do. Recommendations may include self-isolation (which helps minimise the spread of the disease) and contacting a doctor (so that the user can receive the most appropriate care and reduce the likelihood of developing severe complications). The exact recommendations depend on the area in which the user lives, as different policies may apply to different areas. To point the user in the right direction, the app collects their province of domicile during the onboarding process.

As stated, Immuni’s contact tracing is based on Bluetooth Low Energy. This has some advantages, compared to a solution based on location tracking:

- **The system is more accurate.** Unlike geolocalisation, which, in many contexts, has a precision in the order of the tens of metres, Bluetooth Low Energy signals allow for capturing contact occurring within a radius of a few metres of the user. These are the cases of contact relevant to the transmission of SARS-CoV-2.
- **The battery is used more efficiently.** Battery-efficient location-tracking solutions exist. However, Bluetooth Low Energy tends to be outstanding when it comes to energy efficiency. This is important because it is reasonable to expect that the app’s uninstall rate will be correlated with battery consumption.
- **No geolocation data are required.** Thanks to Bluetooth Low Energy, contacts are traced without tracking the users’ location. This may result in the app being more welcomed by people and could facilitate wider adoption, increasing Immuni’s utility.

To implement its contact tracing functionality, Immuni leverages the Apple and Google Exposure Notification framework (see [Apple’s documentation](https://www.apple.com/covid19/contacttracing) and [Google’s documentation](https://www.google.com/covid19/exposurenotifications/)). This allows Immuni to overcome certain technical limitations, thus being more resilient than otherwise would be possible.

### How it works

Below, a high-level, simplified description of the system is provided. For more details, please study the rest of the Immuni documentation.

Once installed and set up on a device (_device_A_), the app generates a _temporary exposure key._ This key is generated randomly and changes daily. The app also starts transmitting a Bluetooth Low Energy signal. The signal contains a _rolling proximity identifier_ (_ID_A1,_ assumed fixed in this example, for simplicity), which is generated from the current temporary exposure key. When another device (_device_B_) running the app receives this signal, it will record _ID_A1_ locally, in its memory. At the same time, _device_A_ will record _device_B_’s identifier (_ID_B1,_ also assumed fixed in this example).

If the user of _device_A_ later tests positive for SARS-CoV-2, following the protocol defined by the National Healthcare Service, they will have the option to upload to the Immuni server the temporary exposure keys from which the Immuni app can derive the rolling proximity identifiers recently broadcast by _device_A_ (including _ID_A1_). Periodically, _device_B_ checks the new keys uploaded to the server—the keys of users who have the virus—against its local list of identifiers. _ID_A1_ will be a match and the app will notify the user of _device_B_ that they may be at risk and provide advice on what to do next, for example, isolating themselves and getting in touch with the National Healthcare Service.

In practice, when it comes to determining whether the user of _device_B_ is at risk, finding that they had been in the proximity of the user of _device_A_ is not enough. Immuni assesses this risk based on the duration of the exposure and the distance between the two devices. This is estimated from the attenuation of the Bluetooth Low Energy signal as received by _device_B._ The longer the exposure and the closer the contact, the higher the risk that a transmission of the virus happened. A contact lasting only a couple of minutes and happening at several metres of distance will generally be considered to be low risk. The risk model may evolve with time as more information about SARS-CoV-2 becomes available.

It should be noted that the estimation of distance is error-prone. In fact, the attenuation of a Bluetooth Low Energy signal depends on factors such as the orientation of the two devices relative to each other and the obstacles (including human bodies) that lie in between. While leveraging this information is likely useful in increasing the accuracy of Immuni’s assessments of the risk of contagion, wrong assessments will happen with some frequency.

The rolling proximity identifier that is broadcast by the app is generated from random temporary exposure keys and does not contain any information about the device, let alone the user. Moreover, it is rolling, meaning that it changes multiple times each hour, further protecting the privacy of Immuni’s users.

To make sure only users who actually tested positive for SARS-CoV-2 upload their keys to the server, the upload procedure can only be performed with the cooperation of an authenticated healthcare operator. The operator asks the user to provide a code generated by the app and inputs it into a back-office tool. The upload can succeed only if the code used by the app to authenticate the data corresponds to that entered in the system by the healthcare operator.

## Analytics

Besides the temporary exposure keys, some additional information is sent to the server and analysed to ensure the proper functioning of the system:

- Epidemiological information
- Technical information

These data are collected and used in compliance with art. 6.2.b and 6.3 of the Law Decree 28/2020. They are essential for the National Healthcare Service to effectively manage the system, including providing optimal healthcare assistance to users.

### Epidemiological information

The only kind of epidemiological data Immuni collects are about the user’s exposure to infected users. The data include:

- The day the exposure occurred
- The duration of the exposure
- Signal attenuation information used for estimating the distance between the two users’ devices during the exposure
- Information on how contagious the infected user was likely to be when the exposure occurred, based on the day of symptom onset, if any (this information is attached to each temporary exposure key the infected user has uploaded with the assistance of a health operator)

There are two moments when the app may send exposure information to the server:

- **Upon assessing transmission risk.** The upload may happen after the app has downloaded new keys from the server and assessed the transmission risk for the user, based on their recent contacts with SARS-CoV-2 positive users. In this case, the ensuing epidemiological data are uploaded automatically.
- **Upon uploading temporary exposure keys.** When a healthcare operator communicates to the user their positivity to a SARS-CoV-2 test, any available epidemiological data from the previous 14 days will be uploaded too. In this case, the data upload needs to be initiated by the user and approved by the healthcare operator.

Collecting these data helps the National Healthcare Service to optimise patient care, thereby minimising the toll of the epidemic on public health. The data help in at least two ways:

- **Optimising the allocation of resources.** By notifying at-risk users, the app will increase the volume of interactions with the National Healthcare Service. Therefore, estimating the number of users the system will notify is fundamental to helping the National Healthcare Service allocate its resources accordingly and thus efficiently. Failure to do so may result in avoidable deaths and users losing trust in the system.
- **Optimising the app’s risk model.** By learning how the epidemiological information (e.g., the duration and distance of the user's exposure) determines whether the user ultimately tests positive or negative for SARS-CoV-2, it is possible to improve the app's risk model and therefore increase its accuracy. Note that the assessment of risk always happens on the user's device, while the latest model can be fetched from the server.

These optimisations are most effective if carried out at the local level. The various Italian regions differ in healthcare policies, resources, and capabilities. Moreover, the epidemic might be at different stages in different locations. Therefore, when sending these epidemiological data to the server, the app attaches the user’s province of domicile as provided by the user during the onboarding process.

To protect the user’s privacy, the data collected about their exposure to potentially contagious users have certain limitations. For example, the duration of the exposure is measured in five-minute increments and capped at 30 minutes for the sum of all contacts with an infected user on any given day. Moreover, Immuni has no way to determine that multiple contacts on different days may have occurred with the same infected user.

### Technical information

In addition to the above, some data on device activity and working condition may automatically be collected and uploaded after assessing transmission risk together with the ensuing epidemiological data.

These data include basic information on the app’s working condition, such as when, after activating in the background and downloading new temporary exposure keys, it successfully assesses the user’s risk of having been infected with SARS-CoV-2 due to exposure to positive users. The data also includes information on whether the device is configured correctly (e.g., if Bluetooth is currently turned on). When uploading these data to the server, the user’s province of domicile is included.

Thanks to these data, it is possible to estimate the level of adoption of the app across the country, not just measured by number of downloads—a largely meaningless metric—but by devices that are actually working properly. This information is fundamental, as we know that the utility of Immuni depends heavily on its uptake within the population. Supported by these data, the National Healthcare Service will be able to make better decisions when it comes to a number of areas critical to making Immuni’s as useful as possible in contrasting the epidemic and providing optimal patient care. Such areas include product development, engineering, and communications.

## Privacy

Immuni has been and continues to be designed and developed while paying a lot of attention to user privacy. It is a fundamental right that we must do everything we can to protect. We also think that outstanding privacy protection is critical to making the app acceptable to the greatest number of people, thereby maximising Immuni’s utility.

Below, we provide a list of some of the measures by which Immuni protects the user’s privacy:

- The app does not collect any personal data that would disclose the user’s identity. For example, it does not collect the user’s name, age, address, email, or phone number.
- The app does not collect any geolocation data, including GPS data. The user’s movements are not tracked in any shape or form.
- The rolling proximity identifier that is broadcast by the app is generated from random temporary exposure keys and does not contain any information about the device, let alone the user. Moreover, it changes multiple times each hour.
- The analytics data collected about the user’s exposure to potentially contagious users have certain limitations. For example, the duration of the exposure is measured in five-minute increments and capped at 30 minutes for the sum of all contacts with an infected user on any given day. Moreover, Immuni has no way to determine that multiple contacts on different days may have occurred with the same infected user.
- The data stored on the device are encrypted.
- All connections between the mobile app and the server are encrypted.
- All data, whether stored on the device or on the server, are deleted when not relevant any longer (generally within a few weeks from collection, and in any case no later than December 31, 2020).
- The Ministry of Health will be the data controller, hence deciding what data to collect and how to use them exactly. In any case, the data will be used solely with the aim of containing the COVID-19 epidemic or, in completely anonymised or aggregated form, for scientific research.
- The data will be stored on servers located in Italy and managed by publicly controlled entities.

## Open points

These are some of the most pressing points on which we are working:

- **Dummy traffic.** We would like to minimise the information that an attacker could gain by analysing network traffic. We are finalising decisions on sampling rules and dummy uploads.
- **Analytics integrity.** We are trying to collect the necessary epidemiological information and technical information while preserving user privacy as much as possible. However, performing such collection without asking the user to authenticate a phone number or email makes it harder to prevent attackers from polluting the data with fake uploads. We have a promising solution in the works that we expect to work for a significant portion of devices.
- **Data retention, retrieval, and erasure.** We are working on finalising the specifics of these important policies.
