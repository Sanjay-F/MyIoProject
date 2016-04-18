
title: 源码探索系列27---插件化基础之类加载器DexClassLoader
date: 2016-03-16 20:50:46
tags: [android,源码,DexClassLoader]
categories: android

------------------------------------------

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/slide_4.jpg)

Java有个概念叫做`类加载器`（ClassLoader），它的作用就是`动态的装载Class文件`。借助`ClassLoader`这个类我们可以装载任何我们想要的Class文件，只需提供路径即可。

像下面这样：

	Class.forName("OurClassName");

啊！这是反射，嗯！应该是下面这样

 	ClassLoader.loadClass("OurClassName")
		
 

等等！
这Java不是可以import就可以了吗，为何还要这个类加载器？
因为用import的话得类文件必须在本地且编译的时候必须有这个类文件，否则报错。若想让程序在运行的时候**动态调用**怎么办呢？显然import不合要求，他已经写死的了，所以我们需要ClassLoader来做这件事。
利用它来加载Class文件到JVM，以供程序使用。
这也是我们能够通过加载补丁包来修复紧急的bug的一个基础

反射和这个类加载有什么关系嘛？后面再讲
<!--more-->

# 起航
 
## ClassLoader本身是如何被加载呢？ 
   
这是一个有趣的先有鸡，还是先有蛋的问题。

BootStrapClassloader(启动类加载器)是JVM实现的一部分(是C++写的二进制代码，非JAVA)，嵌套在Java虚拟机内核里面，在JVM运行的时候会加载Java核心的API以满足Java程序最基本的需求，包括用户定义的ClassLoader，所谓的用户定义是指通过Java程序实现的ClassLoader：
一个是**ExtClassLoader**，这个ClassLoader是用来加载java的扩展API的，也就是/lib/ext中的类，
一个是**AppClassLoader**，这个ClassLoader是用来加载用户机器上CLASSPATH设置目录中的Class的，通常在没有指定ClassLoader的情况下，程序员自定义的类就由该ClassLoader进行加载。 
											 
 ![类加载器默认委派关系图](http://img.my.csdn.net/uploads/201301/03/1357199754_4251.png)

对于Android虽然使用标准的Java编译器编译出Class文件，不过和普通的Java开发不同的是它把Class文件再重新打包成dex类型的文件，这种重新打包会对Class文件内部的各种函数表、变量表等进行**优化**，最终产生了dex文件。dex文件是一种经过android打包工具优化后的Class文件，因此加载这样特殊的Class文件就需要特殊的类装载器，所以android中提供了`DexClassLoader`类。

## DexClassLoader

SDK:23

安卓的动态加载基础是这个dexClassLoader，他的一般调用形式如下

	DexClassLoader calssLoader = new DexClassLoader(dexPath, optimizedDirectory, libraryPath,  classLoader);  
	
	Class<?> clazz = calssLoader.loadClass(pacageName+".Plugin1");  

 那么我们就以下面的接口为切入点来看下背后的实现 

	public class DexClassLoader extends BaseDexClassLoader {
 
	    public DexClassLoader(String dexPath, String optimizedDirectory,
	            String libraryPath, ClassLoader parent) {
	        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
	    }
	}
	
他背后是BaseDexClassLoader，我们来看下

	 public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);//  <-重要的一句
        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
        //这个新new出来的DexPathList，简单的说是负责管理我们想要加载的包的
    }
     
 
	public abstract class ClassLoader {
  
		protected ClassLoader(ClassLoader parentLoader) {
	        this(parentLoader, false);
	    }
	    
		ClassLoader(ClassLoader parentLoader, boolean nullAllowed) {
	        if (parentLoader == null && !nullAllowed) {
	            throw new NullPointerException("parentLoader == null && !nullAllowed");
	        }
	        parent = parentLoader;
	    }
	}

我们看到在构造函数，他传递了一个叫parent的ClassLoader过去。这里涉及到一个概念叫双亲委托，简单说就是我们在调用`calssLoader.loadClass()` 去加载的时候，会先调用这个parent去加载下，没有才轮到自己去加载。
详细的内容会在后面解释，我们继续主线。

