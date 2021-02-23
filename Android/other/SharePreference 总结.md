SharePreferences是一个轻量级的存储类，特别适合用于保存软件配置参数。使用SharedPreferences保存数据，其背后是用xml文件存放数据，文件存放在/data/data/ < package name > /shared_prefs目录下，本文主要介绍下sp的用法并且从源码的角度分析下sp的写入过程。

## 1.获取sp的三种方式

1.this.getPreferences (int mode)
    通过Activity对象获取，获取的是本Activity私有的Preference，保存在系统中的xml形式的文件的名称为这个Activity的名字，因此一个Activity只能有一个，属于这个Activity。

2.context.getSharedPreferences (String name, int mode)
    因为Activity继承了ContextWrapper，因此也是通过Activity对象获取，但是属于整个应用程序，可以有多个，以第一参数的name为文件名保存在系统中。

3.PreferenceManager.getDefaultSharedPreferences(this);
    PreferenceManager的静态函数，保存PreferenceActivity中的设置，属于整个应用程序，但是只有一个，命名为packagename_preferences。
    
## 2. 写入模式
    Activity.MODE_PRIVATE,//默认操作模式，代表该文件是私有数据，只能被应用本身访问，在该模式下，写入的内容会覆盖原文件的内容，如果想把新写入的内容追加到原文件中，可以使用Activity.MODE_APPEND 
    Activity.MODE_APPEND //该模式会检查文件是否存在，存在就往文件追加内容，否则就创建新文件
    以下2种模式已废弃，不安全
    Activity.MODE_WORLD_READABLE,//表示当前文件可以被其他应用读取，  
    Activity.MODE_WORLD_WRITEABLE,//表示当前文件可以被其他应用写入；

## 3.源码分析
### 1.获取SharedPreferences对象
contextImpl.getSharedPreferences()，这里使用到了单例模式，涉及到的几个对象如下：

    ArrayMap<String, File> mSharedPrefsPaths ： 保存sp的路径和文件
    ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache key 是packageName， vlaue是 file 和 对应的spImpl对象。

整个过程分为2步，先获取对应name的sp文件，如果没有就new一个文件放到 mSharedPrefsPaths 中，
再根据文件名获取对应的spImpl对象。

对于一个相同的SharedPreferences name，获取到的都是同一个SharedPreferences对象，也就是SharedPreferencesImpl对象。

    public SharedPreferences getSharedPreferences(String name, int mode) {
        // At least one application in the world actually passes in a null
        // name.  This happened to work because when we generated the file name
        // we would stringify it to "null.xml".  Nice.
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
    
     public SharedPreferences getSharedPreferences(File file, int mode) {
        ...
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
            sp = cache.get(file);
            if (sp == null) {
                sp = new SharedPreferencesImpl(file, mode);
                cache.put(file, sp);
                return sp;
            }
        }
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }
    
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

### 2.sp 的初始化
对于一个SharedPreferences文件name，第一次调用getSharedPreferences时会去创建一个SharedPreferencesImpl对象，它会开启一个子线程，将所有的数据以Map的形式保存在内存中。

     SharedPreferencesImpl(File file, int mode) {
        mFile = file;
        mBackupFile = makeBackupFile(file);
        mMode = mode;
        mLoaded = false;
        mMap = null;
        startLoadFromDisk();
    }
    
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
    
    private void loadFromDisk() {
        synchronized (mLock) {
            if (mLoaded) {
                return;
            }
            if (mBackupFile.exists()) {
                mFile.delete();
                mBackupFile.renameTo(mFile);
            }
        }
    
        // Debugging
        if (mFile.exists() && !mFile.canRead()) {
            Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
        }
    
        Map map = null;
        StructStat stat = null;
        try {
            stat = Os.stat(mFile.getPath());
            if (mFile.canRead()) {
                BufferedInputStream str = null;
                try {
                    str = new BufferedInputStream(
                            new FileInputStream(mFile), 16*1024);
                    map = XmlUtils.readMapXml(str);
                } catch (Exception e) {
                    Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
                } finally {
                    IoUtils.closeQuietly(str);
                }
            }
        } catch (ErrnoException e) {
            /* ignore */
        }
    
        synchronized (mLock) {
            mLoaded = true;
            if (map != null) {
                mMap = map;
                mStatTimestamp = stat.st_mtime;
                mStatSize = stat.st_size;
            } else {
                mMap = new HashMap<>();
            }
            mLock.notifyAll();
        }
    }
