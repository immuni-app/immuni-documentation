# Application Security

## Table of contents
- [Introduction](#introduction)
- [Information disclosure](#information-disclosure)
  - [Protection of data in transit](#protection-of-data-in-transit)
  - [Traffic analysis mitigation](#traffic-analysis-mitigation)
  - [Obfuscation of Apple’s DeviceCheck server-to-server calls](#obfuscation-of-apples-devicecheck-server-to-server-calls)
  - [Protection of data at rest](#protection-of-data-at-rest)
  - [Security of the Software Development Lifecycle](#security-of-the-software-development-lifecycle)
- [Service disruption or alteration](#service-disruption-or-alteration)
  - [Data signing](#data-signing)
  - [Authorization of TEKs uploads](#authorization-of-teks-uploads)
  - [Integrity of Analytics](#integrity-of-analytics)
  - [Malformed requests handling](#malformed-requests-handling)

## Introduction
This document provides an overview of the security measures adopted when designing Immuni. Some of the terminology used throughout assumes that you have already read the following documents:

- [High-Level Description](/High-Level%20Description.md)
- [Technology Description](/Technology%20Description.md)

## Information disclosure
Immuni may have access to sensitive information about a user. For example:

- When a user receives an Exposure Notification, the App implicitly knows that the user may be SARS-CoV-2-positive as a consequence of a Risky Exposure.
- When a user who tested positive for SARS-CoV-2 uploads their TEKs, the App implicitly knows about the user’s positive status.

The utmost care has been taken to mitigate the possibility of an attacker accessing or tampering with this information.

### Protection of data in transit
When the Mobile Client exchanges data with the Backend Services, the communication takes place over potentially insecure networks. To guarantee the confidentiality and integrity of data in transit, the following security measures are in place:

- The communication between the Mobile Client and the Backend Services uses HTTPS with TLS 1.2+ leveraging one of the following ciphers: 
  - ECDHE-RSA-AES256-GCM-SHA384 
  - ECDHE-RSA-AES128-GCM-SHA256
  - TLS_CHACHA20_POLY1305_SHA256 
- To reduce the likelihood of MITM attacks perpetrated by spoofing the Backend Services’ TLS certificate, the App performs strict checks that leverage CA pinning. We enforce CA pinning instead of leaf certificate pinning because it guarantees an adequate level of protection when combined with the other measures described below, while making it possible to easily manage and rotate the Server Certificate.
- A single CA has been authorised to sign certificates for the domain that the Immuni Backend Services use. The CA is Actalis S.p.A., a well-known Italian Certificate Authority whose CA services are fully compliant with the ‘Baseline Requirements Certificate Policy for the Issuance and Management of Publicly Trusted Certificates’ issued by the CA/Browser Forum. The choice of an Italian Certification Authority was based on the requirement that, to the maximum possible extent, the infrastructure should be physically located in Italy.
- A CAA DNS record on the domain has been set that entitles only Actalis S.p.A. to authorise the CA to sign certificates for the Immuni domain. This way, an attacker would not be able to perform a MITM attack by getting another CAA-compliant CA to sign certificates for the Immuni domain.

### Traffic-analysis mitigation
Without appropriate countermeasures, even if the data in transit are encrypted, an attacker could still infer sensitive information by analysing the communication between the Mobile Client and Backend Services. For example, an exchange of information between the Mobile Client and the Exposure Ingestion Service could reveal that the user is uploading their TEKs and has therefore tested positive for SARS-CoV-2. To minimise this risk, the following measures are in place:

- Mobile Clients regularly send dummy requests that are comparable to genuine ones in size and timing. An attacker cannot know beforehand if a network communication that they observe is genuine or dummy, which makes it harder to infer sensitive information simply based on the detection of such requests.
- All genuine and dummy requests made by the Mobile Client to the Analytics Service have the same payload size, irrespective of their content. This makes it impossible for an attacker to use the packet size as a proxy for the information it contains. For example, the standardised size ensures that an attacker cannot use packet size discrepancies to infer that a Risky Exposure has occurred.
- The requests made by the Mobile Clients toward the Exposure Ingestion Service contain dummy bytes to make the size of different requests comparable. This makes it difficult for an attacker to use packet size as a proxy for the information it contains. For example, the comparable size ensures that an attacker cannot use packet size discrepancies to infer with certainty that a request contained the user’s TEKs.

An in-depth description of the techniques in place to thwart traffic analysis attacks is available in the [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md) document.

### Obfuscation of Apple’s DeviceCheck server-to-server calls
As explained below in [Integrity of analytics](#integrity-of-analytics), Immuni leverages [Apple’s DeviceCheck](https://developer.apple.com/documentation/devicecheck) to protect the Analytics Service from attackers who attempt to pollute Analytics data. The solution uses DeviceCheck’s per-device bits to keep track of the type of Analytics a device sends in a given month. Without appropriate countermeasures, an attacker gaining access to the DeviceCheck system could infer sensitive information by inspecting the per-device bits. Although we deem the likelihood of this attack to be very low, we still mitigate it. When the Analytics Service receives a dummy request, it decides with a certain probability whether it should leverage the request for the purpose of obfuscating interactions with DeviceCheck. If it decides to do so, the Analytics Service sets the per-device bits to the configuration representing a Risky Exposure. With this measure in place, an inspection of the per-device bits does not suffice to infer with certainty that a Mobile Client was notified of a Risky Exposure. More information can be found in the Privacy-Preserving Analytics document (coming soon).

### Protection of data at rest
In the case that the Mobile Client is stolen, an attacker may gain access to the data Immuni stored on it by accessing raw data from the device storage. To prevent this, all data stored by the App are encrypted using AES. The AES decryption keys are stored in the OS-specific native key storage databases ([Keychain](https://developer.apple.com/documentation/security/keychain_services) on iOS and [Keystore](https://developer.android.com/training/articles/keystore) on Android).

### Security of the software development lifecycle
Immuni’s App and Backend Services use open-source, third-party libraries. Since these libraries have access to the same data that the Immuni codebase accesses, it is crucial that they are safe to use and cannot compromise the security of the system. For these reasons, all third party libraries are updated to versions without any known vulnerabilities. This check is performed while the dependencies are resolved and installed and it is integrated in the CI/CD pipelines.

Before deploying a code update to production, it needs to pass the following security and quality checks:

- Linters and static code quality checks
- Security-focused static code analysis
- Library and operating system vulnerability management

Moreover, the following checks are performed on a regular basis:

- Dynamic service-level testing and penetration tests (both internal and external)
- Continuous vulnerability assessment

## Service disruption or alteration
To be as effective as possible in fighting the COVID-19 pandemic, Immuni needs to operate reliably. We devised several measures to ensure that the system is highly resistant to attacks that attempt to compromise its availability and modify its intended behaviour.

### Data signing
Immuni signs the TEKs available for download from the Exposure Reporting Service with an ECDSA-P256-SHA256 signing algorithm, identified by the following OID: 

1.2.840.10045.4.3.2.

The associated public key has been shared with Apple and Google and will be used by the mobile devices to verify the downloaded data and guarantee its authenticity and integrity.

Immuni will also implement a signing mechanism for other types of data served through the CDN, such as the Configuration Settings. This is not ready for Immuni’s initial release, but it will be available in a future version of the App.

### Authorisation of TEKs uploads
If any user could upload their TEKs to the Exposure Ingestion Service, an attacker may be able to trigger Exposure Notifications for users who are actually not at risk. If done at scale, such an attack could disrupt the system. The following measures are in place to minimisze this risk:

- Uploads to the Exposure Ingestion Service must be validated with an OTP that is generated randomly on the Mobile Client. The OTP must be validated by a Health Operator before it can be used by the App to authorise an upload request. 
- The OTP has a *time to live*, after which it can no longer be used to authorise an upload. The time to live is set at 2 minutes and 30 seconds initially, but it can be adjusted later if required.
The choice represents a trade-off between two conflicting goals:
  - Limiting the validity of the OTP to mitigate brute-force attacks.
  - Giving the user enough time to complete the upload in a situation of psychological distress.

  Requiring the OTP to be validated by a Healthcare Operator restricts access to the Exposure Ingestion Service to users who tested positive for SARS-CoV-2.
- The OTP is composed of ten characters from a set of 25 symbols (A, E, F, H, I, J, K, L, Q, R, S, U, W, X, Y, Z, 1, 2, 3, 4, 5, 6, 7, 8, 9). The first nine characters are picked randomly, while the last character is computed algorithmically, serving the purpose of a check digit. There are 25<sup>9</sup> possible OTPs, a space equivalent to slightly less than 42 bits of entropy. Even if this increases the probability of collisions, we chose to exclude from the standard alphabet characters that may be ambiguous when read to the Healthcare Operator. This provides the best trade-off between security and user experience.
- Adequate infrastructural rate limiting techniques are in place to slow down or prevent attempts to brute-force the OTPs.
- With a valid OTP, the App is allowed to upload a maximum of 14 TEKs, corresponding to those generated in the previous 14 days. This limit makes it impossible for an attacker in possession of a valid OTP to flood the system by uploading an arbitrarily large number of TEKs that could result in notifying users who are not at risk.

### Integrity of Analytics
The integrity of the Analytics data will be crucial to identify issues, and for the National Healthcare System to allocate the necessary resources to care for the users alerted by the App. Erroneous Analytics may result in erroneous decisions. On the other hand, for privacy reasons, it is not viable to authenticate calls to the Analytics Service. To guarantee the integrity of Analytics while preserving user privacy, the following measures are in place:

- Data uploaded to the Analytics Service must be validated using a method that guarantees the data is being sent from a real device, without identifying either the device or the owner. To fulfil these requirements, we leverage Apple’s DeviceCheck framework. For Android devices, we have not identified a valid technology that satisfies the requirements with sufficient guarantees yet. For this reason, only iOS devices are allowed to upload Analytics. This strategy is an acceptable trade-off between system visibility and user privacy.
- Requests to the Analytics Service must be rate-limited to prevent an attacker from sending an arbitrarily large number of requests from a single genuine device. This is also accomplished by leveraging Apple’s DeviceCheck framework in combination with a novel algorithm that we devised specifically to this end.

An in-depth description of how the Analytics Service leverages DeviceCheck against unreliable uploads by potential attackers is available in the Privacy-Preserving Analytics document (coming soon).

### Handling malformed requests
Attackers could send malformed data to the Backend Services. This could trigger unexpected behaviour, including making the Backend Services unresponsive. We have defined strict data types and patterns for our API endpoints so that malformed requests can be detected and immediately rejected.
