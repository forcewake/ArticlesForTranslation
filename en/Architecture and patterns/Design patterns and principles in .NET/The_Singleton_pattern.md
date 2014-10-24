[Source](http://dotnetcodr.com/2013/05/09/design-patterns-and-practices-in-net-the-singleton-pattern/ "Permalink to Design patterns and practices in .NET: the Singleton pattern")

# Design patterns and practices in .NET: the Singleton pattern

**Introduction**

The idea of the singleton pattern is that a certain class should only have one single instance in the application. All other classes that depend on it should all share the same instance instead of a new one. Usually singletons are only created when they are first needed – the same existing instance is returned upon subsequent calls. This is called lazy construction.

The singleton class is responsible for creating the new instance. It also needs to ensure that only this one instance is created and the existing instance is used in subsequent calls.

If you are sure that there should be only one instance of a class then a singleton pattern is certainly a possible solution. Note the following additional rules:

* The singleton class must be accessible to clients
* The class should not require parameters for its construction, as input parameters are a sign that multiple different versions of the class are created – this breaks the most important rule, i.e. that "there can be only one"

You may have seen public methods that take the following very simple form:



    SingletonClass instance = SingletonClass.GetInstance();


This almost certainly returns a singleton instance. The GetInstance() method is the only way a client can get hold of an instance, i.e. the client cannot call new SingletonClass(). This is due to a private constructor hidden within the SingletonClass implementation.

**Basic demo**

Open Visual Studio and create a blank solution called Singleton. Add a class library to the solution, remove Class1 and add a class called Singleton to it. The most simple implementation of the singleton pattern looks like this:



    public class Singleton
    	{
    		private static Singleton _instance;

    		private Singleton()
    		{
    		}

    		public static Singleton Instance
    		{
    			get
    			{
    				if (_instance == null)
    				{
    					_instance = new Singleton();
    				}
    				return _instance;
    			}
    		}
    	}


Inspect the code and you'll note the following characteristics:

* The class has a single static instance of itself
* The constructor is private
* The object instance is available through the static Instance property
* The property inspects the state of the private instance; if it's null then it creates a new instance otherwise just returns the existing one – lazy loading

Note that this implementation is not thread safe, so don't use this example in case the singleton class is accessed from multiple threads, e.g. in an ASP.NET web application. We'll see a thread-safe example soon.

It's perfectly acceptable that the Singleton class has multiple public methods. You can then access those methods as follows:



    Singleton.Instance.PerformWork();

    Singleton instance = Singleton.Instance;
    instance.PerformWork();

    //pass as parameter
    PerformSomeOtherWork(Singleton.Instance);


Add a new class to the class library called ThreadSafeSingleton with the following implementation:



    public class ThreadSafeSingleton
    	{
    		private ThreadSafeSingleton()
    		{
    		}

    		public static ThreadSafeSingleton Instance
    		{
    			get { return Nested.instance; }
    		}

    		private class Nested
    		{
    			static Nested()
    			{
    			}

    			internal static readonly ThreadSafeSingleton instance = new ThreadSafeSingleton();
    		}
    	}


This is the construction that is recommended for multithreaded environments, such as web applications. Note that it doesn't use locks which would slow down the performance. Note the following:

* As in the previous implementation we have a private constructor
* We also have a public static property to get hold of the singleton instance
* The implementation relies on the way type initialisers work in .NET
* The C# compiler will guarantee that a type initialiser is instantiated lazily if it is not marked with the **beforefieldinit** flag
* We can ensure this for the nested class Nested by including a static constructor
* Apparently there's no need for the static constructor but it does have an important role for the C# compiler
* Within the nested class we have a static ThreadSafeSingleton field
* This field is set to a new ThreadSafeSingleton statically when it's first referenced
* That reference only occurs in the Instance property getter which refers to the nested 'instance' field
* The first time the Instance getter is called a new ThreadSafeSingleton class is initialised using the 'instance' private field of the nested class
* Subsequent requests will simply receive the existing instance of this static field
* This way the "There can be only one" rule is enforced

**Drawbacks**

Singletons introduce tight coupling between the caller and the singleton making the software design more fragile. Singletons are also very difficult to test and are therefore often regarded as an anti-pattern by fierce advocates of testable code. In addition, singletons violate the 'S' in SOLID software design: the Single Responsibility Principle. Managing the object lifetime is not considered the responsibility of a class. This should be performed by a separate class.

However, using an Inversion-of-control (IoC) container we can avoid all of these drawbacks. The demo will show you a possible solution.

**Demo**

The demo will concentrate on an implementation of the pattern where we eliminate its drawbacks outlined above. This means that you should be somewhat familiar with dependency injection and IoC containers in general. You may have come across IoC containers such as StructureMap before. Even if you haven't met these concepts before, it may still be worthwhile to read on, you may learn some new things.

The demo application will simulate the simultaneous use of a file for file writes. The solution will make use of the .NET task library to perform file writes in a multithreaded fashion.

For each dependency we'll need an interface to eliminate the tight coupling mentioned before. Each dependency will be resolved using an IoC container called Unity.

Add a new Console app called FileLoggerAsync to the solution and set it as the startup project. Add the following package reference using NuGet:

![Unity package in NuGet][1]

The file writer will simply write a series of numbers to a text file. Add the following interface to the project:



    public interface INumberWriter
    	{
    		void WriteNumbersToFile(int max);
    	}


The parameter 'max' simply means the upper boundary of the series to save to disk.

We will also need an object that will perform the file writes. This will be our singleton class eventually, but it will be hidden behind an interface:



    public interface IFileLogger
    	{
    		void WriteLineToFile(string value);
    		void CloseFile();
    	}


We don't want the client to be concerned with the creation of the file logger so the creation will be delegated to an abstract factory – more on this topic [here][2]:



    public interface IFileLoggerFactory
    	{
    		IFileLogger Create();
    	}


Not much to comment there I presume.

We'll first implement the singleton file logger which implements the IFileLogger interface:



    public class FileLoggerLazySingleton : IFileLogger
    	{
    		private readonly TextWriter _logfile;
    		private const string filePath = @"c:logfile.txt";

    		private FileLoggerLazySingleton()
    		{
    			_logfile = GetFileStream();
    		}

    		public static FileLoggerLazySingleton Instance
    		{
    			get
    			{
    				return Nested.instance;
    			}
    		}
    		private class Nested
    		{
    			static Nested()
    			{
    			}

    			internal static readonly FileLoggerLazySingleton instance = new FileLoggerLazySingleton();
    		}

    		public void WriteLineToFile(string value)
    		{
    			_logfile.WriteLine(value);
    		}

    		public void CloseFile()
    		{
    			_logfile.Close();
    		}

    		private TextWriter GetFileStream()
    		{
    			return TextWriter.Synchronized(File.AppendText(filePath));
    		}
    	}


You'll recognise most of the code from the thread-safe singleton implementation shown above. The rest handles writing to a file to the specified file path. It is of course not good practice to hard-code the log file like that, but it'll do in this example. Feel free to change this value but make sure that the file exists.

Next we'll implement the IFileLoggerFactory interface:



    public class LazySingletonFileLoggerFactory : IFileLoggerFactory
    	{
    		public IFileLogger Create()
    		{
    			return FileLoggerLazySingleton.Instance;
    		}
    	}


It returns the singleton instance of the FileLoggerLazySingleton class. It's time to implement the INumberWriter interface:



    public class AsyncNumberWriter : INumberWriter
    	{
    		private readonly IFileLoggerFactory _fileLoggerFactory;

    		public AsyncNumberWriter(IFileLoggerFactory fileLoggerFactory)
    		{
    			_fileLoggerFactory = fileLoggerFactory;
    		}

    		public void WriteNumbersToFile(int max)
    		{
    			IFileLogger myLogger = null;
    			Action<int> logToFile = i =>
    			{
    				myLogger = _fileLoggerFactory.Create();
    				myLogger.WriteLineToFile("Ready for next number...");
    				myLogger.WriteLineToFile("Logged number: " + i);
    			};
    			Parallel.For(0, max, logToFile);
    			myLogger.CloseFile();
    		}
    	}


Let's see what's happening here. The class will need a factory to retrieve an instance of IFileLogger – the class will be oblivious to the actual implementation type. Hence we have eliminated the tight coupling problem mentioned above. Then we implement the WriteNumbersToFile method:

* Initially the IFileLogger object will be null
* Then we create an inline method using the Action object
* The Action represents a method which has accepts an integer parameter i
* In the method body we construct the file logger using the file logger factory
* Then we write a couple of things to the file

The Action will be used in a parallel loop. The loop is the parallel version of a standard for loop. The variable 'i' will not be incremented synchronously but in a parallel fashion. The variable will start at 0 and end with the max value. It is injected into the inline function defined by the Action object. So the method defined in the action object will be run in each loop of the Parallel.For construct. It is important to note that with each iteration the IFileLogger object is created using the IFileLoggerFactory object. Thus we simulate that multiple threads access the same file to write some lines.

Now we're ready to hook up the individual elements in Program.cs. Let's first set up the Unity container. Insert the following files to the project:

IoC.cs:



    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading.Tasks;
    using Microsoft.Practices.Unity;

    namespace FileLogger
    {
    	public static class IoC
    	{
    		private static IUnityContainer _container;

    		public static void Initialize(IUnityContainer container)
    		{
    			_container = container;
    		}

    		public static TBase Resolve<TBase>()
    		{
    			return _container.Resolve<TBase>();
    		}
    	}
    }


UnityDependencyResolver.cs:



    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading.Tasks;
    using Microsoft.Practices.Unity;

    namespace FileLogger
    {
    	public class UnityDependencyResolver
    	{
    		private static readonly IUnityContainer _container;
    		static UnityDependencyResolver()
    		{
    			_container = new UnityContainer();
    			IoC.Initialize(_container);
    		}

    		public void EnsureDependenciesRegistered()
    		{
    			_container.RegisterType<IFileLoggerFactory, LazySingletonFileLoggerFactory>();
    		}

    		public IUnityContainer Container
    		{
    			get
    			{
    				return _container;
    			}
    		}
    	}
    }


Don't worry if you don't understand what's going on here. The purpose of these classes is to initialise the Unity dependency container and make sure that when Unity encounters a dependency of type IFileLoggerFactory it creates a LazySingletonFileLoggerFactory ready to be injected.

The last missing bit is Program.cs:



    using System;
    using System.Collections.Generic;
    using System.IO;
    using System.Linq;
    using System.Text;
    using System.Threading.Tasks;
    using Microsoft.Practices.Unity;

    namespace FileLogger
    {
    	class Program
    	{
    		private static UnityDependencyResolver _dependencyResolver;
    		private static INumberWriter _numberWriter;

    		private static void RegisterTypes()
    		{
    			_dependencyResolver = new UnityDependencyResolver();
    			_dependencyResolver.EnsureDependenciesRegistered();
    			_dependencyResolver.Container.RegisterType<INumberWriter, AsyncNumberWriter>();

    		}

    		public static void Main(string[] args)
    		{
    			RegisterTypes();
    			_numberWriter = _dependencyResolver.Container.Resolve<INumberWriter>();
    			_numberWriter.WriteNumbersToFile(100);
                            Console.WriteLine("File write done.");
    			Console.ReadLine();
    		}
    	}
    }


In RegisterTypes we simply register another dependency: INumberWriter is resolved as the concrete type AsyncNumberWriter. In the Main method we then retrieve the number writer dependency and call its WriteNumbersToFile method. Recall that AsyncNumberWriter will then get hold of the file 100 times in each iteration and write a couple of lines to it without closing it at the end of each iteration.

Run the console app and you should see "File write done" almost instantly. The most expensive method, i.e. WriteNumbersToFile has to get hold of a new FileLogger instance only in the first iteration and will get the same instance over and over again in subsequent loops.

Inspect the contents of the file. You'll see that the iteration was indeed performed in a parallel way as the numbers do not follow any specific order, i.e. the outcome is not deterministic:

Ready for next number…
Ready for next number…
Logged number: 50
Ready for next number…
Logged number: 51
Logged number: 25
Ready for next number…
Ready for next number…
Logged number: 52
Logged number: 26
Ready for next number…
Logged number: 27
Ready for next number…
Ready for next number…
Logged number: 53
Ready for next number…
Logged number: 28
Logged number: 54
Ready for next number…
Logged number: 55
Ready for next number…

etc…

So, we have successfully implemented the singleton pattern in a way that eliminates its weaknesses: this solution is threadsafe, testable and loosely coupled.

View the list of posts on Architecture and Patterns [here][3].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.files.wordpress.com/2013/04/unityinnuget.png?w=630&h=100
[2]: http://dotnetcodr.com/2013/05/02/design-patterns-and-practices-in-net-the-factory-patterns-concrete-static-abstract/ "Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract"
[3]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
