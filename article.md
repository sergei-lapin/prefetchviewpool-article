# Фантастические RecyclerView.ViewHolder и где они создаются

Давайте представим, что вы уже cоптимизировали ваш ресайклер вдоль и поперек:

- setHasFixedSize(true)
- DiffUtil
- вьюшки более плоские чем Земля
- самый быстрый биндинг вьюхолдеров на Диком Западе

Но вам этого мало и вы продолжаете искать пути оптимизации. Поздравляю, вы попали в правильную статью! 

## Предыстория

Как-то раз, крася и двигая кнопки, я заметил, что ресайклер вью на одном из наших экранов немного проседал по отрисовке при активном скролле из-за большого разнообразия `viewType` (и, как следствие, частого создания новых вьюхолдеров), и, глядя на это, вспомнил о [старой статье Chet Haase](https://medium.com/google-developers/recyclerview-prefetch-c2f269075710) про то, как появилась возможность нагружать ресайклер работой по префетчингу элементов в моменты простоя мэйн трэда. Но мне показалось этого мало, и захотелось создавать элементы в моменты простоя...не мэйн трэда.

Так появилась идея подтюнить работу `RecyclerView.RecycledViewPool` для того, чтобы он заполнял себя вьюшками созданными off the main thread и отдавал их ресайклеру по запросу, т.е. занимался префетчингом. 

Помимо перфоманс бенефита при скролле через списки с большим разнообразием элементов от этого механизма удастся получить элементы тогда, когда, например, первая страница контента загружается и мы можем создать вьюшки под еще только загружающийся контент, тем самым ускоряя его показ нашему пользователю.

Эту идею я решил разбить на две составляющие: одна будет заниматься созданием вьюшек в бэкграунде (**Supplier**), а другая — их получением и последующей передачей ресайклеру в пользование (**Consumer**).

## RecyclerView.RecycledViewPool

Для начала стоит разобратся с тем, что такое `RecyclerView.RecycledViewPool`, чтобы на его основе собрать наш **Consumer**. 

`RecycledViewPool` — это хранилище переработанных вьюшек, из которого мы можем их доставать для нужного нам `viewType`, тем самым не создавая их лишний раз. Таким образом, именно `RecycledViewPool` лежит в основе богоподобной производительности `RecyclerView`. Помимо просто хранения переработанных вьюшек `RecycledViewPool` также хранит информацию о том, сколько примерно по времени вьюшки создаются и биндятся — с тем, чтобы в дальнейшем, по запросу `GapWorker`, достаточно правдоподобно "предсказывать", удастся ли эффективно утилизировать оставшееся время в кадре для того, чтобы упредительно создать вью того или иного `viewType`.

## Consumer

<details>
    <summary>Код Consumer</summary>

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

Теперь, когда мы разобрались с тем, что за зверь такой `RecyclerView.RecycledViewPool` — можно приступить к модификации его функциональности в угоду нашим потребностям (заблаговременное наполнение не только по запросу от `GapWorker`, но и от нашего **Supplier**)

Прежде всего, мы хотим добавить в API нашего префетчера возможность конфигурировать количество вьюшек того или иного типа, которые следует... префетчить.

```Kotlin
fun setPrefetchBound(viewType: Int, count: Int) {
    recycledViewsBounds[viewType] = max(defaultMaxRecycledViews, count)
    viewHolderSupplier.setPrefetchBound(viewType, count)
}
```

Здесь, помимо сохранения самого значения максимального числа хранимых вьюшек мы также сообщаем `ViewHolderSupplier` данные о том, сколько мы хотим от него вьюшек определенного типа посредством вызова `setPrefetchBound`, тем самым запуская процесс префетчинга. 

Далее, при попытке переработать вьюшку, мы будем сообщать вью пулу количество максимально возможных хранимых вьюшек, чтобы не хранить лишнего и актуализировать знание об этом числе, поскольку оно может меняться с течением времени.

```Kotlin
override fun putRecycledView(scrap: RecyclerView.ViewHolder) {
    val viewType = scrap.itemViewType
    val maxRecycledViews = recycledViewsBounds.getOrPut(viewType) { defaultMaxRecycledViews }
    setMaxRecycledViews(viewType, maxRecycledViews)
    super.putRecycledView(scrap)
}
```

Затем, когда к нашему вью пулу будет приходить запрос от ресайклера на получение ранее переработанной вьюшки, в случае если, в нашем пуле не окажется подготовленного/переработанного экземпляра, мы уведомим наш **Supplier** о том, что ресайклер сам создаст очередную вьюшку этого типа на мэйн трэде.

```Kotlin
override fun getRecycledView(viewType: Int): RecyclerView.ViewHolder? {
    val holder = super.getRecycledView(viewType)
    if (holder == null) viewHolderSupplier.onItemCreatedOutside(viewType)
    return holder
}
```

Осталось только связать жизненный цикл нашего вью пула и **Supplier**, и определить, что наш вью пул является **Consumer**'ом префетчнутых вьюшек:

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

Здесь стоит отметить метод `factorInCreateTime` — это метод, который сохраняет время создания вьюхолдера, на основе которого `GapWorker` будет делать выводы о том, сможет он что-нибудь префетчнуть своими силами в моменты простоя мэйн трэда или нет.

## Supplier

<details>
    <summary>Код Supplier</summary>

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

Теперь, когда мы разобрались с первой частью нашего механизма, давайте разберемся с реализацией второй — **Supplier**. Его основная задача — запускать очередь на создание вьюшек требуемого `viewType`, создавать их где-то вне мэйн трэда и передавать **Consumer**. Помимо этого, чтобы не делать лишней работы, ему следует реагировать на то, что вьюшка была создана за его пределами, уменьшая размер той самой очереди на создание.

### Запускаем очередь

Для запуска очереди на создание проверяем нет ли у нас в очереди уже достаточного количества вьюшек этого типа. Дальше проверим, не насоздавали ли мы уже достаточно вьюшек этого типа. И если оба эти условия проходят — добавляем в очередь запрос на то количество вьюшек, которое достаточно для того, чтобы довести количество уже созданных вьюшек до целевого значения `count`:

```Kotlin
internal fun setPrefetchBound(viewType: Int, count: Int) {
    if (itemsQueued.getOrZero(viewType) >= count) return
    itemsQueued[viewType] = count

    val created = itemsCreated.getOrZero(viewType)
    if (created >= count) return

    repeat(count - created) { enqueueItemCreation(viewType) }
}
```

### Создаем вьюшки

Что касается метода `enqueueItemCreation` — он будет абстрактным и его реализация будет зависеть от выбранного вашей командой подхода к многопоточности.

Но вот, что точно не будет абстрактным, так это метод `createItem`, который предполагается вызывать откуда-то из-за пределов мэйн трэда как раз посредством реализации `enqueueItemCreation`. В этом методе, прежде всего мы проверяем, а, вообще, нужно ли что-то делать или мы уже располагаем достаточным количеством закэшированных вьюшек? Затем мы создаем нашу вьюшку, запоминая время, затраченное на ее создание, игнорируя ошибки (пусть они там у себя в мэйн треде падают). После этого мы сообщим новому вьюхолдеру его `viewType`, просто чтобы он в курсе был, сделаем пометку, что создали еще один элемент с нужным `viewType` и оповестим **Consumer**'а об этом же.:

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

Здесь `viewHolderProducer` это простой `typealias`:

```Kotlin
typealias ViewHolderProducer = (parent: ViewGroup, viewType: Int) -> RecyclerView.ViewHolder
```

(на роль которого подходит `RecyclerView.Adapter.onCreateViewHolder`, например)

Стоит упомянуть, что традиционное создание вью через `LayoutInflater.inflate` — не самый эффективный способ утилизировать возможности представленного в статье механизма в силу синхронизации на аргументах конструктора...

### Реагируем на создание вьюшки вне **Supplier**

В случае создания вьюшки за пределами нашего **Supplier** (метод `onItemCreatedOutside`) — просто обновляем текущее знание о количестве уже созданных вьюшек данного типа:

```Kotlin
internal fun onItemCreatedOutside(viewType: Int) {
    itemsCreated[viewType] = itemsCreated.getOrZero(viewType) + 1
}
```

### Profit!

Теперь достаточно только определить поведение методов `start`, `stop` и `enqueueItemCreation` в зависимости от выбранного вашей командой подхода к работе с многопоточностью и вы получите вундервафлю по созданию вьюхолдеров за пределами основного потока, которую можно даже переиспользовать между экранами и подсовывать различным ресайклерам использующим одинаковые адаптеры/вьюшки, например...

Ну, а в случае, если вам не хочется задумываться над тем, как же все таки написать свою реализацию `ViewHolderSupplier`, то, только сегодня, специально для вас, я сделал [небольшую библиотечку](https://github.com/vivid-money/prefetchviewpool), в которой есть артефакт с `core` функциональностью, описанной в этой статье, а также артефакты под все основные подходы к многопоточности современности (Kotlin Coroutines, RxJava2, RxJava3, Executor).