[Source](http://dotnetcodr.com/2013/11/21/key-size-and-key-storage-in-net-cryptography/ "Permalink to Key size and key storage in .NET cryptography")

# Key size and key storage in .NET cryptography

**Key size considerations**

In the past several blog posts we talked a lot about how keys are used in symmetric and asymmetric cryptography. You've seen that they can come in different sizes – 128bit, 256bit etc.

There's a trade-off between performance and security when it comes to key sizes. The larger the size the harder it is for an attacker to break it by brute force. However, large size keys make the encryption and decryption process slower, so you'll need to consider just how important your data is. You'll probably want to protect your bank account information as much as possible. Also, a bank account number is a short string so using asymmetric encryption in that case will not make must difference to the speed. On the other hand the contents of an unpublished book may not require the same degree of security.

If you go for a very small key, say 2 bits, then it will be easy to break as there are not many different values:

00,01,10,11

As you move up the bit size then the amount of different possibilities grows exponentially. A 512-bit key would take an immense amount of time to break by brute force. So for max security always take the highest available key size in .NET, such as a 256-bit key in Rijndael managed AES symmetric encryption. Then you can take up the fight with other developers who may be more concerned with performance. However, with the CPUs that are available today increasing the bit size from 128 to 256 will only decrease the performance by some milliseconds.

In other words if you are working with data that must be secured then make sure to set a high priority on the security implementation of your app. You can always make adjustments to this implementation if you see that your app is not as responsive as you want it to be.

RSA keys are normally a lot larger than those in AES: 1024, 2048 or 4096 bits. These sizes provide an enormous amount of bit combinations. In practice we take either the 2048 or 4096 bit key size.

**Key storage**

We mentioned key storage here and there throughout the posts on cryptography. Here again you'll need to consider what type of blocking points you want to give to an attacker. In general never underestimate the tools available to an experienced hacker. What may seem impossible to extract to a normal user will be a piece of cake for a professional.

You may think that hardcoding your keys in your source files, such as…



    private string _myKey = "234dfgdsfw4rfdvg";


…will be impossible to read on the server as you only deploy the compiled code, the DLLs, right? Well, there are disassemblers out there, such as [ILDASM][1], that quickly uncover such values. There are many available tools that can do reflection on compiled code so if your keys are important then don't save them in plain text in your source files.

Also, hard-coded strings can be read by other people in your organisation. Normally your source code is stored in some central storage such as GitHub or SVN so anyone who has access to those accounts can pull the data from it. I'm not saying that you should distrust your co-workers but if the keys are used to encrypt some vital data then again, don't put them visible in a string.

We saw in the previous post how you can encrypt the appSettings section of the config file. You can then save the keys as app settings entries and encrypt the appSettings section. If you're concerned that somebody may get access to the config file on the server then this is a good option as they will only get access to jumble of encrypted characters that they cannot read. Here again, other developers will be able to pull out sensitive data programmatically:



    string key = ConfigurationManager.AppSettings["RsaKey"];


.NET will take care of the decryption automatically. We're again talking about distrusting your colleagues, but when it comes to security then you'll need to take on a "you never know" attitude if you're the one responsible for writing the security part of the project.

You may even store the key in the Registry instead of the config file, that's another option.

You can also have a remote service responsible for providing the correct key. In this scenario the key is not physically stored on the deployment server. The attacker who gained access to the server would need to be able to execute code from this server in order to get the key from the service.

With asymmetric encryption we can store the public key on the application server. It won't do any harm if an attacker could read it as it's only used for encryption. The private key can be stored on an intranet server that's protected by the company firewall.

**Other considerations**

For even more security make sure you refresh your keys periodically. Don't use the same keys for encryption and decryption for ever because if some hacker is watching this traffic then they will have a better starting point to guess what the keys are.

This point is important also because you may not even be aware that your cryptography has been broken. Some hacker may be happily sniffing your web traffic and extract all the data they need. They will most likely not tell you about it…

In the next post we'll start looking at a simple cryptography project to see how a sender and a receiver can communicate using encrypted messages with symmetric and asymmetric techniques we've seen up to now.

You can view the list of posts on Security and Cryptography [here][2].

### Like this:

Like Loading...

### _Related_

[1]: http://en.wikipedia.org/wiki/ILDASM "ILDASM on Wikipedia"
[2]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
