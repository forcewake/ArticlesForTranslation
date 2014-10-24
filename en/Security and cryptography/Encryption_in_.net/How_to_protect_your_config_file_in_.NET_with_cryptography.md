[Source](http://dotnetcodr.com/2013/11/18/how-to-protect-your-config-file-in-net-with-cryptography/ "Permalink to How to protect your config file in .NET with cryptography")

# How to protect your config file in .NET with cryptography

**Introduction**

In the past several blog posts we've looked at various cryptography techniques in .NET: hashing, symmetric and asymmetric encryption and digital signatures. Yet another application of encryption in .NET is protecting the config file of your project. The goal here is that if an attacker gains access to the file system on the application server then they shouldn't be able to read any sensitive information from it.

The idea is that certain sensitive sections of the config file are encrypted and stored separate from the app.config or web.config files. These sections are then decrypted by ASP.NET automatically. We don't need to worry about writing complicated decryption code ourselves. In practice this technique is applied to the app settings and connection settings section of the configuration. We can use encryption keys either at the machine or the user level.

We can take one of two approaches: [DPAPI][1] which is built-into Windows or RSA, which we discussed in the posts on asymmetric algorithms and digital signatures.

**DPAPI**

With DPAPI built into Windows we'll use machine-specific keys so you don't need to worry about key storage issues. Key storage is managed by DPAPI in a secure way. This approach is very straightforward to use on a single server as the key is specific to that machine. However, you cannot move that encrypted web.config to another server in a web farm as the other servers will have other keys. The RSA-based solution is better suited for web farms which we'll look at shortly.

**DPAPI demo**

We'll use the aspnet_regiis tool to encrypt and decrypt a file. You'll need to have a project with either a web.config or app.config file. For this be meaningful insert a couple of app settings to the file, they don't need to make any real sense:



    <appSettings>
            <add key="secretKey" value="key" />
    	<add key="rsaKey" value="rsaKey" />
    	<add key="thisisgreat" value="true" />
    	<add key="codetosavetheworld" value="secret" />
    </appSettings>


The aspnet_regiis tool is usually located in one of these folders:

C:WindowsMicrosoft.NETFrameworkv4.0.30319
C:WindowsMicrosoft.NETFramework64v4.0.30319

Open a command prompt and navigate to the correct folder. Take note of the folder of your application where the config file is located. Enter the following command to encrypt the app settings section:

aspnet_regiis -pef "appSettings" C:path-to-folder-with-config-file -prov "DataProtectionConfigurationProvider"

DataProtectionConfigurationProvider is the DPAPI provider so we ask the tool to encrypt the appSettings section of the config file with DPAPI.

If the process succeeded then open the config file and you should see some funny-looking EncryptedData and CipherData nodes:



    <appSettings configProtectionProvider="DataProtectionConfigurationProvider">
      <EncryptedData>
       <CipherData>
        <CipherValue>AQAAANCMnd8BFdERjHoAwE/Cl+sBAAAARlLCV8A5OE2a9bhixy2JFwQAAAACAAAAAAADZgAAwAAAABAAAACtSLR5AiN84R+VKYn18+aPAAAAAASAAACgAAAAEAAAALygmbBS72xqElqrjz32sVmoAQAAFA+MvV+KYs/MZKMsCvYfapjetPvKgWjm/VNsyFkEaTx8A6z9PigAQdB/H64BOyTh5YVCcijhTrO8D6iU2LnXGwdhZeev4Rskk1AkliD+fLXvxfg7f9dnpjlI16694q60FrpDlIL9LQ9lqSUYgjsJvgZmfI48eXifGQ36HYTWYAWAjm2uQc6OUe2HlDdGr27nvZoVBZXy1Lho6JpiqRjD6VmbyH0TctysLCuzRrKx/8hPpIrrcvSVFcuNtFnbk2UGREt4Urh+TgMH8b5F2BM4jtN4jxblnyGbgwJKBMeheaypk2jctsKIkg6Ly8MhFDjgzJ/YqjV43ucnnt4HV13/TwHK1cvvfxQi5Xbs6vAqu4ZnaTfLdgecQ2JScTYbFoJKrti9quTKev3xOqtjj1vq2VC5mkTiSWxDfrVUH/nIZ3zXE6oGE1gQ9NXFZZ0iZ4casOe6Qq6u8hBLCTWVhdwlGftNAEbSILDdDMQUTCC8xeX2NHIukFRfCC9N6MoCqVFI6bkMj9ovDE//BqVGoLT/0VfmfzmDcXvV/PMJyLcuXnpHDJsPlgPyKxQAAADjr4S4XVr4grgNFiaLrr3waoMx3Q==</CipherValue>
       </CipherData>
      </EncryptedData>
     </appSettings>


Later when you try to extract some settings from the appsettings section in code then the decryption process will be performed for you so you can read values as before.

If you want to edit the app settings section then here's the decryption command:

aspnet_regiis -pdf "appSettings" C:path-to-folder-with-config-file

This will enable you to edit the appSettings section as normal.

**RSA**

If you want to reuse the same encrypted config file on multiple servers in a web farm then RSA is the best choice. The flow is the following from a high level:

* Generate an RSA key-pair on the first server
* Set up the local config provider for the customer key pair
* Use aspnet_regiis to encrypt the sensitive sections in the config file
* Grant access to the key for the Network Service account
* Export the key pair for reuse on the other servers of the farm
* On all other servers we import the key pair and grant Network Service access to it

**RSA demo**

In the post on [asymmetric encryption][2] we discussed how to generate and export RSA key pairs using aspnet_regiis. Make sure that the key-pair is exportable using the -exp flag. Let's say that the key container is called "MyCustomKeys":

aspnet_regiis -pc "MyCustomKeys" -exp

We need to set a reference to this container in the config file in a section called configProtectedData within the configuration root node. We also specify that we use RSA to do the file protection:



    <configProtectedData>
    	<providers>
    		<add keyContainerName="MyCustomKeys"
    				 useMachineContainer="true"
    				 name="CustomProvider"
    				 type="System.Configuration.RsaProtectedConfigurationProvider,System.Configuration, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a"/>
    	</providers>
    </configProtectedData>


Again use aspnet_regiis to encrypt the appSettings section referring to the config protection provider we named "CustomProvider" in the configProtectedData:

aspnet_regiis -pef "appSettings" c:path-to-folder-with-config-file -prov "CustomProvider"

You export those keys like this:

aspnet_regiis -px "CustomKeys" "c:myCustomKeys.xml" -pri

This creates an XML file we've seen before.

This is how you import those keys on the other servers:

aspnet_regiis -pi "CustomKeys" "c:myCustomKeys.xml"

Make sure to copy over the XML file of course.

Decrypting a config file is done the same way as before:

aspnet_regiis -pdf "appSettings" c:path-to-folder-with-config-file

Just like with DPAPI you don't need to set any decryption data yourself when you want to read some app settings in the config file, it's performed for you automatically.

We also mentioned that Network Service will need access to those keys. Guess which tool can be used to achieve that… …aspnet_regiis of course!

The following command will do just that:

aspnet_regiis -pa "MyCustomKeys" "NT AuthorityNetwork Service"

If this fails on one of the web farm machines then it might be that you need to grant access to the ID that the app-pool is running under:

aspnet_regiis -pa "CustomKeys" "IIS APPPOOL.NET v4.5″ -full

So in case decryption fails on the web servers then make sure you try both of these commands. You'll obviously need to look up the app-pool ID for your website.

**Protecting XML**

To wrap up this post here comes a little note on XML strings in particular. The config file may not not be the only XML-formatted data the you want to protect. You may work a lot with XML in you code and you may want to protect portions of those strings. You can learn about the System.Security.Cryptography.Xml namespace on [MSDN][3].

You can view the list of posts on Security and Cryptography [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://en.wikipedia.org/wiki/DPAPI "DPAPI in Wikipedia"
[2]: http://dotnetcodr.com/2013/11/11/introduction-to-asymmetric-encryption-in-net-cryptography/ "Introduction to asymmetric encryption in .NET cryptography"
[3]: http://msdn.microsoft.com/en-us/library/system.security.cryptography.xml.aspx "XML crypto on MSDN"
[4]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
