[Source](http://dotnetcodr.com/2014/02/03/introduction-to-oauth2-part-4-the-resource-owner-and-client-flow/ "Permalink to Introduction to OAuth2 part 5: the resource owner and client flow")

# Introduction to OAuth2 part 5: the resource owner and client flow

**Introduction**

The previous two posts introduced the code and implicit flows in OAuth2. We'll now look at the two remaining flows of the OAuth specifications:

* Resource owner flow
* Client flow

**Resource owner credentials flow**

This particular flow is mostly suited for trusted applications. The scenario is similar to what we saw before. A client software installed on a device needs access to a protected resource. We saw earlier how the user had to give consent to the application to collect some limited amount resources before the application could go on with its job.

In this flow this step is missing. It is the client application itself that collects the credentials from the user. Here the application is trusted so the user can enter the username and password without having to go through the consent screen step, where the consent screen comes from the login GUI of the auth server. The user must be able to trust the application and the manufacturer of the application to enter his or her username and password.

A good example is the Twitter app whose login page looks as follows on an iPhone:

![Twitter login iOs][1]

There must be sufficient trust towards an application like that. Twitter has a good brand name and good marketing so it's reasonable to trust an official application that they have endorsed. You can expect that your Twitter credentials won't be misused otherwise that trust will be broken.

The client app then sends a POST request directly to the auth server. The URI requesting the auth token may look similar to the following:



    /token?grant_type=password&amp;scope=the_protected_resource&user_name=username&password=password


The request header will also include a basic auth header with the credentials in order to authenticate with the auth server:

Authorization: Basic clientId:clientPassword

Upon successful authorisation the application receives the usual auth token we saw before, e.g.:

{
"access_token": "somecode",
"expires_in": "expiry time in seconds",
"token_type": "Bearer",
"refresh_token": "somecode"
}

A good practice is that a trusted application collects the password the very first time it is used and it is then forgotten after collecting the auth token from the auth server. The access and refresh tokens are saved on the device but not the password.

The application can then access the protected resource with the auth token:

GET /resourceUri

…with the following header:

Authorization: Bearer access_token

The token can be refreshed with the refresh token once it has expired.

**Client credentials flow**

This flow is most applicable in machine-to-machine or service-to-service communication. The client sends a POST request to the token endpoint of the auth server:

/token?grant_type=client_credentials&;scope=protected_resource

A Basic auth header will be inserted into the request:

Authorization: Basic (clientId:clientSecret)

The client authenticates itself without involving the resource owner in the flow. The resulting token won't be associated with any particular user. The client won't carry out actions on behalf of the user but in its own right.

Read the next part in this series [here][2].

You can view the list of posts on Security and Cryptography [here][3].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.files.wordpress.com/2013/12/twitterioslogin.png?w=630
[2]: http://dotnetcodr.com/2014/02/10/introduction-to-oauth2-part-6-issues/ "Introduction to OAuth2 part 6: issues"
[3]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
