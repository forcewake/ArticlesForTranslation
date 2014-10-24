[Source](http://dotnetcodr.com/2013/11/07/symmetric-algorithms-in-net-cryptography-part-2/ "Permalink to Symmetric algorithms in .NET cryptography part 2")

# Symmetric algorithms in .NET cryptography part 2

In the [previous][1] post we started looking at symmetric encryption algorithms in .NET. We also saw an example on how the encrypt and decrypt a string value. We'll wrap up the discussion with a couple of related topics.

**Generating secret keys**

You cannot just use any key in the encryption process, it must have a valid format with the specified bit size – usually either 128 or 256 bits. You can generate random bytes using the following function:



    public void GenerateRandomBytes(byte[] buffer)
    {
    	RNGCryptoServiceProvider rng = new RNGCryptoServiceProvider();
    	rng.GetBytes(buffer);
    }


We've seen the RNGCryptoServiceProvider object in the previous post. It helps generate cryptographically random numbers.

You can call this function as follows:



    public string GetValidEncryptionKey(int bitSize)
    {
    	byte[] key = new byte[bitSize / 8];
    	GenerateRandomBytes(key);
    	return BitConverter.ToString(key).Replace("-", string.Empty);
    }


You send in the bit size which must be converted to bytes, i.e. divided by 8. We generate the random bytes and convert them to a string. The BitConverter class puts a dash in between each hex value which we need to get rid of, hence the call to the Replace function.

This function can be invoked like this:



    string validKey = symmDemo.GetValidEncryptionKey(128);


**Importance of CBC and padding ISO10126**

Run the application we've been working on a couple of times and you'll see that the cipher text is always different even for the same string being encrypted. This is because we get a unique initialisation vector for each encryption and because we're in CBC mode. Now locate the CreateCipher method and modify the padding and mode values as follows:



    cipher.Padding = PaddingMode.Zeros;
    cipher.Mode = CipherMode.ECB;


Run the application again with the same string to be encrypted. You should see that the cipher text is always the same. We saw in [this][2] post how it can be dangerous to show the same cipher text for string values of limited variability. Hence always use ISO10126 and CBC to add more randomness to the encryption process.

**CryptoStream**

You can integrate encryption and decryption with .NET streams using the CryptoStream object. Say you want to chain together different crypto transforms and save the output in a file. Let's look at how this can be done:



    public void ChainStreamOperations(string plainText)
    {
    	byte[] plaintextBytes = Encoding.UTF8.GetBytes(plainText);
    	RijndaelManaged cipher = CreateCipher();
    	using (FileStream cipherFile = new FileStream(_fileName, FileMode.Create, FileAccess.Write))
    	{
    		ICryptoTransform base64CryptoTransform = new ToBase64Transform();
    		ICryptoTransform cipherTransform = cipher.CreateEncryptor();
    		using (CryptoStream firstCryptoStream = new CryptoStream(cipherFile, base64CryptoTransform, CryptoStreamMode.Write))
    		{
    			using (CryptoStream secondCryptoStream = new CryptoStream(firstCryptoStream, cipherTransform, CryptoStreamMode.Write))
    			{
    				secondCryptoStream.Write(plaintextBytes, 0, plaintextBytes.Length);
    			}
    		}
    	}
    }


…where _fileName is a local variable such as this:



    private string _fileName = @"c:tempresult.txt";


We first create a file stream in a normal .NET way. The purpose of the base 64 crypto transform is that in this case I want to encode the result of the encryption in this way. Then we get the Rijndael crypto transform object as we saw before. You see in the using statements that we create a series of streams. The final goal is to write out the result to a file. Then we have a CryptoStream that ties together the file stream and the base 64 crypto transform. Then on top of that we run the encryption itself. So when we call secondCryptoStream.Write with the plain text it's going to use the CryptoStream, perform the encryption, perform the base 64 encoding and finally the result is written out to the file. So it's very straightforward to do a number of encryptions for different plain text values. The chained streams will perform the same steps on all of them.

If you are used to using streams in .NET then this might be a natural way to encrypt a string and put the result in a file.

You can view the list of posts on Security and Cryptography [here][3].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/11/04/symmetric-encryption-algorithms-in-net-cryptography-part-1/ "Symmetric encryption algorithms in .NET cryptography part 1"
[2]: http://dotnetcodr.com/2013/10/28/hashing-algorithms-and-their-practical-usage-in-net-part-1/ "Hashing algorithms and their practical usage in .NET Part 1"
[3]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
