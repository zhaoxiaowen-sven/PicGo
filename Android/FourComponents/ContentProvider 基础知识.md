# ContentProvider 基础知识

ContentProvider 用户跨进程访问数据，通常和数据库以及ContentResolver配合使用，可以保证数据的安全性。

## 一、ContentResolver

对于每一个应用程序来说，如果想要访问内容提供器中共享的数据，就一定要借助ContentResolver类，可以通过Context中的getContentResolver()方法获取到该类的实例。 ContentResolver中提供了一系列的方法用于对数据进行CRUD操作，其中insert()方法用于添加数据，update()方法用于更新数据，delete()方法用于删除数据，query()方法用于查询数据。有没有似曾相识的感觉？没错，SQLiteDatabase中也是使用的这几个方法来进行 CRUD 操作的，只不过它们在方法参数上稍微有一些区别。不同于SQLiteDatabase，ContentResolver中的增删改查方法都是不接收表名参数的，而是使用一个 Uri参数代替，这个参数被称为内容 URI，下面会详细介绍它的格式定义。在得到了内容URI字符串之后，我们还需要将它解析成Uri对象才可以作为参数传入。 只需要调用 Uri.parse()方法，就可以将内容URI字符串解析成 Uri对象： 

    Uri uri = Uri.parse("content://com.example.app.provider/table1") 

再调用contentResolver的CURD方法就可完成对应的操作。

    Cursor cursor = context.getContentResolver().query(uri, null, null, null, null);


## 二、内容 URI

无论是通过ContentResolver访问数据，还是实现自定义的ContentProvider都需要内容URI才能够通信。URI给内容提供器中的数据建立了唯一标志符，基本格式如下：

    content://Authority/Path/Id

content:// 固定格式
Authority：权限，用于对不同的应用程序进行区分，一般为了避免冲突，都会采用包名.provider
Path: 路径，用于对同一个程序中的不同表做区分
Id: 数据Id，用于区分表中的不同数据
URI的格式主要有如下2种：

       content://com.example.app.provider/table 
       content://com.example.app.provider/table1/1 

以路径结尾表示期望访问该表中所有的数据， 以 id结尾就表示期望访问该表中拥有相应 id的数据。我们可以使用通配符的方式来分别匹 配这两种格式的内容 URI，规则如下。
1. *：表示匹配任意长度的任意字符 
2. #：表示匹配任意长度的数字 
所以，一个能够匹配任意表的内容 URI格式就可以写成：

    content://com.example.app.provider/* 

而一个能够匹配 table1表中任意一行数据的内容 URI格式就可以写成

    content://com.example.app.provider/table1/#

我们再借助UriMatcher这个类就可以轻松地实现匹配内容URI的功能。UriMatcher中提供了一个addURI()方法，这个方法接收三个参数，可以分别把权限、路径和一个自定义代码传进去。这样，当调用UriMatcher的match()方法时，就可以将一个 Uri对象传入，返回值是某个能够匹配这个Uri对象所对应的自定义代码，利用这个代码，我们就可以判断出调用方期望访问的是哪张表中的数据了。


## 三、ContentProvider方法介绍

通过继承ContentProvider，并且实现抽象方法就可完成自定义的ContentProvider共享数据了。

### 1.onCreate() 
初始化内容提供器的时候调用。通常会在这里完成对数据库的创建和升级等操作，返回true表示内容提供器初始化成功，返回false 则表示失败。注意，只有当存在 ContentResolver尝试访问我们程序中的数据时，内容提供器才会被初始化。

### 2.query() 
从内容提供器中查询数据。使用uri参数来确定查询哪张表，projection参数用于确定查询哪些列，selection和selectionArgs参数用于约束查询哪些行，sortOrder参数用于对结果进行排序，查询的结果存放在Cursor对象中返回。 

### 3.insert() 
向内容提供器中添加一条数据。使用uri参数来确定要添加到的表，待添加的数据保存在values参数中。添加完成后，返回一个用于表示这条新记录的 URI。

### 4.update() 
更新内容提供器中已有的数据。使用uri参数来确定更新哪一张表中的数据，新数据保存在values参数中，selection和selectionArgs参数用于约束更新哪些行，受影响的 行数将作为返回值返回。

### 5.delete()
从内容提供器中删除数据。使用uri参数来确定删除哪一张表中的数据，selection和selectionArgs参数用于约束删除哪些行，被删除的行数将作为返回值返回。

### 6.getType() 
根据传入的内容 URI来返回相应的 MIME类型。它是所有的内容提供器都必须提供的一个方法，用于获取Uri对象所对应的MIME类型。一个内容URI所对应的MIME字符串主要由三部分组分，Android对这三个部分做了如下格式规定。
    1. 必须以vnd开头。
    2. 如果内容URI以路径结尾，则后接android.cursor.dir/，如果内容URI以id结尾，则后接android.cursor.item/ 
    3. 最后接上 vnd.authority.path

所以，对于 content://com.example.app.provider/table1这个内容 URI，它所对应的 MIME 类型就可以写成：

    vnd.android.cursor.dir/vnd.com.example.app.provider.table1 

对于 content://com.example.app.provider/table1/1这个内容 URI，它所对应的 MIME类型 就可以写成：

     vnd.android.cursor.item/vnd.com.example.app.provider.table1

## 四、call 方法

上面的方法都是访问或修改数据库的，如果需要跨进程访问里一个应用其他数据，例如sharePreference数据，可以通过call方法来调用自定义的函数。

### 1.provider中复写call 方法，并且添加自定义的方法

    @Override
    public Bundle call(@NonNull String method, @Nullable String arg, @Nullable Bundle extras) {
        switch (method){
            case "getData":
                return getData();
            default:
                break;
        }
        return null;
    }
    
    //自定义函数
    public Bundle getData(){
        Bundle b = new Bundle();
        b.putString("name","call getData");
        return b;
    }

### 2.查询方传入要调用的provider的自定义方法名和参数

    // uri格式最后一定要带一个 /
    Uri uriCall = Uri.parse("content://" + AUTHORITY + "/");
    Bundle b = getContentResolver().call(uriCall, "getData", null, null);
    Log.i(TAG, ""+b.get("name"));

## 五、MatrixCursor
ContentProvider的Query方法返回的是一个cursor，如果要对cursor中的数据做处理后再返回给查询的一方，可以通过MatrixCursor 对现有数据封装后返回。
    
    Cursor cursor1 = db.query("users", null, "id = ?", new String[]{"1"},null, null, null);
    MatrixCursor m = new MatrixCursor(new String[]{"c1", "c2", "c3"});
    while(cursor1.moveToNext()) {
        String name = cursor1.getString(1);
        int age = cursor1.getInt(2);
        String address = cursor1.getString(3);
        Log.i(TAG, name + "age = "+age + "address = "+address);
        m.addRow(new Object[]{name, age, address});
    }
    return m

Demo：

参考资料：
第一行代码——Android 郭霖
https://stackoverflow.com/questions/17224766/what-is-getcontentresolver-call-and-how-to-use-it