## 3. sp的读取
当我们在ui线程种这样调用时：

    SharedPreferences sp = getSharedPreferences("test", Context.MODE_PRIVATE);
    String name = sp.getString("name", null)

调用getString时那个SharedPreferencesImpl构造方法开启的子线程可能还没执行完（比如文件比较大时全部读取会比较久），这时getString当然还不能获取到相应的值，必须阻塞到那个子线程读取完为止，getString方法：

    public String getString(String key, @Nullable String defValue) {
        synchronized (this) {
            awaitLoadedLocked();
            String v = (String)mMap.get(key);
            return v != null ? v : defValue;
        }
    }

显然这个awaitLoadedLocked方法就是用来等this这个锁的，在loadFromDiskLocked方法的最后我们也可以看到它调用了notifyAll方法，这时如果getString之前阻塞了就会被唤醒。那么现在这里有一个问题，我们的getString是写在UI线程中，如果那个getString被阻塞太久了，比如60s，这时就会出现ANR，因此要根据具体情况考虑是否需要把SharedPreferences的读写放在子线程中。这里回答第二个 问题，在UI线程中调用getXXX可能会导致ANR。同时可以回答第三个问题，SharedPreferences只能用来存放少量数据，如果一个SharedPreferences对应的xml文件很大的话，在初始化时会把这个文件的所有数据都加载到内存中，这样就会占用大量的内存，有时我们只是想读取某个xml文件中一个key的value，结果它把整个文件都加载进来了，显然如果必要的话这里需要进行相关优化处理。

## 4.sp的写入
    SharedPreferences.Editor editor = getSharedPreferences("test", Context.MODE_PRIVATE).edit();
    editor.putString("name", "test");
    editor.commit();

首先写一个SharedPreferences文件都是先要调用edit方法获取到一个Editor对象：   
    
    public Editor edit() {
        synchronized (this) {
            awaitLoadedLocked();
        }
        return new EditorImpl();
    }
这个Editor对象是SharedPreferencesImpl的一个内部类：

    public final class EditorImpl implements Editor {
        private final Map<String, Object> mModified = Maps.newHashMap();
        private boolean mClear = false;
        public Editor putString(String key, @Nullable String value) {
            synchronized (this) {
                mModified.put(key, value);
                return this;
            }
        }
        ...
    }
可以看到它有一个Map对象mModified，用来保存修改的数据，也就是你每次put的时候其实是把那个键值对放到这个mModified 中，最后调用apply或者commit才会真正把数据写入文件中。调用commit 和 apply时都会调用到commitToMemory 和 enqueueDiskWrite这2个方法。这里我们以commit 为例先看下整个写入的过程。 
### 4.1 commit   
    public boolean commit() {
        MemoryCommitResult mcr = commitToMemory();
        SharedPreferencesImpl.this.enqueueDiskWrite(
            mcr, null /* sync write on this thread okay */);
        try {
            mcr.writtenToDiskLatch.await();
        } catch (InterruptedException e) {
            return false;
        }
        notifyListeners(mcr);
        return mcr.writeToDiskResult;
    }