我们去看下那个DexPathList

	public DexPathList(ClassLoader definingContext, String dexPath,
            String libraryPath, File optimizedDirectory) {

		...
        this.definingContext = definingContext;
        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        // save dexPath for BaseDexClassLoader
        this.dexElements = makePathElements(splitDexPath(dexPath), optimizedDirectory,
                                            suppressedExceptions);
        //这个函数去加载dex文件，后面我们loadClass就是在从dexElements里面找我们的class，而他的类型Element是静态内部类

        // Native libraries may exist in both the system and
        //application library paths,and we use this search order:
        //
        //   1. This class loader's library path for application libraries 
		        (libraryPath):
        //   1.1. Native library directories
        //   1.2. Path to libraries in apk-files
        //   2. The VM's library path from the system property for system libraries
        //      also known as java.library.path
        //
        // This order was reversed prior to Gingerbread; see http://b/2933456.
        this.nativeLibraryDirectories = splitPaths(libraryPath, false);
        this.systemNativeLibraryDirectories =
                splitPaths(System.getProperty("java.library.path"), true);
        List<File> allNativeLibraryDirectories = new ArrayList<>(nativeLibraryDirectories);
        allNativeLibraryDirectories.addAll(systemNativeLibraryDirectories);

        this.nativeLibraryPathElements = makePathElements(allNativeLibraryDirectories, 
        null,suppressedExceptions);
 

        if (suppressedExceptions.size() > 0) {
            this.dexElementsSuppressedExceptions =
                suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
        } else {
            dexElementsSuppressedExceptions = null;
        }
    }
	
我们稍微的去看下那个makePathElements
 
    private static Element[] makePathElements(List<File> files, File optimizedDirectory,
                                              List<IOException> suppressedExceptions) {
        List<Element> elements = new ArrayList<>();
        /*
         * Open all files and load the (direct or contained) dex files
         * up front.
         */
        for (File file : files) {
            File zip = null;
            File dir = new File("");
            DexFile dex = null;
            String path = file.getPath();
            String name = file.getName();
			//这判断看起来像支持ZIP，DEX，
            if (path.contains(zipSeparator)) {
                String split[] = path.split(zipSeparator, 2);
                zip = new File(split[0]);
                dir = new File(split[1]);
            } else if (file.isDirectory()) {
                // We support directories for looking up resources and native libraries.
                // Looking up resources in directories is useful for running libcore tests.
                elements.add(new Element(file, true, null, null));
            } else if (file.isFile()) {
                if (name.endsWith(DEX_SUFFIX)) {
                    // Raw dex file (not inside a zip/jar).
                    try {
                        dex = loadDexFile(file, optimizedDirectory);
                    } catch (IOException ex) {
                        System.logE("Unable to load dex file: " + file, ex);
                    }
                } else {
                    zip = file;

                    try {
                        dex = loadDexFile(file, optimizedDirectory);
                    } catch (IOException suppressed) {
                       ...
                        suppressedExceptions.add(suppressed);
                    }
                }
            } else {
                System.logW("ClassLoader referenced unknown path: " + file);
            }

            if ((zip != null) || (dex != null)) {
                elements.add(new Element(dir, false, zip, dex));
            }
        }

        return elements.toArray(new Element[elements.size()]);
    }
 
    private static DexFile loadDexFile(File file, File optimizedDirectory)
            throws IOException {
        if (optimizedDirectory == null) {
            return new DexFile(file);
        } else {
	        // 我们的非空，所以看下面这个
            String optimizedPath = optimizedPathFor(file, optimizedDirectory);
            return DexFile.loadDex(file.getPath(), optimizedPath, 0);
        }
    }
	
我们来看下这个optimizedPathFor()的内容

	 /**
     * Converts a dex/jar file path and an output directory to an
     * output file path for an associated optimized dex file.
     */
    private static String optimizedPathFor(File path,
            File optimizedDirectory) {
        /*
         * Get the filename component of the path, and replace the
         * suffix with ".dex" if that's not already the suffix.
         *
         * We don't want to use ".odex", because the build system uses
         * that for files that are paired with resource-only jar
         * files. If the VM can assume that there's no classes.dex in
         * the matching jar, it doesn't need to open the jar to check
         * for updated dependencies, providing a slight performance
         * boost at startup. The use of ".dex" here matches the use on
         * files in /data/dalvik-cache.
         */
        String fileName = path.getName();
        if (!fileName.endsWith(DEX_SUFFIX)) {
            int lastDot = fileName.lastIndexOf(".");
            if (lastDot < 0) {
                fileName += DEX_SUFFIX;
            } else {
                StringBuilder sb = new StringBuilder(lastDot + 4);
                sb.append(fileName, 0, lastDot);
                sb.append(DEX_SUFFIX);
                fileName = sb.toString();
            }
        }

        File result = new File(optimizedDirectory, fileName);
        return result.getPath();
    }

