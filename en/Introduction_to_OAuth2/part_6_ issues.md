[Source](http://dotnetcodr.com/2014/02/10/introduction-to-oauth2-part-6-issues/ "Permalink to Introduction to OAuth2 part 6: issues")

# Introduction to OAuth2 part 6: issues

**Introduction**

OAuth2 is abound in the world of mobile applications, but it has drawn some criticism as well regarding its usefulness and security. A lot of it originates from [Eran Hammer][1] who originally led the specification committee of OAuth2.

**Specification too wide**

The OAuth2 specs started out as a protocol called Web Authorization Protocol. A protocol is a strict set of rules and definitions. All organisations that adhere to a protocol will be interoperable with each other. As OAuth2 was gaining ground more and more companies wanted to take part in the discussions and have their say. As you may certainly know as the number of participants around a topic grows the more difficult it will be to reach an agreement with a concise set of rules. Instead all parties enforced their own little modifications and company-specific enhancements and alternatives. After a while OAuth2 could not be regarded as a protocol anymore as it became too generic.

At that point the protocol was renamed the OAuth2 Authorization Framework. In effect it was a framework of protocol subsets. You can make a quick search in the specification document [here][2]. Search for the words 'may' and 'should'. At the time of writing this post they appeared 69 and 75 times respectively: 'The client should avoid', 'authorization server may establish' etc. Those verbs only denote the possibility of an action, not that it is obligatory, i.e. 'must' be implemented. The specification document simply lacks a clear guidance as to how to implement OAuth2 properly. The implementation will depend on the specific scenario you're working on: the type of the device, type of auth server, absence/presence of a dedicated auth server, communication types etc. So in practice you will have a number of different implementations of the OAuth2 Framework where each implementation can be regarded as an OAuth2 protocol specific to a certain environment.

**Bearer tokens**

You've seen the token type 'bearer' in a number of posts of this series. Why is it called 'bearer' anyway? It denotes any party who is in possession of the token, i.e. it is the bearer of the token. Anyone who holds the token can use it the same way as any other party also in possession of the token can. It is not required that a bearer produce any – cryptographic – proof that it is the legitimate holder of the token. We saw in [this][3] post how this "feature" can be used to hijack a token and impersonate you in another, unsuspecting application.

If you've gone through the series on [Claims and SAML tokens][4] then you'll recall that a SAML token must have a signature to verify its authenticity. Therefore a stolen SAML token cannot be used to make other requests on your behalf.

The lack of signatures in bearer tokens means that the only way to protect the token is to use SSL. Programming against SSL might not be the favourite way to spend time for software developers as SSL is riddled with certificate and other validation errors. So they may look for ways to simply [ignore SSL errors][5]. This can lead to some half-implemented SSL where the validation errors are ignored: a man-in-the-middle can reroute your HTTP request and attach a false certificate.

Also, SSL may not be implemented in the entire communication chain. It can be decrypted to perform caching and optimisation at the backend server where someone within the receiving company's intranet can access your token, password, name, email etc.

A more secure version of OAuth2 tokens are MAC tokens. You can read about them here:

**Security**

The consent screens of companies like Facebook, Google or Twitter can give the everyday user a false sense of security. They are all well-established and generally trusted companies so a web view with the Google logo must be extra safe and reliable, right? An unknown third party application can easily become a trusted app if the user sees that it's possible to log in with Twitter and Facebook. However, there's nothing stopping the developer of the app to create a consent screen identical to that of the well-known auth providers. This technique is often used in spoofing attacks to steal your bank account credentials: instead of <http://www.mytrustedbank.com/login.aspx> you're directed to <http://www.mytrustedbank1.com/login.aspx> and the reader may never see the '1' in the URL or give it any importance. The same technique can be used to spoof the consent screens.

However, there's still a good point in having those consent screens. The developer doesn't need to worry about storing passwords and usernames securely and the client can still click 'refuse' if something looks fishy.

Let's look at something different: issues with the authorisation endpoint URL.

Recall the following sections in the GET request to the authorisation endpoint of the auth server:



    redirect_uri=http://www.mysite/callback&scope=resource&response_type=code


As these sections are part of the URI an attacker can catch your request and start manipulating the string. The redirect_uri can be modified to let the auth server send the token to someone else's server and get hold of a token meant for you. The scope parameter can be modified to grant access to resources – even extended resources – to the attacker. The response type can also be changed from 'code' to 'token' to get hold of the token immediately.

This is a point to take care of if you're planning to build your own OAuth2 authorisation endpoint. You must make sure that the incoming request is validated thoroughly. One way to mitigate the risks is that when the client signs up with your auth server then you register the list of valid redirect URI:s as well and try to limit that list to just one URI. The token then cannot be sent to any arbitrary callback address. The same is true for the other elements in the query string: register all valid scopes and resources for all new clients.

If you manage to hold the range of possible values in the query string to 1 per client then your auth server can even ignore the incoming values in the URI and instead use the registered ones in the database.

Among the resources below you'll find a couple of very specific examples on how the Facebook OAuth2 implementation was successfully hacked.

**Resources**

Here's a couple of articles from Eran Hammmer criticising OAuth2. You may or may not agree with him but you can assume that he is extremely knowledgeable on this topic. Read through the articles and make sure you understand his arguments:

Here are a couple of documents around OAuth2 that were not included in the Framework document mentioned above. If you wish to do any serious work in OAuth2 then you should be familiar with these as well:

There's a web page which lists a whole range of specs around OAuth2, like JWT tokens, authorisation grants, dynamic client registration, and a whole range of things that could easily fill this blog with posts for the rest of the year: [Web Authorization Protocol (oauth)][6]

An excellent overview: [How to Think About OAuth][7]

Regarding security around OAuth2 and the possibilities for attacks you can consult the following pages:

Read the next part in this series [here][8].

You can view the list of posts on Security and Cryptography [here][9].

### Like this:

Like Loading...

### _Related_

[1]: http://hueniverse.com/author/eran/ "About Eran Hammer"
[2]: http://tools.ietf.org/html/rfc6749 "OAuth2 specs PDF"
[3]: http://dotnetcodr.com/2014/02/03/introduction-to-oauth2-part-4-the-resource-owner-and-client-flow/ "Introduction to OAuth2 part 5: the resource owner and client flow"
[4]: http://dotnetcodr.com/2013/02/11/introduction-to-claims-based-security-in-net4-5-with-c-part-1/ "Introduction to Claims based security in .NET4.5 with C# Part 1: the absolute basics"
[5]: https://www.google.se/search?q=how+to+handle+ssl+validation+error&oq=how+to+handle+ssl+validation+error&aqs=chrome..69i57.6727j0j7&sourceid=chrome&espv=210&es_sm=93&ie=UTF-8#q=how%20to%20ignore%20ssl%20certificate%20error%20in%20c%23 "Ignore SSL errors"
[6]: http://datatracker.ietf.org/wg/oauth/ "Web Authorization Protocol"
[7]: http://www.tbray.org/ongoing/When/201x/2013/01/23/OAuth "How to Think About OAuth"
[8]: http://dotnetcodr.com/2014/02/06/introduction-to-oauth2-part-6-openid-connect-basics/ "Introduction to OAuth2 part 7: OpenID Connect basics"
[9]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
