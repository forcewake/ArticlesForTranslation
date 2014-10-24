[Source](http://www.claassen.net/geek/blog/2010/07/don’t-blow-your-stack-c-trampolining.html "Permalink to ILoggable")

# ILoggable

A couple of weeks ago someone posted two links on twitter suggesting the [trampolining][1] superiority of Clojure to C#. _(Can't find that tweet anymore, and yes, that tweet was what finally motivated me to migrate my blog so i could post again.)_

Well, the comparison wasn't really apples to apples. The C# article ["Jumping the trampoline in C# – Stack-friendly recursion"][2] is trying to do a lot more than the Clojure article ["Understanding the Clojure `trampoline'"][3]. But maybe that's the point as well: that with C# people end up building much more complex, verbose machinery when the simple style of Clojure is all that is needed. That determination wanders into the subjective, where language feuds are forged, and i'll avoid that diversion this time around.

What struck me more was that you can do trampolining a lot more simply in C# if we mimic the Clojure example given. Let's start by reproducing the stack overflow handicapped example first:


    Func<int, int> funA, funB = null;
    funA = n => n == 0 ? 0 : funB(--n);
    funB = n => n == 0 ? 0 : funA(--n);

I'd dare say that the Lambda syntax is more compact than the Clojure example ![:\)][4] Ok, ok, the body is artificially small, allowing me to replace it with a single expression, which isn't a realistic scenario. Suffice it to say, you can get quite compact with expressions in C#.

Now, if we call `funA(4)`, we'd get the same call sequence as in Clojure, i.e.


    funA(4) -> funB(3) -> funA(2) -> funB(1) -> funA(0) -> 0

And if you, instead, call `funA(100000)`, you'll get a `StackOverflowException`.

So far so good, but here is where we diverge from Clojure. Clojure is dynamically typed, so it can return a number or an anonymous function that produces that number. We can't do that (lest we return object, _ick), b_ut we can come pretty close.

## Bring in the Trampoline

The idea behind the Trampoline is that it unrolls the recursive calls into sequential calls, by having the functions involved return either a value or a continuation. The trampoline simply does a loop that keeps executing returned continuations until it gets a value, at which point it exits with that value.

What we need for C# then is a return value that can hold either a value or a continuation and with generics, we can create one class to cover this use case universally.


    public class TrampolineValue<T> {
        public readonly T Value;
        public readonly Func<TrampolineValue<T>> Continuation;

        public TrampolineValue(T v) { Value = v; }
        public TrampolineValue(Func<TrampolineValue<T>> fn) { Continuation = fn; }
    }

Basically it's a container for either a value T or a func that produces a new value container. Now we can build our Trampoline:


    public static class Trampoline {
        public static T Invoke<T>(TrampolineValue<T> value) {
            while(value.Continuation != null) {
                value = value.Continuation();
            }
            return value.Value;
        }
    }

Let's revisit our original example of two functions calling each other:


    Func<int, TrampolineValue<int>> funA, funB = null;
    funA = (n) => n == 0
      ? new TrampolineValue<int>(0)
      : new TrampolineValue<int>(() => funB(--n));
    funB = (n) => n == 0
      ? new TrampolineValue<int>(0)
      : new TrampolineValue<int>(() => funA(--n));

Instead of returning an int, we simply return a `TrampolineResult<int>` instead. Now we can invoke funA without worrying about stack overflows like this:


    Trampoline.Invoke(funA(100000));

_Voila_, the stack problem is gone. It may require a bit more plumbing than a dynamic solution, but not a lot more than adding type declarations, which will always be the syntactic differences between statically and dynamically typed. But with lambdas and inference, it doesn't have to be much more verbose.

## But wait, there is more!

Using `Trampoline.Invoke `with `TrampolineValue<T>` is a fairly faithful translation of the Clojure example, but it doesn't feel natural for C# and actually introduces needless verbosity. It's functional rather than object-oriented, which C# can handle but it's not its best face.

What `TrampolineValue<T>` and its invocation really represent are a lazily evaluated value. We really don't care about the intermediaries, nor the plumbing required to handle it.

What we want is for `funA `to return a value. Whether that is the final value or lazily executes into the final value on examination is secondary. Whether or not `TrampolineValue<T>` contains a value or a continuation shouldn't be our concern, neither should passing it to the plumbing that knows what to do about it.

So let's internalize all this into a new return type, `Lazy<T>`:


    public class Lazy<T> {
        private readonly Func<Lazy<T>> _continuation;
        private readonly T _value;

        public Lazy(T value) { _value = value; }
        public Lazy(Func<Lazy<T>> continuation) { _continuation = continuation; }

        public T Value {
            get {
                var lazy = this;
                while(lazy._continuation != null) {
                    lazy = lazy._continuation();
                }
                return lazy._value;
            }
        }
    }

The code for `funA `and `funB `is almost identical, simply replacing `TrampolineValue `with `Lazy`:


    Func<int, Lazy<int>> funA, funB = null;
    funA = (n) => n == 0
      ? new Lazy<int>(0)
      : new Lazy<int>(() => funB(--n));
    funB = (n) => n == 0
      ? new Lazy<int>(0)
      : new Lazy<int>(() => funA(--n));

And since the stackless chaining of continuations is encapsulated by `Lazy`, we can simply invoke it with:


    var result = funA(100000).Value;

This completely hides the difference between a `Lazy<T>` that needs to have its continuation triggered and one that already has a value. Now that's concise and easy to understand.

[1]: http://en.wikipedia.org/wiki/Trampoline_%28computers%29
[2]: http://community.bartdesmet.net/blogs/bart/archive/2009/11/08/jumping-the-trampoline-in-c-stack-friendly-recursion.aspx "Jumping the trampoline in C# – Stack-friendly recursion"
[3]: http://pramode.net/clojure/2010/05/08/clojure-trampoline/ "Understanding the Clojure `trampoline'"
[4]: http://www.claassen.net/geek/blog/wp-includes/images/smilies/icon_smile.gif
