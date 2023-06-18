
# 你的软件可以从ATM机的巧妙设计里学到点什么？

> 原文链接: https://www.simplethread.com/your-software-can-learn-a-lot-from-atms/

[TOC]



## Designed to Fail Well 为良好失败设计

Think about the last time you used an ATM. Chances are, you have one in your vicinity, and you’ve interacted with it more times than you can count.

回忆一下你最后一次使用ATM，很有可能，你附近就有一台，你使用ATM机的次数，你自己都数不清。

Have you ever received an incorrect amount of money from an ATM? I’m guessing your answer is no, despite the millions of bills they distribute every year. The reliability of ATMs, despite handling a task as complex as dispensing the correct amount of cash, is a testament to the ingenuity of their design. But how do they achieve such high reliability?

你从ATM接收到过数目不对的钱嘛？ 我猜的答案一定是没有过，尽管ATM机每年存取数百万的钱。尽管处理像发放正确数量现金这样的复杂任务，ATM机的可靠性证明了设计的独创性。ATM机是如何实现高可靠的。

It may surprise you that the answer is rather straightforward. ATMs employ a relatively simple mechanism that pulls bills one by one from a stack, verifying their thickness as they pass through.

可能你会感到非常惊讶，答案是相当简单的。ATM机采用了一个比较简单的机制，从一叠钞票中一张一张的抽出，然后在通过时验证他们的厚度。

If a bill is thicker than expected—likely because multiple bills are stuck together—it’s rejected and set aside for human inspection. The system isn’t perfect, but it’s designed to “fail well.”

如果抽取的钞票比预期的要厚，很可能是因为多张钞票粘在了一起，取款就会被拒绝并留下来供人工检查，这个系统并不完美，它为良好失败所设计。

## So, What Does This Have to Do With Software Design? 那么这与软件设计有什么关系呢?

ATM给我们提供了好多宝贵的经验

The ATM’s approach offers valuable lessons:

1. Perform a task.

​    执行任务

2. Verify the task.

   验证任务

3. If verification fails, stop and try again.

 验证任务执行失败，停止和再次尝试，

The beauty of this design is in its simplicity. Instead of creating a complex mechanism to ensure 100% reliability, ATMs are designed to handle failure gracefully. It’s a lesson that software developers can take to heart. In order to build reliable systems, here are the steps to follow:

这种设计的美妙之处在于其简洁性，ATM机不是通过创建复杂的机制来确保百分之百的可靠性，而是被设计为优雅的处理故障。这是软件开发者需要用心记下来的一节课。为了构建可靠的系统，下面一些步骤需遵循。

## Somehow Many Developers Didn’t Get This Memo

In order to build reliable systems, you need to:

1. Perform a single, hopefully verifiable, task.
2. Verify said task.
3. If verification failed, then undo what can be undone, and notify someone.

 为了建立高可靠的系统，你需要:

- 执行单一、可验证的任务
- 验证任务的执行结果
- 如果验证失败，就撤销可撤销的操作，通知相关人员。

下面设设计软件中，也需要避免一些做法: 

However, there are also practices to avoid when designing software: 

下图是软件设计中需要遵循的实践

1. Don’t perform multiple operations before verification: Keep things simple and verify each step. 在

   > 在验证之前不要做多个操作: 保持简单, 验证每个步骤。

2. Avoid deleting things for cleanup: Instead, quarantine problematic files to preserve state information that can help resolve issues later.

   > 避免为了清理删除一些东西: 相反，隔离有问题的文件可以保留状态信息，这会帮助我们解决问题。

3. Don’t automatically fix the issue: Unless you’re sure about the problem, writing code to handle errors can cause additional ones.

   > 不要自动去修复问题: 除非你对问题确信无疑，否则编写错误处理代码可能会引发意料之外的问题。

4. Never fail without logging or isolating state: When something fails, gather relevant data and put it in a known error location.

   > 失败的时候要有日志记录或状态隔离: 当出现错误时，收集数据，并将其放置在已知的错误位置。

5. Don’t perform operations in a shared location: If failure is possible, carry out the operation in a staging area first.

   > 不要在共享位置执行操作: 如何可能出现故障，在一个临时区域先执行操作。

## Simplicity, Verification, and Effective Failure Handling

In a nutshell, the philosophy of designing software should be akin to the design of ATMs—embrace simplicity, ensure verification, and handle failure effectively. 

简而言之，软件设计哲学应当和ATM设计类似，崇尚简单，确保验证，有效的处理失败。

Software doesn’t need to be complex to be reliable; it needs to fail well. Let’s continue to draw inspiration from the physical engineering world in designing effective software mechanisms and remember this important lesson. 

软件的可靠并不需要借助复杂，他需要拥有良好的故障处理能力。让我们继续从物理工程里面吸取灵感，在软件设计中记录这个有效的教训。

## 下面是文章的评论

读者Nicholas Piasecki的评论: 

