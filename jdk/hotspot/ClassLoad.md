1

## 类加载

#### Klass模型

* klass
  * InstanceKlass      普通类的元信息的JVM中的存在形式
    * instanceMirrorKlass       class对象(反射用)
    * instanceRefKlass       用于表示java/lang/ref/Reference类的子类
    * instanceClassLoaderKlass   用于遍历某个加载器加载的类
  * Arrayklass
    * TypeArrayKlass      存放基本类型数组的元信息的类
    * ObjArrayKlass         存放对象数组的元信息的类

#### 类加载的时机

* new,getstatic,pustatic,invokestatic
* 反射
* 初始化一个类的子类
* 启动类
* 当使用jdk1.7动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getstatic,REF_putstatic,REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行初始化，则需要先出触发其初始化

#### 类加载的过程

* 加载

  * 通过类的全限定名获取该类的class文件
  * 解析成运行时数据,instanceKlass实例,存放在方法区
  * 在堆区中生成该类的Classs对象,即instanceMirrorklass实例

* 验证

  * 文件格式
  * 元数据
  * 字节码验证
  * 符号引用验证

* 准备

  * 为静态变量分配内存并且赋予初值
  * 如果被final修饰,编译的时候会给属性添加ConstantValue属性,准备阶段直接完成赋值.

* 解析

  将常量池中的符号引用转为直接引用,解析后的信息存储在ConstantPoolCache类的实例中

  * 类或接口的解析
  * 字段解析
  * 方法解析
  * 接口方法解析

  解析的时机?

  思路：

  1、加载阶段解析常量池时

  2、用的时候

  

  openjdk是第二种思路，在执行特定的字节码指令之前进行解析：

  anewarray、checkcast、getfield、getstatic、instanceof、invokedynamic、invokeinterface、invokespecial、invokestatic、invokevirtual、ldc、ldc_w、ldc2_w、multianewarray、new、putfield

* 初始化

  执行静态代码块,完成静态变量的赋值.

  静态字段,静态代码段,字节吗层换会生成clinit方法,执行顺序与代码先后顺序有关.

## 类加载器

### 分类

* BootStrap ClassLoader       :  jre/lib/rt.jar
* Platform ClassLoader          : jre/lib/ext/*.jar
* Application ClassLoader      :CLASS_PATH 指定目录的所有jar
* User ClassLoader                  : 自定义的类加载器

### 类加载器启动流程

```java
//"src/hotspot/share/prims/jvm.cpp"
JVM_ENTRY(jclass, JVM_FindClassFromBootLoader(JNIEnv* env,
                                              const char* name))
  JVMWrapper("JVM_FindClassFromBootLoader");

  // Java libraries should ensure that name is never null...
  if (name == NULL || (int)strlen(name) > Symbol::max_length()) {
    // It's impossible to create this class;  the name cannot fit
    // into the constant pool.
    return NULL;
  }

  TempNewSymbol h_name = SymbolTable::new_symbol(name, CHECK_NULL);
  Klass* k = SystemDictionary::resolve_or_null(h_name, CHECK_NULL);
  if (k == NULL) {
    return NULL;
  }

  if (log_is_enabled(Debug, class, resolve)) {
    trace_class_resolution(k);
  }
  return (jclass) JNIHandles::make_local(env, k->java_mirror());
JVM_END


    private static final BootClassLoader BOOT_LOADER;
    private static final PlatformClassLoader PLATFORM_LOADER;
    private static final AppClassLoader APP_LOADER;
	static {
        // -Xbootclasspath/a or -javaagent with Boot-Class-Path attribute
        String append = VM.getSavedProperty("jdk.boot.class.path.append");
        BOOT_LOADER =
            new BootClassLoader((append != null && !append.isEmpty())
                ? new URLClassPath(append, true)
                : null);
        PLATFORM_LOADER = new PlatformClassLoader(BOOT_LOADER);

        // A class path is required when no initial module is specified.
        // In this case the class path defaults to "", meaning the current
        // working directory.  When an initial module is specified, on the
        // contrary, we drop this historic interpretation of the empty
        // string and instead treat it as unspecified.
        String cp = System.getProperty("java.class.path");
        if (cp == null || cp.isEmpty()) {
            String initialModuleName = System.getProperty("jdk.module.main");
            cp = (initialModuleName == null) ? "" : null;
        }
        URLClassPath ucp = new URLClassPath(cp, false);
        APP_LOADER = new AppClassLoader(PLATFORM_LOADER, ucp);
    }
```



### 加载main函数类

```c
//"src/java.base/share/native/libjli/java.c"

int JNICALL
JavaMain(void * _args){
    ...............
    mainClass = LoadMainClass(env, mode, what);
    CHECK_EXCEPTION_NULL_LEAVE(mainClass);
    ...............
}

static jclass
LoadMainClass(JNIEnv *env, int mode, char *name)
{
    jmethodID mid;
    jstring str;
    jobject result;
    jlong start = 0, end = 0;
    jclass cls = GetLauncherHelperClass(env);
    NULL_CHECK0(cls);
    if (JLI_IsTraceLauncher()) {
        start = CounterGet();
    }S
    NULL_CHECK0(mid = (*env)->GetStaticMethodID(env, cls,
                "checkAndLoadMain",
                "(ZILjava/lang/String;)Ljava/lang/Class;"));

    NULL_CHECK0(str = NewPlatformString(env, name));
    NULL_CHECK0(result = (*env)->CallStaticObjectMethod(env, cls, mid,
                                                        USE_STDERR, mode, str));

    if (JLI_IsTraceLauncher()) {
        end = CounterGet();
        printf("%ld micro seconds to load main class\n",
               (long)(jint)Counter2Micros(end-start));
        printf("----%s----\n", JLDEBUG_ENV_ENTRY);
    }

    return (jclass)result;
}

jclass
GetLauncherHelperClass(JNIEnv *env)
{
    if (helperClass == NULL) {
        NULL_CHECK0(helperClass = FindBootStrapClass(env,
                "sun/launcher/LauncherHelper"));
    }
    return helperClass;
}

jclass
FindBootStrapClass(JNIEnv *env, const char* classname)
{
   if (findBootClass == NULL) {
       findBootClass = (FindClassFromBootLoader_t *)dlsym(RTLD_DEFAULT,
          "JVM_FindClassFromBootLoader");
       if (findBootClass == NULL) {
           JLI_ReportErrorMessage(DLL_ERROR4,
               "JVM_FindClassFromBootLoader");
           return NULL;
       }
   }
   return findBootClass(env, classname);
}

JVM_ENTRY(jclass, JVM_FindClassFromBootLoader(JNIEnv* env,
                                              const char* name))
  JVMWrapper("JVM_FindClassFromBootLoader");

  // Java libraries should ensure that name is never null...
  if (name == NULL || (int)strlen(name) > Symbol::max_length()) {
    // It's impossible to create this class;  the name cannot fit
    // into the constant pool.
    return NULL;
  }

  TempNewSymbol h_name = SymbolTable::new_symbol(name, CHECK_NULL);
  Klass* k = SystemDictionary::resolve_or_null(h_name, CHECK_NULL);
  if (k == NULL) {
    return NULL;
  }

  if (log_is_enabled(Debug, class, resolve)) {
    trace_class_resolution(k);
  }
  return (jclass) JNIHandles::make_local(env, k->java_mirror());
JVM_END
```



```c
//"/usr/include/dlfcn.h"
/* Find the run-time address in the shared object HANDLE refers to
   of the symbol called NAME.  */
