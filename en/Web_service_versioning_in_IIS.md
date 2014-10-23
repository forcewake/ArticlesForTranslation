[Source](http://dotnetcodr.com/2013/12/12/web-service-versioning-in-iis/ "Permalink to Web service versioning in IIS")

# Web service versioning in IIS

**Introduction**

If you have an open web service that your customers can call then you must be prepared to have different versions of the web service available. Your customers may have scheduled calls to your web service so that they can build reports or whatever, without getting an exception. At times a new version of your web service will contain breaking changes. If you deploy that version then the clients of the web service won't be impressed even if you send out emails and prepare them for the change. They will have to follow you and it will consume their time.

Instead, you can build new versions of the web service without affecting the old version and the clients can update their systems if and when they wish. You can just inform them that there's a new version available and this and this call will require an extra parameter.

There are certainly various ways to solve this issue and including the version in the URI path is one of them

Example:

<http://www.mygreatservice.com/v1/products/order>
<http://www.mygreatservice.com/v2/products/order>
<http://www.mygreatservice.com/current/products/order>

…where 'current' always contains the latest version. Some clients may always want to use the most recent version of our service but they need to understand the risks.

You may first think that if this is some MVC-style routing, such as in Web API, you'll need to include the version in the routes as follows:

/{version}/{controller}/{params}

Don't worry, it's not necessary. The base routing will be as follows:

<http://www.mygreatservice.com/products/order>

The version can be inserted using IIS.

**How to do**

On the deployment server you can set up the folder structure as follows:

![IIS versioning folder structure][1]

Within each version folder you can have subfolders following the conventions at your company, e.g. Staging, Live and Backup.

In IIS you create a new website and specify the 'mygreatwebservice' as the physical path:

![Setting up web site in IIS][2]

So you set the top folder as the physical path and not any of the version folders. You'll see the the following under the Sites node in IIS:

![IIS before adjustment][3]

Right-click each folder under the site and select "Convert to Application" in the context menu. Press OK in the Add Application menu that appears. You should see the following picture after this step:

![IIS after converting to applications][4]

This is it actually. The versions can be added to the URL path after the domain name:

<http://www.service.com/version/…>;

Just make sure you deploy the right version in each version folder and IIS will direct the traffic to the correct subfolder based on the first section in the path.

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.files.wordpress.com/2013/11/versioningfolderstructure.png?w=630&h=156
[2]: http://dotnetcodr.files.wordpress.com/2013/11/websitesetup.png?w=630
[3]: http://dotnetcodr.files.wordpress.com/2013/11/iisbeforeadjustment.png?w=630
[4]: http://dotnetcodr.files.wordpress.com/2013/11/iisafteradjustment.png?w=630
