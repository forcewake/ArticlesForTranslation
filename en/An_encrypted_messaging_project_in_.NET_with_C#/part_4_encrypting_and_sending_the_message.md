[Source](http://dotnetcodr.com/2013/12/05/an-encrypted-messaging-project-in-net-with-c-part-4-encrypting-and-sending-the-message/ "Permalink to An encrypted messaging project in .NET with C# part 4: encrypting and sending the message")

# An encrypted messaging project in .NET with C# part 4: encrypting and sending the message

**Introduction**

In the previous post we got as far as retrieving an RSA public key and a message ID from the Receiver. It is now time for the Sender to do the following:

* Encrypt the message using his **symmetric encryption key**
* Encrypt his symmetric encryption key using the RSA public key of the receiver
* Send a message to the Receiver which includes all this information

If you are not familiar with the basics of symmetric and asymmetric encryption algorithms then I highly recommend to read my blog posts: [here][1], [here][2] and [here][3]. I won't explain those concepts here in detail again. The sender will use the Rijndael managed AES algorithm as was introduced in the post on symmetric encryption.

**Demo**

Open the Sender console app we started building in the previous post. Add an app.config file to it where we'll store the AES public key. Add the following app setting:



    <configuration>
    	<appSettings>
    		<add key="RijndaelManagedKey" value="8B50BB68A8162E3D1E102556D2853291"/>
    	</appSettings>
    </configuration>


This is a 128bit key. It consists of 32 characters where each section comes in pairs according to the hexadecimal notation of a byte: 8B-50-BB etc. So there are 16 bytes stored in this string which is equal to 16 * 8 = 128 bits.

Note that the public key is included in the config file for convenience. In reality it should be stored in a more secure place, check out [this][4] post.

Insert a new class called SymmetricAlgorithmService to the project. Add the following private string to it:



    private readonly string _mySymmetricPublicKey = ConfigurationManager.AppSettings["RijndaelManagedKey"];


You'll need to add a reference to the System.Configuration library.

Let's first create our Rijndael managed cipher. Add the following method to the service:



    private RijndaelManaged CreateCipher()
    {
    	RijndaelManaged cipher = new RijndaelManaged();
    	cipher.KeySize = 128;
    	cipher.BlockSize = 128;
    	cipher.Padding = PaddingMode.ISO10126;
    	cipher.Mode = CipherMode.CBC;
    	byte[] key = HexToByteArray(_mySymmetricPublicKey);
    	cipher.Key = key;
    	return cipher;
    }


Consult the above links if you don't know what Mode, PaddingMode etc. mean. HexToByteArray is another private method that does what the method name suggests: convert a hexadecimal string to a byte array.



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


We're now ready to encrypt the message to be sent to the Receiver. Add the following method to the service:



    public SymmetricEncryptionResult Encrypt(string plainText)
    {
    	SymmetricEncryptionResult res = new SymmetricEncryptionResult();
    	RijndaelManaged rijndael = CreateCipher();
    	res.InitialisationVector = Convert.ToBase64String(rijndael.IV);
    	ICryptoTransform cryptoTransform = rijndael.CreateEncryptor();
    	byte[] plain = Encoding.UTF8.GetBytes(plainText);
    	byte[] cipherText = cryptoTransform.TransformFinalBlock(plain, 0, plain.Length);
    	res.CipherText = Convert.ToBase64String(cipherText);
    	return res;
    }


…where SymmetricEncryptionResult is a DTO to include the parameters that the Receiver will need later on:



    public class SymmetricEncryptionResult
    {
    	public string CipherText { get; set; }
    	public string InitialisationVector { get; set; }
    }


The symmetric encryption portion of the Sender is ready. Now insert a class called AsymmetricEncryptionService. It is very slim:



    public class AsymmetricEncryptionService
    {
    	private RSACryptoServiceProvider _rsaPublicKeyForEncryption;

    	public AsymmetricEncryptionService(RSACryptoServiceProvider rsaPublicKeyForEncryption)
    	{
    		_rsaPublicKeyForEncryption = rsaPublicKeyForEncryption;
    	}

    	public string GetCipherText(string plainText)
    	{
    		byte[] data = Encoding.UTF8.GetBytes(plainText);
    		byte[] cipherText = _rsaPublicKeyForEncryption.Encrypt(data, false);
    		return Convert.ToBase64String(cipherText);
    	}
    }


The RSACryptoServiceProvider object will be extracted from the PublicKeyResponse we got from the Receiver.

Locate the Main method. The last bit of code we entered in the previous post was the one that read the message to be sent from the console:



    Console.Write("Public key request successful. Enter your message: ");
    string message = Console.ReadLine();


Add the next two lines underneath:



    SymmetricAlgorithmService symmetricService = new SymmetricAlgorithmService();
    SymmetricEncryptionResult symmetricEncryptionResult = symmetricService.Encrypt(message);


Now we have the cipher text ready at this point. We'll now encrypt the AES public key. Add the following lines immediately after:



    AsymmetricEncryptionService asymmetricService = new AsymmetricEncryptionService(publicKeyResponse.RsaPublicKey);
    string encryptedAesPublicKey =  asymmetricService.GetCipherText(ConfigurationManager.AppSettings["RijndaelManagedKey"]);


