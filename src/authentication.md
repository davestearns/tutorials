# Authenticated Sessions

In the [HTTP](http.md) tutorial, I discussed how its [stateless](http.md#stateless-protocol) nature is a classic software tradeoff---it enables good things like horizontal scaling, but it also makes things like authenticated sessions more difficult to support. In this tutorial I will show you how to support authenticated sessions over HTTP, which is actually a specific case of a more general problem: secure authorization tokens.

> [!IMPORTANT]
> This tutorial assumes you have a basic understanding of cryptographic hashing and digital signatures. If you don't, please read the [intro to cryptography](crypto.md) tutorial first.

## Sessions

Under the hood, HTTP is a stateless protocol, but that's not your typical user experience on the web. When you use social media, or shop online, or pay bills electronically, you sign-in once, and then perform multiple operations as that authenticated user. You don't have to provide your credentials every time you do something that interacts with the server.

This implies that you have some sort of authenticated session with the server, but if HTTP is stateless, how is that possible? If your sign-in request went to one [downstream server](building-blocks.md#load-balancers-and-api-gateways), but your subsequent request went to a different server, how does that different server know who you are?

Although we can't keep any state on the server related to the network connection, we can pass something back and forth between the client and the servers. This value is effectively a key to session state stored in a server-side database or cache, encoded into a secure token. The flow goes like this:

[![](https://mermaid.ink/img/pako:eNp1Uj1z6jAQ_Cuaqw3PJtgYTYYmNKnTZdwI6TCaYMnRRxLC8N_fCXAIA3EjS7t7qz3dHqRVCBw8vkc0EpdatE50jWH09cIFLXUvTGCSCc-ethpNuAV9Al_QfaC7BVUClyKIlfB4guVosfCced2akTbsU4fN48r9W0iHigy02PoT0RNRceZQUBUpbRzc1ehU4nxIDGmduogIosvo9Y79qnn0aNGgEwGZR--1Nex5eeX16fQVeBQJo5gPpLpyuJSiIKh-RMG-oRmYRJUpge-t8XjJeoc8tCWuTs-RYtHqw1GQlPctfsf9gzA08QxfN3HQ3CYUMWys09843OS2Yu_0R-oBJbTRSYn3g0MGHbpOaEXTtk-cBsIGO2yA06_CtYjb0EBjDkQlX_uyMxJ4cBEzcDa2G-BrekTaxV6R43lUf05bl2qf-WgUuqc0GsCns1kGNIqv1nYDgbbA9_AFvJgX43xaTiZlXuTzuqyqDHbA63Jcl9NZnVcPVVUVeXnI4PtYIB_P5g_ErOq6mJQETQ__AfOuErk?type=png)](https://mermaid.live/edit#pako:eNp1Uj1z6jAQ_Cuaqw3PJtgYTYYmNKnTZdwI6TCaYMnRRxLC8N_fCXAIA3EjS7t7qz3dHqRVCBw8vkc0EpdatE50jWH09cIFLXUvTGCSCc-ethpNuAV9Al_QfaC7BVUClyKIlfB4guVosfCced2akTbsU4fN48r9W0iHigy02PoT0RNRceZQUBUpbRzc1ehU4nxIDGmduogIosvo9Y79qnn0aNGgEwGZR--1Nex5eeX16fQVeBQJo5gPpLpyuJSiIKh-RMG-oRmYRJUpge-t8XjJeoc8tCWuTs-RYtHqw1GQlPctfsf9gzA08QxfN3HQ3CYUMWys09843OS2Yu_0R-oBJbTRSYn3g0MGHbpOaEXTtk-cBsIGO2yA06_CtYjb0EBjDkQlX_uyMxJ4cBEzcDa2G-BrekTaxV6R43lUf05bl2qf-WgUuqc0GsCns1kGNIqv1nYDgbbA9_AFvJgX43xaTiZlXuTzuqyqDHbA63Jcl9NZnVcPVVUVeXnI4PtYIB_P5g_ErOq6mJQETQ__AfOuErk)

We will dig into the details of each of these things below, but here is a brief overview the steps in that diagram:

1. When you sign in, the client sends your credentials to the server.
1. The server loads the account record associated with those credentials from the database.
1. The server verifies the provided credentials against the account record.
1. The server generates a unique identifier for the session, known as the **session ID**. This should be a random value that is effectively impossible to guess given previous examples---e.g., a 128 or 256-bit value from a [cryptographically-secure pseudorandom number generator (CSPRNG)](https://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator).
1. The server inserts a Session record into the database (or cache) using the session ID as the primary key. The record also contains the ID of the authenticated account, and any other data you want to associate with the session.
1. The server generates a **digitally-signed session token** from the session ID, encodes it into an ASCII-safe text format (e.g., [base-64](https://en.wikipedia.org/wiki/Base64)).
1. The server includes this signed session token (not the bare session ID) in the response.
1. The client holds on to this value, and sends it back in a header in all subsequent HTTP requests.
1. When one of the servers receives the subsequent request, it extracts the session token from the request header and verifies the signature.
1. The server extracts the session ID from the token, and loads the associated Session record from the database (or cache). 
1. The server now knows who the authenticated account is, and can decide if the current request is something that account is allowed to do.

At a high level this is how authenticated sessions work over HTTP, but let's expand on each of these concepts in more detail.

## Accounts

To use your system, a customer (person, corporation, or agent) needs to create an **account**. First-time system designers will often assume that an account will always belong to just one person, and one person will create only one account, but that's hardly every true in the long run. Instead:

- A given customer will probably create multiple accounts. For example, it's common to create separate work and personal accounts if your system could be used for both purposes.
- A given account will probably be used by multiple people, especially if your service costs money. Nearly every media streaming service learned this the hard way.

You can avoid some of these issues by designing your accounts to be **hierarchical** from the start. For example, let customers create one main account to handle the service subscription and billing centrally, and then create child accounts for each of their family members, or for different roles, perhaps with restricted permissions. The same setup would be applicable for corporation with multiple subsidiaries, or a platform with multiple customers of their own.

Existing accounts should also be able to band together under a new parent account for centralized billing and monitoring---e.g., when two corporations merge, or when two people with individual accounts get married and want to take advantage of a family sharing discount.

The authorization rules for parent accounts will depend on your particular system. In some cases it will make sense to let parent accounts view all resources that belong to child accounts, but in others those child resources might need to remain private. In some cases it might also make sense for parent accounts to create new resources or adjust configuration on behalf of their child accounts, and in others this should not be allowed. Think about what would make sense in your scenario and move forward accordingly.

## Credentials

Regardless of your account structure, when a customer creates an account they must also provide some **credentials** they can use to prove their identity when signing in again in the future. These credentials are something the customer knows (password), something customer has (phone or hardware security key), or something customer is (biometrics tied to a passkey). Systems that manage particularly sensitive data might require multiple of these.

### Something You Know: Email and Password

Typically systems start out requiring only an email and password. Email names are already unique, and account holders can prove their ownership of that email address by sending them a verification email (more on that [below](#other-kinds-of-authorization-tokens)). Once verified, the email address can be used for notifications and account recovery. 

But **don't use an email address as the account's primary key** in your database---people sometimes need to change their email address, for example when leaving a school or corporation. Use an [application-assigned ID](ids.md) for the account, and store the email address as part of the account's credentials.

Passwords are again a classic design tradeoff. They are simple and familiar, and can be used anywhere the account holder happens to be, including shared computers or public kiosks. But they are also a shared secret, so if an attacker learns an account's password, they immediately gain unrestricted access. This would be OK if people were disciplined about creating unique and strong passwords on each site, but sadly most people are not. Even those who are can be tricked into revealing their passwords on phishing sites that look like the real sign-in pages of popular services.

### Something You Are: Passkeys

Thankfully there is now an alternative to passwords that is well supported by the major browsers and client operating systems: **passkeys**. These use [asymmetric cryptography](crypto.md#asymmetric-encryption) to securely authenticate without ever passing a shared secret like a password over the network.

Passkeys are relatively simple to understand in principle, but their actual implementation can get complex, so you should definitely leverage the [official libraries](https://passkeys.dev/docs/tools-libraries/libraries/) for your client and server programming languages. To help you understand how they work, let's look at a simplified version of the sign-up flow:

[![](https://mermaid.ink/img/pako:eNp9kk2PmzAQhv-K5UNPkLIhhICqSBG57K0SyqXi4pgJsQq264920yj_veOQLNndtlxgmHfemXnsM-WqBVpSCz88SA5bwTrDhkYSfPbqhVS9AOnGWDPjBBeaSUd2hFmys2A-pqqQ2mjdC86cUPKjYnNVeHdE6yBSNxeQ7dS5BvPzb_b1tfjr8z8F2yDYMsf2zMI74yper-uSmLCudVhm7Xc4YdwJ68zDuHWMyioorVayJb-EOxKpEBFhGCqc3RClQ4EdK6YGm6mBU4QbYA7Cqw3rsn4UblC4K8leqAGcEZywicfrFLt4tMM9xUGAJeLq4U53j_uQzhtpifZ7hE5wo-ihH3neNvLN-hZ5_E88AlivtyjFw4FHZ_Lpy958XiNixrnyyPv9ZhO6CSqxnnOw9uD7G9L70VTxCMKKTsZeE64G3YPDLI3oAGZgosULeg7qhiKiARpa4mcLB-Z719BGXlCK-FR9kpyWzniIqFG-O9LywHqLkdctHsLtdr_-7UzwvumRCZgqbETLVRZRvEvflBrueQxpeaYvtHzKs9lysciz1SrPilWyzCN6wt9JOpsv07RIsiQr5sVidYno76tDMsuzdFHkaTLP0vlTsbz8AV7jITY?type=png)](https://mermaid.live/edit#pako:eNp9kk2PmzAQhv-K5UNPkLIhhICqSBG57K0SyqXi4pgJsQq264920yj_veOQLNndtlxgmHfemXnsM-WqBVpSCz88SA5bwTrDhkYSfPbqhVS9AOnGWDPjBBeaSUd2hFmys2A-pqqQ2mjdC86cUPKjYnNVeHdE6yBSNxeQ7dS5BvPzb_b1tfjr8z8F2yDYMsf2zMI74yper-uSmLCudVhm7Xc4YdwJ68zDuHWMyioorVayJb-EOxKpEBFhGCqc3RClQ4EdK6YGm6mBU4QbYA7Cqw3rsn4UblC4K8leqAGcEZywicfrFLt4tMM9xUGAJeLq4U53j_uQzhtpifZ7hE5wo-ihH3neNvLN-hZ5_E88AlivtyjFw4FHZ_Lpy958XiNixrnyyPv9ZhO6CSqxnnOw9uD7G9L70VTxCMKKTsZeE64G3YPDLI3oAGZgosULeg7qhiKiARpa4mcLB-Z719BGXlCK-FR9kpyWzniIqFG-O9LywHqLkdctHsLtdr_-7UzwvumRCZgqbETLVRZRvEvflBrueQxpeaYvtHzKs9lysciz1SrPilWyzCN6wt9JOpsv07RIsiQr5sVidYno76tDMsuzdFHkaTLP0vlTsbz8AV7jITY)

1. The client makes a request to the server to create a new passkey for the account.
1. The server responds with a unique and unguessable value, which is typically called a 'nonce'.
1. The client application asks its security hardware (known as the 'Authenticator') to create a new asymmetric key pair scoped to the server domain, encrypt the nonce with the private key, and return: the encrypted nonce; the associated public key; and a unique ID for the passkey to the application. This triggers the device's biometric sensors to authenticate the person using the device (e.g., touch or face ID).
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
- **hardware key:** A physical USB device like a [Yubikey](https://www.yubico.com/) can provide either time-based codes like an authenticator app, or act like a passkey described above (preferred). Since it plugs into a USB port, and is relatively small, account holders can carry the key with them and plug it into any device they happen to be using. Newer keys also work with mobile devices that support Near Field Communication (NFC), which is the same technology used for mobile payments.

These extra credentials are typically prompted for after successfully authenticating with a password or passkey, acting as a "second factor" that increases your confidence that the person signing in is the actual account holder. To break into an account, an attacker would need to not only compromise the account holder's password or passkey, but also steal their physical hardware key, which is tough to do when the attacker is actually located in another part of the world.

## Password Hashing

If you do collect password credentials from your account holders, _never store those passwords in plain text_! Sadly, [several major sites in the early 2000s did just this](https://www.reddit.com/r/netsec/comments/hy1n9/154_websites_that_store_your_password_in_plain/), and after they were hacked, millions of passwords were leaked on the Internet.

Instead, you should always hash passwords using an approved password-hashing algorithm, and only store the resulting hash in your database. As I discussed in the [cryptography](crypto.md#cryptographic-hashing) tutorial, hashing is a good way to store data that you don't need to reconstruct, but you do need to verify in the future. Because hashing algorithms are deterministic, the same password provided during a subsequent sign-in will hash to the same value as the one stored in your database. But because they are also irreversible, an attacker can't directly recompute the original password from a stolen hash.

I say "directly" because it is of course possible for an attacker to simply hash every known password and compare these pre-computed results to the stolen hash. This is why password hashing algorithms differ from ordinary hashing algorithms like SHA-256 in two important ways:

- They add a unique and unguessable 'salt' value to each password before hashing it.
- They are purposely designed to be relatively slow, though still fast enough to be used during sign-in.

The combination of these two qualities makes it impossible for attackers to use pre-computed hashes on a new batch of stolen password hashes, and prohibitively slow for those attackers to recompute hashes for all known passwords plus a given salt value. They can still do a brute-force attack given enough time, but the forced delay also gives you time to discover the breach, revoke all existing authenticated sessions, and force your customers to reset their passwords.

The currently recommended password hashing algorithm (in 2025) is [argon2](https://en.wikipedia.org/wiki/Argon2). Libraries are available for all major programming languages, and most are very easy to use. Typically the implementation will generate the salt value automatically, and return an encoded string containing the salt value as well as the hash. Store that in your database, and feed it back into the verify method to verify a password provided during sign-in.

In Python, it's a simple as this:

```python
from argon2 import PasswordHasher, VerificationError

hasher = PasswordHasher()

# During sign-up...
sign_up_password_hash = hasher.hash(sign_up_password)
# Store password_hash in your database

# During sign-in...
try:
	hasher.verify(sign_up_password_hash, sign_in_password)
except (VerificationError):
	# Password didn't match!
```

## Session IDs and Tokens

Regardless of which kinds of credentials you require, once the account holder is successfully authenticated, you need to return something to the client that the client can send back in all subsequent requests. But what should this value be? What qualities should it have?

- It should be unique and unguessable so an attacker can't guess another valid session ID given one of their own.
- It should be [digitally-signed](crypto.md#digital-signatures) using a secret key known only to the server so that even if an attacker can guess another valid session ID, they can't sign it because they don't know the secret signing key.

Your best bet for a session ID is a sufficiently long random value generated by a cryptographically secure pseudo-random number generator (CSPRNG). Given that you will be signing it, 128 bits is likely sufficient for current computing hardware, but you can increase that to 256 bits if you're paranoid. 

Generating a 128-bit cryptographically-random value in Python is as easy as this:

```python
import secrets

session_id = secrets.randbits(128)
```

Other programming languages offer similar functionality, so ask your favorite AI tool how to do it in your chosen language.

To digitally sign this value, use a symmetric digital signature algorithm like [HMAC](https://en.wikipedia.org/wiki/HMAC). This algorithm requires a key that you need to keep secret on the server, but since the server is the only thing signing and verifying, that is relatively easy to do. In Python, the code looks like this:

```python
import os
from hmac import HMAC, compare_digest

# Get the secret signing key from an env var,
# or some other secure mechanism like a secrets service.
secret_key = os.getenv("SESSION_TOKEN_SIGNING_KEY")

session_id_bytes = session_id.to_bytes(16), # 128 bits / 8 = 16 bytes

hmac = HMAC(
	key=secret_key, 
	msg=session_id_bytes,
	digestmod=hashlib.sha256, # use SHA-256 for hashing
)
signature = hmac.digest()
```

Again, you should be able to find an HMAC library for all major programming languages, so this isn't something exclusive to Python. Just paste this code into your favorite AI tool and ask it how to do this same thing in your desired programming language.

Now that you have the ID and a digital signature for that ID, you can combine them together into a single binary **token**. You can then encode that into an ASCII-safe format like [Base 64](https://en.wikipedia.org/wiki/Base64) to include in your HTTP response. In Python that looks like this:

```python
from base64 import urlsafe_b64decode

combined = signature + session_id_bytes
token = urlsafe_b64encode(combined).decode("ascii")
```

When the client sends this token back to your server during subsequent requests, you can verify it using code like this:

```python
from base64 import urlsafe_b64decode
from hmac import HMAC, compare_digest

SIGNATURE_BYTES_LEN = 32 # SHA 256 bits / 8 = 32 bytes

# Decode the base-64 string back into bytes.
decoded = urlsafe_b64decode(token)

# Split the signature and session ID bytes.
signature = decoded[:SIGNATURE_BYTES_LEN]
session_id_bytes = decoded[SIGNATURE_BYTES_LEN:]

# Recalculate what the signature of session_id_bytes 
# should be using our secret key.
hmac = HMAC(
	key=secret_key, 
	msg=session_id_bytes,
	digestmod=hashlib.sha256, # use SHA-256 for hashing
)
expected_signature = hmac.digest()

# Compare them to make sure they are the same
token_is_valid = compare_digest(signature, expected_signature)
```

Note that we use `compare_digest()` here and not a simple `==` comparison. The former is constant-time, meaning it will take the same amount of time regardless of how similar or different the two signatures are. This prevents a sophisticated form of attack, known as a **timing attack**, where the attacker uses the request latency differences to detect how close a tampered token is to being valid. A simple `==` comparison will stop as soon as it encounters a byte on the left that is different from the corresponding byte on the right.

## Session Token Transmission

Now that you have a session token, the remaining question is how should we transmit it in the HTTP response to the client, and how should the client send it back in subsequent HTTP requests? There a few options:

- **Secure HTTP-Only Cookie (generally preferred):** HTTP already has a built-in mechanism for values that should be automatically sent back and forth between the client and a particular [origin](http.md#origin), known as 'cookies'. The server adds a `Set-Cookie` header to the response containing the session token and some cookie configuration options, the most important of which are [`Secure`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie#secure) and [`HttpOnly`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie#httponly). The former requires an HTTPS connection, and the latter stops client-side JavaScript running in a web browser from accessing the cookie value. If your API is used only by your own web UI served from the same origin, set [`SameSite=Strict`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie#samesitesamesite-value) as well. All web browsers and nearly all HTTP client libraries have built-in support for cookies.
- **Authorization Header:** HTTP also has another standard header named `Authorization` that can be used for things like session tokens. This is not automatically handled by clients, however, so your client code must look for this response header, store the value somewhere secure, and manually send it back in the `Authorization` header during subsequent requests.

Using cookies is a tradeoff: they are automatically handled so your client code doesn't need to do anything to support them, but because they are automatically sent, they can be abused by cross-site scripting in web pages, especially if your API can be used by any origin. For example, if one of your account holders has previously signed in, and is then tricked into loading a page from `evil.com`, that page can make JavaScript `fetch()` requests to your server's API, and the browser will automatically send along any cookies it has for your server's origin, including your authenticated session token! That code will also be able to read all responses, and forward them back to `evil.com`.

You can stop this behavior by setting `SameSite=Strict` on the cookie, which will cause the browser to send that cookie only when the JavaScript making the request came from the same site as your API. But this is only appropriate if your API is only be used by web applications served from the same origin. If you intend for your API to be callable by web applications served from _any_ origin, this setting will block cookies for those legitimate sites as well.

Note that this only applies to web applications running with a web browser. Native mobile applications are isolated and don't typically download and execute JavaScript from random origins. The HTTP library used by your native mobile app will likely handle cookies automatically (though you may need to explicitly enable this), and no other code running on the same device can get access to the stored cookies.

The `Authorization` header is not handled automatically, so it doesn't have this same cross-site scripting vulnerability that cookies do in a web browser. But it has its own, different vulnerability: your client-side JavaScript must store the session token somewhere, and in a web browser that means local storage.

Browser local storage is isolated by origin already, but if an attacker is able to _inject_ JavaScript into a page that was served by your origin, that JavaScript can then read the previously stored session token and use it to make authenticated requests.

This sort of injection is easier than you might think in a web application that displays content submitted by users (e.g., social media, product reviews, etc.). These sites must turn a previously-submitted comment into HTML that is rendered by the browser, and if the code for that isn't careful, it can end up rendering executable JavaScript. That script will be running within the same origin as the served page, so it can then read any values previously written to local storage by pages that came from the same origin.

The best solution depends on what sort of client applications you want to support:

- If you only support native mobile apps, use cookies with Secure, HttpOnly, and SameSite=Strict. 
- If you only support web apps served from the same origin as your API, use cookies with Secure, HttpOnly, and SameSite=Strict.
- If you only support web apps served from a specific set of origins that are different from your API origin, use cookies with Secure and HttpOnly. Also check the `Origin` request header to ensure it matches one of your allowed origins.
- If you want to support web apps from _any_ origin, the Authorization header might be safer, but apps will need to ensure they properly filter user-supplied content when rendering it into HTML.


## Other Kinds of Authorization Tokens

Session tokens are actually a specific form of a more general concept known as **authorization tokens**. These are tokens that, once verified, authorize access to something without requiring authentication. A verified session token actually authorizes access to the session state, which happens to contain the previously-authenticated account, but it's really just an authorization token in the end.

Authorization tokens, like session tokens, have the following qualities:

- They contain a unique, unguessable value.
- They also contain a digitally signature of that value, generated using a signing key known only to the server.
- They may be associated with an existing account in the database.
- They may be deleted/consumed after first use.

Other examples of authorization tokens include:

- **Email/Phone Verification:** Many systems will ask you to verify your email address or phone number by sending a message to that address/number with a link. The link contains an authorization token that was previously created, signed, and associated with your account. If the submitted token signature is valid, it proves you have control over that email account or phone number. These tokens are typically deleted immediately after they are used to prevent replay attacks.
- **Account Recovery:** If you forget your password or lose your passkey, most systems will send an email to the account's email address containing a link you can click to recover your account. Just like the verification scenario, that link will contain an authorization token. When you follow the link, the server will verify the token, and then let you reset your password or create a new passkey. These tokens are also typically deleted immediately after use.
- **Public Access Keys:** When you share a document with "anyone who has the link," or create a video conference meeting link, those contain an authorization token that lets anonymous users access the document or join the call. The server will verify the signature on the token, but since it's not associated with just one account, and can be used multiple times, the server won't delete the token until it expires (if ever).
- **Magic Sign-In Links:** Some low-risk systems will let account holders sign-in by requesting a magic link sent to their registered email address or phone number. Just as in the verification scenario, that link will contain a one-time-use authorization token. When the account holder follows the link, the server verifies the token, starts a new authenticated session, and deletes the token.

Since these kinds of authorization tokens are really the same thing as session tokens, you can use the same code to generate and verify them!

> [!NOTE]
> Many thanks to my friend Clinton Campbell, founder of [Quirktree](https://www.quirktree.com/), for patiently explaining security concepts and techniques to me over the years, and reviewing drafts of these tutorials.
