---
layout: post
title: "SharePreferences 分析及正确使用姿势"
date: 2021-05-01
description: "SharePreferences 分析及正确使用姿势"
tag: SharePreferences
---
### 1.Android 常见数据存储方式

在 Android 中，常用数据存储方式通常有以下几类：

**文件存储**：将数据存储在文件中。文件存储根据位置不同，可以存储在应用包下，成为内部存储；也可以存储在 storage文件夹（当然也有可能是mnt文件夹，不同的手机厂商可能不一样）上，称之为外部存储

![store](https://github.com/RalfNick/PicRepository/raw/master/SharedPreferences/sp_store.png)

> - SharedPreferences 存储：SharedPreferences 是 Android 提供的用来存储一些简单配置信息的一种机制，核心原理是：保存基于 XML 文件存储的 key-value 键值对数据。通常使用该种方式用来存储一些简单信息，例如：应用版本信息，应用主题类型，开关配置，用户信息等等。其采用 Map 数据结构来存储数据，以键值对方式存储，使得读取与写入很方便，属于一种轻量级的存储机制。
> - SQLite数据库存储：Android 系统中轻量级关系型数据，允许用户进行创建表结构，存储应用数据等操作
> - 使用 ContentProvider 存储数据：在应用程序之间，共享或者传递相关信息时，往往可以使用 Content Provider 和 ContentResolver 实现
> - 网络获取：存储在服务器上，通过接口数据从服务器后台获取，需要网络访问

### 2.SharedPreferences

SharedPreferences 本身是一个接口，无法直接创建 SharedPreferences 实例。可以通过 Context 提供的 getSharedPreferences(String name, int mode)方法来获取 SharedPreferences 实例，第一个参数表示要操作的xml文件名，第二个参数表示操作模式：MODE_PRIVATE、MODE_WORLD_READABLE、MODE_WORLD_WRITEABLE，推荐使用 MODE_PRIVATE。

Editor：SharedPreferences 只能获取数据，不能存储和修改。存储修改是通过 SharedPreferences.edit() 获取的内部接口 Editor 对象实现。

SharedPreferences 简单使用：

```java
val sp = getSharedPreferences("sp_test", Context.MODE_PRIVATE)
val edit = sp.edit()
edit.putString("key", "123")
edit.putInt("number", 10)
edit.apply()
println("sp_string " + sp.getString("key", ""))
println("sp_string " + sp.getInt("number", 0))
```

SharedPreferences 对应的 xml 文件位置：/data/data/package name/shared_prefs/

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <int name="number" value="10" />
    <string name="key">123</string>
</map>
```

### 3. SharedPreferences 源码分析

在分析 SharedPreferences 源码之前，我们带着几个问题来看：

> - 为什么 SharedPreferences 适用于少量数据存储？
> - SharedPreferences 的 commit 和 apply 有什么区别，分别在什么场景下使用？
> - 为什么 SharedPreferences 会导致卡顿、ANR 等问题？
> - 如何正确使用 SharedPreferences，以及我们可以做哪些优化？
> - 多进程下使用 SharedPreferences 是否安全？

#### 3.1 SharedPreferences 实例获取

SharedPreferences 实例获取是通过 Context 来获取，具体实现在 ContextImpl 中。在 ContextImpl 中有这几个变量：

```java
// 记录所有的 SharedPreferences 文件，key 为文件名，value 为文件
private ArrayMap<String, File> mSharedPrefsPaths;

// 以包名为key, 二级key是以SP文件, 以 SharedPreferencesImpl 为value 的嵌套 map 结构. 注意：sSharedPrefsCache 是静态类成员变量, 每个进程只保存唯一一份, 且由 ContextImpl.class 锁保护.
private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache;

// 记录 SharedPreferences 所在文件路径，/data/data/包名/shared_prefs/
private File mPreferencesDir;
```
下面来看看如果获取 SharedPreferences 实例：

首先调用 getSharedPreferences 方法，该方法先找到名字为 name 的 xml 文件，不存在的话则创建一个新文件。可以看到 mSharedPrefsPaths 在这个方法中创建，mSharedPrefsPaths 不是 static 变量，相当于每个 Context 都有一个 mSharedPrefsPaths，但相当于 static 变量，因为创建时是以 ContextImpl.class 加锁保护的，也就是说同一个文件名的 xml 文件，有且仅有一个。

```java
@Override
public SharedPreferences getSharedPreferences(String name, int mode) {
    if (mPackageInfo.getApplicationInfo().targetSdkVersion <
            Build.VERSION_CODES.KITKAT) {
        if (name == null) {
            name = "null";
        }
    }
    File file;
    synchronized (ContextImpl.class) {
        if (mSharedPrefsPaths == null) {
            mSharedPrefsPaths = new ArrayMap<>();
        }
        file = mSharedPrefsPaths.get(name);
        if (file == null) {
            file = getSharedPreferencesPath(name);
            mSharedPrefsPaths.put(name, file);
        }
    }
    return getSharedPreferences(file, mode);
}
```

获取指定路径下的 Sharepreferences 文件

```java
// 返回的是获取的 Sharepreferences 文件 File，文本全路径为 /data/data/包名/shared_prefs/name.xml
@Override
public File getSharedPreferencesPath(String name) {
    return makeFilename(getPreferencesDir(), name + ".xml");
}

// mPreferencesDir 为 Sharepreferences 的文件路径，即 /data/data/包名/shared_prefs/
@UnsupportedAppUsage
private File getPreferencesDir() {
    synchronized (mSync) {
        if (mPreferencesDir == null) {
            mPreferencesDir = new File(getDataDir(), "shared_prefs");
        }
        return ensurePrivateDirExists(mPreferencesDir);
    }
}
```

有了 name.xml 文件，开始创建 SharedPreferencesImpl 对象，注意只是首次创建，后面不再创建，因为内存中有缓存该对象。

```java
@Override
public SharedPreferences getSharedPreferences(File file, int mode) {
    SharedPreferencesImpl sp;
    synchronized (ContextImpl.class) {
        final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
        sp = cache.get(file);
        if (sp == null) {
            checkMode(mode);
            if (getApplicationInfo().targetSdkVersion >= android.os.Build.VERSION_CODES.O) {
                if (isCredentialProtectedStorage()
                        && !getSystemService(UserManager.class)
                                .isUserUnlockingOrUnlocked(UserHandle.myUserId())) {
                    throw new IllegalStateException("SharedPreferences in credential encrypted "
                            + "storage are not available until after user is unlocked");
                }
            }
            // 创建 SharedPreferencesImpl 对象并缓存起来
            sp = new SharedPreferencesImpl(file, mode);
            cache.put(file, sp);
            return sp;
        }
    }
    // MODE_MULTI_PROCESS 已经废弃，不再建议使用，多进程下不能保证数据安全
    if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
        getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
        // If somebody else (some other process) changed the prefs
        // file behind our back, we reload it.  This has been the
        // historical (if undocumented) behavior.
        sp.startReloadIfChangedUnexpectedly();
    }
    return sp;
}

