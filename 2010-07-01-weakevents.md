[WeakReference Event Handlers by Paul Stovell](http://paulstovell.com/blog/weakevents)
===============================================

A good rule of thumb to live by is that **long-lived** objects should avoid referencing **short-lived** objects.

The reason for this is that the .NET garbage collector uses a [mark and sweep algorithm][1] to detemine if it can delete and reclaim an object. If it determines that a long-lived object should be kept alive (because you are using it, or because it's in a static field somewhere), it also assumes anything it references is being kept alive.

 [1]: http://en.wikipedia.org/wiki/Garbage_collection_(computer_science)#Tracing_garbage_collectors

Conversely, going the other way is fine - a short-lived object can reference a long-lived object because the garbage collector will happily delete it if nothing else uses it.

For example:

 1. You shouldn't add items to a static collection, if those items won't be around for a while
 2. You shouldn't subscribe to static events from a short-lived object

The second example often throws people not familiar with how events work in .NET. When you subscribe to an event, the event handler keeps a list of subscribers. When the event is raised, it loops through the subscribers and notifies each one - it's a simple form of the [observer][2] pattern.

 [2]: http://www.dofactory.com/Patterns/PatternObserver.aspx

If you do find yourself needing to write this kind of code, and there isn't a good alternative design, then you generally need to have an unhook option. You might have a way to "remove" the short-lived object from the collection managed by the long-lived object, or you might unsubscribe from an event.

When unsubscribing isn't an option (because you don't trust people to call your Dispose/Unsubscribe method), you can make use of **weak event handlers**. WPF has its own implementation, but it's too complex for my feeble mind. Here's a simple snippet that I use:

```cs
[DebuggerNonUserCode]
public sealed class WeakEventHandler<TEventArgs> where TEventArgs : EventArgs
{
    private readonly WeakReference _targetReference;
    private readonly MethodInfo _method;

    public WeakEventHandler(EventHandler<TEventArgs> callback)
    {
        _method = callback.Method;
        _targetReference = new WeakReference(callback.Target, true);
    }

    [DebuggerNonUserCode]
    public void Handler(object sender, TEventArgs e)
    {
        var target = _targetReference.Target;
        if (target != null)
        {
            var callback = (Action<object, TEventArgs>)Delegate.CreateDelegate(typeof(Action<object, TEventArgs>), target, _method, true);
            if (callback != null)
            {
                callback(sender, e);
            }
        }
    }
}
```

When subscribing to events, instead of writing:

```cs
alarm.Beep += Alarm_Beeped;
```

Just write:

```cs
alarm.Beeped += new WeakEventHandler<AlarmEventArgs>(Alarm_Beeped).Handler;
```
Your subscriber can now be garbage collected without needing to manually unsubscribe (and without having to remember to). Here are some tests:

```cs
[TestFixture]
public class WeakEventsTests
{
    #region Example

    public class Alarm
    {
        public event PropertyChangedEventHandler Beeped;

        public void Beep()
        {
            var handler = Beeped;
            if (handler != null) handler(this, new PropertyChangedEventArgs("Beep!"));
        }
    }

    public class Sleepy
    {
        private readonly Alarm _alarm;
        private int _snoozeCount;

        public Sleepy(Alarm alarm)
        {
            _alarm = alarm;
            _alarm.Beeped += new WeakEventHandler<PropertyChangedEventArgs>(Alarm_Beeped).Handler;
        }

        private void Alarm_Beeped(object sender, PropertyChangedEventArgs e)
        {
            _snoozeCount++;
        }

        public int SnoozeCount
        {
            get { return _snoozeCount; }
        }
    }

    #endregion

    [Test]
    public void ShouldHandleEventWhenBothReferencesAreAlive()
    {
        var alarm = new Alarm();
        var sleepy = new Sleepy(alarm);
        alarm.Beep();
        alarm.Beep();

        Assert.AreEqual(2, sleepy.SnoozeCount);
    }

    [Test]
    public void ShouldAllowSubscriberReferenceToBeCollected()
    {
        var alarm = new Alarm();
        var sleepyReference = null as WeakReference;
        new Action(() =>
        {
            // Run this in a delegate to that the local variable gets garbage collected
            var sleepy = new Sleepy(alarm);
            alarm.Beep();
            alarm.Beep();
            Assert.AreEqual(2, sleepy.SnoozeCount);
            sleepyReference = new WeakReference(sleepy);
        })();

        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();

        Assert.IsNull(sleepyReference.Target);
    }

    [Test]
    public void SubscriberShouldNotBeUnsubscribedUntilCollection()
    {
        var alarm = new Alarm();
        var sleepy = new Sleepy(alarm);

        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();

        alarm.Beep();
        alarm.Beep();
        Assert.AreEqual(2, sleepy.SnoozeCount);
    }
}
```
