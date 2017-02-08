---
title: Dalvik中PreVerify问题
date: 2016-11-24 13:01
categories: Android
tags: Dalvik
---
## PreVerify（预校验）的由来
Dalvik 虚拟机在启动的时候，会有许多的启动参数，其中有一项就是verify，当verify被打开的时候，doVerify变量为true，则进行类的校验（dvmVerifyClass方法调用）。若校验成功，则这个类会被打上标记：CLASS_ISPREVERIFIED。
上面描述的过程是dex -> odex (dexopt 过程)时，做的一个优化。当从classes.dex 变为 odex 后，才会被拿去执行。

## Class 被 Preverify 的过程
在dex 被 dexopt 的过程中，源码中有校验和优化 Class 相关的操作：

```
/*
 * Verify and/or optimize a specific class.
 */
static void verifyAndOptimizeClass(DexFile* pDexFile, ClassObject* clazz,
    const DexClassDef* pClassDef, bool doVerify, bool doOpt)
{
    const char* classDescriptor;
    bool verified = false;

    if (clazz->pDvmDex->pDexFile != pDexFile) {
        /*
         * The current DEX file defined a class that is also present in the
         * bootstrap class path.  The class loader favored the bootstrap
         * version, which means that we have a pointer to a class that is
         * (a) not the one we want to examine, and (b) mapped read-only,
         * so we will seg fault if we try to rewrite instructions inside it.
         */
        ALOGD("DexOpt: not verifying/optimizing '%s': multiple definitions",
            clazz->descriptor);
        return;
    }

    classDescriptor = dexStringByTypeIdx(pDexFile, pClassDef->classIdx);

    /*
     * First, try to verify it.
     */
    if (doVerify) {
        if (dvmVerifyClass(clazz)) {
            /*
             * Set the "is preverified" flag in the DexClassDef.  We
             * do it here, rather than in the ClassObject structure,
             * because the DexClassDef is part of the odex file.
             */
            assert((clazz->accessFlags & JAVA_FLAGS_MASK) ==
                pClassDef->accessFlags);
            ((DexClassDef*)pClassDef)->accessFlags |= CLASS_ISPREVERIFIED;
            verified = true;
        } else {
            // TODO: log when in verbose mode
            ALOGV("DexOpt: '%s' failed verification", classDescriptor);
        }
    }

    if (doOpt) {
        bool needVerify = (gDvm.dexOptMode == OPTIMIZE_MODE_VERIFIED ||
                           gDvm.dexOptMode == OPTIMIZE_MODE_FULL);
        if (!verified && needVerify) {
            ALOGV("DexOpt: not optimizing '%s': not verified",
                classDescriptor);
        } else {
            dvmOptimizeClass(clazz, false);// 优化 Class 操作

            /* set the flag whether or not we actually changed anything */
            ((DexClassDef*)pClassDef)->accessFlags |= CLASS_ISOPTIMIZED;// Class被优化过后，也会打上被优化过的标记 CLASS_ISOPTIMIZED
        }
    }
}
```

dvmVerifyClass 的具体过程，源码中是这样描述的：

```
/*
 * Verify a class.
 *
 * By the time we get here, the value of gDvm.classVerifyMode should already
 * have been factored in.  If you want to call into the verifier even
 * though verification is disabled, that's your business.
 *
 * Returns "true" on success.
 */
bool dvmVerifyClass(ClassObject*clazz) {
    int i;
    if (dvmIsClassVerified(clazz)) {
        ALOGD("Ignoring duplicate verify attempt on %s", clazz -> descriptor);
        return true;
    }
    for (i = 0; i < clazz -> directMethodCount; i++) {
        if (!verifyMethod( & clazz -> directMethods[i])){
            LOG_VFY("Verifier rejected class %s", clazz -> descriptor);
            return false;
        }
    }
    for (i = 0; i < clazz -> virtualMethodCount; i++) {
        if (!verifyMethod( & clazz -> virtualMethods[i])){
            LOG_VFY("Verifier rejected class %s", clazz -> descriptor);
            return false;
        }
    }
    return true;
}
```

其中，dvmVerifyClass中又去调用了 方法的校验 verifyMethod, 再去追源码：