把odex后缀改成了dex.第一次听说odex，查了下，发现是ROM上的优化，把apk中的dex文件抽出来，有加快软件加载速度和开机速度的功效！同时某种程序来防止盗程序。嗯！感觉又有一个优化性能的馊主意出现了。

我们继续回主线，看下那个loadDex（）背后的内容

	static public DexFile loadDex(String sourcePathName, String outputPathName,
        int flags) throws IOException {

	        /*
	         * TODO: we may want to cache previously-opened DexFile objects.
	         * The cache would be synchronized with close().  This would help
	         * us avoid mapping the same DEX more than once when an app
	         * decided to open it multiple times.  In practice this may not
	         * be a real issue.
	         */
	        return new DexFile(sourcePathName, outputPathName, flags);
	    }
	    
看到这里略微有点被骗的感觉，这静态方法背后就是去new一个，还load.....
这哥们注释写着想缓存Dex，不过还没做.


	private DexFile(String sourceName, String outputName, int flags) 
	throws IOException {
	
	        if (outputName != null) {
	            try {
	                String parent = new File(outputName).getParent();
	                if (Libcore.os.getuid() != Libcore.os.stat(parent).st_uid) {
	                    //对比这个UID的值，不如不给执行，避免注入攻击！
	                    //我们在共享数据时候有用到UID，在manifest里面
		                //android:sharedUserId="yourId"
	                    throw new IllegalArgumentException("Optimized data directory " + 
	                    parent+ " is not owned by the current user. Shared storage 
                    cannot protect"+"your application from code injection attacks.");
                    
	                }
	            } catch (ErrnoException ignored) {
	                // assume we'll fail with a more contextual error later
	                //哈哈，连个打印都不给
	            }
	        }
	
	        mCookie = openDexFile(sourceName, outputName, flags);
	        mFileName = sourceName;
	        guard.open("close");
	        //System.out.println("DEX FILE cookie is " + mCookie);
	 }

    /*
     * Open a DEX file.  The value returned is a magic VM cookie.  
     *  On failure, an IOException is thrown.
     */
     private static int openDexFile(String sourceName, String outputName,
        int flags) throws IOException {
        
        return openDexFileNative(new File(sourceName).getCanonicalPath(),
                                 (outputName == null) ?null :
                                  new File(outputName).getCanonicalPath(),flags);
    }
    
	native private static int openDexFileNative(String sourceName, String outputName,
       int flags) throws IOException;
       //背后是一个native，我们就不深究了，理论上只是加载文件，到这吧-.-

看到这里我们看完一半的加载过程，另外在native代码的加载就跳过。
回主线的加载类流程可好。接下来我们看下`calssLoader.loadClass（）`的内容。

这个方法只在ClassLoader有，我们去看下

	 @Override
    protected Class<?> loadClass(String className, boolean resolve)
           throws ClassNotFoundException {
        Class<?> clazz = findLoadedClass(className);
        //查缓存
        if (clazz == null) {
            clazz = findClass(className);
        }

        return clazz;
    }
    
	 /**
     * Overridden by subclasses, throws a {@code ClassNotFoundException} by
     * default. This method is called by {@code loadClass} after the parent
     * {@code ClassLoader} has failed to find a loaded class of the same name.
     */
	  protected Class<?> findClass(String className) throws ClassNotFoundException {
        throw new ClassNotFoundException(className);
        
        //我们的子类必须实现他！所以我们就去看下我们的BaseDexClassLoder
        //不要问我为何他不设置抽象,尽管这个类就是抽象类!
    }

我们去看下他的subClasses之BaseDexClass

	@Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        //我们的pathList又出场啦，在构造函数哪里提到过，别忘了哟
        ...
        return c;
    }
我们去DexPathList类看下

	public Class findClass(String name, List<Throwable> suppressed) {
	
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;
            
            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }

前面在它构造函数有这么句：

        this.dexElements = makePathElements(splitDexPath(dexPath), optimizedDirectory,
                                            suppressedExceptions);
        //这个函数去加载dex文件，后面我们loadClass就是在从dexElements里面找我们的class，而他的类型Element是静态内部类

我在注释就说了我们的loadClass就是在dexElements里面找我们的class，没骗你吧，因为我看了才写这文章的嘛！哈哈哈哈

我们去DexFile里面看下他的内容

	public Class loadClassBinaryName(String name, ClassLoader loader, List<Throwable> suppressed) {
        return defineClass(name, loader, mCookie, suppressed);
    }

    private static Class defineClass(String name, ClassLoader loader, int cookie,
                                     List<Throwable> suppressed) {
        Class result = null;         
        
        result = defineClassNative(name, loader, cookie);
       ...
        return result;
    }
    
    private static native Class defineClassNative(String name, ClassLoader loader, int cookie) throws ClassNotFoundException, NoClassDefFoundError;


我们看到参数里面有一个有趣的变量名叫mCookie，嗯，很可能这个人以前是写网站的啊，才以这个做标识符。
到这里，我们继续不去看native的代码，知道流程了。今晚回家再继续看下。哈哈,去吃饭了


# 双亲委托

每一个自定义XXXClassLoader都必须继承ClassLoader这个抽象类

	public abstract class ClassLoader {
 	       
	    private ClassLoader parent; //<- 重要的一个变量
	    
		protected ClassLoader() {
	        this(getSystemClassLoader(), false);
	    }
	     
	    protected ClassLoader(ClassLoader parentLoader) {
	        this(parentLoader, false);
	    }
	         
	    ClassLoader(ClassLoader parentLoader, boolean nullAllowed) {
	        if (parentLoader == null && !nullAllowed) {
	            throw new NullPointerException("parentLoader == null && !nullAllowed");
	        }
	        parent = parentLoader;
	    }	    
	    ....
    } 
    
每个ClassLoader都会有一个叫`parent`的ClassLoader变量，这个parent不是**被继承的类**，而是在实例化该ClassLoader时指定的一个ClassLoader，如果这个parent为null，那么就默认是`BootClassLoader`（上面的getSystemClassLoader()返回的就是这个）。

**那么，这个parent有什么用呢？** 

假设我们弄了一个叫`MyClassLoader`的，现在想加载`java.lang.String`这个类，那么背后实际是靠他的parent ClassLoader去加载的。让我们来一段代码 

	protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
	
	    Class<?> clazz = findLoadedClass(className);	 
	    if (clazz == null) {
	        try {
	            clazz = parent.loadClass(className, false);
	            //找parent帮忙加
	        } catch (ClassNotFoundException e) {
				 ...
	        }
	        	 
	        if (clazz == null) {
	            clazz = findClass(className);
	            //加不出来才靠自己加
	        }
	    }
	    ...	 
	    return clazz;
	}

	protected final Class<?> findLoadedClass(String className) {
        ClassLoader loader;
        if (this == BootClassLoader.getInstance())
            loader = null;
        else
            loader = this;
        return VMClassLoader.findLoadedClass(loader, className);
    }
    
    native static Class findLoadedClass(ClassLoader cl, String name);
    
因为在任何一个自定义ClassLoader加载一个类之前，都会先委托它的**父亲**`ClassLoader`进行加载，只有当父亲ClassLoader无法加载成功后，才会由自己加载。 例如我们的`java.lang.String`是属于Java核心API的一个类，所以当使用MyClassLoader加载它的时候，该ClassLoader会先委托它的父亲ClassLoader进行加载，上面讲过，当ClassLoader的parent为null时，ClassLoader的parent就是`BootStrapClassloader`，所以在ClassLoader的**最顶层**就是BootStrapClassloader，因此最终委托到bootstrap classloader的时候，bootstrap classloader就会返回String的Class。
每个被ClassLoader加载的class文件，最终都会以Class类的实例被程序员引用，我们可以把Class类当作是普通类的一个模板，JVM根据这个模板生成对应的实例，最终被程序员所使用。  
 
简单的说：    

> LoadClass 会先看这类是否被加载过，没有则去他的**parent**去找，如此递归，因此称之为双亲委托。


