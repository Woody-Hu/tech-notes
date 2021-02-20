# About Unbounded Channel
## Questions
1. The relationship between `ConcurrentQueue`.
2. What happend under write async.
3. What happend under read async.
## Assumption
1. Channel may use concurrent queue directly for reuse
2. For read async channel may use [TaskCompletionSource](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcompletionsource-1?view=net-5.0) to achive read async.
3. Write async should return directly for unbounded channel.

## Source Code Analysis

1. about unbounded channnel class
``` C#
internal sealed class UnboundedChannel<T> : Channel<T>, IDebugEnumerable<T>
{
    private readonly TaskCompletionSource<VoidResult> _completion;
    private readonly ConcurrentQueue<T> _items = new ConcurrentQueue<T>();
    private readonly Deque<AsyncOperation<T>> _blockedReaders = new Deque<AsyncOperation<T>>();
    private readonly bool _runContinuationsAsynchronously;
    private AsyncOperation<bool>? _waitingReadersTail;
    private Exception? _doneWriting;
    private object SyncObj => _items;

    internal UnboundedChannel(bool runContinuationsAsynchronously)
    {
        _runContinuationsAsynchronously = runContinuationsAsynchronously;
        _completion = new TaskCompletionSource<VoidResult>(runContinuationsAsynchronously ? TaskCreationOptions.RunContinuationsAsynchronously : TaskCreationOptions.None);
        Reader = new UnboundedChannelReader(this);
        Writer = new UnboundedChannelWriter(this);
    }
```
we can find channel use ConcurrentQueue to store items directly, _blockedReaders and _waitingReadersTail should realte to read async, _completion and _doneWriting should realte to write complet logic. Moreover we can find reader and wriiter are singleton for a specific channel.

2. about unbounded channel writer 
``` c#
private sealed class UnboundedChannelWriter : ChannelWriter<T>, IDebugEnumerable<T>
{
    internal readonly UnboundedChannel<T> _parent;
    internal UnboundedChannelWriter(UnboundedChannel<T> parent) => _parent = parent;

    public override ValueTask<bool> WaitToWriteAsync(CancellationToken cancellationToken)
    {
        Exception? doneWriting = _parent._doneWriting;
        return
            cancellationToken.IsCancellationRequested ? new ValueTask<bool>(Task.FromCanceled<bool>(cancellationToken)) :
            doneWriting == null ? new ValueTask<bool>(true) : 
            doneWriting != ChannelUtilities.s_doneWritingSentinel ? new ValueTask<bool>(Task.FromException<bool>(doneWriting)) :
            default;
    }
}
```
For unbounded channel wait to wirte just return.

``` c#
public override bool TryWrite(T item)
{
    UnboundedChannel<T> parent = _parent;
    while (true)
    {
        AsyncOperation<T>? blockedReader = null;
        AsyncOperation<bool>? waitingReadersTail = null;
        lock (parent.SyncObj)
        {
            parent.AssertInvariants();
            if (parent._doneWriting != null)
            {
                return false;
            }

            if (parent._blockedReaders.IsEmpty)
            {
                parent._items.Enqueue(item);
                waitingReadersTail = parent._waitingReadersTail;
                if (waitingReadersTail == null)
                {
                    return true;
                }
                parent._waitingReadersTail = null;
            }
            else
            {
                blockedReader = parent._blockedReaders.DequeueHead();
            }
        }

        if (blockedReader != null)
        {
            if (blockedReader.TrySetResult(item))
            {
                return true;
            }
        }
        else
        {
            ChannelUtilities.WakeUpWaiters(ref waitingReadersTail, result: true);
            return true;
        }
    }
}

public override ValueTask WriteAsync(T item, CancellationToken cancellationToken) =>
    cancellationToken.IsCancellationRequested ? new ValueTask(Task.FromCanceled(cancellationToken)) :
    TryWrite(item) ? default :
    new ValueTask(Task.FromException(ChannelUtilities.CreateInvalidCompletionException(_parent._doneWriting)));

```
For try write unbound channel use a while true internal for retry, and will do below logics:
    2.1 lock the queue to make sure just one thread can modify states such as blocked reader, compelted.
    2.2 if this channel is finished just reutrn.
    2.3 if do not have blocked readers, put item to the queue directly and get waiting readers(the reads who are waiting to read) and these readers will try to read this item latter.
    2.4 else get first blocked readers and set result for it.
And Write Async just combine try write and return directly.

``` c#
public override bool TryComplete(Exception? error)
{
    UnboundedChannel<T> parent = _parent;
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
    ChannelUtilities.WakeUpWaiters(ref parent._waitingReadersTail, result: false, error: error);
    return true;
}
```
When complete writer will use lock to make asuer just one thread can modify states, due to the lock so if one thread set channel is completed other threads can not write to it. And notice blocked and waiting readers.

