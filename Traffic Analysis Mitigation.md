# Immuni's Traffic Analysis Mitigation

## Table of contents

- [Introduction](#introduction)
- [Analytics Service](#analytics-service)
  - [Genuine analytics uploads](#genuine-analytics-uploads)
    - [Operational Info with Exposure](#operational-info-with-exposure)
    - [Operational Info without Exposure](#operational-info-without-exposure)
  - [Dummy analytics uploads](#dummy-analytics-uploads)
- [Exposure Ingestion Service](#exposure-ingestion-service)
    - [Genuine TEK upload sequences](#genuine-tek-upload-sequences)
    - [Dummy TEK upload sequences](#dummy-tek-upload-sequences)
      - [TEK upload sequence simulation](#tek-upload-sequence-simulation)
      - [Traffic pattern modulation](#traffic-pattern-modulation)
        - [iOS](#ios)
        - [Android](#android)
- [Configurability](#configurability)

## Introduction

This document details the measures Immuni adopts to mitigate the risk that an attacker may infer information about a user by analysing the encrypted traffic between the user's Mobile Client and the Backend Services. It assumes that you have already read the [High-Level Description](/README.md), the [Technology Description](/Technology%20Description.md), and Privacy-Preserving Analytics (coming soon). Certain terms—written with a capital letter at their beginning—are defined in the Technology Description's glossary.

Our focus is on protecting the interaction between the App and two Backend Services: the Analytics Service and the Exposure Ingestion Service. Without appropriate countermeasures, interactions with the former could reveal that a user was warned about a Risky Exposure. Similarly, interactions with the latter could reveal that a user tested positive for SARS-CoV-2.

## Analytics Service

We begin by describing the genuine traffic between the Mobile Client and the Analytics Service, including the precautions in place to minimise the information an attacker might gain by analysing such traffic. Then, we describe how we generate dummy traffic to further hinder an attacker.

### Genuine analytics uploads

The Mobile Client sends to the Analytics Service data composed of Operational Info and the user’s Province of Domicile. The Operational Info includes:

- Whether permission to leverage the A/G Framework is granted
- Whether the device’s Bluetooth is enabled
- Whether permission to send local notifications is granted
- Whether the user was notified of a Risky Exposure after the last Exposure Detection
- The date on which the last Risky Exposure took place, if any

Of the data transmitted to the server, the most sensitive is _whether the user was notified of a Risky Exposure after the last Exposure Detection._

Let us define these data as Operational Info _with Exposure_ when the user was notified of a Risky Exposure after the last Exposure Detection, and as Operational Info _without Exposure_ when they were not.

To hinder an attacker from inferring whether the Mobile Client is uploading Operational Info with Exposure or without Exposure, we take the following precautions:

- **Timing and sequencing.** The timing of the upload and its sequencing relative to other requests do not depend on the type of Operational Info. In both cases, the upload is attempted straight after the Exposure Detection is performed.
- **Packet size.** Neither the request’s headers nor the size of the payload depends on the type of Operational Info. The App uses an integer (1 byte: '0' or '1') rather than a boolean (4 or 5 bytes: 'true' or 'false') for the field of the JSON-encoded payload determining whether the Operational Info is with Exposure or without Exposure. In the case that no Risky Exposure occurred, a dummy date is entered in the field dedicated to the date when the last Risky Exposure took place. We use a fixed dummy date, as the algorithms we employ for HTTPS encryption encrypt the same payload into a different sequence of bytes for each request, preventing an attacker from identifying in the packet a certain sequence of bytes associated with the dummy date and, therefore, inferring that no Risky Exposure has been detected. For all other fields, we also use integers rather than booleans, or, when using strings, we ensure that all possible values are of equal length. Besides further helping to protect the user’s privacy, this makes it easier to generate credible dummy traffic (more on this below).

The Mobile Client may attempt to upload Operational Info (together with the user’s Province of Domicile) right after each Exposure Detection.

However, due to the strategy we devised to ensure the integrity of analytics data (discussed in Privacy-Preserving Analytics, coming soon), such uploads are rate-limited. In particular, a Mobile Client cannot send Operational Info with Exposure and Operational Info without Exposure more than once each in a calendar month. Moreover, sampling is performed to decrease the level of traffic the infrastructure needs to support, while still retaining the informative nature of the analytics.

We will explain the implementation of such rate-limiting and sampling to clarify the rationale behind the design choices for the related dummy traffic (which we describe later in this document).

#### Operational Info with Exposure

After an Exposure Detection completes and identifies a Risky Exposure, the Mobile Client may attempt the upload of Operational Info with Exposure to the Analytics Service.

The Mobile Client starts by carrying out a _sampling test._ This involves the generation of a uniformly distributed random number between 0 and 1. If the value is less than a parameter _p<sub>AE</sub>,_ then the Mobile Client attempts the upload. Otherwise, it does not. In any case, the Mobile Client will refrain from performing another sampling test and, therefore, attempting another upload of Operational Info with Exposure until the next calendar month.

In practice, we expect to do no or minimal sampling of Operational Info with Exposure, as the information of how many users are being notified is important for the management of the epidemic. This means that _p<sub>AE</sub>_ should be as close to 1 as possible.

#### Operational Info without Exposure

After an Exposure Detection completes without a Risky Exposure being detected, the Mobile Client may attempt the upload of Operational Info without Exposure.

In all likelihood, an upload of Operational Info without Exposure will occur more frequently than an upload of Operational Info with Exposure. This is because the Mobile Client performs an Exposure Detection several times per day, and we expect only a fraction of them to reveal a Risky Exposure. Hence, the upload of Operational Info without Exposure needs to be sampled more aggressively.

The rate limit imposed on analytics data applies to the calendar month. This makes it necessary to schedule uploads in a way that prevents the Mobile Clients from generating a spike in analytics uploads on the first day of each calendar month.

To solve this issue, the first time the App activates during the current calendar month, it generates a random number _dT<sub>A</sub>_ uniformly distributed between 0 and (_n<sub>days</sub>_-1) * 86400, where _n<sub>days</sub>_ represents the number of days in the current calendar month and 86400 is the number of seconds in a day. Let us call _T<sub>A</sub>_ the timestamp (measured in seconds) corresponding to 12 am of the first day of the current month. The value of _dT<sub>A</sub>_ represents after how many seconds from _T<sub>A</sub>_ the window of opportunity will open for the App to send its monthly Operational Info without Exposure. The upload will happen no later than 24 hours after _T<sub>A</sub>_+_dT<sub>A</sub>._

We must consider that the user may install the App in the middle of a calendar month. In such a case, _dT<sub>A</sub>_ may represent a point in time in the past (i.e., earlier in the calendar month). This is by design. It ensures that the rate of sampling remains constant, no matter the distribution of installs within the month.

After performing the Exposure Detection, the Mobile Client verifies if the current time is between _T<sub>A</sub>_+_dT<sub>A</sub>_ and _T<sub>A</sub>_+_dT<sub>A</sub>_+86400. If so, the Mobile Client performs a sampling test similar to the one described for Operational Info with Exposure. It generates a uniformly distributed random number between 0 and 1. If the value is less than a parameter _p<sub>ANE</sub>,_ the Mobile Client attempts to perform the upload. Note that, if a sampling test for Operational Info without Exposure has previously been performed in the calendar month, the Mobile Client refrains from performing it again. It does not attempt to upload Operational Info without Exposure more than once in the same month. If, after performing an Exposure Detection, the current time is beyond _T<sub>A</sub>_+_dT<sub>A</sub>_+86400, then the Mobile Client does nothing. Instead, it waits for the next calendar month before again attempting an upload of Operational Info without Exposure.

### Dummy analytics uploads

The size of packets representing Operational Info with Exposure and without Exposure is the same. Therefore, it is virtually impossible for an attacker to distinguish between the two. However, the rate limit imposed on analytics uploads from the same Mobile Client exposes the user to a potential attack. An attacker could count the number of interactions between the Mobile Client and the Analytics Service. If they noticed that more than one attempt at uploading analytics data had been performed during the same calendar month, they would be sure that one was as a consequence of a Risky Exposure.

To mitigate this issue, the Mobile Client sends dummy requests that, while encrypted in transit, look just like any genuine analytics request. However, they do not contain any analytics data and are used solely to mitigate the risk that an attacker may identify genuine requests.

A deterministic approach, whereby the amount of dummy uploads on a monthly basis is fixed, would not solve the issue. For example, suppose the Mobile Client was to consistently send three dummy requests per calendar month. An attacker who then saw five requests sent in a given month would be certain that the user was notified of a Risky Exposure, due to the per-device rate limit described above.

Instead, we use a probabilistic approach. The model for the upload of dummy analytics allows for the sending of any number of requests within a month. This makes it difficult for an attacker to tell whether or not a Mobile Client uploaded Operational Info with Exposure by simply counting the number of analytics requests sent. To ensure the precaution is effective, these requests’ packets are the same size as those of the genuine ones.

When the App is first launched, it picks a duration _dT<sub>AD</sub>_ (measured in seconds) from an exponential distribution with mean _E_(_dT<sub>AD</sub>_), a parameter we can use for tuning purposes. It also stores the current date and time _T<sub>AD</sub>_ (a timestamp, also measured in seconds).

Subsequently, the following takes place after every occasion that an Exposure Detection is performed (note that Exposure Detections do not occur more than once each time the App activates in the background):

- If the App verifies that the current time is between _T<sub>AD</sub>_+_dT<sub>AD</sub>_ and _T<sub>AD</sub>_+_dT<sub>AD</sub>_+86400, it attempts to send a dummy request, updates _T<sub>AD</sub>_ to the current date and time, and picks a new value for _dT<sub>AD</sub>._
- If, instead, the App recognises that the current time is beyond _T<sub>AD</sub>_+_dT<sub>AD</sub>_+86400, it updates _T<sub>AD</sub>_ to the current date and time, picks a new _dT<sub>AD</sub>,_ but does not attempt to send a dummy request. This can happen if the device was not connected to the Internet between _T<sub>AD</sub>_+_dT<sub>AD</sub>_ and _T<sub>AD</sub>_+_dT<sub>AD</sub>_+86400, or if the App did not activate in the background during the same period.

In the unlikely event that both a dummy request and a genuine request are scheduled to be sent after the same Exposure Detection, only the genuine upload is attempted. Both _T<sub>AD</sub>_ and _dT<sub>AD</sub>_ are updated, as would be the case if the upload of the dummy data was actually attempted. This is important because two dummy uploads cannot be attempted after the same Exposure Detection, and the Mobile Client performs only one Exposure Detection each time it activates in the background. Hence, if an attacker were to observe that two uploads were attempted at roughly the same time, they would know that at least one of them was genuine.

When uploading data to the Analytics Service, the Mobile Client sets a dedicated header in the request. This lets the Analytics Service know whether the payload needs to be discarded (if it was a dummy one) or not (if it was a genuine one). The Exposure Ingestion Service waits for a random amount of time (picked from an exponential distribution) before it responds to a dummy request, so as to simulate the response time of a genuine request.

To make the encrypted dummy traffic indistinguishable from genuine analytics traffic while in transit, we adopt the following measures:

- All requests—regardless of whether they are genuine or dummy—are made to the same IP address resolved by the same domain.
- The request header used to indicate whether the data are genuine or dummy is always set and its length is the same in either case.
- The size of the payload of dummy requests is the same as that of genuine ones. This is easy to achieve, because genuine analytics payloads have a fixed size, as explained earlier.

## Exposure Ingestion Service

First, we describe the genuine interactions between the Mobile Client and the Exposure Ingestion Service, including details on how this genuine traffic is configured to hinder traffic analysis. Then, we focus on how dummy traffic is generated to further limit the information an attacker may infer by studying the interactions between the Mobile Client and the Exposure Ingestion Service.

### Genuine TEK upload sequences

When a user tests positive for SARS-CoV-2, they can choose to upload their TEKs so that the users at risk can be notified. Within the same upload, they will also send to the server their Province of Domicile and any Epidemiological Info from the previous 14 days.

The operation takes place with the help of an authorised Healthcare Operator. The user navigates to a specific section in the App and dictates the OTP they find there to the Healthcare Operator. The Healthcare Operator enters the OTP into the HIS, unlocking the upload of the data for the user.

Then, the user generally goes through the following steps:

1. **OTP validation.** The user taps a button, which causes the Mobile Client to contact the Exposure Ingestion Server and validate that the OTP has been authorised by a Healthcare Operator.
2. **Data upload.** After seeing a recap of the data that are about to be sent, the user gives their confirmation and the Mobile Client contacts the Exposure Ingestion Service again, beginning the upload.

To be completed successfully, each step requires at least a server request to be attempted by the Mobile Client.

In reality, there are several reasons why either of these two steps may fail and require repeating. For example, the user could make a mistake, such as attempting to validate the OTP before it is authorised by the Healthcare Operator. Therefore, in some cases, we may witness more than two requests being sent to the Exposure Ingestion Service before the user’s TEKs are uploaded. The user may also quit the process after attempting to validate the OTP. For example, they may have decided that they no longer want to share their data. In that case, the process ends after only one request.

We define the _TEK upload sequence_ as the sequence of any number of server requests attempted in order to complete the OTP validation and data upload steps for the same Mobile Client. A genuine TEK upload sequence always starts with an attempt at validating an OTP. It may end right there or include one or more additional requests.

To make it more difficult for an attacker to infer information from the analysis of the encrypted traffic between the Mobile Client and the Exposure Ingestion Service, and to make it easier to generate credible dummy traffic, the OTP validation and data upload steps are made indistinguishable:

- They both make requests to the same IP address resolved by the same domain
- Padding bytes are added to make their distribution of payload sizes as similar as possible

Let us explain how padding bytes are added to both OTP validation and data upload requests.

Starting with the data upload requests:

- _b<sub>OTP</sub>_ fixed bytes are added to simulate the presence of an OTP in the payload. The use of a fixed sequence of bytes will not expose the user to an attacker analysing the traffic generated by the user’s Mobile Client. This is because the algorithms we use for HTTPS encryption ensure that the same payload is encrypted into a different sequence of bytes with every new request.
- _b<sub>TEK</sub>_ represents the number of bytes for a TEK (it is fixed). The number of padding bytes added is (14-_n<sub>TEKs</sub>_) * _b<sub>TEK</sub>,_ where _n<sub>TEKs</sub>_ is the number of TEKs actually available for upload. This simulates a set of TEKs that is always full, no matter how recently the user installed the App. A fixed sequence of bytes is used for dummy TEKs. The Exposure Ingestion Service will recognise these TEKs as dummy, and discard them.
- _b<sub>UN</sub>_ random bytes (both the amount of bytes and their content are random) are added to make the payload more noisy. This helps in two ways. First, it makes it more difficult to infer information about the presence of Epidemiological Info in the payload. If no noise were to be added, an attacker who knows the fixed size of a packet when no Epidemiological Info is included could then observe larger packets and infer that Epidemiological Info must be included. Second, the more noisy the genuine upload request’s packet size, the easier this is to mimic, both with the genuine OTP validation requests and with the dummy ones we discuss in the next section. _b<sub>UN</sub>_ is picked from an exponential distribution with mean _E_(_b<sub>UN</sub>_) and capped at _M_(_b<sub>UN</sub>_), parameters we can use for tuning purposes.

Concerning the OTP validation requests, we add padding bytes to their payloads as follows, with the stated goal of making them indistinguishable from data upload requests:

- _b<sub>PD</sub>_ fixed bytes are added to simulate the presence of the Province of Domicile in the payload.
- 14 * _b<sub>TEK</sub>_ fixed bytes are added to simulate a full set of TEKs.
- _b<sub>VN</sub>_ random bytes (both the amount of bytes and their content are random) are added to make the payload more noisy, so to simulate the presence in the payload of both Epidemiological Info and the noisy padding added to the data upload requests. _b<sub>VN</sub>_ is picked from an exponential distribution with mean _E_(_b<sub>VN</sub>_) and capped at _M_(_b<sub>VN</sub>_), parameters we can use for tuning purposes. Note that _E_(_b<sub>VN</sub>_) and _M_(_b<sub>VN</sub>_) are larger than _E_(_b<sub>UN</sub>_) and _M_(_b<sub>UN</sub>_) respectively, to simulate the presence of Epidemiological Info.

When it comes to the amount of noise bytes (_b<sub>UN</sub>_ and _b<sub>VN</sub>_), a different value is picked every time the Mobile Client needs to send a relevant server request. Otherwise, an attacker could tell an OTP validation request and a data upload request apart simply by analysing the size of their respective packets and the order in which they occur.

### Dummy TEK upload sequences

Only users who tested positive for SARS-CoV-2 can initiate a genuine TEK upload sequence. This is the only case in which a Mobile Client generates genuine traffic towards the Exposure Ingestion Service. Hence, without proper countermeasures, an attacker observing such traffic could infer that the user tested positive for SARS-CoV-2.

To prevent this from happening, the Mobile Client generates dummy traffic towards the Exposure Ingestion Service on a regular basis. A dedicated header allows the App to indicate to the Exposure Ingestion Service that a certain request is a dummy one. The Exposure Ingestion Service immediately discards the payload upon receiving dummy requests. The Exposure Ingestion Service waits for a random amount of time (picked from an exponential distribution) before it responds to a dummy request, so as to simulate the response time of a genuine request.

To make the encrypted dummy requests indistinguishable from genuine OTP validation or data upload requests while in transit, we adopt the following measures:

- All requests—regardless of whether they are genuine or dummy—are made to the same IP address resolved by the same domain
- The header is set to the appropriate value for each request, whether dummy or genuine, and both values have the same size
- The size of the payload of dummy requests mimics that of genuine ones

To implement this last measure, we add the following padding to the payload of dummy requests (note that the parameters are the same described above for the padding of genuine requests):

- _b<sub>OTP</sub>_ fixed bytes are added to simulate the presence of an OTP in the payload.
- _b<sub>PD</sub>_ fixed bytes are added to simulate the presence of the Province of Domicile in the payload.
- 14 * _b<sub>TEK</sub>_ fixed bytes are added to simulate a full set of TEKs.
- _b<sub>TD</sub>_ random bytes (both the amount of bytes and their content are random) are added to make the payload more noisy, so as to simulate the presence in the payload of both Epidemiological Info and the noisy padding added to the data upload requests. _b<sub>TD</sub>_ is picked from an exponential distribution with mean _E_(_b<sub>VN</sub>_) and capped at _M_(_b<sub>VN</sub>_)—this is the same distribution used for the padding of OTP validation requests’ payloads. A new value of _b<sub>TD</sub>_ is picked every time the Mobile Client needs to send a relevant dummy server request.

Let us break down the issue of generating effective dummy traffic into the following subproblems:

- **TEK upload sequence simulation.** We must simulate the TEK upload sequence so that it resembles the genuine one closely enough that an attacker cannot discern whether or not a sequence is genuine merely by analysing the sequence itself.
- **Traffic pattern modulation.** We must also trigger the simulation of the TEK upload sequence with the proper frequency and within an appropriate overall pattern of network traffic, so that an attacker cannot discern whether or not a sequence is genuine merely by analysing the pattern of TEK upload sequences and other traffic occurring over time.

#### TEK upload sequence simulation

Simulating a TEK upload sequence is more difficult than mimicking other interactions between the Mobile Client and the Backend Services. This is because two different requests are involved (OTP validation and data upload). Additionally, the sequence of requests and timings of a genuine TEK sequence depends on the interaction between the user and a Healthcare Operator, which results in a sequence that may include any number of requests with delays of variable length in between.

For our simulation to be effective, we must ensure the following:

- **Most simulated sequences could be genuine.** This means that, if an attacker analyses such sequences, they cannot tell that the sequences are not genuine simply based on their publicly observable characteristics. The model powering the simulated sequences must produce credible sequences in almost all cases.
- **Most genuine sequences could be simulated.** This means that an attacker cannot be sure that a certain sequence is genuine simply based on its publicly observable characteristics. Most of the possible genuine sequences must be explainable through the model powering the simulated sequences.
- **The distribution of different types of simulated sequences follows that of genuine ones.** If a somewhat common genuine sequence is almost never simulated, an attacker observing such a sequence would be able to infer that it is most likely a genuine one.

The diagram below represents a simulated TEK upload sequence. The steps involved are as follows:

- **Send a request.** The yellow diamond represents the request being sent. Whether that request simulates an OTP validation or a data upload is immaterial, as the two cases are indistinguishable based on their publicly observable characteristics (as described earlier). The exit from the yellow diamond occurs after the request receives a response from the server, or when it times out.
- **Stop.** The orange diamond represents the end of the simulated TEK upload sequence.
- **Add a delay.** The blue diamond represents a delay added to make the simulated sequence more realistic by mimicking user-introduced delays.

_[Diagram coming soon]_

The model uses the following parameters:

- _dT<sub>TR</sub>_ is the value that determines the delay between a server response or timeout and the subsequent request attempt. It is picked from an exponential distribution with mean _E_(_dT<sub>TR</sub>_), a parameter we can use for tuning purposes.
- _p<sub>T</sub>_(_i_), with _i_≥0, is the probability that a user would continue the TEK upload sequence, attempting to send a new request, after the (_i_+1)th request has been completed or has timed out.

Such a model allows the Mobile Client to replicate a wide array of genuine sequences. This ensures that most simulated sequences could be genuine and most genuine sequences could be simulated.

It is also crucial that the most common genuine sequences are also common when simulated, and vice versa. The model supports this too.

With a genuine TEK upload sequence, a likely pattern is that a data upload soon follows a correct OTP validation. Another realistic, albeit less likely, outcome is that after an initial error when validating the OTP, it is validated correctly during the second attempt, and a correct data upload follows. _p<sub>T</sub>_(_i_) helps to model such a pattern. For example, we could set _p<sub>T</sub>_(0) to 0.95 and _p<sub>T</sub>_(_i_) to 0.1 for _i_>1. This configuration makes it likely that the simulated TEK upload sequence will have two requests before stopping, but it also accounts for the possibility that more requests will be simulated. This is to mimic a genuine fail-retry cycle.

#### Traffic pattern modulation

Let us start by defining two types of sessions:

- **Foreground sessions.** These sessions are initiated manually by a user when opening the App.
- **Background sessions.** The operating system initiates these sessions following a schedule set by the operating system itself (in the case of iOS) or by the App (in the case of Android). With Immuni, they are leveraged primarily to perform the Exposure Detection (including downloading new TEK Chunks) without having to wait for the user to open the App.

Genuine TEK upload sequences can only occur in foreground sessions, as they must be initiated by the user. The Mobile Client must not predictably perform any requests during a foreground session in addition to those strictly necessary to perform the TEK upload sequence. Otherwise, any TEK upload sequence taking place without these other expected requests would be identifiable as non-genuine by an attacker. Moreover, if most simulated TEK upload sequences could be identified as non-genuine, the attacker would also know with a high degree of probability that the other TEK upload sequences must be genuine. It is crucial to avoid such a scenario.

For example, if the Mobile Client fetched the Configuration Settings every time a user opens the App, this request would have to be simulated as well. Failure to do so would give an attacker a heuristic to distinguish genuine TEK upload sequences from simulated ones: ‘When simulated, TEK upload sequences are not preceded by a call to the Configuration Settings endpoint’.

For the above-mentioned reasons plus the complexity of credibly simulating additional requests, no requests are performed during foreground sessions besides those related to genuine TEK upload sequences (which we must simulate). One exception is made: when a user reads the Terms of Use or the Privacy Notice, which we assume will occur very infrequently. Noticing that neither of these two requests has occurred shortly before or after a TEK upload sequence does not provide any useful information to an attacker. Instead, the Mobile Clients leverage background sessions to perform all the necessary server requests other than those belonging to the TEK upload sequence (e.g., Configuration Settings and FAQ fetching). The only exception is the App’s first launch, when some additional requests are performed.

As briefly mentioned above, there is a difference between the way background sessions are scheduled on iOS and on Android. On iOS, the App has no control over the schedule—the operating system triggers background sessions once every few hours. On Android, the App has full control over the background session schedule. Due to this difference, we handle the simulation of TEK upload sequences differently on iOS and on Android.

##### iOS

According to our experimental observation (the behaviour is not documented), iOS background sessions are spaced in time by at least a couple of hours (there are some exceptions, such as when the phone is charging, in which case it seems that background sessions take place more frequently). It appears extremely unlikely for a background session to be followed shortly after by another background session. Moreover, when the user starts a foreground session, this appears to introduce a delay to the subsequent background session.

This behavior provides an attacker with a heuristic to identify at least some foreground sessions as such: ‘Any session that started less than two hours after another is likely to be a foreground session’. Since, as explained, at least in some cases it is possible for an attacker to distinguish between background and foreground sessions by analysing network traffic, simulating TEK upload sequences during background sessions is not an optimal choice. In that case, an attacker may infer that the sequences are simulated, because no genuine TEK upload sequence takes place during a background session. Therefore, TEK upload sequences are simulated during foreground sessions only.

To avoid introducing patterns that an attacker could exploit, the scheduling of simulated TEK upload sequences takes place in a probabilistic way, similar to the case of dummy analytics. When the App is first launched, it generates a value _dT<sub>T,iOS</sub>_ (a timestamp, measured in seconds) from an exponential distribution with mean _E_(_dT<sub>T,iOS</sub>_) (a parameter we can use for tuning purposes), and it stores the current date and time _T<sub>T</sub>_ (a timestamp, measured in seconds). The App is given an _opportunity window_ to simulate a TEK upload sequence, starting at time _T<sub>T</sub>_+_dT<sub>T,iOS</sub>_ and ending at time _T<sub>T</sub>_+_dT<sub>T,iOS</sub>_+_W<sub>T,iOS</sub>,_ where _WT<sub>T,iOS</sub>_ is a fixed parameter representing the duration of the opportunity window (measured in seconds).

Whenever the user starts a foreground session by opening the App, the App checks if the current time falls within the opportunity window. There are three possible outcomes:

- The current time is before the start of the opportunity window. No simulation occurs.
- The current time is beyond the opportunity window. The App picks another value _dT<sub>T,iOS</sub>_ and updates _T<sub>T</sub>_ to the current date and time. No simulation occurs.
- The current time falls within the opportunity window. In this case, the App simulates the TEK upload sequence. As soon as the simulation starts, a new _dT<sub>T,iOS</sub>_ is picked and _T<sub>T</sub>_ is updated to the current date and time.

When a background session starts, if the current time is beyond the opportunity window, the App picks another value _dT<sub>T,iOS</sub>_ and updates _T<sub>T</sub>_ to the current date and time.

In the case that a user almost never opens the App, it may be unlikely that simulated TEK upload sequences are performed for that user. However, this is not a problem. No interactions occur between the Mobile Client and the Backend Services during foreground sessions. Therefore, in the case of a TEK upload sequence, an attacker would have no way of knowing the frequency with which a certain user opens the App simply based on analysing that user’s Mobile Client’s traffic. It follows that, when they observe a TEK upload sequence for that Mobile Client, such a piece of information is insufficient for them to determine whether the user actually tested positive for SARS-CoV-2 or whether the sequence was simulated.

If the user initiates a genuine TEK upload sequence when a simulated one is ongoing, an attacker may infer that some of the requests are genuine. To mitigate this risk, rather than starting the simulated TEK upload sequence immediately, the App waits for a random time interval _dT<sub>TS,iOS</sub>_ from the beginning of the foreground session. This delay is generated from an exponential distribution with mean _E_(_dT<sub>TS,iOS</sub>_), a parameter we can use for tuning purposes.

_dT<sub>TS,iOS</sub>_ must be a random variable. If it were a fixed parameter, in some cases an attacker might infer that an observed TEK upload sequence is genuine. For example, if _dT<sub>TS,iOS</sub>_ was always equal to 10 seconds and the attacker observed a request to fetch the Terms of Use occur, say, 12 seconds before a TEK upload sequence starts, they would know that the sequence is likely genuine.

If a user navigates to the screen dedicated to initiating the TEK upload sequence, any TEK upload sequence simulations already running are aborted. Any scheduled to occur during the same foreground session are also cancelled. This is to prevent too many requests being sent within the same foreground session, from which an attacker might otherwise conclude that at least one is likely genuine.

Note that this solution does not prevent gainful traffic analysis in the unlikely case that a user starts a TEK upload sequence while a server request of a simulated sequence is occurring. For this to happen, a few conditions need to be met. First, a simulated sequence should start within the same foreground session during which the user performs their genuine sequence. Second, the simulated sequence should not have been completed by the time the user initiates the genuine sequence. Third, when the genuine sequence starts, a request from the simulated one should be ongoing. This last condition is especially unlikely to be verified. This is because most requests only last in the order of seconds or tenths of a second, and any request of a simulated sequence that is not already underway is cancelled the moment the user reaches the screen dedicated to initiating a genuine TEK upload sequence.

While it is possible to mitigate this risk, we deem it too remote to justify further complicating the business logic of the iOS App.

##### Android

As mentioned earlier, Android makes it possible for an App to choose when a background session should start. Moreover, unlike with the iOS App, there is no interference between foreground sessions and background sessions. Hence, on Android, we can simulate TEK upload sequences during background sessions scheduled specifically for this use, rather than having to wait for the user-initiated foreground sessions. Furthermore, since no server requests are performed during a foreground session other than those related to the TEK upload sequence, an attacker would not be able to determine that the sequence occurred in the background and, therefore, could not be genuine.

Similarly to iOS, on Android we use _T<sub>T</sub>_ and _dT<sub>T,And</sub>_ to determine when the Mobile Client should simulate the TEK upload sequence (that is, when it should schedule the next dedicated background session). These values are set at the first launch of the Android App. On Android, we are simulating the TEK upload sequence in background sessions that occur precisely on a schedule. Therefore, we do not need a parameter defining the duration of an opportunity window, such as _W<sub>T,iOS</sub>_ for the iOS App. Also, the mean of the variable _dT<sub>T,And</sub>,_ _E_(_dT<sub>T,And</sub>_), may differ from _E_(_dT<sub>T,iOS</sub>_).

As soon as the dedicated background session starts, the first request of the simulated TEK upload sequence is fired.

The variables _T<sub>T</sub>_ and _dT<sub>T,And</sub>_ are updated in the following cases:

- When an attempt to simulate a TEK upload sequence starts
- During a background or foreground session, if the App determines that the current time is beyond _T<sub>T</sub>_+_dT<sub>T,And</sub>_ (this may occur in the case that the scheduled background session failed to start)

In the unlikely event that the planned simulated TEK upload sequence should occur while a foreground session is ongoing, the simulated sequence is cancelled and rescheduled. If a foreground session starts while the simulated sequence is underway, the simulation is immediately aborted.

## Configurability

All of the parameters that allow for the tuning of the traffic-analysis mitigation measures described above can be changed from the server side. This is achieved by modifying the Configuration Settings served by the App Configuration Service.

These parameters include the following (sorted by the order in which they appear above):

- **_p<sub>AE</sub>_.** The sampling rate for Operational Info with Exposure.
- **_p<sub>ANE</sub>_.** The sampling rate for Operational Info without Exposure.
- **_E_(_dT<sub>AD</sub>_).** The mean (measured in seconds) of the exponential distribution determining how frequently dummy analytics uploads occur.
- **_b<sub>OTP</sub>_.** The size of an OTP in bytes.
- **_b<sub>TEK</sub>_.** The size of a TEK in bytes.
- **_E_(_b<sub>UN</sub>_).** The mean (measured in bytes) of the exponential distribution determining the amount of noise bytes added to the payload of data upload requests within a genuine TEK upload sequence.
- **_M_(_b<sub>UN</sub>_).** The cap (measured in bytes) applied to values picked from the exponential distribution determining the amount of noise bytes added to the payload of data upload requests within a genuine TEK upload sequence.
- **_b<sub>PD</sub>_.** The size of the Province of Domicile in bytes.
- **_E_(_b<sub>VN</sub>_).** The mean (measured in bytes) of the exponential distribution determining the amount of noise bytes added to the payload of OTP validation requests within a simulated TEK upload sequence.
- **_M_(_b<sub>VN</sub>_).** The cap (measured in bytes) applied to values picked from the exponential distribution determining the amount of noise bytes added to the payload of OTP validation requests within a simulated TEK upload sequence.
- **_E_(_dT<sub>TR</sub>_).** The mean (measured in seconds) of the exponential distribution determining the length of the delay between one request’s completion or timeout and the next request attempt in a simulated TEK upload sequence.
- **_p<sub>T</sub>_.** An array of the probabilities (float, ranging between 0 and 1) of the Mobile Client attempting a new request within a simulated TEK upload sequence after a request has been sent and a response has been received or a timeout has occurred. *p<sub>T</sub>*(*i*) represents the probability of the Mobile Client continuing after the (*i*+1)th request. If *N* is the length of the array, *p<sub>T</sub>*(*N*-1) is used by the Mobile Client to determine whether or not to make any attempt following the *N*th.
- **_E_(_dT<sub>T,iOS</sub>_).** The mean (measured in seconds) of the exponential distribution determining how frequently simulated TEK upload sequences occur on iOS.
- **_dW<sub>T,iOS</sub>_.** The duration (in seconds) of the opportunity window to simulate TEK upload sequences on iOS.
- **_E_(_dT<sub>TS,iOS</sub>_).** The mean (measured in seconds) of the exponential distribution determining the length of the delay between the start of a foreground session and the start of a simulated TEK upload sequence scheduled to occur within that session on iOS.
- **_E_(_dT<sub>T,And</sub>_).** The mean (measured in seconds) of the exponential distribution determining how frequently simulated TEK upload sequences occur on Android.
