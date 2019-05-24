# Answers to [Security and Privacy Questionnaire](https://www.w3.org/TR/security-privacy-questionnaire/)

### 3.1 Does this specification deal with personally-identifiable information?

No (though it is conceivable that a user could open sensitive information in a web app, perhaps editing their tax return in a web app made for that prupose).


### 3.2 Does this specification deal with high-value data?

Yes. It will grant read (and possibly write) access to some of a users files (if they choose to open them with a PWA). 


### 3.3 Does this specification introduce new state for an origin that persists across browsing sessions?

No.


### 3.4 Does this specification expose persistent, cross-origin state to the web?

No.


### 3.5 Does this specification expose any other data to an origin that it doesn’t currently have access to?

Yes. The origin may be granted access to files on a users machine. However, this will be building on top of the [native-file-system](https://github.com/WICG/native-file-system/blob/master/EXPLAINER.md) proposal and is more another avenue for getting access, than completely new data.

### 3.6 Does this specification enable new script execution/loading mechanisms?

No.


### 3.7 Does this specification allow an origin access to a user’s location?

No.


### 3.8 Does this specification allow an origin access to sensors on a user’s device?

No.


### 3.9 Does this specification allow an origin access to aspects of a user’s local computing environment?

Yes. This is another avenue of being granted access to certain files in the user's local computing environment. The user explicitly grants permission by a) installing the website and b) choosing to open a file with the installed website.


### 3.10 Does this specification allow an origin access to other devices?

No.


### 3.11 Does this specification allow an origin some measure of control over a user agent’s native UI?

No.


### 3.12 Does this specification expose temporary identifiers to the web?

No.


### 3.13 Does this specification distinguish between behavior in first-party and third-party contexts?

Third party contexts will not be able to see files that the first party was launched with.


### 3.14 How should this specification work in the context of a user agent’s "incognito" mode?

Sites may not be registered as a file handler unless they are installed, which is not possible in incognito mode. Thus, they will never receive access to any files via this API and cannot expect to (as they are not installed).


### 3.15 Does this specification persist data to a user’s local device?

Not directly, but once a site has been granted write access to a file it may persist data. The duration of access to files is undecided but we will defer to what is decided by the [native-file-system](https://github.com/WICG/native-file-system/blob/master/EXPLAINER.md) folks.


### 3.16 Does this specification have a "Security Considerations" and "Privacy Considerations" section?

Yes See the [explainer](explainer.md#security-and-privacy-considerations).


### 3.17 Does this specification allow downgrading default security characteristics?

No.