### 4.2 commitToMemory
这个方法对应了editor的增删改查方法，这里涉及了2个对象，mMap 和mModified，一个保存当前sp中的键值对，一个保存了修改的键值对。遍历mModified键值对时可以看到这个方法中首先处理了clear标志，它调用的是mMap.clear()，然后再遍历mModified将新的键值对put进mMap，也就是说在一次commit事务中，如果同时put一些键值对和调用clear，那么clear掉的只是之前的键值对，这次put进去的键值对还是会被写入的。

    // Returns true if any changes were made
    private MemoryCommitResult commitToMemory() {
        MemoryCommitResult mcr = new MemoryCommitResult();
        ...
            synchronized (this) {
                if (mClear) {
                    if (!mMap.isEmpty()) {
                        mcr.changesMade = true;
                        mMap.clear();
                    }
                    mClear = false;
                }
    
                for (Map.Entry<String, Object> e : mModified.entrySet()) {
                    String k = e.getKey();
                    Object v = e.getValue();
                    // "this" is the magic value for a removal mutation. In addition,
                    // setting a value to "null" for a given key is specified to be
                    // equivalent to calling remove on that key.
                    if (v == this || v == null) {
                        if (!mMap.containsKey(k)) {
                            continue;
                        }
                        mMap.remove(k);
                    } else {
                        if (mMap.containsKey(k)) {
                            Object existingValue = mMap.get(k);
                            if (existingValue != null && existingValue.equals(v)) {
                                continue;
                            }
                        }
                        mMap.put(k, v);
                    }
    
                    mcr.changesMade = true;
                    if (hasListeners) {
                        mcr.keysModified.add(k);
                    }
                }
    
                mModified.clear();
            }
        }
        return mcr;
    }

遍历mModified时，需要处理一个特殊情况，就是如果一个键值对的value是this（SharedPreferencesImpl）或者是null那么表示将此键值对删除，这个在remove方法中可以看到： 

    public Editor remove(String key) {
        synchronized (this) {
            mModified.put(key, this);
            return this;
        }
    }

## 4.3 enqueueDiskWrite
先定义一个Runnable，注意实现Runnable与继承Thread的区别，Runnable表示一个任务，不一定要在子线程中执行，一般优先考虑使用Runnable。这个Runnable中先调用writeToFile进行写操作，写操作需要先获得mWritingToDiskLock，也就是写锁。然后执行mDiskWritesInFlight–，表示正在等待写的操作减少1。最后判断postWriteRunnable是否为null，调用commit时它为null，而调用apply时它不为null。 
Runnable定义完，就判断这次是commit还是apply，如果是commit，即isFromSyncCommit为true，而且有1个写操作需要执行，那么就调用writeToDiskRunnable.run()，注意这个调用是在当前线程中进行的。如果不是commit，那就是apply，这时调用QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable)，这个QueuedWork类其实很简单，里面有一个SingleThreadExecutor，用于异步执行这个writeToDiskRunnable。 

    private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        final Runnable writeToDiskRunnable = new Runnable() {
                public void run() {
                    synchronized (mWritingToDiskLock) {
                        writeToFile(mcr);
                    }
                    synchronized (SharedPreferencesImpl.this) {
                        mDiskWritesInFlight--;
                    }
                    if (postWriteRunnable != null) {
                        postWriteRunnable.run();
                    }
                }
            };
    
        final boolean isFromSyncCommit = (postWriteRunnable == null);
    
        // Typical #commit() path with fewer allocations, doing a write on
        // the current thread.
        if (isFromSyncCommit) {
            boolean wasEmpty = false;
            synchronized (SharedPreferencesImpl.this) {
                wasEmpty = mDiskWritesInFlight == 1;
            }
            if (wasEmpty) {
                writeToDiskRunnable.run();
                return;
            }
        }
    
        QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);
    }

