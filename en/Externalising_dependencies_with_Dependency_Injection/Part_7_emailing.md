[Source](http://dotnetcodr.com/2014/09/22/externalising-dependencies-with-dependency-injection-in-net-part-7-emailing/ "Permalink to Externalising dependencies with Dependency Injection in .NET part 7: emailing")

# Externalising dependencies with Dependency Injection in .NET part 7: emailing

**Introduction**

In the [previous][1] post of this series we looked at how to hide the implementation of file system operations. In this post we'll look at abstracting away emailing. Emails are often sent out in professional applications to users in specific scenarios: a new user signs up, the user places an order, the order is dispatched etc.

If you don't know who to send emails using .NET then check the out mini-series on [this][2] page.

The first couple of posts of this series went through the process of making hard dependencies to loosely coupled ones. I won't go through that again – you can refer back to the previous parts of this series to get an idea. You'll find a link below to view all posts of the series. [Another post on the Single Responsibility Principle][3] includes another example of breaking out an INotificationService that you can look at.

We'll extend the Infrastructure layer of our demo app we've been working on so far. So have it ready in Visual Studio and let's get to it.

**The interface**

As usual we'll need an abstraction to hide the concrete emailing logic. Emailing has potentially a considerable amount of parameters: from, to, subject, body, attachments, HTML contents, embedded resources, SMTP server and possibly many more. So the interface method(s) should accommodate all these variables somehow. One approach is to create overloads of the same method, like…



    void Send(string to, string from, string subject, string body, string smtpServer);
    void Send(string to, string from, string subject, string body, string smtpServer, bool isHtml);
    void Send(string to, string from, string subject, string body, string smtpServer, bool isHtml, List<string> attachments);
    .
    .
    .


…and so on including all combinations, e.g. a method with and without "isHtml". I think this is not an optimal and future-proof interface. It is probably better to give room to all those arguments in an object and use that object as the parameter to a single method in the interface.

Also, we need to read the response of the operation. Add a new folder call Email to the Infrastructure.Common library. Insert the following object into the folder:



    public class EmailSendingResult
    {
    	public bool EmailSentSuccessfully { get; set; }
    	public string EmailSendingFailureMessage { get; set; }
    }


The parameters for sending the email will be contained by an object called EmailArguments:



    public class EmailArguments
    {
    	private string _subject;
    	private string _message;
    	private string _to;
    	private string _from;
    	private string _smtpServer;
    	private bool _html;

    	public EmailArguments(string subject, string message, string to, string from, string smtpServer, bool html)
    	{
    		if (string.IsNullOrEmpty(subject))
    			throw new ArgumentNullException("Email subject");
    		if (string.IsNullOrEmpty(message))
    			throw new ArgumentNullException("Email message");
    		if (string.IsNullOrEmpty(to))
    			throw new ArgumentNullException("Email recipient");
    		if (string.IsNullOrEmpty(from))
    			throw new ArgumentNullException("Email sender");
    		if (string.IsNullOrEmpty(smtpServer))
    			throw new ArgumentNullException("Smtp server");
    		this._from = from;
    		this._message = message;
    		this._smtpServer = smtpServer;
    		this._subject = subject;
    		this._to = to;
    		this._html = html;
    	}

    	public List EmbeddedResources { get; set; }

    	public string To
    	{
    		get
    		{
    			return this._to;
    		}
    	}

    	public string From
    	{
    		get
    		{
    			return this._from;
    		}
    	}

    	public string Subject
    	{
    		get
    		{
    			return this._subject;
    		}
    	}

    	public string SmtpServer
    	{
    		get
    		{
    			return this._smtpServer;
    		}
    	}

    	public string Message
    	{
    		get
    		{
    			return this._message;
    		}
    	}

    	public bool Html
    	{
    		get
    		{
    			return this._html;
    		}
    	}
    }


…where EmbeddedEmailResource is a new object, we'll show it in a second.

So 6 parameters are made compulsory: to, from, subject, smtpServer, message body and whether the message is HTML. We can probably make this list shorter by excluding the subject and isHtml parameters but it's a good start.

Embedded email resources come in many different forms. Insert the following enum in the Email folder:



    public enum EmbeddedEmailResourceType
    {
    	Jpg
    	, Gif
    	, Tiff
    	, Html
    	, Plain
    	, RichText
    	, Xml
    	, OctetStream
    	, Pdf
    	, Rtf
    	, Soap
    	, Zip
    }


Embedded resources will be represented by the EmbeddedEmailResource object:



    public class EmbeddedEmailResource
    {
    	public EmbeddedEmailResource(Stream resourceStream, EmbeddedEmailResourceType resourceType
    		, string embeddedResourceContentId)
    	{
    		if (resourceStream == null) throw new ArgumentNullException("Resource stream");
    		if (String.IsNullOrEmpty(embeddedResourceContentId)) throw new ArgumentNullException("Resource content id");
    		ResourceStream = resourceStream;
    		ResourceType = resourceType;
    		EmbeddedResourceContentId = embeddedResourceContentId;
    	}

    	public Stream ResourceStream { get; set; }
    	public EmbeddedEmailResourceType ResourceType { get; set; }
    	public string EmbeddedResourceContentId { get; set; }
    }


If you're familiar with [how to add embedded resources to an email][4] then you'll know why we need a Stream and a content id.

The solution should compile at this stage.

We're now ready for the great finale, i.e. the IEmailService interface:



    public interface IEmailService
    {
    	EmailSendingResult SendEmail(EmailArguments emailArguments);
    }


Both the return type and the single parameter are custom objects that can be extended without breaking the code for any existing callers. You can add new public getters and setters to EmailArguments instead of creating the 100th different version of SendEmail in the version with primitive parameters.

**Implementation**

We'll of course use the default emailing techniques built into .NET to implement IEmailService. Add a new class called SystemNetEmailService to the Email folder:



    public EmailSendingResult SendEmail(EmailArguments emailArguments)
    {
    	EmailSendingResult sendResult = new EmailSendingResult();
    	sendResult.EmailSendingFailureMessage = string.Empty;
    	try
    	{
    		MailMessage mailMessage = new MailMessage(emailArguments.From, emailArguments.To);
    		mailMessage.Subject = emailArguments.Subject;
    		mailMessage.Body = emailArguments.Message;
    		mailMessage.IsBodyHtml = emailArguments.Html;
    		SmtpClient client = new SmtpClient(emailArguments.SmtpServer);

    		if (emailArguments.EmbeddedResources != null && emailArguments.EmbeddedResources.Count > 0)
    		{
    			AlternateView avHtml = AlternateView.CreateAlternateViewFromString(emailArguments.Message, Encoding.UTF8, MediaTypeNames.Text.Html);
    			foreach (EmbeddedEmailResource resource in emailArguments.EmbeddedResources)
    			{
    				LinkedResource linkedResource = new LinkedResource(resource.ResourceStream, resource.ResourceType.ToSystemNetResourceType());
    				linkedResource.ContentId = resource.EmbeddedResourceContentId;
    				avHtml.LinkedResources.Add(linkedResource);
    			}
    			mailMessage.AlternateViews.Add(avHtml);
    		}

    		client.Send(mailMessage);
    		sendResult.EmailSentSuccessfully = true;
    	}
    	catch (Exception ex)
    	{
    		sendResult.EmailSendingFailureMessage = ex.Message;
    	}

    	return sendResult;
    }


The code won't compile due to the [extension method][5] ToSystemNetResourceType(). The LinkedResource object cannot work with our EmbeddedEmailResourceType type directly. It needs a string instead like "text/plain" or "application/soap+xml". Those values are in turn stored within the [MediaTypeNames][6] object.

Add the following static class to the Email folder:



    public static class EmailExtensions
    {
    	public static string ToSystemNetResourceType(this EmbeddedEmailResourceType resourceTypeEnum)
    	{
    		string type = MediaTypeNames.Text.Plain;
    		switch (resourceTypeEnum)
    		{
    			case EmbeddedEmailResourceType.Gif:
    				type = MediaTypeNames.Image.Gif;
    				break;
    			case EmbeddedEmailResourceType.Jpg:
    				type = MediaTypeNames.Image.Jpeg;
    				break;
    			case EmbeddedEmailResourceType.Html:
    				type = MediaTypeNames.Text.Html;
    				break;
    			case EmbeddedEmailResourceType.OctetStream:
    				type = MediaTypeNames.Application.Octet;
    				break;
    			case EmbeddedEmailResourceType.Pdf:
    				type = MediaTypeNames.Application.Pdf;
    				break;
    			case EmbeddedEmailResourceType.Plain:
    				type = MediaTypeNames.Text.Plain;
    				break;
    			case EmbeddedEmailResourceType.RichText:
    				type = MediaTypeNames.Text.RichText;
    				break;
    			case EmbeddedEmailResourceType.Rtf:
    				type = MediaTypeNames.Application.Rtf;
    				break;
    			case EmbeddedEmailResourceType.Soap:
    				type = MediaTypeNames.Application.Soap;
    				break;
    			case EmbeddedEmailResourceType.Tiff:
    				type = MediaTypeNames.Image.Tiff;
    				break;
    			case EmbeddedEmailResourceType.Xml:
    				type = MediaTypeNames.Text.Xml;
    				break;
    			case EmbeddedEmailResourceType.Zip:
    				type = MediaTypeNames.Application.Zip;
    				break;
    		}

    		return type;
    	}
    }


This is a simple converter extension to transform our custom enumeration to media type strings.

There you are, this is a good starting point to cover the email purposes in your project.

In the [next part][7] of this series we'll look at hiding the cryptography logic or our application.

View the list of posts on Architecture and Patterns [here][8].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/09/18/externalising-dependencies-with-dependency-injection-in-net-part-6-file-system/ "Externalising dependencies with Dependency Injection in .NET part 6: file system"
[2]: http://dotnetcodr.com/emailing/ "Emailing"
[3]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[4]: http://dotnetcodr.com/2014/09/03/how-to-send-emails-in-net-part-8-adding-images-to-html-contents/ "How to send emails in .NET part 8: adding images to HTML contents"
[5]: http://dotnetcodr.com/2014/03/06/extension-methods-part-1-the-basics/ "Extension methods in C# .NET part 1: the basics"
[6]: http://msdn.microsoft.com/en-us/library/system.net.mime.mediatypenames(v=vs.110).aspx "MediaTypeNames object on MSDN"
[7]: http://dotnetcodr.com/2014/09/25/externalising-dependencies-with-dependency-injection-in-net-part-8-symmetric-cryptography/ "Externalising dependencies with Dependency Injection in .NET part 8: symmetric cryptography"
[8]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
