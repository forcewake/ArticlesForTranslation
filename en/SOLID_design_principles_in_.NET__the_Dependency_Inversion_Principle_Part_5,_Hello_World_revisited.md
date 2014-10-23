[Source](http://dotnetcodr.com/2013/09/09/solid-design-principles-in-net-the-dependency-inversion-principle-part-5-hello-world-revisited/ "Permalink to SOLID design principles in .NET: the Dependency Inversion Principle Part 5, Hello World revisited")

# SOLID design principles in .NET: the Dependency Inversion Principle Part 5, Hello World revisited

**Introduction**

I realise that the [previous][1] post should have been the last one on the Dependency Inversion Principle but I decided to add one more, albeit a short one. It can be beneficial to look at one more example where we take a very easy starting point and expand it according to the guidelines we've looked at in this series.

**Demo**

The starting point of the exercise is the good old one-liner Hello World programme:



    static void Main(string[] args)
    {
    	Console.WriteLine("Hello world");
    }


Now that we're fluent in DIP and SOLID we can immediately see a couple of flaws with this solution:

* We can only write to the Console – if we want to write to a file then we'll have to modify Main
* We can only print Hello world to the console – we have to manually overwrite this bit of code if we want to print something else
* We cannot easily extend this application in a sense that it lacks any **seams** that we discussed before – what if we want to add logging or security checks?

Let's try to rectify these shortcomings. We'll tackle the problem of message printing first. The [Adapter][2] pattern solves the issue by abstracting away the Print operation in an interface:



    public interface ITextWriter
    {
    	void WriteText(string text);
    }


We can then implement the Console-based solution as follows:



    public class ConsoleTextWriter : ITextWriter
    {
    	public void WriteText(string text)
    	{
    		Console.WriteLine(text);
    	}
    }


Next let's find a solution for collecting what the text writer needs to output. We'll take the same approach and follow the adapter pattern:



    public interface IMessageCollector
    {
    	string CollectMessageFromUser();
    }


…with the corresponding Console-based implementation looking like this:



    public class ConsoleMessageCollector : IMessageCollector
    {
    	public string CollectMessageFromUser()
    	{
    		Console.Write("Type your message to the world: ");
    		return Console.ReadLine();
    	}
    }


These loose dependencies must be injected into another object, let's call it PublicMessage:



    public class PublicMessage
    {
    	private readonly IMessageCollector _messageCollector;
    	private readonly ITextWriter _textWriter;

    	public PublicMessage(IMessageCollector messageCollector, ITextWriter textWriter)
    	{
    		if (messageCollector == null) throw new ArgumentNullException("Message collector");
    		if (textWriter == null) throw new ArgumentNullException("Text writer");
    		_messageCollector = messageCollector;
    		_textWriter = textWriter;
    	}

    	public void Shout()
    	{
    		string message = _messageCollector.CollectMessageFromUser();
    		_textWriter.WriteText(message);
    	}
    }


You'll realise some of the most basic techniques we've looked at in this series: constructor injection, guard clause, readonly private backing fields.

We can use these objects from Main as follows:



    static void Main(string[] args)
    {
    	IMessageCollector messageCollector = new ConsoleMessageCollector();
    	ITextWriter textWriter = new ConsoleTextWriter();
    	PublicMessage publicMessage = new PublicMessage(messageCollector, textWriter);
    	publicMessage.Shout();

    	Console.ReadKey();
    }


Now we're free to inject any implementation of those interfaces: read from a database and print to file; read from a file and print to an email; read from the console and print to some web service. The PublicMessage class won't care, it's oblivious of the concrete implementations.

This solution is a lot more extensible. We can use the [decorator][3] pattern to add functionality to the text writer. Let's say we want to add logging to the text writer through the following interface:



    public interface ILogger
    {
    	void Log();
    }


We can have some default implementation:



    public class DefaultLogger : ILogger
    {
    	public void Log()
    	{
    		//implementation ignored
    	}
    }


We can wrap the text printing functionality within logging as follows:



    public class LogWriter : ITextWriter
    {
    	private readonly ILogger _logger;
    	private readonly ITextWriter _textWriter;

    	public LogWriter(ILogger logger, ITextWriter textWriter)
    	{
    		if (logger == null) throw new ArgumentNullException("Logger");
    		if (textWriter == null) throw new ArgumentNullException("TextWriter");
    		_logger = logger;
    		_textWriter = textWriter;
    	}

    	public void WriteText(string text)
    	{
    		_logger.Log();
    		_textWriter.WriteText(text);
    	}
    }


In Main you can have the following:



    static void Main(string[] args)
    {
    	IMessageCollector messageCollector = new ConsoleMessageCollector();
    	ITextWriter textWriter = new LogWriter(new DefaultLogger(), new ConsoleTextWriter());
    	PublicMessage publicMessage = new PublicMessage(messageCollector, textWriter);
    	publicMessage.Shout();

    	Console.ReadKey();
    }


Notice that we didn't have to do anything to PublicMessage. We passed in the interface dependencies as before and now we have the logging function included in message writing. Also, note that Main is tightly coupled to a range of objects, but it is acceptable in this case. We construct our objects in the entry point of the application, i.e. the _composition root_ which is the correct place to do that. We don't new up any dependencies within PublicMessage.

This was of course a very contrived example. We expanded the original code to a lot more complex solution with a lot higher overhead. However, real life applications, especially enterprise ones are infinitely more complicated where requirements change a lot. Customers are usually not sure what they want and wish to include new and updated features in the middle of the project. It's vital for you as a programmer to be able to react quickly. Enabling loose coupling like that will make your life easier by not having to change several seemingly unrelated parts of your code.

View the list of posts on Architecture and Patterns [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/09/05/solid-design-principles-in-net-the-dependency-inversion-principle-part-4-interception-and-conclusions/ "SOLID design principles in .NET: the Dependency Inversion Principle Part 4, Interception and conclusions"
[2]: http://dotnetcodr.com/2013/04/25/design-patterns-and-practices-in-net-the-adapter-pattern/ "Design patterns and practices in .NET: the Adapter Pattern"
[3]: http://dotnetcodr.com/2013/05/13/design-patterns-and-practices-in-net-the-decorator-design-pattern/ "Design patterns and practices in .NET: the Decorator design pattern"
[4]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
