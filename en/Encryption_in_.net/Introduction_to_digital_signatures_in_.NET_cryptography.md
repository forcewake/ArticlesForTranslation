[Source](http://dotnetcodr.com/2013/11/14/introduction-to-digital-signatures-in-net-cryptography/ "Permalink to Introduction to digital signatures in .NET cryptography")

# Introduction to digital signatures in .NET cryptography

**Introduction**

I've devoted the last few posts to cryptography in .NET: [hashing][1], [asymmetric][2] and [symmetric][3] encryption. Digital signatures are yet another application of cryptography. They provide data integrity and non-repudiation. This latter means that a user cannot claim that he or she wasn't the one how signed a particular – digital – document. As public keys can be distributed to other users for encryption purposes they are not good candidates for ensuring non-repudiation. So we need to use the sender's private key with which the contents of a message can be signed. The contents will be hashed. "Signing" means encrypting with the sender's private key.

Here's a flow diagram of the process:

![Digital signatures flow][4]

In this scenario we'd like to send some information to another person in a secure way. We want to encrypt it so that only that person can see it and we also want to sign that message so that the receiver will know that it came from us.

The message we want to send is shown on the left of the diagram as "plain text". The next step is the same as what we saw in asymmetric encryption: we encrypt the message with the receiver's public key. We get the cipher text as the result. We then hash that cipher text so that we know that it won't be tampered with while it's transmitted to the receiver. In addition it's not necessary to sign the entire data content as it can potentially grow very large and asymmetric encryption is rather slow – we don't want to waste a lot of time just signing large size data.

So instead of signing the actual cipher text we get the hash of it, we encrypt the hash using our own private key. The result is a digital signature and a cipher text. The cipher can only be encrypted by the receiver using their private key. On top of that the hash is signed with our private key. The receiver can decrypt the signature using our public key since – recall from the post on asymmetric encryption – the private and public keys are inverses of each other. The receiver will have access to our public key. If the decryption succeeds then the receiver will know that it must have come from someone who has the correct private key.

This all probably sounds a bit complex at first but I hope the demo will make it clear how it all works.

The creation of the cipher text is not actually required for digital signatures. The main purpose of digital signatures is tamper-proof messaging. In the demo we'll go for the full encryption cycle – if you don't require the encryption bit in your code then you can simply ignore it.

**Demo**

Create a new Console app in Visual Studio. Insert a class called Sender. Add two class level fields to Sender.cs:



    private string _myRsaKeys = "...";
    private string _receiversPublicKey = "...";


I've shown you some techniques in the previous post on how to generate valid RSA keys, I won't repeat it here. The important thing here is that _myRsaKeys will be an XML string including both my private and public keys, so copy the entire XML contents there. Then generate another set of RSA keys and save the public key portion of it in _receiversPublicKey, but don't throw away the rest, it will be needed later when we look at the Receiver. Again, refer to the previous post to see how to keep the private and public key portions of an RSA key pair in an XML string variable. This is a simulation of a case where I received the public key of a partner whome I wish to send encrypted messages.

We'll need RSA ciphers for the sender and receiver so add the following methods to Sender.cs:



    private RSACryptoServiceProvider GetSenderCipher()
    {
    	RSACryptoServiceProvider sender = new RSACryptoServiceProvider();
    	sender.FromXmlString(_myRsaKeys);
    	return sender;
    }

    private RSACryptoServiceProvider GetReceiverCipher()
    {
    	RSACryptoServiceProvider sender = new RSACryptoServiceProvider();
    	sender.FromXmlString(_receiversPublicKey);
    	return sender;
    }


This should look familiar from the previous post. We'll use our own private key in the GetSenderCipher method for creating the signature – if you get lost you can always refer back to the diagram above. The receiver's public key will be needed in order to encrypt the plain text and create the cipher text. They will use their private key to do the decryption.

We can compute the hash of the cipher text using the following method:



    private byte[] ComputeHashForMessage(byte[] cipherBytes)
    {
    	SHA1Managed alg = new SHA1Managed();
    	byte[] hash = alg.ComputeHash(cipherBytes);
    	return hash;
    }


This should look familiar from the post on hashing techniques in .NET. We'll sign this hash instead of the entire data transmitted.

There's a built in signature provider in .NET represented by the RSAPKCS1SignatureFormatter object which will come handy in the following method:



    private byte[] CalculateSignatureBytes(byte[] hashToSign)
    {
    	RSAPKCS1SignatureFormatter signatureFormatter = new RSAPKCS1SignatureFormatter(GetSenderCipher());
    	signatureFormatter.SetHashAlgorithm("SHA1");
    	byte[] signature = signatureFormatter.CreateSignature(hashToSign);
    	return signature;
    }


The RSAPKCS1SignatureFormatter object then accepts an RSA provider to sign with which in this case will be our private key. We specify SHA1 as the hash algorithm for the signature. This is the signature that the receiver will verify using our public key.

The methods can be connected in the following public method:



    public DigitalSignatureResult BuildSignedMessage(string message)
    {
    	byte[] messageBytes = Encoding.UTF8.GetBytes(message);
    	byte[] cipherBytes = GetReceiverCipher().Encrypt(messageBytes, false);
    	byte[] cipherHash = ComputeHashForMessage(cipherBytes);
    	byte[] signatureHash = CalculateSignatureBytes(cipherHash);

    	string cipher = Convert.ToBase64String(cipherBytes);
    	string signature = Convert.ToBase64String(signatureHash);
    	return new DigitalSignatureResult() { CipherText = cipher, SignatureText = signature };
    }


…where DigitalSignatureResult is a simple DTO:



    public class DigitalSignatureResult
    {
    	public string CipherText { get; set; }
    	public string SignatureText { get; set; }
    }


The steps in the BuildSignedMessage correspond to the flow diagram: we encrypt the message, compute a hash of it and finally sign it.

Let's test from Program.cs if it looks OK up to this point:



    static void Main(string[] args)
    {
    	Sender sender = new Sender();
    	DigitalSignatureResult res = sender.BuildSignedMessage("Hello digital sig!");
    	Console.WriteLine(res.CipherText);
    	Console.WriteLine(res.SignatureText);

    	Console.ReadKey();
    }


Run the programme and if everything went well then you should see two sets of character-jungles in the console window.

Now let's see what the receiver looks like. Add a new class called Receiver. Insert the following class level private fields:



    private string _myRsaKeys = "...";
    private string _senderPublicKey = "...";


Here the values will be the inverses of the _myRsaKeys and _receiversPublicKey fields of Sender.cs. Receiver._myRsaKeys will be the full XML version of Sender._receiversPublicKey. Conversely Receiver._senderPublicKey will be the reduced public-key-only version of Sender._myRsaKeys. The sender's public key will be used to verify their signature.

To make this clearer I have the following values in Receiver.cs:



    private string _myRsaKeys = "<RSAKeyValue><Modulus>vU3Yfu1Z4nFknj9daoDmh+I0CzR+aLnTjUSejQyNJ0IgMb59x4mVe17C6U+bl4Cry7gXAk3LEmmE/BRxjlF8HKlXixoBWak1dpmr89Ye7iaD2UWwl5Dmn07Q9s27NGdywy0BsD1vDcFSgno3LUbVznkw/0hypbnOPxWKlBCao2c=</Modulus><Exponent>AQAB</Exponent><P>6veL+pbUjOr0PAiFcvBRwNlTz/+8T1iLHqkCggRPDSsTg25ybSqDa98mP5NQj9LHSYCECjOGZkiN4NoxgPPDxw==</P><Q>zj/l0Z36A/iD2IrVQzrEsvp31cmU6f9VCyPIGiM0FSEXbj23JuPNUPCzSo5oAAiSZfs/hR9uuAx1xQFAfTzjYQ==</Q><DP>dsW7VGh5+OGro80K6BbivIEfBL1ZCyLO8Ciuw9o5u4ZSztU9skETPawHQYvN5WW+p0D3fdCd14ZFcavZ6j1OcQ==</DP><DQ>YSQBRzgjsEkVOCEzjsWYLUAAvwWBiLCEyolgzsaz2hvK4FZa9AspAa1MlJn768Ady8CJS1bhm/fqZA5R5GqQIQ==</DQ><InverseQ>zEGFnyMtfxSYHwRv8nZ4xVcFctnU2pYmmXXYv8NV5FvhZi8Z1f1GE3tmS8qDyIuDTrXjmII2cffLMjPOVmLKoQ==</InverseQ><D>Ii97qDg+oijuDbHNsd0DRIix81AQf+MG9BzvMPOSTgOgAruuxSjwaK4NLsrkgzCGVayx4wWfZXzOuiMK+rN2YPr6IPeut3O14uuwLH7brxkit+MnhclsCtKpdT2iuUGOnbEhWccepCO7YLyyczhT9GE0rEtbEK6S7wvVKab/osE=</D></RSAKeyValue>";


    private string _senderPublicKey = "<RSAKeyValue><Modulus>rW0Prd+S+Z6Wv0gEakgSp/v8Pu4xJ6OjaVCHKTIcf/C5nZvE77454lii3Ne6odV+76oaM2Pn3I9kKehK7CtqklI7rc1+05WRE3u8O5tC5v2ECjEDPMULAcZVTjXSyZtSAOiqk+6nEcJGRED65aGXwFgZuxEY8y4FbUma3I311aM=</Modulus><Exponent>AQAB</Exponent></RSAKeyValue>";


…and in Sender.cs:



    private string _myRsaKeys = "<RSAKeyValue><Modulus>rW0Prd+S+Z6Wv0gEakgSp/v8Pu4xJ6OjaVCHKTIcf/C5nZvE77454lii3Ne6odV+76oaM2Pn3I9kKehK7CtqklI7rc1+05WRE3u8O5tC5v2ECjEDPMULAcZVTjXSyZtSAOiqk+6nEcJGRED65aGXwFgZuxEY8y4FbUma3I311aM=</Modulus><Exponent>AQAB</Exponent><P>5TYzDyoQBT4C8eqyuWlfNbg0XfnJAUHzonOiz/5az86E9y8V3oxDH3B3GMECDzvcLRJnp5x/G1Lectu1p3ckDw==</P><Q>wbHOTIh7l/p9FszFj/uMdvLlITyABeOZVJEPJhw6fkMSqiRqnx4F2dtqRcGUDBhpWbG6kbTXi9ijMVL8u+iRLQ==</Q><DP>h0KOqvo1bgKEFmJbiZKm/rpvHK3UcguLTGhUwczlpg/G419D1oqK6biib1cmcfrvGSHtTTnKwEMMxlblQafK/Q==</DP><DQ>u80hQFVouF+Xn16mA0eb1s0FWmdlndAin7sSHBpsoHV6CFvMwUCD3cp/TOk3GU8l/mBzi8jy4NYIzM8w2yTQdQ==</DQ><InverseQ>1rYDocFlo3EEs28Miieqa/fE8uzESz6YWONuZPoKHWO/1m9Tf0K01+TtPqDBFRhFBaTNKBJ2lyCGGRIEA41CYg==</InverseQ><D>dZvsciGYbqfZ20ZfmCPgYwNEAPlPZG5Yt2bhAlL1eN4rQnMMjvkWECXD7Lhv3KgIOUfGFOu/pZeoebMKfDbFQe6uA9f4jSYiC3yI0lyGiZQ+SpyJPRKetSSSqiOcK/vnnn2+03RgOVnyU3T52hRXVsb3oXtT5xacWm4IeGABB2E=</D></RSAKeyValue>";


    private string _receiversPublicKey = "<RSAKeyValue><Modulus>vU3Yfu1Z4nFknj9daoDmh+I0CzR+aLnTjUSejQyNJ0IgMb59x4mVe17C6U+bl4Cry7gXAk3LEmmE/BRxjlF8HKlXixoBWak1dpmr89Ye7iaD2UWwl5Dmn07Q9s27NGdywy0BsD1vDcFSgno3LUbVznkw/0hypbnOPxWKlBCao2c=</Modulus><Exponent>AQAB</Exponent></RSAKeyValue>";


We'll need to set up sender and receiver RSA providers in the Receiver class as well:



    private RSACryptoServiceProvider GetSenderCipher()
    {
    	RSACryptoServiceProvider sender = new RSACryptoServiceProvider();
    	sender.FromXmlString(_senderPublicKey);
    	return sender;
    }

    private RSACryptoServiceProvider GetReceiverCipher()
    {
    	RSACryptoServiceProvider sender = new RSACryptoServiceProvider();
    	sender.FromXmlString(_myRsaKeys);
    	return sender;
    }


We'll also need the same hash computation method that was employed on the Sender's side.



    private byte[] ComputeHashForMessage(byte[] cipherBytes)
    {
    	SHA1Managed alg = new SHA1Managed();
    	byte[] hash = alg.ComputeHash(cipherBytes);
    	return hash;
    }


I realise that this is a lot of duplication but imagine that the Sender and Receiver are different applications with no possibility to have a shared project. It's important they they set up details such as "SHA1″ in the ComputeHashForMessage methods for consistency. If one specifies SHA1 and the other one SHA256 then the process will fail of course so the two sides must agree on a common platform. Usually the person who needs to send a message to someone else will need to comply with what the receiver has set up on their side.

The RSAPKCS1SignatureDeformatter object has a VerifySignature method that is very useful in our case:



    private void VerifySignature(byte[] computedHash, byte[] signatureBytes)
    {
    	RSACryptoServiceProvider senderCipher = GetSenderCipher();
    	RSAPKCS1SignatureDeformatter deformatter = new RSAPKCS1SignatureDeformatter(senderCipher);
    	deformatter.SetHashAlgorithm("SHA1");
    	if (!deformatter.VerifySignature(computedHash, signatureBytes))
    	{
    		throw new ApplicationException("Signature did not match from sender");
    	}
    }


We'll need to pass in the computed hash of the cipher text received from the sender and the signature bytes so that we can verify the authenticity of the message. If this method returns true then the receiver will know that the message must have originated from someone with the correct private key. Otherwise there's reason to suspect that the message has been tampered with. In fact we need to recompute that hash of the cipher the same way the sender computed the hash. You can see that in the following public method in Receiver.cs:



    public string ExtractMessage(DigitalSignatureResult signatureResult)
    {
    	byte[] cipherTextBytes = Convert.FromBase64String(signatureResult.CipherText);
    	byte[] signatureBytes = Convert.FromBase64String(signatureResult.SignatureText);
    	byte[] recomputedHash = ComputeHashForMessage(cipherTextBytes);
    	VerifySignature(recomputedHash, signatureBytes);
    	byte[] plainTextBytes = GetReceiverCipher().Decrypt(cipherTextBytes, false);
    	return Encoding.UTF8.GetString(plainTextBytes);
    }


This is the inverse of the BuildSignedMessage of Sender.cs. We send in the DigitalSignatureResult object that resulted from the encryption process on the sender's side. The cipher and signature are converted back to byte arrays. Then the hash is recomputed and the signature is verified. If we get this far then we know that the message is authentic and we can decrypt the message.

The ExtractMessage can be called in Program.cs as follows:



    static void Main(string[] args)
    {
    	Sender sender = new Sender();
    	DigitalSignatureResult res = sender.BuildSignedMessage("Hello digital sig!");
    	Console.WriteLine(res.CipherText);
    	Console.WriteLine(res.SignatureText);

    	String decryptedText = new Receiver().ExtractMessage(res);
    	Console.WriteLine(decryptedText);

    	Console.ReadKey();
    }


Run the application and you should see that the signature is correct and the message is correctly decrypted by the receiver.

**Digital certificates**

We'll wrap up this post on a short intro into digital certificates. Certificates are another, more advanced way to make sure that a message is coming from the person who claims to be the sender of the message. They are often used in conjunction with web traffic. If you see https:// in the URL then you know that some certificate is involved. In that case it's not plain TCP that's used for data transmission but SSL – Secure Sockets Layer – and TLS – Transport Layer Security.

A digital certificate is basically a public key assigned to a particular entity, like what a registration number is to a car. The number plate can be used to verify the owner of the car. The driver presents an ID card and if the data on the ID card matches what the police registry says about the owner of the car then the "certificate" is accepted. Otherwise the driver may have stolen the car or the number plate.

The format that a digital certificate follows is usually an X509 format. It contains an issuer, a validity period, the public key and the issuer's signature. The digital signature helps us validate the public key of the sender. This process requires a certain level of trust in the issuer of the certificate. There are trusted issuers, so called Certificate Authorities (CAs), that the browsers have on their trusted issuers' list.

It is the CA that signs the public key. They basically tell you like "we attest that the public key in this certificate comes from the owner of the public key". The browser will have a list of those approved CAs and their public keys so they can validate signatures attached to these certificates.

So when the owner of the car presents an ID then it cannot just be any type of ID, it must have been issued by some authority. The authority attests through the ID that the person is really the one the ID claims them to be. This ties in well with [claims-based security][5].

TLS and SSL are responsible to do the encryption between web browsers and web servers to make sure that the messages are not tampered with. You can use self-signed certificates for testing but if you use it in production then the web browser will warn you that the certificate didn't come from one of the trusted CAs.

In other cases you may see that the browser warns that the certificate comes from a trusted CA, so the certificate is itself correct, but the domain name within the certificate is different from the domain name I'm visiting. It may be a sign that someone stole someone else's certificate.

You can view the list of posts on Security and Cryptography [here][6].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/10/28/hashing-algorithms-and-their-practical-usage-in-net-part-1/ "Hashing algorithms and their practical usage in .NET Part 1"
[2]: http://dotnetcodr.com/2013/11/11/introduction-to-asymmetric-encryption-in-net-cryptography/ "Introduction to asymmetric encryption in .NET cryptography"
[3]: http://dotnetcodr.com/2013/11/04/symmetric-encryption-algorithms-in-net-cryptography-part-1/ "Symmetric encryption algorithms in .NET cryptography part 1"
[4]: http://dotnetcodr.files.wordpress.com/2013/10/digitalsignaturesflow1.png?w=630&h=212
[5]: http://dotnetcodr.com/2013/02/11/introduction-to-claims-based-security-in-net4-5-with-c-part-1/ "Introduction to Claims based security in .NET4.5 with C# Part 1: the absolute basics"
[6]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