> ATMs can still give out the wrong amounts if the technician swaps the $10 and $20 cartridges, yielding the depressing truth that there is always room for error. I know because it happened to me once, and I immediately marched on over to a nearby branch and joined quite the line of folks who were not amused … imagine getting $80 deducted but only getting $40 in cash doled out! (Some machines seem to have mitigated this by saying "20s only.")
>
> 如果技术人员调换了10美元和20美元 墨盒，那么ATM机就会给出错误的金额，这揭示了一个令人沮丧的事实，就是总有出错空间。因为我遇到过一次这样的情况，我立刻走入了附近的一家分行，加入了沮丧人群的队伍，他们都不太高兴，想象一下，被扣掉80美元，但只拿到了40美元。(有些机器已经通过只有20美元来解决这个问题)
>
> Still, this is good advice. Like traffic lights–if a timing error causes lights in opposing directions to go green at once, then a physical fuse is blown, causing them all to go to the flashing yellows/reds circuit. A lot of software could use the "fuse technique," where it just throws up its hands and says "I give up, nobody does anything till somebody takes a good hard look at me." Unfortunately, though, this kind of fail-safe in software usually means "crash" instead of "reduced functionality mode."
>
> 但这仍然是一个不错的建议，就像红绿灯一样，如果时间错误导致两个相反方向的的红绿灯都出现了绿灯，那么保险丝会被熔断，导致所有的信号灯都进入到闪烁的“黄灯/红灯模式”。很多软件都可以采用类似的保险丝技术。他放弃了继续运行，直到有人来检查我为止。但是不幸的是，这种故障保护在软件中意味着“崩溃”而不是降级。

> Especially true if failure is rare. Why have brittle exception code that is rarely tested or exercised when you can just hit "Retry"
>
> 尤其是故障很少发生时，你可以直接点重试，而不是编写很少使用的异常代码。

读者mat roberts的评论:

> I’ve had incorrect money from an ATM…only once mind, and it was a hardware problem – the note was old a crinkled and got lost in the mechanism somehow.
>
> 我曾经在ATM机里面取出错误的金额，只有一次，那是一个硬件问题--纸币又旧又皱，不知道是什么原因，在机器里面被卡住了。

作者回复Nicholas Piasecki:

> This is the internet, I knew as soon as I posted this I would get 1000 comments about people who got wrong amounts from an ATM
>
> 这就是互联网，我知道我一发这篇文章就能收到1000条从ATM机里面取到错误金额的评论。

Jonathan Pryor的评论:

> So, to summarize the entire article…
>
> KISS [0].
> KISS [0].
> KISS [0].
> KISS [0].
>
> Oh, and KISS [0].
>
> Plus, verify (which is simpler to do because of KISS [0]).
>
> Which is why we have developer guidelines about NOT using `catch(Exception)` (unless followed by a quick software exit), because it’s (1) not simple, and (2) it’s impossible to know what exactly you’re catching (OutOfMemoryException, anyone?) and whether it’s actually safe for your program to continue executing…
>
> Any thoughts on how to remove complexity from systems? I find that many developers take perverse joy in over-complicating things… 
>
> [0] Keep it Simple, Stupid [1]
> [1] Because if we think we’re all that, we’ve already lost. We need to keep reminding ourselves that we’re really Not So Smart, and by doing so we’ll ensure that the software we write can actually be understood by Mere Mortals, which behooves us all because we [i]are[/i] mere mortals…
>
> 总结这篇文章就是保持简单，保持简单，保持简单，保持简单。哦，还是保持简单(由于保持简单而容易验证)，这也是我们为什么要有开发者准则，不要捕获异常(Exception 异常的基类)，除非后面跟着软件的退出，因为它不简单而且无法确定具体捕获的是什么异常(比如OutOfMemoryException)，程序是否安全的继续执行。
>
> 保持简单、愚蠢，如果我们认为很厉害，我们已经失败了，我们需要时刻提醒自己我们并不聪明，这样做，我们将确保我们的软件可以被普通人所理解，这对我们都有好处，因为我们都是凡人。

读者**Al Tenhundfeld**的评论:

> It isn’t identical, but the goal to "try not to let failure cases complicate your design" feels very similar to the "let it fail" approach of Erlang.
>
> 这与Erlang的“让其失败”原则有些类似，尽管不完全相同。目标是尽量避免失败复杂化你的设计。

> I haven’t worked in it, but from what I understand, Erlang has an interesting SRP take on handling failure. You don’t muddy up your domain ？implementation with ton of error compensating code. Instead you have supervisor processes that watch your implementation process and decide what to do if a failure occurs. Is that about right?
>
> 我没有使用Erlang工作过，但据我了解，Erlang在处理故障的时候使用了一种有趣的单一职责方法，你不会在你的领域实现里面混杂大量的错误补偿代码。相反，你有监督进程来观察你的实现进程，并决定在失败的时候做些什么？ 这样做对嘛？

> On a completely different note, this also reminds me of one of the ways I get a lot of value from TDD. When practicing TDD, I find I’m much more inclined to think about how the code should fail. And often, I’m able to redesign the API so that instead of compensating for a error, the API doesn’t allow the error state to exist.
>
> 另外，这也让我想起了测试驱动开发获得的很多有价值方法中的一个。当我实践TDD的时候，我发现更倾向于思考代码应该如何失败。而且，我经常能够重新设计API，使其不对错误进行补偿，而是不允许错误状态的存在。

作者对Al Tenhundfeld的回复:

> It is funny that you say that, because this whole post almost turned into a post about failures in software. I agree, in most cases polluting your code with error handling is just a waste. It is usually better to instead spend time logging, so when an unexpected error occurs, you know what happened and are able to compensate by fixing the software, not having it go through elaborate gyrations in an attempt to fix itself.
>
> 你说的很有趣，因为整个文章都变成了关于软件故障。我同意，在大多数情况下，用错误处理来污染你的代码只是一种浪费。通常情况下，更好的做法是花时间记录日志，这样当错误发生的时候，你就知道发生了什么，并且能够通过修复软件进行补偿，而不是让软件经历繁琐的操作来尝试自我修复。

