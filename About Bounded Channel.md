# About Bounded Channel
## Question
1. How bounded channel handle bounded concept

## Assumption
1. Should use AsyncOperation to handle bounded logic

## Source Code Analysis

1. Bounded Channel

``` c#
internal sealed class BoundedChannel<T> : Channel<T>, IDebugEnumerable<T>
{
    private readonly BoundedChannelFullMode _mode;
    private readonly int _bufferedCapacity;

    .....

    internal BoundedChannel(int bufferedCapacity, BoundedChannelFullMode mode, bool runContinuationsAsynchronously)
    {
        Debug.Assert(bufferedCapacity > 0);
        _bufferedCapacity = bufferedCapacity;
        _mode = mode;
        _runContinuationsAsynchronously = runContinuationsAsynchronously;
        _completion = new TaskCompletionSource<VoidResult>(runContinuationsAsynchronously ? TaskCreationOptions.RunContinuationsAsynchronously : TaskCreationOptions.None);
        Reader = new BoundedChannelReader(this);
        Writer = new BoundedChannelWriter(this);
    }
}
```

we can found bounded channel also provide singleton reader and writer.

2. Reader

The most of logic of BoundedChannelReader is like UnboundedChannelReader.
But reader will notice writer 
Try read method
``` c#
public override bool TryRead([MaybeNullWhen(false)] out T item)
{
    BoundedChannel<T> parent = _parent;
    lock (parent.SyncObj)
    {
        parent.AssertInvariants();

        // Get an item if there is one.
        if (!parent._items.IsEmpty)
        {
            item = DequeueItemAndPostProcess();
            return true;
        }
    }

    item = default;
    return false;
}
```

will use lock and handle waiting writers.

DequeueItemAndPostProcess Method
``` c#
private T DequeueItemAndPostProcess()
{
    BoundedChannel<T> parent = _parent;
    Debug.Assert(Monitor.IsEntered(parent.SyncObj));
    T item = parent._items.DequeueHead();

    if (parent._doneWriting != null)
    {
        if (parent._items.IsEmpty)
        {
            ChannelUtilities.Complete(parent._completion, parent._doneWriting);
        }
    }
    else
    {
        while (!parent._blockedWriters.IsEmpty)
        {
            VoidAsyncOperationWithData<T> w = parent._blockedWriters.DequeueHead();
            if (w.TrySetResult(default))
            {
                parent._items.EnqueueTail(w.Item!);
                return item;
            }
        }

        ChannelUtilities.WakeUpWaiters(ref parent._waitingWritersTail, result: true);
    }

    return item;
}

```

the logic will try to complete channel and try to find a blocked writer get the value to the end of queue and notice all wairing writers.

ReadAsync Method
``` c#
public override ValueTask<T> ReadAsync(CancellationToken cancellationToken)
{
    if (cancellationToken.IsCancellationRequested)
    {
        return new ValueTask<T>(Task.FromCanceled<T>(cancellationToken));
    }

    BoundedChannel<T> parent = _parent;
    lock (parent.SyncObj)
    {
        parent.AssertInvariants();
        if (!parent._items.IsEmpty)
        {
            return new ValueTask<T>(DequeueItemAndPostProcess());
        }

        if (parent._doneWriting != null)
        {
            return ChannelUtilities.GetInvalidCompletionValueTask<T>(parent._doneWriting);
        }

        if (!cancellationToken.CanBeCanceled)
        {
            AsyncOperation<T> singleton = _readerSingleton;
            if (singleton.TryOwnAndReset())
            {
                parent._blockedReaders.EnqueueTail(singleton);
                return singleton.ValueTaskOfT;
            }
        }

        var reader = new AsyncOperation<T>(parent._runContinuationsAsynchronously | cancellationToken.CanBeCanceled, cancellationToken);
        parent._blockedReaders.EnqueueTail(reader);
        return reader.ValueTaskOfT;
    }
}
```

For read async method the different part is when can read some thing notice writer. the last logic is same, check if can complete write, if can reuse singleton blocker readers.

