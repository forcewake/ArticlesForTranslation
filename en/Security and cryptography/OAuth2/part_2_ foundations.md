[Source](http://dotnetcodr.com/2014/01/23/introduction-to-oauth2-part-2-foundations/ "Permalink to Introduction to OAuth2 part 2: foundations")

# Introduction to OAuth2 part 2: foundations

**Introduction**

OAuth and OAuth2 has become an important term in web and mobile security. According to the [OAuth homepage][1] it's "an open protocol that allows secure authorisation in a simple and standard method from web, mobile and desktop applications." About OAuth2 is says specifically that it is an "authorisation framework that enables a third-party application to obtain limited access to a HTTP service."

In this post we'll look at the foundations of OAuth2: some theory and background and the most important terminology. We'll continue the series with the different auth flows in OAuth2 in the following posts.

**OAuth2 basics**

The starting point is simple: you as a user would like to retrieve some data from an online resource. You may do so from a web browser, a mobile phone, a tablet or some other similar device with internet connection. In traditional enterprise software all elements of an authorisation system belonged to the same organisation: the client software, the backend resources, the separate authorisation server if any. If a user wanted to access your resources then they navigated to your homepage, logged on and performed what they were allowed to do depending on their access rights.

In the world of the web services, mobile devices and mobile applications we need to separate the parts a little. According to the OAuth specification there's a distinction between a "client" and a "resource owner". The distinction becomes important as nowadays we can have a multitude of applications consuming a resource that belongs to someone else. Building web services that expose some type of data is a huge thing in the world of software development. It is one of the drivers behind the popularity of Twitter, Facebook etc. Would you like to write your own little Facebook app? [Here][2] are the public endpoints with the SDKs, happy coding! In this case it's not unthinkable that the owner of the software is not the same – i.e. does not belong to the same trusted subsystem – as the owner of the resource that the software is using. In OAuth therefore the client is the software itself that accesses a certain resource. The resource owner will still be part of the trusted system but naturally such third-party tools may fall outside. Just because you write a Twitter-app your software will not become a trusted application.

A frequent analogy used to describe OAuth2 is [Valet parking][3] which is quite common in the US. You give the key of your car to a person who will park it for you. You are the resource owner, the car is the resource, the key is the password and the person who parks your car – the valet – is the third party system which is more or less trusted. You shouldn't of course provide your password to anyone like that thinking that the person won't do anything harmful. The most expensive cars often have two sets of keys – a "normal" key and a valet-parking key. The valet-parking key provides limited access to the car: you can drive it for a couple of miles, you cannot open the boot (trunk) etc. The target in OAuth2 is similar in the digital world: you as the resource owner have 2 sets of keys. You have the password which gives full access to a resource and another one which gives limited rights in a digital transaction.

There are 4 important components in the OAuth setup:

* Client: as mentioned above, this is the software accessing a protected resource. The client may be private or public and trusted or untrusted
* Resource owner: the person/organisation who owns the resource the software is trying to access
* Resource server: the machine where the resource is stored
* Authorisation server: the server that can issue the limited access keys

By default the resource server and the authorisation server must belong to the same trust system. The auth server knows about the protected resources and the resource server knows how to read the limited access key issued by the auth server.

In enterprise software the trusted software would very likely be part of the same trusted system. Nowadays we can have partially – or not at all – trusted clients built by millions of mobile app developers in the world. The organisation who owns the resource will probably not have any control over the quality of those third-party applications.

The client will need some sort of a limited access key to access the protected resource. This is the purpose of the authorisation server which can issue such tokens to registered clients. Using that key the client will be able to access the resource. This is very similar to the process of single sign-on we saw [here][4] where an external authorisation server provides a SAML token with some initial amount of claims to registered users. The SAML token can be then used to access the protected service. The main difference is that in the enterprise-like WS-Trust specification with the SAML tokens the client is implicitly trusted.

**Flows**

OAuth2 must work with a wide range of consuming applications so no wonder that there are various ways to get hold of the limited access token.

OAuth2 defines a number of different flows as to how to access a protected resource. We'll look at these in more detail later, this is just to give some idea on what's coming up:

1. Authorisation code flow: used in conjunction with web application clients
2. Implicit flow: used with native – user-agent based – clients
3. Resource owner password credential flow: is used with trusted clients
4. Client credential flow: used in client-to-service communication

**Useful resources**

We'll see very little code in this series. I'll instead concentrate on giving an introduction to this relatively new field of security. Also, unlike with claims, there's no native C# library with objects around OAuth2 and OpenID in .NET at the time of writing these posts. By "native" I mean something that is e.g. part of the System.Security namespace. So writing these classes on my own in my free time can be a challenge.

However, you don't need to start from scratch. Like we saw in the series on claims, [Thinktecture][5] has prepared a lot of work in this area as well. You can install their library from NuGet and use it on your .NET project:

![Thinktecture library in NuGet][6]

You can read about this project with examples on its [GitHub repository][7].

We briefly looked at the ThinkTecture identity server in the claims series. You can find its GitHub page [here][8]. Check out its Wiki page for extensive demos and how-tos.

We'll continue the series on a discussion of the different auth flows. Read the next instalment [here][9].

You can view the list of posts on Security and Cryptography [here][10].

### Like this:

Like Loading...

### _Related_

[1]: http://oauth.net/ "OAuth homepage"
[2]: https://developers.facebook.com/ "Facebook developer page"
[3]: http://en.wikipedia.org/wiki/Valet_parking "Valet Parking on Wikipedia"
[4]: http://dotnetcodr.com/2013/03/18/external-authentication-with-claims-and-ws-federation-in-mvc4-net4-5-part-4-single-signout-and-single-signon/ "External authentication with Claims and WS-Federation in MVC4 .NET4.5 Part 4: Single SignOut and Single SignOn"
[5]: http://www.thinktecture.com/ "ThinkTecture homepage"
[6]: http://dotnetcodr.files.wordpress.com/2013/12/thinktecture-library-in-nuget.png?w=630&h=300
[7]: https://github.com/thinktecture/Thinktecture.IdentityModel.45 "ThinkTecture identity model on GitHub"
[8]: https://github.com/thinktecture/Thinktecture.IdentityServer.v2 "ThinkTecture identity server on GitHub"
[9]: http://dotnetcodr.com/2014/01/27/introduction-to-oauth2-part-3-the-code-flow/ "Introduction to OAuth2 part 3: the code flow"
[10]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
