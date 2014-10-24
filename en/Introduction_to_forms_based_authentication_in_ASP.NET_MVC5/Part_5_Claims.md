[Source](http://dotnetcodr.com/2014/07/07/introduction-to-forms-based-authentication-in-asp-net-mvc5-part-5-claims/ "Permalink to Introduction to forms based authentication in ASP.NET MVC5 Part 5: Claims")

# Introduction to forms based authentication in ASP.NET MVC5 Part 5: Claims

**Introduction**

Claims in authorisation have received a lot of attention recently. Claims are simply key-value pairs where the key describes the type of claim, such as "first name" and the value provides the value of that claim, e.g. "Elvis". Think of a passport which usually has a page with the photo and lots of claims: first name, last name, maiden name, expiry date etc. Those are all key-value pairs that describe the owner of the passport and provide reliable evidence that the person is really the one they are claiming to be.

I have a long series on claims on this blog. If you don't anything about them then I recommend you at least go through the basics starting [here][1]. That series takes up claims in MVC 4. MVC 5 brings a couple of new features as far as claims are concerned.

**Demo**

We'll be using the same demo application as before in this series so have it open in Visual Studio 2013.

As we said before authentication in MVC 5 is built using [Katana components][2] which are activated with extension methods on the incoming IAppBuilder object. These Katana components work independently of each other: e.g. you can turn on Google authentication at will as we saw in the [previous][3] part. At the same time you can even turn off the traditional cookie authentication, it won't affect the call to app.UseGoogleAuthentication().

When we turned on Google authentication the consent screen stated that the application, i.e. the demo web app, will have access to the user's email. That's in fact a type of claim where the key is "email" or most probably some more complete namespace. Where's that email? How can we read it?

Locate the method called ExternalLoginCallback in the AccountController. That method is fired when the user has successfully logged in using an external auth provider like Google or Facebook. The method will first need to extract the login information using…



    var loginInfo = await AuthenticationManager.GetExternalLoginInfoAsync();


This login information contains the tokens, expiry dates, claims etc. As the login is taken care of by OWIN, MVC will need to tap into the corresponding Katana component to read those values. This is an example of a piece of logic outside OWIN that needs to find out something from it. The AccountController will need this information to handle other scenarios such as signing out. The key object here is the AuthenticationManager.

The AuthenticationManager which is of type IAuthenticationManager is extracted using the following private property accessor in AccountController:



    private IAuthenticationManager AuthenticationManager
    {
                get
                {
                    return HttpContext.GetOwinContext().Authentication;
                }
    }


The interface lies in the Microsoft.Owin.Security namespace. The AuthenticationManager is retrieved from the current OWIN context. The GetOwinContext extension method provides a gateway into OWIN from ASP.NET. It allows you to retrieve the request environment dictionary we saw in [the series on OWIN][2]:



    HttpContext.GetOwinContext().Request.Environment;


I encourage you to type 'HttpContext.GetOwinContext().' in the editor and look through the available choices with IntelliSense. The Authentication property exists so that ASP.NET can read authentication related information from the OWIN context as all that is now handled by Katana components.

If you look further down in the code you'll see the SignInAsync method. The method body shows how to sign users in and out using an external cookie-based login:



    AuthenticationManager.SignOut(DefaultAuthenticationTypes.ExternalCookie);
    AuthenticationManager.SignIn(new AuthenticationProperties() { IsPersistent = isPersistent }, identity);


Now that we know a little bit about the authentication manager we can return to the ExternalLoginCallback method. Let's see what the GetExternalLoginInfoAsync method returns in terms of login information. Add a breakpoint within the method and start the application. Click the Log in link and sign in with Google. Code execution stops within the method. Inspect the contents of the "loginInfo" variable. It doesn't contain too much information: the user name, the provider name – Google – and a provider key. So there's nothing about any provider specific related claims such as the user's email address.

Note that the GetExternalLoginInfoAsync method only provides an object which includes properties common to all providers. The list is not too long as we've seen. However, the method cannot know in advance what the Google auth provider will provide in terms of claims. The list of available data will be different across providers. Some providers may provide a more generous list of claims than just an email: a URL to the user's image, the user's contacts, first and last names etc. Insert the following line of code above the call to GetExternalLoginInfoAsync:



    AuthenticateResult authenticateResult = await AuthenticationManager.AuthenticateAsync(DefaultAuthenticationTypes.ExternalCookie);


Leave the breakpoint as it is and restart the application. Sign in with Google and inspect the authenticateResult variable. You'll see that it provides a lot more information than the login info above. The claims are found
within the Identity property:

![Claims from Google authentication][4]

You can see that the claim types are identified in the standard URI way we saw in the series on claims. You can query the Claims collection to see if it includes the email claim. If your application explicitly requires the email address of the user then make sure to indicate it when you set it up with Google or any other auth provider.

You can save the email of the user in at least two ways:

* Temporarily in the session using the ExternalLoginConfirmationViewModel object further down in the ExternalLoginCallback method. That view model doesn't by default include any property for emails, you'll need to extend it
* In the database using the UserManager object we saw before in this series

Let's see how we can achieve these. Locate the ExternalLoginConfirmationViewModel object in AccountViewModels.cs and extend it as follows:



    public class ExternalLoginConfirmationViewModel
    {
            [Required]
            [Display(Name = "User name")]
            public string UserName { get; set; }

    	public string EmailFromProvider { get; set; }
    }


Add the following method to AccountController.cs to read the email claim from the claims list:



    private string ExtractEmailFromClaims(AuthenticateResult authenticateResult)
    {
    	string email = string.Empty;
    	IEnumerable<Claim> claims = authenticateResult.Identity.Claims;
    	Claim emailClaim = (from c in claims where c.Type == ClaimTypes.Email select c).FirstOrDefault();
    	if (emailClaim != null)
    	{
    		email = emailClaim.Value;
    	}
    	return email;
    }


You can read this value in the body of ExternalLoginCallback and store it in a ExternalLoginConfirmationViewModel like this:



    public async Task<ActionResult> ExternalLoginCallback(string returnUrl)
    {
    	AuthenticateResult authenticateResult = await AuthenticationManager.AuthenticateAsync(DefaultAuthenticationTypes.ExternalCookie);

            var loginInfo = await AuthenticationManager.GetExternalLoginInfoAsync();
             if (loginInfo == null)
                {
                    return RedirectToAction("Login");
                }

                // Sign in the user with this external login provider if the user already has a login
                var user = await UserManager.FindAsync(loginInfo.Login);

    			string emailClaimFromAuthResult = ExtractEmailFromClaims(authenticateResult);

                if (user != null)
                {
                    await SignInAsync(user, isPersistent: false);
                    return RedirectToLocal(returnUrl);
                }
                else
                {
                    // If the user does not have an account, then prompt the user to create an account
                    ViewBag.ReturnUrl = returnUrl;
                    ViewBag.LoginProvider = loginInfo.Login.LoginProvider;
                    return View("ExternalLoginConfirmation", new ExternalLoginConfirmationViewModel
    					{ UserName = loginInfo.DefaultUserName, EmailFromProvider = emailClaimFromAuthResult });
         }
    }


This page redirects to a View called ExternalLoginConfirmation when the user first signs up in the "else" clause. Locate ExternalLoginConfirmation.cshtml in the Views/Account folder. You can use the incoming model to view the extracted email claim:



    <p>
        We've found the following email from the login provider: @Model.EmailFromProvider
    </p>


I've deleted all rows in the AspNetUsers table so that I can view this extra information. Also, clear the cookies in your browser otherwise Google will remember you. You can also run the application from a brand new browser window. The email was successfully retrieved:

![Found Google email claim][5]

We can store the email claim in the database within ExternalLoginCallback as follows:



    if (user != null)
    {
            await SignInAsync(user, isPersistent: false);
            IList<Claim> userClaimsInDatabase = UserManager.GetClaims<ApplicationUser>(user.Id);
    	Claim emailClaim = (from c in userClaimsInDatabase where c.Type == ClaimTypes.Email select c).FirstOrDefault();
    	if (emailClaim == null)
    	{
            	IdentityResult identityResult = UserManager.AddClaim<ApplicationUser>(user.Id, new Claim(ClaimTypes.Email, emailClaimFromAuthResult));
    	}

            return RedirectToLocal(returnUrl);
    }


First we check the claims stored in the database using the UserManager.GetClaims method. Then we check if the email Claim is present. If not then we add it to the database. The identityResult helps you check the result of the operation through the Errors and Succeeded properties.

The claims by default end up in the AspNetUserClaims table:

![Email claim saved in the aspnetuserclaims table][6]

You can of course use the AspNetUserClaims table to store any kind of claim you can think of: standard claims found in the ClaimTypes list or your own custom ones, such as <http://mycompany.com/claims/customer-type>.

You can view the list of posts on Security and Cryptography [here][7].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/02/11/introduction-to-claims-based-security-in-net4-5-with-c-part-1/ "Introduction to Claims based security in .NET4.5 with C# Part 1: the absolute basics"
[2]: http://dotnetcodr.com/2014/04/14/owin-and-katana-part-1-the-basics/ "OWIN and Katana part 1: the basics"
[3]: http://dotnetcodr.com/2014/07/03/introduction-to-forms-based-authentication-in-asp-net-mvc5-part-4/ "Introduction to forms based authentication in ASP.NET MVC5 Part 4"
[4]: http://dotnetcodr.files.wordpress.com/2014/05/claims-from-google-authentication.png?w=630&h=217
[5]: http://dotnetcodr.files.wordpress.com/2014/05/found-google-email-claim.png?w=630&h=52
[6]: http://dotnetcodr.files.wordpress.com/2014/05/email-claim-saved-in-the-aspnetuserclaims-table.png?w=300&h=39
[7]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
