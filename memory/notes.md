#Java Memory model
Memory consists of: 
* **Stack** (per thread)
* **Metaspace** (formely PermGen)
it has metadata, classes, methods code, static vars, constant pools (that reside in .class files)
Main difference from PermGen is that it can grow (take more memory)
* **Heap** (contains string pool since Java 8, before it was located in PermGen):
* **Young Generation** is split to Eden, S0 and S1 (S=survivor).  
* **Old generation**

Objects are initially created in Eden, after "minor" garbage collection, live objects go to one of survivor spaces (S0).
Next GC moves all live objects from Eden and S0 to S1, so  that Eden and S0 is free.
After several GCs (8 mostly, but depends) objects are moved in Old generation during "major" gc.

## Java GC types

1. **Serial Garbage Collector:** Serial garbage collector works by holding all the application threads. It is designed for the single-threaded environments. 
It uses just a single thread for garbage collection. The way it works by freezing all the application threads while doing garbage collection may not be suitable for a server environment. 
It is best suited for simple command-line programs. Turn on the -XX:+UseSerialGC JVM argument to use the serial garbage collector.

2. **Parallel Garbage Collector:** Parallel garbage collector is also called as throughput collector. 
It is the default garbage collector of the JVM. Unlike serial garbage collector, this uses multiple threads for garbage collection. 
Similar to serial garbage collector this also freezes all the application threads while performing garbage collection. 
Turn on the -XX:+UseParallelGC JVM argument to use the parallel garbage collector.

3. **CMS Garbage Collector:** Concurrent Mark Sweep (CMS) garbage collector uses multiple threads to scan the heap memory to mark instances for eviction and then sweep the marked instances. 
CMS garbage collector holds all the application threads in the following two scenarios only:
    * While marking the referenced objects in the tenured (old) generation space.
    * If there is a change in heap memory in parallel while doing the garbage collection.

    In comparison with parallel garbage collector, CMS collector uses more CPU to ensure better application throughput. 
If we can allocate more CPU for better performance then CMS garbage collector is the preferred choice over the parallel collector. 
Turn on the XX:+USeParNewGC JVM argument to use the CMS garbage collector.

4. **G1 Garbage Collector:** G1 garbage collector is used for large heap memory areas. It separates the heap memory into regions and does collection within them in parallel. 
G1 also does compacts the free heap space on the go just after reclaiming the memory. But CMS garbage collector compacts the memory on stop the world (STW) situations. 
G1 collector prioritizes the region based on most garbage first. Turn on the –XX:+UseG1GC JVM argument to use the G1 garbage collector.

## Java Class Loader types 

Types of Built-in Class Loaders:
* application;
* extension;
* bootstrap; 

The **application class loader** loads the class where the example method is contained. An application or system class loader loads our own files in the classpath.
Next, the **extension class loader** loads the Logging class. **Extension class loaders** load classes that are an extension of the standard core Java classes.
Finally, the **bootstrap class loader** loads the ArrayList class. A **bootstrap or primordial class loader** is the parent of all the others.
However, we can see that the last out, for the ArrayList it displays null in the output. This is because the bootstrap class loader is written in native code, not Java – so it doesn’t show up as a Java class. 
Due to this reason, the behavior of the bootstrap class loader will differ across JVMs.

When the JVM requests a class, the class loader tries to locate the class and load the class definition into the runtime using the fully qualified class name.
The java.lang.ClassLoader.loadClass() method is responsible for loading the class definition into runtime. It tries to load the class based on a fully qualified name.
If the class isn’t already loaded, it delegates the request to the parent class loader. This process happens recursively.
Eventually, if the parent class loader doesn’t find the class, then the child class will call java.net.URLClassLoader.findClass() method to look for classes in the file system itself.
If the last child class loader isn’t able to load the class either, it throws java.lang.NoClassDefFoundError or java.lang.ClassNotFoundException.

    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
 
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
	
	
JIT compiler works on JVM to transform bytecode to native code for performance optimization.

## Java Object reference types

Before deleting object GC puts it in the ReferenceQueue and then decides whether to delete it.

**Soft Reference** used for caching.
There's a special formula of how long will the reference live, but it's platform dependant.
Вот как Hotspot принимает решение об удалении SoftReference: если посмотреть на реализацию SoftReference, то видно, что в классе есть 2 переменные — private static long clock и private long timestamp. Каждый раз при запуске GC, он сетит текущее время в переменную clock. Каждый раз при создании SoftReference, в timestamp записывается текущее значение clock. timestamp обновляется каждый раз при вызове метода get() (каждый раз, когда мы создаем strong-ссылку на объект). Это позволяет вычислить, сколько времени существует soft-ссылка после последнего обращения к ней. Обозначим этот интервал буквой I. Буквой F обозначим количество свободного места в куче в MB(мегабайтах). Константой MSPerMB обозначим количество миллисекунд, сколько будет существовать soft-ссылка для каждого свободного мегабайта в куче. 
Дальше все просто, если I <= F * MSPerMB, то не удаляем объект. Если больше то удаляем. 
Для изменения MSPerMB используем ключ -XX:SoftRefLRUPolicyMSPerMB. Дефалтовое значение — 1000 ms, а это означает что soft-ссылка будет существовать (после того как strong-ссылка была удалена) 1 секунду за каждый мегабайт свободной памяти в куче. Главное не забыть что это все примерные расчеты, так как фактически soft-ссылка удалится только после запуска GC.
Обратите внимание на то, что для удаления объекта, I должно быть строго больше чем F * MSPerMB. Из этого следует что созданная SoftReference проживет минимум 1 запуск GC. (*если не понятно почему, то это останется вам домашним заданием).
В случае VM от IBM, привязка срока жизни soft-ссылки идет не к времени, а к количеству переживших запусков GC.

