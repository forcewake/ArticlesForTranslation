[Source](http://dotnetcodr.com/2013/03/18/external-authentication-with-claims-and-ws-federation-in-mvc4-net4-5-part-4-single-signout-and-single-signon/ "Permalink to External authentication with Claims and WS-Federation in MVC4 .NET4.5 Part 4: Single SignOut and Single SignOn")

# External authentication with Claims and WS-Federation in MVC4 .NET4.5 Part 4: Single SignOut and Single SignOn

In the [previous post][1] we left off with the shortcomings of the Logout function: we log out of the web application but the session is still alive on the STS. Would like to end all our sessions when we press Logoff on the main screen. Similarly if there are multiple web sites that share the same STS then we would prefer to log in once on the STS login page and then be able to use all those applications.

These techniques are called Single SignOn and Single SignOut.

We will build on the MVC4 application we have been working on in this series of blog posts.

**Single SignOn**

Single SignOn is really nothing else but an application of the widely used "Remember Me" function. Say you have two web applications, SiteA and SiteB that share the same STS. You start the day by logging in to SiteA and do some work there. You'll of course use the STS login page. The STS will establish a login session with the client. The browser will send the FedAuth cookie with all subsequent requests.

Then at some point you need to use SiteB. SiteB also needs authentication and redirects the user to the same STS. However, the STS will recognise the user's FedAuth cookie and will issue another token for SiteB without having to log in again.

Therefore we get Single SignOn accross all applications that use the same STS.

**Single SignOut**

This requires some more steps. It is necessary to clean the local session cookie but it's not enough. We need to inform the STS that we want to sign out, so please dear STS, end the auth session at your side too. The STS will need to clear its own session cookie so that the Single SignOn session is terminated. Optionally it can also tell the other relying parties that the user wants to sign out, but this is not mandatory. If you only clear the local session cookie of SiteA and the session at the STS, then when you go over to SiteB the STS will not recognise you any more as you killed the auth session. So you will need to sign in again.

If you want to end the auth sessions in all relying parties you have accessed before you click the Logoff button then the STS will need to keep track of all apps you have used since your first log in. Then when you log out of SiteA then the STS will 'contact' SiteB to end the auth session there too.

You'll recall from our discussion of the URL format for logging that it had the section '?wa=wsignin1.0′. There is a similar URL format for signing out: '?wa=wsignout1.0′. The client will send a GET request to the STS with this URL. The STS in turn will look up in its records which relying parties the user has logged in to since they first logged on. The STS will respond the client with a HTML that may look like this:



    <p>
        <img src="https://relyingparty1/?wa=wsignoutcleanup1.0" />
    </p>
    <p>
        <img src="https://relyingparty2/?wa=wsignoutcleanup1.0" />
    </p>


And possibly many possible solutions exist as to how to send the signoutcleanup message, the above HTML is not unique at all.

The main bit is the signoutcleanup message: <http://relyingparty1/?wa=wsignoutcleanup1.0>

The client browser will contact all the relying parties in the message and send them a signoutcleanup message.

To recap:

1. The client clear his/her local auth cookie
2. The client contacts the STS
3. The STS clears its own session cookie
4. STSs replies with a page similar to the above markup
5. The client uses that message to inform the other relying parties about the logoff event
6. The relying parties clear their own auth cookies

**Demo**

Open the MVC4 appluication we've been working on in this series. Run the application and press F12 to open the Developer Tools just as we did before in the Claims series. Select Start capturing on the Network tab. On the home page of the MVC4 application select the About link to trigger a sign in. This time check the Remember Me checkbox:

![STS remember me][2]

The STS will set two cookies: idsrvauth and wsfedsignout:

![Sts own cookies][3]

The idsrvauth cookie is the logon session with the STS itself. The wsfedsignout cookie is a tool for the STS to keep track of the relying parties the user has logged into.

The STS will issue a cookie to establish a logon session with the client. You can see the FedAuth cookie issued by the STS in Developer Tools:

![FedAuth cookie set by the STS][4]

Close the browser running the MVC4 app on localhost and re-run the application. We do this in order to simulate two things:

* The user wants to re-enter the same web app after closing it first
* The user is trying to enter another relying party which uses the same STS for authentication

Click the About link to trigger the authentication. You should see the following:

* You are redirected to the STS login page
* The login page never actually loads
* You are redirected to the About page automatically

This is the effect of the Remember Me checkbox. The STS recognises the previously established logon session with the user and does not require the login name and password again. The STS simply issues a new auth token.

This is the essence of Single SignOn and this is the solution for multiple relying parties on different web servers.

We now have to implement Single SignOut. Locate the LogOff action within AccountController.cs. Currently it may look similar to the following:



    [HttpPost]
            [ValidateAntiForgeryToken]
            public ActionResult LogOff()
            {
                FederatedAuthentication.WSFederationAuthenticationModule.SignOut();
                return RedirectToAction("Index", "Home");
            }


This will only end the auth session in the MVC4 web app but not at the STS. Test it for yourself: run the application, click on 'About' and after logging in click the Log off link in the top right hand corner. You have successfully logged out of the MVC4 app. However, click About again. You will be redirected to the MVC4 without having to log in. This may be a good thing in some cases, but a Log off should mean LOG OFF, right?

Let's update the LogOff action as follows:



    [HttpPost]
            [ValidateAntiForgeryToken]
            public ActionResult LogOff()
            {
                WSFederationAuthenticationModule authModule = FederatedAuthentication.WSFederationAuthenticationModule;

                //clear local cookie
                authModule.SignOut(false);

                //initiate federated sign out request to the STS
                SignOutRequestMessage signOutRequestMessage = new SignOutRequestMessage(new Uri(authModule.Issuer), authModule.Realm);
                String queryString = signOutRequestMessage.WriteQueryString();
                return new RedirectResult(queryString);
            }


We first get a reference to the WS Federation authentication module. Then we clear the local cookie using the SignOut method. Then we construct a signout request message. The constructor requires the Issuer Uri and the Realm which will be read from the web.config. The WriteQueryString() method will form a valid 'wsignout' URL which will be understood by the STS. It will 'know' that the user wants to log out. You can set a breakpoint at 'String queryString = signOutRequestMessage.WriteQueryString();' to see what the signout request looks like.

Run the application and click the About link as usual. Click the Log off link and code execution should stop at the breakpoint within the LogOff action. You can now inspect the queryString variable and see what a signout URI looks like. We are redirected to the STS and we're greeted with the following message:

![STS federated signout][5]

The signout URI will look similar to the following:

[https://andras1/idsrv/issue/wsfed?wa=wsignout1.0&wreply=http%3a%2f%2flocalhost%3a2533%2f][6]

You'll see the 'wsignout' command and the 'wreply' which provides a way to return to the calling application.

Inspect the HTML source of the SignOut page of the STS. You'll find the wssignoutcleanup command embedded in an iframe:

![WS signoutcleanup message in HTML response][7]

Click 'Return to the application" and the select the About link. You will see that you have to provide your username and password again on the STS homepage meaning that we successfully killed off the auth session at the STS as well. The session was ended at all relying parties in the STS records as well.

This finishes up this blog post. The next one which will be the last post in this series about Claims in .NET4.5 will discuss the following more advanced topics:

* Home realm discovery
* Federated Sign-In
* Resource STS and Federation Gateway

You can view the list of posts on Security and Cryptography [here][8].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/03/14/claims-based-authentication-in-net4-5-mvc4-with-c-external-authentication-with-ws-federation-part-3-various-advanced-topics/ "Claims-based authentication in .NET4.5 MVC4 with C#: External authentication with WS-Federation Part 3 Various advanced topics"
[2]: http://dotnetcodr.files.wordpress.com/2013/02/remebermeonsts.png?w=630
[3]: http://dotnetcodr.files.wordpress.com/2013/02/cookiessetbystsonsts.png?w=630&h=146
[4]: http://dotnetcodr.files.wordpress.com/2013/02/fedauthcookiests.png?w=630&h=195
[5]: http://dotnetcodr.files.wordpress.com/2013/02/stsfederatedsignout.png?w=630
[6]: https://andras1/idsrv/issue/wsfed?wa=wsignout1.0&wreply=http%3a%2f%2flocalhost%3a2533%2f
[7]: http://dotnetcodr.files.wordpress.com/2013/03/wssignoutcleanupcommand.png?w=630&h=116
[8]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