```
/*
 * Perform verification on a single method.
 */
static bool verifyMethod(Method* meth)
{
    bool result = false;

    /*
     * Verifier state blob.  Various values will be cached here so we
     * can avoid expensive lookups and pass fewer arguments around.
     */
    VerifierData vdata;
#if 1   // ndef NDEBUG
    memset(&vdata, 0x99, sizeof(vdata));
#endif

    vdata.method = meth;
    vdata.insnsSize = dvmGetMethodInsnsSize(meth);
    vdata.insnRegCount = meth->registersSize;
    vdata.insnFlags = NULL;
    vdata.uninitMap = NULL;
    vdata.basicBlocks = NULL;

    /*
     * If there aren't any instructions, make sure that's expected, then
     * exit successfully.  Note: for native methods, meth->insns gets set
     * to a native function pointer on first call, so don't use that as
     * an indicator.
     */
    if (vdata.insnsSize == 0) {
        if (!dvmIsNativeMethod(meth) && !dvmIsAbstractMethod(meth)) {
            LOG_VFY_METH(meth,
                "VFY: zero-length code in concrete non-native method");
            goto bail;
        }

        goto success;
    }

    /*
     * Sanity-check the register counts.  ins + locals = registers, so make
     * sure that ins <= registers.
     */
    if (meth->insSize > meth->registersSize) {
        LOG_VFY_METH(meth, "VFY: bad register counts (ins=%d regs=%d)",
            meth->insSize, meth->registersSize);
        goto bail;
    }

    /*
     * Allocate and populate an array to hold instruction data.
     *
     * TODO: Consider keeping a reusable pre-allocated array sitting
     * around for smaller methods.
     */
    vdata.insnFlags = (InsnFlags*) calloc(vdata.insnsSize, sizeof(InsnFlags));
    if (vdata.insnFlags == NULL)
        goto bail;

    /*
     * Compute the width of each instruction and store the result in insnFlags.
     * Count up the #of occurrences of certain opcodes while we're at it.
     */
    if (!computeWidthsAndCountOps(&vdata))
        goto bail;

    /*
     * Allocate a map to hold the classes of uninitialized instances.
     */
    vdata.uninitMap = dvmCreateUninitInstanceMap(meth, vdata.insnFlags,
        vdata.newInstanceCount);
    if (vdata.uninitMap == NULL)
        goto bail;

    /*
     * Set the "in try" flags for all instructions guarded by a "try" block.
     * Also sets the "branch target" flag on exception handlers.
     */
    if (!scanTryCatchBlocks(meth, vdata.insnFlags))
        goto bail;

    /*
     * Perform static instruction verification.  Also sets the "branch
     * target" flags.
     */
    if (!verifyInstructions(&vdata))
        goto bail;

    /*
     * Do code-flow analysis.
     *
     * We could probably skip this for a method with no registers, but
     * that's so rare that there's little point in checking.
     */
    if (!dvmVerifyCodeFlow(&vdata)) {
        //ALOGD("+++ %s failed code flow", meth->name);
        goto bail;
    }

success:
    result = true;

bail:
    dvmFreeVfyBasicBlocks(&vdata);
    dvmFreeUninitInstanceMap(vdata.uninitMap);
    free(vdata.insnFlags);
    return result;
}
```

dvmVerifyClass 具体过程中，会去校验 这个 Class中的所有的 directMethod 方法，和 virtualMethod 方法。具体这些方法包含哪些呢？ 其中时包含了：
* static 方法
* private 方法
* 构造方法
* ... ...

由此可知：  如果这些方法中直接引用到的类（第一层级关系，不会进行递归搜索） 和 class (源码中的clazz) 在同一个 dex 中的话，这个类就会被打上  CLASS_ISPREVERIFY 标记

## PreVerify 缘由

* 一方面，Dalvik 虚拟机在安装期间，为Class 打上 CLASS_ISPREVERIFIED 是为了提高性能，下次使用时，则会省去校验操作，提高访问效率。
* 另一方面，被标上“CLASS_ISPREVERIFIED”的类，dvm在运行期载入Class时候，会对其内存中对应的直接引用类进行校验，如果该类存在与直接引用类所在的dex不是同一个，则直接报“pre-verification” 错误，该类无法加载。