// 创建缓存集合 sSharedPrefsCache，key 是包名，value 是以 file 为key，value 为 SharedPreferencesImpl 对象的嵌套 map 集合，也就是获取该当前包名下 ArrayMap<File, SharedPreferencesImpl>，file 和 SharedPreferencesImpl 一一对应。
@GuardedBy("ContextImpl.class")
private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
    if (sSharedPrefsCache == null) {
        sSharedPrefsCache = new ArrayMap<>();
    }

    final String packageName = getPackageName();
    ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
    if (packagePrefs == null) {
        packagePrefs = new ArrayMap<>();
        sSharedPrefsCache.put(packageName, packagePrefs);
    }

    return packagePrefs;
}
```

android N 以后不再支持 MODE_WORLD_READABLE 和 MODE_WORLD_WRITEABLE。

```java
private void checkMode(int mode) {
    if (getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.N) {
        if ((mode & MODE_WORLD_READABLE) != 0) {
            throw new SecurityException("MODE_WORLD_READABLE no longer supported");
        }
        if ((mode & MODE_WORLD_WRITEABLE) != 0) {
            throw new SecurityException("MODE_WORLD_WRITEABLE no longer supported");
        }
    }
}
```

下面来看下 MODE_MULTI_PROCESS 这种扩进程方式是不安全的，在 MODE_MULTI_PROCESS 模式下，如果文件被修改过或重新加载，使得数据是最新的。

```java
@UnsupportedAppUsage
void startReloadIfChangedUnexpectedly() {
    synchronized (mLock) {
        // TODO: wait for any pending writes to disk?
        if (!hasFileChangedUnexpectedly()) {
            return;
        }
        startLoadFromDisk();
    }
}

