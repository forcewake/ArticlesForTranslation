[Source](http://weblogs.asp.net/jongalloway//working-around-a-powershell-call-depth-disaster-with-trampolines "Permalink to Jon Galloway - Working around a Powershell Call Depth Disaster With Trampolines")

# Jon Galloway - Working around a Powershell Call Depth Disaster With Trampolines

[I just posted about an update to my NuGet package downloader script which included a few fixes, including a fix to handle paging][1]. That sounds boring, but wait until you hear about the trampolines.

## Recursion: Seemed like a good idea at the time

The whole idea of this script is to download package files referenced in the public NuGet package feed. The NuGet package is in OData format, and it's paged. That's a good thing - requesting the feed shouldn't make you wait while up to 16,000 (and growing) package descriptions are downloaded, and it shouldn't tie up the server with that either. Each page currently returns 100 items, but that's a service implementation detail that could change at any time.

Paging in OData is done using a <link rel="next" href="http://url.com?$skiptoken=sometoken> element at the end of each page, like this:


    <?xml version="1.0" encoding="iso-8859-1" standalone="yes"?>
    <feed xml:base="http://packages.nuget.org/v1/FeedService.svc/" xmlns:d="http://schemas.microsoft.com/ado/2007/08/dataservices"
    xmlns:m="http://schemas.microsoft.com/ado/2007/08/dataservices/metadata" xmlns="http://www.w3.org/2005/Atom">
      <title type="text">Packages</title>
      <id>http://packages.nuget.org/v1/FeedService.svc/Packages</id>
      <updated>2011-12-08T23:46:46Z</updated>
      <link rel="self" title="http://weblogs.asp.net/Packages" href="http://weblogs.asp.net/Packages" />

      <entry>
        <id>http://packages.nuget.org/v1/FeedService.svc/Packages(Id='adjunct-Some.Package',Version='1.0.0.0')</id>
        <title type="text">Some.Package</title>
        <summary type="text">First!!!! w00t!!!</summary>
        < more stuff here />
      </entry>
      <entry>
        < ... entry stuff for another package ... />
      </entry>

      < etc - 98 more entry elements />

      <entry>
        <id>http://packages.nuget.org/v1/FeedService.svc/Packages(Id='adjunct-Last.On.The.Page',Version='1.0.0.0')</id>
        <title type="text">Last.On.The.Page</title>
        <summary type="text">Last package on the first page</summary>
        < more stuff here />
      </entry>

      <link rel="next" href="http://packages.nuget.org/v1/FeedService.svc/Packages?$skiptoken='adjunct-Last.On.The.Page','1.0.0.0'" />
    </feed>

The important bit is that the link element tells you how to request the next page of information. Here it's a link to the same feed with a skip token indicating the last item we've seen. I'm simplifying this a bit - the feed also handles searching, ordering, etc., but I'm just focusing on the paging bit. If you make a big request, the server returns you a chunk of the results with a link to get the next chunk.

My original implementation was to do something like this:


    function DownloadEntries {
     param ([string]$feedUrl)

        // Download all items on the page

        $nextUrl = href from last link on the page

        DownloadEntries $nextUrl
    }

    DownloadEntries $firstPage

_Note: If you're new to PowerShell, that's the syntax for writing a function called DownloadEntries which takes a string parameter called feedUrl._

So DownloadEntries gets passed a URL, downloads all entries on that page, and calls itself with the URL to the next page in the list. This works just fine for a while, but the problem is that every page is adding another level to the recursive depth of the call graph, meaning we'll run into trouble soon.

## Recursion, Call Stacks, and Stack Overflows

_Note: I know I'm going to get __[#wellactually_][2]_'d to death. I'm bracing for it. I'm not a Recursion Expert of Recursions, and if you are I'm sure I'll hear about it in the comments. Also, you're a big, mean jerk. Actually, I'm happy to learn more, so fire away. Now with that out of the way..._

The most common example of recursion in computer science is when a function calls itself. I first stumbled across it as a junior programmer, when I "re-invented" it to solve a tricky problem. I needed to generate a massive Word document report based on a complex object model, and eventually came up with a function that took a node object, wrote out that node's report information, and called itself for each child node. It worked, and when my boss said, "Hey, nice use of recursion!" I hurried off to read what he was talking about.

Recursion can be really useful, but things can get out of hand. In stack-based languages, recursion requires maintaining the state for everything in the call stack. That means that the memory used by a function grows with the recursion call depth. Since memory is limited, it's easy to write code that will exhaust your available memory, and you'll get a dreaded stack overflow error. Wikipedia lists [infinite recursion as the most common cause of stack overflows][3], but you don't actually have to get all the way to infinity to run out of memory - a lot of recursion is enough.

## PowerShell's Call Depth Limit

Rather than wait for a stack overflow to happen - or even for a script to use up an obscene amount of memory - [PowerShell caps the call depth at 100][4]. That means that, even if there were no memory overhead to each call, the above function would fail at 100 pages. [Doug Finke shows a very easy way to demonstrate this][5]:


    function simpleTest ($n)
    {
        if($n -ne 0) { $n; simpleTest ($n-1) }
    }

    PS C:> simpleTest 10
    10
    9
    8
    7
    6
    5
    4
    3
    2
    1

    PS C:> simpletest 99
    99
    98
    97
    96
    95

    The script failed due to call depth overflow.
    The call depth reached 101 and the maximum is 100.

### Side Note: PowerShell has the same call depth limit (100) on x64 and x86

Also from Doug's post:

> _Recursion depth limit is fixed in version 1. Deep recursion was causing problems in 64bit mode because of the way exceptions were being processed. It was causing cascading out-of-memory errors. The net result was that we hard-limited the recursion depth on all platforms to help ensure that scripts would be portable to all platforms._
\- Bruce Payette, co-designer of PowerShell

## Tail-Recursive Calls and Trampolines

There are ways to work around this problem, but to me they all seem to boil down to replacing deep call stacks with loops - either automatically at the language level or via fancy pants programming. In my case, I needed to convert from recursion to trampoline style - albeit a very simple conversion. With the disclaimer that I'm not an expert here, I thought I'd overview what I read about trampolines, then simplify it down to the basic trampoline solution I ended up with.

### Tail Call Optimization

In some cases, languages can optimize their way around this problem. One example is tail call optimization. A tail call is a call which is happening immediately prior to returning from the function, meaning that there's no remaining logic in the calling function and thus no need to retain the stack. In a tail-recursive call, it's returning that value to itself.

Languages can effectively unroll this, since in a tail call situation they don't need to maintain the nested call stack as the calling function has completed all its logical operation. If you're using one of those languages and make sure that your recursive calls are tail-recursive (so no additional logic is performed after the recursive call), the language will handle this for you and you'll end up with the benefits of recursion without the baggage of building a big call stack.

Matthew Podwysocki runs through this in [more detail with a comparison of how this works in F# and C#][6] (with an interesting difference between 32 and 64 bit operation). He shows an example of a case where a single line of code after the recursive call breaks the tail call optimization, resulting in a stack overflow.

It's interesting to note that [the common language runtime supports tail calls][7], so it comes down to whether the language and compiler make use of it. PowerShell doesn't support tail call optimization, so I needed to alter my code. That's where I learned about trampolines.

### Trampolines

The common pattern I read about for dealing with this is to manage state yourself using [a trampoline: a piece of code that repeatedly calls functions][8]. And, depending on the language and recursive requirements, that little "a piece of code that repeatedly calls functions" can get really pretty complex.

If my implementation was really making deep use of recursion in such a way that I really needed to emulate nested calls, that trampoline implementation would need to be pretty sophisticated. While that's implemented differently in different languages, in modern .NET programming I'm reading that's usually done using a function factory trampoline class which calls functions for you. Here are some posts showing how to do that:

In my simple case, though, I could just to convert the recursive call so that it's called from a looping construct, returning state from each call. I'd call this option a strategic retreat, since this is now in no way recursive anymore. But, by calling it a refactoring to implement trampoline-style programming, I'll still keep my pride.

Less Talk, More Code

Right.

So since the reason I was using recursion was just to continue calling the next page after completing the work on the current one, my trampoline can be implemented as a simple for loop into a function that returns the URL of the next page. Remembering that we started with something like this:


    function DownloadEntries {
     param ([string]$feedUrl)

        // Download all items on the page

        $nextUrl = href from last link on the page

        DownloadEntries $nextUrl
    }

    DownloadEntries $firstPage

We can rewrite that as this:


    function DownloadEntries {
     param ([string]$feedUrl)

        // Download all items on the page

        $nextUrl = href from last link on the page

        return $nextUrl
    }

    while($feedUrl -ne $null) {
        $feedUrl = DownloadEntries $feedUrl
    }

This can now handle unlimited pages, since it's just a simple loop. The only downside is that I can no longer feel quite as sophisticated, since while I call that a trampoline, you'd probably call it a boring loop that calls into a function.

Here's the full script so you can see it in context:

[1]: http://weblogs.asp.net/jgalloway/archive/2011/12/14/nuget-powershell-downloader-update-adding-failed-download-retries-better-paging-support.aspx
[2]: http://tirania.org/blog/archive/2011/Feb-17.html
[3]: http://en.wikipedia.org/wiki/Stack_overflow#Infinite_recursion
[4]: http://msdn.microsoft.com/en-us/library/system.management.automation.scriptcalldepthexception(VS.85).aspx
[5]: http://www.dougfinke.com/blog/index.php/2007/05/12/not-tail-recursive/
[6]: http://weblogs.asp.net/podwysocki/archive/2008/07/07/recursing-into-linear-tail-and-binary-recursion.aspx
[7]: http://msdn.microsoft.com/en-us/library/system.reflection.emit.opcodes.tailcall.aspx
[8]: http://en.wikipedia.org/wiki/Tail-recursive_function#Through_trampolining
