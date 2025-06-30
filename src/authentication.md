# Authenticated Sessions

In the [HTTP](http.md) tutorial, I discussed how its [stateless](http.md#stateless-protocol) nature is a classic software tradeoff---it enables good things like horizontal scaling, but it also makes things like authenticated sessions more difficult to support. In this tutorial I will show you how to support authenticated sessions over HTTP, which is actually a specific case of a more general problem: creating and verifying secure authorization tokens.

## Sessions

Under the hood, HTTP is a stateless protocol, but that's not your typical user experience on the web. When you use social media, or shop online, or pay bills electronically, you sign-in once, and then perform multiple operations as that authenticated user. You don't have to provide your credentials every time you do something that interacts with the server.

This implies that you have some sort of authenticated session with the server, but if HTTP is stateless, how is that possible? If your sign-in request went to one downstream server, but your subsequent request went to a different server, how does that different server know you are authenticated, much less who you are?

Although we can't keep any state on the server side related to the network connection, we can pass something back and forth between the client and the servers. This value is effectively a key to session state stored in the servers' database or cache. The flow goes like this:

[![](https://mermaid.ink/img/pako:eNp1UktTwjAQ_iuZPVe00BcZh4tcPHtzegnJUjK2CSapigz_3S2lFFR6adPvtY_sQVqFwMHje4tG4lKLyommNIyerXBBS70VJjDJhGdPtUYT_oK-A1_QfaD7C6oOXIogVsJjD8u7xcJz5nVl7rRhnzpsHlfufiEdKgrQovY90RNRERFrlIEJKW1Ljqsdw0bouueo3mwAHUrr1CgniMrS6x27cD-mVWjQiYBk7722hj0vr1PFxyV21AijmA8kugoYnagjVGdRsG9oRqbkVJ3fWuNx7Pkf7jCedtWvpWuK3j4cBZ3yVsLY7A3COMyBQMPU6mqSA_Krz4vqIYIGHa1A0dXZd5QSwgYbLIHTp8K1aOtQQmkORBVtsC87I4EH12IEzrbVBvia9kCndqso5nTvzn8r13mf-GgUuqduu8CnRTKPgC7Wq7XNwKAj8D18AY-L6SRLilk2y9JslqQR7IDn-SSLszzP03kSp2kSHyL4PsofJkU8L9J5FsdZmpN3cfgBzMgCSQ?type=png)](https://mermaid.live/edit#pako:eNp1UktTwjAQ_iuZPVe00BcZh4tcPHtzegnJUjK2CSapigz_3S2lFFR6adPvtY_sQVqFwMHje4tG4lKLyommNIyerXBBS70VJjDJhGdPtUYT_oK-A1_QfaD7C6oOXIogVsJjD8u7xcJz5nVl7rRhnzpsHlfufiEdKgrQovY90RNRERFrlIEJKW1Ljqsdw0bouueo3mwAHUrr1CgniMrS6x27cD-mVWjQiYBk7722hj0vr1PFxyV21AijmA8kugoYnagjVGdRsG9oRqbkVJ3fWuNx7Pkf7jCedtWvpWuK3j4cBZ3yVsLY7A3COMyBQMPU6mqSA_Krz4vqIYIGHa1A0dXZd5QSwgYbLIHTp8K1aOtQQmkORBVtsC87I4EH12IEzrbVBvia9kCndqso5nTvzn8r13mf-GgUuqduu8CnRTKPgC7Wq7XNwKAj8D18AY-L6SRLilk2y9JslqQR7IDn-SSLszzP03kSp2kSHyL4PsofJkU8L9J5FsdZmpN3cfgBzMgCSQ)

I will explain each step in more detail below, but here is a brief description the steps in that diagram:

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

Let's dive more deeply into each of these steps.

## Credentials

### Name/Email and Password

### Passkeys


## Session IDs and Tokens


## Session Token Transmission



## Other Kinds of Authentication Tokens