We now have all the necessary ingredients. We'll wrap all these in a JSON message:



    SendMessageArguments sendMessageArgs = new SendMessageArguments() { EncryptedPublicKey = encryptedAesPublicKey,
    	SymmetricEncryptionArgs = symmetricEncryptionResult, MessageId = publicKeyResponse.MessageId };
    string jsonifiedArgs = JsonConvert.SerializeObject(sendMessageArgs);


…where SendMessageArguments is a wrapper DTO:



    public class SendMessageArguments
    {
    	public SymmetricEncryptionResult SymmetricEncryptionArgs { get; set; }
    	public string EncryptedPublicKey { get; set; }
    	public Guid MessageId { get; set; }
    }


We use Json.Net to build a JSON string out of this object.

Let's see if it works up to now. Open and start the Receiver object. Then start the Sender. After it received the public key from the Receiver it will prompt the user for a message. Enter something and follow the code execution using F11. If everything goes well then you should have a JSON string ready to be sent to the Receiver. The structure of the JSON string should look like the following:

{"SymmetricEncryptionArgs":{"CipherText":"QPELLoPSNj9EsjnXNoK6ow==","InitialisationVector":"pq4reDIUEI3uNloYlwc06Q=="},"EncryptedPublicKey":"a jungle of characters","MessageId":"0509dc4e-3766-4308-afe7-2ecbe6504e81"}

The exact values will of course differ from what you got.

We're now ready to send this message to the Receiver. If you haven't done so yet then open the Receiver project. We'll need to create a Controller for the incoming message. Right-click the Controllers folder in the Web layer and add an empty API controller called MessageController. We're going to POST the message from the Sender and send along the JSON which should be translated into an object.

We'll make the JSON translation as easy as possible so that we don't need to be sidetracked by serialisation issues. If you haven't done so yet add a folder called Requests to the ApplicationService layer and in it an object called ProcessSecretMessageRequest. It will have the same structure as SendMessageArguments on the Sender's side.

Add the following objects to the Requests folder:



    public class SymmetricEncryptionArguments
    {
    	public string CipherText { get; set; }
    	public string InitialisationVector { get; set; }
    }


ProcessSecretMessageRequest takes the following form:



    public class ProcessSecretMessageRequest
    {
    	public SymmetricEncryptionArguments SymmetricEncryptionArgs { get; set; }
    	public string EncryptedPublicKey { get; set; }
    	public Guid MessageId { get; set; }
    }


We'll also want to notify the sender of the success/failure of the message transfer so add the following object to the Responses folder of the ApplicationService layer:



    public class ProcessSecretMessageResponse : ServiceResponseBase
    {
    	public string SecretMessageProcessResult { get; set; }
    }


Put the following Post method into MessageController:



    public HttpResponseMessage Post(ProcessSecretMessageRequest secretMessageRequest)
    {
    	throw new NotImplementedException();
    }


Let's set up the Sender's message-sending function. Add the following method to Program.cs:



    private static void SendSecretMessageToReceiver(string jsonArguments)
    {
    	try
    	{
            	HttpRequestMessage requestMessage = new HttpRequestMessage(HttpMethod.Post, _secretMessageServiceUri);
    		requestMessage.Headers.ExpectContinue = false;
    		requestMessage.Content = new StringContent(jsonArguments, Encoding.UTF8, "application/json");
    		HttpClient httpClient = new HttpClient();
    		Task<HttpResponseMessage> httpRequest = httpClient.SendAsync(requestMessage,
    		HttpCompletionOption.ResponseContentRead, CancellationToken.None);
            	HttpResponseMessage httpResponse = httpRequest.Result;
    		HttpStatusCode statusCode = httpResponse.StatusCode;
    		HttpContent responseContent = httpResponse.Content;
    		if (responseContent != null)
    		{
    			Task<String> stringContentsTask = responseContent.ReadAsStringAsync();
    			String stringContents = stringContentsTask.Result;
                            Console.Write(stringContents);
    		}
    	}
    	catch (Exception ex)
    	{
    		Console.WriteLine("Exception caught while sending the secret message to the Receiver: "
    			+ ex.Message);
    	}
    }


…where _secretMessageServiceUri holds the URI to the MessageController:



    private static Uri _secretMessageServiceUri = new Uri("http://localhost:7695/Message");


Modify the port as necessary. So we simply send a POST message to the web service and await a string response. Put a break point within the – yet unimplemented – Post() method of the MessageController. Start the web service application and then run the Sender app as well. Enter your secret message in the console and then you should see that the ingredients of the secret message have been translated by the JSON formatter:

![Secret message values translated in web service][5]

We'll see in the next post how the Receiver processes and decrypts this message.

You can view the list of posts on Security and Cryptography [here][6].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/11/04/symmetric-encryption-algorithms-in-net-cryptography-part-1/ "Symmetric encryption algorithms in .NET cryptography part 1"
[2]: http://dotnetcodr.com/2013/11/07/symmetric-algorithms-in-net-cryptography-part-2/ "Symmetric algorithms in .NET cryptography part 2"
[3]: http://dotnetcodr.com/2013/11/11/introduction-to-asymmetric-encryption-in-net-cryptography/ "Introduction to asymmetric encryption in .NET cryptography"
[4]: http://dotnetcodr.com/2013/11/21/key-size-and-key-storage-in-net-cryptography/ "Key size and key storage in .NET cryptography"
[5]: http://dotnetcodr.files.wordpress.com/2013/10/secretmessagevaluestranslated.png?w=630&h=137
[6]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
