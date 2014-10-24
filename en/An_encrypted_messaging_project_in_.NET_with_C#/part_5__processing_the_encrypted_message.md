[Source](http://dotnetcodr.com/2013/12/09/an-encrypted-messaging-project-in-net-with-c-part-5-processing-the-encrypted-message/ "Permalink to An encrypted messaging project in .NET with C# part 5: processing the encrypted message")

# An encrypted messaging project in .NET with C# part 5: processing the encrypted message

We finished off the previous post with the Sender sending the ingredients of the secret message to the Receiver:

* The unique message ID
* The encrypted AES public key
* The AES cipher text
* The AES initialisation vector

Now it's up to the Receiver to decrypt the message. In case you don't understand some code specific to encryption in .NET please refer to my previous blog posts on the topic.

**Demo**

Open the Receiver project we've been working on. Add the following interface to the Interfaces folder of the ApplicationService layer:



    public interface ISecretMessageProcessingService
    {
    	ProcessSecretMessageResponse ProcessSecretMessage(ProcessSecretMessageRequest processSecretMessageRequest);
    }


We constructed the Request-Response objects in the previous post.

Add the following implementation stub to the Implementations folder:



    public class SecretMessageProcessingService : ISecretMessageProcessingService
    {
    	public ProcessSecretMessageResponse ProcessSecretMessage(ProcessSecretMessageRequest processSecretMessageRequest)
    	{
    		throw new NotImplementedException();
    	}
    }


The first step in the process will be to retrieve the AsymmetricKeyPairGenerationResult object based on the incoming message id. If you recall we have an interface called IAsymmetricKeyRepository for exactly that purpose. Add the following private field and public constructor to the service:



    private readonly IAsymmetricKeyRepositoryFactory _asymmetricKeyRepositoryFactory;

    public SecretMessageProcessingService(IAsymmetricKeyRepositoryFactory asymmetricKeyRepositoryFactory)
    {
    	if (asymmetricKeyRepositoryFactory == null) throw new ArgumentNullException("Asymmetric key repository factory.");
    	_asymmetricKeyRepositoryFactory = asymmetricKeyRepositoryFactory;
    }


Add the following stub to the ProcessSecretMessage method:



    ProcessSecretMessageResponse response = new ProcessSecretMessageResponse();
    try
    {
    	AsymmetricKeyPairGenerationResult asymmetricKeyInStore = _asymmetricKeyRepositoryFactory.Create().FindBy(processSecretMessageRequest.MessageId);				           _asymmetricKeyRepositoryFactory.Create().Remove(processSecretMessageRequest.MessageId);
    }
    catch (Exception ex)
    {
            response.SecretMessageProcessResult = string.Concat("Exception during the message decryption process: ", ex.Message);
    	response.Exception = ex;
    }
    return response;


Nothing awfully complicated up to now I hope: we retrieve the asymmetric key pair stored in the repository and remove it immediately. Recall that our implementation, i.e. InMemoryAsymmetricKeyRepository.cs throws an exception if the key is not found.

The next step is to decrypt the sender's public key. We need to extend the IAsymmetricCryptographyService interface in the Infrastructure layer. Open IAsymmetricCryptographyService.cs and add the following method:



    string DecryptCipherText(string cipherText, AsymmetricKeyPairGenerationResult keyPair);


Open RsaCryptographyService.cs and implement the new method as follows:



    public string DecryptCipherText(string cipherText, XDocument keyXml)
    {
    	RSACryptoServiceProvider rsa = new RSACryptoServiceProvider();
    	rsa.FromXmlString(keyPair.FullKeyPairXml.ToString());
    	byte[] original = rsa.Decrypt(Convert.FromBase64String(cipherText), false);
    	return Encoding.UTF8.GetString(original);
    }


Add the following private field to SecretMessageProcessingService.cs:



    private readonly IAsymmetricCryptographyService _asymmetricCryptoService;


Modify its constructor as follows:



    public SecretMessageProcessingService(IAsymmetricKeyRepositoryFactory asymmetricKeyRepositoryFactory
    	, IAsymmetricCryptographyService asymmetricCryptoService)
    {
    	if (asymmetricKeyRepositoryFactory == null) throw new ArgumentNullException("Asymmetric key repository factory.");
    	if (asymmetricCryptoService == null) throw new ArgumentNullException("Asymmetric crypto service.");
    	_asymmetricKeyRepositoryFactory = asymmetricKeyRepositoryFactory;
    	_asymmetricCryptoService = asymmetricCryptoService;
    }


Add the following line within the try-statement of the ProcessSecretMessage method:



    String decryptedPublicKey = _asymmetricCryptoService.DecryptCipherText(processSecretMessageRequest.EncryptedPublicKey
    	, asymmetricKeyInStore);


Now that we have the decrypted public key from the Sender we can build in the symmetric encryption service into the Receiver. Add the following interface into the Cryptography folder of the Infrastructure layer:



    public interface ISymmetricEncryptionService
    {
    	string Decrypt(string initialisationVector, string publicKey, string cipherText);
    }