``` c#
public override ValueTask<bool> WaitToReadAsync(CancellationToken cancellationToken)
{
    if (cancellationToken.IsCancellationRequested)
    {
        return new ValueTask<bool>(Task.FromCanceled<bool>(cancellationToken));
    }

    BoundedChannel<T> parent = _parent;
    lock (parent.SyncObj)
    {
        parent.AssertInvariants();
        if (!parent._items.IsEmpty)
        {
            return new ValueTask<bool>(true);
        }

        if (parent._doneWriting != null)
        {
            return parent._doneWriting != ChannelUtilities.s_doneWritingSentinel ?
                new ValueTask<bool>(Task.FromException<bool>(parent._doneWriting)) :
                default;
        }

        if (!cancellationToken.CanBeCanceled)
        {
            AsyncOperation<bool> singleton = _waiterSingleton;
            if (singleton.TryOwnAndReset())
            {
                ChannelUtilities.QueueWaiter(ref parent._waitingReadersTail, singleton);
                return singleton.ValueTaskOfT;
            }
        }

        var waiter = new AsyncOperation<bool>(parent._runContinuationsAsynchronously | cancellationToken.CanBeCanceled, cancellationToken);
        ChannelUtilities.QueueWaiter(ref _parent._waitingReadersTail, waiter);
        return waiter.ValueTaskOfT;
    }
}
```

the logic is same as unbounded channel reader.

2. Writer

``` c#
private sealed class BoundedChannelWriter : ChannelWriter<T>, IDebugEnumerable<T>
{
    internal readonly BoundedChannel<T> _parent;
    private readonly VoidAsyncOperationWithData<T> _writerSingleton;
    private readonly AsyncOperation<bool> _waiterSingleton;

    internal BoundedChannelWriter(BoundedChannel<T> parent)
    {
        _parent = parent;
        _writerSingleton = new VoidAsyncOperationWithData<T>(runContinuationsAsynchronously: true, pooled: true);
        _waiterSingleton = new AsyncOperation<bool>(runContinuationsAsynchronously: true, pooled: true);
    }

    public override bool TryComplete(Exception? error)
    {
        BoundedChannel<T> parent = _parent;
        bool completeTask;
        lock (parent.SyncObj)
        {
            parent.AssertInvariants();
            if (parent._doneWriting != null)
            {
                return false;
            }

            parent._doneWriting = error ?? ChannelUtilities.s_doneWritingSentinel;
            completeTask = parent._items.IsEmpty;
        }

        if (completeTask)
        {
            ChannelUtilities.Complete(parent._completion, error);
        }


        ChannelUtilities.FailOperations<AsyncOperation<T>, T>(parent._blockedReaders, ChannelUtilities.CreateInvalidCompletionException(error));
        ChannelUtilities.FailOperations<VoidAsyncOperationWithData<T>, VoidResult>(parent._blockedWriters, ChannelUtilities.CreateInvalidCompletionException(error));
        ChannelUtilities.WakeUpWaiters(ref parent._waitingReadersTail, result: false, error: error);
        ChannelUtilities.WakeUpWaiters(ref parent._waitingWritersTail, result: false, error: error);
        return true;
    }
}
```

in bounded channel try complete have more logices
``` c#
        ChannelUtilities.FailOperations<VoidAsyncOperationWithData<T>, VoidResult>(parent._blockedWriters, ChannelUtilities.CreateInvalidCompletionException(error));
        ChannelUtilities.WakeUpWaiters(ref parent._waitingWritersTail, result: false, error: error);
```

for unbounded channel we also need to notify waiting writers. This because bounded channel need to notice writer too.

TryWriteMethod
``` c#
 public override bool TryWrite(T item)
{
    AsyncOperation<T>? blockedReader = null;
    AsyncOperation<bool>? waitingReadersTail = null;

    BoundedChannel<T> parent = _parent;
    lock (parent.SyncObj)
    {
        parent.AssertInvariants();

        if (parent._doneWriting != null)
        {
            return false;
        }

        int count = parent._items.Count;
        if (count == 0)
        {
            while (!parent._blockedReaders.IsEmpty)
            {
                AsyncOperation<T> r = parent._blockedReaders.DequeueHead();
                if (r.UnregisterCancellation()) 
                {
                    blockedReader = r;
                    break;
                }
            }

            if (blockedReader == null)
            {
                parent._items.EnqueueTail(item);
                waitingReadersTail = parent._waitingReadersTail;
                if (waitingReadersTail == null)
                {
                    return true;
                }
                parent._waitingReadersTail = null;
            }
        }
        else if (count < parent._bufferedCapacity)
        {
            parent._items.EnqueueTail(item);
            return true;
        }
        else if (parent._mode == BoundedChannelFullMode.Wait)
        {
            return false;
        }
        else if (parent._mode == BoundedChannelFullMode.DropWrite)
        {
            return true;
        }
        else
        {
            if (parent._mode == BoundedChannelFullMode.DropNewest)
            {
                parent._items.DequeueTail();
            }
            else
            {
                parent._items.DequeueHead();
            }
            parent._items.EnqueueTail(item);
            return true;
        }
    }

    if (blockedReader != null)
    {
        Debug.Assert(waitingReadersTail == null, "Shouldn't have any waiters to wake up");
        bool success = blockedReader.TrySetResult(item);
        Debug.Assert(success, "We should always be able to complete the reader.");
    }
    else
    {
        ChannelUtilities.WakeUpWaiters(ref waitingReadersTail, result: true);
    }

    return true;
}
```

