[Source](http://dotnetcodr.com/2013/01/28/web-security-in-net4-5-mvc4-with-c-cross-site-request-forgery/ "Permalink to Web security in .NET4.5 MVC4 with C#: Cross site request forgery")

# Web security in .NET4.5 MVC4 with C#: Cross site request forgery

In this post we'll discuss what a cross-site request forgery means and how you can protect your MVC4 web application against such an attack. The term is abbreviated as CSRF – pronounced 'see-surf'.

A CSRF is an attack a malicious user can execute using the identity of an unsuspecting user who is logged onto the web application. The effects of the attack can range from annoyances, such as logging out the user from the site, to outright dangerous actions such as stealing data.

Imagine the following: a user with no administrative rights wants to do something on your site that only admins are allowed to do, such as create a new customer in the database. Let's say that your website is called <http://www.mysite.com>. This malicious user will then build a site which a user who is logged on to <http://www.mysite.com> will be tricked into using. Let's say that the malicious user builds a simple site called <http://www.fakesite.com>. The attacker will have to know at least a little bit about the HTML of the target site, but that's easy. He can simply select View Source in IE and similar commands in FF and Chrome and there it is.

Within <http://www.mysite.com> there will be a page where the user can enter the details of the new customer. The attacker will have to inspect the form element on that site, copy its raw HTML source and paste it to <http://www.fakesite.com>. He won't need to build a fully functioning web page: a single-page app will suffice. Let's say that this single page is called nicecars.html. Let's imagine that the attacker will actually place some pictures of good cars on this webpage so that it looks real.

Imagine that the original source HTML of the rendered page on <http://www.mysite.com> looks as follows:



    <form action="/Customers/Create" method="post">
        <input id="Name" name="Name" type="text" value="" />
        <input id="City" name="City" type="text" value="" />
        <input id="Country" name="Country" type="text" value="" />
    </form>


The attacker will now take this HTML, paste it in <http://www.fakesite.com/nicecars.html> and insert his own values as follows:



    <form action="/Customers/Create" method="post">
        <input id="Name" name="Name" type="text" value="You have been tricked" />
        <input id="City" name="City" type="text" value="Hello" />
        <input id="Country" name="Country" type="text" value="Neverland" />
    </form>


The attacker is aiming to insert these values into the database through an authenticated administrator on <http://www.mysite.com>. In reality these values may be more dangerous such as transferring money from one account to another.

The attacker will also dress up the form a little bit:



    <form id="fakeForm" style="display: none;" action="http://www.mysite.com/Customers/Create" method="post">
        <input id="Name" name="Name" type="text" value="You have been tricked" />
        <input id="City" name="City" type="text" value="Hello" />
        <input id="Country" name="Country" type="text" value="Neverland" />
    </form>


Note the inline style element: the idea is not to show this form to the administrator. The form must stay invisible on nicecars.html. It's enough to build a form that has the same structure that <http://www.mysite.com> expects. Also, as <http://www.fakesite.com> has nothing to do with Customers, the attacker will not want to post back to his own fake website. Instead he will want to post the data against <http://www.mysite.com>, hence the updated 'action' attribute of the form tag.

You may be wondering how this form will be submitted as there is no submit button. The good news for the attacker is that there's no need for a button. The form can be submitted using a little JavaScript right underneath the form:



    <script>
        setTimeout(function () { window.fakeForm.submit(); }, 2000);
    </script>


This code will submit the form after 2 seconds.

If the attacker will try to run his own malicious form then he will be redirected to the Login page of <http://www.mysite.com> as only admins are allowed to add customers. However, he has no admin rights, so what can he do? He can copy the link to the webpage with the malicious form, i.e. <http://www.fakesite.com/nicecars.html>, and send it to some user with admin rights on <http://www.mysite.com>. Examples: insert a link in an email, on Twitter, on Facebook, etc., and then hope that the user will click the link thinking they will see some very nice cars. If the admin actually clicks the link then the damage is done: the values in the malicious form of <http://www.fakesite.com/nicecars.html> will be posted to <http://www.mysite.com/Customers/Create> and as the user has admin rights, the new Customer object will be inserted into the database.

The attacker can make it even less obvious that <http://www.mysite.com> is involved in any way: the form can be submitted with Ajax. The attacker uses the administrative user as a proxy to submit his own malicious data.

This will work seamlessly if the administrative user is logged onto <http://www.mysite.com> when they click the link to <http://www.fakesite.com>. Why? When the admin is logged on then the browser will send the authentication cookie .ASPXAUTH along with every subsequent request – even to <http://www.fakesite.com>. So <http://www.fakesite.com> will have access to the cookie and send it back to <http://www.mysite.com> when the malicious form is submitted.

It is clear that the usual authentication and authorisation techniques will not be sufficient against such an attack. You will also want to make sure that the form data that is sent to your application originated from your application and not from an external one. Fortunately this is easy to achieve in MVC4 through the AntiForgeryToken object.

You will need to place this token in two places.

1\. On the Controller action that accepts the form data. Example:



    [HttpPost]
    [ValidateAntiForgeryToken]
    public ActionResult Create(CustomerViewModel customerViewModel)


If you now try to enter a new customer by posting against the Create method you'll receive the following exception:

"The required anti-forgery token from field "_RequestVerificationToken" is not present."

The reason is that the form that posts against the Create method must have an anti-forgery token which can be verified. You can add this token using a simple Html helper in the .cshtml Razor view. This token will hold a cryptographically significant value. This value will be added to a cookie which the browser will carry along when the form is posted. The value in this cookie will be verified before the request is allowed to go through.

Even if a malicious attacker manages to have an admin click the link to <http://www.fakesite.com/nicecars.html> they will not be able to set the right cookie value. Even if the attacker knew the exact value of the verification token websites don't allow setting cookies for another website. This is how an anti-forgery token will prevent a CSRF attack.

2\. We set up the verification token on the form as well.

This is easy to do using a Html helper as follows:



    @using (Html.BeginForm())
    {
        @Html.AntiForgeryToken()
        <fieldset>

        </fieldset>
    }


With very little effort you can protect your website against a cross-site request forgery.

You can view the list of posts on Security and Cryptography [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
