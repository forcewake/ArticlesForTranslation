[Source](http://dotnetcodr.com/2013/11/04/symmetric-encryption-algorithms-in-net-cryptography-part-1/ "Permalink to Symmetric encryption algorithms in .NET cryptography part 1")

# Symmetric encryption algorithms in .NET cryptography part 1

**Introduction**

Symmetric encryption and decryption are probably what most people understand under "cryptography". A symmetric algorithm is one where the encryption and decryption key is the same and is shared among the parties involved in the encryption/decryption process.

Ideally only a small group of reliable people should have access to this key. Attackers typically use brute force to find the key in an attempt to decipher an encrypted message rather than defeating the algorithm itself. The key can vary in size so the attacker will need to know this first. Once they know this then they will try combinations of possible key characters.

A clear disadvantage with this approach is that distributing and storing keys in a safe and reliable manner is difficult. On the other hand symmetric algorithms are fast.

Here's the graphical representation of the algorithm at work:

![Symmetric algorithm flow][1]

We start with the plain text to be encrypted. The encryption algorithm runs using the common secret key. The plain text becomes cyphertext which is decrypted using the same secret key and algorithm.

A common algorithm is called AES – Advanced Encryption Standard. This has been a US government standard since 2001 when it replaced DES – Data Encryption Standard.

AES uses the so-called Rijndael algorithm with 128 bit block sizes. You can read about the details of the algorithm [here][2]. If you need to work with external partners that use disparate systems then AES is a good choice as it's widely supported in different encryption libraries in Java, Ruby, .NET, Objective C, etc.

In .NET all symmetric algorithms derive from the SymmetricAlgorithm abstract class. AES with Rijndael is not the only implementation available, here are some others:

* [Triple DES][3]: applies DES encryption 3 times
* [DES][4]: used to be the standard but there were successful approaches to break the key due to its small key size of 56 bits. It is not recommended to use this algorithm in new systems – use it only if you have to support backward-compatibility or legacy systems
* [RC2][5]: this was another competitor to replace DES

There are other symmetric algorithms out there, such as Mars, RC6, Serpent, TwoFish, but there's no .NET implementation of them at the time of writing this post. Make sure to pick AES/Rijndael as your first choice if you need to select a symmetric algorithm in your project.

**In .NET**

The .NET implementations if symmetric algorithm are called _block ciphers_. This only means that the encryption process takes the provided plain text and breaks it up into fixed size blocks, such as 128 mentioned above. The algorithm is performed on each individual block.

It is of course difficult to guarantee that the plain text will fit into those exact block boundaries – this is where **padding** enters the scene. Padding is data added to fill the last block to the correct size where if it doesn't fit the given bit size. We need to fill up this last block as the algorithm requires fix sized blocks.

Padding data can be a bunch of zeros. Another approach is called [PKCS7][6]. This one says that if there are e.g. 8 bits remaining to fill the block we'll use that number for each one of those spots. Yet another way to fill the missing spots is called ISO10126, which fills that block with random data. This is also the recommended approach as it provides more randomness in the process which is always a good way to put extra layers of protection on your encryption mechanism.

_Mode_ is also a factor in these algorithms. The most common one is called ECB which means that each block of the plain text will be encrypted independently of all the others. The recommended approach here is CBC – this means that a block will not only be encrypted but that a given block will be used as input to encrypt the subsequent block. So CBC adds some more randomness to the process which is always good.

In case you go with CBC then another term you'll need to be familiar with is the IV – Initialization Vector. The IV determines what kind of random data you're going to use for the first block simply because there's no block before the first block to be used as input. The IV is just some random data that will be used as input in the encryption of the first block. The IV doesn't need to be some secret string – it must be redistributed along with the cipher text to the receiver of our message. The only rule is not to reuse the IV to keep the randomness that comes with it.

**Demo**

Now it's time see some code after all the theory. Create a Console application in Visual Studio. We first want to set up the Rijndael encryption mechanism with its properties. Consider the following code:



    private RijndaelManaged CreateCipher()
    {
    	RijndaelManaged cipher = new RijndaelManaged();
    	cipher.KeySize = 256;
    	cipher.BlockSize = 128;
    	cipher.Padding = PaddingMode.ISO10126;
    	cipher.Mode = CipherMode.CBC;
    	byte[] key = HexToByteArray("B374A26A71490437AA024E4FADD5B497FDFF1A8EA6FF12F6FB65AF2720B59CCF");
    	cipher.Key = key;
    	return cipher;
    }


We instantiate a RijndaelManaged class which I guess you know what it might represent. We then set its key and block size. As mentioned above padding is set to ISO10126 and mode to CBC. The encryption key must be a valid AES key that you can find plenty of on the Internet. Note that you can set a lower key size, e.g 128, but make sure that the AES key is a valid 128 bit array in that case. Here I set the block size to 128 to be fully AES compliant, but generally the higher the key and block size the more secure the message. I've put the key as plain text into the code but it can be stored in the web.config file, in the database, it's up to you. However, this key must remain secret, so storage is not a trivial issue.

The HexToByteArray method may look familiar from the [previous][7] post on hashing:



    public byte[] HexToByteArray(string hexString)
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


In the below method we perform the actual encryption:



    public void Encrypt(string plainText)
    {
    	RijndaelManaged rijndael = CreateCipher();
    	Console.WriteLine(Convert.ToBase64String(rijndael.IV));
    	ICryptoTransform cryptoTransform = rijndael.CreateEncryptor();
    	byte[] plain = Encoding.UTF8.GetBytes(plainText);
    	byte[] cipherText = cryptoTransform.TransformFinalBlock(plain, 0, plain.Length);
    	Console.WriteLine(Convert.ToBase64String(cipherText));
    }


We let the IV and the cipher text be printed on the Console window. The IV is randomly generated by .NET, you don't need to set it yourself. We get hold of the encryptor using the CreateEncryptor method as it is implemented by the RijndaelManaged object. It implements the ICryptoTransform interface. We use this object to transform the plain text bytes to the AES-encrypted cipher text.

Run the application and inspect the IV and cipher text values. Save those values in class properties:



    public string IV { get; set; }
    public string CipherText { get; set; }


You can save these in the Encrypt method:



    CipherText = Convert.ToBase64String(cipherText);
    IV = Convert.ToBase64String(rijndael.IV);


The Decrypt method is the exact reverse of Encrypt():



    public void Decrypt(string iv, string cipherText)
    {
    	RijndaelManaged cipher = CreateCipher();
    	cipher.IV = Convert.FromBase64String(iv);
    	ICryptoTransform cryptTransform = cipher.CreateDecryptor();
    	byte[] cipherTextBytes = Convert.FromBase64String(cipherText);
    	byte[] plainText = cryptTransform.TransformFinalBlock(cipherTextBytes, 0, cipherTextBytes.Length);

    	Console.WriteLine(Encoding.UTF8.GetString(plainText));
    }


We'll need the cipher text and the IV we saved in the Encrypt method. This time we construct a Decryptor and decrypt the cipher text using the provided key and IV. It's important to set up the Rijndael managed object the same way as during the encryption process – key and block size, same mode etc. – otherwise the decryption will fail.

Test the entire cycle with some text, such as "Hello Crypto" and you'll see that it's correctly encrypted and decrypted.

You can view the list of posts on Security and Cryptography [here][8].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.files.wordpress.com/2013/11/symmetricalgorithmflow.png?w=630
[2]: http://en.wikipedia.org/wiki/Rijndael "Rijndael on Wikipedia"
[3]: http://en.wikipedia.org/wiki/Triple_DES "Triple DES on Wikipedia"
[4]: http://en.wikipedia.org/wiki/Data_Encryption_Standard "DES on Wikipedia"
[5]: http://en.wikipedia.org/wiki/RC2 "RC2 on Wikipedia"
[6]: http://en.wikipedia.org/wiki/PKCS7 "PKCS7 on Wikipedia"
[7]: http://dotnetcodr.com/2013/10/31/hashing-algorithms-and-their-practical-usage-in-net-part-2/ "Hashing algorithms and their practical usage in .NET Part 2"
[8]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