3. about unbounded channel reader
``` c#
private sealed class UnboundedChannelReader : ChannelReader<T>, IDebugEnumerable<T>
{
    internal readonly UnboundedChannel<T> _parent;
    private readonly AsyncOperation<T> _readerSingleton;
    private readonly AsyncOperation<bool> _waiterSingleton;

    internal UnboundedChannelReader(UnboundedChannel<T> parent)
    {
        _parent = parent;
        _readerSingleton = new AsyncOperation<T>(parent._runContinuationsAsynchronously, pooled: true);
        _waiterSingleton = new AsyncOperation<bool>(parent._runContinuationsAsynchronously, pooled: true);
    }

    public override Task Completion => _parent._completion.Task;

    public override bool TryRead([MaybeNullWhen(false)] out T item)
    {
        UnboundedChannel<T> parent = _parent;

        // Dequeue an item if we can
        if (parent._items.TryDequeue(out item))
        {
            CompleteIfDone(parent);
            return true;
        }

        item = default;
        return false;
    }

    private void CompleteIfDone(UnboundedChannel<T> parent)
    {
        if (parent._doneWriting != null && parent._items.IsEmpty)
        {
            ChannelUtilities.Complete(parent._completion, parent._doneWriting);
        }
    }
}
```

Readere's try read method just base on concurrent queue's try dequeue method, and will try to complete channel if we can.

``` c#
public override ValueTask<T> ReadAsync(CancellationToken cancellationToken)
{
    if (cancellationToken.IsCancellationRequested)
    {
        return new ValueTask<T>(Task.FromCanceled<T>(cancellationToken));
    }

    UnboundedChannel<T> parent = _parent;
    if (parent._items.TryDequeue(out T item))
    {
        CompleteIfDone(parent);
        return new ValueTask<T>(item);
    }

    lock (parent.SyncObj)
    {
        parent.AssertInvariants();
        if (parent._items.TryDequeue(out item))
        {
            CompleteIfDone(parent);
            return new ValueTask<T>(item);
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

        var reader = new AsyncOperation<T>(parent._runContinuationsAsynchronously, cancellationToken);
        parent._blockedReaders.EnqueueTail(reader);
        return reader.ValueTaskOfT;
    }
}
```
read async have more logic, reader will:
1. try read to achive better performance.
2. lock the queue(due we need to modify block readers)
3. check again to avoid resource lost.
4. check if channel has been close write.
5. append a waiting reader (try to reuse singleton first)

``` c#
 public override ValueTask<bool> WaitToReadAsync(CancellationToken cancellationToken)
{
    if (cancellationToken.IsCancellationRequested)
    {
        return new ValueTask<bool>(Task.FromCanceled<bool>(cancellationToken));
    }

    if (!_parent._items.IsEmpty)
    {
        return new ValueTask<bool>(true);
    }

    UnboundedChannel<T> parent = _parent;
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

        var waiter = new AsyncOperation<bool>(parent._runContinuationsAsynchronously, cancellationToken);
        ChannelUtilities.QueueWaiter(ref parent._waitingReadersTail, waiter);
        return waiter.ValueTaskOfT;
    }
}
```
like ReadAsync Wait to read do the nearly same logic, but read async append a blocked reader wait to read append a waiting reader.

