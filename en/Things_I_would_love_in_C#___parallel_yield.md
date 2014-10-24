[Source](http://ecogamedesign.wordpress.com/2013/10/09/things-i-would-love-in-c-parallel-yield/ "Permalink to Things I would love in C# : parallel yield")

# Things I would love in C# : parallel yield

yield is a very useful keyword in C# that allows you to create dynamic enumerators of stuffs without having to fill in a List<T> of things you want to return first. The generated code builds a state machine that will go forward, step by step, whenever the caller calls MoveNext on the IEnumerable it gets back (usually using foreach).



            static void Main(string[] args)
            {
                var input = new[] { "A", "B", "C" };

                Console.WriteLine(String.Join("n", Process(input)));
            }

            static IEnumerable<string> Process(IEnumerable<string> input)
            {
                if (input == null)
                    throw new ArgumentNullException("input");

                foreach (var item in input)
                {
                    yield return "Process " + item;
                }
            }

            // Output :
            // Process A
            // Process B
            // Process C


The second really useful thing is Parallel.ForEach. Whenever you have to process loads of independent data that are CPU hungry you can launch the process in parallel and have it scheduled for you by the .NET runtime.



            static void Main(string[] args)
            {
                var input = new[] { "A", "B", "C" };

                Stopwatch w = new Stopwatch();
                w.Start();
                Process(input);
                w.Stop();

                Console.WriteLine("{0}s ", w.Elapsed.TotalSeconds);
            }

            static void Process(IEnumerable<string> input)
            {
                if (input == null)
                    throw new ArgumentNullException("input");

                Parallel.ForEach(input, item =>
                    {
                        // Something long
                        Thread.Sleep(1000);
                    });
            }

            // Output :
            // ~ 1s although we had 3 stuffs that each took 1s


What if we could write that?



            static void Main(string[] args)
            {
                var input = new[] { "A", "B", "C" };

                Stopwatch w = new Stopwatch();
                w.Start();
                var result = Process(input);
                w.Stop();

                Console.WriteLine("{0}s ", w.Elapsed.TotalSeconds);
                Console.WriteLine(String.Join("n", result));
            }

            static IEnumerable<string> Process(IEnumerable<string> input)
            {
                if (input == null)
                    throw new ArgumentNullException("input");

                Parallel.ForEach(input, item =>
                    {
                        // Something long
                        Thread.Sleep(1000);

                        // >>>> THE POWER !!
                        yield return "Process " + item;
                    });
            }


But no we can't because "**The yield statement cannot be used inside an anonymous method or lambda expression**".
You obviously can't mix those two pieces of magic together. In fact, if we could this would produce some of the weirdest bugs ever if something went wrong. But maybe we can't build something that looks like it, for the sake of doing it?



            static void Main(string[] args)
            {
                var input = new[] { "A", "B", "C" };

                Stopwatch w = new Stopwatch();
                w.Start();
                var result = Process(input);

                Trace.WriteLine(String.Format("{0}s before enumerating", w.Elapsed.TotalSeconds));
                Trace.WriteLine(String.Format(String.Join("n", result)));
                Trace.WriteLine(String.Format("{0}s after enumerating", w.Elapsed.TotalSeconds));
            }

            static IEnumerable<string> Process(IEnumerable<string> input)
            {
                if (input == null)
                    throw new ArgumentNullException("input");

                var workQueue = new ConcurrentQueue<string>();
                Exception workException = null;
                var completed = false;

                // This task will run on its side in another thread
                Task.Run(() =>
                {
                    try
                    {
                        Parallel.ForEach(input, item =>
                            {
                                // Something long
                                Thread.Sleep(1000);

                                // Push our results here
                                workQueue.Enqueue("Process " + item);
                            });
                    }
                    catch (Exception ex)
                    {
                        // We don't want a crash to be lost in another thread
                        // so we have to bring it back somehow on the main thread
                        workException = ex;
                    }

                    completed = true;
                });

                // Not the most elegant, but it's just for fun so ...
                while (!completed)
                {
                    string item;
                    if (workQueue.TryDequeue(out item))
                        yield return item;
                    else
                        Thread.Sleep(1); // remember, just for fun
                }

                // We don't want to lose the last items
                string finalItem;
                while (workQueue.TryDequeue(out finalItem))
                    yield return finalItem;

                // Buble the exception if there's one
                if (workException != null)
                {
                    throw new Exception("Error while doing stuffs", workException);
                }
            }


Output:

> 0.00039s before enumerating
Process B
Process A
Process C
1.0398348s after enumerating

Notice than we did have an "yield behavior" as the processing was not executed on the call to Process, and how the whole process took 1s where it has 3 items that each took 1s. We also don't have a reliable order anymore, it could have been any combination of B, A and C.

But anyway, **nailed it** :)

### Like this:

Like Loading...

### _Related_