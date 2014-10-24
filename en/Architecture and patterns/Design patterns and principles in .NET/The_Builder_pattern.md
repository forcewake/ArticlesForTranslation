[Source](http://dotnetcodr.com/2013/07/22/design-patterns-and-practices-in-net-the-builder-pattern/ "Permalink to Design patterns and practices in .NET: the Builder pattern")

# Design patterns and practices in .NET: the Builder pattern

**Introduction**

According to the standard definition the Builder pattern separates the construction of a complex object from its representation so that the same construction process can create different representations. This is probably difficult to understand at first as is the case with most pattern definitions.

We have a complex object to start with. The construction is the process, i.e. the logic and its representation is the data, so we're separating the logic from the data. The reason why we separate them is that we want to be able to reuse this logic to work with a different set of data to build the same type of thing. The only difference is going to be the data. So the basic rule is simple: separate the data from the logic and reuse that logic.

Imagine a restaurant with no set menus. The process of ordering food can become a tedious dialog with the waiter: what kind of cheese would you like? What types of ingredient? What type of side-dish? This conversation can go on and on especially if the food you'd like is complex. And if you come back the following day to order then you'll go through the same dialog. It would be a lot easier to pick your lunch off of the menu, right?

You may have such dialogs in your code where your backend code asks the client for all the "ingredients", i.e. all the parameters necessary to construct an object. The next step you may take is to put all those parameters into the constructor of the object resulting in a large constructor. Within the constructor you might have extra code to construct other objects, check the validity and call other objects like services to finally construct the target object. This is actually a step towards the Builder pattern: we send in the ingredients to build an object. The construction logic of the object will be the same every time, but the data will be different. To apply this to the restaurant example we can imagine that the restaurant sells pasta-based dishes and customers come in with their ingredients list. They then give that list to the chef who will then "build" the pasta dish based on the ingredients, i.e. the parameters. The construction process will be the same every time, it is the "data" that is different. The customer will not tell the chef HOW to prepare the dish, only what should constitute that dish. The customer will not even know the steps of making the dish. The ingredients list replaces the dialog: whenever the chef has questions he or she can consult the list to get your inputs.

**Demo**

Let's start with a BAD example. Open Visual Studio and create a console application. We'll simulate a custom made graphing solution which requires a lot of parameters. Insert a new class called ClassyGraph:



    public class ClassyGraph
    	{
    		private List<double> _primarySeries;
    		private List<double> _secondarySeries;
    		private bool _showShadow;
    		private bool _largeGraphSize;
    		private double _offset;
    		private GraphType _graphType;
    		private GraphColourPackage _colourType;

    		public ClassyGraph(List<double> primarySeries, List<double> secondarySeries,
    			bool showShadow, bool largeGraphSize, double offset,
    			GraphType graphType, GraphColourPackage colourType)
    		{
    			_primarySeries = primarySeries;
    			_secondarySeries = secondarySeries;
    			_showShadow = showShadow;
    			_offset = offset;
    			_largeGraphSize = largeGraphSize;
    			_graphType = graphType;
    			_colourType = colourType;
    		}

    		public override string ToString()
    		{
    			StringBuilder sb = new StringBuilder();
    			sb.Append("Graph type: ").Append(_graphType).Append(Environment.NewLine)
    				.Append("Colour settings: ").Append(_colourType).Append(Environment.NewLine)
    				.Append("Show shadow: ").Append(_showShadow).Append(Environment.NewLine)
    				.Append("Is large: ").Append(_largeGraphSize).Append(Environment.NewLine)
    				.Append("Offset: ").Append(_offset).Append(Environment.NewLine);
    			sb.Append("Primary series: ");
    			foreach (double d in _primarySeries)
    			{
    				sb.Append(d).Append(", ");
    			}
    			sb.Append(Environment.NewLine).Append("Secondary series: ");
    			foreach (double d in _secondarySeries)
    			{
    				sb.Append(d).Append(", ");
    			}
    			return sb.ToString();
    		}
    	}

    	public enum GraphType
    	{
    		Bar
    		, Line
    		, Stack
    		, Pie
    	}

    	public enum GraphColourPackage
    	{
    		Sad
    		, Beautiful
    		, Ugly
    	}


We use this in the Main method as follows:



    static void Main(string[] args)
    {
    	ClassyGraph graph = new ClassyGraph(new List<double>() { 1, 2, 3, 4, 5 },
    		new List<double>() { 4, 5, 6, 7, 8 }, true, false,
    		1.2, GraphType.Bar, GraphColourPackage.Sad);
    	Console.WriteLine(graph);
    	Console.ReadKey();
    }


There's nothing complicated in this code I believe. However, it has some issues. We've got our nice ClassyGraph which has a big constructor with 7 parameters. You'll see a construction example in Main and with so many parameters it can be confusing which one is which, which ones are needed, which ones can be null or empty, what the order is etc. You can see such constructors in old code where the object just kept growing and new parameters were introduced to accommodate the changes. Also, such classes typically have multiple constructors that are chained together with some defaults. This can become a real mess: which constructor should we call? When?

Run the application and you'll see the output in the console, so at least the application performs what it's supposed to.

One immediate "solution" is to get rid of the constructor and change your private variables into public properties like this:



    public List<double> PrimarySeries{get;	set;}
    public List<double> SecondarySeries { get; set; }


Then the caller defines the attributes through the property setters:



    ClassyGraph cg = new ClassyGraph();
    cg.PrimarySeries = new List<double>() { 1, 2, 3, 4, 5 };
    cg.SecondarySeries = new List<double>() { 4, 5, 6, 7, 8 };


We at least solved the problem of the large constructor. If you call the property setters instead then the code is a bit cleaner as you actually see what "true" and "1.2" mean just by looking at the code. Here comes the revised code so far:



    public class ClassyGraph
    	{
    		public List<double> PrimarySeries{get;	set;}
    		public List<double> SecondarySeries { get; set; }
    		public bool ShowShadow{get;	set;}
    		public bool LargeGraphSize{get;	set;}
    		public double Offset{get;	set;}
    		public GraphType GraphType{get;	set;}
    		public GraphColourPackage ColourType { get; set; }

    		public override string ToString()
    		{
    			StringBuilder sb = new StringBuilder();
    			sb.Append("Graph type: ").Append(GraphType).Append(Environment.NewLine)
    				.Append("Colour settings: ").Append(ColourType).Append(Environment.NewLine)
    				.Append("Show shadow: ").Append(ShowShadow).Append(Environment.NewLine)
    				.Append("Is large: ").Append(LargeGraphSize).Append(Environment.NewLine)
    				.Append("Offset: ").Append(Offset).Append(Environment.NewLine);
    			sb.Append("Primary series: ");
    			foreach (double d in PrimarySeries)
    			{
    				sb.Append(d).Append(", ");
    			}
    			sb.Append(Environment.NewLine).Append("Secondary series: ");
    			foreach (double d in SecondarySeries)
    			{
    				sb.Append(d).Append(", ");
    			}
    			return sb.ToString();
    		}
    	}

    	public enum GraphType
    	{
    		Bar
    		,Line
    		,Stack
    		, Pie
    	}

    	public enum GraphColourPackage
    	{
    		Sad
    		,Beautiful
    		, Ugly
    	}




    static void Main(string[] args)
    {
            ClassyGraph cg = new ClassyGraph();
            cg.PrimarySeries = new List<double>() { 1, 2, 3, 4, 5 };
            cg.SecondarySeries = new List<double>() { 4, 5, 6, 7, 8 };
    	cg.ColourType = GraphColourPackage.Sad;
    	cg.GraphType = GraphType.Line;
    	cg.LargeGraphSize = false;
    	cg.Offset = 1.2;
    	cg.ShowShadow = true;
    	Console.WriteLine(cg);
    	Console.ReadKey();
    }


We have solved one problem but created another at the same time. The caller has to remember to call all these properties, keep track of which ones are already set, in what order etc. The code is certainly easier to read but you might forget to set some of the properties. Also, we are not controlling the order in which the properties are set. It may be an important part of the logic to set the series before the graph type or vice versa. With public properties this is difficult to achieve. The caller may miss some important property so that you have to introduce a lot of validation logic: "don't call this yet, call that before", "you forgot to set this and that" etc.

Let's see how a builder object can solve this. Add a new class called GraphBuilder to the project:



    public class GraphBuilder
    {
    	private ClassyGraph _classyGraph;

    	public ClassyGraph GetGraph()
    	{
    		return _classyGraph;
    	}

    	public void CreateGraph()
    	{
    		_classyGraph = new ClassyGraph();
    		_classyGraph.PrimarySeries = new List<double>() { 1, 2, 3, 4, 5 };
    		_classyGraph.SecondarySeries = new List<double>() { 4, 5, 6, 7, 8 };
    		_classyGraph.ColourType = GraphColourPackage.Sad;
    		_classyGraph.GraphType = GraphType.Line;
    		_classyGraph.LargeGraphSize = false;
    		_classyGraph.Offset = 1.2;
    		_classyGraph.ShowShadow = true;
    	}
    }


So now we encapsulate the graph creation in a builder where we can control the properties, their order etc. The caller doesn't need to remember the steps and worry about forgetting something.

You can think of this builder as a meal on a menu. One option is that you take a recipe about how to make a lasagne to a restaurant. You will then tell the chef to get some eggs, flour, oil, make the dough, so on and so forth. That's like setting the properties of an object one by one. Alternatively you can open the menu and tell the chef that you want a lasagne. He is a lasagne builder and will know how to make it. The GraphBuilder class builds a very specific type of graph – just like asking for a lasagne will get you a very specific type of dish, and not, say, a hamburger. You can call the builder as follows:



    class Program
    {
    	static void Main(string[] args)
    	{
    		GraphBuilder graphBuilder = new GraphBuilder();
    		graphBuilder.CreateGraph();
    		ClassyGraph cg = graphBuilder.GetGraph();
    		Console.WriteLine(cg);
    		Console.ReadKey();
    	}
    }


Run the app and you'll see the same output as before. Now we don't have all the calls to the property setters in Main. It's encapsulated in a special builder class which takes care of that. We get our specially made graph from the builder. Let's introduce a bit of creation logic into the builder to represent the steps of building the graph:



    private void BuildGraphType()
    {
    	_classyGraph.GraphType = GraphType.Line;
    	_classyGraph.Offset = 1.2;
    }

    private void ApplyAppearance()
    {
    	_classyGraph.ColourType = GraphColourPackage.Sad;
    	_classyGraph.LargeGraphSize = false;
    	_classyGraph.ShowShadow = true;
    }

    private void ApplySeries()
    {
    	_classyGraph.PrimarySeries = new List<double>() { 1, 2, 3, 4, 5 };
    	_classyGraph.SecondarySeries = new List<double>() { 4, 5, 6, 7, 8 };
    }

    private void InitialiseGraph()
    {
    	_classyGraph = new ClassyGraph();
    }


We'll call these methods from CreateGraph:



    public void CreateGraph()
    {
    	InitialiseGraph();
    	ApplySeries();
    	ApplyAppearance();
    	BuildGraphType();
    }


All we've done is a bit of refactoring but I think the CreateGraph method reads quite well now with clearly laid-out steps. You can always come back to the individual steps and modify the method bodies, which is analogous to changing a way a lasagne is prepared. You still get a lasagne but the steps the chef – the builder – follows may be different. You as the customer will probably not care. You can also introduce extra validation within each step where the validation result will depend on the previous step.

This is all well and good but we have a new problem now. Let's add another builder to the app:



    public class SpecialGraphBuilder
    {

    }


It's kind of empty, right? Well, we've just realised that we don't have a standard way to build a graph. One quick and dirty solution is to copy and paste the code we have in GraphBuilder.cs and modify it. You'll probably agree however that copy-paste is not a good approach in software engineering. Instead we have to force all types of graphbuilder to follow the same steps, i.e. to standardise the graph building logic. You'll probably see where we are going: abstraction, correct. So let's create an abstract class:



    public abstract class GraphBuilderBase
    {
           private ClassyGraph _classyGraph;

    	public ClassyGraph Graph
    	{
    		get
    		{
    			return _classyGraph;
    		}
    	}

    	public void InitialiseGraph()
    	{
    		_classyGraph = new ClassyGraph();
    	}

    	public abstract void ApplySeries();
    	public abstract void ApplyAppearance();
    	public abstract void BuildGraphType();
    }


The class initialisation will probably be a common step to all builders so there's no point in making that abstract. You'll recognise the abstract methods. Make GraphBuilder inherit from this base class:



    public class GraphBuilder : GraphBuilderBase
    {
    	public void CreateGraph()
    	{
    		InitialiseGraph();
    		ApplySeries();
    		ApplyAppearance();
    		BuildGraphType();
    	}

    	public override void BuildGraphType()
    	{
    		Graph.GraphType = GraphType.Line;
    		Graph.Offset = 1.2;
    	}

    	public override void ApplyAppearance()
    	{
    		Graph.ColourType = GraphColourPackage.Sad;
    		Graph.LargeGraphSize = false;
    		Graph.ShowShadow = true;
    	}

    	public override void ApplySeries()
    	{
    		Graph.PrimarySeries = new List<double>() { 1, 2, 3, 4, 5 };
    		Graph.SecondarySeries = new List<double>() { 4, 5, 6, 7, 8 };
    	}
    }


We can then have many different builders that all implement the abstract base. However, you'll notice that CreateGraph, i.e. the creation logic is still controlled by each implementing class, it is not part of the base. So the client cannot call GraphBuilderBase.CreateGraph. We'll separate out the creation itself to another class called GraphCreator. The main purpose of this class is to build a graph regardless of what type of graph we're building:



    public class GraphCreator
    {
    	private GraphBuilderBase _graphBuilder;
    	public GraphCreator(GraphBuilderBase graphBuilder)
    	{
    		_graphBuilder = graphBuilder;
    	}

    	public void CreateGraph()
    	{
    		_graphBuilder.InitialiseGraph();
    		_graphBuilder.ApplySeries();
    		_graphBuilder.ApplyAppearance();
    		_graphBuilder.BuildGraphType();
    	}

    	public ClassyGraph GetClassyGraph()
    	{
    		return _graphBuilder.Graph;
    	}
    }


We inject an abstract graph builder into the constructor and we delegate the creation logic to it. In terms of the pattern the GraphCreator object is called the **director**. The director uses the **builder** to build an object. The director won't care about the exact builder – a **concrete builder** – type it receives and how the abstract methods are implemented. Its purpose is to hold the creation logic of the ClassyGraph object – called the **product** – if it's given a "recipe", i.e. a builder.

The revised GraphBuilder class looks as follows:



    public class GraphBuilder : GraphBuilderBase
    {
    	public override void BuildGraphType()
    	{
    		Graph.GraphType = GraphType.Line;
    		Graph.Offset = 1.2;
    	}

    	public override void ApplyAppearance()
    	{
    		Graph.ColourType = GraphColourPackage.Sad;
    		Graph.LargeGraphSize = false;
    		Graph.ShowShadow = true;
    	}

    	public override void ApplySeries()
    	{
    		Graph.PrimarySeries = new List<double>() { 1, 2, 3, 4, 5 };
    		Graph.SecondarySeries = new List<double>() { 4, 5, 6, 7, 8 };
    	}
    }


This looks very much like an ingredient list, a list of steps with no other "noise": a data class with zero logic. It is the GraphCreator that controls the creation process. We can now implement the SpecialGraphBuilder class:



    public class SpecialGraphBuilder : GraphBuilderBase
    {
    	public override void ApplySeries()
    	{
    		Graph.GraphType = GraphType.Bar;
    		Graph.Offset = 1.0;
    	}

    	public override void ApplyAppearance()
    	{
    		Graph.ColourType = GraphColourPackage.Beautiful;
    		Graph.LargeGraphSize = true;
    		Graph.ShowShadow = true;
    	}

    	public override void BuildGraphType()
    	{
    		Graph.PrimarySeries = new List<double>() { 1, 2, 3, 8, 10 };
    		Graph.SecondarySeries = new List<double>() { 4, 5, 9, 3, 4 };
    	}
    }


We can use this new structure in Main as follows:



    static void Main(string[] args)
    {
    	GraphCreator graphCreator = new GraphCreator(new SpecialGraphBuilder());
    	graphCreator.CreateGraph();
    	ClassyGraph cg = graphCreator.GetClassyGraph();
    	Console.WriteLine(cg);
    	Console.ReadKey();
    }


It is the GraphCreator that will be responsible to create the graph, we only send in a "recipe" through its constructor. If you instead send in a new GraphBuilder object then the results will be according to that recipe.

**Wrap-up**

There you are, this is the full implementation of the builder pattern. If you want to create a new type of classy graph then just inherit from the builder base class and inject it into the graph creator constructor.

Note that the pattern is probably an overkill if you only have one concrete builder. The main goal here is to reuse the standardised steps that are generalised in the abstract builder. If you only have one set of data then there is not much value added by applying the pattern. Also, the product that the director is building – and holds an instance of – should be a complex one for the pattern to be useful. The point is that the object is not easy to build: it involves a lot of steps and parameters and validation logic. Creating a product with a single property should not be encapsulated in the builder pattern. In such a case you only increase the complexity of the code instead of reducing it.

Also, bear in mind that the product should always be the same type, i.e. we're building ClassyGraphs but their internal data is different. Resist the temptation to build a hierarchy of products. The builder is designed to fill in the data and not construct different types of object.

Before I finish up this post: the StringBuilder object in .NET is not an application of the builder pattern. It certainly builds up a string – a product – out of the elements provided in the Append method, but it lacks all the elements of the pattern we talked about in this post: there is no director and there are no multiple implementations of a builder base class or interface.

By the same token fluent APIs are not builders either. Example:



    new Graph().Type(GraphType.Line).Colour(Colour.Sad).Offset(1.2) etc.


You've probably seen such APIs where you can chain together the methods like that. It looks great and everything but it has the same problem as the StringBuilder class: it lacks the components necessary to constitute the correct implementation of the builder pattern. Also, there's no enforcing of any kind of creation process.

View the list of posts on Architecture and Patterns [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
