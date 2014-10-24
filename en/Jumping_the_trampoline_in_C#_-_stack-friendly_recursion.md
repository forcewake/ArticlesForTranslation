[Source](http://community.bartdesmet.net/blogs/bart/archive/2009/11/08/jumping-the-trampoline-in-c-stack-friendly-recursion.aspx "Permalink to Jumping the trampoline in C# – Stack-friendly recursion")

# Jumping the trampoline in C# – Stack-friendly recursion

Recursion is a widely known technique to decompose a problem in smaller "instances" of the same problem. For example, performing tree operations (e.g. in the context of data structures, user interfaces, hierarchical stores, XML, etc) can be expressed in terms of a navigation strategy over the tree where one performs the same operation to subtrees. A base case takes care of the algorithm's "bounding"; in case of tree operations that role is played by the leaf nodes of the tree.

Looking at mathematical definitions, one often finds recursive definitions, as well as more imperative style operations:

> **Imperative**
>
> ![clip_image002][1]
>
> **Recursive**
>
> ![clip_image004][2]

In here, the first definition lends itself nicely for implementation in an imperative language like C#, e.g. using a foreach-loop. Or, in a more declarative and functionally inspired style, one could write this one using LINQ's Aggregate operator (which really is a [catamorphism][3]):

> Func<int, int> fac = n => Enumerable.Range(1, n).Aggregate(1, (p, i) => p * i);

It's left as an exercise to the reader to define all other catamorphic operators in LINQ in terms of the Aggregate operator:

But this is not what we're here for today. Instead, we're going to focus on the recursive definition. We all know how to write this down in C#, as follows:

> int fac(int n)
{
    return n == 0 ? 1 : n * fac(n – 1);
}

Or, we could go for lambda expressions, requiring a little additional trick to make this work:

> Func<int, int> fac = null;
fac = n => n == 0 ? 1 : n * **fac**(n – 1);

The intermediate assignment with the null literal is required to satisfy the definite assignment rules at the point fac is used in the body of the lambda expression, as indicated in bold. What goes on here is quite interesting. When the compiler sees that the fac local variable is used inside a lambda's body, it's hoisted into a closure object. In other words, the local variable is not that local:

> var closure = new { fac = (Func<int, int>)null };
closure.fac = n => n == 0 ? 1 : n * closure.fac(n – 1);

Because of the heap-allocated nature of a closure, we can pass it around – including all of its "context" – to other locations in the code, potentially lower on the call stack. Let's not go there, but focus on the little null-assignment trick we had to play here. Turns out we can eliminate this.

 

Our two-step recursive definition of a lambda expression isn't too bad, but it should stimulate the reader's curiosity: can't we do a one-liner recursive definition instead? The following doesn't work for reasons alluded to above (try it yourself in your C# compiler):

> Func<int, int> fac = n => n == 0 ? 1 : n * **fac**(n – 1);

In languages like F#, a separate recursive definition variant of "let" exists:

> let rec fac n = if n = 0 then 1 else n * fac (n – 1)

An interesting (well, in my opinion at least) question is whether we can do something similar in C#, realizing "anonymous recursion". What's anonymous about it? Well, just having a single _expression_, without any variable assignments, that captures the intent of the recursive function. In other words, I'd like to be able to:

> ThisMethodExpectsUnaryFunctionIntToInt(/* I want to pass the factorial function here, defining it inline */)

To do this, in the factorial-function-defining expression we can't _refer _to a _local variable_, as we did in the C# (and the F#) fragment. Yet, we need to be able to _refer _to the function being defined to realize the recursion. If it ain't a local variable, and we need to refer to it, it ought to be a parameter to the lambda expression:

> **fac **=> n => n == 0 ? 1 : n * **fac**(n – 1)

Now we can start to think about types here. On the outer level, we have a function that takes in a "fac" and produces a function "n => …". The latter function, at the inner level, is a function that takes in "n" and produces "n == 0 ? …". That last part is simple to type: Func<int, int>. Back to the outer layer of the lambda onion, what has to be the type of fac? Well, we're _using_ fac inside the lambda expression, giving it an int and expecting an int back (see "fac(n – 1)"), hence it needs to be a Func<int, int> as well. Pasting all pieces together, the type of the thing above is:

> Func<Func<int, int>, Func<int, int>>

Or, in a full-typed form, the expression looks as follows:

> Func<Func<int, int>, Func<int, int>> f = (Func<int, int> fac) => ((int n) => n == 0 ? 1 : n * fac(n - 1))

But the "ThisMethodExpectsUnaryFunctionIntToInt" method expects, well, a Func<int, int>. We somehow need to shake off one of the seemingly redundant Func<int, int> parts of the resulting lambda expression. In fact, we need to _fix_ the lambda expression by eliminating the fac parameter, substituting it for the recursive function itself. So far, we can misuse the function above:

> f(**n => n * 2**)(5) –> 40

The bold part somehow needs to be the factorial function itself. This can be realized by means of a **fixpoint combinator**. From a typing perspective, it has the following meaning:

> Func<int, int> Fix(Func<Func<int, int>, Func<int, int>> f)

In other words, we should be able to:

> ThisMethodExpectsUnaryFunctionIntToInt(**Fix**(fac => n => n == 0 ? 1 : n * fac(n – 1)))

and leave the magic of realizing the recursion to Fix. This method can be define as follows (warning: danger for brain explosion):

> Func<T, R> Fix<T, R>(Func<Func<T, R>, Func<T, R>> f)
{
    FuncRec<T, R> fRec = r=> t => f(r(r))(t);
    return fRec(fRec);
}
>
> delegate Func<T, R> FuncRec<T, R>(FuncRec<T, R> f);

To see how the Fix method works, step through it, feeding it our factorial definition. The mechanics of it are less interesting in the context of this blog post, suffice to say it can be done. By the way, this Fix method is inspired by the [Y combinator][4], a fixpoint combinator created by lambda calculus high priests.

 

So far, so good. We have a generic fixpoint method called "Fix", allowing us to define the factorial function (amongst others of course) as follows:

> **Fix**(fac => n => n == 0 ? 1 : n * fac(n – 1))

Since factorial grows incredibly fast, our Int32-based calculation will overflow in no time, so feel free to use .NET 4's new BigInteger instead:

>
>     var factorial = Fix<BigInteger, BigInteger>(fac => n => n == 0 ? 1 : n * fac(n - 1));
>     factorial(10000);

Either way, let's see what happens under the debugger:

> ![image][5]

That doesn't look too good, does it? All the magic that Fix did was to realize the recursion, but we're still using recursive calls to compute the value. After some 5000 recursive calls, the call stack blew up. Clearly we need to do something if we are to avoid this, whilst staying in the comfort zone of recursive algorithm implementations. One such technique is a trampoline. But before we go there, it's worthwhile to talk about tail calls.

 

One of the inherent problems with this kind of recursion is the fact we need the result of the recursive call after we return from a recursive call. That seems logical but think about it for a while. When we're computing factorial of 5, we really are doing this:

> fac 5 =
          fac 4 =
                   fac 3 =
                           fac 2 =
                                   fac 1 =
                                   1
                           2 = **2 *** 1
                   6 = **3 *** 2
          24 = **4 *** 6
120 = **5 *** 24

What happens here is that after we return from the recursive call, we still have to carry out a multiplication. It's from this observation it follows that we need a call stack frame to keep track of the computation going on. One way to improve on the situation is by avoiding the need to do computation after a recursive call returns. This can achieved by accumulating the result of recursive calls, effectively carrying the result "forward" till the point we hit the base case. In essence, we're dragging along the partial computed result on every recursive call. In the case of factorial this accumulator will contain the partial multiplication, starting with a value of 1:

> fac 5 1 =
          fac 4 (5 * 1) =
                           fac 3 (4 * 5) = 
                                           fac 2 (3 * 20) =
                                                            fac 1 (2 * 60) =
                                                                             120

In here, the second parameter is the accumulated product so far. In the base case, we simply return the accumulated value. Now we don't need to do any more work after the recursive call returns. In other words, we've eliminated a "tail" of computation after a recursive call. Compilers can come to this insight and eliminate the recursive call. Below is a sample of an accumulating factorial definition in F#:

> ![image][6]

If we compile this code in the F# compiler (instead of just staring at F# interactive) and disassemble it, we get to see exactly this optimization carried out by the compiler:

> ![image][7]

In fact, this code is equivalent to the following piece of C#:

> int Fac(int n, int a)
{
    while (n > 1)
    {
        a *= n;
        n—;
    }
    return a;
}

Wonderful, isn't it? While we preserved a recursive definition, we really got the performance of an imperative loop-construct and are not exhausting the call stack in any way. The C# compiler on the other hand wouldn't figure this out. In what follows, we will be using this definition of factorial in combination with a trampoline to realize the same kind of stack-friendly recursion in C#.

 

One main characteristics of trampolines is that they bounce back. Jump on them and you'll be catapulted in the air because you're given a kinetic energy boost. While in the air you can make funny transformations (corresponding to the body of the recursive function as we shall see), but in the end you'll end up on the trampoline again. The whole cycle repeats till you run out of energy and just stay at rest on the trampoline. That state will correspond to the end of the recursion.

> ![image][8]

This all may sound very vague but things will become clear in a moment. The core idea of a trampoline is to throw a (recursive) function call on a trampoline, let it compute and have it land on the trampoline with a new function. It's important to see that both the function _and_ its arguments are jumping on there. Compare it to an acrobat that jumps on the trampoline and counts down every time he bounces. The function is the acrobat, the argument is the counter he maintains. When that counter reaches a base case, the _breaks_ from the bouncing by carefully landing next to the trampoline.

How can we realize such a thing is C# for functions of various arities? To grasp the concept, it helps to start from the simplest case, i.e. an Action with no arguments and – obviously, as we're talking about actions – no return value. We want to be able to write something like this, but without exhausting the call stack:

> void Motivate()
{
    Console.WriteLine("Go!");
    Motivate();
}

It goes without saying this can be achieved using a simple loop construct, but it's no surprise the base case of our investigation is trivial. Keep in mind most of my blog blog posts are about esoteric programming, so don't ask "Why?" just yet. To realize this recursion, we should start from the signature of a recursive action delegate. To get the trampoline behavior, a recursive action should not just return "void" but return another instance of itself to signal the trampoline what to call next. Compare it with the acrobat again: his capability (a "function" that can be called by the ring master initially: "start jumping!") to jump up returns the capability (a function again, to be called by the trampoline upon landing) to jump another time. This leads to the following (type-recursive!) signature:

>
>     delegate ActionRec ActionRec();

To write an anonymous recursive function, we use the same fixpoint technique as we saw before. In other words, the action is going to be passed as a parameter to a lambda expression, so that it can be called – to realize the recursion – inside the lambda expression's body. For example, our Motivation sample can be written like this:

>
>     Func<ActionRec, Func<ActionRec>> _motivate = motivate => () =>
>     {
>     Console.WriteLine("Go!");
>     return motivate();
>     };

Read the lambda expression from left to right to grasp it: given an ActionRec (which will be fixed to the while action itself further on by means of a Fix call), we're tasked with providing something the trampoline can call (with no arguments in our simple case) to run the next "bounce". This by itself should return an ActionRec to facilitate the further recursion. Apart from the return keyword and some lambda arrow tokens this looks quite similar to the typical recursive C# method shown earlier. To get the real recursive function with a regular Action signature, we'll call a fixpoint method called Fix:

>
>     Action Motivate = _motivate.Fix();

Now can can call Motivate and should see no StackOverflowException even though the function will run forever. The obvious question is how the Fix method works. Since we have no control over the Func delegate type used to define the non-fixed _motivate delegate, it ought to be an extension method. The signature therefore looks like this:

>
>     public static Action Fix(this Func<ActionRec, Func<ActionRec>> f)

Now let's reason about what the Fix method can do. Obviously it has to return an Action, which looks like "return () => { /* TODO */ };". Question is what the body of the action has to do. Well, it will have to call f at some point, passing in an ActionRec. This returns a function that, when called, will give us another of those ActionRec delegates. As long as a non-null delegate object is returned (null will be used later on as a way to break from the recursion), we can keep calling it in a loop. And that's where the stack-friendly nature comes from: we realize the recursion using a **loop**. Here's how it looks:

>
>     public static Action Fix(this Func<ActionRec, Func<ActionRec>> f)
>     {
>         return () =>
>         {
>             ActionRec a = null;
>             for (a = () => a; a != null; a = f(a)())
>                 ;
>         };
>     }

The last part of the for-statement is the most explanatory one: it calls the user-defined function with the recursive action, which returns an ActionRec. That gets called with the arguments, which for a plain vanilla action are empty, (). To get started, we use the definite assignment "closure over self" trick we saw at the very start of the post (starting with null):

> Func<int, int> _fac_ = _null_;
_fac_ = n => n == 0 ? 1 : n * **_fac_**(n – 1);

That's essentially the fixpoint part of Fix. It will definitely help the reader to trace through the code for the Motivate sample step by step. You'll see how code in the Fix trampoline will get interleaved with calls to your own delegate:

> ![image][9]

The second frame in the callstack is the trampoline that lives in the anonymous action inside the Fix method. We're currently broken in the debugger inside the recursive call to our own delegate. Notice though the call stack's depth is constant at two frames (ignoring Main), even though we've already made calls. Contrast this to the original C#-style Motive definition, which would already have grown the stack to 10 frames:

> ![image][10]

The way to break from a trampoline-based recursion is by returning null from the trampolined function. While that works, we want to add a bit of syntactical surface to it for reasons that will become apparent later (hint: we'll need a place to stick return values on). So, we define a trivial Break extension method that will return a null-valued ActionRec:

>
>     public static ActionRec Break(this ActionRec a) { return null; }

Based on certain conditions we can now decide to break out of the recursion, simply by calling Break on the ActionRec passed in. For example, we could capture a local variable from the outer scope, to act as a counter:

>
>     Console.WriteLine("Action of arity 0");
>     {
>         int i = 0;
>         Func<ActionRec, Func<ActionRec>> f = a => () => { Console.WriteLine("Go! " + i++); return i < 10 ? a() : a.Break(); };
>         f.Fix()();
>     }
>     Console.WriteLine();

This will just print 10 Go! messages. Notice I've omitted an intermediate variable for the f.Fix() result and call the resulting Action delegate in one go.

 

To do something more useful, we want to support higher arities for recursive Action and Func delegates. Let's start by looking at the Action delegates since we've already looked at the simplest case of a recursive Action delegate before. Below is a sample of a recursive Action delegate with one parameter, printing the powers of two with exponents 0 to 9:

>
>     Console.WriteLine("Action of arity 1");
>     {
>         int i = 0;
>         Func<ActionRec<int>, Func<int, ActionRec<int>>> f = a => x => { Console.WriteLine("2^" + i++ + " = " + x); return i < 10 ? a(x * 2) : a.Break(); };
>         f.Fix()(1);
>     }
>     Console.WriteLine();

Notice we're cheating a bit by using a captured outer local variable to restrict the number of recursive calls. It's left as an exercise to the reader to define another such recursive function where the input parameter is used to represent the "to" argument, i.e. specifying the largest exponent to calculate a power of two for.

In here, the ActionRec<T> delegate represents a recursive action delegate with one generic argument:

>
>     delegate ActionRec<T> ActionRec<T>(T t);

In order to define the recursive action that produces the powers of two, we use a regular function that maps such a recursive action onto a function that can create a new one of those, given an int as the input. Changing the names of the parameters may help to grasp this:

>
>     Console.WriteLine("Action of arity 1");
>     {
>         int i = 0;
>         Func<ActionRec<int>, Func<int, ActionRec<int>>> _printPowersOfTwo = printPowersOfTwo => x =>
>     {
>     Console.WriteLine("2^" + i++ + " = " + x);
>     return i < 10 ? printPowersOfTwo(x * 2) : printPowersOfTwo.Break();
>     };
>         Action<int> PrintPowersOfTwo = _printPowersOfTwo.Fix();
>         PrintPowersOfTwo(1);
>     }
>     Console.WriteLine();

Now the indented block reads like "void printPowersOfTo(int x) { … }". The Fix method's trampoline is a bit more tricky than the one we saw before, as it needs to deal with the one parameter that has to be fed to the called delegate. There's a bit of voodoo here since the argument can change every time one makes a recursive call. After all, it's an argument to the delegate. In the sample above, printPowersOfTwo is fed consecutive powers of two. The little hack is shown below:

>
>     public static Action<T> Fix<T>(this Func<ActionRec<T>, Func<T, ActionRec<T>>> f)
>     {
>         return t =>
>         {
>             ActionRec<T> a = null;
>             for (a = **t_** => { **t = t_;** return a; }; a != null; a = f(a)(t))
>                 ;
>         };
>     }

Trace through this for the PrintPowersOfTwo sample, where t starts as value 1. Clearly, a is non-null at that point (due to the assigned lambda expression in the initializer of the for-loop), so we get to call f with that action and argument 1. Now we're in our code where 1 got assigned to parameter x, causing 2^0 = 1 to be printed to the screen. Ultimately this results in a call to printPowersOfTwo with argument 2. This happens on the action delegate "a" created by the first iteration of the trampoline's for-loop:

> a = **t_** => { **t = t_;** return a; }

So, as a side-effect of calling this delegate, the local variable t got assigned the value 2. And the returned object from this call, "a", gets assigned in the trampoline's driver loop to the local variable "a". In the next iteration, 2 will be fed to the recursive delegate. And so on:

> ![image][11]

Increasing the number of arguments with one more is done in a completely similar way:

>
>     Console.WriteLine("Action of arity 2");
>     {
>         int i = 0;
>         Func<ActionRec<int, int>, Func<int, int, ActionRec<int, int>>> f = a => (x, y) => { Console.WriteLine("2^" + x + " = " + y); return ++i < 10 ? a(x + 1, y * 2) : a.Break(); };
>         f.Fix()(0, 1);
>     }
>     Console.WriteLine();

Where the new ActionRec delegate takes two generic parameters:

>
>     delegate ActionRec<T1, T2> ActionRec<T1, T2>(T1 t1, T2 t2);

In this sample we use two input parameters, on to represent the exponents and one to accumulate the powers of two. The Fix method now has to deal with two input parameters that need to be captured upon recursive calls. This is achieved as follows:

>
>     public static Action<T1, T2> Fix<T1, T2>(this Func<ActionRec<T1, T2>, Func<T1, T2, ActionRec<T1, T2>>> f)
>     {
>         return (t1, t2) =>
>         {
>             ActionRec<T1, T2> a = null;
>             for (a = (t1_, t2_) => { t1 = t1_; t2 = t2_; return a; }; a != null; a = f(a)(t1, t2))
>                 ;
>         };
>     }

What we haven't mentioned over and over again is the definition of the Break method that returns null to signal to break from the recursion. Here they are for completeness:

>
>     public static ActionRec<T> Break<T>(this ActionRec<T> a) { return null; }
>     public static ActionRec<T1, T2> Break<T1, T2>(this ActionRec<T1, T2> a) { return null; }

Below is an insight-providing screenshot illustrating the way recursion happens:

> ![image][12]

In Main, we called the fixed delegate with arguments 0 and 1. This caused us to enter the outermost lambda expression in Fix with t1 and t2 respectively set to 0 and 1. This is the second frame on the call stack (read from the bottom). The for-loop has proceeded to the first call of its update expression, resulting in a call to f with argument a and a subsequent invocation on the result with arguments 0 and 1. As a result, our lambda expression, lexically nested in the Main method, got called as observed by the third frame on the call stack, with x and y respectively set to 0 and 1. Here the recursive call happens by invoking the a delegate with arguments 1 (x + 1) and 2 (y * 2). Finally, this put us back in the trampoline where those two values will be captured in t1 and t2, and that's where the debugger is currently sitting.

Moving on from here, we'll back out of the trampoline and return the result of the apparent recursive call on "a" from lambda "f" in Main. This by itself puts us back in the driver for-loop, where "a" will be tested for null (which it isn't yet) and the whole cycle starts again. This illustrates the key essence of the trampoline: instead of having the user directly causing a recursive call, callbacks to the trampoline code cause it to capture enough state information to make the call later on. This effectively flattens recursive calls into the for-loop. What we lost is the ability to do work after the recursive call returns (something we could work around by getting into the land of continuations).

 

The essential tricks to deal with input parameters have been explored above. However, Func delegate types have one more property we haven't investigated just yet: the ability to return a value. We've seen the Break method before, but for Action delegates it doesn't do anything but returning null. In case of recursive Func types, we'll have to do something in addition to this, in order to return an object to the caller.

Let's get started by defining the FuncRec delegate types. Again, those are mirrored after the regular Func delegates, but we have to sacrifice the return type position for a FuncRec:

>
>     delegate FuncRec<R> FuncRec<R>();
>     delegate FuncRec<T, R> FuncRec<T, R>(T t);
>     delegate FuncRec<T1, T2, R> FuncRec<T1, T2, R>(T1 t1, T2 t2);

Returning from a recursive FuncRec delegate will be done through the Break methods that now will take an argument for the return value:

>
>     public static FuncRec<R> Break<R>(this FuncRec<R> a, R res) { _brr[a] = res; return a; }
>     public static FuncRec<T, R> Break<T, R>(this FuncRec<T, R> a, R res) { _brr[a] = res; return a; }
>     public static FuncRec<T1, T2, R> Break<T1, T2, R>(this FuncRec<T1, T2, R> a, R res) { _brr[a] = res; return a; }

What's happening inside those Break methods will be discussed further on. For now, it suffices to see the signatures, taking in an R parameter to hold the return value of the recursive call. Also notice how those methods return "a" instead of null.

Before we dig any deeper in the implementation, let's see a couple of recursive functions in action:

>
>     Console.WriteLine("Function of arity 0");
>     {
>         int i = 0;
>         Func<FuncRec<**_int_**>, Func<FuncRec<**_int_**>>> f = a => () => { Console.WriteLine("Fun! " + i++); return i < 10 ? a() : **_a.Break(i)_**; };
>         Console.WriteLine("Result: " + f.Fix()());
>     }
>     Console.WriteLine();
>
>     Console.WriteLine("Function of arity 1");
>     {
>         int i = 0;
>         Func<FuncRec<int, **_int_**>, Func<int, FuncRec<int, **_int_**>>> f = a => x => { Console.WriteLine("2^" + i++ + " = " + x); return i < 10 ? a(x * 2) : **_a.Break(i)_**; };
>         Console.WriteLine("Result: " + f.Fix()(1));
>     }
>     Console.WriteLine();
>
>     Console.WriteLine("Function of arity 2");
>     {
>         int i = 0;
>         Func<FuncRec<int, int, **_int_**>, Func<int, int, FuncRec<int, int, **_int_**>>> f = a => (x, y) => { Console.WriteLine("2^" + x + " = " + y); return ++i < 10 ? a(x + 1, y * 2) : **_a.Break(i)_**; };
>         Console.WriteLine("Result: " + f.Fix()(0, 1));
>     }
>     Console.WriteLine();

We bound the recursion again by means of some outer local variable, but this is not a requirement. But in order to show all functions without one running away, such a bound is desirable. Concerning the input parameters, things look identical to the ActionRec samples. What's different are the Break calls and the output types specified in the FuncRec type parameters. We've simply used the bounding variable "i" as the return value for illustration purposes. Later, when we see factorial again, the output value will be more interesting.

How does Fix work this time? Let's show one sample for the function with one argument:

>
>     public static Func<T, R> Fix<T, R>(this Func<FuncRec<T, R>, Func<T, FuncRec<T, R>>> f)
>     {
>         return t =>
>         {
>             object res_;
>             FuncRec<T, R> a = null;
>             for (a = t_ => { t = t_; return a; }; !_brr.TryGetValue(a, out res_); a = f(a)(t))
>                 ;
>             var res = (R)res_;
>             _brr.Remove(a);
>             return res;
>         };
>     }

I'm using an ugly trick here to store the return value. Have a look at the Break methods that do stick the specified result in a dictionary, which is typed as follows:

>
>     // Would really like to store result on a property on the delegate,
>     // but can't derive from Delegate manually in C#... This is "brr".
>     private static Dictionary<Delegate, object> _brr = new Dictionary<Delegate, object>();

Break add the return value to this dictionary, while the trampoline driver loop checks for such a value repeatedly. If one is found, a Break call has been done and the loop terminates, stopping the recursion and sending the answer to the caller. Alternative potentially cleaner tricks can be thought of, but I haven't spent much more time thinking about this.

All in all, the core Fix is pretty much the same as for the action-based delegates, apart from the TryGetValue call in the condition, and some dictionary-related cleanup code. Below is our destination factorial sample:

>
>     Console.WriteLine("Factorial");
>     {
>         Func<FuncRec<int, int, int>, Func<int, int, FuncRec<int, int, int>>> fac_ = f => (x, a) => x <= 1 ? f.Break(a) : f(x - 1, a * x);
>         Func<int, int> fac = (int n) => fac_.Fix()(n, 1);
>         Enumerable.Range(1, 10).Select(n => new { n, fac = fac(n) }).Do(Console.WriteLine).Run();
>     }
>     Console.WriteLine();

The type of the intermediate function definition is quite impressive due to the fixpoint structure, but the essence of the function is quite easy to grasp:

> f => (x, a) => x <= 1 ? f.Break(a) : f(x - 1, a * x)

Given a function (that will represent the fixed factorial definition, i.e. itself) and two arguments, one to count down and one to represent the accumulated product, we simply continue multiplying till we hit the base case, where we return (using Break) the accumulated value. The next line creates a simple wrapper function to hide away the accumulator base value of 1:

> Func<int, int> fac = (int n) => fac_.Fix()(n, 1);

And now we have a simple factorial function we can call in the regular manner we're used to, using delegate invocation syntax. To illustrate it for multiple values, I'm using a simple LINQ _statement_, projecting each value from 1 to 10 onto an anonymous object with both that number and the corresponding factorial value. The Do and Run methods will be introduced in the Reactive Framework as new extensions to IEnumerable:

> public static IEnumerable<T> Do<T>(this IEnumerable<T> src, Action<T> a)
{
    foreach (var item in src)
    {
        a(item);
        yield return item;
    }
}
>
> public static void Run<T>(this IEnumerable<T> src)
{
    foreach (var _ in src)
        ;
}

To prove the stack utilization remains constant, we can extend the sample using the handy System.Diagnostics.StackTrace class and the .NET 4.0 Tuple class. In the non-trampolined version, we'd see the stack grow on every call, reaching its maximum depth at the point we return from the base case. So, watching the stack depth at the point of the base case's return call (using Break) will be a good metric of success:

>
>     Console.WriteLine("Factorial + stack analysis");
>     {
>         Func<FuncRec<int, int, Tuple<int, int>>, Func<int, int, FuncRec<int, int, Tuple<int, int>>>> fac_ =
>     f => (x, a) => x <= 1 ? f.Break(new Tuple<int,int>(a, _new StackTrace().FrameCount_)) : f(x - 1, a * x);
>         Func<int, Tuple<int, int>> fac = (int n) => fac_.Fix()(n, 1);
>         (from n in Enumerable.Range(1, 10)
>          let f = fac(n)
>          select new { n, fac = f.Item1, _stack = f.Item2_ }).Do(Console.WriteLine).Run();
>     }
>     Console.WriteLine();

The result is shown below:

> ![image][13]

This looks good, doesn't it? If you get tried of the long generic Func types, simply call the Fix method directly, passing in the types of the arguments and return value:

>
>     var fac_ = Ext.Fix<int, int, Tuple<int, int>>(f => (x, a) =>
>     x <= 1 ? f.Break(new Tuple<int, int>(a, new StackTrace().FrameCount)) : f(x - 1, a * x)
>     );

Beautiful! Almost reads like a regular C# method declaration (with plenty of imagination the author possesses).

 

Since readers often want to try out the thing as a whole, here's the implementation of my latest Esoteric namespace:

>
>     // Trampoline for tail recursive Action and Func delegate creation and invocation in constant stack space
>     // bartde - 10/29/2009
>
>     using System;
>     using System.Collections.Generic;
>
>     namespace Esoteric
>     {
>         delegate ActionRec ActionRec();
>         delegate ActionRec<T> ActionRec<T>(T t);
>         delegate ActionRec<T1, T2> ActionRec<T1, T2>(T1 t1, T2 t2);
>
>         delegate FuncRec<R> FuncRec<R>();
>         delegate FuncRec<T, R> FuncRec<T, R>(T t);
>         delegate FuncRec<T1, T2, R> FuncRec<T1, T2, R>(T1 t1, T2 t2);
>
>         static class Ext
>         {
>             public static ActionRec Break(this ActionRec a) { return null; }
>             public static ActionRec<T> Break<T>(this ActionRec<T> a) { return null; }
>             public static ActionRec<T1, T2> Break<T1, T2>(this ActionRec<T1, T2> a) { return null; }
>
>             public static Action Fix(this Func<ActionRec, Func<ActionRec>> f)
>             {
>                 return () =>
>                 {
>                     ActionRec a = null;
>                     for (a = () => a; a != null; a = f(a)())
>                         ;
>                 };
>             }
>
>             public static Action<T> Fix<T>(this Func<ActionRec<T>, Func<T, ActionRec<T>>> f)
>             {
>                 return t =>
>                 {
>                     ActionRec<T> a = null;
>                     for (a = t_ => { t = t_; return a; }; a != null; a = f(a)(t))
>                         ;
>                 };
>             }
>
>             public static Action<T1, T2> Fix<T1, T2>(this Func<ActionRec<T1, T2>, Func<T1, T2, ActionRec<T1, T2>>> f)
>             {
>                 return (t1, t2) =>
>                 {
>                     ActionRec<T1, T2> a = null;
>                     for (a = (t1_, t2_) => { t1 = t1_; t2 = t2_; return a; }; a != null; a = f(a)(t1, t2))
>                         ;
>                 };
>             }
>
>             // Would really like to store result on a property on the delegate,
>             // but can't derive from Delegate manually in C#... This is "brr".
>             private static Dictionary<Delegate, object> _brr = new Dictionary<Delegate, object>();
>
>             public static FuncRec<R> Break<R>(this FuncRec<R> a, R res) { _brr[a] = res; return a; }
>             public static FuncRec<T, R> Break<T, R>(this FuncRec<T, R> a, R res) { _brr[a] = res; return a; }
>             public static FuncRec<T1, T2, R> Break<T1, T2, R>(this FuncRec<T1, T2, R> a, R res) { _brr[a] = res; return a; }
>
>             public static Func<R> Fix<R>(this Func<FuncRec<R>, Func<FuncRec<R>>> f)
>             {
>                 return () =>
>                 {
>                     object res_;
>                     FuncRec<R> a = null;
>                     for (a = () => a; !_brr.TryGetValue(a, out res_); a = f(a)())
>                         ;
>                     var res = (R)res_;
>                     _brr.Remove(a);
>                     return res;
>                 };
>             }
>
>             public static Func<T, R> Fix<T, R>(this Func<FuncRec<T, R>, Func<T, FuncRec<T, R>>> f)
>             {
>                 return t =>
>                 {
>                     object res_;
>                     FuncRec<T, R> a = null;
>                     for (a = t_ => { t = t_; return a; }; !_brr.TryGetValue(a, out res_); a = f(a)(t))
>                         ;
>                     var res = (R)res_;
>                     _brr.Remove(a);
>                     return res;
>                 };
>             }
>
>             public static Func<T1, T2, R> Fix<T1, T2, R>(this Func<FuncRec<T1, T2, R>, Func<T1, T2, FuncRec<T1, T2, R>>> f)
>             {
>                 return (t1, t2) =>
>                 {
>                     object res_;
>                     FuncRec<T1, T2, R> a = null;
>                     for (a = (t1_, t2_) => { t1 = t1_; t2 = t2_; return a; }; !_brr.TryGetValue(a, out res_); a = f(a)(t1, t2))
>                         ;
>                     var res = (R)res_;
>                     _brr.Remove(a);
>                     return res;
>                 };
>             }
>         }
>     }

Another sample illustrating the stack-friendly nature of the trampoline, is shown below:

>
>     Console.WriteLine("Forever! (CTRL-C to terminate)");
>     {
>         bool boom = false;
>         Console.CancelKeyPress += (s, e) =>
>         {
>             boom = true;
>             e.Cancel = true;
>         };
>
>
>         Func<ActionRec, Func<ActionRec>> f = a => () =>
>     {
>     if (boom)
>     throw new Exception("Stack use is constant!");
>     return a();
>     };
>
>
>         try
>         {
>             f.Fix()();
>         }
>         catch (Exception ex)
>         {
>             // Inspect stack trace here
>             Console.WriteLine(ex);
>         }
>     }

This function never returns unless you force it by pressing CTRL-C. At that point, you'll see the exception's stack trace being printed, revealing the constant stack space:

> ![image][14]

It also illustrates how the trampoline is sandwiched between our call to the recursive function (f.Fix()**()**) and the callback to the code we wrote (**f**).

 

The reader is invited to think about realizing mutual recursion in a stack-friendly way. For example, the sample below is an F# mutual recursive set of two functions used to determine whether a number is odd or even:

> let rec isEven n =
  if n = 0 then
    true
  else
    isOdd (n - 1)
and isOdd n =
  if n = 0 then
    false
  else
    isEven (n - 1)

Its use is shown below:

> ![image][15]

In fact, the F# implementation generates mutually recursive calls here, but in a stack-friendly way by using tail calls (only shown for the isEven function below, but similar for isOdd):

> ![image][16]

Tail calls reuse the current stack frame, therefore not exhausting the stack upon recursion. The same can be achieved by means of a trampoline if you're brave enough to give it a try. Hint: notice how mutually recursive functions in F# are subtly bundled by means of an "and" keyword.

As an additional piece of homework, think about ways we could use a trampoline to call functions that still return a useful value, to be used in code after the call returns. As an example, consider the classic definition of factorial:

> int fac(int n)
{
    return n == 0 ? 1 : **n * **fac(n – 1);
}

How would you realize exactly the code above, using trampolines and whatnot, without exhausting call stack space? Recall the problem with the above is the fact we need to do a multiplication after the recursive fac call returns. Hint: think of continuations and maybe even the typical "von Neumann machine trade-off" between code (CPU) and data (memory).

Happy jumping!

[Del.icio.us][17] | [Digg It][18] | [Technorati][19] | [Blinklist][20] | [Furl][21] | [reddit][22] | [DotNetKicks][23]

Filed under: [Functional programming][24], [Crazy Sundays][25]

[1]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/clip_image002_thumb.gif "clip_image002"
[2]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/clip_image004_thumb.gif "clip_image004"
[3]: http://en.wikipedia.org/wiki/Catamorphism
[4]: http://en.wikipedia.org/wiki/Fixed_point_combinator
[5]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/image_thumb.png "image"
[6]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/image_thumb_3.png "image"
[7]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/image_thumb_4.png "image"
[8]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/image_thumb_5.png "image"
[9]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/image_thumb_6.png "image"
[10]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/image_thumb_7.png "image"
[11]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/image_thumb_8.png "image"
[12]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/image_thumb_9.png "image"
[13]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/image_thumb_10.png "image"
[14]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/image_thumb_11.png "image"
[15]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/image_thumb_12.png "image"
[16]: http://bartdesmet.info/images_wlw/JumpingthetrampolineinCStackfriendlyrecu_C5F5/image_thumb_13.png "image"
[17]: http://del.icio.us/post?url=http%3a%2f%2fcommunity.bartdesmet.net%2fblogs%2fbart%2farchive%2f2009%2f11%2f08%2fjumping-the-trampoline-in-c-stack-friendly-recursion.aspx&tags=&title=Jumping+the+trampoline+in+C%23+%e2%80%93+Stack-friendly+recursion
[18]: http://digg.com/submit?phase=2&url=http%3a%2f%2fcommunity.bartdesmet.net%2fblogs%2fbart%2farchive%2f2009%2f11%2f08%2fjumping-the-trampoline-in-c-stack-friendly-recursion.aspx&title=Jumping+the+trampoline+in+C%23+%e2%80%93+Stack-friendly+recursion&tags=
[19]: http://technorati.com/cosmos/search.html?url=http%3a%2f%2fcommunity.bartdesmet.net%2fblogs%2fbart%2farchive%2f2009%2f11%2f08%2fjumping-the-trampoline-in-c-stack-friendly-recursion.aspx&tags=
[20]: http://blinklist.com/index.php?Action=Blink/addblink.php&url=http%3a%2f%2fcommunity.bartdesmet.net%2fblogs%2fbart%2farchive%2f2009%2f11%2f08%2fjumping-the-trampoline-in-c-stack-friendly-recursion.aspx&Title=Jumping+the+trampoline+in+C%23+%e2%80%93+Stack-friendly+recursion&tags=
[21]: http://furl.net/storeIt.jsp?t=Jumping+the+trampoline+in+C%23+%e2%80%93+Stack-friendly+recursion&u=http%3a%2f%2fcommunity.bartdesmet.net%2fblogs%2fbart%2farchive%2f2009%2f11%2f08%2fjumping-the-trampoline-in-c-stack-friendly-recursion.aspx&tags=
[22]: http://reddit.com/submit?url=http%3a%2f%2fcommunity.bartdesmet.net%2fblogs%2fbart%2farchive%2f2009%2f11%2f08%2fjumping-the-trampoline-in-c-stack-friendly-recursion.aspx&title=Jumping+the+trampoline+in+C%23+%e2%80%93+Stack-friendly+recursion&tags=
[23]: http://www.dotnetkicks.com/submit/?url=http%3a%2f%2fcommunity.bartdesmet.net%2fblogs%2fbart%2farchive%2f2009%2f11%2f08%2fjumping-the-trampoline-in-c-stack-friendly-recursion.aspx&title=Jumping+the+trampoline+in+C%23+%e2%80%93+Stack-friendly+recursion&tags=
[24]: /blogs/bart/archive/tags/Functional+programming/default.aspx
[25]: /blogs/bart/archive/tags/Crazy+Sundays/default.aspx
