[Source](http://dotnetcodr.com/2013/10/24/the-dont-repeat-yourself-dry-design-principle-in-net-part-3/ "Permalink to The Don’t-Repeat-Yourself (DRY) design principle in .NET Part 3")

# The Don’t-Repeat-Yourself (DRY) design principle in .NET Part 3

We'll finish up the DRY series with the **Repeated Execution Pattern**. This pattern can be used when you see similar chunks of code repeated at several places. Here we talk about code bits that are not 100% identical but follow the same pattern and can clearly be factored out.

Here's an example:



    static void Main(string[] args)
    {
    	Console.WriteLine("About to run the DoSomething method");
    	DoSomething();
    	Console.WriteLine("Finished running the DoSomething method");
    	Console.WriteLine("About to run the DoSomethingAgain method");
    	DoSomethingAgain();
    	Console.WriteLine("Finished running the DoSomethingAgain method");
    	Console.WriteLine("About to run the DoSomethingMore method");
    	DoSomethingMore();
    	Console.WriteLine("Finished running the DoSomethingMore method");
    	Console.WriteLine("About to run the DoSomethingExtraordinary method");
    	DoSomethingExtraordinary();
    	Console.WriteLine("Finished running the DoSomethingExtraordinary method");

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


We're simulating a simple logging function every time we run we run one of these "dosomething" methods. The pattern is clear: write a message to the console, carry out an action and write another message to the console. The actions have an identical void, parameterless signature. The logging message all have the same format, it's only the method name that varies. If this chain of actions continues to grow then we have to come back here and add the same type of logging messages. Also, if you later wish to change the logging message format then you'll have to do it in many different places.

The first step is to factor out a single console-action-console chunk to its own method:



    private static void ExecuteStep()
    {
    	Console.WriteLine("About to run the DoSomething method");
    	DoSomething();
    	Console.WriteLine("Finished running the DoSomething method");
    }


This is of course not good enough as the method is very rigid. It is hard coded to execute the first step only. We can vary the action to be executed using the Action object:



    private static void ExecuteStep(Action action)
    {
    	Console.WriteLine("About to run the DoSomething method");
    	action();
    	Console.WriteLine("Finished running the DoSomething method");
    }


We can call this method as follows:



    static void Main(string[] args)
    {
    	ExecuteStep(DoSomething);
    	ExecuteStep(DoSomethingAgain);
    	ExecuteStep(DoSomethingExtraordinary);
    	ExecuteStep(DoSomethingMore);
    	Console.ReadLine();
    }


Except that we're not logging the method names correctly. That's still hard coded to "DoSomething". That's easy to fix as the Action object has public properties to read off the method name:



    private static void ExecuteStep(Action action)
    {
    	string methodName = action.Method.Name;
    	Console.WriteLine("About to run the {0} method", methodName);
    	action();
    	Console.WriteLine("Finished running the {0} method", methodName);
    }


We're almost done. If you look at the Main method then the ExecuteStep(somemethod) is called 4 times. That is also a form of DRY-violation. Imagine that you have a long workflow, such as the steps in a chemical experiment. In that case you may need to repeat the call to ExecuteStep many times.

We can instead put the methods to be executed in a collection of actions:



    private static IEnumerable<Action> GetExecutionSteps()
    {
    	return new List<Action>()
    	{
    		DoSomething
    		, DoSomethingAgain
    		, DoSomethingExtraordinary
    		, DoSomethingMore
    	};
    }


You can use this from within Main as follows:



    static void Main(string[] args)
    {
    	IEnumerable<Action> actions = GetExecutionSteps();
    	foreach (Action action in actions)
    	{
    		ExecuteStep(action);
    	}
    	Console.ReadLine();
    }


Now it's not the responsibility of the Main method to define the steps to be executed. It only iterates through a loop and calls ExecuteStep for each action.

View the list of posts on Architecture and Patterns [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