Add the following private variable to SecretMessageProcessingService.cs:



    private readonly ISymmetricEncryptionService _symmetricCryptoService;


…and modify its constructor as follows:



    public SecretMessageProcessingService(IAsymmetricKeyRepositoryFactory asymmetricKeyRepositoryFactory
    	, IAsymmetricCryptographyService asymmetricCryptoService, ISymmetricEncryptionService symmetricCryptoService)
    {
    	if (asymmetricKeyRepositoryFactory == null) throw new ArgumentNullException("Asymmetric key repository factory.");
    	if (asymmetricCryptoService == null) throw new ArgumentNullException("Asymmetric crypto service.");
    	if (symmetricCryptoService == null) throw new ArgumentException("Symmetric crypto service.");
    	_asymmetricKeyRepositoryFactory = asymmetricKeyRepositoryFactory;
    	_asymmetricCryptoService = asymmetricCryptoService;
    	_symmetricCryptoService = symmetricCryptoService;
    }


Add the following statements to the try-block of ProcessSecretMessage:



    String decryptedMessage = _symmetricCryptoService.Decrypt(processSecretMessageRequest.SymmetricEncryptionArgs.InitialisationVector
    	, decryptedPublicKey, processSecretMessageRequest.SymmetricEncryptionArgs.CipherText);
    response.SecretMessageProcessResult = string.Concat("Message received and deciphered: ", decryptedMessage);


We let the symmetric crypto service decrypt the message using the ingredients that came from the Sender. We finally assign some message to the response. Normally you shouldn't of course send back the decrypted message in the receipt, but in this case we want to test on the sender's side if everything has gone well.

We'll use the built-in RijndaelManaged class to do the symmetric encryption bit. Add the following implementation to the Cryptography folder of the Infrastructure layer:



    public class RijndaelSymmetricCryptographyService : ISymmetricEncryptionService
    {
    	public string Decrypt(string initialisationVector, string publicKey, string cipherText)
    	{
    		RijndaelManaged cipherForDecryption = CreateCipherForDecryption(initialisationVector, publicKey);
    		ICryptoTransform cryptoTransform = cipherForDecryption.CreateDecryptor();
    		byte[] cipherTextBytes = Convert.FromBase64String(cipherText);
    		byte[] plainText = cryptoTransform.TransformFinalBlock(cipherTextBytes, 0, cipherTextBytes.Length);
    		return Encoding.UTF8.GetString(plainText);
    	}

    	private RijndaelManaged CreateCipherForDecryption(string initialisationVector, string publicKey)
    	{
    		RijndaelManaged cipher = new RijndaelManaged();
    		cipher.KeySize = CalculateKeySize(publicKey);
    		cipher.IV = RecreateInitialisationVector(initialisationVector);
    		cipher.BlockSize = 128;
    		cipher.Padding = PaddingMode.ISO10126;
    		cipher.Mode = CipherMode.CBC;
    		byte[] key = HexToByteArray(publicKey);
    		cipher.Key = key;
    		return cipher;
    	}

    	private int CalculateKeySize(string publicKey)
    	{
    		return (publicKey.Length / 2) * 8;
    	}

    	private byte[] RecreateInitialisationVector(string initialisationVector)
    	{
    		return Convert.FromBase64String(initialisationVector);
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


Most of this code should look familiar from the posts on symmetric encryption so I won't go into the details here.

We need to declare in IoC.cs in the Web layer that we want to use RijndaelSymmetricCryptographyService when StructureMap stumbles upon an ISymmetricEncryptionService interface dependency. Add the following line to the Initialize block:



    x.For<ISymmetricEncryptionService>().Use<RijndaelSymmetricCryptographyService>();


Start the Receiver and the Sender. Enter your message to be sent to the Receiver. I encourage you to step through the Receiver project with F11 to see how the bits and pieces are connected. If everything goes well then the Sender should get a JSON response similar to this:

{"$id":"1","SecretMessageProcessResult":"Message received and deciphered: your message","Exception":null}

That's it really as far as a basic implementation is concerned. The system can be extended in a number of ways, e.g.:

* Add an expiry date to the message ID from the Receiver so that the temporary RSA key pair must be consumed within a certain time limit
* Add digital signatures so that the Sender can check the authenticity of the message from the Receiver
* Use a one-time AES public key for the Sender as well. Even if an attacker manages to somehow figure out the public key of the Sender, it will be different when the next message is sent.

I'll be covering SOA in a number of posts soon which will show an example of adding expiry date checking to the message chain. Regarding digital signatures you can consult [this][1] post, there should be enough to get you started. The third point is straightforward to implement: just generate a new valid AES public key when the Sender sends out the message instead of reading it from the config file.

You can view the list of posts on Security and Cryptography [here][2].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/11/14/introduction-to-digital-signatures-in-net-cryptography/ "Introduction to digital signatures in .NET cryptography"
[2]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