// 获取文件 mFile 的 StructStat 信息，根据文件的时间戳和文件大小判定是不是修改过，注意虽然加了 mLock，包括重新加载方法 startLoadFromDisk 也是加锁的，但是在多进程下，是有多个虚拟机的，加载对于多进程是无效的，所以对于多进程下使用 MODE_MULTI_PROCESS 是不安全的。
private boolean hasFileChangedUnexpectedly() {
    synchronized (mLock) {
        if (mDiskWritesInFlight > 0) {
            // If we know we caused it, it's not unexpected.
            if (DEBUG) Log.d(TAG, "disk write in flight, not unexpected.");
            return false;
        }
    }
    final StructStat stat;
    try {
        BlockGuard.getThreadPolicy().onReadFromDisk();
        stat = Os.stat(mFile.getPath());
    } catch (ErrnoException e) {
        return true;
    }
    //
    synchronized (mLock) {
        return !stat.st_mtim.equals(mStatTimestamp) || mStatSize != stat.st_size;
    }
}
```

#### 3.2 SharedPreferences 分析

![class_pic](https://github.com/RalfNick/PicRepository/raw/master/SharedPreferences/shared_preference.jpeg)

SharedPreferences 与 Editor 是两个接口,SharedPreferencesImpl 和 EditorImpl 分别实现了对应接口. SharedPreferencesImpl 提供查询各个类型数据的方法，getXX(String key, float defValue),而如果是存储数据和删除等操作，需要通过 edit() 方法获取 EditorImpl 实例来操作。

- 先来看下 SharedPreferencesImpl 构造

```java
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    mLoaded = false;
    mMap = null;
    mThrowable = null;
    startLoadFromDisk();
}
```

这里主要做两件事，一个是先备份 name.xml 文件，备份文件名字是 name.bak。另一件是从文件中加载数据到内存中。下面主要看下加载过程。

```java
// 通过 mLock 锁来确保 mLoaded 状态同步，然后开启一个线程来加载文件
@UnsupportedAppUsage
private void startLoadFromDisk() {
    synchronized (mLock) {
        mLoaded = false;
    }
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}

