[Source](http://dotnetcodr.com/2013/11/11/introduction-to-asymmetric-encryption-in-net-cryptography/ "Permalink to Introduction to asymmetric encryption in .NET cryptography")

# Introduction to asymmetric encryption in .NET cryptography

**Introduction**

In the previous two blog posts we looked at symmetric encryption in .NET. Recall that it's called "symmetric" as both the receiver and the sender must have access to the same public key. Asymmetric encryption differs in that it involves two complementary keys: a public key and a private key. Asymmetric algorithms are also called Public Key Cryptography.

These algorithms are up to 100-1000 times slower than symmetric ones. They are therefore often used to encrypt small size data such as a symmetric key.

We can graphically show the data flow as follows:

![Asymmetric algorithm flow][1]

If I want to send some secret data to a particular receiver then I'll need their public key. Then I'll do the encryption with that public key which creates the cipher text. When the receiver gets the cipher text they will decrypt it using their private key. So these keys come in pairs in a sense that you encrypt with one and decrypt with the other.

In .NET asymmetric algorithms are represented by the AsymmetricAlgorithm abstract base class. There are a few implementations readily available:

The most common is RSA. DSA and EXDiffieHellman are only used in conjunction with digital signatures.

**The public and private keys are inverses of each other**: a message encrypted with the public key can be decrypted with the private key. Similarly, the same message encrypted with the private key and be decrypted by the public key.

**Demo**

Open Visual Studio and create a new Console application. Insert a new class called AsymmetricAlgoService. RSA is represented by the RSACryptoServiceProvider object so we'll need to create it first for the encryption part. Insert the following method:



    private RSACryptoServiceProvider CreateCipherForEncryption()
    {
    	RSACryptoServiceProvider cipher = new RSACryptoServiceProvider();
    	cipher.FromXmlString(_rsaKeyForEncryption);
    	return cipher;
    }


…where _rsaKeyForEncryption is a private string field with some XML that we'll look at in a second.

There are several ways to generate and store the RSA keys. There's a FromXmlString method which populates the appropriate properties of the RSACryptoServiceProvider object based on an XML string. The XML string is populated from a previously exported set of RSA keys. You can also do this using the CspParameters object as follows:



    CspParameters cspParams = new CspParameters();
    cspParams.KeyContainerName = "RsaKeys";
    cspParams.Flags = CspProviderFlags.UseMachineKeyStore;
    RSACryptoServiceProvider crypto = new RSACryptoServiceProvider(cspParams);


This will use an existing RSA protected key container called RsaKeys. If the container doesn't exist then it auto-generates one. The public-private key pair will be stored in Windows key storage, i.e. on the machine level, hence the selected flag value.

You can use the aspnet_regiis tool to generate and export RSA keys. This tool is usually located in one of these folders:

C:WindowsMicrosoft.NETFrameworkv4.0.30319
C:WindowsMicrosoft.NETFramework64v4.0.30319

Open a Command prompt and navigate to the correct folder. Insert the following command:

aspnet_regiis -pc "MyKeys" -exp

You can specify a different key container name if you want of course. If everything went well then you should see "Creating RSA Key container… Succeeded!" in the command window. The -exp flag means that the key pair can be exported. In order to export the contents of the key container to an xml file you can use the following command:

aspnet_regiis -px "MyKeys" c:tempmykeys.xml -pri

The -pri flag indicates that we want to export the private key as well. We'll only need it in our code, that's not something you want to redistribute.

Check the contents of the XML file. It will have the following elements: modulus, exponent, p, q, dp, dq, inverseq and d. Copy the modulus and exponent elements to the _rsaKeyForEncryption private field mentioned above, so the field should have the following format:



    private const string _rsaKeyForEncryption = @"<RSAKeyValue>
    	<Modulus>blahblah</Modulus>
    	<Exponent>blahblah</Exponent>
    </RSAKeyValue>";


This is the XML that you can distribute to the programmes and partners that need to send encrypted data to your application(s). You can already now create a second private field and assign the full XML to it:



    private const string _rsaKeyForDecryption = @"<RSAKeyValue>
    	<Modulus>blah</Modulus>
    	<Exponent>blah</Exponent>
    	<P>blah</P>
    	<Q>blah</Q>
    	<DP>blah</DP>
    	<DQ>blah</DQ>
    	<InverseQ>blah</InverseQ>
            <D>blah</D>
    </RSAKeyValue>";


This is only one way to create and export a valid RSA key pair. You can achieve this programmatically as well:



    public void ProgrammaticRsaKeys()
    {
    	RSACryptoServiceProvider myRSA = new RSACryptoServiceProvider();
    	RSAParameters publicKey = myRSA.ExportParameters(false);
    	string xml = myRSA.ToXmlString(true);
    }


This creates a new instance of an RSA algorithm and exports its auto-generated public key to an RSAParameters object. The 'false' parameter sent to ExportParameters method says please don't want to export the private key. Which is fine as the private key should stay by you only. The ToXmlString(true) shows the xml output including the private key. If you set that to false then the XML will only show the Modulus and Exponent elements.

The most interesting sections in the extended version of the XML string are D, Modulus and Exponent. D denotes the private key. Exponent is the short part of the public key whereas Modulus is the long part. You won't need to access the other elements in the XML string.

There's also a distinction between user-level and machine-level keys. A currently logged-on user can access the machine-level keys and those user-level ones that they have generated for themselves. When a different user logs onto the same computer then they won't have access to the user-level keys assigned to other users. However, they will still have access to the machine-level keys. Inspect the CspProviderFlags enumeration mentioned above. You'll see that it includes an option for user-level key storage. You can read more about this distinction on MSDN [here][2].

We can encrypt any string input as follows:



    public string GetCipherText(string plainText)
    {
    	RSACryptoServiceProvider cipher = CreateCipherForEncryption();
    	byte[] data = Encoding.UTF8.GetBytes(plainText);
    	byte[] cipherText = cipher.Encrypt(data, false);
    	return Convert.ToBase64String(cipherText);
    }


For decryption we'll need a cipher that reads the full XML string:



    private RSACryptoServiceProvider CreateCipherForDecryption()
    {
    	RSACryptoServiceProvider cipher = new RSACryptoServiceProvider();
    	cipher.FromXmlString(_rsaKeyForDecryption);
    	return cipher;
    }


The decryption method is the inverse of the one performing the encryption part:



    public string DecryptCipherText(string cipherText)
    {
    	RSACryptoServiceProvider cipher = CreateCipherForDecryption();
    	byte[] original = cipher.Decrypt(Convert.FromBase64String(cipherText), false);
    	return Encoding.UTF8.GetString(original);
    }


You can call these methods from Program.cs as follows:



    AsymmetricAlgoService asymmService = new AsymmetricAlgoService();
    string cipherText = asymmService.GetCipherText("Hello");
    Console.WriteLine(cipherText);
    string original = asymmService.DecryptCipherText(cipherText);
    Console.WriteLine(original);


Run the code and you should see that the translates the string back to its original value. Just for verification purposes modify the CreateCipherForDecryption() method so that the encryption XML is used instead i.e. without the private key:



    cipher.FromXmlString(_rsaKeyForEncryption);


Decryption still succeeds but a cryptographic exception will be thrown within the Decryption method saying that the key does not exist.

**Considerations**

In practice you wouldn't use RSA to encrypt and decrypt arbitrarily long strings. You can encrypt your string using a symmetric algorithm, like AES discussed [here][3] then encrypt the AES secret public key using the RSA public key. The other party will be able to decrypt the secret AES key using the RSA private key. Then using the key they'll be able to decrypt your string. We will build a model application which shows how to do this in a later post.

Often a web site needs sensitive information, such as a bank account number. Normally such data should be encrypted in some way. We could take a symmetric algorithm but if you are familiar with that approach then you know why it's not a good idea – the secret key needs to be shared between the encryption and decryption.

Instead we need to look into what asymmetric encryption has to offer. As we saw before we need to start by generating an RSA key pair and store the public key on the web server. We can share the public key with anyone who needs to send encrypted data to us. We then would need to make sure that the private key is stored safely for the decryption so that no attacker has access to it. The private key could be stored on an internal web server behind a firewall. That web server could be dedicated to data decryption. The public key would then be stored on an "open" web server which hosts your web site. That server could do the encryption and in the absence of the private key would have no possibility to decrypt even the message that it encrypted itself.

Due to the slow speed of asymmetric encryption this approach works well only with small amounts of data – use RSA directly to encrypt and decrypt that small string. As we hinted at at the end of the previous post RSA would rather be used to encrypt a symmetric key and encrypt the actual data using a symmetric algorithm. The symmetric key should be generated on the fly – it is propagated once to the requesting party and never reused. This is a better solution for encrypting and decrypting large amounts of data as symmetric algorithms are a lot faster. Asymmetric algorithms in this case provide a good way around the key-storage problem. Mixing these strategies will provide the advantages of each: speed, safe key storage, maximum message security.

However, you still need to be careful. If you want to send encrypted data to someone then you must be sure that you are encrypting using the correct public key. E.g. if the public key is printed on a blog for everyone to see then an attacker may take over that site and put his/her own public key there instead. In that case you may be sending the secret data to the attacker where they can decrypt it using their private key.

You can view the list of posts on Security and Cryptography [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.files.wordpress.com/2013/11/asymmetricflow.png?w=630
[2]: http://msdn.microsoft.com/en-us/library/f5cs0acs%28v=vs.90%29.aspx "Understanding Machine-Level and User-Level RSA Key Containers"
[3]: http://dotnetcodr.com/2013/11/04/symmetric-encryption-algorithms-in-net-cryptography-part-1/ "Symmetric encryption algorithms in .NET cryptography part 1"
[4]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