### 4.4 writeToFile
SharedPreferences在写入时会先把之前的xml文件改成名成一个备份文件mBackupFile，然后再将要写入的数据写到一个新的文件中，如果这个过程执行成功的话，就会把备份文件删除。由此可见每次即使只是添加一个键值对，也会重新写入整个文件的数据，这也说明SharedPreferences只适合保存少量数据，文件太大会有性能问题。


    private void writeToFile(MemoryCommitResult mcr) {
        // Rename the current file so it may be used as a backup during the next read
        if (mFile.exists()) {
            if (!mcr.changesMade) {
                // If the file already exists, but no changes were
                // made to the underlying map, it's wasteful to
                // re-write the file.  Return as if we wrote it
                // out.
                mcr.setDiskWriteResult(true);
                return;
            }
            if (!mBackupFile.exists()) {
                if (!mFile.renameTo(mBackupFile)) {
                    Log.e(TAG, "Couldn't rename file " + mFile
                          + " to backup file " + mBackupFile);
                    mcr.setDiskWriteResult(false);
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
                mcr.setDiskWriteResult(false);
                return;
            }
            XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);
            FileUtils.sync(str);
            str.close();
            ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);
            try {
                final StructStat stat = Os.stat(mFile.getPath());
                synchronized (this) {
                    mStatTimestamp = stat.st_mtime;
                    mStatSize = stat.st_size;
                }
            } catch (ErrnoException e) {
                // Do nothing
            }
            // Writing was successful, delete the backup file if there is one.
            mBackupFile.delete();
            mcr.setDiskWriteResult(true);
            return;
        } catch (XmlPullParserException e) {
            Log.w(TAG, "writeToFile: Got exception:", e);
        } catch (IOException e) {
            Log.w(TAG, "writeToFile: Got exception:", e);
        }
        // Clean up an unsuccessfully written file
        if (mFile.exists()) {
            if (!mFile.delete()) {
                Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
            }
        }
        mcr.setDiskWriteResult(false);
    }

### 5.使用apply导致的anr
其实无节制的使用apply方法也时会造成anr的，在主线程中无节制的使用apply其实也会造成anr，在调用apply时，会将等待写入到文件系统的任务awaitCommit放在QueuedWork的等待完成队列里。

    public void apply() {
        final MemoryCommitResult mcr = commitToMemory();
        final Runnable awaitCommit = new Runnable() {
                public void run() {
                    try {
                        mcr.writtenToDiskLatch.await();
                    } catch (InterruptedException ignored) {
                    }
                }
            };
    
        QueuedWork.add(awaitCommit);
    
        Runnable postWriteRunnable = new Runnable() {
                public void run() {
                    awaitCommit.run();
                    QueuedWork.remove(awaitCommit);
                }
            };
    
        SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);
    
        // Okay to notify the listeners before it's hit disk
        // because the listeners should always get the same
        // SharedPreferences instance back, which has the
        // changes reflected in memory.
        notifyListeners(mcr);
    }

所以如果我们使用SharedPreference的apply方法,虽然该方法可以很快返回， 并在其它线程里将键值对写入到文件系统， 但是当Activity的onPause等方法被调用时，会调用等待写入到文件系统的任务完成，

    /**
     * Finishes or waits for async operations to complete.
     * (e.g. SharedPreferences$Editor#startCommit writes)
     *
     * Is called from the Activity base class's onPause(), after
     * BroadcastReceiver's onReceive, after Service command handling,
     * etc.  (so async work is never lost)
     */
    public static void waitToFinish() {
        Runnable toFinish;
        while ((toFinish = sPendingWorkFinishers.poll()) != null) {
            toFinish.run();
        }
    }

在执行任务writeToDiskRunnable时，会先等待postrunable执行完成，也就是awaitCommit执行完成，
    
    final Runnable awaitCommit = new Runnable() {
                public void run() {
                    try {
                        mcr.writtenToDiskLatch.await();
                    } catch (InterruptedException ignored) {
                    }
                }
            };
所以如果写入比较慢，主线程就会出现ANR问题。

http://www.cloudchou.com/android/post-988.html

结论：
1.对于一个相同的SharedPreferences name，获取到的都是同一个SharedPreferences对象，它其实是SharedPreferencesImpl对象。
2.在UI线程中调用getXXX可能会导致ANR。
3.SharedPreferences只能用来存放少量数据，如果一个SharedPreferences对应的xml文件很大的话，在初始化时会把这个文件的所有数据都加载到内存中，这样就会占用大量的内存，有时我们只是想读取某个xml文件中一个key的value，结果它把整个文件都加载进来了，显然如果必要的话这里需要进行相关优化处理。
4.commit的写操作是在调用线程中执行的，而apply内部是用一个单线程的线程池实现的，因此写操作是在子线程中执行的。
5.SharedPreferences每次写入都是整个文件重新写入，不是增量写入。
6.apply也会造成anr。   
参考：
http://blog.csdn.net/u012619640/article/details/50940074