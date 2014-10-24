[Source](http://dotnetcodr.com/2013/11/25/an-encrypted-messaging-project-in-net-with-c-part-1-foundations/ "Permalink to An encrypted messaging project in .NET with C# part 1: foundations")

# An encrypted messaging project in .NET with C# part 1: foundations

**Introduction**

In the past several posts we've discussed a number of issues in .NET cryptography. We'll take now what we've learn and will apply it in a model project in .NET. We'll focus mostly on symmetric and asymmetric encryption which we discussed [here][1], [here][2] and [here][3]. The goals of the project are as follows:

* Practice cryptography in a realistic web API project – at least as realistic as can be expected from a blog as opposed to an entire book
* Show how asymmetric and symmetric encryption could be used in parallel to increase messaging security
* Show how the public key of the asymmetric key pair can be used only once and refreshed for each new message
* Be independent of digital certificates

You may be wondering about this last point so let me explain more. You may not always have access to a valid certificate from a CA. Also, a certificate has an expiry date and if you forget to renew it then messaging to and from your server can not be verified. So the goal here is to have secure messaging in place without having to purchase, install and maintain certificates.

**Messaging flow**

Recall the following from the posts on encryption mechanisms mentioned above:

* Asymmetric encryption is safe but slow – in practice it is only used to encrypt short strings
* Symmetric encryption has the problem of sharing public keys, but symmetric algorithms are fast

However, we still want to transfer large texts in a secure and fast way. The solution is to combine the two techniques. Imagine the following conversation between the Sender (S) and the Receiver (R):

S: Hello R, I want to send you a message so I need your asymmetric public key
R: Hello S, OK, here you are, you get two things: my **asymmetric public key** and a message ID. Use this message ID in your message to me
S: Thanks! I'll encrypt my message using my **symmetric public key**, encrypt my symmetric public key with your asymmetric public key and attach the message ID as well. Here you are!
R: Great! I'll verify if the message ID is still valid. If so, then I'll decrypt your symmetric public key using my asymmetric private key and decrypt your message using the decrypted symmetric public key. I'll then remove the asymmetric public key from the list of valid IDs.

I hope you could follow along. The process should be a lot clearer when we look at some code.

The front end of the receiver in this model project will be a [Web API][4] based web service. The overall structure of the application will follow that of the model DDD skeleton project we built up [here][5] although in a very simplified format. Here we don't care about domains and domain logic, but we still want to build a layered application with good [SOLID][6] design. We could introduce a Domain project as well, but I don't want to digress too much otherwise we'll end up discussing stuff that's not related to the main topic: entity base, value objects etc. In case you'd like to extend this cryptography project to a more Domain Driven Design type of application then check out the provided link – you'll probably find enough information there to proceed.

We'll of course test the application with a sender. The sender will be a simple console project, we don't need anything more complicated.

**The receiver**

Open Visual Studio 2012/2013 and create a new blank solution called Receiver.

Add a class library called Receiver.Infrastructure to it. Delete Class1.cs and add a folder called Cryptography. If you don't know the purpose of this layer, then in short the infrastructure layer is used to include classes that encapsulate cross-cutting concerns such as logging, caching, identification. It's a container of objects and services that do not have any domain-specific roles and can be used in any other projects and layers.

The first function we want to implement in the project is the creation of a one-time asymmetric key pair that the sender can use. If you are familiar with good software engineering practices then you'll know that it's a good idea to abstract away dependencies. Here we want to hide the asymmetric key pair generation technique behind an interface. In most cases RSA will be the chosen technique, but there's nothing stopping you from employing other algorithms such as DSA. Also, unit testing will be easier if you can inject any type of algorithm into the object that has this dependency. You can read more about unit testing and TDD [here][7].

Add the following interface into the folder:



    public interface IAsymmetricCryptographyService
    {
    	AsymmetricKeyPairGenerationResult GenerateAsymmetricKeys();
    }


…where AsymmetricKeyPairGenerationResult looks like this:



    public class AsymmetricKeyPairGenerationResult
    {
    	private XDocument _fullKeyPairXml;
    	private XDocument _publicKeyOnlyXml;

    	public AsymmetricKeyPairGenerationResult(XDocument fullKeyPairXml, XDocument publicKeyOnlyXml)
    	{
    		if (fullKeyPairXml == null) throw new ArgumentNullException("Full key pair XML");
    		if (publicKeyOnlyXml == null) throw new ArgumentNullException("Public key only XML");
    		_fullKeyPairXml = fullKeyPairXml;
    		_publicKeyOnlyXml = publicKeyOnlyXml;
    	}

    	public XDocument FullKeyPairXml
    	{
    		get
    		{
    			return _fullKeyPairXml;
    		}
    	}

    	public XDocument PublicKeyOnlyXml
    	{
    		get
    		{
    			return _publicKeyOnlyXml;
    		}
    	}
    }


The purpose of the AsymmetricKeyPairGenerationResult object is to encapsulate the full asymmetric key pair and its public-key-only part in an object. If you recall from our discussion on asymmetric algorithms then the public key can be sent to anyone who wants to send a message to us. The full XML which includes the private key must stay safe. Therefore it's reasonable to store them in two different XML objects.

You may be wondering why I chose to store them as XDocuments. I wanted to make sure that the implementing class returns the keys as valid XML so that it's guaranteed that the sender will be able to extract the public key from it. The XDocument object will throw an error if you try to parse a string that's not formatted correctly.

We know that RSA is supported in .NET. However, we cannot make the RSACryptoServiceProvider object implement our interface, right? The [Adapter][8] pattern is an easy way to bridge this problem. Insert the following implementation of this interface in the same folder:



    public class RsaCryptographyService : IAsymmetricCryptographyService
    {
    	public AsymmetricKeyPairGenerationResult GenerateAsymmetricKeys()
    	{
    		RSACryptoServiceProvider myRSA = new RSACryptoServiceProvider();
    		XDocument publicKeyXml = XDocument.Parse(myRSA.ToXmlString(false));
    		XDocument fullKeyXml = XDocument.Parse(myRSA.ToXmlString(true));
    		return new AsymmetricKeyPairGenerationResult(fullKeyXml, publicKeyXml);
    	}
    }


The RsaCryptographyService encapsulates the built-in RSACryptoServiceProvider object. With the RSACryptoServiceProvider object it's very easy to generate new asymmetric key pair and export them to XML strings. The 'false' parameter means that we don't want to put the private key into the XDocument object. We finally return the key generation result that holds the full key and the public key as XML.

In case you want to test a different asymmetric algorithm then you just create a different implementation of the IAsymmetricCryptographyService interface and encapsulate that algorithm.

That's it for the Infrastructure layer. In the next post we'll continue with the Repository layer.

You can view the list of posts on Security and Cryptography [here][9].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/11/04/symmetric-encryption-algorithms-in-net-cryptography-part-1/ "Symmetric encryption algorithms in .NET cryptography part 1"
[2]: http://dotnetcodr.com/2013/11/07/symmetric-algorithms-in-net-cryptography-part-2/ "Symmetric algorithms in .NET cryptography part 2"
[3]: http://dotnetcodr.com/2013/11/11/introduction-to-asymmetric-encryption-in-net-cryptography/ "Introduction to asymmetric encryption in .NET cryptography"
[4]: http://www.asp.net/web-api "Web API home page"
[5]: http://dotnetcodr.com/2013/09/12/a-model-net-web-service-based-on-domain-driven-design-part-1-introduction/ "A model .NET web service based on Domain Driven Design Part 1: introduction"
[6]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[7]: http://dotnetcodr.com/2013/03/25/test-driven-development-in-net-part-1-the-absolute-basics-of-red-green-refactor/ "Test Driven Development in .NET Part 1: the absolute basics of Red, Green, Refactor"
[8]: http://dotnetcodr.com/2013/04/25/design-patterns-and-practices-in-net-the-adapter-pattern/ "Design patterns and practices in .NET: the Adapter Pattern"
[9]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