## 为什么要使用这种双亲委托模式呢？ 

1. 避免重复加载
当父亲已经加载了该类的时候，就没有必要让子ClassLoader再加一次。 
2. 安全因素的考虑
 我们试想一下，如果不使用这种**委托模式**，那我们就可以随时使用自定义的**String**来**动态替代**Java核心API中定义DE 类型，这样会存在非常大的安全隐患，而双亲委托的方式可以避免这种情况，因为String已经在启动时被加载，所以用户自定义类是无法加载一个自定义的ClassLoader。 

# ClassLoader和 反射Class.forName()


Class类中有个静态方法forName，这个方法和ClassLoader中的loadClass方法的目的一样，都是用来加载class，但是两者在作用上却有所区别。 

	Class<?> loadClass(String name) 
	Class<?> loadClass(String name, boolean resolve) 
	
我们看到上面两个方法声明，第二个方法的第二个参数是用于设置加载类的时候是否**连接该类**，true就连接，否则就不连接。 我们知道在JVM加载类时候，需经过三个步骤，**装载、连接、初始化**。

根据上面loadClass方法的第二个参数来判定是否需要解释，所谓的解释根据就是根据类中的符号引用查找相应的**实体**，再把符号引用替换成一个**直接引用**的过程。（有点绕口）  

我们通过loadClass加载类实际上就是加载的时候并不对该类进行解释，一般调用的反射默认是会的，虽然我们也可以调用另外一个函数设置为不初始化(即调用类的静态块的语句及初始化静态成员变量。)，这也是为何我们需要主动的去调用`class.NewInstance()`的原因。
	
	public static Class<?> forName(String className) throws ClassNotFoundException {
        return forName(className, true, VMStack.getCallingClassLoader());
    }
    public static Class<?> forName(String className, boolean shouldInitialize,
            ClassLoader classLoader)

 我们来看下那个BootClassLoader的代码，它最终的找类代码就是用反射的false配置。
	
	class BootClassLoader extends ClassLoader { 	
	    ...
	
	    @Override
	    protected Class<?> findClass(String name) throws ClassNotFoundException {
	        return Class.classForName(name, false, null);// <-默认为false
	    }
	    
	    ....
	}
	
#  ClassLoader 隔离问题

大家觉得一个运行程序中有没有可能同时存在两个包名和类名完全一致的类？
这是有可能做到的，因为JVM和Dalvik对类唯一的识别的判断依据是
`ClassLoaderID` +`PackageName`+ `ClassName`，所以一个运行程序中是有可能存在两个包名和类名完全一致的类的。并且如果这两个”类”不是由一个 ClassLoader加载，是无法将一个类的示例强转为另外一个类的，这就是 ClassLoader 隔离。

	 所以像能不能自己写个类叫java.lang.System？是可以做到的
	 
 System类是Bootstrap加载器加载的，就算自己重写，也总是使用Java系统提供的System，自己写的System类根本没有机会得到加载。

但是，我们可以自己定义一个类加载器来达到这个目的，为了避免双亲委托机制，这个类加载器也必须是特殊的。由于系统自带的三个类加载器都加载特定目录下的类，如果我们自己的类加载器放在一个特殊的目录，那么系统的加载器就无法加载，也就是最终还是由我们自己的加载器加载。

# 后记

前面看那些插件化/热补丁的时候有些不清楚的地方，所以对涉及到的知识点在这里做些笔记，希望帮助自己更好的掌握相关的内容，改好前面几篇文章，再打算写下Binder的native层的内容和对他们的更深入解析文章


# 参考资料：
[Java ClassLoader基础及加载不同依赖 Jar 中的公共类](http://www.trinea.cn/android/java-loader-common-class/)，Trinea的一篇文章

[安卓高手之路之ClassLoader](http://daojin.iteye.com/blog/1847093)

 [深入JVM系列（三）之类加载、类加载器、双亲委派机制与常见问题](http://blog.csdn.net/vernonzheng/article/details/8461380)
 
[Android动态加载基础 ClassLoader工作机制](https://segmentfault.com/a/1190000004062880)

[解读ClassLoader](http://www.iteye.com/topic/83978)
 
[在线的系统源码查看网站](http://androidxref.com/6.0.1_r10)