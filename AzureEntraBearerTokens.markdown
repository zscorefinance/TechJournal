
You have an application and you want Azure Entra to manage your application security - Both user authN/authZ (not covered in this post) and application authN (covered in this post).

Azure Entra needs to identify your app and act as the auth provider, can produce tokens for external apps on your behalf and help you verify the token sent over by the external app during API calls.

Application authentication meaning, your application exposes APIs to other apps and when those other apps consume APIs in your application, they need to obtain and send an `access_token` as **bearer token** which you then need to verify.

How do you set this up?

### Setting up application registration and client secrets

Log into the Azure portal and go to app registraion.

Setup your app registration! This app registration represents your application for everything security. 

It will show a tenant ID (your org) and a client ID (representing your app).

Once done...

Click `Overview` → Set a meaningful Application ID URI, in dev we have set it as `api://zzzz`.

`Certificates and Secrets` → Create a new client secret. This client secret should be unique for consuming applications.

That means, for each external application that wants to consume your API, you need to create a client secret.

### How to get a token (this process to be followed by the external application to get a token from Azure Entra)

Gather following info from your Azure Entra app registration both available on the Overview page of the app registration. 

```bash
tenant_id = *Directory (tenant) ID - Your org ID* 
client_id = *Application (client) ID - Your application ID as Entra understands it*
client_secret = *as obtained during creation*
scope = {Application ID URI}/.default
```

To get a token the external app must invoke the following API (Use Postman for example):

URL `POST https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token` 

Send request body as `x-www-form-urlencoded`:

```bash
grant_type=client_credentials
client_id=xxxx
client_secret=wwww
scope=api://zzzz/.default
```

Same way, by invoking this API, the external consumer will get an `access_token` (JWT).

Using the `access_token` as `Bearer cccc` in `Authorization` header, the external consumer will make API calls to your application.

### How do you verify the token

The token was generated from your app registration in Entra by the external app, and they sent the token in Authentication header as a bearer token when they invoked one of your APIs.

How do you verify? You need to obtain the public keys from Entra, using which you can verify the bearer token.

You need a middleware. For example in Node.js you can create a simple middleware like below:

```javascript
const axios = require('axios').default;
const jwt = require('jsonwebtoken');

const audience = 'api://zzzz'; // You set it earlier in Entra, this is the Application ID URI only
const tenantID = '....'; // Your org ID in Entra - Available on the overview page in app reg
const appID = '....'; // ID using which Entra identifies your app - Available on the overview page in app reg

const isTokenExpired = (epoch) => epoch*1000 < Date.now();

exports.checkAuthToken = async (req, res, next) => {
    try {
        // Extract token from the header
        const accessToken = ((req.headers['Authorization'] || req.headers['authorization']))?.split('Bearer ')[1];
        if (!accessToken) {
            res.status(401).send('Unauthorized');
        }

        // Decode the token
        const decoded = jwt.decode(accessToken, {complete: true});

        const kid = decoded.header.kid || undefined; // token tells which key-ID was used to sign it
        const alg = decoded.header.alg || undefined; // token tells which algorithm was used to sign it

        // Obtain public keys - Public keys will be always same (you can cache it also) - Will retrieve multiple keys
        const getPublicKeysEndpoint = `https://login.microsoftonline.com/${tenantID}/discovery/keys?appid=${appID}`;
        const pubKeysResponse = await axios.get(getPublicKeysEndpoint);
        const keys = pubKeysResponse.data.keys || [];

        // Identify the particular key among the public keys using the key-ID that you identified in the bearer token
        const publicKeysMap = {};
        keys.forEach(k => publicKeysMap[`${k.kid}`] = k.x5c[0] || undefined);

        const signingKey = publicKeysMap[`${kid}`];

        // Form well-structured signing key, and use this to verify the bearer token
        const formedSigningKey = `-----BEGIN CERTIFICATE-----\n${signingKey}\n-----END CERTIFICATE-----`;

        // Verify the token using the structured/well-formed key and algorithm
        const verified = jwt.verify(accessToken, formedSigningKey, { algorithms: [alg] })
        const exp = verified.exp || undefined;
        const audVerified = verified.aud || undefined;

        // Check for expiry
        if (exp && audVerified && !isTokenExpired(exp) && audVerified === audience) {
            return next();
        } else {
            res.status(401).send('Either token has expired, or wrong audience');
        }
    } catch (error) {
        logger.error('Error in checAuthToken: ', error);
        res.status(500).send('Internal Server Error, ' + error.message);
    }
}
```

**Steps**:

1. Middleware extracts the **bearer token** (JWT) from header
2. Decodes the JWT using jsonwebtoken library
3. Identifies key ID `kid` and algorithm `alg` from JWT header
4. Gets all public keys for this authentication server (authentication server is Entra/app registration set up for current environment)
`GET https://login.microsoftonline.com/{tenant_id}/discovery/keys?appid={client_id}`
5. May have multiple `kid`s pointing to multiple `x5c` keys
6. Find which `x5c` was can verify this token by matching with the `kid` obtained in **step #3**
7. Verify using well formed signed key and algorithm
8. Verify token is not expired and audience match (Audience match is not mandatory, added just as additional check)

**Note**:

This article shows how to establish application to application integration using `access_tokens` from Azure Entra. 
This method doesn't have any user credentials involved in this.

If you want to enstablish an auth flow that involves user credentials, then you need `id_tokens`. 
In order to get an `id_token` you need to develop a user auth flow like MSAL or OIDC that will help you get an `id_token`.
