[Source](http://dotnetcodr.com/2014/10/02/externalising-dependencies-with-dependency-injection-in-net-part-10-hashing-and-digital-signatures/ "Permalink to Externalising dependencies with Dependency Injection in .NET part 10: hashing and digital signatures")

# Externalising dependencies with Dependency Injection in .NET part 10: hashing and digital signatures

**Introduction**

In the [previous][1] post of this series we looked at how to hide the implementation of asymmetric encryption operations. In this post we'll look at abstracting away hashing and digital signatures as they often go hand in hand. Refer back to [this][2] post on hashing and [this][3] on digital signatures if you are not familiar with those concepts.

We'll continue directly from the previous post and extend the Infrastructure layer of our demo app we've been working on so far. So have it ready in Visual Studio and let's get to it.

**Preparation**

Add two new subfolders called Hashing and DigitalSignature to the Cryptography folder of the Infrastructure.Common library. We'll follow the same methodology as in the case of asymmetric encryption, i.e. we'll communicate with proper objects as parameters and return types instead of writing a large amount of overloaded methods.

Recall that we had a common abstract class to all return types in the IAsymmetricCryptographyService interface: ResponseBase. We'll reuse it here.

**Hashing**

In short hashing is a one way encryption. A plain text is hashed and there's no key to decipher the hashed value. A common application of hashing is storing passwords. Any hashing algorithm should be able to hash plain string and that's the only expectation we have. Add the below interface to the Hashing folder:



    public interface IHashingService
    {
    	HashResult Hash(string message);
    	string HashAlgorithmCode();
    }


…where HashResult will hold the readable and byte array formats of the hashed value and has the following form:



    public class HashResult : ResponseBase
    {
    	public string HexValueOfHash { get; set; }
    	public byte[] HashedBytes { get; set; }
    }


What is HashAlgorithmCode() doing there? The implementation if the interface will return a short description of itself, such as "SHA1″ or "SHA512″. We'll see later that it's required by the digital signature implementation.

This can actually be seen as a violation of '[L][4]' in SOLID as we're specifying the type of the implementation in some form. However, I think it serves a good purpose here and it will not be misused in some funny switch statement where we check the string returned by HashAlgorithmCode() in order to direct the logic. Let's insert two implementations of IHashingService to the Hashing folder. Here comes SHA1:



    public class Sha1ManagedHashingService : IHashingService
    {
    	public HashResult Hash(string message)
    	{
    		HashResult hashResult = new HashResult();
    		try
    		{
    			SHA1Managed sha1 = new SHA1Managed();
    			byte[] hashed = sha1.ComputeHash(Encoding.UTF8.GetBytes(message));
    			string hex = Convert.ToBase64String(hashed);
    			hashResult.HashedBytes = hashed;
    			hashResult.HexValueOfHash = hex;
    		}
    		catch (Exception ex)
    		{
    			hashResult.ExceptionMessage = ex.Message;
    		}
    		return hashResult;
    	}

    	public string HashAlgorithmCode()
    	{
    		return "SHA1";
    	}
    }


…and here's SHA512:



    public class Sha512ManagedHashingService : IHashingService
    {
    	public HashResult Hash(string message)
    	{
    		HashResult hashResult = new HashResult();
    		try
    		{
    			SHA512Managed sha512 = new SHA512Managed();
    			byte[] hashed = sha512.ComputeHash(Encoding.UTF8.GetBytes(message));
    			string hex = Convert.ToBase64String(hashed);
    			hashResult.HashedBytes = hashed;
    			hashResult.HexValueOfHash = hex;
    		}
    		catch (Exception ex)
    		{
    			hashResult.ExceptionMessage = ex.Message;
    		}
    		return hashResult;
    	}

    	public string HashAlgorithmCode()
    	{
    		return "SHA512";
    	}
    }


That's enough for starters.

**Digital signatures**

Digital signatures in a nutshell are used to electronically sign the hash of a message. The receiver of the message can then check if the message has been tampered with in any way. Also, a digital signature can prove that it was the sender who sent a particular message. E.g. a receiver may only accept signed messages so that the sender cannot claim that it was someone else who sent the message.

It is not "compulsory" to encrypt a signed message. You can sign the hash of an unencrypted message. However, for extra safety the hash is also often encrypted. The process is the following:

* The Sender encrypts a message with the Receiver's public key
* The Sender then hashes the cipher text
* The Sender signs the hashed cipher with their private key
* The Sender transmits the cipher text, the hash and the signature to the Receiver
* The Receiver verifies the signature with the Sender's public key
* If the signature is correct then the Receiver deciphers the cipher text with their full asymmetric key

The following interface will reflect what we need from a digital signature service: sign a message and then verify the signature:



    public interface IEncryptedDigitalSignatureService
    {
    	DigitalSignatureCreationResult Sign(DigitalSignatureCreationArguments arguments);
    	DigitalSignatureVerificationResult VerifySignature(DigitalSignatureVerificationArguments arguments);
    }


DigitalSignatureCreationResult will hold the signature and the cipher text:



    public class DigitalSignatureCreationResult : ResponseBase
    {
    	public byte[] Signature { get; set; }
    	public string CipherText { get; set; }
    }


DigitalSignatureCreationArguments will carry the message to be encrypted, hashed and signed. It will also hold the the full key set for the signature and the public key of the receiver for encryption:



    public class DigitalSignatureCreationArguments
    {
    	public string Message { get; set; }
    	public XDocument FullKeyForSignature { get; set; }
    	public XDocument PublicKeyForEncryption { get; set; }
    }


Then we have DigitalSignatureVerificationResult which has properties to check if the signatures match and the decoded text:



    public class DigitalSignatureVerificationResult : ResponseBase
    {
    	public bool SignaturesMatch { get; set; }
    	public string DecodedText { get; set; }
    }


…and finally we have DigitalSignatureVerificationArguments which holds the properties for the Sender's public key for signature verification, the Receiver's full key for decryption, the cipher text and the signature:



    public class DigitalSignatureVerificationArguments
    {
    	public XDocument PublicKeyForSignatureVerification { get; set; }
    	public XDocument FullKeyForDecryption { get; set; }
    	public string CipherText { get; set; }
    	public byte[] Signature { get; set; }
    }


**RsaPkcs1 digital signature provider**

We'll use the built-in RSA-based signature provider in .NET for the implementation. It will need to hash the cipher text so it's good that we've prepared IHashingService in advance. Here comes the implementation where you'll see how the HashAlgorithmCode() method is used in specifying the hash algorithm of the signature formatter and deformatter:



    public class RsaPkcs1DigitalSignatureService : IEncryptedDigitalSignatureService
    {
    	private readonly IHashingService _hashingService;

    	public RsaPkcs1DigitalSignatureService(IHashingService hashingService)
    	{
    		if (hashingService == null) throw new ArgumentNullException("Hashing service");
    		_hashingService = hashingService;
    	}

    	public DigitalSignatureCreationResult Sign(DigitalSignatureCreationArguments arguments)
    	{
    		DigitalSignatureCreationResult res = new DigitalSignatureCreationResult();
    		try
    		{
    			RSACryptoServiceProvider rsaProviderReceiver = new RSACryptoServiceProvider();
    			rsaProviderReceiver.FromXmlString(arguments.PublicKeyForEncryption.ToString());
    			byte[] encryptionResult = rsaProviderReceiver.Encrypt(Encoding.UTF8.GetBytes(arguments.Message), false);
    			HashResult hashed = _hashingService.Hash(Convert.ToBase64String(encryptionResult));

    			RSACryptoServiceProvider rsaProviderSender = new RSACryptoServiceProvider();
    			rsaProviderSender.FromXmlString(arguments.FullKeyForSignature.ToString());
    			RSAPKCS1SignatureFormatter signatureFormatter = new RSAPKCS1SignatureFormatter(rsaProviderSender);
    			signatureFormatter.SetHashAlgorithm(_hashingService.HashAlgorithmCode());
    			byte[] signature = signatureFormatter.CreateSignature(hashed.HashedBytes);

    			res.Signature = signature;
    			res.CipherText = Convert.ToBase64String(encryptionResult);
    			res.Success = true;
    		}
    		catch (Exception ex)
    		{
    			res.ExceptionMessage = ex.Message;
    		}

    		return res;
    	}

    	public DigitalSignatureVerificationResult VerifySignature(DigitalSignatureVerificationArguments arguments)
    	{
    		DigitalSignatureVerificationResult res = new DigitalSignatureVerificationResult();
    		try
    		{
    			RSACryptoServiceProvider rsaProviderSender = new RSACryptoServiceProvider();
    			rsaProviderSender.FromXmlString(arguments.PublicKeyForSignatureVerification.ToString());
    			RSAPKCS1SignatureDeformatter deformatter = new RSAPKCS1SignatureDeformatter(rsaProviderSender);
    			deformatter.SetHashAlgorithm(_hashingService.HashAlgorithmCode());
    			HashResult hashResult = _hashingService.Hash(arguments.CipherText);
    			res.SignaturesMatch = deformatter.VerifySignature(hashResult.HashedBytes, arguments.Signature);
    			if (res.SignaturesMatch)
    	        	{
            			RSACryptoServiceProvider rsaProviderReceiver = new RSACryptoServiceProvider();
      	        		rsaProviderReceiver.FromXmlString(arguments.FullKeyForDecryption.ToString());
    		        	byte[] decryptedBytes = rsaProviderReceiver.Decrypt(Convert.FromBase64String(arguments.CipherText), false);
    				res.DecodedText = Encoding.UTF8.GetString(decryptedBytes);
    			}

    			res.Success = true;
    		}
    		catch (Exception ex)
    		{
    			res.ExceptionMessage = ex.Message;
    		}

    		return res;
    	}
    }


The whole chain can be tested as follows:



    private static void TestDigitalSignatureService()
    {
    	IAsymmetricCryptographyService asymmetricService = new RsaAsymmetricCryptographyService();
    	AsymmetricKeyPairGenerationResult keyPairGenerationResultReceiver = asymmetricService.GenerateAsymmetricKeys(1024);
    	AsymmetricKeyPairGenerationResult keyPairGenerationResultSender = asymmetricService.GenerateAsymmetricKeys(1024);

    	IEncryptedDigitalSignatureService digitalSignatureService = new RsaPkcs1DigitalSignatureService(new Sha1ManagedHashingService());

    	DigitalSignatureCreationArguments signatureCreationArgumentsFromSender = new DigitalSignatureCreationArguments()
    	{
    		Message = "Text to be encrypted and signed"
    		, FullKeyForSignature = keyPairGenerationResultSender.FullKeyPairXml
    		, PublicKeyForEncryption = keyPairGenerationResultReceiver.PublicKeyOnlyXml
    	};
    	DigitalSignatureCreationResult signatureCreationResult = digitalSignatureService.Sign(signatureCreationArgumentsFromSender);
    	if (signatureCreationResult.Success)
    	{
    		DigitalSignatureVerificationArguments verificationArgumentsFromReceiver = new DigitalSignatureVerificationArguments();
    		verificationArgumentsFromReceiver.CipherText = signatureCreationResult.CipherText;
    		verificationArgumentsFromReceiver.Signature = signatureCreationResult.Signature;
    		verificationArgumentsFromReceiver.PublicKeyForSignatureVerification = keyPairGenerationResultSender.PublicKeyOnlyXml;
    		verificationArgumentsFromReceiver.FullKeyForDecryption = keyPairGenerationResultReceiver.FullKeyPairXml;
    		DigitalSignatureVerificationResult verificationResult = digitalSignatureService.VerifySignature(verificationArgumentsFromReceiver);
    		if (verificationResult.Success)
    		{
    			if (verificationResult.SignaturesMatch)
    			{
    				Console.WriteLine("Signatures match, decoded text: {0}", verificationResult.DecodedText);
    			}
    			else
    			{
    				Console.WriteLine("Signature mismatch!!");
    			}
    		}
    		else
    		{
    			Console.WriteLine("Signature verification failed: {0}", verificationResult.ExceptionMessage);
    		}
    	}
    	else
    	{
    		Console.WriteLine("Signature creation failed: {0}", signatureCreationResult.ExceptionMessage);
    	}
    }


Check how the keys of each party in the messaging conversation are used for encrypting, signing, verifying and decrypting the message.

So that's it for digital signatures.

This was the last post in the series on object dependencies. We could take up a number of other cross cutting concerns for the infrastructure layer, such as:

* Http calls
* User-level security, such as a role provider, a claims provider etc.
* Session provider
* Random password generation

…and possibly a lot more. By now you have a better understanding how to externalise tightly coupled object dependencies in your classes to make them more flexible and testable.

View the list of posts on Architecture and Patterns [here][5].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/09/29/externalising-dependencies-with-dependency-injection-in-net-part-9-cryptography-part-2/ "Externalising dependencies with Dependency Injection in .NET part 9: cryptography part 2"
[2]: http://dotnetcodr.com/2013/10/28/hashing-algorithms-and-their-practical-usage-in-net-part-1/ "Hashing algorithms and their practical usage in .NET Part 1"
[3]: http://dotnetcodr.com/2013/11/14/introduction-to-digital-signatures-in-net-cryptography/ "Introduction to digital signatures in .NET cryptography"
[4]: http://dotnetcodr.com/2013/08/19/solid-design-principles-in-net-the-liskov-substitution-principle/ "SOLID design principles in .NET: the Liskov Substitution Principle"
[5]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