Example: big image stored in memory (not to load it from the disk every time).
It will be garbage collected only if it's really needed and memory is not enough.
java.lang.Class тоже использует SoftReference для кэширования. Он кэширует данные о конструкторах, методах и полях класса.
 
**WeakReference**
Object referenced only by week ref will be garbage collected when there's no strong ref for it.
Not suitable for cache, but good for intermediate calculations.
WeakHashMap. Это реализация Map<K,V> которая хранит ключ, используя weak-ссылку. И когда GC удаляет ключ с памяти, то удаляется вся запись с Map.
WeakHashMap не предназначена для использования в качестве кэша. WeakReference создается для ключа а не для значения. И данные будут удалены только после того как в программе не останется strong-ссылок на ключ а не на значение. В большинстве случаев это не то чего вы хотите достичь кэшированием.
Данные с WeakHashMap будут удалены не сразу после того как GC обнаружит что ключ доступен только через weak-ссылки. Фактически очистка произойдет при следующем обращении к WeakHashMap.
В первую очередь WeakHashMap предназначен для использования с ключами, у которых метод equals проверяет идентичность объектов (использует оператор ==). Как только доступ к ключу потерян, его уже нельзя создать заново.

Допустим нам нужно создать XML документ для пользователя. Конструированием документа будут заниматься несколько сервисов, которые на вход будут получать org.w3c.Node в который будут добавлять необходимые элементы. Так же для сервисов нужно много информации о пользователе с Базы Данных. Эти данные мы будем складировать в классе UserInfo. Класс UserInfo занимает много места в памяти и актуален только для построения конкретного XML документа. Кешировать UserInfo не имеет смысла. Нам нужно только ассоциировать его с документом и желательно удалить из памяти, когда документ более не используется программой. Все что нам нужно сделать:

    private static final NODE_TO_USER_MAP = new WeakHashMap<Node, UserInfo>();

Создание XML документа будет выглядеть примерно так:
    
    Node mainDocument = createBaseNode();
    NODE_TO_USER_MAP.put(mainDocument, loadUserInfo());

Ну а вот чтение:

    UserInfo userInfo = NODE_TO_USER_MAP.get(mainDocument);
    If(userInfo != null) {
	    // …
    }

UserInfo будет находиться в WeakHashMap до тех пор пока GC не заметит, что на mainDocument остались только weak-ссылки. 

**PhantomReference**

Особенности GC
Особенностей у этого типа ссылок две. Первая это то, что метод get() всегда возвращает null. Именно из-за этого PhantomReference имеет смысл использовать только вместе с ReferenceQueue. 
Вторая особенность – в отличие от SoftReference и WeakReference, GC добавит phantom-ссылку в ReferenceQueue послетого как выполниться метод finalize(). 
Тоесть фактически, в отличии от SoftReference и WeakReference, объект еще есть в памяти.

Вернемся к PhantomReference. Этот тип ссылок в комбинации с ReferenceQueue позволяет нам узнать, когда объект более недоступен и на него нет других ссылок. 
Это позволяет нам сделать очистку ресурсов, используемых объектом, на уровне приложения. В отличии от finalize() мы сами контролируем процесс очистки ресурсов. 
Помимо этого, мы можем контролировать процесс создания новых объектов. Допустим у нас есть фабрика, которая будет возвращать нам объект HdImage. 
Мы можем контролировать, сколько таких объектов будет загружено в память:

    public HdImageFabric {
	    public static final int IMAGE_LIMIT = 10;
	    public static int count = 0;
	    public static ReferenceQueue<HdImage> queue = new ReferenceQueue<HdImage>();

	    public HdImage loadHdImage(String imageName) {
		    while (true) {
			    if (count < IMAGE_LIMIT) {
				    return	wrapImage(loadImage(imageName));	
			    } else {
				    Reference<HdImage> ref = queue.remove(500);
				    if (ref != null) {
					    count--;
					    System.out.println(“remove old image”);
				    }
		    	}
		    }
	    }

	    private HdImage wrapImage(HdImage image) {
		    PhantomReference<HdImage> refImage = new PhantomReference(image, queue);
		    count++;
		    return refImage ;
	    }
    }
