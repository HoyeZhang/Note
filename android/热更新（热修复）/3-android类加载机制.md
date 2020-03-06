# 虚拟机与加载器
- 我们知道Android系统也是仿照java搞了一个虚拟机，不过它不叫JVM，它叫Dalvik/ART VM。

- Dalvik/ART VM 虚拟机加载类和资源也是要用到ClassLoader，不过Jvm通过ClassLoader加载的class字节码，而Dalvik/ART VM通过ClassLoader加载则是dex。

- Android的类加载器分为两种,PathClassLoader和DexClassLoader，两者都继承自BaseDexClassLoader

- PathClassLoader ： 用来加载系统类和应用类
- DexClassLoader ： 用来加载jar、apk、dex文件.加载jar、apk也是最终抽取里面的Dex文件进行加载.
# Dalvik虚拟机类加载器源码分析
- Android的类加载器主要有两个PathClassLoader和DexClassLoader，其中PathClassLoader是默认的类加载器，下面我们就来说说两者的区别与联系。

- PathClassLoader：支持加载DEX或者已经安装的APK（因为存在缓存的DEX）。
- DexClassLoader：支持加载APK、DEX和JAR，也可以从SD卡进行加载。
- DexClassLoader和PathClassLoader都属于符合双亲委派模型的类加载器（因为它们没有重载loadClass方法）。也就是说，它们在加载一个类之前，回去检查自己以及自己以上的类加载器是否已经加载了这个类。如果已经加载过了，就会直接将之返回，而不会重复加载。

- PathClassLoader还是DexClassLoader继承于BaseDexClassLoader，BaseDexClassLoader继承鱼ClassLoader，下面我们就以一个类的加载流程来分析各个加载器的源码实现。

- 要加载一个类，必须先初始化一个类加载器实例，我们拿DexClassLoader来举例，它的构造方法如下所示
<pre>
 public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
</pre>
- 该函数中的参数含义如下所示:

	- String dexPath：加载APK、DEX和JAR的路径。这个类可以用于Android动态加载DEX/JAR。
	- String optimizedDirectory：是DEX的输出路径。
	- String libraryPath：加载DEX的时候需要用到的lib库，libraryPath一般包括/vendor/lib和/system/lib。
	- ClassLoader parent：DEXClassLoader指定的父类加载器
- 关于DexClassLoader，除了它的构造函数以外，它的源码注释里还提到以下三点：

- 这个类加载器加载的文件是.jar或者.apk文件，并且这个.jar或.apk中是包含classes.dex这个入口文件的，主要是用来执行那些没有被安装的一些可执行文件的。
- 这个类加载器需要一个属于应用的私有的，可以的目录作为它自己的缓存优化目录，其实这个目录也就作为下面，这个构造函数的第二个参数，至于怎么实现，注释中也已经给出了答案；
- 不要把上面第二点中提到的这个缓存目录设为外部存储，因为外部存储容易收到代码注入的攻击。
- 通过DexClassLoader的构造函数，我们可以发现DexClassLoader的构造函数会调用父类的构造函数进行初始化，DexClassLoader的父类就是- - BaseDexXClassLoader，我们继续来看一下BaseDexClassLoader的构造函数：

<pre>
   public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }
</pre>
- 我们可以发现在执行BaseDexClassLoader的构造函数时，会先调用父类ClassLoader的构造方法：
<pre>
 /**
     * Constructs a new instance of this class with the system class loader as
     * its parent.
     */
    protected ClassLoader() {
        this(getSystemClassLoader(), false);
    }

    /**
     * Constructs a new instance of this class with the specified class loader
     * as its parent.
     *
     * @param parentLoader
     *            The {@code ClassLoader} to use as the new class loader's
     *            parent.
     */
    protected ClassLoader(ClassLoader parentLoader) {
        this(parentLoader, false);
    }

    /*
     * constructor for the BootClassLoader which needs parent to be null.
     */
    ClassLoader(ClassLoader parentLoader, boolean nullAllowed) {
        if (parentLoader == null && !nullAllowed) {
            throw new NullPointerException("parentLoader == null && !nullAllowed");
        }
        parent = parentLoader;
    }
</pre>
- 通过ClassLoader的构造函数源码可以发现，BaseDexClassLoader里的parentLoader对象经过层层传递，传递给了parent对象，parent对象是ClassLoader类里的私有变量，如下所示：

<pre>
 /**
     * The parent ClassLoader.
     */
    private ClassLoader parent;
</pre>
- 这一步做完以后，BaseDexClassLoader的构造函数紧接着就初始化了一个DexPathList对象，这是一个描述DEX文相关资源文件的条目列表。
接下来看下DexPathList的构造函数
<pre>
  public DexPathList(ClassLoader definingContext, String dexPath,
            String libraryPath, File optimizedDirectory) {
        if (definingContext == null) {
            throw new NullPointerException("definingContext == null");
        }

        if (dexPath == null) {
            throw new NullPointerException("dexPath == null");
        }

        if (optimizedDirectory != null) {
            if (!optimizedDirectory.exists())  {
                throw new IllegalArgumentException(
                        "optimizedDirectory doesn't exist: "
                        + optimizedDirectory);
            }

            if (!(optimizedDirectory.canRead()
                            && optimizedDirectory.canWrite())) {
                throw new IllegalArgumentException(
                        "optimizedDirectory not readable/writable: "
                        + optimizedDirectory);
            }
        }

        this.definingContext = definingContext;
        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions);
        if (suppressedExceptions.size() > 0) {
            this.dexElementsSuppressedExceptions =
                suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
        } else {
            dexElementsSuppressedExceptions = null;
        }
        this.nativeLibraryDirectories = splitLibraryPath(libraryPath);
    }
</pre>
- 可以看到DexPathList里比较重要dexElements也是在这里被初始化，热修复就是倒腾这个东东做到修复bug的。
<pre>
 this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory, suppressedExceptions);
</pre>