# Investigating an Identity-Based Attack in Entra ID

## Scenario
A microsoft's azure tenant was compromised in last 24 hours through identity plane, although normal sign-in logs have been observed and no alerts has been triggered. The security team only observed some new items in a legacy application's app registration. This document will aim to reconstruct the attack chain, through foresnic analysis of two linked app registrations.

## Environment
Live, multi-user Azure training tenant with read-only access.

## Investigation
### Step 1: Entry > User got phished
 I opened the app registration through Azure portal and opened the reported legacy app's app registration. A note under 'Branding and properties' blade, left by incident team(fig.2). It suggests: A user was phished, completed MFA and had the resulting session token stolen. That token carried an MFA-satisfied claim, hence it bypassed Conditional Access policies. Apart from the evidence from azure portal, incident team reported that the user was also an Owner of an internal legacy connector app. 

![all_app_registrations]()

![internal_note]()

### Step 2: Escalate > Created New Client Secret
Under that legacy app's registration, I opened Certificates & secrets blade where I noticed a client secret with an expiry date(31/12/2099) in 73 years (fig. 3). It suggests that the attacker might create this key, to get higher privileged access as service principal(fig. 4) rather than staying as the user who has only owner rights for the app. Attacker not only intended to have elevated access without the need of signing in as the user but also to have indefinite access.

![certificates_and_secrets]()

![app_elevated_access]()

### Step 3: Pivot > Rogue App's Service Principal Set to Owner
Going back to App Registrations page, newly created app registration can be found. Incident team confirmed that it was created by the attacker.Inside it, under 'Branding & properties' blade, a note was left by incident team(fig. 5). Under the legacy app's registration in Owners blade, a service principal(linked to newly created app) was listed as the legacy app's owner(fig. 6). Drawing from these 2 evidence: The Owners list on the legacy app held a service principal belonging to a second, attacker-created app, which meant they could mint fresh credentials indefinitely.

![app_owner]()

![new_app_note]()

### Step 4: Persistence > Creating Custom Scope
Under legacy app's registration, on 'Expose an API' blade, a custom scope was published(fig. 7). This turns the legacy app into a callable backend resource, which means the attacker's own app can request delegated access to it. The Expose an API blade only defines the scope. No durable access exists yet at this point. The persisting artifact, an OAuth is only created later, when a user is tricked into consenting.

![expose_an_api]()

### Step 5: Loot > Consent-phishing URL and Confused Deputy
 In rogue app's registration, under Authentication blade, 2 redirect URI was found (fig. 9). Combining the rogue app's client ID, that redirect URI and the exposed API scope produces a working phishing URL. A victim who is already signed in on a corporate device clicks Accept on a consent prompt, the rogue app gets a delegated token scoped to exactly that exposed API, acting as that user. This is consent phishing which bypasses all the steps of a regular credential phishing. While that does NOT get the legacy app's application permissions, it could allow for the attacker to act as the user on the legacy apps backend code to run a sync that does utilize the privileged API permissions. More dangerous part is: the resulting OAuth2PermissionGrant is not removed by a password rese nor by revoking sessions nor by enforcing MFA. Most standard containment playbooks leaves it in place.

![redirect_URI]()

![redirect_URI2]()

## What surprised me
- Without knowing account password, without needing MFA an account can be phished.
- Password, MFA can be bypassed, using consent phishing
- Not having Least privilege access, a tenant can be deeply compromised
- A user doesn't have same level access as a service principal, even if the user is a owner of the app
- A owner of an app, can create a new app and set the new app's service principal as owner of the previous app
- How creating client secret can enable indefinite access to an app, and it doesn't even require a human log-in
- Persistance can be achieved and maintained in different ways. Attention to detail and hacker mindset might help to defend against this kind of intrusion.

## Findings and recommendations
The investigation identified several potential weaknesses in the tenant's identity and application-access management:
1. Excessive application ownership: A standard employee reportedly held ownership rights over a highly privileged legacy application, creating an indirect privilege-escalation risk.
2. Suspicious application credentials: An unexpected Client Secret with an unusually distant expiry date suggested an attempt to establish an additional authentication route.
3. Application-based persistence: An attacker-created application's Service Principal was listed as an Owner of the legacy application, potentially allowing further management activity if the necessary permissions were present.
4. Potential consent-phishing preparation: A custom API scope and redirect URI configurations suggested the possibility of further OAuth-based attacks.

### Recommendations
. Revoke the client secret 
· Remove the rogue service principal from Owners 
· Delete the custom exposed API scope 
· Revoke the OAuth2PermissionGrant explicitly, because containment does not remove it 
· Remove the attacker redirect URI 
· Review and reduce the Graph application permissions 
· Disable default user app registration 
· Audit every app registration's Owners list the same way directory role membership is audited 
· Alert on new client secrets and new redirect URIs.
. Enforce least privilege access
. Phishing resistant MFA should be considered, instead of regular MFA. For service principal, instead of using client secret, stronger authentication method like certificate or, Managed Identity should be used

## What I learned
This investigation strengthened my understanding of Microsoft Entra ID, particularly the relationship between user identities, Service Principals, client credentials and API permissions.

It also demonstrated how attackers can exploit excessive privileges and application misconfigurations to establish persistence.

Most importantly, I learned that effective identity security requires more than MFA. Least-privilege access, secure application credentials, controlled consent and continuous monitoring are equally important.