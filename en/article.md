# Fantastic RecyclerView.ViewHolder and where to create them

![Fantastic RecyclerView.ViewHolder and where to create them](headline_image_en.png)

Let's assume that you've already optimized your `RecyclerView` back and forth:

- setHasFixedSize(true)
- DiffUtil
- views are more flat than Earth
- fastest `ViewHolder` binding in the Wild West

But you aren't satisfied with that and eager to find new ways of optimization. Congrats, you've landed in the right article!

## Background

Once upon a time, while painting and moving buttons, I noticed that `RecyclerView` on one of our screens was skipping frames a bit during active scrolling due to big variety of `viewType`s used (and therefore frequent `ViewHolder` creation). That reminded me about [old article by Chet Haase](https://medium.com/google-developers/recyclerview-prefetch-c2f269075710) about introducing prefetching mechanism to `RecyclerView` in order to prefetch items that are about to appear on screen during idle gaps on main thread. But it seemed to me that this wasn't enough, so I decided to try to create items during idle gaps on... background thread.

That's how the idea came about to tune work of `RecyclerView.RecycledViewPool` so that it would fill itself with views created off the main thread and offer them to `RecyclerView` on demand (i.e. prefetch).

Aside of performance benefits for scrolling through lists with big variety of elements, one could also get elements from that mechanism when, for example, first page of content is being loaded and views for that content can be created in advance, therefore speeding up their showing to the user.

I decided to break up that idea in two pieces: the first piece will do all the heavy-lifting on creating the views in background (**Supplier**), second will receive them and offer to `RecyclerView` on demand (**Consumer**)

## RecyclerView.RecycledViewPool

To begin with, it's worth understanding what `RecyclerView.RecycledViewPool` is, in order to build our **Consumer** on its basis.

`RecycledViewPool` — is a storage for... recycled views, from which we can retrieve them for the needed `viewType`, thereby not creating them once again. Thus, it is the `RecycledViewPool` that underlies famous `RecyclerView`'s performance. In addition to just storing recycled views `RecycledViewPool` also stores information about approximate time required for view creation and binding — so that in the future, at the request of `GapWorker`, it is plausible enough to "predict" whether it will be possible to effectively utilize the remaining time in the frame in order to preemptively create a view of a particular `viewType`.

## Consumer

<details>
    <summary>Full Consumer code</summary>

```Kotlin
class PrefetchViewPool(
    private val defaultMaxRecycledViews: Int,
    private val viewHolderSupplier: ViewHolderSupplier
) : RecyclerView.RecycledViewPool() {

    private val recycledViewsBounds = mutableMapOf<Int, Int>()

    init {
        attachToPreventFromClearing()
        viewHolderSupplier.viewHolderConsumer = ::putViewFromSupplier
        viewHolderSupplier.start()
    }

    fun setPrefetchBound(viewType: Int, count: Int) {
        recycledViewsBounds[viewType] = max(defaultMaxRecycledViews, count)
        viewHolderSupplier.setPrefetchBound(viewType, count)
    }

    override fun putRecycledView(scrap: RecyclerView.ViewHolder) {
        val viewType = scrap.itemViewType
        val maxRecycledViews = recycledViewsBounds.getOrPut(viewType) { defaultMaxRecycledViews }
        setMaxRecycledViews(viewType, maxRecycledViews)
        super.putRecycledView(scrap)
    }

    override fun getRecycledView(viewType: Int): RecyclerView.ViewHolder? {
        val holder = super.getRecycledView(viewType)
        if (holder == null) viewHolderSupplier.onItemCreatedOutside(viewType)
        return holder
    }

    override fun clear() {
        super.clear()
        viewHolderSupplier.stop()
    }

    private fun putViewFromSupplier(scrap: RecyclerView.ViewHolder, creationTimeNanos: Long) {
        factorInCreateTime(scrap.itemViewType, creationTimeNanos)
        putRecycledView(scrap)
    }
}
```
</details>

Now, when we understand what is `RecyclerView.RecycledViewPool` — we can start modifying its functionality in order to conform our needs (filling in advance not only by the request of `GapWorker`, but from out **Supplier** as well)

First of all, we'd want to extend an API of our prefetcher with an ability to configure amount of views of desired type, which are to be... prefetched.

```Kotlin
fun setPrefetchBound(viewType: Int, count: Int) {
    recycledViewsBounds[viewType] = max(defaultMaxRecycledViews, count)
    viewHolderSupplier.setPrefetchBound(viewType, count)
}
```

Besides saving the value of maximum number of stored views we are also informing `ViewHolderSupplier` about amount of views that we'd like to eventually get (by calling `setPrefetchBound`) and that actually triggers the process of prefetching.

Then, when recycler will try to recycle view, we are going to update view pool with maximum amount of stored views so as not to store redundant views and to actualize the knowledge about that number since it can vary over time.

```Kotlin
override fun putRecycledView(scrap: RecyclerView.ViewHolder) {
    val viewType = scrap.itemViewType
    val maxRecycledViews = recycledViewsBounds.getOrPut(viewType) { defaultMaxRecycledViews }
    setMaxRecycledViews(viewType, maxRecycledViews)
    super.putRecycledView(scrap)
}
```

After that, when our view pool will get a request from recycler for retrieving a recycled view and we'll not have one (previously recycled or prefetched) at its disposal then we'll notify our **Supplier** that recycler is going to create another view of that type on main thread.

```Kotlin
override fun getRecycledView(viewType: Int): RecyclerView.ViewHolder? {
    val holder = super.getRecycledView(viewType)
    if (holder == null) viewHolderSupplier.onItemCreatedOutside(viewType)
    return holder
}
```

The only thing that's left is to connect *lifecycle* of our view pool and the **Supplier**, and determine that our view pool is going to be a **Consumer** for our prefetched views:

```Kotlin
init {
    attachToPreventFromClearing()
    viewHolderSupplier.viewHolderConsumer = ::putViewFromSupplier
    viewHolderSupplier.start()
}

private fun putViewFromSupplier(scrap: RecyclerView.ViewHolder, creationTimeNanos: Long) {
    factorInCreateTime(scrap.itemViewType, creationTimeNanos)
    putRecycledView(scrap)
}

override fun clear() {
    super.clear()
    viewHolderSupplier.stop()
}
```

Method `factorInCreateTime` worth mentioning here. It's a method that saves time of view holder creation and with that knowledge `GapWorker` is going to make assumptions whether it's going to be able to prefetch views during gap between frames or not.

## Supplier

<details>
    <summary>Full Supplier code</summary>

```Kotlin
typealias ViewHolderProducer = (parent: ViewGroup, viewType: Int) -> RecyclerView.ViewHolder
typealias ViewHolderConsumer = (viewHolder: RecyclerView.ViewHolder, creationTimeNanos: Long) -> Unit


abstract class ViewHolderSupplier(
    context: Context,
    private val viewHolderProducer: ViewHolderProducer
) {

    internal lateinit var viewHolderConsumer: ViewHolderConsumer

    private val fakeParent: ViewGroup by lazy { FrameLayout(context) }
    private val mainHandler: Handler = Handler(Looper.getMainLooper())
    private val itemsCreated: MutableMap<Int, Int> = ConcurrentHashMap<Int, Int>()
    private val itemsQueued: MutableMap<Int, Int> = ConcurrentHashMap<Int, Int>()
    private val nanoTime: Long get() = System.nanoTime()

    abstract fun start()

    abstract fun enqueueItemCreation(viewType: Int)

    abstract fun stop()

    protected fun createItem(viewType: Int) {
        val created = itemsCreated.getOrZero(viewType) + 1
        val queued = itemsQueued.getOrZero(viewType)
        if (created > queued) return

        val holder: RecyclerView.ViewHolder
        val start: Long
        val end: Long

        try {
            start = nanoTime
            holder = viewHolderProducer.invoke(fakeParent, viewType)
            end = nanoTime
        } catch (e: Exception) {
            return
        }
        holder.setItemViewType(viewType)
        itemsCreated[viewType] = itemsCreated.getOrZero(viewType) + 1

        mainHandler.postAtFrontOfQueue { viewHolderConsumer.invoke(holder, end - start) }
    }

    internal fun setPrefetchBound(viewType: Int, count: Int) {
        if (itemsQueued.getOrZero(viewType) >= count) return
        itemsQueued[viewType] = count

        val created = itemsCreated.getOrZero(viewType)
        if (created >= count) return

        repeat(count - created) { enqueueItemCreation(viewType) }
    }

    internal fun onItemCreatedOutside(viewType: Int) {
        itemsCreated[viewType] = itemsCreated.getOrZero(viewType) + 1
    }

    private fun Map<Int, Int>.getOrZero(key: Int) = getOrElse(key) { 0 }
}
```
</details>  

Now, that we've dealt with the first part of our tool, let's dive into implementation of the second one — the **Supplier**. Its main goal is to enqueue item creation for certain `viewType`, create them somewhere off the main thread and pass them to the **Consumer**. Apart from that, in order to avoid unnecessary work it should react to the view being created outside of it in a way that it will decrease the amount of enqueued creations.

### Launching the queue

In order to launch the queue of item creation we check if there is enough items already enqueued. Then we check if we've already created enough views of that type. In case if both of these conditions meet — we can enqueue request for creating an amount of views that would be sufficient to reach target value of `count`:

```Kotlin
internal fun setPrefetchBound(viewType: Int, count: Int) {
    if (itemsQueued.getOrZero(viewType) >= count) return
    itemsQueued[viewType] = count

    val created = itemsCreated.getOrZero(viewType)
    if (created >= count) return

    repeat(count - created) { enqueueItemCreation(viewType) }
}
```

### Creating views

As for the `enqueueItemCreation` method — it will be abstract and its implementation will depend on the approach to multithreading chosen by your team.

But what will definitely not be abstract is the `createItem` method, which is supposed to be called from somewhere outside the main thread inside implemented `enqueueItemCreation`. In this method, first of all, we check whether something needs to be done or we already have a sufficient number of cached views? Then we create our view, remembering the time spent on its creation, ignoring errors (let them fail on main thread). After that, we will inform the new viewholder of its `viewType`, just so that he is aware, we will make a note that we have created another element with the necessary `viewType` and notify **Consumer** about the same:

```Kotlin
protected fun createItem(viewType: Int) {
    val created = itemsCreated.getOrZero(viewType) + 1
    val queued = itemsQueued.getOrZero(viewType)
    if (created > queued) return

    val holder: RecyclerView.ViewHolder
    val start: Long
    val end: Long

    try {
        start = nanoTime
        holder = viewHolderProducer.invoke(fakeParent, viewType)
        end = nanoTime
    } catch (e: Exception) {
        return
    }
    holder.setItemViewType(viewType)
    itemsCreated[viewType] = itemsCreated.getOrZero(viewType) + 1

    mainHandler.postAtFrontOfQueue { viewHolderConsumer.invoke(holder, end - start) }
}
```

Here `viewHolderProducer` is just a simple `typealias`:

```Kotlin
typealias ViewHolderProducer = (parent: ViewGroup, viewType: Int) -> RecyclerView.ViewHolder
```

(e.g. `RecyclerView.Adapter.onCreateViewHolder` can do the job)

It's worth mentioning that traditional view creation via `LayoutInflater.inflate` might be not the most effective way of utilising capabilities of the described above mechanism due to syncronization on constructor's arguments... 

### React on view created outside of **Supplier**

In case if view was created outside of our **Supplier** (`onItemCreatedOutside` method) we will just update current amount of total views created for that type: 

```Kotlin
internal fun onItemCreatedOutside(viewType: Int) {
    itemsCreated[viewType] = itemsCreated.getOrZero(viewType) + 1
}
```

### Profit!

Now you just need to determine the behavior of the `start`, `stop` and `enqueueItemCreation` methods, depending on the approach chosen by your team to work with multithreading, and you will get a wunderwafl for creating viewholders outside the main thread, which can even be reused between screens and supplied to different recyclers using the same adapters/views, for example...

In case you don't want to think about how to write your own implementation of `ViewHolderSupplier`, then, just today, especially for you, I made [a small library](https://github.com/vivid-money/prefetchviewpool), which has an artifact with the 'core' functionality described in this article, as well as artifacts for all the main approaches to modern multithreading (Kotlin Coroutines, RxJava2, RxJava3, Executor).