extern void *dlsym (void *__restrict __handle,
		    const char *__restrict __name) __THROW __nonnull ((2));
```

#### 

```java
//"src/java.base/share/classes/sun/launcher/LauncherHelper.java"

    public static Class<?> checkAndLoadMain(boolean printToStderr,
                                            int mode,
                                            String what) {
        initOutput(printToStderr);

        Class<?> mainClass = null;
        switch (mode) {
            case LM_MODULE: case LM_SOURCE:
                mainClass = loadModuleMainClass(what);
                break;
            default:
                mainClass = loadMainClass(mode, what);
                break;
        }

        // record the real main class for UI purposes
        // neither method above can return null, they will abort()
        appClass = mainClass;

        /*
         * Check if FXHelper can launch it using the FX launcher. In an FX app,
         * the main class may or may not have a main method, so do this before
         * validating the main class.
         */
        if (JAVAFX_FXHELPER_CLASS_NAME_SUFFIX.equals(mainClass.getName()) ||
            doesExtendFXApplication(mainClass)) {
            // Will abort() if there are problems with FX runtime
            FXHelper.setFXLaunchParameters(what, mode);
            mainClass = FXHelper.class;
        }

        validateMainClass(mainClass);
        return mainClass;
    }


 private static Class<?> loadMainClass(int mode, String what) {
        // get the class name
        String cn;
        switch (mode) {
            case LM_CLASS:
                cn = what;
                break;
            case LM_JAR:
                cn = getMainClassFromJar(what);
                break;
            default:
                // should never happen
                throw new InternalError("" + mode + ": Unknown launch mode");
        }

        // load the main class
        cn = cn.replace('/', '.');
        Class<?> mainClass = null;
        ClassLoader scl = ClassLoader.getSystemClassLoader();
        try {
            try {
                mainClass = Class.forName(cn, false, scl);
            } catch (NoClassDefFoundError | ClassNotFoundException cnfe) {
                if (System.getProperty("os.name", "").contains("OS X")
                        && Normalizer.isNormalized(cn, Normalizer.Form.NFD)) {
                    try {
                        // On Mac OS X since all names with diacritical marks are
                        // given as decomposed it is possible that main class name
                        // comes incorrectly from the command line and we have
                        // to re-compose it
                        String ncn = Normalizer.normalize(cn, Normalizer.Form.NFC);
                        mainClass = Class.forName(ncn, false, scl);
                    } catch (NoClassDefFoundError | ClassNotFoundException cnfe1) {
                        abort(cnfe1, "java.launcher.cls.error1", cn,
                                cnfe1.getClass().getCanonicalName(), cnfe1.getMessage());
                    }
                } else {
                    abort(cnfe, "java.launcher.cls.error1", cn,
                            cnfe.getClass().getCanonicalName(), cnfe.getMessage());
                }
            }
        } catch (LinkageError le) {
            abort(le, "java.launcher.cls.error6", cn,
                    le.getClass().getName() + ": " + le.getLocalizedMessage());
        }
        return mainClass;
    }
```

### 父子关系的建立

### 双亲委派

#### 缺陷:打破双亲委派

* 自定义类加载器
* spi机制

### 沙箱安全