// 开始加载文件到内存中
private void loadFromDisk() {
    synchronized (mLock) {
        if (mLoaded) {
            return;
        }
        删除原有文件，并将备份文件重命名为 mFile
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }

    // Debugging
    if (mFile.exists() && !mFile.canRead()) {
        Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
    }

    // 这段主要操作是通过流操作来读取 XML 文件，将里面的 key-value 读取到 Map 中
    Map<String, Object> map = null;
    StructStat stat = null;
    Throwable thrown = null;
    try {
        stat = Os.stat(mFile.getPath());
        if (mFile.canRead()) {
            BufferedInputStream str = null;
            try {
                str = new BufferedInputStream(
                        new FileInputStream(mFile), 16 * 1024);
                map = (Map<String, Object>) XmlUtils.readMapXml(str);
            } catch (Exception e) {
                Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
            } finally {
                IoUtils.closeQuietly(str);
            }
        }
    } catch (ErrnoException e) {
        // An errno exception means the stat failed. Treat as empty/non-existing by
        // ignoring.
    } catch (Throwable t) {
        thrown = t;
    }

    // 读取完成后的操作
    // 1.记录文件的操作时间和文件大小
    // 2.读取完成后则通知其他等待的位置唤醒，继续执行操作，因为等待的地方是在等待文件加载完成，也就是说任何操作都是需要在文件加载完成的基础上执行的
    synchronized (mLock) {
        mLoaded = true;
        mThrowable = thrown;

        // It's important that we always signal waiters, even if we'll make
        // them fail with an exception. The try-finally is pretty wide, but
        // better safe than sorry.
        try {
            if (thrown == null) {
                if (map != null) {
                    mMap = map;
                    mStatTimestamp = stat.st_mtim;
                    mStatSize = stat.st_size;
                } else {
                    mMap = new HashMap<>();
                }
            }
            // In case of a thrown exception, we retain the old map. That allows
            // any open editors to commit and store updates.
        } catch (Throwable t) {
            mThrowable = t;
        } finally {
            mLock.notifyAll();
        }
    }
}
```

文件未加载前任何操作都需要等待,getXX 方法和 edit 方法

```java
@GuardedBy("mLock")
private void awaitLoadedLocked() {
    if (!mLoaded) {
        // Raise an explicit StrictMode onReadFromDisk for this
        // thread, since the real read will be in a different
        // thread and otherwise ignored by StrictMode.
        BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    while (!mLoaded) {
        try {
            mLock.wait();
        } catch (InterruptedException unused) {
        }
    }
    if (mThrowable != null) {
        throw new IllegalStateException(mThrowable);
    }
}
```

- 查询方法

查询方法比较简单，文件加载完成后则直接从内存中读取数据，即从 Map 中查询数据，以 getString 方法为例：

```java
@Override
@Nullable
public String getString(String key, @Nullable String defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}
```
注意，读取操作是同步操作，因为 mMap 中数据是有可能改变的，这里猜想一下，为什么不用 ConcurrentHashMap 呢？是不就可以不用加锁处理了？个人认为是可以的，但是有一点 ConcurrentHashMap 是在 jdk 5.0 才出现的，意味着之前的版本还是得用加锁处理，索性干脆直接加锁处理。这里除了加锁处理还有一个 awaitLoadedLocked() 方法，即上面提到的需要在文件加载完成才可以查询，否则会阻塞在这里。

- edit 方法

```java
@Override
public Editor edit() {
    synchronized (mLock) {
        awaitLoadedLocked();
    }

    return new EditorImpl();
}
```
同样需要等待文件加载完成，否则也会等待

- EditorImpl

EditorImpl 实现 Editor 接口，完成 SharePreferences 的添加、删除、清除、commit 和 apply 操作。EditorImpl 有两个变量 mModified 和 mClear。putXX,remove 操作都是基于 mModified 进行的，mClear 用于 clear 方法。

下面看下 putString 和 remove 方法，方法操作都加了所操作，保证数据的同步性，remove 方法有点特殊，待删除的值用 EditorImpl 自己来占位

```java
@Override
public Editor putString(String key, @Nullable String value) {
    synchronized (mEditorLock) {
        mModified.put(key, value);
        return this;
    }
}

@Override
public Editor remove(String key) {
    synchronized (mEditorLock) {
        mModified.put(key, this);
        return this;
    }
}
```

调用 EditorImpl 的 putXX 等方法仅仅将要操作的元素放在 mModified，还没有和 SharePreferences 中的元素同步以及写入文件中，调用 commit 或者 apply 方法才会进行提交到内存并写入文件中。

- 数据提交

commit 方法：

```java
@Override
public boolean commit() {
    long startTime = 0;
    // 提交到内存中，将 mModified 中数据和 SharePreferences 中 Map 同步
    MemoryCommitResult mcr = commitToMemory();

    // 开启写入文件任务，可以看到注释中提示 commit 方法是同步操作，即在当前线程直接指定写文件操作，这一点需要注意
    SharedPreferencesImpl.this.enqueueDiskWrite(
        mcr, null /* sync write on this thread okay */);
        // 这里虽然调用了 await，并不会阻塞，因为 commit 方法是同步操作执行完成时 writtenToDiskLatch 中当前的 count 是 0，，会直接向下执行。
    try {
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    }
    notifyListeners(mcr);
    return mcr.writeToDiskResult;
}
```

commit 是同步操作方法，也是耗时方法，毕竟需要进行写文件操作。执行分为两步：

> - 将修改的数据提交到内存中 commitToMemory
> - enqueueDiskWrite,执行写文件操作

```java
// Returns true if any changes were made
private MemoryCommitResult commitToMemory() {
    long memoryStateGeneration;
    boolean keysCleared = false;
    List<String> keysModified = null;
    Set<OnSharedPreferenceChangeListener> listeners = null;
    // 最终需要写入文件的数据
    Map<String, Object> mapToWriteToDisk;

    synchronized (SharedPreferencesImpl.this.mLock) {

        // 如果正在写文件，则拷贝一份 mMap
        if (mDiskWritesInFlight > 0) {
            // We can't modify our mMap as a currently
            // in-flight write owns it.  Clone it before
            // modifying it.
            // noinspection unchecked
            mMap = new HashMap<String, Object>(mMap);
        }
        // mapToWriteToDisk 在这里时代表的旧的数据
        mapToWriteToDisk = mMap;
        mDiskWritesInFlight++;

        boolean hasListeners = mListeners.size() > 0;
        if (hasListeners) {
            keysModified = new ArrayList<String>();
            listeners = new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
        }

        synchronized (mEditorLock) {
            boolean changesMade = false;

           // 调用 EditorImpl 的 clear 方法会走这里
            if (mClear) {
                if (!mapToWriteToDisk.isEmpty()) {
                    changesMade = true;
                    mapToWriteToDisk.clear();
                }
                keysCleared = true;
                mClear = false;
            }
            // 这里遍历 EditorImpl 中的 mModified，并和 mapToWriteToDisk 中的数据对比，此时 mapToWriteToDisk 中是修改前的数据，遍历完成后则修改为最终新的数据
            for (Map.Entry<String, Object> e : mModified.entrySet()) {
                String k = e.getKey();
                Object v = e.getValue();
                // "this" is the magic value for a removal mutation. In addition,
                // setting a value to "null" for a given key is specified to be
                // equivalent to calling remove on that key.
                // 注意这里的 this，上面提到过 this 在删除时用于占位，被删除的 key，则会从 mapToWriteToDisk 中删除
                if (v == this || v == null) {
                    if (!mapToWriteToDisk.containsKey(k)) {
                        continue;
                    }
                    mapToWriteToDisk.remove(k);
                } else {
                    // mapToWriteToDisk 中已经有了 mModified 中的元素，则不进行操作
                    if (mapToWriteToDisk.containsKey(k)) {
                        Object existingValue = mapToWriteToDisk.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    // 否则不存在或者元素修改过，则添加 mapToWriteToDisk 中或者覆盖旧的元素
                    mapToWriteToDisk.put(k, v);
                }

                changesMade = true;
                if (hasListeners) {
                    keysModified.add(k);
                }
            }

            mModified.clear();

            if (changesMade) {
                mCurrentMemoryStateGeneration++;
            }

            memoryStateGeneration = mCurrentMemoryStateGeneration;
        }
    }
    return new MemoryCommitResult(memoryStateGeneration, keysCleared, keysModified,
            listeners, mapToWriteToDisk);
}
```

commitToMemory() 方法中代码虽然有点多，但是逻辑还是比较清晰的，主要的操作就是先将 SharePreferences 中 mMap 赋给 mapToWriteToDisk，此时 mapToWriteToDisk 中是修改前的元素，再和 EditorImpl 中的 mModified 中元素对比，得到最终的元素，这样就将修改结果同步到内存中了。

enqueueDiskWrite 是执行写文件的入口，commit 方法调用时传入的 postWriteRunnable 为 null，所以 isFromSyncCommit 为 true，代表同步操作，不需要提交到 QueuedWork 的队列中。

```java
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {
    final boolean isFromSyncCommit = (postWriteRunnable == null);

    final Runnable writeToDiskRunnable = new Runnable() {
            @Override
            public void run() {
                synchronized (mWritingToDiskLock) {
                    writeToFile(mcr, isFromSyncCommit);
                }
                synchronized (mLock) {
                    mDiskWritesInFlight--;
                }
                if (postWriteRunnable != null) {
                    postWriteRunnable.run();
                }
            }
        };

    // Typical #commit() path with fewer allocations, doing a write on
    // the current thread.
    // 在 commitToMemory 中 mDiskWritesInFlight++，所以 对于 commit 方法，会直接调用 writeToDiskRunnable.run()，直接进行写文件操作
    if (isFromSyncCommit) {
        boolean wasEmpty = false;
        synchronized (mLock) {
            wasEmpty = mDiskWritesInFlight == 1;
        }
        if (wasEmpty) {
            writeToDiskRunnable.run();
            return;
        }
    }

    QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}
```

写文件操作

```java
@GuardedBy("mWritingToDiskLock")
private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
    long startTime = 0;
    long existsTime = 0;
    long backupExistsTime = 0;
    long outputStreamCreateTime = 0;
    long writeTime = 0;
    long fsyncTime = 0;
    long setPermTime = 0;
    long fstatTime = 0;
    long deleteTime = 0;

    boolean fileExists = mFile.exists();

    // Rename the current file so it may be used as a backup during the next read
    // 文件存在情况下，先进行备份，防止异常情况下数据丢失
    if (fileExists) {
        boolean needsWrite = false;

        // Only need to write if the disk state is older than this commit
        // 当写入计数小于内存计数时才进行写入
        if (mDiskStateGeneration < mcr.memoryStateGeneration) {
        // 如果是同步写，即调用 commit 则需要写入
            if (isFromSyncCommit) {
                needsWrite = true;
            } else {
            // 调用 apply 方法时，需要保持 mcr 中数据和  SharePreferences 中的数据保持一致才进行写入，比如对于同一个sp文件，连续调用 n 次 apply ,就会有 n 次写入磁盘任务执行，实际上只需要最后执行最后那次就可以了，最后那次提交对应内存的 map 是持有最新的数据，所以就可以省掉前面 n-1 次的执行，这个就是 android 8.0 中做的优化
                synchronized (mLock) {
                    // No need to persist intermediate states. Just wait for the latest state to
                    // be persisted.
                    if (mCurrentMemoryStateGeneration == mcr.memoryStateGeneration) {
                        needsWrite = true;
                    }
                }
            }
        }

        // 不需要写入则直接放回
        if (!needsWrite) {
            mcr.setDiskWriteResult(false, true);
            return;
        }

        // 备份文件,防止异常情况，写入成功后再删除
        boolean backupFileExists = mBackupFile.exists();
        if (!backupFileExists) {
            if (!mFile.renameTo(mBackupFile)) {
                Log.e(TAG, "Couldn't rename file " + mFile
                      + " to backup file " + mBackupFile);
                mcr.setDiskWriteResult(false, false);
                return;
            }
        } else {
            mFile.delete();
        }
    }

    // Attempt to write the file, delete the backup and return true as atomically as
    // possible.  If any exception occurs, delete the new file; next time we will restore
    // from the backup.
    try {
        FileOutputStream str = createFileOutputStream(mFile);

        if (str == null) {
            mcr.setDiskWriteResult(false, false);
            return;
        }
        XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);

        writeTime = System.currentTimeMillis();

        FileUtils.sync(str);

        fsyncTime = System.currentTimeMillis();

        str.close();
        ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);

        try {
            final StructStat stat = Os.stat(mFile.getPath());
            synchronized (mLock) {
                mStatTimestamp = stat.st_mtim;
                mStatSize = stat.st_size;
            }
        } catch (ErrnoException e) {
            // Do nothing
        }

        // 写入成功，删除备份文件
        mBackupFile.delete();
        mDiskStateGeneration = mcr.memoryStateGeneration;

        // 结果通知，写入成功，唤醒阻塞的位置
        mcr.setDiskWriteResult(true, true);

        long fsyncDuration = fsyncTime - writeTime;
        mSyncTimes.add((int) fsyncDuration);
        mNumSync++;

        return;
    } catch (XmlPullParserException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    } catch (IOException e) {
        Log.w(TAG, "writeToFile: Got exception:", e);
    }

    // 写入失败请款下则删除 mFile,下次从备份文件恢复
    if (mFile.exists()) {
        if (!mFile.delete()) {
            Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
        }
    }
    mcr.setDiskWriteResult(false, false);
}
```

写入文件主要是将 commitToMemory() 方法中的 mapToWriteToDisk 集合数据写入文件中，可以看出每次写入都是全量写入文件,如果写入成功则删除备份文件,如果写入失败则删除mFile.
可见, 每次commit是把全部数据更新到文件, 所以每个文件的数据量必须保证足够精简。每次修改数据进行一次性调用 commit，而且尽量不要在主线程中调用，因为 commit 方法是同步操作，写文件耗时操作会在当前线程中执行，在主线程调用可能会导致卡顿、ANR 等问题。

再来看看 apply 方法

```java
@Override
public void apply() {
    final long startTime = System.currentTimeMillis();

    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
            @Override
            public void run() {
                try {
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }

                if (DEBUG && mcr.wasWritten) {
                    Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                            + " applied after " + (System.currentTimeMillis() - startTime)
                            + " ms");
                }
            }
        };

    QueuedWork.addFinisher(awaitCommit);

    Runnable postWriteRunnable = new Runnable() {
            @Override
            public void run() {
                awaitCommit.run();
                QueuedWork.removeFinisher(awaitCommit);
            }
        };

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);
    notifyListeners(mcr);
}
```

apply 方法中多了一个 awaitCommit，这是一个等待操作，android 系统会在 Activity 的 onStop ,onPause 等生命周期中，调用 QueuedWork.waitToFinish，等待落盘的任务队列执行完成，但是如果任务队列中的任务很多，或者待写入的数据量很大时(sp 文件是全量读写的)，在一些 io 性能差的中低端机型上就会很容易出现 anr。

在 Android 8.0 下进行了优化，8.0 之前只会等待 awaitCommit，在低端机上多次调用 apply 时很容易出现 ANR，8.0 之后 会主动触发 processPendingWork 取出写任务列表中依次执行，而不是只在在等待。同时 8.0 之后对多次调用 apply 方法也进行了优化，比如对于同一个sp文件，连续调用 n 次 apply ,就会有 n 次写入磁盘任务执行，实际上只需要最后执行最后那次就可以了，最后那次提交对应内存的 map 是持有最新的数据，所以就可以省掉前面 n-1 次的执行，这个就是 android 8.0 中做的优化。

### 4. SharePreferences 优化

#### 4.1 apply 和 commit 主要区别

> - 在于 apply 的写入文件操作是在单线程的线程池来完成。apply方法开始的时候, 会把 awaitCommit 放入 QueuedWork;文件写入操作完成, 则会把相应的 awaitCommit 从 QueuedWork 中移除。QueuedWork 在这里存在的价值主要是用于在 Stop Service, finish BroadcastReceiver 过程用于判定是否处理完所有的异步 SP 操作.
> - apply没有返回值, commit 有返回值能知道修改是否提交成功
> - apply是将修改提交到内存，再异步提交到磁盘文件; commit 是同步的提交到磁盘文件;
> - 多并发的提交 commit 时，需等待正在处理的 commit 数据更新到磁盘文件后才会继续往下执行，从而降低效率; 而 apply 只是原子更新到内存，后调用 apply 函数会直接覆盖前面内存数据，从一定程度上提高很多效率。


#### 4.2 SharedPreferences 使用建议

> - 在工作线程中写入 sp 时，直接调用 commit 就可以，不必调用 apply,这种情况下，commit 开销更小
> - 在主线程中写入 sp 时，不要调用 commit，要调用 apply
> - sp 对应的文件尽量不要太大，按照模块创建不同的 sp 文件，而不是一个整个应用都读写一个  sp 文件。适当地拆分文件, 可以减少同步锁竞争，并提高写入效率。
> - sp 适合读写轻量的、小的配置信息，不适合保存大数据量的信息，比如大的 json 字符串，有助于减少卡顿/anr
> - 使用 SharedPreferences 是最好不要一上来就执行 getSharedPreferences().edit()，将 getSharedPreferences() 和 edit 过程拆分，从源码分析中可以看到，edit 会等待文件加载完成，此时会造成阻塞，容易导致卡顿等问题出现
> - 当有连续的调用 putXX 方法操作时（特别是循环中），当确认不需要立即读取时，最后一次调用 commit 或 apply 即可
> - 不要使用MODE_MULTI_PROCESS

#### 4.3 SharedPreferences 其他优化

在 8.0 以下没优化之前，如果防止等待任务导致的卡顿和 ANR 问题呢？要想解决的话，可以尝试清理锁队列，风险就是可能导致数据存储失败。

具体操作：

Activity 的 onStop，以及 Service 的 onStop 和 onStartCommand 都是通过 ActivityThread 触发的，ActivityThread 中有一个 Handler 变量，我们通过 Hook 拿到此变量，给此 Handler 设置一个 callback，Handler 的 dispatchMessage 中会先处理 callback。

```java
try {
    Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
    Method currentAtyThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
    Object activityThread = currentAtyThreadMethod.invoke(null);

    Field mHField = activityThreadClass.getDeclaredField("mH");
    mHField.setAccessible(true);
    Handler handler = (Handler) mHField.get(activityThread);

    Field mCallbackField = Handler.class.getDeclaredField("mCallback");
    mCallbackField.setAccessible(true);
    mCallbackField.set(handler,new SpCompatCallback());
    Log.d(TAG,"hook success");
} catch (ClassNotFoundException e) {
    e.printStackTrace();
} catch (NoSuchFieldException e) {
    e.printStackTrace();
} catch (NoSuchMethodException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
} catch (InvocationTargetException e) {
    e.printStackTrace();
} catch (Throwable e){
    e.printStackTrace();
}

