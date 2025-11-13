---
title: 双栈实现的队列，以及其使用场景（无锁并发队列）
published: 2025-11-13
description: 基于双栈实现无锁队列的思路
tags: [数据结构, 多线程]
category: 编程
draft: false
---

似乎挺经典的一道面试题

``` c#
class Queue<T>
{
    private readonly Stack<T> _in = [];
    private readonly Stack<T> _out = [];

    public void Enqueue(T item) => _in.Push(item);

    public T Dequeue()
    {
        if (_out.TryPop(out var item))
            return item;
        
        while (_in.TryPop(out var pop))
            _out.Push(pop);

        if (_out.TryPop(out var item))
            return item;
        else 
            throw new InvalidOperation("No element");
    }
}
```

实现很简单，但是问题在于这有啥用。看起来对比一般默认的环形缓冲区实现甚至`List<T>`模拟都没有任何优势。

于是在群里问了一下。

> Trarizon: 说起来我刚刚技术面的时候被问了怎么用两个stack拼成一个queue
>
> Trarizon: 我感觉解法很抽象，有没有会的
>
> ...
>
> 盐酸: *\*口述了一下上述实现\**
>
> 盐酸: 这种算法我可能实现过亿些次，用来做原子的内嵌队列。盐酸: 不是纯toy问题，是有用的
>
> 盐酸: 另外为什么会有这种用stack不用list的场景，总之就是atomic queue的话你要维护的invariant太多就只能加锁了

> 盐酸: [睦子米ぜーんぶ弾けなーい.gif]

嗯。。。虽然感觉我用C#写前端遇不到这个场景，但还是去看了一下

Gemini给的解释。分离之后，两个栈分别提供给producer和consumer，因此可以使用原子操作实现无锁并发，仅在dequeue且`out_stack`为空时，需要从`input_stack`转移数据。

c++可以直接`atomic_pot<T>`保证原子操作，但是我好久没写c++了还是c#吧

> [!WARNING]
> 本人并没有什么手搓并发的经验所以以下代码很可能并不安全，也没有经过测试，甚至会爆一堆nullability warning，仅供参考。在.NET中需要这东西请使用ConcurrentQueue<T>，如果需要拿.NET手搓unsafe基建的水平那我想并不需要看这篇文章（

``` c#
partial class Queue<T>
{
    private Node _in;
    private Node _out;
    private bool _transferring;
    class Node
    {
        public T Value;
        public Node? Next;
    }
}
```

`Enqueue`就是一个很标准的Compare and exchange，保证producer的无锁并发。

``` c#
partial class Queue<T>
{
    public void Enqueue(T item)
    {
        var node = new Node { Value = item };
        while (true)
        {
            var old_in = _in;
            node.Next = old_in;
            if (old_in == Interlocked.CompareExchange(ref _in, node, old_in))
                return;
        }
    }
}
```

`Dequeue`首先如果`out_stack`中存在item，那么也是一个很标准的compare and exchange。但是如果`out_stack`为空，就需要将`in_stack`中的元素移到`out_stack`。

``` c#
partial class Queue<T>
{
    public T Dequeue()
    {
        while (true)
        {
            var old_out = _out;
            if (old_out != null)
            {
                var new_out = old_out.Next;
                if (old_out == Interlocked.CompareExchange(ref _out, new_out, old_out))
                    return old_out.Value;
                goto Retry;
            }

            // 确保只有一个线程在处理数据转移，其他线程等着就是
            if (Interlocked.CompareExchange(ref _transferring, true, false) == false)
            {
                var old_in = Interlocked.Exchange(ref _in, null);
                if (old_in is null) {
                    Interlocked.Exchange(ref _transferring, false);
                    throw new InvalidOperationException("Empty");
                }

                _out = Reverse(old_in);
                Interlocked.Exchange(ref _transferring, false);
                goto Retry;
            }
            else
            {
                var spinner = new SpinWait();
                while (_transferring)
                    spinner.SpinOnce();
                goto Retry;
            }
        }

        static Node Reverse([DisallowNull] Node? node)
        {
            Node? prev = null;
            while (node != null) {
                var next = node.Next;
                node.Next = prev;
                prev = node;
                node = next;
            }
            return prev!;
        }
    }
}
```

写完感觉很符合我对无锁并发一句话十行的刻板印象.jpg，嘛反正就是通过两个stack结构分离了producer和consumer，提供了更安全的并发方式。