```
ClassObject* dvmResolveClass(const ClassObject* referrer, u4 classIdx,
    bool fromUnverifiedConstant)
{
    DvmDex* pDvmDex = referrer->pDvmDex;
    ClassObject* resClass;
    const char* className;
    /*
     * Check the table first -- this gets called from the other "resolve"
     * methods.
     */
    resClass = dvmDexGetResolvedClass(pDvmDex, classIdx); // 预先在dex的缓存表里查
    if (resClass != NULL)
        return resClass;
    LOGVV("--- resolving class %u (referrer=%s cl=%p)",
        classIdx, referrer->descriptor, referrer->classLoader);
    /*
     * Class hasn't been loaded yet, or is in the process of being loaded
     * and initialized now.  Try to get a copy.  If we find one, put the
     * pointer in the DexTypeId.  There isn't a race condition here --
     * 32-bit writes are guaranteed atomic on all target platforms.  Worst
     * case we have two threads storing the same value.
     *
     * If this is an array class, we'll generate it here.
     */
    className = dexStringByTypeIdx(pDvmDex->pDexFile, classIdx);
    if (className[0] != '\0' && className[1] == '\0') {
        /* primitive type */
        resClass = dvmFindPrimitiveClass(className[0]);
    } else {
        resClass = dvmFindClassNoInit(className, referrer->classLoader);
    }
    if (resClass != NULL) {
        /*
         * If the referrer was pre-verified, the resolved class must come
         * from the same DEX or from a bootstrap class.  The pre-verifier
         * makes assumptions that could be invalidated by a wacky class
         * loader.  (See the notes at the top of oo/Class.c.)
         *
         * The verifier does *not* fail a class for using a const-class
         * or instance-of instruction referring to an unresolveable class,
         * because the result of the instruction is simply a Class object
         * or boolean -- there's no need to resolve the class object during
         * verification.  Instance field and virtual method accesses can
         * break dangerously if we get the wrong class, but const-class and
         * instance-of are only interesting at execution time.  So, if we
         * we got here as part of executing one of the "unverified class"
         * instructions, we skip the additional check.
         *
         * Ditto for class references from annotations and exception
         * handler lists.
         */
        if (!fromUnverifiedConstant &&
            IS_CLASS_FLAG_SET(referrer, CLASS_ISPREVERIFIED))
        {
            ClassObject* resClassCheck = resClass;
            if (dvmIsArrayClass(resClassCheck))
                resClassCheck = resClassCheck->elementClass;
            if (referrer->pDvmDex != resClassCheck->pDvmDex &&
                resClassCheck->classLoader != NULL)  // 校验dex是否是安装时的同一个dex
            {
                ALOGW("Class resolved by unexpected DEX:"
                     " %s(%p):%p ref [%s] %s(%p):%p",
                    referrer->descriptor, referrer->classLoader,
                    referrer->pDvmDex,
                    resClass->descriptor, resClassCheck->descriptor,
                    resClassCheck->classLoader, resClassCheck->pDvmDex);
                ALOGW("(%s had used a different %s during pre-verification)",
                    referrer->descriptor, resClass->descriptor);
                dvmThrowIllegalAccessError(
                    "Class ref in pre-verified class resolved to unexpected "
                    "implementation");
                return NULL;
            }
        }
        LOGVV("##### +ResolveClass(%s): referrer=%s dex=%p ldr=%p ref=%d",
            resClass->descriptor, referrer->descriptor, referrer->pDvmDex,
            referrer->classLoader, classIdx);
        /*
         * Add what we found to the list so we can skip the class search
         * next time through.
         *
         * TODO: should we be doing this when fromUnverifiedConstant==true?
         * (see comments at top of oo/Class.c)
         */
        dvmDexSetResolvedClass(pDvmDex, classIdx, resClass);
    } else {
        /* not found, exception should be raised */
        LOGVV("Class not found: %s",
            dexStringByTypeIdx(pDvmDex->pDexFile, classIdx));
        assert(dvmCheckException(dvmThreadSelf()));
    }
    return resClass;
}
```

实际上，这一步dex的一致性判断，也是google为了防止外部DEX注入的一个安全方案，即保证运行期的Class与其直接引用类之间所在的DEX关系要与安装时候一致


<br>
源码链接:
http://osxr.org:8080/android/source/dalvik/vm/analysis/DexPrepare.cpp
http://osxr.org:8080/android/source/dalvik/vm/analysis/DexVerify.cpp
http://osxr.org:8080/android/source/dalvik/vm/oo/Resolve.cpp

