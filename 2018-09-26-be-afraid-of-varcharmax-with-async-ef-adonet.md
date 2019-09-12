Be afraid of varchar(max) with async EF or ADO .NET
==================================================
> This article located in: https://blog.codeinside.eu/2018/09/26/be-afraid-of-varcharmax-with-async-ef-adonet/ , wrote by Robert Muehsig

Last month we had changed our WCF APIs to async implementations, because we wanted all those glorious scalability improvements in our codebase.

The implementation was quite easy, because our service layer did most of the time just some simple EntityFramework 6 queries.

The field test went horribly wrong
----------------------------------
After we moved most of the code to async we did a small test and it worked quite good. Our gut feelings were OK-ish, because we knew that we didn’t do a full stress test.

As always: Things didn’t work as expected. We deployed the code on our largest customer and it did: Nothing.

100% CPU
--------
We knew that after the deployment we would hit a high load and at first it seems to “work” based on the CPU workload, but nothing happend. I checked the SQL monitoring and noticed that the throughput was ridiculous low. One query (which every client needed to execute) caught my attention, because the query itself was super simple, but somehow was the showstopper for everyone.

The “bad query”
---------------
I checked the code and it was more or less something like this (with the help of EntityFramework 6)

```csharp
var result = await dbContext.Configuration.ToListAsync();
```

The “Configuration” itself is a super simple table with a Key & Value column.

Be aware that the same code worked OK with the non async implementation!

“Cause”
-------
This call was extremely costly in terms of performance, but why? It turns out, that this customer installation had a pretty large configuration. One value was around 10MB, which doesn’t sound that much, but if this code is executed in parallel with 5000 clients, it can hurt.

**On top of that**: The async implementation tries to be smart, but this leads to thousand of task creations, which will slow down everything.

This [stackoverflow answer](https://stackoverflow.com/a/28619983) really helped me to understand this problem. Just look at those figures:

>First, in the first case we were having just 3500 hit counts along the full call path, here we >have 118 371. Moreover, you have to imagine all the synchronization calls I didn’t put on the >screenshoot…
>
>Second, in the first case, we were having “just 118 353” calls to the TryReadByteArray() method, >here we have 2 050 210 calls ! It’s 17 times more… (on a test with large 1Mb array, it’s 160 >times more)
>
>Moreover there are:
>
> * 120 000 Task instances created
> * 727 519 Interlocked calls
> * 290 569 Monitor calls
> * 98 283 ExecutionContext instances, with 264 481 Captures
> * 208 733 SpinLock calls
>
>My guess is the buffering is made in an async way (and not a good one), with parallel Tasks >trying to read data from the TDS. Too many Task are created just to parse the binary data. …


Switch to ADO .NET, damn EF, right?
----------------------------------
If you are now thinking: “Yeah… EF sucks, right, use just plain ADO.NET!” you will end up in the same mess, because the default ExecuteAsync-reader is used in the EntityFramework.


I use EF Core, am I save?
-------------------------
The same problem applies to EF Core, just checkout [this comment](https://github.com/aspnet/EntityFramework6/issues/88#issuecomment-256103455) by the EF Team.

How can we solve this problem then?

Solution 1: Async, but with Sequential read
-------------------------------------------
I changed the code to use plain ADO .NET, but with [CommandBehavior.Sequential](https://docs.microsoft.com/en-us/dotnet/api/system.data.idbcommand.executereader?view=netframework-4.7.2) access.

This way it seems that the async implementation is much smarter how to read large chunks of data. I’m not an ADO .NET expert, but with the default strategy ADO .NET tries to read the whole row and stores it in memory. With the sequential access it can use the memory more effective - at least, it seems to work much better.

Your code also needs to be implemented with sequential access in mind, otherwise it will fail.

Solution 2: Avoid large type like nvarchar(max)
----------------------------------------------
This advice comes from the EF team:

*Avoid using NTEXT, TEXT, IMAGE, TVP, UDT, XML, [N]VARCHAR(MAX) and VARBINARY(MAX) – the maximum data size for these types is so large that it is very unusual (or even impossible) that they would happen to be able to fit within a single packet.*

When we need to store large content, we typically use a separat blob table and stream those values to the clients. This works quite well, but we forgot our “configuration” table :-)

When I now look at this problem it seems obvious, but we had some hard days to fix the issue.

**Hope this helps.**

Helpful links:

 * [Why EF async methods are slower than non-async?](https://entityframework.net/why-ef-async-methods-are-slow)
 * [EF GitHub Issue: Performance issue when querying varbinary(MAX), varchar(MAX), nvarchar(MAX) or XML with Async](https://github.com/aspnet/EntityFramework6/issues/88)
 * [Stackoverflow answer](https://stackoverflow.com/a/28619983)
