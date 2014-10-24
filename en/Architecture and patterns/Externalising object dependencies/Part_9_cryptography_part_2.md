[Source](http://dotnetcodr.com/2014/09/29/externalising-dependencies-with-dependency-injection-in-net-part-9-cryptography-part-2/ "Permalink to Externalising dependencies with Dependency Injection in .NET part 9: cryptography part 2")

# Externalising dependencies with Dependency Injection in .NET part 9: cryptography part 2

**Introduction**

In the [previous][1] post of this series we looked at how to hide the implementation of symmetric encryption operations. In this post we'll look at abstracting away asymmetric cryptography. Refer back to [this][2] post in case you are not familiar with asymmetric encryption in .NET.

We'll continue directly from the previous post and extend the Infrastructure layer of our demo app we've been working on so far. So have it ready in Visual Studio and let's get to it.

**Preparation**

Add a new subfolder called Asymmetric to the Cryptography folder of the Infrastructure.Common library. We'll follow the same methodology as in the case of [emailing][3] and symmetric encryption, i.e. we'll communicate with proper objects as parameters and return types instead of writing a large amount of overloaded methods.

Recall that we had a common abstract class to all return types in the ISymmetricCryptographyService interface: ResponseBase. We'll reuse it here. There's one more class we'll use again here and that is CipherTextDecryptionResult.

**Asymmetric encryption**

In summary asymmetric encryption means that you have a pair of keys: a public and a private key. This is in contrast to symmetric encryption where there is only one key. The public key can be distributed to those who wish to send you an encrypted message. The private key will be used to decrypt the incoming cipher text. The public and private keys are in fact also symmetric: you can encrypt a message with the public key and decrypt with the private one and vice versa. As asymmetric encryption is considerably slower – but much more secure – than symmetric encryption it's usually not used to encrypt large blocks of plain text. Instead, the sender encrypts the text with a symmetric key and then encrypts the symmetric key with the receiver's public asymmetric key. The receiver will then decrypt the symmetric key using their asymmetric private key and decrypt the message with the decrypted symmetric key. If you want to see that in action I have a sample project which takes up this process starting [here][4].

Our expectations from any reasonable asymmetric encryption mechanism are the same as for symmetric algorithms:

* Create a valid key – actually a pair of keys – of a given size in bits
* Encrypt a text
* Decrypt a cipher text

Add the following interface to the Asymmetric folder to reflect these expectations:



    public interface IAsymmetricCryptographyService
    {
    	AsymmetricKeyPairGenerationResult GenerateAsymmetricKeys(int keySizeInBits);
    	AsymmetricEncryptionResult Encrypt(AsymmetricEncryptionArguments arguments);
    	CipherTextDecryptionResult Decrypt(AsymmetricDecryptionArguments arguments);
    }


We already know CipherTextDecryptionResult, here's a reminder:



    public class CipherTextDecryptionResult : ResponseBase
    {
    	public string DecodedText { get; set; }
    }


AsymmetricKeyPairGenerationResult will represent the keys in XML format. We'll need to represent the keys in two forms. The full representation is where the private key is included and should be kept with you. Another representation is where only the public key part of the pair is shown which can be used to distribute to your partners. Insert a class called AsymmetricKeyPairGenerationResult to the Asymmetric folder:



    public class AsymmetricKeyPairGenerationResult : ResponseBase
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


The keys are of type XDocument to make sure that they are in fact valid XML.

AsymmetricEncryptionResult will have two properties: the cipher text as a base64 string and as a byte array:



    public class AsymmetricEncryptionResult : ResponseBase
    {
    	public string CipherText { get; set; }
    	public byte[] CipherBytes { get; set; }
    }


To encrypt a message we'll need a plain text and the public-key-only part of the asymmetric keys:



    public class AsymmetricEncryptionArguments
    {
    	public XDocument PublicKeyForEncryption { get; set; }
    	public string PlainText { get; set; }
    }


Finally in order to decrypt a cipher text we'll need the complete set of keys and the cipher text itself:



    public class AsymmetricDecryptionArguments
    {
    	public XDocument FullAsymmetricKey { get; set; }
    	public string CipherText { get; set; }
    }


The project should compile at this stage.

**The implementation: RSA**

We'll go for one of the most widely used implementation of asymmetric encryption, RSA. .NET has good support for it so the implementation if straightforward. Insert a class called RsaAsymmetricCryptographyService into the Asymmetric folder:



    public class RsaAsymmetricCryptographyService : IAsymmetricCryptographyService
    {
    	public AsymmetricKeyPairGenerationResult GenerateAsymmetricKeys(int keySizeInBits)
    	{
    		try
    		{
    			RSACryptoServiceProvider myRSA = new RSACryptoServiceProvider(keySizeInBits);
    			XDocument publicKeyXml = XDocument.Parse(myRSA.ToXmlString(false));
    			XDocument fullKeyXml = XDocument.Parse(myRSA.ToXmlString(true));
    			return new AsymmetricKeyPairGenerationResult(fullKeyXml, publicKeyXml) { Success = true };
    		}
    		catch (Exception ex)
    		{
    			return new AsymmetricKeyPairGenerationResult(new XDocument(), new XDocument()) { ExceptionMessage = ex.Message };
    		}
    	}

    	public AsymmetricEncryptionResult Encrypt(AsymmetricEncryptionArguments arguments)
    	{
    		AsymmetricEncryptionResult encryptionResult = new AsymmetricEncryptionResult();
    		try
    		{
    			RSACryptoServiceProvider myRSA = new RSACryptoServiceProvider();
    			string rsaKeyForEncryption = arguments.PublicKeyForEncryption.ToString();
    			myRSA.FromXmlString(rsaKeyForEncryption);
    			byte[] data = Encoding.UTF8.GetBytes(arguments.PlainText);
    			byte[] cipherText = myRSA.Encrypt(data, false);
    			encryptionResult.CipherBytes = cipherText;
    			encryptionResult.CipherText = Convert.ToBase64String(cipherText);
    			encryptionResult.Success = true;
    		}
    		catch (Exception ex)
    		{
    			encryptionResult.ExceptionMessage = ex.Message;
    		}
    		return encryptionResult;
    	}

    	public CipherTextDecryptionResult Decrypt(AsymmetricDecryptionArguments arguments)
    	{
    		CipherTextDecryptionResult decryptionResult = new CipherTextDecryptionResult();
    		try
    		{
    			RSACryptoServiceProvider cipher = new RSACryptoServiceProvider();
    			cipher.FromXmlString(arguments.FullAsymmetricKey.ToString());
    			byte[] original = cipher.Decrypt(Convert.FromBase64String(arguments.CipherText), false);
    			decryptionResult.DecodedText = Encoding.UTF8.GetString(original);
    			decryptionResult.Success = true;
    		}
    		catch (Exception ex)
    		{
    			decryptionResult.ExceptionMessage = ex.Message;
    		}
    		return decryptionResult;
    	}
    }


If you're already familiar with asymmetric cryptography in .NET this should be straightforward. Otherwise consult the above link for the technical details, I won't repeat them here.

The methods in the interface can be chained together in an example as follows:



    private static void TestAsymmetricEncryptionService()
    {
    	IAsymmetricCryptographyService asymmetricSevice = new RsaAsymmetricCryptographyService();
    	AsymmetricKeyPairGenerationResult keyPairGenerationResult = asymmetricSevice.GenerateAsymmetricKeys(1024);
    	if (keyPairGenerationResult.Success)
    	{
    		AsymmetricEncryptionArguments encryptionArgs = new AsymmetricEncryptionArguments()
    		{
    			PlainText = "Text to be encrypted"
    			, PublicKeyForEncryption = keyPairGenerationResult.PublicKeyOnlyXml
    		};
    		AsymmetricEncryptionResult encryptionResult = asymmetricSevice.Encrypt(encryptionArgs);
    		if (encryptionResult.Success)
    		{
    			Console.WriteLine("Ciphertext: {0}", encryptionResult.CipherText);
    			AsymmetricDecryptionArguments decryptionArguments = new AsymmetricDecryptionArguments()
    			{
    				CipherText = encryptionResult.CipherText
    				, FullAsymmetricKey = keyPairGenerationResult.FullKeyPairXml
    			};
    			CipherTextDecryptionResult decryptionResult = asymmetricSevice.Decrypt(decryptionArguments);
    			if (decryptionResult.Success)
    			{
    				Console.WriteLine("Decrypted text: {0}", decryptionResult.DecodedText);
    			}
    			else
    			{
    				Console.WriteLine("Decryption failure: {0}", decryptionResult.ExceptionMessage);
    			}
    		}
    		else
    		{
    			Console.WriteLine("Encryption failure: {0}", encryptionResult.ExceptionMessage);
    		}
    	}
    	else
    	{
    		Console.WriteLine("Asymmetric key generation failure: {0}", keyPairGenerationResult.ExceptionMessage);
    	}
    }


If you run this code you'll see that the keys are generated, the plain text is encrypted and then decrypted again.

In the [next post][5], which will be last post of this series on dependencies, we'll look at two other aspects in cryptography: hashing and digital signatures.

View the list of posts on Architecture and Patterns [here][6].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/09/25/externalising-dependencies-with-dependency-injection-in-net-part-8-symmetric-cryptography/ "Externalising dependencies with Dependency Injection in .NET part 8: symmetric cryptography"
[2]: http://dotnetcodr.com/2013/11/11/introduction-to-asymmetric-encryption-in-net-cryptography/ "Introduction to asymmetric encryption in .NET cryptography"
[3]: http://dotnetcodr.com/2014/09/22/externalising-dependencies-with-dependency-injection-in-net-part-7-emailing/ "Externalising dependencies with Dependency Injection in .NET part 7: emailing"
[4]: http://dotnetcodr.com/2013/11/25/an-encrypted-messaging-project-in-net-with-c-part-1-foundations/ "An encrypted messaging project in .NET with C# part 1: foundations"
[5]: http://dotnetcodr.com/2014/10/02/externalising-dependencies-with-dependency-injection-in-net-part-10-hashing-and-digital-signatures/ "Externalising dependencies with Dependency Injection in .NET part 10: hashing and digital signatures"
[6]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
