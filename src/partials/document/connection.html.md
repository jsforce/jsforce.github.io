---
---

## Connection

### Username and Password Login

When you have a Salesforce org username and password (and maybe security token, if required),
you can use `Connection.login(username, password)` to establish a connection to the org.

By default, it uses the SOAP API login endpoint (so, no OAuth2 client information is required).

```javascript
const jsforce = require('jsforce');

const conn = new jsforce.Connection({
  // you can change loginUrl to connect to sandbox or prerelease env.
  // loginUrl : 'https://test.salesforce.com'
});

const userInfo = await conn.login(username, password)

// logged in user property
console.log("User ID: " + userInfo.id);
console.log("Org ID: " + userInfo.organizationId);
```

### Username and Password Login (OAuth2 Resource Owner Password Credential)

When OAuth2 client information is given, `Connection.login(username, password + security_token)` uses OAuth2 Resource Owner Password Credential flow to login to Salesforce.

```javascript
const jsforce = require('jsforce');

const conn = new jsforce.Connection({
  oauth2 : {
    // you can change loginUrl to connect to sandbox or prerelease env.
    // loginUrl : 'https://test.salesforce.com',
    clientId : '<your Salesforce OAuth2 client ID is here>',
    clientSecret : '<your Salesforce OAuth2 client secret is here>',
    redirectUri : '<callback URI is here>'
  }
});

const userInfo = await conn.login(username, password)

// logged in user property
console.log("User ID: " + userInfo.id);
console.log("Org ID: " + userInfo.organizationId);
```

> [!WARNING]
> If you create your org in Summer ’23 or later, the OAuth 2.0 username-password flow is blocked by default as described in this [article](https://help.salesforce.com/s/articleView?id=release-notes.rn_security_username-password_flow_blocked_by_default.htm&release=244&type=5). The username-password flow presents security risks. We recommend using the OAuth 2.0 client credentials flow instead.

### Session ID

If you have a Salesforce session ID and its server URL information use it to authenticate.


```javascript
const jsforce = require('jsforce');

const conn = new jsforce.Connection({
  instanceUrl : '<your Salesforce server URL (e.g. https://na1.salesforce.com) is here>',
  serverUrl : '<your Salesforce server URL (e.g. https://na1.salesforce.com) is here>',
  sessionId : '<your Salesforce session ID is here>'
});
```

### Access Token

After the login API call or OAuth2 authorization, you can get the Salesforce access token and its instance URL.
Next time you can use them to establish a connection.

```javascript
const jsforce = require('jsforce');

const conn = new jsforce.Connection({
  instanceUrl : '<your Salesforce server URL (e.g. https://na1.salesforce.com) is here>',
  accessToken : '<your Salesforrce OAuth2 access token is here>'
});
```

### Access Token with Refresh Token

If a refresh token is provided in the constructor, the connection will automatically refresh the access token when it has expired.

<b>NOTE</b>: Refresh token is only available for OAuth2 authorization code flow.

```javascript
const jsforce = require('jsforce');

const conn = new jsforce.Connection({
  oauth2 : {
    clientId : '<your Salesforce OAuth2 client ID is here>',
    clientSecret : '<your Salesforce OAuth2 client secret is here>',
    redirectUri : '<your Salesforce OAuth2 redirect URI is here>'
  },
  instanceUrl : '<your Salesforce server URL (e.g. https://na1.salesforce.com) is here>',
  accessToken : '<your Salesforrce OAuth2 access token is here>',
  refreshToken : '<your Salesforce OAuth2 refresh token is here>'
});
conn.on("refresh", (accessToken, res) => {
  // Refresh event will be fired when renewed access token
  // to store it in your storage for next request
});

// Alternatively, you can manually request an updated access token by passing the refresh token to the `oauth2.refreshToken` method.
const res = await conn.oauth2.refreshToken(refreshToken)
console.log(res.access_token)
```

<b>NOTE</b>: You don't need to listen to the `refresh` event and grab the new access token manually, this is done automatically (see the `Session refresh handler` section).

### Session refresh handler

You can define a function that refreshes an expired access token:

```javascript
const jsforce = require('jsforce');

const conn = new jsforce.Connection({
  loginUrl : '<your Salesforce server URL (e.g. https://na1.salesforce.com) is here>',
  instanceUrl : '<your Salesforce server URL (e.g. https://na1.salesforce.com) is here>',
  refreshFn: async(conn, callback) => {
    try {
      // re-auth to get a new access token
      await conn.login(username, password);
      if (!conn.accessToken) {
        throw new Error('Access token not found after login');
      }

      console.log("Token refreshed")

      // 1st arg can be an `Error` or null if successful
      // 2nd arg should be the valid access token
      callback(null, conn.accessToken);
    } catch (err) {
      if (err instanceof Error) {
        callback(err);
      } else {
        throw err;
      }
    }
  }
});
```

The refresh function will be executed whenever the API returns a `401`, the new access token passed in the callback will be set in the
`Connection` instance and the request will be re-issued.

`Connection.login` sets the same `refreshFn` function above in the example.

You can use this feature to handle session refresh in different OAuth methods like JWT or Client Credentials.


### Logout

Call `Connection.logout()` to logout from the server and invalidate current session.
It is valid for both SOAP API based sessions and OAuth2 based sessions.

```javascript
const jsforce = require('jsforce');

const conn = new jsforce.Connection({
  sessionId : '<session id to logout>',
  serverUrl : '<your Salesforce Server url to logout>'
});

await conn.logout()
```

