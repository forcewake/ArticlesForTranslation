[Source](http://dotnetcodr.com/2014/09/18/externalising-dependencies-with-dependency-injection-in-net-part-6-file-system/ "Permalink to Externalising dependencies with Dependency Injection in .NET part 6: file system")

# Externalising dependencies with Dependency Injection in .NET part 6: file system

**Introduction**

In the [previous][1] post we looked at logging with log4net and saw how easy it was to switch from one logging strategy to another. By now you probably understand why it can be advantageous to remove hard dependencies from your classes: flexibility, testability, SOLID and more.

The steps to factor out your hard dependencies to abstractions usually involve the following steps:

* Identify the hard dependencies: can the class be tested in isolation? Does the test result depend on an external object such as a web service? Can the implementation of the dependency change?
* Identify the tasks any reasonable implementation of the dependency should be able to perform: what should a caching system do? What should any decent logging framework be able to do?
* Build an abstraction – usually an interface – to represent those expected functions: this is so that the interface becomes as future-proof as possible. As noted before this is easier said than done as you don't know in advance what a future implementation might need. You might need to revisit your interface and add an extra method or an extra parameter. This can be alleviated if you work with objects as parameters to the interface functions, e.g. LoggingArguments, CachingArguments – you'll see what I mean in the next post where we'll take up emailing
* Inject the abstraction into the class that depends on it through one of the [Dependency Injection patterns][2] where constructor injection should be your default choice if you're uncertain
* The calling class will then inject a concrete implementation for the interface – alternatively you can use of the Inversion-of-control tools, like [StructureMap][3]

In this and the remaining posts of this series we won't be dealing with the Console app in the demo anymore. The purpose of the console app was to show the goal of abstractions and dependency injection through examples. We've seen enough of that so we can instead concentrate on building the infrastructure layer. So open the demo solution in VS let's add file system operations to Infrastructure.Common.

**File system**

.NET has an excellent built-in library for anything you'd like to do with files and directories. In the previous post on log4net we saw an example of checking if a file exists like this:



    FileInfo log4netSettingsFileInfo = new FileInfo(_contextService.GetContextualFullFilePath(_log4netConfigFileName));
    if (!log4netSettingsFileInfo.Exists)
    {
    	throw new ApplicationException(string.Concat("Log4net settings file ", _log4netConfigFileName, " not found."));
    }


You can have File.WriteAllText, File.ReadAllBytes, File.Copy etc. directly in your code and you may not think that it's a real dependency. It's admittedly very unlikely that you don't want to use the built-in features of .NET for file system operations and instead take some other library. So the argument of "flexibility" might not play a big role here.

However, [unit testing with TDD][4] shows that you shouldn't make the outcome of your test depend on external elements, such as the existence of a file if the method being tested wants in fact to perform some operation on a physical file. Instead, you should be able to declare the outcome of those operations through TDD tools such as Moq which is discussed in the TDD series referred to in the previous sentence. If you see that you must create a specific file before a test is run and delete it afterwards then it's a brittle unit test. Most real-life business applications are auto-tested by test runners in continuous integration (CI) systems such as [TeamCity][5] or [Jenkins][6]. In that case you'll need to create the same file on the CI server(s) as well so that the unit test passes.

Therefore it still makes sense to factor out the file related stuff from your consuming classes.

**The abstraction**

File system operations have many facets: reading, writing, updating, deleting, copying, creating files and much more. Therefore a single file system interface is going to be relatively large. Alternatively you can break out the functions to separate interfaces such as IFileReaderService, IFileWriterService, IFileInformationService etc. You can also have a separate interface for directory-specific operations such as creating a new folder or reading the drive name.

Here we'll start with out easy. Add a new folder called FileOperations to the Infrastructure.Common C# library. Insert an interface called IFileService:



    public interface IFileService
    {
    	bool FileExists(string fileFullPath);
    	long LastModifiedDateUnix(string fileFullPath);
    	string RetrieveContentsAsBase64String(string fileFullPath);
    	byte[] ReadContentsOfFile(string fileFullPath);
    	string GetFileName(string fullFilePath);
    	bool SaveFileContents(string fileFullPath, byte[] contents);
    	bool SaveFileContents(string fileFullPath, string base64Contents);
    	string GetFileExtension(string fileName);
    	bool DeleteFile(string fileFullPath);
    }


That should be enough for starters.

**The implementation**

We'll of course use the standard capabilities in .NET to implement the file operations. Add a new class called DefaultFileService to the FileOperations folder:



    public class DefaultFileService : IFileService
    {
    	public bool FileExists(string fileFullPath)
    	{
    		FileInfo fileInfo = new FileInfo(fileFullPath);
    		return fileInfo.Exists;
    	}

    	public long LastModifiedDateUnix(string fileFullPath)
    	{
    		FileInfo fileInfo = new FileInfo(fileFullPath);
    		if (fileInfo.Exists)
    		{
    			DateTime epoch = new DateTime(1970, 1, 1, 0, 0, 0);
    			TimeSpan timeSpan = fileInfo.LastWriteTimeUtc - epoch;
    			return Convert.ToInt64(timeSpan.TotalMilliseconds);
    		}

    		return -1;
    	}

    	public string RetrieveContentsAsBase64String(string fileFullPath)
    	{
    		byte[] contents = ReadContentsOfFile(fileFullPath);
    		if (contents != null)
    		{
    			return Convert.ToBase64String(contents);
    		}
    		return string.Empty;
    	}

    	public byte[] ReadContentsOfFile(string fileFullPath)
    	{
    		FileInfo fi = new FileInfo(fileFullPath);
    		if (fi.Exists)
    		{
    			return File.ReadAllBytes(fileFullPath);
    		}

    		return null;
    	}

    	public string GetFileName(string fullFilePath)
    	{
    		FileInfo fi = new FileInfo(fullFilePath);
    		return fi.Name;
    	}

    	public bool SaveFileContents(string fileFullPath, byte[] contents)
    	{
    		try
    		{
    			File.WriteAllBytes(fileFullPath, contents);
    			return true;
    		}
    		catch
    		{
    			return false;
    		}
    	}

    	public bool SaveFileContents(string fileFullPath, string base64Contents)
    	{
    		return SaveFileContents(fileFullPath, Convert.FromBase64String(base64Contents));
    	}

    	public string GetFileExtension(string fileName)
    	{
    		FileInfo fi = new FileInfo(fileName);
    		return fi.Extension;
    	}

    	public bool DeleteFile(string fileFullPath)
    	{
    		FileInfo fi = new FileInfo(fileFullPath);
    		if (fi.Exists)
    		{
    			File.Delete(fileFullPath);
    		}

    		return true;
    	}
    }


There you have it. Feel free to break out the file system functions to separate interfaces as suggested above.

The [next post][7] in this series will take up emailing.

View the list of posts on Architecture and Patterns [here][8].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/09/15/externalising-dependencies-with-dependency-injection-in-net-part-5-logging-with-log4net/ "Externalising dependencies with Dependency Injection in .NET part 5: logging with log4net"
[2]: http://dotnetcodr.com/2013/08/29/solid-design-principles-in-net-the-dependency-inversion-principle-part-2-di-patterns/ "SOLID design principles in .NET: the Dependency Inversion Principle Part 2, DI patterns"
[3]: http://docs.structuremap.net/ "StructureMap homepage"
[4]: http://dotnetcodr.com/test-driven-development/ "Test Driven Development"
[5]: http://www.jetbrains.com/teamcity/ "TeamCity homepage"
[6]: http://jenkins-ci.org/ "Jenkins homepage"
[7]: http://dotnetcodr.com/2014/09/22/externalising-dependencies-with-dependency-injection-in-net-part-7-emailing/ "Externalising dependencies with Dependency Injection in .NET part 7: emailing"
[8]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
