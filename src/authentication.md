# Authenticated Sessions

In the [HTTP](http.md) tutorial, I discussed how its [stateless](http.md#stateless-protocol) nature is a classic software tradeoff---it enables good things like horizontal scaling, but it also makes things like authenticated sessions more difficult to support. In this tutorial I will show you how to support authenticated sessions over HTTP, which is actually a specific case of a more general problem: secure authorization tokens.

## Sessions

Under the hood, HTTP is a stateless protocol, but that's not your typical user experience on the web. When you use social media, or shop online, or pay bills electronically, you sign-in once, and then perform multiple operations as that authenticated user. You don't have to provide your credentials every time you do something that interacts with the server.

This implies that you have some sort of authenticated session with the server, but if HTTP is stateless, how is that possible? If your sign-in request went to one downstream server, but your subsequent request went to a different server, how does that different server know who you are?

Although we can't keep any state on the server related to the network connection, we can pass something back and forth between the client and the servers. This value is effectively a key to session state stored in the the server-side database or cache. The flow goes like this:

[![](https://mermaid.ink/img/pako:eNp1Uj1z6jAQ_Cuaqw3PJtgYTYYmNKnTZdwI6TCaYMnRRxLC8N_fCXAIA3EjS7t7qz3dHqRVCBw8vkc0EpdatE50jWH09cIFLXUvTGCSCc-ethpNuAV9Al_QfaC7BVUClyKIlfB4guVosfCced2akTbsU4fN48r9W0iHigy02PoT0RNRceZQUBUpbRzc1ehU4nxIDGmduogIosvo9Y79qnn0aNGgEwGZR--1Nex5eeX16fQVeBQJo5gPpLpyuJSiIKh-RMG-oRmYRJUpge-t8XjJeoc8tCWuTs-RYtHqw1GQlPctfsf9gzA08QxfN3HQ3CYUMWys09843OS2Yu_0R-oBJbTRSYn3g0MGHbpOaEXTtk-cBsIGO2yA06_CtYjb0EBjDkQlX_uyMxJ4cBEzcDa2G-BrekTaxV6R43lUf05bl2qf-WgUuqc0GsCns1kGNIqv1nYDgbbA9_AFvJgX43xaTiZlXuTzuqyqDHbA63Jcl9NZnVcPVVUVeXnI4PtYIB_P5g_ErOq6mJQETQ__AfOuErk?type=png)](https://mermaid.live/edit#pako:eNp1Uj1z6jAQ_Cuaqw3PJtgYTYYmNKnTZdwI6TCaYMnRRxLC8N_fCXAIA3EjS7t7qz3dHqRVCBw8vkc0EpdatE50jWH09cIFLXUvTGCSCc-ethpNuAV9Al_QfaC7BVUClyKIlfB4guVosfCced2akTbsU4fN48r9W0iHigy02PoT0RNRceZQUBUpbRzc1ehU4nxIDGmduogIosvo9Y79qnn0aNGgEwGZR--1Nex5eeX16fQVeBQJo5gPpLpyuJSiIKh-RMG-oRmYRJUpge-t8XjJeoc8tCWuTs-RYtHqw1GQlPctfsf9gzA08QxfN3HQ3CYUMWys09843OS2Yu_0R-oBJbTRSYn3g0MGHbpOaEXTtk-cBsIGO2yA06_CtYjb0EBjDkQlX_uyMxJ4cBEzcDa2G-BrekTaxV6R43lUf05bl2qf-WgUuqc0GsCns1kGNIqv1nYDgbbA9_AFvJgX43xaTiZlXuTzuqyqDHbA63Jcl9NZnVcPVVUVeXnI4PtYIB_P5g_ErOq6mJQETQ__AfOuErk)

We will dig into the details of each of these things below, but here is a brief overview the steps in that diagram:

1. When you sign in, the client sends your credentials to the server.
1. The server loads the account record associated with those credentials from the database. If no account is found, it returns a `401 Unauthorized` response.
1. The server verifies the provided credentials against the account record. If that fails, it returns a `401 Unauthorized` response.
1. The server generates a unique identifier for the session, known as the **session ID**. This should be a random value that is effectively impossible to guess given previous examples (e.g., a 128 or 256-bit value from a [cryptographically-secure random number generator](https://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator)).
1. The server inserts a Session record into the database (or cache) using the session ID as the primary key. The record also contains the ID of the authenticated account, and any other data you want to associate with the session.
1. The server generates a **digitally-signed session token** from the session ID, encodes it into an ASCII-safe text format (e.g., [base-64](https://en.wikipedia.org/wiki/Base64)).
1. The server includes this signed session token (not the bare session ID) in the response as an HTTP header.
1. The client holds on to this value, and sends it back in a header in all subsequent HTTP requests.
1. When one of the servers receives the subsequent request, it extracts the session token from the request header and verifies the signature. If the signature is invalid, the token has been tampered with, so the server returns a `401 Unauthorized` response.
1. If the signature is valid, the server extracts the session ID from the token, and loads the associated Session record from the database (or cache). 
1. The server now knows who the authenticated account is, and can decide if the current request is something that account is allowed to do. If not, the server returns a `403 Forbidden` response.

At a high level this is how authenticated sessions work over HTTP, but we can expand on each of these concepts in more detail.

## Accounts

To use your system, a customer (person, corporation, or agent) needs to create an **account**. First-time system designers will often assume that an account will always belong to just one person, and one person will create only one account, but that's hardly every true in the long run. Instead:

- A given customer will probably create multiple accounts. For example, it's common to create separate work and personal accounts on social media platforms.
- A given account will probably be used by multiple people, especially when your service costs money. Nearly every media streaming service learned this the hard way.

You can avoid some of these issues by designing your accounts to be **hierarchical** from the start. For example, let a customer create one main account to handle the service subscription and billing centrally, and then create child accounts for each of their family members, perhaps with restricted permissions. The same setup would be applicable for corporation with multiple subsidiaries, or a platform with multiple customers of their own.

Existing accounts should also be able to band together under a new parent account for centralized billing and monitoring---e.g., when two corporations merge, or when two people with individual accounts get married and want to take advantage of a family sharing discount.

The authorization rules for parent accounts will depend on your particular system. In some cases it will make sense to let parent accounts view all resources that belong to child accounts, but in others those child resources might need to remain private. In some cases it might also make sense for parent accounts to create new resources or adjust configuration on behalf of their child accounts, and in others this should not be allowed. Think about what would make sense in your scenario and move forward accordingly.

## Credentials

Regardless of your account structure, when a customer creates an account they must also provide some **credentials** they can use to prove their identity when signing in again in the future. These credentials are something the customer knows (password), something customer has (phone or hardware security key), or something customer is (biometrics tied to a passkey). Systems that manage particularly sensitive data might multiple of these.

### Something You Know: Email and Password

Typically systems start out requiring only an email and password. Email names are already unique, and account holders can prove their ownership of that email address by sending them a verification email with a link containing a secret one-time-use code (which is just another kind of [authorization token](#other-kinds-of-authorization-tokens)). Once verified, the email address can be used for notifications and account recovery. But don't use an email address as the account's primary key in your database---people sometimes need to change their email address, for example when leaving a school or corporation. So use an [application-assigned ID](ids.md) for the account, and store the email address as part of the account's credentials.

Passwords are again a classic design tradeoff. They are simple and familiar, and can be used anywhere the account holder happens to be, including shared computers or public internet terminals. But they are also a shared secret, so if an attacker learns an account's password, they immediately gain unrestricted access. This would be OK if people were disciplined about creating unique and strong passwords on each site, but sadly most people are not. Even those who are can be tricked into revealing their passwords on phishing sites that look like the real sign-in pages of popular services.

### Something You Are: Passkeys

Thankfully there is now an alternative to passwords that is well supported by the major browsers and client operating systems: **passkeys**. These use [asymmetric cryptography](crypto.md#asymmetric-encryption) to securely authenticate without ever passing a shared secret like a password over the network.

Passkeys are relatively simple to understand in principle, but their actual implementation can get complex, so you should definitely leverage the existing [official libraries](https://passkeys.dev/docs/tools-libraries/libraries/) for your client and server programming languages. To help you understand how they work, let's look at a simplified version of the sign-up flow:

[![](https://mermaid.ink/img/pako:eNp9Uk1vozAU_CuWD3siWRq-AlohRXDZM8ql4uKYF2It2Kw_dptG-e99Lkmp2qpczPOM570Z-0K56oAW1MBfB5JDLViv2dhKgt_EtBVcTExasifMkL0B_RmqPFQNAqT9DO48uHP2hKjgzKovBBrPaUD_-0q99mDNLDswA62cCdWqLJuCaD-1scg35g-cse6FsZpZoW7EZoXMyjPNpGRH_gt7IlKhU8KwVDiXJmryB8x8YmmwWxpYRbgGZsEvnbfChpm4Q-K-IAehRrBacMIWr29T7FezHBoURwGGiFcNe75r3Ie0TktDJncYUAkdBe_6kd_1B_sG0MI35DmAsqyRisHDe2Xy49dB_ywxW8a5chj0R2dLdEuoxDjOwZijG26R-juZR5qDMKKXKzcRrsZpAIsoDegIemSiw3d28eyWYkQjtLTA3w6OzA22pa28IhXjU81ZclpY7SCgWrn-RIsjGwxWburwEm6P9G231177xsdMQFfeES0e4jQJKD6jR6XGOwNLWlzoE8JZsk7jOEu22yzJt2GaBfSM22G03qRRlIdJmOSbPN5eA_r8qhCusySK8ywKN0m0ecjT6wtgMRAW?type=png)](https://mermaid.live/edit#pako:eNp9Uk1vozAU_CuWD3siWRq-AlohRXDZM8ql4uKYF2It2Kw_dptG-e99Lkmp2qpczPOM570Z-0K56oAW1MBfB5JDLViv2dhKgt_EtBVcTExasifMkL0B_RmqPFQNAqT9DO48uHP2hKjgzKovBBrPaUD_-0q99mDNLDswA62cCdWqLJuCaD-1scg35g-cse6FsZpZoW7EZoXMyjPNpGRH_gt7IlKhU8KwVDiXJmryB8x8YmmwWxpYRbgGZsEvnbfChpm4Q-K-IAehRrBacMIWr29T7FezHBoURwGGiFcNe75r3Ie0TktDJncYUAkdBe_6kd_1B_sG0MI35DmAsqyRisHDe2Xy49dB_ywxW8a5chj0R2dLdEuoxDjOwZijG26R-juZR5qDMKKXKzcRrsZpAIsoDegIemSiw3d28eyWYkQjtLTA3w6OzA22pa28IhXjU81ZclpY7SCgWrn-RIsjGwxWburwEm6P9G231177xsdMQFfeES0e4jQJKD6jR6XGOwNLWlzoE8JZsk7jOEu22yzJt2GaBfSM22G03qRRlIdJmOSbPN5eA_r8qhCusySK8ywKN0m0ecjT6wtgMRAW)

1. The client makes a request to the server to create a new passkey for the account.
1. The server responds with a unique and unguessable value, which is typically called a 'nonce'.
1. The client application asks its security hardware (known as the 'Authenticator') to create a new asymmetric key pair, encrypt the nonce with the private key, and return the encrypted nonce, the associated public key, and a unique ID for the passkey to the application. This triggers the device's biometric sensors to authenticate the person using the device (e.g., touch or face ID).
1. The client sends the encrypted nonce, public key, and passkey ID to the server.
1. The server uses the public key to decrypt the encrypted nonce and verify it matches the one the server sent in the the step 2.
1. The server stores the public key and its ID in the database as one of the account's passkey credentials.

Signing in works almost the same way, except that the key pair is already generated and the server already has the public key, so it only needs to verify the encrypted nonce against the previously-stored public key.

1. The client requests to sign-in.
1. The server responds with a unique and unguessable nonce.
1. The client application asks the Authenticator to encrypt the nonce with the previously created private key. This triggers the device's biometric sensors to authenticate the person using the device.
1. The client sends the encrypted nonce and the passkey ID to the server.
1. The server loads the previously-stored public key associated with the passkey ID, and uses it to decrypt the encrypted nonce, comparing the result to the nonce sent in step 2.
1. If it matches, the server knows that the client is in control of the associated private key, and is therefore authenticated.

Most client devices will now synchronize passkeys across devices signed into the same account. For example, Apple devices will share passkeys with all other Apple devices signed into the same iCloud account. This allows you to sign-in on your phone with a passkey originally created on your laptop, or vice-versa.

Passkeys will likely become the new standard for authentication, but they are still relatively new and unfamiliar, and some of your potential customers might not yet have a device capable of generating or using a passkey. Even if they do, if the device manufacturer doesn't offer a mechanism to backup and sync passkeys, your customers will lose access if their device is lost, stolen, or damaged. Without another credential, like an email and password, they may not be able to prove their identity during an account recovery flow.

That said, password managers like 1Password can act as software-based passkey Authenticators on devices that lack native passkey support. They also backup those keys to the cloud, and synchronize them across all kinds of devices from various manufacturers. Their mobile apps can even scan QR codes shown on a public computer screen (e.g., check-in kiosk) to perform passkey authentication via your phone. So passkeys can be used in a wide variety of scenarios provided your target customers are willing to install a password manager when needed.

### Something You Have: Phones and Hardware Keys

If your system manages particularly sensitive data, you might want to require another credential that is something the account holder _has_. For example:

- **mobile phone with text messaging:** The account holder can register a mobile phone number during sign-up, and during sign-in your system can send a unique, unguessable, one-time use code via text messaging that the account holder must enter into a challenge form. This benefits from the ubiquity of text messaging, but it's also not as secure, as SMS text messages are not encrypted and can be intercepted.
- **authenticator app:** The account holder can install an authenticator application on their phone or laptop, add your site during sign-up, and provide the current code shown on the screen during sign-in. During sign-up your server provides a seed value, and the algorithm generates time-based codes from that seed that rotate every 30 seconds or so. Since both your server and the authenticator app know the initial seed value, both can generate the same codes, so your server knows which code should be provided at any given time. But if an attacker learns that seed value, they too can generate valid codes.
- **hardware key:** A physical USB device like a [Yubikey](https://www.yubico.com/) can provide either time-based codes like an authenticator app, or act like a passkey described above (preferred). Since it plugs into a USB port, and is relatively small, account holders can carry the key with them and plug it into any device they happen to be using. Newer keys also work with mobile devices that support Near Field Communication (NFC), which is the same technology used for contactless payments.

These extra credentials are typically prompted for after successfully authenticating with a password or passkey, acting as a "second factor" that increases your confidence that the person signing in is the actual account holder. An attacker would need to not only compromise the account holder's password or passkey, but also steal their physical hardware key in order to break into the account.

## Password Hashing

If you do collect password credentials from your account holders, _never store those passwords in plain text_! Sadly, [several major sites in the early 2000s did just this](https://www.reddit.com/r/netsec/comments/hy1n9/154_websites_that_store_your_password_in_plain/), and after they were hacked, millions of passwords were leaked on the Internet.

Instead, you should always hash passwords using an approved password-hashing algorithm, and only store the resulting hash in your database. As I discussed in the [cryptography](crypto.md#cryptographic-hashing) tutorial, hashing is a good way to store data that you don't need to reconstruct, but you do need to verify in the future. Because hashing algorithms are deterministic, the same password provided during a subsequent sign-in will hash to the same value as the one stored in your database. But because they are also irreversible, an attacker can't directly recompute the original password from a stolen hash.

I say "directly" because it is of course possible for an attacker to simply hash every known password and compare these pre-computed results to the stolen hash. This is why password hashing algorithms differ from ordinary hashing algorithms like SHA-256 in two important ways:

- They add a unique and unguessable 'salt' value to each password before hashing it.
- They are purposely designed to be relatively slow, though still fast enough to be used during sign-in.

The combination of these two qualities makes it impossible for attackers to use pre-computed hashes on a new batch of stolen password hashes, and prohibitively slow for those attackers to recompute hashes for all known passwords plus a given salt value. They can still do a brute-force attack given enough time, but the forced delay also gives you time to discover the breach, revoke all existing authenticated sessions, and force your customers to reset their passwords.

The currently recommended password hashing algorithm (in 2025) is [argon2](https://en.wikipedia.org/wiki/Argon2). Libraries are available for all major programming languages, and most are very easy to use. Typically the implementation will generate the salt value automatically, and return an encoded string containing the salt value as well as the hash. Store that in your database, and feed it back into the verify method to verify a password provided during sign-in.

## Session IDs and Tokens


## Session Token Transmission



## Other Kinds of Authorization Tokens