```

自定义callbak:SpCompatCallback,在这个方法中做清理等待锁列表的操作：

```java
public class SpCompatCallback implements Handler.Callback {


    public SpCompatCallback(){
    }

    //handleServiceArgs
    private static final int SERVICE_ARGS = 115;
    //handleStopService
    private static final int STOP_SERVICE = 116;
    //handleSleeping
    private static final int SLEEPING = 137;
    //handleStopActivity
    private static final int STOP_ACTIVITY_SHOW = 103;
    //handleStopActivity
    private static final int STOP_ACTIVITY_HIDE = 104;
    //handlePauseActivity
    private static final int PAUSE_ACTIVITY = 101;
    //handlePauseActivity
    private static final int PAUSE_ACTIVITY_FINISHING = 102;

    @Override
    public boolean handleMessage(Message msg) {
        switch (msg.what){
            case SERVICE_ARGS:
                SpHelper.beforeSpBlock("SERVICE_ARGS");
                break;
            case STOP_SERVICE:
                SpHelper.beforeSpBlock("STOP_SERVICE");
                break;
            case SLEEPING:
                SpHelper.beforeSpBlock("SLEEPING");
                break;
            case STOP_ACTIVITY_SHOW:
                SpHelper.beforeSpBlock("STOP_ACTIVITY_SHOW");
                break;
            case STOP_ACTIVITY_HIDE:
                SpHelper.beforeSpBlock("STOP_ACTIVITY_HIDE");
                break;
            case PAUSE_ACTIVITY:
                SpHelper.beforeSpBlock("PAUSE_ACTIVITY");
                break;
            case PAUSE_ACTIVITY_FINISHING:
                SpHelper.beforeSpBlock("PAUSE_ACTIVITY_FINISHING");
                break;
            default:
                break;
        }
        return false;
    }
}
```
清理操作实际上是反射调用  sPendingWorkFinishers.clear();

```java
public class SpHelper {
    private static final String TAG = "SpHelper";
    private static boolean init = false;
    private static String CLASS_QUEUED_WORK = "android.app.QueuedWork";
    private static String FIELD_PENDING_FINISHERS = "sPendingWorkFinishers";
    private static ConcurrentLinkedQueue<Runnable> sPendingWorkFinishers = null;

