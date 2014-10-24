[Source](http://dotnetcodr.com/2014/09/25/externalising-dependencies-with-dependency-injection-in-net-part-8-symmetric-cryptography/ "Permalink to Externalising dependencies with Dependency Injection in .NET part 8: symmetric cryptography")

# Externalising dependencies with Dependency Injection in .NET part 8: symmetric cryptography

**Introduction**

In the [previous post][1] of this series we looked at how to hide the implementation of emailing operations. In this post we'll look at abstracting away cryptography. Cryptography is often needed to communicate between two systems in a safe way by e.g. exchanging public keys first.

If you don't know how to use cryptography in a .NET project check out the posts on [this][2] page.

The first couple of posts of this series went through the process of making hard dependencies to loosely coupled ones. I won't go through that again – you can refer back to the previous parts of this series to get an idea. You'll find a link below to view all posts of the series. [Another post on the Single Responsibility Principle][3] includes an example of breaking out different dependencies that you can look at.

We'll extend the Infrastructure layer of our demo app we've been working on so far. So have it ready in Visual Studio and let's get to it.

**Cryptography in .NET**

We discussed Cryptography in .NET on this blog before:

If you're interested in the technical details then please consult those pages. I won't go into cryptography in .NET but rather concentrate on the dependency injection side of it. We'll abstract away all four of the above topics. As it's a lot of code I've decided to break up the post into three parts:

* This post: symmetric encryption
* Next post: asymmetric encryption
* Last post: hashing and digital signatures

**Preparation**

Add a new folder called Cryptography to the Infrastructure.Common library. We'll follow the same methodology as in the case of emailing and communicate with proper objects as parameters and return types instead of writing a large amount of overloaded methods.

All return types will have 2 properties in common: Success and ExceptionMessage. A lot of things can go wrong in cryptography: wrong key size, unsupported block size, incomplete public/private key etc. The boolean property Success will indicate if the process was carried out successfully. ExceptionMessage will store the exception message in case Success is false. Insert the following abstract class to the Cryptography folder:



    public abstract class ResponseBase
    {
    	public bool Success { get; set; }
    	public string ExceptionMessage { get; set; }
    }


**Symmetric encryption**

Symmetric algorithm in a nutshell means that both the message sender and receiver have access to the same cryptographic key for both encryption and decryption.

Insert a subfolder called Symmetric within the Cryptography folder.

I think any reasonable symmetric encryption implementation should be able to perform the following 3 operations:

* Create a valid key of a given size in bits
* Encrypt a text
* Decrypt a cipher text

Add the following interface to the Symmetric folder to reflect these expectations:



    public interface ISymmetricCryptographyService
    {
    	SymmetricKeyGenerationResult GenerateSymmetricKey(int bitSize);
    	SymmetricEncryptionResult Encrypt(SymmetricEncryptionArguments arguments);
    	CipherTextDecryptionResult Decrypt(SymmetricDecryptionArguments arguments);
    }


SymmetricKeyGenerationResult is simple: it will contain the symmetric key and inherit from ResponseBase:



    public class SymmetricKeyGenerationResult : ResponseBase
    {
    	public string SymmetricKey { get; set; }
    }


SymmetricEncryptionResult will contain the encoded version of the plain text, called cipher text, and an initialisation vector (IV) for increased security. Note that IV is not always used but it's highly recommended as it adds randomness to the resulting cipher text making brute force decryption harder. Insert the following class to the Symmetric folder:



    public class SymmetricEncryptionResult : ResponseBase
    {
    	public string CipherText { get; set; }
    	public string InitialisationVector { get; set; }
    }


There's a number of arguments that will be necessary to both encryption and decryption such as the key size and the common key. Add the following abstract class to the subfolder:



    public class CommonSymmetricCryptoArguments
    {
    	public int KeySize { get; set; }
    	public int BlockSize { get; set; }
    	public PaddingMode PaddingMode { get; set; }
    	public CipherMode CipherMode { get; set; }
    	public string SymmetricPublicKey { get; set; }
    }


Both SymmetricEncryptionArguments and SymmetricDecryptionArguments will inherit from the above base class. Insert the following classes to the Asymmetric subfolder:



    public class SymmetricEncryptionArguments : CommonSymmetricCryptoArguments
    {
    	public string PlainText { get; set; }
    }




    public class SymmetricDecryptionArguments : CommonSymmetricCryptoArguments
    {
    	public string CipherTextBase64Encoded { get; set; }
    	public string InitialisationVectorBase64Encoded { get; set; }
    }


So we'll send in the plain text to be encrypted in SymmetricEncryptionArguments along with all other common arguments in CommonSymmetricCryptoArguments. Decryption will need the cipher text, the IV and the common arguments in CommonSymmetricCryptoArguments.

The last missing piece is CipherTextDecryptionResult. This object will be re-used for the asymmetric encryption section and will only include one extra property, namely the decoded text. Therefore add it to the Cryptography and not the Symmetric folder:



    public class CipherTextDecryptionResult : ResponseBase
    {
    	public string DecodedText { get; set; }
    }


The code should compile at this stage so we can move on with the implementation of the ISymmetricCryptographyService interface.

**The implementation: Rijndael**

For the implementation we'll take the widely used Rijndael managed implementation of symmetric encryption available in .NET. Add a new class called RijndaelManagedSymmetricEncryptionService to the Symmetric folder:



    public class RijndaelManagedSymmetricEncryptionService : ISymmetricCryptographyService
    {
    	public SymmetricEncryptionResult Encrypt(SymmetricEncryptionArguments arguments)
    	{
    		SymmetricEncryptionResult res = new SymmetricEncryptionResult();
    		try
    		{
    			RijndaelManaged rijndael = CreateCipher(arguments);
    			res.InitialisationVector = Convert.ToBase64String(rijndael.IV);
    			ICryptoTransform cryptoTransform = rijndael.CreateEncryptor();
    			byte[] plain = Encoding.UTF8.GetBytes(arguments.PlainText);
    			byte[] cipherText = cryptoTransform.TransformFinalBlock(plain, 0, plain.Length);
    			res.CipherText = Convert.ToBase64String(cipherText);
    			res.Success = true;
    		}
    		catch (Exception ex)
    		{
    			res.ExceptionMessage = ex.Message;
    		}
    		return res;
    	}

    	public CipherTextDecryptionResult Decrypt(SymmetricDecryptionArguments arguments)
    	{
    		CipherTextDecryptionResult res = new CipherTextDecryptionResult();
    		try
    		{
    			RijndaelManaged cipher = CreateCipher(arguments);
    			cipher.IV = Convert.FromBase64String(arguments.InitialisationVectorBase64Encoded);
    			ICryptoTransform cryptTransform = cipher.CreateDecryptor();
    			byte[] cipherTextBytes = Convert.FromBase64String(arguments.CipherTextBase64Encoded);
    			byte[] plainText = cryptTransform.TransformFinalBlock(cipherTextBytes, 0, cipherTextBytes.Length);
    			res.DecodedText = Encoding.UTF8.GetString(plainText);
    			res.Success = true;
    		}
    		catch (Exception ex)
    		{
    			res.ExceptionMessage = ex.Message;
    		}
    		return res;
    	}

    	public SymmetricKeyGenerationResult GenerateSymmetricKey(int bitSize)
    	{
    		SymmetricKeyGenerationResult res = new SymmetricKeyGenerationResult();
    		try
    		{
    			byte[] key = new byte[bitSize / 8];
    			RNGCryptoServiceProvider rng = new RNGCryptoServiceProvider();
    			rng.GetBytes(key);
    			res.SymmetricKey = BitConverter.ToString(key).Replace("-", string.Empty);
    			res.Success = true;
    		}
    		catch (Exception ex)
    		{
    			res.ExceptionMessage = ex.Message;
    		}
    		return res;
    	}

    	private RijndaelManaged CreateCipher(CommonSymmetricCryptoArguments arguments)
    	{
    		RijndaelManaged cipher = new RijndaelManaged();
    		cipher.KeySize = arguments.KeySize;
    		cipher.BlockSize =  arguments.BlockSize;
    		cipher.Padding = arguments.PaddingMode;
    		cipher.Mode = arguments.CipherMode;
    		byte[] key = HexToByteArray(arguments.SymmetricPublicKey);
    		cipher.Key = key;
    		return cipher;
    	}

    	private byte[] HexToByteArray(string hexString)
    	{
    		if (0 != (hexString.Length % 2))
    		{
    			throw new ApplicationException("Hex string must be multiple of 2 in length");
    		}
    		int byteCount = hexString.Length / 2;
    		byte[] byteValues = new byte[byteCount];
    		for (int i = 0; i < byteCount; i++)
    		{
    			byteValues[i] = Convert.ToByte(hexString.Substring(i * 2, 2), 16);
    		}

    		return byteValues;
    	}
    }


This code should be quite straightforward – again, consult the posts on symmetric encryption if this is new to you.

The methods in the interface can be chained together in an example as follows:



    private static void TestSymmetricEncryptionService()
    {
    	ISymmetricCryptographyService symmetricService = new RijndaelManagedSymmetricEncryptionService();
    	SymmetricKeyGenerationResult validKey = symmetricService.GenerateSymmetricKey(128);
    	if (validKey.Success)
    	{
    		Console.WriteLine("Key used for symmetric encryption: {0}", validKey.SymmetricKey);

    		SymmetricEncryptionArguments encryptionArgs = new SymmetricEncryptionArguments()
    		{
    			BlockSize = 128
    			,CipherMode = System.Security.Cryptography.CipherMode.CBC
    			,KeySize = 256
    			,PaddingMode = System.Security.Cryptography.PaddingMode.ISO10126
    			,PlainText = "Text to be encoded"
    			,SymmetricPublicKey = validKey.SymmetricKey
    		};

    		SymmetricEncryptionResult encryptionResult = symmetricService.Encrypt(encryptionArgs);

    		if (encryptionResult.Success)
    		{
    			Console.WriteLine("Cipher text from encryption: {0}", encryptionResult.CipherText);
    			Console.WriteLine("Initialisation vector from encryption: {0}", encryptionResult.InitialisationVector);
    			SymmetricDecryptionArguments decryptionArgs = new SymmetricDecryptionArguments()
    			{
    				BlockSize = encryptionArgs.BlockSize
    				,CipherMode = encryptionArgs.CipherMode
    				,CipherTextBase64Encoded = encryptionResult.CipherText
    				,InitialisationVectorBase64Encoded = encryptionResult.InitialisationVector
    				,KeySize = encryptionArgs.KeySize
    				,PaddingMode = encryptionArgs.PaddingMode
    				,SymmetricPublicKey = validKey.SymmetricKey
    			};
    			CipherTextDecryptionResult decryptionResult = symmetricService.Decrypt(decryptionArgs);
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
    		Console.WriteLine("Symmetric key generation failure: {0}", validKey.ExceptionMessage);
    	}
    }


In short we generate a valid symmetric key of 128 bits, populate the common encryption arguments, provided a plain text, encrypt it and then decrypt the cipher text using the common arguments and the IV.

In the [next post][4] we'll look at asymmetric encryption.

View the list of posts on Architecture and Patterns [here][5].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/09/22/externalising-dependencies-with-dependency-injection-in-net-part-7-emailing/ "Externalising dependencies with Dependency Injection in .NET part 7: emailing"
[2]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
[3]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[4]: http://dotnetcodr.com/2014/09/29/externalising-dependencies-with-dependency-injection-in-net-part-9-cryptography-part-2/ "Externalising dependencies with Dependency Injection in .NET part 9: cryptography part 2"
[5]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
