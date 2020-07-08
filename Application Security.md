# Application Security

## Table of contents
- [Introduction](#introduction)
- [Information disclosure](#information-disclosure)
	- [Minimisation of App permissions](#minimisation-of-app-permissions)
		- [iOS](#ios)
		- [Android](#android)
	- [Protection of data in transit](#protection-of-data-in-transit)
	- [Traffic analysis mitigation](#traffic-analysis-mitigation)
	- [Protection of data at rest](#protection-of-data-at-rest)
	- [Security of the software development lifecycle](#security-of-the-software-development-lifecycle)
- [Service disruption or alteration](#service-disruption-or-alteration)
	- [Data signing](#data-signing)
	- [Authorisation of TEK uploads](#authorisation-of-tek-uploads)
	- [Integrity of analytics](#integrity-of-analytics)
	- [Handling malformed requests](#handling-malformed-requests)

## Introduction
This document provides an overview of the application security measures adopted when designing Immuni. Some of the terminology used throughout assumes that you have already read the following documents:

- [High-Level Description](/High-Level%20Description.md)
- [Technology](/Technology.md)

Certain terms—written with a capital letter at their beginning—are defined in Technology’s glossary.

## Information disclosure
Immuni may have access to sensitive information about a user. For example:

- When it detects a Risky Exposure, the App implicitly knows that the user may be SARS-CoV-2-positive as a consequence
- When a user who tested positive for SARS-CoV-2 uploads their TEKs, the App implicitly knows about the user’s positive status

The utmost care has been taken to mitigate the possibility of an attacker accessing or tampering with this information.

### Minimisation of App permissions

The App asks only for those permissions strictly necessary to provide the intended functionality. This is to minimise the information about a user to which it has access. As there are differences between iOS and Android, we list permissions separately for each platform.

#### iOS

The App asks only for two permissions: to access the Exposure Notification framework and to display push notifications. The user is explicitly asked to grant these permissions at runtime. The App cannot function as intended without the acceptance of both permission requests. The user may revoke their permission at any time.

| Permission | Reason |
| :--------- | :----- |
| COVID‑19 Exposure Notifications | To access Apple’s COVID‑19 Exposure Notification framework and inform the user about a Risky Exposure |
| Push notifications | To display notifications to inform the user when the service is inactive or about a required update—these notifications are generated locally, not sent from a server |

In addition, the App declares the following entitlements:

| Entitlement | Value | Reason |
| :---------- | :---- | :----- |
| _com.apple.developer.exposure-notification_ | _true_ | To access Apple’s COVID-19 Exposure Notification framework |
| _com.apple.developer.default-data-protection_ | _NSFileProtectionCompleteUntilFirstUserAuthentication_ | To protect the App’s data until the user unlocks their device for the first time after reboot—this protects the data in the case of device theft |
| _aps-environment_ | _production_ | To display push notifications to the user—these notifications are generated locally, not sent from a server |

#### Android

The App asks only for permission to access the Exposure Notification framework. The user is explicitly asked to grant this permission at runtime. The App cannot function as intended without the acceptance of this permission request. The user may revoke their permission at any time.

| Permission | Reason |
| :--------- | :----- |
| COVID‑19 Exposure Notifications | To access Google’s COVID‑19 Exposure Notification framework |

In addition, the App lists the following permissions in the App’s manifest file:

| Permission | Reason |
| :--------- | :----- |
| _android.permission.INTERNET_ | To communicate with the Immuni Backend Services |
| _android.permission.BLUETOOTH_ | To use the Exposure Notification framework, as specified in [Exposure Notifications Android API Documentation](https://www.google.com/covid19/exposurenotifications/pdfs/Android-Exposure-Notification-API-documentation-v1.3.2.pdf) (page 3) |


### Protection of data in transit
When the Mobile Client exchanges data with the Backend Services, the communication takes place over potentially insecure networks. To guarantee the confidentiality and integrity of data in transit, the following security measures are in place:

- The communication between the Mobile Client and the Backend Services uses HTTPS with TLS 1.2+, leveraging one of the following ciphers: 

| URL        | Ciphers |
| :--------- | :------ |
| _get.immuni.gov.it_ | TLS 1.3:<br>_TLS\_CHACHA20\_POLY1305\_SHA256_<br><br>TLS 1.2:<br>_TLS\_ECDHE\_RSA\_WITH\_AES\_256\_GCM\_SHA384_<br>_TLS\_ECDHE\_RSA\_WITH\_AES\_128\_GCM\_SHA256_ |
| _upload.immuni.gov.it_<br>_analytics.immuni.gov.it_ | TLS 1.2:<br>_TLS\_ECDHE\_RSA\_WITH\_AES\_256\_GCM\_SHA384_<br>_TLS\_ECDHE\_RSA\_WITH\_AES\_128\_GCM\_SHA256_ |

- To reduce the likelihood of MITM attacks perpetrated by spoofing the Backend Services’ TLS certificate, the App performs strict checks that leverage Certificate Authority (CA) pinning. We enforce CA pinning rather than leaf certificate pinning because it guarantees an adequate level of protection when combined with the other measures described below, while making it possible to easily manage and rotate the Server Certificate.
- A single CA has been authorised to sign certificates for the domain used by the Immuni Backend Services. The CA is Actalis S.p.A., a well-known Italian Certificate Authority whose CA services are fully compliant with the ‘[Baseline Requirements Certificate Policy for the Issuance and Management of Publicly Trusted Certificates](https://cabforum.org/wp-content/uploads/CAB-Forum-BR-1.3.0.pdf)’ issued by the CA/Browser Forum. The choice of an Italian Certification Authority was based on the requirement that, to the maximum possible extent, the infrastructure should be physically located in Italy.
- A CAA DNS record on the domain has been set that entitles only Actalis S.p.A. to authorise the CA to sign certificates for the Immuni domain. This way, an attacker would not be able to perform a MITM attack by getting another CAA-compliant CA to sign certificates for the Immuni domain.

### Traffic analysis mitigation
Without appropriate countermeasures, even if the data are encrypted by the time they are in transit, an attacker could still infer sensitive information by analysing the communication between the Mobile Client and Backend Services. For example, an exchange of information between the Mobile Client and the Exposure Ingestion Service could reveal that the user is uploading their TEKs and has therefore tested positive for SARS-CoV-2. 

An in-depth description of the techniques in place to thwart traffic analysis attacks is available in [Traffic Analysis Mitigation](/Traffic%20Analysis%20Mitigation.md).

### Protection of data at rest
In the case that the Mobile Client is stolen, an attacker may gain access to the data Immuni stored on it by accessing raw data from the device storage. To prevent this, all data stored by the App are encrypted using AES. The AES decryption keys are stored in the OS-specific native key storage databases ([keychain](https://developer.apple.com/documentation/security/keychain_services) on iOS and [keystore](https://developer.android.com/training/articles/keystore) on Android).

### Security of the software development lifecycle
Immuni’s App and Backend Services use open-source, third-party libraries. Since these libraries have access to the same data that the Immuni codebase accesses, it is crucial that they are safe to use and cannot compromise the security of the system. For these reasons, all third-party libraries are updated to versions without any known vulnerabilities. This check is performed while the dependencies are resolved and installed and it is integrated in the CI/CD pipelines.

Before deploying a code update to production, it needs to pass the following security and quality checks:

- Linters and static code quality checks
- Security-focused static code analysis
- Library and operating system vulnerability management

Moreover, the following checks are performed on a regular basis:

- Dynamic service-level testing and penetration tests (both internal and external)
- Continuous vulnerability assessment

## Service disruption or alteration
To be as effective as possible in fighting the COVID-19 epidemic, Immuni needs to operate reliably. We have devised several measures to ensure that the system is highly resistant to attacks that attempt to compromise its availability and modify its intended behaviour.

### Data signing
Immuni signs the TEKs available for download from the Exposure Reporting Service with the ECDSA using Curve P-256 and SHA-256. The OID is 1.2.840.10045.4.3.2.

The associated public key has been shared with Apple and Google and is used by the Mobile Clients to verify the downloaded data and guarantee its authenticity and integrity.

Immuni will also implement a signing mechanism for other types of data served through the CDN, such as the Configuration Settings. This is not ready for Immuni’s initial release, but it will be available in a future version of the App.

### Authorisation of TEK uploads
If just any user could upload their TEKs to the Exposure Ingestion Service, an attacker may be able to trigger Exposure Notifications for users who are actually not at risk. If done at scale, such an attack could disrupt the system. The following measures are in place to minimise this risk:

- Uploads to the Exposure Ingestion Service must be validated with an OTP that is generated randomly on the Mobile Client. The OTP must be validated by a Healthcare Operator before it can be used by the Mobile Client to authorise an upload request. 
- The OTP has a _time to live,_ after which it can no longer be used to authorise an upload. The time to live is set at 2 minutes and 30 seconds. The choice represents a trade-off between two conflicting goals:
  - Limiting the validity of the OTP to mitigate brute-force attacks
  - Giving the user enough time to complete the upload in a situation of psychological distress

  Requiring the OTP to be validated by a Healthcare Operator restricts access to the Exposure Ingestion Service to only those users who tested positive for SARS-CoV-2.
- The OTP is composed of ten characters from a set of 25 symbols (_A, E, F, H, I, J, K, L, Q, R, S, U, W, X, Y, Z,_ 1, 2, 3, 4, 5, 6, 7, 8, 9). The first nine characters are picked randomly, while the last character is computed algorithmically, serving the purpose of a check digit. There are 25^9 possible OTPs, equivalent to slightly less than 42 bits of entropy. Even if it increases the probability of collisions, we chose to exclude from the standard alphabet characters that may be ambiguous when read to the Healthcare Operator. This provides a good trade-off between security and user experience.
- Adequate infrastructural rate-limiting techniques are in place to slow down or prevent attempts to brute-force the OTPs.
- With a valid OTP, the Mobile Client is allowed to upload a maximum of 14 TEKs, corresponding to those generated in the previous 14 days. This limit makes it impossible for an attacker in possession of a valid OTP to flood the system by uploading an arbitrarily large number of TEKs.

### Integrity of analytics
Immuni collects two types of analytics data:
- Epidemiological Info
- Operational Info

These data are crucial for identifying issues, and for the National Healthcare Service to allocate the necessary resources to care for the users alerted by the App.

Only users who test positive for SARS-CoV-2 may decide to upload Epidemiological Info to the Exposure Ingestion Service. In this case, they do so with the help of a Healthcare Operator and their upload is authenticated with an OTP.

Operational Info is uploaded to the Analytics Service automatically by the Mobile Client. To protect user privacy, the data are uploaded without requiring the user to authenticate in any way (e.g., no phone number or email verification). This exposes the Analytics Service to attacks intended to pollute the collected data. This is a major problem, because if the data cannot be trusted to be free of substantial inaccuracies, no reliable insight can be extracted from them.

An in-depth description of how the Analytics Service mitigates the risk of unreliable uploads by potential attackers is available in [Privacy-Preserving Analytics](/Privacy-Preserving%20Analytics.md).

### Handling malformed requests
Attackers could send malformed data to the Backend Services. This could trigger unexpected behaviour, including making the Backend Services unresponsive. We have defined strict data types and patterns for our API endpoints so that malformed requests can be detected and immediately rejected.
