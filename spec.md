# AUTH47 AUTHENTICATION PROTOCOL

## ABSTRACT

Auth47 is an authentication protocol inspired from [BitId](https://github.com/bitid/bitid/blob/master/BIP_draft.md) and from [SLIP013](https://github.com/satoshilabs/slips/blob/master/slip-0013.md).

The goal of Auth47 is:
- to provide a mechanism allowing simple and secure authentication with a Bitcoin Payment Code (see [BIP47](https://github.com/bitcoin/bips/blob/master/bip-0047.mediawiki))
- to support the authentication to different types of resources (Websites, Soroban)
- to support multiple protocols for the callback URIs (HTTP (default), [Soroban Protocol](https://code.samourai.io/wallet/samourai-soroban))
- to support ephemeral anonymous authentication tokens with loose Bitcoin addresses (in a future version) 

This document defines the first version of the protocol.

## MENU

- [Workflow](#workflow)
- [Auth47 URI](#auth47-uri)
  - [ABNF grammar](#abnf-grammar)
  - [Parameters](#parameters)
  - [Examples](#examples)
- [Signature](#signature)
  - [Preparation of the Auth47 challenge](#preparation-of-the-auth47-challenge)
  - [Signature of the Auth47 challenge](#signature-of-the-auth47-challenge)
- [Response](#response)
  - [Auth47 Response payload](#auth47-response-payload)
  - [Response over HTTP](#response-over-http)
  - [Response over Soroban Protocol](#response-over-soroban)
- [Response validation](#response-validation)
- [Future extensions](#future-extensions)


## WORKFLOW

The typical workflow for the authentication to a website with Auth47 follows this steps

- Server generates an Auth47 challenge that is displayed to the user as a QRCode
- Alice opens her Auth47-compatible Bitcoin wallet and scans the QRCode
- Wallet detects that the QRCode encodes an Auth47 challenge. It validates that the challenge is well-formed and displays a popup asking to confirm the authentication to the website with the BIP47 payment code managed by the wallet.
- Alice confirms that she want to authenticate with her payment code
- Wallet signs the challenge with the private key associated to the notification address of the payment code
- Wallet posts the response to the callback URI defined in the challenge. Response is composed of the challenge, the signature and the payment code
- When it receives the response, Server checks that all components of the response are well-formed and valid. If everything is fine, Server notifies Alice that she's authenticated with her payment code. 

## AUTH47 URI

### ABNF grammar

```
auth47urn = "auth47://" nonce "?" callbackparam [ othersparams ]
nonce = *( ALPHA / DIGIT )
callbackparam = "c=" callbackuri
callbackuri = httpuri / srbnuri
httpuri = httpscheme "://" host [ ":" port ] path-abempty 
httpscheme = "http" / "https"
srbnuri = srbnscheme "://" srbnchannel [ "@" srbninstanceuri ]
srbnscheme = "srbn" / "srnbs"
srbnchannel = 16HEXDIG
srbninstanceuri = host [ ":" port ] path-abempty 
othersparams = "&" otherparam [ othersparams ]
otherparam = [ eparam / rparam ]
eparam = "e=" *DIGIT
rparam = "r=" resourceuri
resourceuri = "srbn" / httpuri
```

Notes:
- host, port, path-abempty follow the definition given in [RFC3986](https://tools.ietf.org/html/rfc3986)
- The callback URI musn't have a Query or a Fragment component.

### Parameters

Supported parameters are
- c (mandatory): Callback URI that must be used to post the response to the challenge.
- e (optional): Expiration date/hour of the authentication. Expressed as a UNIX timestamp (GMT timezone) 
- r (optional): URI of the resource covered by the authentication. It can be a HTTP URI if the resource is a website or `srbn` if the resource is Soroban. If this parameter isn't set explicitely in the auth47 URI, the following rules apply:
  - if `c` is set with an HTTP URI then the value of `r` is implicitely the value of `c`
  - if `c` is set with a Soroban URI then the value of `r` is implicitely `srbn`

### Examples

#### Valid URI

```
# An auth47 URI which should be returned over HTTPS
auth47://aftE53gsSDFZDFQcserezfsdfvx422?c=https://samouraiwallet.com/callback

# An auth47 URI which should be returned over HTTPS on a specific port (446)
auth47://aftE53gsSDFZDFQcserezfsdfvx422?c=https://samouraiwallet.com:446/callback

# An auth47 URI which should be returned to an onion address over HTTP with expiry date/hour set
auth47://aftE53gsSDFZDFQcserezfsdfvx422?c=http://samouraivlsuevop.onion/callback&e=1609277967

# An auth47 URI which should be returned to a Soroban channel
auth47://aftE53gsSDFZDFQcserezfsdfvx422?c=srbn://1ea24efcbb89a25e@sorobanivlsuevop.onion/callback

# An auth47 URI which should be returned to a Soroban channel (over HTTPS)
auth47://aftE53gsSDFZDFQcserezfsdfvx422?c=https://soroban.samouraiwallet.com/
```

#### Invalid URI

```
# An auth47 URI with an invalid character in the nonce
auth47://a#t22?c=https://samouraiwallet.com/callback

# An auth47 URI with an invalid scheme for the fallback UTI
auth47://azt22?c=ftp://samouraiwallet.com

# An auth47 URI with a query component in the callback URI
auth47://azt22?c=https://samouraiwallet.com/callback?tag=ohno

```

## SIGNATURE

### Preparation of the Auth47 challenge

The Auth47 challenge that will be signed is derived from the received Auth47 URI by following these steps:
- If the Auth47 URI doesn't have a 'r' parameter, add this parameter to the URI and initialize its value with:
  - the value of the `c` parameter if `c` is set with an HTTP URI
  - `srbn` if `c` is set with a Soroban URI
- Remove the `c` parameter from the Auth47 URI
- The result is the Auth47 challenge that will be signed

**Example - Authentication to a website**

```
# Auth47 URI
auth47://aftE...?c=https://samouraiwallet.com/callback

# Initialize the r parameter
auth47://aftE...?c=https://samouraiwallet.com/callback&r=https://samouraiwallet.com/callback

# Remove the c parameter
auth47://aftE...?r=https://samouraiwallet.com/callback

# The Auth47 challenge that will be signed is auth47://aftE...?r=https://samouraiwallet.com/callback
```

**Example - Authentication to Soroban**

```
# Auth47 URI
auth47://aftE...?c=srbn://1ea...@sorobanivlsuevop.onion

# Initialize the r parameter
auth47://aftE...?c=srbn://1ea...@sorobanivlsuevop.onion&r=srbn

# Remove the c parameter
auth47://aftE...?r=srbn

# The Auth47 challenge that will be signed is auth47://aftE...?r=srbn
```

### Signature of the Auth47 challenge

The Auth47 challenge is signed with the common format `\x18Bitcoin Signed Message:\n#{auth47_challenge.size.chr}#{auth47_challenge}`

The challenge is signed with the private key associated to the notification address of the payment code.


## RESPONSE

### Auth47 Response payload

The payload of the response is composed of the following attributes:
- auth47_response (mandatory): The version of the Auth47 protocol. Should be set to `1.0`
- challenge (mandatory): The Auth47 challenge signed by the wallet
- signature (mandatory): The signature of the challenge with the private key associated to the first payment code of the wallet or with a loose address
- nym (optional): The payment code used for the authentication encoded in Base58Check form. Required if authentication is made with a payment code
- address (optional): The Bitcoin address used for the authentication. Required is authentication is made with a loose address

### Response over HTTP

The response must be sent as a POST HTTP request to the callback URI encoded in the `c` parameter of the challenge.

the request must have the `content-type` header set to `application/json`

The response payload must be json encoded and must be sent in the body of the request

**Example**

```
# Auth47 URI
auth47://aftE53gsSDFZDFQcserezfsdfvx422?c=https://samouraiwallet.com/callback

# Auth47 challenge
auth47://aftE53gsSDFZDFQcserezfsdfvx422?r=https://samouraiwallet.com/callback

# Response
POST https://samouraiwallet.com/callback
content-type: application/json

{
  "auth47_response": "1.0",
  "challenge": "auth47://aftE53gsSDFZDFQcserezfsdfvx422?r=https://samouraiwallet.com/callback",
  "signature":"...",
  "nym":"PM8T..."
}
```

### Response over Soroban

The response must be sent as a JSON RPC 2.0 request using the HTTP/HTTPS protocols for its transport.
- If the `srbn` scheme is encoded in the callback URI, response should be sent over HTTP
- If the `srbns` scheme is encoded in the callback URI, response should be sent over HTTPS

The request should have the `content-type` header set to `application/json`

The payload sent to Soroban must have the following structure:
- jsonrpc (mandatory): "2.0"
- id (mandatory): id of the JSON RPC request sent by the wallet to the server
- method (mandatory): Soroban method. Should be set to `directory.Add`
- params (mandatory): Array of Soroban messages. Composed of a single message with the following structure:
  - Name (mandatory): Soroban channel on which will be published the message
  - Entry (mandatory): Auth47 response payload (JSON encoded)
  - Mode (mandatory): Publication mode. Should be set to `short`

The payload sent to Soroban must be json encoded and must be sent in the body of the request

**Example**

```
# Auth47 URI
auth47://aftE53gsSDFZDFQcserezfsdfvx422?c=srbn://1ea24efcbb89a25e@sorobanivlsuevop.onion

# Auth47 challenge
auth47://aftE53gsSDFZDFQcserezfsdfvx422?r=srbn

# Response
POST http://sorobanivlsuevop.onion
content-type: application/json

{
  "jsonrpc": "2.0",
  "id: 0,
  "method": "directory.Add",
  "params": [{
    "Name": "1ea24efcbb89a25e",
    "Entry": "{\"auth47_response\": \"1.0\",\"challenge\": \"auth47://aftE53g...", \"signature\":\"...\",\"nym\":\"PM8T...\"}",
    "Mode": "short"
  }]
}

```

## RESPONSE VALIDATION

The verifier of an Auth47 response enforces the following controls:
- The challenge extracted from the response must be a well-formed Auth47 challenge
- The nonce extracted from the challenge must be a valid nonce (depends on the use case)
- The resource extracted from the challenge must be a resource expected by the verifier (depends on the use case)
- If the challenge contains the expiry parameter `e` then the value of this parameter must be greater than the UNIX timestamp of the current date/hour (GMT timezone)
- The payment code extracted from the response is a valid BIP47 payment code 
- The signature extracted from the response is a valid ECDSA signature
- The signature is a valid signature of the challenge by the private key associated to the notification address of the payment code 

If all conditions are checked then the response is considered as validated and the payment code is considered as authenticated.
