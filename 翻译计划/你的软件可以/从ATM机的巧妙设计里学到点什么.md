
# 你的软件可以从ATM机的巧妙设计里学到点什么？



## Designed to Fail Well 为良好失败设计

Think about the last time you used an ATM. Chances are, you have one in your vicinity, and you’ve interacted with it more times than you can count.

回忆一下你最后一次使用ATM，很有可能，你附近就有一台，你使用ATM机的次数，你自己都数不清。

Have you ever received an incorrect amount of money from an ATM? I’m guessing your answer is no, despite the millions of bills they distribute every year. The reliability of ATMs, despite handling a task as complex as dispensing the correct amount of cash, is a testament to the ingenuity of their design. But how do they achieve such high reliability?

你从ATM接收到过数目不对的钱嘛？ 我猜的答案一定是没有过，尽管ATM机每年存取数百万的钱。尽管处理像发放正确数量现金这样的复杂任务，ATM机的可靠性证明了设计的独创性。ATM机是如何实现高可靠的。

It may surprise you that the answer is rather straightforward. ATMs employ a relatively simple mechanism that pulls bills one by one from a stack, verifying their thickness as they pass through.

可能你会感到非常惊讶，答案是相当简单的。ATM机采用了一个比较简单的机制，从一叠钞票中一张一张的抽出，然后再通过时验证他们的厚度。

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

1. Don’t perform multiple operations before verification: Keep things simple and verify each step.

​     

1. Avoid deleting things for cleanup: Instead, quarantine problematic files to preserve state information that can help resolve issues later.
2. Don’t automatically fix the issue: Unless you’re sure about the problem, writing code to handle errors can cause additional ones.
3. Never fail without logging or isolating state: When something fails, gather relevant data and put it in a known error location.
4. Don’t perform operations in a shared location: If failure is possible, carry out the operation in a staging area first.

## Simplicity, Verification, and Effective Failure Handling

In a nutshell, the philosophy of designing software should be akin to the design of ATMs—embrace simplicity, ensure verification, and handle failure effectively. 

简而言之，软件设计哲学应当和ATM设计类似，崇尚简单，确保验证，有效的处理失败。

Software doesn’t need to be complex to be reliable; it needs to fail well. Let’s continue to draw inspiration from the physical engineering world in designing effective software mechanisms and remember this important lesson. 

软件的可靠并不需要借助复杂，他需要拥有良好的故障处理能力。让我们继续从物理工程里面吸取灵感，在软件设计中记录这个有效的教训。



## 下面是文章的评论



