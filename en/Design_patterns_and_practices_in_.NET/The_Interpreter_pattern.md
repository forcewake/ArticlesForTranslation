[Source](http://dotnetcodr.com/2013/06/03/design-patterns-and-practices-in-net-the-interpreter-pattern/ "Permalink to Design patterns and practices in .NET: the Interpreter pattern")

# Design patterns and practices in .NET: the Interpreter pattern

**Introduction**

The Interpreter pattern is somewhat different from the other design patterns. It's easily overlooked and many find it difficult to understand and apply. Formally the pattern is about handling languages based on a set of rules. So we have a language – any language – and its rules, i.e. the grammar. We also have an interpreter that takes the set of rules to interpret the sentences of the language. You will probably not use this pattern in your everyday programming work – except maybe if you work with robots that need to read some formal representation of an object.

Barcodes are real life examples of the pattern. Barcodes are usually not readable by humans because we don't know the rules of the barcode language. However, we can take an interpreter, i.e. a barcode reader which will use the barcode rules stored within it to tell us what kind of product we have just bought. The language is represented by the bars of the barcode. The grammar is represented by the numerical values of the bars.

What does all that have to do with programming??? We'll try to find out.

**Demo**

Open Visual Studio and create a new Console application. In our demo we'll simulate a sandwich builder. We'll create a sandwich language and we want to print the instructions for building the sandwich.

We want to represent a sandwich as follows. We have the top bread and the bottom bread with as many ingredients in between as you like. The ingredients can be condiments such as ketchup or mayo and other ingredients such as ham or vegetables. An additional rule is that the ingredients cannot be applied in any order. We first have one or more condiments, then some 'normal' ingredients such as chicken, followed by some more condiments.

We will have the following spices: mayo, mustard and ketchup. Ingredients include lettuce, tomato and chicken. Bread can be either white bread or wheat bread. The ingredients are grouped into a condiment list and an ingredient list. Each list can contain 0 or more elements. In an extreme case we can have a very plain sandwich with only the top and bottom bread.

The goal is to give instructions to a machine which will build sandwiches for us. We won't go overboard with our notations so that the result can be understood by a human, but you can replace the ingredients and bread types with any symbol you like.

The ingredients of our sandwich – bread, condiment, etc. – and the sandwich itself are the sentences or expressions in our sandwich language. This will be the first element in our programme, the IExpression interface:



    public interface IExpression
    	{
    		void Interpret(Context context);
    	}


Each expression has a meaning so it can be interpreted, hence the Interpret method. The sandwich means something. The list of condiments and ingredients mean something, they are all expressions. In order to understand the meaning of a sandwich we need to know the meaning of each ingredient and condiment. The Context class represents the context within which the Expression is interpreted. In this case it's a very simple class:



    public class Context
    	{
    		public string Output { get; set; }
    	}


We only use the context for our output.

Each condiment implements the ICondiment interface which is extremely simple:



    public interface ICondiment : IExpression { }


Each ingredient implements the IIngredient interface which again is very simple:



    public interface IIngredient : IExpression{}


Here come the condiment types, they should be easy to follow. The Interpret method appends the name of the condiment to the string output in each case. This is possible as a single condiment doesn't have any children so it can interpret itself:



    public class KetchupCondiment : ICondiment
    	{
    		public void Interpret(Context context)
    		{
    			context.Output += string.Format(" {0} ", "Ketchup");
    		}
    	}




    public class MayoCondiment : ICondiment
    	{
    		public void Interpret(Context context)
    		{
    			context.Output += string.Format(" {0} ", "Mayo");
    		}
    	}




    public class MustardCondiment : ICondiment
    	{
    		public void Interpret(Context context)
    		{
    			context.Output += string.Format(" {0} ", "Mustard");
    		}
    	}


The condiments are grouped in a Condiment list:



    public class CondimentList : IExpression
    	{
    		private readonly List<ICondiment> condiments;

    		public CondimentList(List<ICondiment> condiments)
    		{
    			this.condiments = condiments;
    		}

    		public void Interpret(Context context)
    		{
    			foreach (ICondiment condiment in condiments)
    				condiment.Interpret(context);
    		}
    	}


The Interpret method simply iterates through the members of the Condiment list and calls the interpret method on each.

Here come the ingredients which implement the Interpret method the same way:



    public class LettuceIngredient : IIngredient
    	{
    		public void Interpret(Context context)
    		{
    			context.Output += string.Format(" {0} ", "Lettuce");
    		}
    	}




    public class ChickenIngredient : IIngredient
    	{
    		public void Interpret(Context context)
    		{
    			context.Output += string.Format(" {0} ", "Chicken");
    		}
    	}


The ingredients are also grouped into an ingredient list:



    public class IngredientList : IExpression
    	{
    		private readonly List<IIngredient> ingredients;

    		public IngredientList(List<IIngredient> ingredients)
    		{
    			this.ingredients = ingredients;
    		}

    		public void Interpret(Context context)
    		{
    			foreach (IIngredient ingredient in ingredients)
    				ingredient.Interpret(context);
    		}
    	}


Now all we need is to represent the bread somehow:



    public interface IBread : IExpression{}


Here come the bread types:



    public class WheatBread : IBread
    	{
    		public void Interpret(Context context)
    		{
    			context.Output += string.Format(" {0} ", "Wheat-Bread");
    		}
    	}




    public class WhiteBread : IBread
    	{
    		public void Interpret(Context context)
    		{
    			context.Output += string.Format(" {0} ", "White-Bread");
    		}
    	}


Now we have all the elements ready to build our Sandwich class:



    public class Sandwhich : IExpression
    	{
    		private readonly IBread topBread;
    		private readonly CondimentList topCondiments;
    		private readonly IngredientList ingredients;
    		private readonly CondimentList bottomCondiments;
    		private readonly IBread bottomBread;

    		public Sandwhich(IBread topBread, CondimentList topCondiments, IngredientList ingredients, CondimentList bottomCondiments, IBread bottomBread)
    		{
    			this.topBread = topBread;
    			this.topCondiments = topCondiments;
    			this.ingredients = ingredients;
    			this.bottomCondiments = bottomCondiments;
    			this.bottomBread = bottomBread;
    		}

    		public void Interpret(Context context)
    		{
    			context.Output += "|";
    			topBread.Interpret(context);
    			context.Output += "|";
    			context.Output += "<--";
    			topCondiments.Interpret(context);
    			context.Output += "-";
    			ingredients.Interpret(context);
    			context.Output += "-";
    			bottomCondiments.Interpret(context);
    			context.Output += "-->";
    			context.Output += "|";
    			bottomBread.Interpret(context);
    			context.Output += "|";
    			Console.WriteLine(context.Output);
    		}
    	}


We build the sandwich using the 5 objects in the constructor: the top bread, top condiments, ingredients in the middle, bottom condiments and finally the bottom bread. The Interpret method builds our sandwich machine language:

* We start with a delimiter '|'
* Followed by the top bread interpretation
* Then comes the bread delimiter again '|'
* "<–"; indicates the start of the things in the sandwich
* Then come the top condiments interpretation
* Followed by a '-' delimiter
* Ingredients
* Againt followed by the '-' delimiter
* Bottom condiments
* The sandwich filling is then closed with "–>"
* The bottom bread is surrounded by pipe characters like the top bread

Note that every element in the sandwich, including the sandwich itself, can interpret itself. This is of course due to the fact that every element here is an expression, a sentence that has a meaning and can be interpreted.

The Interpret method in each implementation builds up the grammar of our sandwich language. The ultimate Interpret method in the Sandwich class builds up the sentences in sandwich language according to the rules of that grammar. We let each expression interpret itself – it is a lot easier to let each element do it instead of going through some complicated string operation and if-else statements trying build up our sentences. Not just that – we built our object model with our domain knowledge in mind so the solution is a lot more object oriented and reflects our business logic.

Let's see how this can be used by a client. Let's insert the following in the Main method:



    class Program
    	{
    		static void Main(string[] args)
    		{
    			Sandwhich sandwhich = new Sandwhich(
    				new WheatBread(),
    				new CondimentList(
    					new List<ICondiment> { new MayoCondiment(), new MustardCondiment() }),
    				new IngredientList(
    					new List<IIngredient> { new LettuceIngredient(), new ChickenIngredient() }),
    				new CondimentList(new List<ICondiment> { new KetchupCondiment() }),
    				new WheatBread());

    			sandwhich.Interpret(new Context());


    			Console.ReadKey();
    		}
    	}


We build a sandwich using the Sandwich constructor where we pass in each element of the sandwich. As the sandwich itself is also an expression we can call its interpret method to output the representation of the sandwich.

Run the application and you'll see our beautiful instructions to build a sandwich. Feel free to change the ingredients and check the output in the console.

View the list of posts on Architecture and Patterns [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