WaitToWriteAsync Method
``` c#
public override ValueTask<bool> WaitToWriteAsync(CancellationToken cancellationToken)
{
    if (cancellationToken.IsCancellationRequested)
    {
        return new ValueTask<bool>(Task.FromCanceled<bool>(cancellationToken));
    }

    BoundedChannel<T> parent = _parent;
    lock (parent.SyncObj)
    {
        parent.AssertInvariants();
        if (parent._doneWriting != null)
        {
            return parent._doneWriting != ChannelUtilities.s_doneWritingSentinel ?
                new ValueTask<bool>(Task.FromException<bool>(parent._doneWriting)) :
                default;
        }

        if (parent._items.Count < parent._bufferedCapacity || parent._mode != BoundedChannelFullMode.Wait)
        {
            return new ValueTask<bool>(true);
        }

        if (!cancellationToken.CanBeCanceled)
        {
            AsyncOperation<bool> singleton = _waiterSingleton;
            if (singleton.TryOwnAndReset())
            {
                ChannelUtilities.QueueWaiter(ref parent._waitingWritersTail, singleton);
                return singleton.ValueTaskOfT;
            }
        }

        var waiter = new AsyncOperation<bool>(runContinuationsAsynchronously: true, cancellationToken);
        ChannelUtilities.QueueWaiter(ref parent._waitingWritersTail, waiter);
        return waiter.ValueTaskOfT;
    }
}

``` c#
public override ValueTask WriteAsync(T item, CancellationToken cancellationToken)
{
    if (cancellationToken.IsCancellationRequested)
    {
        return new ValueTask(Task.FromCanceled(cancellationToken));
    }

    AsyncOperation<T>? blockedReader = null;
    AsyncOperation<bool>? waitingReadersTail = null;

    BoundedChannel<T> parent = _parent;
    lock (parent.SyncObj)
    {
        parent.AssertInvariants();
        if (parent._doneWriting != null)
        {
            return new ValueTask(Task.FromException(ChannelUtilities.CreateInvalidCompletionException(parent._doneWriting)));
        }

        int count = parent._items.Count;

        if (count == 0)
        {
            while (!parent._blockedReaders.IsEmpty)
            {
                AsyncOperation<T> r = parent._blockedReaders.DequeueHead();
                if (r.UnregisterCancellation()) 
                {
                    blockedReader = r;
                    break;
                }
            }

            if (blockedReader == null)
            {
                parent._items.EnqueueTail(item);
                waitingReadersTail = parent._waitingReadersTail;
                if (waitingReadersTail == null)
                {
                    return default;
                }
                parent._waitingReadersTail = null;
            }
        }
        else if (count < parent._bufferedCapacity)
        {
            parent._items.EnqueueTail(item);
            return default;
        }
        else if (parent._mode == BoundedChannelFullMode.Wait)
        {
            if (!cancellationToken.CanBeCanceled)
            {
                VoidAsyncOperationWithData<T> singleton = _writerSingleton;
                if (singleton.TryOwnAndReset())
                {
                    singleton.Item = item;
                    parent._blockedWriters.EnqueueTail(singleton);
                    return singleton.ValueTask;
                }
            }

            var writer = new VoidAsyncOperationWithData<T>(runContinuationsAsynchronously: true, cancellationToken);
            writer.Item = item;
            parent._blockedWriters.EnqueueTail(writer);
            return writer.ValueTask;
        }
        else if (parent._mode == BoundedChannelFullMode.DropWrite)
        {
            return default;
        }
        else
        {
            if (parent._mode == BoundedChannelFullMode.DropNewest)
            {
                parent._items.DequeueTail();
            }
            else
            {
                parent._items.DequeueHead();
            }
            parent._items.EnqueueTail(item);
            return default;
        }
    }

    if (blockedReader != null)
    {
        // Transfer the written item to the blocked reader.
        bool success = blockedReader.TrySetResult(item);
        Debug.Assert(success, "We should always be able to complete the reader.");
    }
    else
    {
        ChannelUtilities.WakeUpWaiters(ref waitingReadersTail, result: true);
    }

    return default;
}
```

