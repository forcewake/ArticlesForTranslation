[Source](http://dotnetcodr.com/2014/02/06/introduction-to-oauth2-part-6-openid-connect-basics/ "Permalink to Introduction to OAuth2 part 7: OpenID Connect basics")

# Introduction to OAuth2 part 7: OpenID Connect basics

**Introduction**

OpenID is a specification that is built on OAuth2. OpenID reuses ideas from the OAuth2 specs like the query strings and token formats, but it specialises in authentication. There are a couple of issues with OAuth2 authentication that OpenID Connect aims to resolve.

**OAuth2 authentication issues**

According to the specifications OAuth2 is meant for authorisation and not authentication. If you followed along the posts in this series then you know how OAuth2 works, how the client gets a token with limited rights from an auth server, so it's an example of delegated authorisation using the consent screens. The 3rd party application will then have access to a limited set of features: read but not write tweets on Twitter, view but not add contacts on Facebook etc. Authentication is only the first part of the picture when you fill in your username and password on the consent screen. The auth server then hands out the limited set of rights which forms the authorisation part of the chain.

However, there are cases where the application doesn't need those rights at all. Instead all it needs is to have an authenticated user. Think of the following scenarios:

* The app doesn't have any dedicated user store with the username and password safely tucked away – it is convenient to rely on a trusted service to provide authentication
* The app needs to know who the user is to make adjustments to the UI – if you log on with Twitter then show Twitter-related links etc.
* The app needs to limit access to certain features depending on the auth provider, where the features are not directly linked to the type of provider – e.g. give free access to a photo viewer for Twitter users but make it a paid feature for Facebook users, i.e. the photo-viewing feature is unrelated to Twitter or Facebook

If you see the well known "Sign in with Facebook/Twitter/Google/etc" where the application itself has nothing to do with those providers then it can be a sign that those applications use OAuth2 in a way that goes against the specifications.

Another example is using Azure Mobile Services where you can limit the CRUD operations in your backend SQL tables to authorised users. The mobile app can use the most common auth providers to authenticate the user and get hold of an auth ticket, it is built into the Azure SDK. The application however may or may not use the limited features provided by the token and what the user has given his/her consent to.

If you recall from the previous parts then the client may contact the auth server with a URL similar to this in the [implicit flow][1]:



    GET /authorize?client_id=mymobileapp&redirect_uri=http://www.myapp.com&scope=signin&response_type=token&state=123


The application wants to authenticate the user and thus contacts the auth server with some special scope. The scope is called "signin" which can be an agreed-upon value so that the auth server knows that this is only about authentication and not authorisation. The application is asking for a User Info Endpoint which holds the user's name, email address etc. If the app manages to get hold of this endpoint then it can safely assume that the user is authenticated and the auth server replies with an access token.

Consider the following problem: you have a malicious app which requires you to sign in with one of the well known auth providers. People usually trust those buttons that say "Log in with Facebook/Twitter/etc" and they happily provide their username and password in the consent screen. The malicious application will receive an auth token to your User Info Endpoint. The app now has access to a token – in fact it steals the token – that can be used to impersonate you in another – legitimate – application. The malicious app swaps some access token in the legitimate application with your token. Your token has been used to impersonate you in another application that you might not even know of. The legitimate application only sees a trusted User Info Endpoint token from a trusted auth provider and it has no way to determine whether it was the user who has just typed in their credentials in a consent screen.

**Resources**

You'll find more detailed accounts on the problems with OAuth2 authentication under the following links:

**OpenID connect**

As stated above [OpenID Connect][2] is built on top of OAuth2 and aims to rectify its shortcomings regarding authentication. When the biggest auth providers saw the issues with OAuth2 being used as a purely authentication platform then they started to add specific steps to recheck the User Info Endpoint token. This led to compatibility issues between them as these actions were not co-ordinated. OpenID Connect brings these provider-specific platforms to the same level so that the developers don't need to tweak their code depending on the auth provider.

OpenID uses two code flows that we saw before: Authorization code flow and Implicit flow. There's a new token type though called the ID Token. Actually, the concept of User Info Endpoint arose when the OpenID Connect specs were formalised. The specs also add other things that are not found at all in the OAuth2 framework, such as session management, i.e. how to log out users.

**Authorization code flow with OpenID connect**

The following actors are the same as we saw before in OAuth2:

* The client app, i.e. the user agent
* The identity provider, such as Twitter or Google
* The auth server with the auth endpoint
* The token endpoint which provides the auth token
* The identity provider: Google, Twitter, Facebook etc.

The client first sends a GET request to the auth endpoint. The request may take the following form:



    /authorize?client_id=mymobileapp&redirect_uri=https://mymobileapp/cb&scope=openid%20profile&response_type=code&state=123


This is basically the same as before – keep in mind that most of the OAuth2 specs are reused in OpenID. The major "invention" here is that OpenID related requests must be identified in the scope parameter where its value will start with "openid". After the "openid" string you can attach other specifiers such as "profile", "email" etc. to further specify what claims the application needs. "Profile" provides what it implies: first and last name, email address and the like. The following values are applicable along with the claims they provide:

* profile: name, family name, given name, middle name, nickname, preferred username, picture, website, gender, birthdate, timezone info, locale, the date when the profile was updated
* email: email and verified email
* address: address
* phone: phone number and verified phone number
* offline_access: this is a request for a refresh token

The next step is the consent screen provided by the appropriate Identity provider. The user types in their username and password as usual. The consent screen may be specific for the purposes of authentication. Instead of saying "this app will have access to your tweets" it may state that "this app will have access to your Twitter profile" which is an even more limited set of rights compared to the pure OAuth2 variant. If you click "yes" then the auth code is sent back to the client:



    GET /the_callback_url?code=abc&state=123


The code is used to retrieve the token from the token endpoint using a POST request:

POST /token

…with the auth header Authorization: Basic client_id:client_pw and also sends the following parameters:



    grant_type=authorization_code&authorization_code=abc&redirect_uri=https://mymobileapp/cb


The token endpoint responds with a JSON similar to this one:

{
"access_token" : "abc"
, "id_token": "xyz"
, "expires_in": "3600"
, "token_type": "Bearer"
, "refresh_token" : "mno"
}

…which is very similar to the token we saw in the OAuth2 flow. The new element is the ID token which was missing from the OAuth2 token. The ID token contains a set of JWT formatted claims about the authentication event:

* Issuer ('iss'): where the token has originated from, i.e. if it comes from the same ID provider the user has just consulted
* Subject ('sub'): identifies the user
* Audience ('aud'): identifies which client the token is for. It must be the same client that initiated the request. The client can use this claim to verify if the token was meant for itself. This feature was missing from the OAuth2 type of log in
* Expiration ('exp'): how long the token is valid

The ID token is signed and it must be validated by the client. Upon validation success the application will know that the response from the token endpoint was meant for it.

This is the value added by OpenID on top of OAuth2 to provide authentication. We no longer accept just any token and assume that authentication must have succeeded. With the ID token we can validate the token on the client side. The above scenario with the impersonation cannot happen as the legitimate app can validate the incoming token.

If the username is all the application needs to know to function then we're basically done. The username is available in the subject claim. Otherwise yet another GET request is made from the client to the User Info Endpoint:

GET /userinfo

…with the auth header Authorization: Bearer access_token

The access token is used to retrieve more info about the user from the User Info Endpoint. The User Info Endpoint sends back the claims in JSON format similar to this one:

{
"sub" : "56346552345"
, "name": "John Smith"
, "email": "john@smith.com"
}

You can view the list of posts on Security and Cryptography [here][3].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/01/30/introduction-to-oauth2-part-4-the-implicit-flow/ "Introduction to OAuth2 part 4: the implicit flow"
[2]: http://openid.net/connect/ "OpenID Connect homepage"
[3]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
