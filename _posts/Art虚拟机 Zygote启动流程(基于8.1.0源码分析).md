# Art虚拟机 Zygote启动流程（基于8.1.0源码分析）
## 1. 在Android虚拟机中，所有的应用进程和系统服务进程都是Zygote进程fork出来的，
zygote所对应的可执行程序app_process，所对应的源文件是app_main.cpp，进程名为zygote，源代码位于 `frameworks/base/cmds/app_process/app_main.cpp`
创建一个AppRuntime变量，然后调用start函数，AppRuntime为ActivityRuntime子类也是在本文件中定义。

```

// 在init.zygote32.rc中 service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server

/system/bin/app_process     Zygote进程程序所在位置
 --zygote                   使用ZygoteInit类的main函数，Zygote进程的Java层入口
--start-system-server       需要Zygote进程启动SystemServer进程

int main(int argc, char* const argv[]){

// .......省略 ，zygote为true

    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
       fprintf(stderr, "Error: no class name or --zygote supplied.\n");
       app_usage();
       LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

## 2. 继续看ActivityRuntime的start函数:

```
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote){
    // ......省略
    // 初始化jni，加载art 或 dvm 虚拟机
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    
    // 创建虚拟机
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }

    // 注册Android方法
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
    
    // 调用了com.android.internal.os.ZygoteInit类的main函数。
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
            // ......省略
        }
    }
    free(slashClassName);
}
```
start函数做了一下4件事:

1. 创建JniInvocation实例，调用init初始化，
    1.  Init中调用GetLibrary函数中读取系统属性persist.sys.dalvik.vm.lib.2，这个值要么等于libdvm.so，要么等于libart.so。当等于libdvm.so时，就表示当前用的是Dalvik虚拟机，而当等于libart.so时，就表示当前用的是ART虚拟机。无论哪个so，都会FindSymbol出JNI_GetDefaultJavaVMInitArgs、JNI_CreateJavaVM和JNI_GetCreatedJavaVMs这3个接口，保存在JniInvocation类的三个成员变量JNI_GetDefaultJavaVMInitArgs_、JNI_CreateJavaVM_和JNI_GetCreatedJavaVMs_中 源代码路径 `libnativehelper/JniInvocation.cpp`
        
        ```
        bool JniInvocation::Init(const char* library) {
            // ......省略
              library = GetLibrary(library, buffer); // 获取dlopen的库，libart.so，libdvm.so
            // ......省略
              handle_ = dlopen(library, kDlopenFlags);
            // ......省略
            if (!FindSymbol(reinterpret_cast<void**>(&JNI_GetDefaultJavaVMInitArgs_),
                  "JNI_GetDefaultJavaVMInitArgs")) {
                return false;
              }
            if (!FindSymbol(reinterpret_cast<void**>(&JNI_CreateJavaVM_),
                  "JNI_CreateJavaVM")) {
                return false;
              }
            if (!FindSymbol(reinterpret_cast<void**>(&JNI_GetCreatedJavaVMs_),
                  "JNI_GetCreatedJavaVMs")) {
                return false;
              }
            return true;
        }
        ```
        
2. startVm虚拟机
    1. 启动虚拟机，指定一系列参数，然后调用另外一个函数JNI_CreateJavaVM来创建和初始化虚拟机，创建Runtime对象，文件路径 /art/runtime/java_vm_ext.cc
        
    ```
    int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
        // ...省略
        if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
            ALOGE("JNI_CreateJavaVM failed\n");
            return -1;
        }
        return 0;
    }
    ```
    
        
3. startReg // 注册Android方法
    
    ```
    /*
 * Register android native functions with the VM.
 */
/*static*/ int AndroidRuntime::startReg(JNIEnv* env)
{
    // ...省略 
    // 设置线程创建函数指针gCreateThreadFn指向javaCreateThreadEtc.
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    // ...省略
    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
    
    // ...省略
    return 0;
}

    #define REG_JNI(name)      { name }
    struct RegJNIRec {
        int (*mProc)(JNIEnv*);
    };

    static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)
    {
        for (size_t i = 0; i < count; i++) {
        // mProc是函数指针，
        // REG_JNI(register_com_android_internal_os_RuntimeInit).mProc，等于以下代码
        // RegJNIRec regJniRec = REG_JNI(register_com_android_internal_os_RuntimeInit)
        // regJniRec.mProc(env)
            if (array[i].mProc(env) < 0) {
                // ...省略
                return -1;
            }
        }
        return 0;
    }
    ```
    
4. 调用ZygoteInit的main函数，进入java代码:
    
    ```
        public static void main(String argv[]) {
            ZygoteServer zygoteServer = new ZygoteServer();
             // ...省略
             // 注册ServerSocket，使用文件描述符来创建的，文件描述符地址/dev/socket/zygote
             zygoteServer.registerServerSocket(socketName);
             if (!enableLazyPreload) {
                // ...省略
                // 加载Classes,Resources,OpenGL等资源
                preload(bootTimingsTraceLog);
            } else {
                Zygote.resetNicePriority();
            }
            
            if (startSystemServer) {
                // fork子进程，用于运行system_server, 会返回2次
                Runnable r = forkSystemServer(abiList, socketName, zygoteServer);
                if (r != null) {
                    r.run();
                    return;
                }
            }
             // ...省略
             // 调用runSelectLoopMode函数进入一个无限循环，等待ActivityManagerService请求创建新的应用程序进程。
             caller = zygoteServer.runSelectLoop(abiList);
        }
    ```
    
    
            

            
    






