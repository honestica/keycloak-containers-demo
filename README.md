# RH SSO Example

This project contains a RH-SSO (Keycloak) client implementation for testing purposes.

## Create a client on RH SSO Admin Console

Create a client for the JS console by clicking on `clients` then `create`.

Fill in the following values:

* Client ID: `js-console`
* Click `Save`

On the next form fill in the following values:

* Valid Redirect URIs: `http://localhost:8000/*`
* Web Origins: `http://localhost:8000`

Go to the `Installation` tab, then download the `Keycloak OIDC JSON` configuration.


## Start JS Console application

The JS Console application provides a playground to play with tokens issued by Keycloak.

1. First replace the file `src/keycloak.json` by the one downloaded in the previous section.

2. Change in `src/index.html` (line 20) the location of `keycloak.js` script.
The base URL to use is the value of `auth-server-url` in `src/keycloak.json`.

3. Then run it with:

    ```
    docker-compose up
    ```


## Change login behaviors

You can edit some configuration in the file `src/index.html` and the RH SSO Admin Console in order to change some behaviors of the flow.

### Login options

You can customize some login options.
Example: **requiring additionnal scopes**.

Check [here](https://github.com/keycloak/keycloak-documentation/blob/master/securing_apps/topics/oidc/javascript-adapter.adoc#loginoptions) the list of options.

**How ?**

Edit the `login` object in `src/index.html`. The object looks like :
```
var login = {
    // scope: 'openid profile email offline_access'
}
```

### Enable auto-login

When accessing the web app, the login button has to be triggered in order to start login flow.
Enabling the auto-login configuration force user to authenticate before accessing the page.

**How ?**

Uncomment `onLoad: 'login-required'` in the `initOptions` of the `src/index.html` page.


### Require consent for the application

So far we've assumed the JS Console is an internal trusted application, but what if it's
a third party application? In that case we probably want the user to grant access to what the application wants to have
access to.

**How ?**

Open the RH SSO Admin Console.

Go to `Clients`, select `JS Console` and turn on `Consent Required`.

Go back to the [JS Console](http://localhost:8000) and click `Login` again. Now Keycloak will prompt the user to grant
access to the application.

## Keys and Signing Algorithms

By default Keycloak signs tokens with RS256, but we have support for other signing
algorithms. We also have support for a single realm to have multiple keys.

It's even possible to change signing algorithms and signing keys transparently without
any impact to users.

Let's first try to change the signing algorithm for the JS console client.

First let's see what algorithm is currently in use. Open the [JS Console](http://localhost:8000),
login, then click on `ID Token`. This will display a rather cryptic string, which is
the encoded token. Copy this value, making sure you select everything.

In a different tab open the [JWT validation extension](http://localhost:8080/auth/realms/demo/jwt).
This is a custom extension to Keycloak that allows decoding a token as well as verifying the
signature of the token.

What we're interested is in the header. The field `alg` will show what signing algorithm is
used to sign the token. It should show `RS256`.

Now let's change this to the new cool kid on the block, `ES256`.

Open the [Keycloak Admin Console](http://localhost:8080/auth/admin/) in a new tab. Make sure you
keep the JS Console open as we want to show how it gets new tokens without having to re-login.

Click on `Clients` and select the `js-console` client. Under `Fine Grained OpenID Connect Configuration`
switch `Access Token Signature Algorithm` and `ID Token Signature Algorithm` to `ES256`.

Now go back to the JS Console and click on `Refresh`. This will use the Refresh Token to
obtain new updated ID and Access tokens.

Click on `ID Token`, copy it and open the [JWT validation extension](http://localhost:8080/auth/realms/demo/jwt)
again. Notice that now the tokens are signed with `ES256` instead of `RS256`.

While you're looking at the `ID Token` take a note of the kid, try to remember the first few characters.
The `kid` refers to the keypair used to sign the token.

Go back to the [Keycloak Admin Console](http://localhost:8080/auth/admin/). Go to
`Realm Settings` then `Keys`. What we're going to do now is introduce a new set of active keys and
mark the previous keys as no longer the active keys.

Click on `Providers`. From the drop-down select `ecdsa-generated`. Set the priority to `200` and click
Save. As the priority is higher than the current active keys the new keys will be used next time
tokens are signed.

Now go back to the JS Console and clik on `Refresh`. Copy the token to the [JWT validation extension](http://localhost:8080/auth/realms/demo/jwt).
Notice that the `kid` has now changed.

What this does is provide a seamless way of changing signatures and keys. Currently logged-in users
will receive new tokens and cookies over time and after a while you can safely remove the old keys
without affecting any logged-in users.

## Sessions

Make sure you have the [JS Console](http://localhost:8000) open in a tab and you're logged-in.
Open the [Keycloak Admin Console](http://localhost:8080/auth/admin/) in another tab.

Find the user you are logged-in as and click on `Sessions`. Click on `Logout all session`.

Go back to the [JS Console](http://localhost:8000) and click `Refresh`. Notice how you are
now longer authenticated.

Not only can admins log out users, but users themselves can logout other sessions from the
account management console.
