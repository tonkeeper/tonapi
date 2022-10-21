# TonAPI Auth

## Registering your API key
At the moment you need special key to be able to use TonAPI, othervise you requests will be limited.

To obtain special key wich we call **serverSideKey** or **clientSideKey** in this doc - you need to use telegram bot https://t.me/tonapi_bot

Bot support two commands: **/get_server_key** and **/get_client_key**.

Tonapi can be used both from client side as well as from server side. From code perspective there is not much of a difference, but its important to not use **serverSideKey** anywhere in client side, and at the same time to use **clientSideKey** only on client side. The reason for this is because client side key has additional limitations per IP, while serverside key can be banned in case of large amount of flood requests to the api, so its usage should be limited by the developer.

## Performing API requests
Once you have API key you can perform simple requests to the api.

One of the basic methods is **/v1/blockchain/getAccount**

So you can make an **GET** http request to the url
```
https://tonapi.io/v1/blockchain/getAccount?account=EQCD39VS5jcptHL8vMjEXrzGaRcCVYto7HUn4bpAOg8xqB2N
```
But as mentioned above there an **Authorization** header should be passed to access the method:
```
Bearer eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCJ9...
```
Here is an **javascript** code example to perform such request:
```javascript
fetch("https://tonapi.io/v1/blockchain/getAccount?" + new URLSearchParams({
    account: 'EQCD39VS5jcptHL8vMjEXrzGaRcCVYto7HUn4bpAOg8xqB2N',
}), {
    method: 'GET', 
    headers: new Headers({
        'Authorization': 'Bearer '+serverSideKey, 
    }), 
})
```
Also take a look at the same example but using **Go**:
```go
req, err := http.NewRequest("GET", "https://tonapi.io/v1/blockchain/getAccount", nil)
if err != nil {
    log.Println(err)
    os.Exit(1)
}

q := req.URL.Query()
q.Add("account", "EQCD39VS5jcptHL8vMjEXrzGaRcCVYto7HUn4bpAOg8xqB2N")
req.URL.RawQuery = q.Encode()

req.Header.Add("Authorization", "Bearer "+serverSideKey)

// Send req using http Client
client := &http.Client{}
resp, err := client.Do(req)
if err != nil {
    log.Println("Error on response.\n[ERROR] -", err)
}
defer resp.Body.Close()
```

## Using an SDK
To make things simpler for developers we introduced an SDK: https://github.com/startfellows/tonapi-sdk-js

Also since tonapi is build with **swagger** you can generate SDK for any language you perfer. Please use swaggerfile available on this URL: https://tonapi.io/swagger/swagger.json

## TON authorization overview
### 1) Redirect user to the auth page
To perform auth user must be redirected to special page where auth can be checked
tonapi.io/login?{params}

**Params supported:**
* **redirect_url** – *[optional]* *string*, url where user will be redirected after successful auth
* **callback_url** – *[optional]* *string*, url which will be called from backend in after successfull auth
* **app_id** – *string*, identifier of the app. Name and icon of the app will be used on the authorization page. (not supported yet)


One of the params **redirect_url** or **callback_url** must be passed. Please note that **authToken** wich you will get after authorization flow is **ONE TIME USE**, **SHORT LIVING** token wich should be exchanged to persistent token serverside via tonapi.io/v1/oauth/getToken. Just receiving authToken is not a proof of successful user authorisation and can be possibly swapped or be stolen by attacker.

In case of success the **callback_url** or **redirect_url** will be triggered with following GET params added:
* **success** – boolean, true in case if auth was successfully performed (not supported yet)
* **auth_token** – *[optional]* *string*, one-time-use token  
* **error_code** – *[optional]* *string*, in case of success=false short text code of error (not supported yet)
* **error_text** – *[optional]* *string*, in case of success=false text human readable description of error (not supported yet)

**Examples:**
```javascript
{
    "success": true,
    "auth_token": "abcd..."
}
```
```javascript
{
    "success": false,
    "error_code": "auth_rejected",
    "error_text": "User canceled authorization"
}
```

### 2) Fetching persistant token via tonapi.io/v1/oauth/getToken method.
After successfully obtaining **auth_token** via process described below /auth method should be called from server side to check that the **auth_token** is valid.

Authorization header must be passed to this method the same way as any other methods in tonapi.io API. Token can be obtained with t.me/tonapi_bot telegram bot. [Learn more about serverside and clientside flows.](#serverside-and-clientside-flows)

Example header:
```
Authorisation: Bearer AppKeyHere
```
**Serverside auth header:**
```javascript
var options = {
    host: 'tonapi.io',
    path: '/v1/nft/getCollections',
    headers: {
        'Authorization': 'Bearer ' + serverSideKey,
    }
};
http.request(options, () => {}).end();
```

**Clientside auth header:**
```javascript
var options = {
    method: 'get', 
    headers: new Headers({
        'Authorization': 'Bearer '+clientSideKey, 
    }), 
}
fetch("https://tonapi.io/v1/nft/getCollections", options)
```

There are two types of AppKeys that can be generated by t.me/tonapi_bot, serverside key and clientside key.


Following POST params needed by this method:
* **auth_token**, *string*, the token wich was returned by the method below
* **rate_limit**, *number*, request per seconds
* **token_type**, *string [client, server]*, type of token which will be used to indicate the app

**Examples:**
```javascript
{
    "success": true,
    "user_token": "abcd...",
    "address": "EQrt...s7Ui",
    "pubkey": "Pub6...2k3y", // base64-encoded Ed25519 public key
    "signature": "Gt562...g5s8D=", // base64-encoded ed25519 signature
    "wallet_version": "v4R2", // supported values: "v3R1", "v3R2", "v4R1", "v4R2"
    "client_id": "abc"
}
```
```javascript
{
    "success": false,
    "error_code": "auth_rejected",
    "error_text": "User canceled authorization"
}
```

## Decentralised proof of ownership
It is possible to check proof of ownership, without fully relying in TONAPI. Here is the example of code needed to check signature and be sure that user have access to provided wallet.
https://github.com/tonkeeper/ton-connect/blob/main/tonconnect-server/src/TonConnectServerV1.ts#L36



## OAuth demo
Check out simple demo of Authroisation flow:

[View Demo](https://beta.stickerface.io/tonapi-oauth-demo/)

***
```javascript
Quick start guide:

1) git clone git@github.com:startfellows/tonapi-oauth-demo.git
2) cd tonapi-oauth-demo
3) yarn
4) yarn start
```

Look at the source code for more [details](https://github.com/startfellows/tonapi-oauth-demo/blob/master/src/App.tsx)