# Privacy-Preserving Analytics

## Table of contents

- [Introduction](#introduction)
- [Epidemiological Info](#epidemiological-info)
	- [Uploaded data](#uploaded-data)
	- [Upload triggers](#upload-triggers)
	- [Uses for the data](#uses-for-the-data)
	- [Privacy and security](#privacy-and-security)
- [Operational Info](#operational-info)
	- [Uploaded data](#uploaded-data-1)
	- [Upload triggers](#upload-triggers-1)
		- [Operational Info with Exposure](#operational-info-with-exposure)
		- [Operational Info without Exposure](#operational-info-without-exposure)
	- [Uses for the data](#uses-for-the-data-1)
	- [Data integrity](#data-integrity)
		- [Apple’s DeviceCheck](#apple-s-devicecheck)
		- [Data integrity solution](#data-integrity-solution)
		- [Preventing race conditions](#preventing-race-conditions)
	- [Privacy and security](#privacy-and-security-1)
- [Configurability](#configurability)

## Introduction

This document focuses on Immuni’s analytics data. It details the data that Immuni collects, and when and why it does so. It also details the measures in place to protect user privacy and to preserve the integrity of the data. It assumes that you have already read [High-Level Description](/README.md) and [Technology](/Technology.md). Certain terms—written with a capital letter at their beginning—are defined in Technology's glossary.

Immuni collects two types of analytics data:
- Epidemiological Info
- Operational Info

When sending these data to the relevant Backend Service, Immuni includes the user’s Province of Domicile.

## Epidemiological Info

### Uploaded data

Epidemiological Info is any set of Exposure Detection Summaries and Exposure Info. For convenience, we provide below the definitions of Exposure Detection Summaries and Exposure Info as found in [Technology](/Technology.md).

**Exposure Detection Summary.** A summary of aggregate information related to the Exposures of a Mobile Client for a set of TEKs. When the A/G Framework on the Mobile Client is provided with a set of TEKs, it can check whether they Match any of the locally stored RPIs. After completing this check, the Mobile Client generates an Exposure Detection Summary. Calculated for the given set of TEKs and their respective Exposures, an Exposure Detection Summary includes the following:
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

### Upload triggers

When a user tests positive for SARS-CoV-2, they have the opportunity to upload their TEKs, so that at-risk users may be notified. The operation takes place with the support of an authorised Healthcare Operator. The user provides an OTP located in the App to the Healthcare Operator, who then validates and authorises the code.

In the context of this data upload, the user will also share:
- Their Province of Domicile
- Any Epidemiological Info from the previous 14 days

The Province of Domicile and the Epidemiological Info are uploaded to the Exposure Ingestion Service, which then provides them to the Analytics Service.

### Uses for the data
Collecting these data helps the National Healthcare Service to optimise the App’s risk model. By learning how the Epidemiological Info (e.g., the duration of the Exposure) correlates with the user ultimately testing positive for SARS-CoV-2, it may be possible to improve the App's risk model and, therefore, increase its accuracy. This would, in turn, increase the effectiveness of the Exposure notification system. Note that the assessment of risk always takes place on the user's device, while the latest risk model can be fetched from the server.

The user’s Province of Domicile is a valuable piece of information in the context of optimising Immuni’s risk model. It makes it possible to study the extent to which the spread of the virus in a certain geographical area impacts the probability of a user testing positive for SARS-CoV-2, every other factor being equal.

### Privacy and security

Some privacy and security considerations concerning the collection of Epidemiological Info follow:
- The Exposure Detection Summaries and the Exposure Info do not contain the TEKs for which they were computed or the RPIs that Matched the TEKs.
- The date of the Exposure is expressed with the granularity of the day, with no reference to the time of day.
- The duration of each Exposure is measured in five-minute increments and capped at 30 minutes. As such, Immuni cannot infer whether, on the day of the Exposure, the user’s Mobile Client was in close proximity with the Mobile Client of a potentially contagious user for 30 minutes or for several hours.
- Immuni has no way to determine that Exposures occurring on different days may have involved the same Mobile Client.
- When the App attempts to access the Exposure Info, the operating system alerts the user that this is happening. The App cannot bypass this measure.
- It is difficult for an attacker to upload fake Epidemiological Info at scale. To do so, they would need to access OTPs that have been authorised by Healthcare Operators. Moreover, the OTPs expire 2 minutes and 30 seconds after they are authorised. Finally, for each OTP, only a limited amount of Epidemiological Info can be uploaded.
- The risk that an attacker may infer any sensitive information about the user by analysing the traffic related to the validation of the OTP or the upload of Epidemiological Info is mitigated by a number of measures. Please refer to [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md).

## Operational Info

### Uploaded data

The Operational Info includes:
- Whether the device runs iOS or Android
- Whether permission to leverage the A/G Framework is granted
- Whether the device’s Bluetooth is enabled
- Whether permission to send local notifications is granted
- Whether the user was notified of a Risky Exposure after the last Exposure Detection
- The date on which the last Risky Exposure took place, if any

Let us define these data as Operational Info _with Exposure_ when the user was notified of a Risky Exposure after the last Exposure Detection, and as Operational Info _without Exposure_ when they were not.

### Upload triggers

The Mobile Client may attempt to upload Operational Info (together with the user’s Province of Domicile) right after each Exposure Detection.

However, due to the strategy we devised to ensure the integrity of analytics data (described in detail below), the uploads of Operational Info are rate-limited. In particular, a Mobile Client will not attempt to send Operational Info with Exposure and Operational Info without Exposure more than once each in a calendar month. Moreover, sampling is performed to decrease the level of traffic the infrastructure needs to support, while still retaining the informative nature of the analytics.

Uploads are directed to the Analytics Service.

#### Operational Info with Exposure

After an Exposure Detection completes and identifies a Risky Exposure, the Mobile Client may attempt the upload of Operational Info with Exposure to the Analytics Service.

The Mobile Client starts by carrying out a _sampling test._ This involves the generation of a uniformly distributed random number between 0 and 1. If the value is less than a parameter _p<sub>OWE</sub>,_ then the Mobile Client attempts the upload. Otherwise, it does not. In any case, the Mobile Client refrains from performing another sampling test and, therefore, attempting another upload of Operational Info with Exposure until the next calendar month.

In practice, we expect to do minimal or no sampling of Operational Info with Exposure, as the information regarding how many users are being notified is important for the management of the epidemic. This means that _p<sub>OWE</sub>_ should be as close to 1 as possible.

With the approach described above, no more than one upload of Operational Info may occur during the same calendar month. This means that the Analytics Service would not know if a user had been notified of multiple Risky Exposures within the same calendar month. This is not a material problem. Once a user is notified, they should contact their general practitioner. From that point forward, based on the current procedures, the response by the National Healthcare Service will be largely independent of whether or not the user is notified of a Risky Exposure again in the short term. Therefore, collecting additional Operational Info with Exposure from that user’s Mobile Client, while certainly useful, is not critical. We believe it wise to sacrifice the small potential benefit of an additional data upload in favour of better data integrity and privacy protection.

#### Operational Info without Exposure
After an Exposure Detection completes without a Risky Exposure being detected, the Mobile Client may attempt the upload of Operational Info without Exposure to the Analytics Service.

In all likelihood, an upload of Operational Info without Exposure will occur more frequently than an upload of Operational Info with Exposure. This is because the Mobile Client performs an Exposure Detection several times per day, and we expect only a fraction of them to reveal a Risky Exposure. Hence, the upload of Operational Info without Exposure needs to be sampled more aggressively.

The rate limit imposed on analytics data applies to the calendar month. This makes it necessary to schedule uploads in a way that prevents the Mobile Clients from generating a spike in analytics uploads on the first day of each calendar month.

To solve this issue, the first time the App activates during the current calendar month, it generates a random number _dT<sub>OWOE</sub>_ uniformly distributed between 0 and (_n<sub>days</sub>_-1) * 86400, where _n<sub>days</sub>_ represents the number of days in the current calendar month and 86400 is the number of seconds in a day. Let us call _T<sub>OWOE</sub>_ the timestamp (measured in seconds) corresponding to 00:00:00 UTC of the first day of the current month. The value of _dT<sub>OWOE</sub>_ represents after how many seconds from _T<sub>OWOE</sub>_ the window of opportunity will open for the App to send its monthly Operational Info without Exposure. The upload will happen no later than 24 hours after _T<sub>OWOE</sub>_+_dT<sub>OWOE</sub>._

We must consider that the user may install the App in the middle of a calendar month. In such a case, _dT<sub>OWOE</sub>_ may represent a point in time in the past (i.e., earlier in the calendar month). This is by design, as it ensures that the rate of sampling remains constant, no matter the distribution of installs within the month.

After performing the Exposure Detection, the Mobile Client verifies if the current time is between _T<sub>OWOE</sub>_+_dT<sub>OWOE</sub>_ and _T<sub>OWOE</sub>_+_dT<sub>OWOE</sub>_+86400. If so, the Mobile Client performs a sampling test similar to the one described for Operational Info with Exposure. It generates a uniformly distributed random number between 0 and 1. If the value is less than a parameter _p<sub>OWOE</sub>,_ the Mobile Client attempts to perform the upload. Note that, if a sampling test for Operational Info without Exposure has previously been performed in the calendar month, the Mobile Client refrains from performing it again. It does not attempt to upload Operational Info without Exposure more than once in the same calendar month. If, after performing an Exposure Detection, the current time is beyond _T<sub>OWOE</sub>_+_dT<sub>OWOE</sub>_+86400, then the Mobile Client does nothing. Instead, it waits for the next calendar month before again attempting an upload of Operational Info without Exposure.

It follows that a Mobile Client will not perform more than one upload of Operational Info without Exposure in a calendar month. Although frequent uploads would improve the accuracy of the estimates of adoption and correct usage of the App by the population, we believe that the added accuracy would not justify the detriment to user privacy.

### Uses for the data
Thanks to these data, it is possible to estimate the level of the App’s adoption across the country, not just measured by number of downloads—a largely meaningless metric—but by Mobile Clients that are actually working properly. This information is very helpful, as the utility of Immuni depends heavily on its uptake within the population. Supported by these data, the National Healthcare Service can make better decisions in a number of areas critical to maximising the effectiveness of Exposure notifications and providing optimal patient care. Such areas include product development, engineering, and communications.

The data would also help with optimising the allocation of resources. By notifying at-risk users, the App will increase the volume of interactions with the National Healthcare Service. Therefore, estimating the number of users that the system will notify may help the National Healthcare Service to allocate its resources accordingly and, thus, efficiently. Knowing the date on which the last Risky Exposure took place helps with estimating when symptoms may appear. Such factors enable the National Healthcare Service to adjust its response even more accurately. Being able to break down the data geographically is necessary to ensure that the response is commensurate with the local need.

### Data integrity

To protect user privacy, the data are uploaded without requiring the user to authenticate in any way (e.g., no phone number or email verification). This exposes the Analytics Service to attacks intended to pollute the collected data. This is a major problem, because if the data cannot be trusted to be free of substantial inaccuracies, no reliable insight can be extracted from them. It follows that, if we cannot ensure the integrity of the Operational Info collected by the Analytics Service, the data should not be collected in the first place.

Below, we describe our solution for iOS Mobile Clients. Gathering reliable data from iOS Mobile Clients is sufficient to provide estimates for the population as a whole, as long as the penetration of iOS and Android devices in different Italian provinces is known. A solution that will support certain Android Mobile Clients is being developed. We will share it as soon as possible.

Before we describe our solution for the iOS Mobile Client in detail, we must describe Apple’s DeviceCheck, a technology that is a key component of our method.

#### Apple’s DeviceCheck
Apple’s [DeviceCheck](https://developer.apple.com/documentation/devicecheck) is composed of an iOS API and a server API.

**DeviceCheck iOS API.** The DeviceCheck iOS API of a genuine iOS device can be locally queried by an app to obtain a _DeviceCheck token_ from the operating system. A new DeviceCheck token is generated and returned by the operating system at every query. The DeviceCheck token contains encrypted information about the specific device, the app’s developer, and the time of generation. Only Apple can decrypt such information, which is, therefore, inaccessible to Immuni.

**DeviceCheck server API.** The DeviceCheck server API can be queried with a given DeviceCheck token. Doing so permits the following to take place:

* Checking the validity of the DeviceCheck token (i.e., for the server to make sure the DeviceCheck token was generated by a genuine iOS device). DeviceCheck tokens are ephemeral, meaning that they expire and that any attempt to validate them with the DeviceCheck server API after their expiration will fail.

* Getting or setting a small amount of data (2 bits), if the DeviceCheck token is valid. These data will persist between calls to the DeviceCheck server API when using DeviceCheck tokens generated by the same iOS device for apps by the same developer. When retrieving the bits, the API returns also the month and year of their last modification.


The naming convention used by Apple for the data that can be bound to a specific iOS device for a specific developer by using the DeviceCheck server API is the following:
* **bit0.** The value of the first bit (boolean). This bit is readable and writable if a valid DeviceCheck token is available.

* **bit1.** The value of the second bit (boolean). This bit is readable and writable if a valid DeviceCheck token is available.

* **last\_update\_time.** The month of the last modification of any of the 2 bits, in _YYYY-MM_ UTC format (string). This is read-only and automatically updated whenever any of the 2 bits is set. It is _null_ if the bits were never set.

_bit0_ and _bit1_ are also called the _per-device bits._

Please note that, since the DeviceCheck token changes every time it is requested from the DeviceCheck iOS API, it cannot be used to uniquely identify the iOS device.

#### Data integrity solution
Our solution to mitigate the risk of an attacker successfully uploading fake Operational Info at scale is based on the use of _analytics tokens_. An analytics token is a random identifier generated by an iOS Mobile Client to authenticate the requests to upload Operational Info. Analytics tokens are valid for at most two calendar months, including the calendar month during which they are authorised (more on this below). For better privacy protection, the analytics token does not contain any information on the iOS Mobile Client and changes once each calendar month.

The solution works as follows:
- **A genuine iOS device is needed to authorise an analytics token.** This is accomplished by requiring that a request to the Analytics Service to authorise an analytics token includes a valid DeviceCheck token. Therefore, the only way for an attacker to authorise an analytics token is to control a genuine iOS device. The number of possible analytics tokens is high enough to make brute-force attacks aimed at hitting upon a valid analytics token practically impossible.
- **Operational Info can only be uploaded with an authorised analytics token.** If uploading Operational Info did not require any authentication, an attacker could easily pollute the analytics data at scale. Therefore, the Analytics Service only accepts the upload of Operational Info with a valid analytics token. This approach to authenticating requests is particularly privacy-friendly, as it does not require the user to provide and verify information such as a phone number or an email address.
- **The number of analytics tokens that can be authorised with a genuine iOS device is rate-limited.** It is possible for an attacker to authorise an analytics token. Therefore, we must ensure that an attacker cannot authorise an arbitrarily large number of them. This is accomplished by using the DeviceCheck server per-device bits and _last\_update\_time_ to rate-limit the creation of analytics tokens to—at most—one per calendar month for each genuine iOS device that the attacker controls. Because _last\_update\_time_ is expressed as _YYYY-MM,_ the rate-limiting is synchronised with the passing of calendar months.
- **The amount of uploads of Operational Info that can be performed with an analytics token is limited.** It is possible for an attacker to authorise one analytics token per calendar month for each genuine iOS device that they control. Therefore, without a cap on the number of uploads of Operational Info that an attacker could perform with one such token, they would be able to pollute the analytics data at scale. Instead, the Analytics Service does not accept more than four uploads of Operational Info (two with Exposure and two without Exposure) for any given token.


This solution helps to make uploading fake Operational Info at scale more impractical and expensive. Below, we describe it in greater detail.

The iOS App generates its first analytics token when it is launched for the first time. At the same time, it generates a random number _dT<sub>AT</sub>_ uniformly distributed between 0 and (_n<sub>days</sub>_-1) * 86400, where _n<sub>days</sub>_ represents the number of days in the next calendar month and 86400 is the number of seconds in a day. Let us call _T<sub>AT</sub>_ the timestamp (measured in seconds) corresponding to 00:00:00 UTC of the first day of the next calendar month. The value of _dT<sub>AT</sub>_ represents after how many seconds from _T<sub>AT</sub>_ the iOS App will begin to be allowed to pick a new analytics token. The iOS App computes _T<sub>AT</sub>_ + _dT<sub>AT</sub>_ and stores this value as _T<sub>ATN</sub>._ The first time that the iOS App performs an Exposure Detection after _T<sub>ATN</sub>,_ it generates a new analytics token and sets a new value for _T<sub>ATN</sub>._

Figure 1 shows a plausible timeline for the generation and subsequent modifications of the analytics token throughout four months:
1. The iOS App is launched in June for the first time. This being the first launch, it generates its first analytics token and sets a value for _T<sub>ATN</sub>,_ which must be at a point in July.
2. The first time the iOS App performs an Exposure Detection after _T<sub>ATN</sub>,_ it generates a new analytics token and sets a new value for _T<sub>ATN</sub>,_ which must be at a point in the following month, August.
3. In August, a corner case occurs: no Exposure Detections take place between _T<sub>ATN</sub>_ and the end of the month.
4. The first Exposure Detection after _T<sub>ATN</sub>_ occurs in the new month (September). Again, the iOS App generates a new analytics token and sets a new value for _T<sub>ATN</sub>._

![Figure 1](/Figures/Privacy-Preserving%20Analytics%20-%20Figure%201.png)
<p align="center"><b>Figure 1.</b> Analytics tokens generation timeline.</p>



Whenever the iOS Mobile Client generates a new analytics token, the iOS Mobile Client authorises it with the server before using it.

The iOS Mobile Client includes a DeviceCheck token in each analytics token authorisation request. The Analytics Service then proceeds as follows:
1. It checks if the received analytics token has already been authorised. 
2. It sends a response to the iOS Mobile Client that generated the request, indicating whether the provided analytics token has already been authorised. If it has already been authorised, no further steps are required. Otherwise, it will subsequently attempt authorisation.
3. It checks if the DeviceCheck token comes from a genuine iOS device by contacting the DeviceCheck server API. If not, it ignores the request to authorise the analytics token.
4. It checks if the current calendar month is after the _last\_update\_time_ reported by the DeviceCheck server API, or if _last\_update\_time_ is _null._ If not, it ignores the request to authorise the analytics token.
5. It contacts the DeviceCheck server API and arbitrarily sets both per-device bits to 0 to ensure that the DeviceCheck server updates _last\_update\_time_ to the current calendar month.
6. It stores the received—and now authorised—analytics token and the current date and time.

The authorisation of an analytics token by the Analytics Service is performed asynchronously. This means that the iOS Mobile Client has to send at least two requests with the same analytics token to the Analytics Service—one to request the authorisation of the analytics token and another to verify that it has been authorised.
The iOS Mobile Client sends one—and only one—analytics token authorisation request to the Analytics Service each time it performs an Exposure Detection. If the operation is unsuccessful (including if it times out), the iOS Mobile Client sends the same request with the same analytics token to the Analytics Service at the next opportunity. The iOS Mobile Client stops sending an analytics token to the Analytics Service only if one of two conditions is verified:
- It has received confirmation from the Analytics Service that the analytics token has been authorised
- It has generated a new analytics token

After a full calendar month has passed from the initial authorisation of an analytics token, the Analytics Service stops storing it. At this point, the token is considered no longer valid. Analytics tokens authorised in January stop being valid at the end February. Those authorised in February stop being valid at the end of March, and so forth.

Every time that the iOS Mobile Client attempts to upload Operational Info, it does so while including the last authorised analytics token in its request. The iOS Mobile Client refrains from attempting to send Operational Info if it does not have a valid analytics token. The Analytics Service proceeds as follows:
1. It checks that the analytics token is valid. It discards any request not containing a valid analytics token.
2. It checks how many instances of Operational Info with Exposure and Operational Info without Exposure have been successfully uploaded with that particular analytics token. For each analytics token, it does not accept more than two uploads of Operational Info with Exposure and two of Operational Info without Exposure. This caps the number of uploads an attacker can perform for each genuine iOS device they control per calendar month, as they can only authorise one analytics token for each genuine iOS device per calendar month. Based on the logic explained above, four uploads of Operational Info is the maximum an iOS Mobile Client that has not been tampered with will attempt with one analytics token.
3. It stores any Operational Info not discarded due to the above limitations.

Figure 2 shows how the iOS Mobile Client, the Analytics Service, and the DeviceCheck server interact. The communication between the Analytics Service and the DeviceCheck server is more complex than how it has been represented so far. It is described in more detail in [Preventing race conditions](#preventing-race-conditions).

![Figure 2](/Figures/Privacy-Preserving%20Analytics%20-%20Figure%202.png)
<p align="center"><b>Figure 2.</b> The device successfully authorises an analytics token. Later, it uses the token to upload Operational Info with Exposure. It then generates and successfully authorises a new analytics token the next calendar month.</p>



As explained, the iOS Mobile Client leverages DeviceCheck to authorise analytics tokens. We could also have the iOS Mobile Client do likewise for other requests, such as those for uploads of Operational Info. However, we have elected to not do this, for the following reasons:

- The integrity of the analytics data would not be substantially improved.
- We would create a correlation between requests to the DeviceCheck server API and the upload of Operational Info with Exposure. This may allow an entity with access to the DeviceCheck server to infer some information about certain users potentially having been infected.

#### Preventing race conditions
During the process of authorising an analytics token, the Analytics Service reads and writes the per-device bits and reads _last\_update\_time._ These operations are not atomic. Therefore, multiple authorisations of analytics tokens may take place concurrently. It follows that an attacker could exploit the time between a read and a write to send many requests to the Analytics Service using the same genuine iOS device.

We devised a mitigation strategy to address this issue. It leverages DeviceCheck’s per-device bits to allow the Analytics Service’s workers to spot concurrent operations.

When one of the Analytics Service’s workers processes an analytics token authorisation request, it performs the following steps:
1. **First read.** It reads the DeviceCheck per-device bits, expecting to find them both set to 0.
2. **Second read.** If the condition is met, the worker pauses for a random amount of time (_read\_time_), then reads the per-device bits again.
3. **First write.** If the condition continues to be met, the worker changes _bit0_ to 1.
4. **Third read.** Then, the worker pauses for another period of time (_check\_time_). It reads the per-device bits one last time.
5. **Authorisation and second write.** If the _bit0_ is set to 1 and _bit1_ to 0, the worker authorises the analytics token and sets _bit0_ back to 0.

If during either the first or the second read the per-device bits are not both set to 0, or if during the third and last read they are set to values other then _bit0_ to 1 and _bit1_ to 0, the worker will know that an attacker sent more than one request in a short time window including DeviceCheck tokens generated with the same iOS device (and possibly the same DeviceCheck token).

The worker, therefore, rejects the Operational Info and records the abusive behaviour by setting both per-device bits to 1, a configuration reserved for marking malicious iOS devices. From that moment on, if a worker reads the per-device bits and finds both set to 1, it will know that the iOS device that was used to generate the DeviceCheck token included in the request is controlled by an attacker, and it will not authorise the analytics token.

In table 1, we present a brief recap of the meaning that the Analytics Service assigns to the different combinations of DeviceCheck per-device bits.

| _bit1_ | _bit0_ | Integer representation | Meaning |
| :--- | :--- | :--------------------- | :---------- |
| 0    | 0    | 0                      |  Default value |
| 0    | 1    | 1                      |  The worker is waiting for the third read before it can authorise the last analytics token it received from the device |
| 1    | 0    | 2                      |  Reserved for future use |
| 1    | 1    | 3                      |  The device is blacklisted |

<p align="center"><b>Table 1.</b> The meaning of the various configurations of the DeviceCheck per-device bits.</p>



Figure 3 shows what happens when an attacker sends two analytics token authorisation requests directly after one another in an attempt to exploit the race condition. _Worker 1,_ the worker that processes the first request, reads the DeviceCheck per-device bits, then waits a while before reading them again. The per-device bits do not change, so it writes _bit0_ to 1 and _bit1_ to 0 and then waits. In the meantime, another worker, _Worker 2,_ processes the second request, but by the time the worker reads the per-device bits again, they have changed. Worker 2 knows that another analytics token authorisation request has been made using a DeviceCheck token generated by the same iOS device, so it discards the request and sets both per-device bits to 1 to blacklist the iOS device. When Worker 1 reads the per-device bits for the last time, it detects that they are both set to 1, from which it infers that an attacker has made multiple requests in a short span of time. It discards the request.

![Figure 3](/Figures/Privacy-Preserving%20Analytics%20-%20Figure%203.png)
<p align="center"><b>Figure 3.</b> Sequence diagram representing the race condition attack mitigation in action.</p>



_read\_time_ should be drawn from a uniformly distributed random variable with [0, _max\_read\_time_] support.

To choose _check\_time_ so that it is as short as possible while reliably preventing any attack of this kind, let us analyse the worst-case scenario. Worker 1 reads the per-device bits in their original configuration right before Worker 2 changes them to a different value. This means that, for Worker 2 to determine that this is an attack and thus avoid authorising the analytics token, it will have to wait at least until Worker 1 has read the per-device bits again and set them both to 1. This takes place after _read\_time._ Therefore, _check\_time_ should be at least _max\_read\_time_+2\*_device\_check\_server\_timeout,_ where _device\_check\_server\_timeout_ is the maximum time a worker will wait before timing out if a response from Apple’s DeviceCheck server is not received.

The described mitigation against attackers attempting to exploit race conditions exposes the system to a replay attack. An attacker could intercept a legitimate analytics token authorisation request and replay it very quickly, leading to the blacklisting of the legitimate iOS Mobile Client. Since a blacklisting does not really entail any loss of functionality for the user, and given that it is challenging to stage such an attack on a large scale, we think that it is pragmatic to accept this inherent vulnerability. Moreover, pinning the certificate authority on the iOS Mobile Client’s side makes a man-in-the-middle attack—a prerequisite for a replay attack—much harder to carry out.

### Privacy and security

Some privacy and security considerations on the collection of Operational Info follow:
- The integrity of the Operational Info stored by the Analytics Service is protected with the methodology described in [Data integrity solution](#data-integrity-solution) above, which is based on Apple’s DeviceCheck and on analytics tokens.
- Immuni cannot extract any information on the user’s iOS device from the DeviceCheck tokens. Moreover, the DeviceCheck token changes every time it is requested from the DeviceCheck iOS API. Therefore, it cannot be used to uniquely identify the device.
- The DeviceCheck server only receives requests from Immuni when a new iOS App is launched for the first time and then, for each iOS Mobile Client, once per month on random days. These requests carry with them no sensitive information whatsoever, either directly or indirectly. It follows that an entity with access to the DeviceCheck server cannot infer any sensitive information about the user solely based on these requests.
- The analytics token is generated randomly and contains no information on the device. Moreover, it is replaced after no more than two months. The Analytics Service cannot associate the uploads of Operational Info from multiple analytics tokens to the same iOS Mobile Client.
- Operational Info includes the bare minimum amount of information that the National Healthcare Service needs to monitor Immuni’s working condition and allocate resources effectively. Only one upload of Operational Info with Exposure and one of Operational Info without Exposure are performed by a Mobile Client per calendar month, at most.
- The risk of an attacker exploiting race conditions to circumvent the rate-limits we impose on the authorisation of analytics tokens is mitigated with the methodology described in [Preventing race conditions](#preventing-race-conditions) above.
- The risk of an attacker inferring any sensitive information about the user by analysing the traffic related to the upload of Operational Info is mitigated by a number of measures. Please refer to [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md).

## Configurability

All of the parameters that allow for the tuning of the Operational Info upload can be changed from the server side. This is achieved by modifying the Configuration Settings served by the App Configuration Service.
- ***p<sub>OWE</sub>.*** The sampling rate for Operational Info with Exposure.
- ***p<sub>OWOE</sub>.*** The sampling rate for Operational Info without Exposure.