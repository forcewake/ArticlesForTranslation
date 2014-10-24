[Source](http://dotnetcodr.com/2013/10/31/hashing-algorithms-and-their-practical-usage-in-net-part-2/ "Permalink to Hashing algorithms and their practical usage in .NET Part 2")

# Hashing algorithms and their practical usage in .NET Part 2

In the [previous][1] post we looked at hashing in general and its usage in hashing query strings. In this post we'll look at some other areas of applicability.

**Passwords**

Storing passwords in clear text is obviously not a good idea. If an attacker gets hold of your user data then it should not be easy for them to retrieve user names and passwords. It's a well-established practice to store passwords in a hashed format as they cannot be deciphered. When your user logs onto your site the provided password is hashed using the same algorithm that was employed when the same user signed up with your site. The hashed values, i.e. not the plain text passwords will be compared upon log-in. Not even you as the database owner will be able to read the password selected by the user.

However, you must still be careful to protect the hashed passwords. As we mentioned in the previous post there are not too many hashing algorithms available. So if an attacker has access to the hashed passwords then they can simply get a list of the most common passwords – 'secret', 'password', 'passw0rd', etc. – from the internet, iterate through the available hashing algorithms and compare them to the hashed values in your user data store. They can even write a small piece of software that loops through an online dictionary and hashes those words. This is called a "dictionary attack". It's only a matter of time before they find a match. Therefore it may not be a blocking point that an attacker cannot reverse a hashed password.

Hashing passwords can be done in the same way as hashing a query string which we looked at previously. Here comes a reminder showing how to hash a plain text value using the SHA1 algorithm:



    byte[] textBytes = Encoding.UTF8.GetBytes("myPassword123");
    SHA1 hashAlgorithm = new SHA1Managed();
    byte[] hash = hashAlgorithm.ComputeHash(textBytes);
    string hashedString = BitConverter.ToString(data);


**Salted passwords**

So you see that storing hashed values like that may not be secure enough for the purposes of your application. An extra level of security comes in the form of salted passwords which means adding some unique random data to each password. This unique data is called 'salt'. So we take the password and the salt and hash their joint value. This increases the work required from the attacker to perform a dictionary attack against all passwords. They would need to compute salt values as well.

Again, keep in mind that if you don't protect your user data store then the attacker will gain access to the salt values as well and again it will be a matter of time before they find a match. It may take longer of course but they can concentrate on some specific valuable accounts, like the Administrator, instead of spending time trying to find the password of a 'normal' user.

This is how you can generate a salt:



    private static string GenerateSalt(int byteCount)
    {
    	RNGCryptoServiceProvider rng = new RNGCryptoServiceProvider();
    	byte[] salt = new byte[byteCount];
    	rng.GetBytes(salt);
    	return Convert.ToBase64String(salt);
    }


You'll need cryptographically strong salt values. One of the available strong random number generators in .NET is the RNGCryptoServiceProvider class. Here we're telling it that we need a 'byteCount'-length of random salt. The salt is then saved along with the computed hash in the database.

The extended ComputeHash method accepts this salt:



    public static string ComputeHash(string password, string salt)
    {
    	SHA512Managed hashAlg = new SHA512Managed();
    	byte[] hash = hashAlg.ComputeHash(Encoding.UTF8.GetBytes(password + salt));
    	return Convert.ToBase64String(hash);
    }


So when the user logs in then you must first find the user in the database for the given username. If the user exists, then you have to retrieve the salt generated at the sign-up phase. Finally take the password provided in the login form, compute the hash using the ComputeHash method which accepts the salt and compare the hashed values.

We can add an extra layer or security by providing entropy – a term we mentioned in the previous post. We can extend the ComputeHash method as follows:



    public static string ComputeHash(string password, string salt, string entropy)
    {
    	SHA512Managed hashAlg = new SHA512Managed();
    	byte[] hash = hashAlg.ComputeHash(Encoding.UTF8.GetBytes(password + salt + entropy));
    	return Convert.ToBase64String(hash);
    }


The entropy can be a constant that is common to all users, e.g. another cryptographically strong salt that is not stored in the user database. We can take a random value such as 'xl1k5ss5NTE='. The updated ComputeHash function can be called as follows:



    string salt = GenerateSalt(8);
    Console.WriteLine("Salt: " + salt);
    string password = "secret";
    string constant = "xl1k5ss5NTE=";
    string hashedPassword = ComputeHash(password, salt, constant);

    Console.WriteLine(hashedPassword);
    Console.ReadKey();


Alternatively you can use a Keyed Hash Algorithm which we also discussed in the previous post.

**Examples from .NET**

The .NET framework uses hashing in a couple of places, here are some examples:

* ViewState in ASP.NET web forms is hashed using the MAC address of the server so that an attacker cannot tamper with this value between the server and the client. This feature is turned on by default.
* ASP.NET Membership: if you have worked with the built-in Membership features of ASP.NET then you'll know that both the username and password are hashed and that a salt is saved along with the hashed password in the ASP.NET membership tables
* Minification and boundling: if you don't know what these terms mean then start [here][2]. Hashing in this case is used to differentiate between the versions of the bundled files, such as JS and CSS so that the browser 'knows' that there's a new version to be collected from the server and not use the cached one

We'll start looking at symmetric encryption in the next post.

You can view the list of posts on Security and Cryptography [here][3].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/10/28/hashing-algorithms-and-their-practical-usage-in-net-part-1/ "Hashing algorithms and their practical usage in .NET Part 1"
[2]: http://dotnetcodr.com/2013/01/14/web-optimisation-resource-bundling-and-minification-in-net4-5-mvc4-with-c/ "Web optimisation: resource bundling and minification in .NET4.5 MVC4 with C#"
[3]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
