[Source](http://dotnetcodr.com/2013/10/17/the-dont-repeat-yourself-dry-design-principle-in-net-part-1/ "Permalink to The Don’t-Repeat-Yourself (DRY) design principle in .NET Part 1")

# The Don’t-Repeat-Yourself (DRY) design principle in .NET Part 1

**Introduction**

The idea behind the Don't-Repeat-Yourself (DRY) design principle is an easy one: a piece of logic should only be represented once in an application. In other words avoiding the repetition of any part of a system is a desirable trait. Code that is common to at least two different parts of your system should be factored out into a single location so that both parts call upon in. In plain English all this means that you should stop doing copy+paste right away in your software. Your motto should be the following:

Repetition is the root of all software evil.

Repetition does not only refer to writing the same piece of logic twice in two different places. It also refers to repetition in your processes – testing, debugging, deployment etc. Repetition in logic is often solved by abstractions or some common service classes whereas repetition in your process is tackled by automation. A lot of tedious processes can be automated by concepts from [Continuous Integration][1] and related automation software such as [TeamCity][2]. Unit testing can be automated by testing tools such as [nUnit][3]. You can read more on Test Driven Development and unit testing [here][4].

In this ahort series on DRY I'll concentrate on the 'logic' side of DRY. DRY is known by other names as well: Once and Only Once, and Duplication is Evil (DIE).

**Examples**

_Magic strings_

These are hard-coded strings that pop up at different places throughout your code: connection strings, formats, constants, like in the following code example:



    class Program
    {
    	static void Main(string[] args)
    	{
    		DoSomething();
    		DoSomethingAgain();
    		DoSomethingMore();
    		DoSomethingExtraordinary();
    		Console.ReadLine();
    	}

    	private static void DoSomething()
    	{
    		string address = "Stockholm, Sweden";
    		string format = "{0} is {1}, lives in {2}, age {3}";
    		Console.WriteLine(format, "Nils", "a good friend", address, 30);
    	}

    	private static void DoSomethingAgain()
    	{
    		string address = "Stockholm, Sweden";
    		string format = "{0} is {1}, lives in {2}, age {3}";
    		Console.WriteLine(format, "Christian", "a neighbour", address, 54);
    	}

    	private static void DoSomethingMore()
    	{
    		string address = "Stockholm, Sweden";
    		string format = "{0} is {1}, lives in {2}, age {3}";
    		Console.WriteLine(format, "Eva", "my daughter", address, 4);
    	}

    	private static void DoSomethingExtraordinary()
    	{
    		string address = "Stockholm, Sweden";
    		string format = "{0} is {1}, lives in {2}, age {3}";
    		Console.WriteLine(format, "Lilly", "my daughter's best friend", address, 4);
    	}
    }


This is obviously a very simplistic example but imagine that the methods are located in different sections or even different modules in your application. In case you want to change the address you'll need to find every hard-coded instance of the address. Likewise if you want to change the format you'll need to update it in several different places. We can put these values into a separate location, such as Constants.cs:



    public class Constants
    {
    	public static readonly string Address = "Stockholm, Sweden";
    	public static readonly string StandardFormat = "{0} is {1}, lives in {2}, age {3}";
    }


If you have a database connection string then that can be put into the configuration file app.config or web.config.

The updated programme looks as follows:



    class Program
    {
    	static void Main(string[] args)
    	{
    		DoSomething();
    		DoSomethingAgain();
    		DoSomethingMore();
    		DoSomethingExtraordinary();
    		Console.ReadLine();
    	}

    	private static void DoSomething()
    	{
    		string address = Constants.Address;
    		string format = Constants.StandardFormat;
    		Console.WriteLine(format, "Nils", "a good friend", address, 30);
    	}

    	private static void DoSomethingAgain()
    	{
    		string address = Constants.Address;
    		string format = Constants.StandardFormat;
    		Console.WriteLine(format, "Christian", "a neighbour", address, 54);
    	}

    	private static void DoSomethingMore()
    	{
    		string address = Constants.Address;
    		string format = Constants.StandardFormat;
    		Console.WriteLine(format, "Eva", "my daughter", address, 4);
    	}

    	private static void DoSomethingExtraordinary()
    	{
    		string address = Constants.Address;
    		string format = Constants.StandardFormat;
    		Console.WriteLine(format, "Lilly", "my daughter's best friend", address, 4);
    	}
    }


This is a step to the right direction. If we change the constants in Constants.cs then the change will be propagated through the application. However, we still repeat the following bit over and over again:



    string address = Constants.Address;
    string format = Constants.StandardFormat;


The VALUES of the constants are now stored in one place, but what if we change the location of our constants to a different file? Or decide to read them from a file or a database? Then again we'll need to revisit all these locations. We can move those variables to the class level and use them in our code as follows:



    class Program
    	{
    		private static string address = Constants.Address;
    		private static string format = Constants.StandardFormat;

    		static void Main(string[] args)
    		{
    			DoSomething();
    			DoSomethingAgain();
    			DoSomethingMore();
    			DoSomethingExtraordinary();
    			Console.ReadLine();
    		}

    		private static void DoSomething()
    		{
    			Console.WriteLine(format, "Nils", "a good friend", address, 30);
    		}

    		private static void DoSomethingAgain()
    		{
    			Console.WriteLine(format, "Christian", "a neighbour", address, 54);
    		}

    		private static void DoSomethingMore()
    		{
    			Console.WriteLine(format, "Eva", "my daughter", address, 4);
    		}

    		private static void DoSomethingExtraordinary()
    		{
    			Console.WriteLine(format, "Lilly", "my daughter's best friend", address, 4);
    		}
    	}


We've got rid of the magic string repetition, but we can do better. Notice that each method performs basically the same thing: write to the console. This is an example of duplicate logic. The data written to the console is very similar in each case, we can factor it out to another method:



    private static void WriteToConsole(string name, string description, int age)
    {
    	Console.WriteLine(format, name, description, address, age);
    }


The updated Program class looks as follows:



    class Program
    	{
    		private static string address = Constants.Address;
    		private static string format = Constants.StandardFormat;

    		static void Main(string[] args)
    		{
    			DoSomething();
    			DoSomethingAgain();
    			DoSomethingMore();
    			DoSomethingExtraordinary();
    			Console.ReadLine();
    		}

    		private static void DoSomething()
    		{
    			WriteToConsole("Nils", "a good friend", 30);
    		}

    		private static void DoSomethingAgain()
    		{
    			WriteToConsole("Christian", "a neighbour", 54);
    		}

    		private static void DoSomethingMore()
    		{
    			WriteToConsole("Eva", "my daughter", 4);
    		}

    		private static void DoSomethingExtraordinary()
    		{
    			WriteToConsole("Lilly", "my daughter's best friend", 4);
    		}

    		private static void WriteToConsole(string name, string description, int age)
    		{
    			Console.WriteLine(format, name, description, address, age);
    		}
    	}


_Magic numbers_

It's not only magic strings that can cause trouble but magic numbers as well. Imagine that you have the following class in your application:



    public class Employee
    {
    	public string Name { get; set; }
    	public int Age { get; set; }
    	public string Department { get; set; }
    }


We'll imitate a database lookup as follows:



    private static IEnumerable<Employee> GetEmployees()
    {
    	return new List<Employee>()
    	{
    		new Employee(){Age = 30, Department="IT", Name="John"}
    		, new Employee(){Age = 34, Department="Marketing", Name="Jane"}
    		, new Employee(){Age = 28, Department="Security", Name="Karen"}
    		, new Employee(){Age = 40, Department="Management", Name="Dave"}
    	};
    }


Notice the usage of the index 1 in the following method:



    private static void DoMagicInteger()
    {
    	List<Employee> employees = GetEmployees().ToList();
    	if (employees.Count > 0)
    	{
    		Console.WriteLine(string.Concat("Age: ", employees[1].Age, ", department: ", employees[1].Department
    			, ", name: ", employees[1].Name));
    	}
    }


So we only want to output the properties of the second employee in the list, i.e. the one with index 1. One issue is a conceptual one: why are we only interested in that particular employee? What's so special about him/her? This is not clear for anyone investigating the code. The second issue is that if we want to change the value of the index then we'll need to do it in three places. If this particular index is important elsewhere as well then we'll have to visit those places too and update the index.

We can solve both issues using the same simple techniques as in the previous example. Set a new constant in Constants.cs:



    public class Constants
    {
    	public static readonly string Address = "Stockholm, Sweden";
    	public static readonly string StandardFormat = "{0} is {1}, lives in {2}, age {3}";
    	public static readonly int IndexOfMyFavouriteEmployee = 1;
    }


Then introduce a new class level variable in Program.cs:



    private static int indexOfMyFavouriteEmployee = Constants.IndexOfMyFavouriteEmployee;


The updated DoMagicInteger() method looks as follows:



    private static void DoMagicInteger()
    {
    	List<Employee> employees = GetEmployees().ToList();
    	if (employees.Count > 0)
    	{
    		Employee favouriteEmployee = employees[indexOfMyFavouriteEmployee];
    		Console.WriteLine(string.Concat("Age: ", favouriteEmployee.Age,
    			", department: ", favouriteEmployee.Department
    			, ", name: ", favouriteEmployee.Name));
    	}
    }


View the list of posts on Architecture and Patterns [here][5].

### Like this:

Like Loading...

### _Related_

[1]: http://en.wikipedia.org/wiki/Continuous_integration "Continuous Integration on Wikipedia"
[2]: http://www.jetbrains.com/teamcity/ "Teamcity homepage"
[3]: http://www.nunit.org/ "NUnit homepage"
[4]: http://dotnetcodr.com/2013/03/25/test-driven-development-in-net-part-1-the-absolute-basics-of-red-green-refactor/ "Test Driven Development in .NET Part 1: the absolute basics of Red, Green, Refactor"
[5]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