    public static void beforeSpBlock(String tag){
        if(!init){
            getPendingWorkFinishers();
            init = true;
        }
        Log.d(TAG,"beforeSpBlock "+tag);
        if(sPendingWorkFinishers != null){
            sPendingWorkFinishers.clear();
        }
    }

    private static void getPendingWorkFinishers() {
        Log.d(TAG,"getPendingWorkFinishers");
        try {
            Class clazz = Class.forName(CLASS_QUEUED_WORK);
            Field field = clazz.getDeclaredField(FIELD_PENDING_FINISHERS);
            field.setAccessible(true);
            sPendingWorkFinishers = (ConcurrentLinkedQueue<Runnable>) field.get(null);
            Log.d(TAG,"getPendingWorkFinishers success");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (Throwable e){
            e.printStackTrace();
        }

    }
}
```

### 5 参考

[全面剖析SharedPreferences](http://gityuan.com/2017/06/18/SharedPreferences/)

[剖析 SharedPreference apply 引起的 ANR 问题](https://mp.weixin.qq.com/s/IFgXvPdiEYDs5cDriApkxQ)

[SharedPreferences ANR问题分析和解决 & Android 8.0的优化](https://www.jianshu.com/p/3f64caa567e5?utm_source=desktop&utm_medium=timeline)
