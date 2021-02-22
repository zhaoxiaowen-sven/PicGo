# DiskLruCache原理分析

## 一、概述

diskLruCache，顾名思义，支持Lru算法的磁盘缓存。[github地址](https://github.com/JakeWharton/DiskLruCache)

## 二、构造方法

DiskLruCache构造方法需要传入4个参数，含义如下：

```java
private DiskLruCache(File directory, int appVersion, int valueCount, long maxSize) {
    this.directory = directory;
    this.appVersion = appVersion;
    this.journalFile = new File(directory, JOURNAL_FILE);
    this.journalFileTmp = new File(directory, JOURNAL_FILE_TEMP);
    this.journalFileBackup = new File(directory, JOURNAL_FILE_BACKUP);
    this.valueCount = valueCount;
    this.maxSize = maxSize;
}
```

- directory -- 缓存路径

- appVersion -- 缓存版本

- valueCount -- 一个key可以对应几个值

- maxSize -- 缓存文件大小

- redundantOpCount -- 冗余操作数，避免journal文件过大的限制，冗余操作如果大于2000 && 大于当前所有的缓存文件。

  ```java
  private boolean journalRebuildRequired() {
      final int redundantOpCompactThreshold = 2000;
      return redundantOpCount >= redundantOpCompactThreshold 
          && redundantOpCount >= lruEntries.size();
  }
  ```

- journalFile、journalFileTmp、journalFileBackup --  日志文件、临时日志文件、备份日志文件。

  DiskLruCache创建时，会自动创建一个journal文件，负责缓存数据的维护，journal文件中会保存DiskLrucache版本以及文件的CRUD操作。

## 三、lruEntries

介绍日志文件读取之前，Entry和Editor对象。

Entry对象保存再lruEntries中，lruEntries是一个LinkedHashMap，记录着缓存文件（dirty或clean）的信息，文件名、大小，写入状态等等。

```java
private final LinkedHashMap<String, Entry> lruEntries =
    new LinkedHashMap<String, Entry>(0, 0.75f, true);
```

accessOrder = true，LRU算法是通过LinkedHashMap实现的。看一下Entry对象：

```java
private final class Entry {
    private final String key; //文件名，也是lruEntries的key

    /**
     * Lengths of this entry's files.
     */
    private final long[] lengths; //文件长度 对应clean记录的最后的数字，通常lengths数组的长度都为1；若同一个key若有多条记录，则大于1。

    /**
     * True if this entry has ever been published. 
     */
    private boolean readable; //标记是否可读，只有clean的记录才可读

    /**
     * The ongoing edit or null if this entry is not being edited.
     */
    private Editor currentEditor; //正在写入的记录，也用来区分clean和dirty数据

    /**
     * The sequence number of the most recently committed edit to this entry.
     */
    private long sequenceNumber; //日志提交的次数

	// clean文件名的格式：key+“.”+index
	public File getCleanFile(int i) {
		return new File(directory, key + "." + i);
   	}
    // dirty文件名:key+“.”+index + ".tmp"
	public File getDirtyFile(int i) {
		return new File(directory, key + "." + i + ".tmp");
	}
}
```

## 四、journal文件

journal文件格式：

```java
cat journal
libcore.io.DiskLruCache
1
8960
1

DIRTY bda205e0e3cec1055dc7a5e91960cab5
CLEAN bda205e0e3cec1055dc7a5e91960cab5 5830
READ bda205e0e3cec1055dc7a5e91960cab5
REMOVE bda205e0e3cec1055dc7a5e91960cab5
```

### 4.1 文件头部分

- 第一行是个固定的字符串“libcore.io.DiskLruCache”，标志着我们使用的是DiskLruCache技术。
- 第二行是DiskLruCache的版本号，这个值是恒为1的。
- 第三行是应用程序的版本号，我们在open()方法里传入的版本号是什么这里就会显示什么。
- 第四行是valueCount，一个key可以对应几个值。
- 第五行是一个空行。

### 4.2 内容部分 

  journal 文件的内容部分记录着缓存文件的操作。

- DIRTY表示写入记录，写入成功后面会紧跟一条clean记录，失败会紧跟一条REMOVE记录。也就是说，每一行DIRTY的key，后面都应该有一行对应的CLEAN或者REMOVE的记录，否则这条数据就是“脏”的，会被自动删除掉。

- REMOVE记录除了写入失败的场景，手动调用remove时也会写入一条remove记录。

- CLEAN记录的最后还有一串数字，表示的是文件的大小，单位是字节。

- READ记录，每当我们调用get()方法去读取一条缓存数据时，就会向journal文件中写入一条READ记录。

## 五、获取

DiskLruCache的构造方法是私有的，没有向外暴露，获取DiskLruCache需要通过open 方法获取。

### 5.1 open

```java
public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
    throws IOException {
    if (maxSize <= 0) {
        throw new IllegalArgumentException("maxSize <= 0");
    }
    if (valueCount <= 0) {
        throw new IllegalArgumentException("valueCount <= 0");
    }

    // If a bkp file exists, use it instead.
    File backupFile = new File(directory, JOURNAL_FILE_BACKUP);
    if (backupFile.exists()) {
        File journalFile = new File(directory, JOURNAL_FILE);
        // If journal file also exists just delete backup file.
        if (journalFile.exists()) {
            backupFile.delete();
        } else {
            renameTo(backupFile, journalFile, false);
        }
    }

    // Prefer to pick up where we left off.
    DiskLruCache cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
    if (cache.journalFile.exists()) {
        try {
            cache.readJournal();
            cache.processJournal();
            return cache;
        } catch (IOException journalIsCorrupt) {
            System.out
                .println("DiskLruCache "
                    + directory
                    + " is corrupt: "
                    + journalIsCorrupt.getMessage()
                    + ", removing");
            cache.delete();
        }
    }

    // Create a new empty cache.
    directory.mkdirs();
    cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
    cache.rebuildJournal();
    return cache;
}
```

主要分为3个流程：

10 - 15行，存在journal备份文件时，若journal文件也存在，删除备份文件；否则将备份文件做为journal文件。

23 - 38行，读取journal文件，初始化lruEntries数组，将标记为clean的文件信息（包括文件名，大小等等）加载到内存lruEntries中；将标记为dirty的文件删除。

41 - 41行, journal 文件不存在，重新创建缓存。

### 5.2 readJournal

```java
private void readJournal() throws IOException {
    StrictLineReader reader = new StrictLineReader(new FileInputStream(journalFile), Util.US_ASCII);
    try {
       //  读取文件头
        String magic = reader.readLine();
        String version = reader.readLine();
        String appVersionString = reader.readLine();
        String valueCountString = reader.readLine();
        String blank = reader.readLine();
        if (!MAGIC.equals(magic)
            || !VERSION_1.equals(version)
            || !Integer.toString(appVersion).equals(appVersionString)
            || !Integer.toString(valueCount).equals(valueCountString)
            || !"".equals(blank)) {
            throw new IOException("unexpected journal header: [" + magic + ", " + version + ", "
                + valueCountString + ", " + blank + "]");
        }

        int lineCount = 0;
        while (true) {
            try {
            	// 读取文件内容部分
                readJournalLine(reader.readLine());
                lineCount++;
            } catch (EOFException endOfJournal) {
                break;
            }
        }
        redundantOpCount = lineCount - lruEntries.size();

        // If we ended on a truncated line, rebuild the journal before appending to it.
        if (reader.hasUnterminatedLine()) {
            rebuildJournal();
        } else {
            journalWriter = new BufferedWriter(new OutputStreamWriter(
                new FileOutputStream(journalFile, true), Util.US_ASCII));
        }
    } finally {
        Util.closeQuietly(reader);
    }
```

5 - 17行：读取jorunal文件头部分，如果文件信息和初始化传入的信息不存在，直接抛出异常在open中将当前journal文件删除，重新创建

20 - 28行：读取现有journal文件。

32- 36行：如果行尾返回-1，重新构建下文件；否则将已有信息写入journal文件。

#### 5.2.1 readJournalLine

```java
// journal日志格式
DIRTY bda205e0e3cec1055dc7a5e91960cab5
CLEAN bda205e0e3cec1055dc7a5e91960cab5 5830

private void readJournalLine(String line) throws IOException {
    int firstSpace = line.indexOf(' ');
    if (firstSpace == -1) {
        throw new IOException("unexpected journal line: " + line);
    }

    int keyBegin = firstSpace + 1;
    int secondSpace = line.indexOf(' ', keyBegin);
    final String key;
    // key值为文件名
    if (secondSpace == -1) {
        // 非clean记录，key值为第一个空格到行尾
        key = line.substring(keyBegin);
        // remove 记录处理
        if (firstSpace == REMOVE.length() && line.startsWith(REMOVE)) {
            lruEntries.remove(key);
            // 注意这里会return，也就是说初始化时remove记录不添加到lruEntries中，否则后面会崩掉
            return;
        }
    } else {
        // 对于clean记录，key值，第一个空格到第二个空格间
        key = line.substring(keyBegin, secondSpace);
    }

    Entry entry = lruEntries.get(key);
    if (entry == null) {
        entry = new Entry(key);
        lruEntries.put(key, entry);
    }

    if (secondSpace != -1 && firstSpace == CLEAN.length() && line.startsWith(CLEAN)) {
       //clean记录处理：
       // 1.读取有多少个记录长度
       // 2.标记readable为true 
       // 3.currentEditor 标记为null 
       // 4.设置长度数组
        String[] parts = line.substring(secondSpace + 1).split(" ");
        entry.readable = true;
        entry.currentEditor = null;
        entry.setLengths(parts);
    } else if (secondSpace == -1 && firstSpace == DIRTY.length() && line.startsWith(DIRTY)) {
     //dirty 记录给currentEditor赋值
        entry.currentEditor = new Editor(entry);
    } else if (secondSpace == -1 && firstSpace == READ.length() && line.startsWith(READ)) {
        // This work was already done by calling lruEntries.get().
    } else {
        throw new IOException("unexpected journal line: " + line);
    }
}
```

7 - 18行：解析记录的类型和文件名，并将remove记录从lruEntries中删除

28 - 48行：将clean、dirty等记录加到lruEntries中

#### 5.2.2 processJournal

processJournal会继续对上一步加入到lruEntries中的数据进行处理。

```
private void processJournal() throws IOException {
    deleteIfExists(journalFileTmp);
    for (Iterator<Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
        Entry entry = i.next();
        // 1.计算所有cleanfile的大小
        if (entry.currentEditor == null) {
            for (int t = 0; t < valueCount; t++) {
                size += entry.lengths[t];
            }
        } else {
        // 2.删除所有脏数据
            entry.currentEditor = null;
            for (int t = 0; t < valueCount; t++) {
                deleteIfExists(entry.getCleanFile(t));//?
                deleteIfExists(entry.getDirtyFile(t));
            }
            i.remove();
        }
    }
}
```

6 - 8行：统计所有clean数据的大小，也就是当前缓存数据的大小

12 - 13行：删除孤立的dirty数据，clean和dirty记录是配对出现的，且上一步中clean在dirty数据之后，所以currentEditor此时一定为null。	

### 5.3 rebuildJournal

```java
private synchronized void rebuildJournal() throws IOException {
    if (journalWriter != null) {
        journalWriter.close();
    }

    Writer writer = new BufferedWriter(
        new OutputStreamWriter(new FileOutputStream(journalFileTmp), Util.US_ASCII));
    try {
        writer.write(MAGIC);
        writer.write("\n");
        writer.write(VERSION_1);
        writer.write("\n");
        writer.write(Integer.toString(appVersion));
        writer.write("\n");
        writer.write(Integer.toString(valueCount));
        writer.write("\n");
        writer.write("\n");

        for (Entry entry : lruEntries.values()) {
            if (entry.currentEditor != null) {
                writer.write(DIRTY + ' ' + entry.key + '\n');
            } else {
                writer.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
            }
        }
    } finally {
        writer.close();
    }

    if (journalFile.exists()) {
        renameTo(journalFile, journalFileBackup, true);
    }
    renameTo(journalFileTmp, journalFile, false);
    journalFileBackup.delete();

    journalWriter = new BufferedWriter(
        new OutputStreamWriter(new FileOutputStream(journalFile, true), Util.US_ASCII));
}
```

6 - 17行：创建journal临时文件，写journal文件头信息。

19 - 20行：将lruEntries里的所有信息写入到journal,tmp中。

30 - 34行:  处理journal文件，防止当次写入过程中出现异常，避免journal文件丢失。主要过程：1.将当前的journal重命名为备份文件；2.将临时文件命名为journal文件。3.将信息都写入journal文件。

## 六、写入

一个标准的写入操作如下：

```java
DiskLruCache.Editor editor = null;
Writer bw = null;
key = FileKeyGenerator.hashKeyForDisk(key);
try {
	editor = getDiskCache().edit(key);
	OutputStream os = editor.newOutputStream(0);
	bw = new OutputStreamWriter(os);
	bw.write(value);
	bw.flush();
	editor.commit();
} catch (Exception e) {
	...
} finally {
      CloseUtils.closeIO(bw);
}

private synchronized DiskLruCache getDiskCache() throws IOException {
	if (mDiskLruCache == null) {
		mDiskLruCache = DiskLruCache.open(mDirectory, APP_VERSION, VALUE_COUNT, mFileSize);
	}
	return mDiskLruCache;
}
```

### 6.1 DiskLruCache.edit

获取editor对象，进行写入操作。

```java
private synchronized Editor edit(String key, long expectedSequenceNumber) throws IOException {
    checkNotClosed();
    validateKey(key);
    Entry entry = lruEntries.get(key);
    if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER && (entry == null
        || entry.sequenceNumber != expectedSequenceNumber)) {
        return null; // Snapshot is stale.
    }
    if (entry == null) {
        entry = new Entry(key);
        lruEntries.put(key, entry); // put 到lruEntries 
    } else if (entry.currentEditor != null) { //editor != null，说明正在被写入
        return null; // Another edit is in progress.
    }

    // 走到这里有2种可能，新增一条记录或者更新现有的记录
    Editor editor = new Editor(entry);
    entry.currentEditor = editor;

    // Flush the journal before creating files to prevent file leaks.
    journalWriter.write(DIRTY + ' ' + key + '\n');
    journalWriter.flush();
    return editor;
}
```

第3行：验证key，必须是字母、数字、下划线、横线（-）组成，且长度在1-120之间。

9 - 17行：创建一个entry，如果editor 不为空，说明正在被写入。

20 - 23行：journal中写入一条DIRTY记录，并将editor对象返回。

### 6.2 Editor

Editor对象是DiskLruCache写入数据时提供给调用者进行写入操作的对象。

```java
public final class Editor {
    private final Entry entry; // 
    private final boolean[] written; // 同一个key多个值的可写状态，只有dirty是才是可写的
    private boolean hasErrors;//io过程是否出了错
    private boolean committed;//标记是否提交
}
```

再看一下写入的过程

```java
    OutputStream os = editor.newOutputStream(0);
	bw = new OutputStreamWriter(os);
	bw.write(value);
	bw.flush(); //一定要刷新，否则文件没有大小
	editor.commit();
```

#### 6.2.1 editor.newOutputStream

将dirty文件和FaultHidingOutputStream绑定，向文件中写缓存。

```java
public OutputStream newOutputStream(int index) throws IOException {
    if (index < 0 || index >= valueCount) {
        throw new IllegalArgumentException("Expected index " + index + " to "
            + "be greater than 0 and less than the maximum value count "
            + "of " + valueCount);
    }
    synchronized (DiskLruCache.this) {
        if (entry.currentEditor != this) { // entry和editor是不是匹配的
            throw new IllegalStateException();
        }
        if (!entry.readable) { // dirty记录设置为可读
            written[index] = true;
        }
        File dirtyFile = entry.getDirtyFile(index); //获取dirty文件，就是edit是创建的
        FileOutputStream outputStream;
        try {
            outputStream = new FileOutputStream(dirtyFile);
        } catch (FileNotFoundException e) {
            // Attempt to recreate the cache directory.
            directory.mkdirs();
            try {
                outputStream = new FileOutputStream(dirtyFile);
            } catch (FileNotFoundException e2) {
                // We are unable to recover. Silently eat the writes.
                return NULL_OUTPUT_STREAM;
            }
        }
        return new FaultHidingOutputStream(outputStream);
    }
}
```

28行：editor.newOutputStream创建的是一个FaultHidingOutputStream，如果write过程中出现IOException，hasErrors会被置为true。

```java
private class FaultHidingOutputStream extends FilterOutputStream {
    private FaultHidingOutputStream(OutputStream out) {
        super(out);
    }

    @Override
    public void write(int oneByte) {
        try {
            out.write(oneByte);
        } catch (IOException e) {
            hasErrors = true;
        }
    }

    @Override
    public void write(byte[] buffer, int offset, int length) {
        try {
            out.write(buffer, offset, length);
        } catch (IOException e) {
            hasErrors = true;
        }
    }
	...
}
```

#### 6.2.2 editor.commit

```java
public void commit() throws IOException {
    if (hasErrors) {
        completeEdit(this, false);
        remove(entry.key); // The previous entry is stale.
    } else {
        completeEdit(this, true);
    }
    committed = true;
}
```

写入过程出现异常hasError == true，就删除这条数据，否则就说明写入成功，那么hasError指的是什么呢？回头看下写入的过程，秘密就在editor.newOutputStream(0)

```java
OutputStream os = editor.newOutputStream(0);
bw = new OutputStreamWriter(os);
bw.write(value);
bw.flush();
```

第3行调用write操作其实调用的是editor.newOutputStream(0).write。

#### 6.2.3 completeEdit

```java
private synchronized void completeEdit(Editor editor, boolean success) throws IOException {
    Entry entry = editor.entry;
    if (entry.currentEditor != editor) {
        throw new IllegalStateException();
    }

    // If this edit is creating the entry for the first time, every index must have a value.
    if (success && !entry.readable) {
        for (int i = 0; i < valueCount; i++) {
            if (!editor.written[i]) {
                editor.abort();
                throw new IllegalStateException("Newly created entry didn't create value for index " + i);
            }
            if (!entry.getDirtyFile(i).exists()) {
                editor.abort();
                return;
            }
        }
    }

    for (int i = 0; i < valueCount; i++) {
        File dirty = entry.getDirtyFile(i);
        if (success) {
            if (dirty.exists()) {
                File clean = entry.getCleanFile(i);
                dirty.renameTo(clean);
                long oldLength = entry.lengths[i];
                long newLength = clean.length();
                entry.lengths[i] = newLength;
                size = size - oldLength + newLength;
            }
        } else {
            deleteIfExists(dirty);
        }
    }

    redundantOpCount++;//记录日志文件的长度
    entry.currentEditor = null;
    if (entry.readable | success) {
        entry.readable = true;
        journalWriter.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
        if (success) {
            entry.sequenceNumber = nextSequenceNumber++;
        }
    } else {
        lruEntries.remove(entry.key);
        journalWriter.write(REMOVE + ' ' + entry.key + '\n');
    }
    journalWriter.flush();

    if (size > maxSize || journalRebuildRequired()) {
        executorService.submit(cleanupCallable);
    }
}
```

8 - 19行：首次创建时：1.如果有多个记录，那么每一个记录都必须有值，否则抛出异常；2.写入的文件是否存在，若不存在，则删除。

21 - 35行：将dirty文件重命名为clean文件；更新clean文件的长度和所有文件的大小。

39 - 49行：写入成功，journal中插入一条成功的记录；否则插入remove记录。

51 - 53行:  大小超过阈值或者需要重建时，重新整理journal文件。

## 七、读取

一个标准的读取流程如下：

```java
InputStream inputStream = null;
ByteArrayOutputStream outStream = null;
key = FileKeyGenerator.hashKeyForDisk(key);
try {
    DiskLruCache.Snapshot snapshot = getDiskCache().get(key);
    if (snapshot == null) {
        Log.e("TAG", "Not find entry from disk , or entry.readable = false");
        return null;
    }
    inputStream = snapshot.getInputStream(0);
    outStream = new ByteArrayOutputStream();
    byte[] data = new byte[BUFFER_SIZE];
    int count = -1;
    while ((count = inputStream.read(data, 0, BUFFER_SIZE)) != -1) {
        outStream.write(data, 0, count);
    }
    return new String(outStream.toByteArray());
} catch (Exception e) {
    Log.e(TAG, "readFromDisk", e);
} finally {
    CloseUtils.closeIO(inputStream, outStream);
}
```

### 7.1 DiskLruCache.get

```java
public synchronized Snapshot get(String key) throws IOException {
    checkNotClosed();
    validateKey(key);
    Entry entry = lruEntries.get(key);
    if (entry == null) {
        return null;
    }

    if (!entry.readable) {
        return null;
    }

    // Open all streams eagerly to guarantee that we see a single published
    // snapshot. If we opened streams lazily then the streams could come
    // from different edits.
    InputStream[] ins = new InputStream[valueCount];
    try {
        for (int i = 0; i < valueCount; i++) {
            ins[i] = new FileInputStream(entry.getCleanFile(i));
        }
    } catch (FileNotFoundException e) {
        // A file must have been deleted manually!
        for (int i = 0; i < valueCount; i++) {
            if (ins[i] != null) {
                Util.closeQuietly(ins[i]);
            } else {
                break;
            }
        }
        return null;
    }

    redundantOpCount++;
    journalWriter.append(READ + ' ' + key + '\n');
    if (journalRebuildRequired()) {
        executorService.submit(cleanupCallable);
    }

    return new Snapshot(key, entry.sequenceNumber, ins, entry.lengths);
}
```

get方法会返回一个SnapShot（快照）对象，对应是key的Entry对象。获取过程如下：

1.判断lruEntries是否存在且可读(clean的记录才是可读的)，从entry中获取clean文件的对应的InputStream

2.冗余记录 ++ ，journal中增加一条read记录，判断下是否要整理

3.返回snapShot对象

### 7.2 Snapshot

Snapshot 中保存着对应key的entry对象及其文件的inputStream，用于上层读取，SnapShot返回数据对象有2个api：

- Snapshot.getInputStream返回的是一个inputStream对象

- Snapshot.getString 返回String对象，内部也是调用了getInputStream对象。

```java
public final class Snapshot implements Closeable {
    private final String key;
    private final long sequenceNumber;
    private final InputStream[] ins;
    private final long[] lengths;

    private Snapshot(String key, long sequenceNumber, InputStream[] ins, long[] lengths) {
        this.key = key;
        this.sequenceNumber = sequenceNumber;
        this.ins = ins;
        this.lengths = lengths;
    }

    /**
     * Returns the unbuffered stream with the value for {@code index}.
     */
    public InputStream getInputStream(int index) {
        return ins[index];
    }

    /**
     * Returns the string value for {@code index}.
     */
    public String getString(int index) throws IOException {
        return inputStreamToString(getInputStream(index));
    }

    /**
     * Returns the byte length of the value for {@code index}.
     */
    public long getLength(int index) {
        return lengths[index];
    }

    public void close() {
        for (InputStream in : ins) {
            Util.closeQuietly(in);
        }
    }
}
```

## 八、其他方法

### 8.1 remove方法

```java
public synchronized boolean remove(String key) throws IOException {
    checkNotClosed();
    validateKey(key);
    Entry entry = lruEntries.get(key);
    if (entry == null || entry.currentEditor != null) {
        return false;
    }

    for (int i = 0; i < valueCount; i++) {
        File file = entry.getCleanFile(i);
        if (file.exists() && !file.delete()) {
            throw new IOException("failed to delete " + file);
        }
        size -= entry.lengths[i];
        entry.lengths[i] = 0;
    }

    redundantOpCount++;
    journalWriter.append(REMOVE + ' ' + key + '\n');
    lruEntries.remove(key);

    if (journalRebuildRequired()) {
        executorService.submit(cleanupCallable);
    }

    return true;
}
```

如果实体存在且不在被编辑，就可以直接进行删除，然后写入一条REMOVE记录。

### 8.2 close方法

```java
public synchronized void close() throws IOException {
    if (journalWriter == null) {
        return; // Already closed.
    }
    for (Entry entry : new ArrayList<Entry>(lruEntries.values())) {
        if (entry.currentEditor != null) {
            entry.currentEditor.abort();
        }
    }
    trimToSize();
    journalWriter.close();
    journalWriter = null;
}
```

与open对应，手动关闭缓存。关闭时若还有正在编辑的数据，直接abort掉，最后关闭journalWriter。

trimToSize，size超过阈值后，从lruEntries第一个元素开始删除，直到小于maxSize

```java
private void trimToSize() throws IOException {
    while (size > maxSize) {
        Map.Entry<String, Entry> toEvict = lruEntries.entrySet().iterator().next();
        remove(toEvict.getKey());
   }
}
```

### 8.3 abort方法

```java
public void abort() throws IOException {
    completeEdit(this, false);
}
```

就是存储失败的时候的逻辑，会向journal文件中写入一条remove记录

## 九、其他

### 9.1 多个value的写入

以写入String为例，其他数据的写入都可通过OutputStreamWriter。

```java
editor = getDiskCache().edit(key);
editor.set(0, "abc");
editor.set(1, "abc");
editor.set(2, "abc");
editor.commit();
```

写入后的journal文件格式如下：clean文件尾带 3个数字字符串，缓存中存在3个缓存文件

```java
journal文件

DIRTY c431ca75aeea1753604a07b33e6a329a
CLEAN c431ca75aeea1753604a07b33e6a329a 3 6 6

缓存文件
-rw------- 1 u0_a394 u0_a394_cache    3 2020-05-14 20:45 c431ca75aeea1753604a07b33e6a329a.0
-rw------- 1 u0_a394 u0_a394_cache    6 2020-05-14 20:45 c431ca75aeea1753604a07b33e6a329a.1
-rw------- 1 u0_a394 u0_a394_cache    6 2020-05-14 20:45 c431ca75aeea1753604a07b33e6a329a.2
-rw------- 1 u0_a394 u0_a394_cache 1567 2020-05-14 20:45 journal
```

### 9.2 字符串写入

editor提供了2种写入操作，写入字符流和字符串。

#### 9.2.1 写入字符串

```java
editor.set(0, "abc");

public void set(int index, String value) throws IOException {
    Writer writer = null;
    try {
        writer = new OutputStreamWriter(newOutputStream(index), Util.UTF_8);
        writer.write(value);
    } finally {
        Util.closeQuietly(writer);
    }
}
```

#### 9.2.2 流写入

通过流的形式写入时一定要执行一次flush，否则写入的文件在journal中会没有大小

```java
OutputStream os = editor.newOutputStream(0);
bw = new OutputStreamWriter(os);
bw.write(value);
bw.flush();
```

## 十、总结

DiskLruCache 内部是通过lruEntries保存所有的缓存文件名也就是路径，lruEntries本身是一个LinkedHashMap，Lru机制其实就是通过LinkedHashMap实现的；

缓存的增删改查都会记录到journal文件中，dirty记录表示的是写入的记录，一条有效的缓存会对应一条dirty和一条clean日志，他们是配对出现的；

初始化过程中会读取journal文件，将所有标记为clean和dirty的文件加入lruEntries中；最后计算所有clean文件的大小并删除所有孤立的标记为dirty文件；

记录总大小size在写入记录和删除记录时都会随之发生变化，当达到设定阈值maxSize时，会从头部数据开始删除，直到小于maxSize；

在get、edit、remove时都会检查清理操作，主要过程有2个：1.移除lruEntries多余元素 2.重建journal文件。

## 参考

[Android DiskLruCache源码解析硬盘缓存的绝佳方案](https://blog.csdn.net/lmj623565791/article/details/47251585)

[Android DiskLruCache完全解析，硬盘缓存的最佳方案](http://blog.csdn.net/guolin_blog/article/details/28863651)