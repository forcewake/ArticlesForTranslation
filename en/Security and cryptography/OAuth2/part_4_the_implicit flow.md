[Source](http://dotnetcodr.com/2014/01/30/introduction-to-oauth2-part-4-the-implicit-flow/ "Permalink to Introduction to OAuth2 part 4: the implicit flow")

# Introduction to OAuth2 part 4: the implicit flow

**Introduction**

We looked at the code flow of OAuth2 in the [previous][1] part of this series. We'll continue by looking at the so-called **implicit flow**.

The implicit flow is mostly used for clients that run locally on a device, such as an app written for iOS or Windows 8. It is also applicable for packaged software that runs within a user agent on a client device. These act like local clients but they are not written as true native software.

We have to solve the same problem as before: the local application wants to access a protected resource on behalf of the resource owner who is not willing to provide the password. There are a lot of applications that use your data from Twitter, Facebook, LinkedIn, blog platforms etc. The majority of them do not originate from the respective companies but were written by third-party app developers. You as a Facebook user may not be happy to provide your password to those apps due to security concerns.

The solution is similar to what we saw in the case of the code flow: you won't give your password to the native application but to the auth server. The client app will then retrieve an access token from the auth server which it can use to access the resource.

You'll see that the implicit flow is really nothing else but a simplified version of the code flow.

**Details**

First the app will need to contact the auth server of the service the client app is using as the authentication platform. You've probably seen options like "sign in with your facebook/google/twitter/etc. account" in some application on iPhone/Android/WindowsPhone. The application must be able to show the login page of the service. The GET request sent to the auth server looks very much like the one we saw previously:



    GET /authorize?client_id=mygreatwindows8app&scope=resource&redirect_uri=http://www.mysite.com/redirect&response_type=token&state=some_id


The only difference is response_type=token, the other elements should look familiar. The native application won't get an access code from the auth server but the token directly. Remember that in the code flow the requesting page had to do some background task where it asked for the token using the code it got from the auth server. This extra step is missing here in the implicit flow.

The native app must be prepared for showing the login page of the auth server. Example: Azure Mobile Services allows a mobile app to register with Microsoft, Facebook, Google and Twitter. So if you write an application and want to let Azure handle the backend service and notifications for you then you can register your app with those services and you can offer your users to log in with the account they wish. The following is the top part of the Azure sign-up page – the full page is too long, but you get the idea:

![OAuth signup page on Azure Mobile Services][2]

The application will then need to show the correct login page and consent screen of the selected provider. Here's an example for the Google login on iOS:

![Google login on iOS][3]

This login page can be either shown in a popup window or as an embedded screen within the app, that's only an implementation detail.

If you give your consent to share your data with the app then the auth server will respond with a GET request to the callback. The request will contain the token details as query parameters in the URL. Example:



    GET /redirect#access_token=some_code&expires_in=300&state=same_as_in_the_original_request


Note the '#', it's not a typo. We won't use a callback '?' but a callback hash in this case. The callback is **hash fragment encoded**. The idea with this is that the query parameters won't be sent to any server, they will be hidden and won't show up in the referrer URL. Everything after the hash is only meant for the immediate recipient, it won't be propagated further on. Otherwise the token could leak to a third party.

The token is anyhow exposed to the native app and the browser where the login page is shown. The client app won't authenticate with the auth server, unlike in the code flow, so usually refresh tokens are not an option. If the client app had to authenticate with the auth server then the client credentials would need to be stored on the device, which is not ideal from a security point of view. Keep in mind that the user won't have full control over the application, so saving the full credentials locally is normally not a good idea.

The application can now request the protected resource using the token. Usually the token is put into the auth header of the request to the resource server as type 'Bearer':

Authorization: Bearer 'the access token'

We said that refresh tokens are normally not provided in the standard way of implementing the flow. However, this doesn't mean that there are no workarounds out there. E.g. Google uses a hidden iframe with a cookie to avoid showing the consent screen again and again. The lifetime of the cookie will then be longer than that of the token and the user will have to log in again after 2-3 weeks.

Read the next part in this series [here][4].

You can view the list of posts on Security and Cryptography [here][5].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/01/27/introduction-to-oauth2-part-3-the-code-flow/ "Introduction to OAuth2 part 3: the code flow"
[2]: http://dotnetcodr.files.wordpress.com/2013/12/oauthsignupinazuremobileservices.png?w=630&h=399
[3]: http://dotnetcodr.files.wordpress.com/2013/12/googleioslogin.png?w=630
[4]: http://dotnetcodr.com/2014/02/03/introduction-to-oauth2-part-4-the-resource-owner-and-client-flow/ "Introduction to OAuth2 part 5: the resource owner and client flow"
[5]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
