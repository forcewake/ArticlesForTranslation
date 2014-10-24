[Source](http://volgarev.me/blog/62412678347 "Permalink to Tail recursion and trampolining in C#")

# Tail recursion and trampolining in C#

There's a very interesting course on Coursera which started earlier this month. It's called "Functional Programming Principles in Scala" and it's being taught by [Martin Odersky][1], the guy behind Scala programming language. Since I wanted to learn Scala for quite some time now, I thought I'd be a good opportunity so I joined. It's been a great experience so far and I feel like I'm re-exploring the principles, paradigms and power of functional programming. I even caught myself on a thought that it would probably be a good idea to teach Scala or Haskell (instead of Basic, Pascal or C) as a first programming language at schools because it instills you the style of writing elegant and performant programs. One of the terms that popped up several times during the course and the one I knew very little about was the term "tail recursion". I immediately started researching whether this is something that can be leveraged in C# and this article is the result of my research (there's actually a plenty of articles on the subject out there and the goal of this one is to give the reader a quick and concise overview rather than stuffing the article with a lot of theory and huge code snippets).

### What is recursion

Generally speaking, recursion is when the function calls itself (usually, with a different set of parameters). This allows solving a problem in terms of solving the same problem but on a smaller scale. For example, finding a factorial of a number N can be expressed as multiplying N by factorial of a number M such as M = N - 1. Of course, many problems can be solved without using recursion and depending on a programming language of choice there're certain pitfalls (like, stack overflow) that you should be aware of when going for recursive solution. On the other hand, there's a wide range of problems where the use of recursion not only greatly generalizes the solution but also makes the code much more readable, maintainable and elegant. An example of such problem can be finding a shortest path in a maze by using Lee algorithm.

As I mentioned earlier, one of the drawbacks of using recursion is a chance of exceeding the maximum stack depth (which is defined by the execution environment) since every time the recursive call happens, the runtime needs to preserve the state of the current stack frame (which might be needed upon the return from the recursive all) and, therefore, create a new one. Let's take a look at the recursive implementation of factorial:

As you can see, the return value of factorial(n) depends on the return value of factorial(n - 1). For every recursive call to factorial(n - 1) the CLR is going to create a new stack frame, so if we assume that the value of a numeric type we're using is never going to overflow (for this purpose I used hypothetical "BigInt" type), then there's a risk of getting "StackOverflowException" for very big initial values of "n".

### What is tail recursion

Tail recursion is a method of implementing recursive solutions when the recursive call happens to be the last action in a method. Let's take a look at the modified implementation of factorial which now uses tail call:

The conceptual difference between the first and the second example is that in the former case the call to factorial(n - 1) is not a tail call since it's not the final action (we multiply the result of a call by the current value of n) whereas in the latter case we always pass around an additional parameter called "current" which serves as an accumulator and makes it possible to implement tail-recursive solution.

### Why implement tail calls

The huge benefit of using tail calls is that since they're the last actions within the method, the current stack frame can be reused for the next recursive call - you basically don't have any state to preserve since you're not going to need it after the return from the recursive call. This means that the entire recursive flow can be executed on the same stack frame eliminating the possible stack overflows (and also speeding up things). This approach is called tail call optimization and in case of .NET or Java, it's a compiler that is responsible for performing it (by emitting the correct IL opcodes). Of course, this kind of optimization is supported by functional programming languages like Scala of F#.

### Tail call optimizations in C#

Unfortunately, tail call optimizations are not performed by C# compiler (although tail calls are fully supported by CLR since .NET 4.0). You can easily verify this by running our tail-recursive factorial example and putting the breakpoint inside the "accumulate" method. Every time you hit the breakpoint, you can see that the stack trace grows in size (Debug -> Windows -> Call Stack). I wonder whether C# compiler team decides to implement the support for tail calls at some point (the faster, the merrier - in my opinion, it's a huge win).

### The technique of trampolining

In case tail call optimizations can't be leveraged, there's a well known workaround of using the technique called "trampolining" which can be implemented in any language that supports higher-order functions (C# is one of those languages). The main idea is to execute the method in a loop whereas the method itself can either return a final result or a new set of arguments. Let's take a look at how we can change the tail-recursive implementation of factorial to use trampolining.

First, let's define the return type for our "bouncing" action (which is to be executed in a loop):

As you can see, we can either pass the next two arguments (the values for the "current" and "n" parameters from the tail-recursive example) or pass the end result (in this case "HasResult" is set to "true" which tells the caller to break from the loop).

Here's the implementation of the trampoline itself:

The implementation is pretty straightforward: the action method continues to "bounce" as long as the final result is not available. In this case, the action method returns the next set of its arguments on every iteration.

The trampoline-enabled factorial implementation will now look like the following:

### Conclusion

Tail recursion is a very powerful technique of implementing recursive solutions without worrying about exceeding the maximum stack size. It's very sad that C# compiler doesn't implement tail call optimizations but as you just saw, trampolining is one of the ways of preserving the recursive nature of our solution but executing it in imperative fashion. Since the action method (which is called "iteration" in our example) is executed in a loop, stack trace has a constant size throughout the entire computation process.

These are the articles I found useful while studying the subject:

Take care!

[1]: https://twitter.com/odersky
