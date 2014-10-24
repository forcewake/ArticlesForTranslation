[Source](http://dotnetcodr.com/2013/10/28/hashing-algorithms-and-their-practical-usage-in-net-part-1/ "Permalink to Hashing algorithms and their practical usage in .NET Part 1")

# Hashing algorithms and their practical usage in .NET Part 1

**Introduction**

Cryptography is an exciting topic. It has two sides: some people try to hide some information and some other people try to uncover it. The information hiding, i.e. hiding some plain text, often happens using some encryption method. Then you have hackers, security experts and the like that try to decrypt the encrypted value.

Hashing is one such encryption algorithm. It converts some string input which can vary in size into a fixed length value, also called a "digest" or a "data fingerprint". It's called a fingerprint because the encrypted value can represent data that is much larger in size. You can take an entire book and create a short hash value out of its entire text.

Hashing is a one-way encryption method that cannot be deciphered (reversed) making it a standard way for storing passwords and usernames.

Note that in the demos I'll concentrate on showing functionality, I won't care about design practices and principles – that's beside the point and it's up to you to create the proper abstractions and services in your own solution. E.g. static helper classes shouldn't be the norm to create services, but they are very straightforward for demo purposes. You'll find many recommendations in among my glob posts on how to orginise your classes into loosely coupled elements.

I'll concentrate on various topics within cryptography in this and the next several blog posts:

* Hashing
* Symmetric encryption
* Asymmetric encryption
* Digital signatures

After that we'll take a look at a model application that makes use of asymmetric and symmetric encryption techniques.

**Hashing in .NET**

Common hash algorithms derive from the HashAlgorithm abstract base class. The most important concrete implementations out of the box include:

* MD5 which creates a 128 bit hash – this algorithm is not considered secure any more so use it only if you need to ensure backward compatibility
* SHA – Secure Hash Algorithm which comes in different forms: SHA1 (160bit), SHA256, SHA384 and SHA512 where the numbers correspond to the bit size of the hash created – the larger the value the harder it is to compromise the hashed value.
* KeyedHashAlgorithms (Message Authentication Code): HMACSHA and MACTripleDES

**Demo: plain text hashing**

Open Visual Studio and create a new Console application. Enter the following code in Main:



    string plainTextOne = "Hello Crypto";
    string plainTextTwo = "Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.";

    SHA512Managed sha512 = new SHA512Managed();

    byte[] hashedValueOfTextOne = sha512.ComputeHash(Encoding.UTF8.GetBytes(plainTextOne));
    byte[] hashedValueOfTextTwo = sha512.ComputeHash(Encoding.UTF8.GetBytes(plainTextTwo));

    string hexOfValueOne = BitConverter.ToString(hashedValueOfTextOne);
    string hexOfValueTwo = BitConverter.ToString(hashedValueOfTextTwo);

    Console.WriteLine(hexOfValueOne);
    Console.WriteLine(hexOfValueTwo);
    Console.ReadKey();


So we use the most secure version of the SHA family, the one that creates a 512 bit hash. We take a short and a long string to demonstrate that the hash size will be the same. The ComputeHash method will return a byte array with 64 elements, i.e. 64 bytes which is exactly 512 bits. This method accepts a byte array input hence the call to the GetBytes method. We then show a hex string representation of the byte arrays. You'll see in the console output that this string is grouped into 2 hex characters per byte. Each pair represents the byte value of 0-255. This is typical behaviour of the BitConverter class. There are other ways though to view a byte array in a string format.

You'll also see that the hashed value length is the same for both strings. I encourage you to change those strings, even by only a single letter, and you'll see very different hashed values every time.

Other hash algorithms in .NET work in a similar way:

* MD5CryptoServiceProvider
* SHA1Managed
* SHA256Managed
* SHA384Managed
* SHA512Managed
* MACTripleDES
* HMACMD5
* HMACSHA256
* HMACMD5
* HMACRIPEMD160
* HMACSHA512

These are all concrete classes that derive from HashAlgorithm and implement the ComputHash(byte[]) method. All of them are quite fast. Actually SHA512Managed is among the fastest among the implementations so you don't need to worry about execution speed if you go for this very strong algorithm.

**Demo: tamperproof query strings**

You must have seen websites that have URLs with some long and – at first sight – meaningless query string parameters. The goal is often to construct tamperproof query strings to protect the integrity of the parameters such as a customer id. The goal is make sure that this parameter has not been modified on the client side. Note that the actual ID will still be visible in the query string, e.g. /Customers.aspx?cid=21 but we extend this URL to include a special hash: /Customers.aspx?cid=21&hash=sdfhshmfuismfsdhmf. If the client modifies either the 'cid' or the 'h' parameter then the request will be rejected.

It's customary to use a Hash-based Message Authentication Code (HMAC), not one of the other hashing algorithms. It computes the hash of a query string when constructed on the server. When the client sends another request to the server we check that the query string was not modified by comparing it to the original hashed value. If they don't match then we know that the client or a man-in-the-middle has tampered with the query string values and reject the request.

We'll also use a key so that the attacker wouldn't be able to create his own valid hash – which is why HMAC is the better choice.

Create an ASP.NET web forms app in Visual Studio. Insert a hyperlink server control somewhere on Default.aspx:



    <asp:HyperLink ID="lnkAbout" runat="server">Go to about page</asp:HyperLink>


Add a new helper class called HashingHelper. Insert the following public constants:



    public static readonly string _hashQuerySeparator = "&h=";
    public static readonly string _hashKey = "C2CE6ACD";


The hash query separator stores the hash param identifier in the query string. The key will be used to stop an attacker from creating an own hash as mentioned above.

The following method will create the hashed query:



    public static string CreateTamperProofQueryString(string basicQueryString)
    {
    	return string.Concat(basicQueryString, _hashQuerySeparator, ComputeHash(basicQueryString));
    }


…where ComputeHash is a private method within this class:



    private static string ComputeHash(string basicQueryString)
    {
            byte[] textBytes = Encoding.UTF8.GetBytes(basicQueryString);
    	HMACSHA1 hashAlgorithm = new HMACSHA1(Conversions.HexToByteArray(_hashKey));
    	byte[] hash = hashAlgorithm.ComputeHash(textBytes);
    	return Conversions.ByteArrayToHex(hash);
    }


…where Conversions is a static utility class:



    public static class Conversions
    {
    	public static byte[] HexToByteArray(string hexString)
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

    	public static string ByteArrayToHex(byte[] data)
    	{
            	return BitConverter.ToString(data);
    	}
    }


This will look familiar from the previous demo. We're using a keyed algorithm – HMACSHA1 – which is based on SHA1 but can accept a key in the form of a byte array. Hence the conversion to a byte array in the Conversions helper.

In the Default.aspx.cs file add the following code:



    protected void Page_Load(object sender, EventArgs e)
    {
    	if (!IsPostBack)
    	{
    		lnkAbout.NavigateUrl = string.Concat("/About.aspx?", HashingHelper.CreateTamperProofQueryString("cid=21&pid=43"));
    	}
    }


Run the project and hover over the generated link. You should see in the bottom of the web browser that the cid and pid parameters have been extended with the extra 'h' hash parameter.

Add a label control somewhere on the About page:



    <asp:Label ID="lblQueryValue" runat="server" Text=""></asp:Label>


Put the following in the code behind:



    protected override void OnLoad(EventArgs e)
    {
    	HashingHelper.ValidateQueryString();
    	base.OnLoad(e);
    }


…where ValidateQueryString looks as follows:



    public static void ValidateQueryString()
    {
    	HttpRequest request = HttpContext.Current.Request;

    	if (request.QueryString.Count == 0)
    	{
    		return;
    	}

    	string queryString = request.Url.Query.TrimStart(new char[] { '?' });

    	string submittedHash = request.QueryString["h"];
    	if (submittedHash == null)
    	{
    		throw new ApplicationException("Querystring validation hash missing!");
    	}

    	int hashPos = queryString.IndexOf(_hashQuerySeparator);
    	queryString = queryString.Substring(0, hashPos);

    	if (submittedHash != ComputeHash(queryString))
    	{
    		throw new ApplicationException("Querystring hash value mismatch");
    	}
    }


We retrieve the current HTTP request and check its query string contents. If there are no query strings then there's nothing to validate. We then extract the entire query string bar the starting '?' character from the URL. We check if any hash parameter has been sent – if not then we throw an exception. We then need to recompute the hash of the cid and pid parameters and compare that value with what's been sent in the URL. If they don't match then we throw an exception.

Run the application and click the link on the Home page. You'll be directed to the About page. On the About page modify the URL in the web browser: change either query string parameter, reload the page and you should get an exception saying the hash values don't match.

Why did we use a keyed algorithm? Let's see first what may happen if you use a different one. In HashingHelper.ComputeHash comment out the line that creates a HMACSHA1 object and add a new line:



    SHA1 hashAlgorithm = new SHA1Managed();


Run the application and you'll see that it still works fine, the hashed value is generated and compared to the submitted value. Try changing the query parameter values and you should get the same exception as before. An attacker will look at this URL will know that this is some protected area. They will quickly figure out that this is some type of hash and they'll start going through the different hash algorithms available out there. There aren't that many – look at the list above with the ones built-into -.NET. It's a matter of a couple of lines to generate hash values in a console app:



    SHA1 hashAlgorithm = new SHA1Managed();
    //calculate hash of cid=123&pid=32
    //convert to string
    //call the web site


If it doesn't work then they'll try a SHA256 and then SHA512 etc. It doesn't take too long for an experienced hacker to iterate through the available implementations and eventually find the correct algorithm. They can then look at any protected value on the About page by replacing the query parameters and the hash they calculated using the little console application.

With the keyed algorithm we built in an extra blocking point for the attacker. So now the attacker will need to know the key as well as the hashing algorithm.

However, in some situations this may not be enough. If the query string parameter values are limited, such as a bit param 0 or 1 then even the keyed hash value will always only have two values: one for '0' and another one for '1'. An attacker that's watching the web traffic can after some time figure out what a '0' hash and a '1' hash looks like and they won't need to deal with any secret keys.

Therefore we need some way to differentiate between two requests with the same parameter. We need to add some randomness to our hashing algorithm – this is called adding some "entropy". In ASP.NET we can use the IP address of the client, their user agent, the session information and other elements that can differentiate one request from another. This is how you can add the unique session key to the data that's being hashed:



    HttpSessionState httpSession = HttpContext.Current.Session;
    basicQueryString += httpSession.SessionID;
    httpSession["HashIndex"] = 10;


Add this to the beginning of the HashingHelper.ComputeHash method. This will ensure that it's not only '0' or '1' being hashed but some other information as well that varies a lot and is difficult to get access to. You can test it with the current setup as well. Run the application and you'll get some hash in the query string. Stop the app, change the browser type in the Visual Studio app bar:

![Change browser type in visual studio][1]

Make sure it's different from what you had in the previous run. Click the link on the default page and you should see that the hash value is different for the same combination of cid and pid query string values.

You can view the list of posts on Security and Cryptography [here][2].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.files.wordpress.com/2013/09/changevisualstudiobrowsertype.png?w=630
[2]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