4. about AsyncOperation
``` c#
internal partial class AsyncOperation<TResult> : AsyncOperation, IValueTaskSource, IValueTaskSource<TResult>
{
    private readonly CancellationTokenRegistration _registration;
    private readonly bool _pooled;
    private readonly bool _runContinuationsAsynchronously;
    private volatile int _completionReserved;
    private TResult? _result;
    private ExceptionDispatchInfo? _error;
    private short _currentId;

    public AsyncOperation(bool runContinuationsAsynchronously, CancellationToken cancellationToken = default, bool pooled = false)
    {
        _continuation = pooled ? s_availableSentinel : null;
        _pooled = pooled;
        _runContinuationsAsynchronously = runContinuationsAsynchronously;
        if (cancellationToken.CanBeCanceled)
        {
            Debug.Assert(!_pooled, "Cancelable operations can't be pooled");
            CancellationToken = cancellationToken;
            _registration = UnsafeRegister(cancellationToken, static s =>
            {
                var thisRef = (AsyncOperation<TResult>)s!;
                thisRef.TrySetCanceled(thisRef.CancellationToken);
            }, this);
        }
    }

    public AsyncOperation<TResult>? Next { get; set; }
    public CancellationToken CancellationToken { get; }
    public ValueTask ValueTask => new ValueTask(this, _currentId);
    public ValueTask<TResult> ValueTaskOfT => new ValueTask<TResult>(this, _currentId);

    public ValueTaskSourceStatus GetStatus(short token)
    {
        if (_currentId != token)
        {
            ThrowIncorrectCurrentIdException();
        }

        return
            !IsCompleted ? ValueTaskSourceStatus.Pending :
            _error == null ? ValueTaskSourceStatus.Succeeded :
            _error.SourceException is OperationCanceledException ? ValueTaskSourceStatus.Canceled :
            ValueTaskSourceStatus.Faulted;
    }

    internal bool IsCompleted => ReferenceEquals(_continuation, s_completedSentinel);

    public TResult GetResult(short token)
    {
        if (_currentId != token)
        {
            ThrowIncorrectCurrentIdException();
        }

        if (!IsCompleted)
        {
            ThrowIncompleteOperationException();
        }

        ExceptionDispatchInfo? error = _error;
        TResult result = _result;
        _currentId++;

        if (_pooled)
        {
            Volatile.Write(ref _continuation, s_availableSentinel);
        }

        error?.Throw();
        return result!;
    }

    void IValueTaskSource.GetResult(short token)
    {
        if (_currentId != token)
        {
            ThrowIncorrectCurrentIdException();
        }

        if (!IsCompleted)
        {
            ThrowIncompleteOperationException();
        }

        ExceptionDispatchInfo? error = _error;
        _currentId++;

        if (_pooled)
        {
            Volatile.Write(ref _continuation, s_availableSentinel); // only after fetching all needed data
        }

        error?.Throw();
    }

    public bool TryOwnAndReset()
    {
        Debug.Assert(_pooled, "Should only be used for pooled objects");
        if (ReferenceEquals(Interlocked.CompareExchange(ref _continuation, null, s_availableSentinel), s_availableSentinel))
        {
            _continuationState = null;
            _result = default;
            _error = null;
            _schedulingContext = null;
            _executionContext = null;
            return true;
        }

        return false;
    }

    public bool TrySetResult(TResult item)
    {
        UnregisterCancellation();

        if (TryReserveCompletionIfCancelable())
        {
            _result = item;
            SignalCompletion();
            return true;
        }

        return false;
    }

    public bool TrySetException(Exception exception)
    {
        UnregisterCancellation();

        if (TryReserveCompletionIfCancelable())
        {
            _error = ExceptionDispatchInfo.Capture(exception);
            SignalCompletion();
            return true;
        }

        return false;
    }

    public bool TrySetCanceled(CancellationToken cancellationToken = default)
    {
        if (TryReserveCompletionIfCancelable())
        {
            _error = ExceptionDispatchInfo.Capture(new OperationCanceledException(cancellationToken));
            SignalCompletion();
            return true;
        }

        return false;
    }
}
```

Async is an `IValueTaskSource<TResult>`, and have a linkedlist structure and porvide pool logic at same time.(I have romve some logic about how to handle continue)

5. about ChannelUtilities
``` c#
internal static class ChannelUtilities
{
    internal static readonly Exception s_doneWritingSentinel = new Exception(nameof(s_doneWritingSentinel));
    internal static readonly Task<bool> s_trueTask = Task.FromResult(result: true);
    internal static readonly Task<bool> s_falseTask = Task.FromResult(result: false);
    internal static readonly Task s_neverCompletingTask = new TaskCompletionSource<bool>().Task;

    internal static void QueueWaiter(ref AsyncOperation<bool>? tail, AsyncOperation<bool> waiter)
    {
        AsyncOperation<bool>? c = tail;
        if (c == null)
        {
            waiter.Next = waiter;
        }
        else
        {
            waiter.Next = c.Next;
            c.Next = waiter;
        }
        tail = waiter;
    }

    internal static void WakeUpWaiters(ref AsyncOperation<bool>? listTail, bool result, Exception? error = null)
    {
        AsyncOperation<bool>? tail = listTail;
        if (tail != null)
        {
            listTail = null;

            AsyncOperation<bool> head = tail.Next!;
            AsyncOperation<bool> c = head;
            do
            {
                AsyncOperation<bool> next = c.Next!;
                c.Next = null;
                bool completed = error != null ? c.TrySetException(error) : c.TrySetResult(result);
                Debug.Assert(completed || c.CancellationToken.CanBeCanceled);

                c = next;
            }
            while (c != head);
        }
    }
}
```

A major logic for channel utilites is about how to handle AsyncOperation's linked list, from code we can found ayncoperation is a loop linked list. Due to channel use lock so it is thread safe.

## Conclution 
1. channel internal use `ConcurrentQueue` and internal use lock and other logics so channel can not achive better performance than `ConcurrentQueue`

2. wait to write do not do any "async" logic

3. write and read will use async operation (an `IValueTaskSource<TResult>`) to achive async logic.

    