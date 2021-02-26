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

for unbounded channel we also need to notify waiting writers.

