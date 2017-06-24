# 浅谈Android Architecture Components

---

[TOC]

## 简介

Google IO 2017发布Android Architecture Components，自己先研究了一下，蛮有意思的，特来记录一下。本文内容主要是参考[官方文档](https://developer.android.com/topic/libraries/architecture/index.html)以及自己的理解，如有错误之处，恳请指出。

## Android Architecture Components

Android Architecture Components是Google发布的一套新的架构组件，使App的架构更加健壮，后面简称AAC。

AAC主要提供了Lifecycle，ViewModel，LiveData，Room等功能，下面依次说明：

* Lifecycle

生命周期管理，把原先Android生命周期的中的代码抽取出来，如将原先需要在onStart()等生命周期中执行的代码分离到Activity或者Fragment之外。

* LiveData

一个数据持有类，持有数据并且这个数据可以被观察被监听，和其他Observer不同的是，它是和Lifecycle是绑定的，在生命周期内使用有效，减少内存泄露和引用问题。

* ViewModel

用于实现架构中的ViewModel，同时是与Lifecycle绑定的，使用者无需担心生命周期。可以在多个Fragment之间共享数据，比如旋转屏幕后Activity会重新create，这时候使用ViewModel还是之前的数据，不需要再次请求网络数据。

* Room

谷歌推出的一个Sqlite ORM库，不过使用起来还不错，使用注解，极大简化数据库的操作，有点类似Retrofit的风格。

AAC的架构是这样的：

![新的架构](https://raw.githubusercontent.com/LiushuiXiaoxia/AndroidArchitectureComponents/master/doc/final-architecture.png)

* Activity/Fragment

UI层，通常是Activity/Fragment等，监听ViewModel，当VIewModel数据更新时刷新UI，监听用户事件反馈到ViewModel，主流的数据驱动界面。

* ViewModel

持有或保存数据，向Repository中获取数据，响应UI层的事件，执行响应的操作，响应数据变化并通知到UI层。

* Repository

App的完全的数据模型，ViewModel交互的对象，提供简单的数据修改和获取的接口，配合好网络层数据的更新与本地持久化数据的更新，同步等

* Data Source

包含本地的数据库等，网络api等，这些基本上和现有的一些MVVM，以及Clean架构的组合比较相似

### Gradle 集成

根目录gradle文件中添加Google Maven Repository

```gradle
allprojects {
    repositories {
        jcenter()
        maven { url 'https://maven.google.com' }
    }
}
```

在模块中添加对应的依赖

如使用Lifecycle,LiveData、ViewModel，添加如下依赖。

```gradle
compile "android.arch.lifecycle:runtime:1.0.0-alpha1"
compile "android.arch.lifecycle:extensions:1.0.0-alpha1"
annotationProcessor "android.arch.lifecycle:compiler:1.0.0-alpha1"
```

如使用Room功能，添加如下依赖。

```gradle
compile "android.arch.persistence.room:runtime:1.0.0-alpha1"
annotationProcessor "android.arch.persistence.room:compiler:1.0.0-alpha1"

// For testing Room migrations, add:
testCompile "android.arch.persistence.room:testing:1.0.0-alpha1"

// For Room RxJava support, add:
compile "android.arch.persistence.room:rxjava2:1.0.0-alpha1"
```

## LifeCycles

Android开发中，经常需要管理生命周期。举个栗子，我们需要获取用户的地址位置，当这个Activity在显示的时候，我们开启定位功能，然后实时获取到定位信息，当页面被销毁的时候，需要关闭定位功能。

下面是简单的示例代码。

```java
class MyLocationListener {
    public MyLocationListener(Context context, Callback callback) {
        // ...
    }

    void start() {
        // connect to system location service
    }

    void stop() {
        // disconnect from system location service
    }
}

class MyActivity extends AppCompatActivity {

    private MyLocationListener myLocationListener;

    public void onCreate(...) {
        myLocationListener = new MyLocationListener(this, (location) -> {
            // update UI
        });
    }

    public void onStart() {
        super.onStart();
        myLocationListener.start();
    }

    public void onStop() {
        super.onStop();
        myLocationListener.stop();
    }
}
```

上面只是一个简单的场景，我们来个复杂一点的场景。当定位功能需要满足一些条件下才开启，那么会变得复杂多了。可能在执行Activity的stop方法时，定位的start方法才刚刚开始执行，比如如下代码，这样生命周期管理就变得很麻烦了。

```java
class MyActivity extends AppCompatActivity {
    private MyLocationListener myLocationListener;

    public void onCreate(...) {
        myLocationListener = new MyLocationListener(this, location -> {
            // update UI
        });
    }

    public void onStart() {
        super.onStart();
        Util.checkUserStatus(result -> {
            // what if this callback is invoked AFTER activity is stopped?
            if (result) {
                myLocationListener.start();
            }
        });
    }

    public void onStop() {
        super.onStop();
        myLocationListener.stop();
    }
}
```

AAC中提供了Lifecycle，用来帮助我们解决这样的问题。LifeCycle使用2个枚举类来解决生命周期管理问题。一个是事件，一个是状态。

事件：

生命周期事件由系统来分发，这些事件对于与Activity和Fragment的生命周期函数。

状态：

Lifecycle的状态，用于追踪中Lifecycle对象，如下图所示。

![Lifecycle的状态](https://raw.githubusercontent.com/LiushuiXiaoxia/AndroidArchitectureComponents/master/doc/lifecycle-states.png)

上面的定位功能代码，使用LifeCycle实现以后是这样的，实现一个`LifecycleObserver`接口，然后用注解标注状态，最后在LifecycleOwner中添加监听。

```java
public class MyObserver implements LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    public void onResume() {
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    public void onPause() {
    }
}

aLifecycleOwner.getLifecycle().addObserver(new MyObserver());
```

上面代码中用到了`aLifecycleOwner`是`LifecycleOwner`接口对象，`LifecycleOwner`是一个只有一个方法的接口`getLifecycle()`，需要由子类来实现。

在Lib中已经有实现好的子类，我们可以直接拿来使用。比如`LifecycleActivity`和`LifecycleFragment`，我们只需要继承此类就行了。

当然实际开发的时候我们会自己定义BaseActivity，Java是单继承模式，那么需要自己实现`LifecycleRegistryOwner`接口。

如下所示即可，代码很近简单

```java
public class BaseFragment extends Fragment implements LifecycleRegistryOwner {

    LifecycleRegistry lifecycleRegistry = new LifecycleRegistry(this);

    @Override
    public LifecycleRegistry getLifecycle() {
        return lifecycleRegistry;
    }
}
```

## LiveData

LiveData 是一个 Data Holder 类，可以持有数据，同时这个数据可以被监听的，当数据改变的时候，可以触发回调。但是又不像普通的`Observable`，LiveData绑定了App的组件，LiveData可以指定在LifeCycle的某个状态被触发。比如LiveData可以指定在LifeCycle的 STARTED 或 RESUMED状体被触发。

```java
public class LocationLiveData extends LiveData<Location> {

    private LocationManager locationManager;

    private SimpleLocationListener listener = new SimpleLocationListener() {
        @Override
        public void onLocationChanged(Location location) {
            setValue(location);
        }
    };

    public LocationLiveData(Context context) {
        locationManager = (LocationManager) context.getSystemService(Context.LOCATION_SERVICE);
    }

    @Override
    protected void onActive() {
        locationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 0, 0, listener);
    }

    @Override
    protected void onInactive() {
        locationManager.removeUpdates(listener);
    }
}
```

* onActive()

这个方法在LiveData在被激活的状态下执行，我们可以开始执行一些操作。

* onActive()

这个方法在LiveData在的失去活性状态下执行，我们可以结束执行一些操作。

* setValue()

执行这个方法的时候，LiveData可以触发它的回调。

`LocationLiveData`可以这样使用。

```java
public class MyFragment extends LifecycleFragment {

    public void onActivityCreated (Bundle savedInstanceState) {
        LiveData<Location> myLocationListener = ...;
        Util.checkUserStatus(result -> {
            if (result) {
                myLocationListener.addObserver(this, location -> {
                    // update UI
                });
            }
        });
    }
}
```

注意，上面的`addObserver`方法，必须传`LifecycleOwner`对象，也就是说添加的对象必须是可以被LifeCycle管理的。

如果LifeCycle没有触发对对应的状态（STARTED or RESUMED），它的值被改变了，那么Observe就不会被执行，

如果LifeCycle被销毁了，那么Observe将自动被删除。

实际上LiveData就提供一种新的供数据共享方式。可以用它在多个Activity、Fragment等其他有生命周期管理的类中实现数据共享。

还是上面的定位例子。

```java
public class LocationLiveData extends LiveData<Location> {

    private static LocationLiveData sInstance;

    private LocationManager locationManager;

    @MainThread
    public static LocationLiveData get(Context context) {
        if (sInstance == null) {
            sInstance = new LocationLiveData(context.getApplicationContext());
        }
        return sInstance;
    }

    private SimpleLocationListener listener = new SimpleLocationListener() {
        @Override
        public void onLocationChanged(Location location) {
            setValue(location);
        }
    };

    private LocationLiveData(Context context) {
        locationManager = (LocationManager) context.getSystemService(Context.LOCATION_SERVICE);
    }

    @Override
    protected void onActive() {
        locationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 0, 0, listener);
    }

    @Override
    protected void onInactive() {
        locationManager.removeUpdates(listener);
    }
}
```

然后在Fragment中调用。

```java
public class MyFragment extends LifecycleFragment {

    public void onActivityCreated (Bundle savedInstanceState) {

        Util.checkUserStatus(result -> {
            if (result) {
                LocationLiveData.get(getActivity()).observe(this, location -> {
                   // update UI
                });
            }
        });
  }
}
```

从上面的示例，可以得到使用LiveData优点：

* 没有内存泄露的风险，全部绑定到对应的生命周期，当LifeCycle被销毁的时候，它们也自动被移除

* 降低Crash，当Activity被销毁的时候，LiveData的Observer自动被删除，然后UI就不会再接受到通知

* 实时数据，因为LiveData是持有真正的数据的，所以当生命周期又重新开始的时候，又可以重新拿到数据

* 正常配置改变，当Activity或者Fragment重新创建的时候，可以从LiveData中获取上一次有用的数据

* 不再需要手动的管理生命周期


### Transformations

有时候需要对一个LiveData做Observer，但是这个LiveData是依赖另外一个LiveData，有点类似于RxJava中的操作符，我们可以这样做。

* Transformations.map()

用于事件流的传递，用于触发下游数据。

```java
LiveData<User> userLiveData = ...;
LiveData<String> userName = Transformations.map(userLiveData, user -> {
    user.name + " " + user.lastName
});
```

* Transformations.switchMap()

这个和map类似，只不过这个是用来触发上游数据。

```java
private LiveData<User> getUser(String id) {
  // ...
}

LiveData<String> userId = ...;
LiveData<User> user = Transformations.switchMap(userId, id -> getUser(id) );
```

## ViewModel

ViewModel是用来存储UI层的数据，以及管理对应的数据，当数据修改的时候，可以马上刷新UI。

Android系统提供控件，比如Activity和Fragment，这些控件都是具有生命周期方法，这些生命周期方法被系统调用。

当这些控件被销毁或者被重建的时候，如果数据保存在这些对象中，那么数据就会丢失。比如在一个界面，保存了一些用户信息，当界面重新创建的时候，就需要重新去获取数据。当然了也可以使用控件自动再带的方法，在`onSaveInstanceState`方法中保存数据，在onCreate中重新获得数据，但这仅仅在数据量比较小的情况下。如果数据量很大，这种方法就不能适用了。

另外一个问题就是，经常需要在Activity中加载数据，这些数据可能是异步的，因为获取数据需要花费很长的时间。那么Activity就需要管理这些数据调用，否则很有可能会产生内存泄露问题。最后需要做很多额外的操作，来保证程序的正常运行。

同时Activity不仅仅只是用来加载数据的，还要加载其他资源，做其他的操作，最后Activity类变大，就是我们常讲的上帝类。也有不少架构是把一些操作放到单独的类中，比如MVP就是这样，创建相同类似于生命周期的函数做代理，这样可以减少Activity的代码量，但是这样就会变得很复杂，同时也难以测试。

AAC中提供ViewModel可以很方便的用来管理数据。我们可以利用它来管理UI组件与数据的绑定关系。ViewModel提供自动绑定的形式，当数据源有更新的时候，可以自动立即的更新UI。

下面是一个简单的代码示例。

```java
public class MyViewModel extends ViewModel {

    private MutableLiveData<List<User>> users;
    public LiveData<List<User>> getUsers() {
        if (users == null) {
            users = new MutableLiveData<List<Users>>();
            loadUsers();
        }
        return users;
    }

    private void loadUsers() {
        // do async operation to fetch users
    }
}
```

```java
public class MyActivity extends AppCompatActivity {

    public void onCreate(Bundle savedInstanceState) {
        MyViewModel model = ViewModelProviders.of(this).get(MyViewModel.class);
        model.getUsers().observe(this, users -> {
            // update UI
        });
    }
}
```

当我们获取ViewModel实例的时候，ViewModel是通过ViewModelProvider保存在LifeCycle中，ViewModel会一直保存在LifeCycle中，直到Activity或者Fragment销毁了，触发LifeCycle被销毁，那么ViewModel也会被销毁的。下面是ViewModel的生命周期图。

![](https://raw.githubusercontent.com/LiushuiXiaoxia/AndroidArchitectureComponents/master/doc/viewmodel-lifecycle.png)

## Room

Room是一个持久化工具，和ormlite、greenDao类似，都是ORM工具。在开发中我们可以利用Room来操作sqlite数据库。

Room主要分为三个部分:

* Database

使用注解申明一个类，注解中包含若干个Entity类，这个Database类主要负责创建数据库以及获取数据对象的。

* Entity

表示每个数据库的总的一个表结构，同样也是使用注解表示，类中的每个字段都对应表中的一列。

* DAO

DAO是 Data Access Object的缩写，表示从从代码中直接访问数据库，屏蔽sql语句。

下面是官方给的结构图。

![](https://raw.githubusercontent.com/LiushuiXiaoxia/AndroidArchitectureComponents/master/doc/room_architecture.png)

代码示例：

```java
// User.java
@Entity
public class User {
    @PrimaryKey
    private int uid;

    @ColumnInfo(name = "first_name")
    private String firstName;

    @ColumnInfo(name = "last_name")
    private String lastName;

    // Getters and setters are ignored for brevity,
    // but they're required for Room to work.
}

// UserDao.java
@Dao
public interface UserDao {
    @Query("SELECT * FROM user")
    List<User> getAll();

    @Query("SELECT * FROM user WHERE uid IN (:userIds)")
    List<User> loadAllByIds(int[] userIds);

    @Query("SELECT * FROM user WHERE first_name LIKE :first AND "
           + "last_name LIKE :last LIMIT 1")
    User findByName(String first, String last);

    @Insert
    void insertAll(User... users);

    @Delete
    void delete(User user);
}

// AppDatabase.java
@Database(entities = {User.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}
```

最后在代码中调用Database对象

```java
AppDatabase db = Room.databaseBuilder(getApplicationContext(), AppDatabase.class, "database-name").build();
```

**注意：** Database最好设计成单利模式，否则对象太多会有性能的影响。


### Entities

```java
@Entity
class User {
    @PrimaryKey
    public int id;

    @ColumnInfo(name = "first_name")
    public String firstName;

    @ColumnInfo(name = "last_name")
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

@Entity用来注解一个实体类，对应数据库一张表。默认情况下，Room为实体中定义的每个成员变量在数据中创建对应的字段，我们可能不想保存到数据库的字段，这时候就要用道@Ignore注解。

**注意：** 为了保存每一个字段，这个字段需要有可以访问的gettter/setter方法或者是public的属性

### Entity的参数 primaryKeys

每个实体必须至少定义1个字段作为主键，即使只有一个成员变量，除了使用@PrimaryKey 将字段标记为主键的方式之外，还可以通过在@Entity注解中指定参数的形式

```java
@Entity(primaryKeys = {"firstName", "lastName"})
class User {
    public String firstName;
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

### Entity的参数 tableName

默认情况下，Room使用类名作为数据库表名。如果你想表都有一个不同的名称，就可以在@Entity中使用tableName参数指定

```java
@Entity(tableName = "users")
class User {
    ...
}
```

和tableName作用类似； @ColumnInfo注解是改变成员变量对应的数据库的字段名称。

### Entity的参数 indices

indices的参数值是@Index的数组，在某些情况写为了加快查询速度我们可以需要加入索引

```java
@Entity(indices = {@Index("name"), @Index("last_name", "address")})
class User {
    @PrimaryKey
    public int id;

    public String firstName;
    public String address;

    @ColumnInfo(name = "last_name")
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

有时，数据库中某些字段或字段组必须是唯一的。通过将@Index的unique 设置为true，可以强制执行此唯一性属性。

下面的代码示例防止表有两行包含FirstName和LastName列值相同的一组：

```java
@Entity(indices = {@Index(value = {"first_name", "last_name"}, unique = true)})
class User {
    @PrimaryKey
    public int id;

    @ColumnInfo(name = "first_name")
    public String firstName;

    @ColumnInfo(name = "last_name")
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

### Entity的参数 foreignKeys

因为SQLite是一个关系型数据库，你可以指定对象之间的关系。尽管大多数ORM库允许实体对象相互引用，但Room明确禁止。实体之间没有对象引用。

不能使用直接关系，所以就要用到foreignKeys（外键）。

```java
@Entity(foreignKeys = @ForeignKey(entity = User.class,
                                  parentColumns = "id",
                                  childColumns = "user_id"))
class Book {
    @PrimaryKey
    public int bookId;

    public String title;

    @ColumnInfo(name = "user_id")
    public int userId;
}
```

外键是非常强大的，因为它允许指定引用实体更新时发生的操作。例如，级联删除，你可以告诉SQLite删除所有书籍的用户如果用户对应的实例是由包括OnDelete =CASCADE在@ForeignKey注释。ON_CONFLICT ： @Insert(onConflict=REPLACE) REMOVE 或者 REPLACE

有时候可能还需要对象嵌套这时候可以用@Embedded注解

```java
class Address {
    public String street;
    public String state;
    public String city;

    @ColumnInfo(name = "post_code")
    public int postCode;
}

@Entity
class User {
    @PrimaryKey
    public int id;

    public String firstName;

    @Embedded
    public Address address;
}
```

### Dao

```java
@Dao
public interface UserDao {
    @Query("SELECT * FROM user")
    List<User> getAll();

    @Query("SELECT * FROM user WHERE uid IN (:userIds)")
    List<User> loadAllByIds(int[] userIds);

    @Query("SELECT * FROM user WHERE first_name LIKE :first AND "
           + "last_name LIKE :last LIMIT 1")
    User findByName(String first, String last);

    @Insert
    void insertAll(User... users);

    @Delete
    void delete(User user);
}
```

数据访问对象Data Access Objects (DAOs)是一种抽象访问数据库的干净的方式。

### DAO的Insert 操作

当创建DAO方法并用@Insert注释它时，生成一个实现时会在一个事务中完成插入所有参数到数据库。

```java
@Dao
public interface MyDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    public void insertUsers(User... users);

    @Insert
    public void insertBothUsers(User user1, User user2);

    @Insert
    public void insertUsersAndFriends(User user, List<User> friends);
}
```

### DAO的Update、Delete操作

与上面类似

```java
@Dao
public interface MyDao {
    @Update
    public void updateUsers(User... users);

    @Delete
    public void deleteUsers(User... users);
}
```

### DAO的Query 操作

一个简单查询示例

```java
@Dao
public interface MyDao {
    @Query("SELECT * FROM user")
    public User[] loadAllUsers();
}
```

稍微复杂的，带参数的查询操作

```
@Dao
public interface MyDao {
    @Query("SELECT * FROM user WHERE age > :minAge")
    public User[] loadAllUsersOlderThan(int minAge);
}
```

也可以带多个参数

```java
@Dao
public interface MyDao {
    @Query("SELECT * FROM user WHERE age BETWEEN :minAge AND :maxAge")
    public User[] loadAllUsersBetweenAges(int minAge, int maxAge);

    @Query("SELECT * FROM user WHERE first_name LIKE :search OR last_name LIKE :search")
    public List<User> findUserWithName(String search);
}
```

### 返回子集

上面示例都是查询一个表中的所有字段，结果用对应的Entity即可，但是如果我只要其中的几个字段，那么该怎么使用呢？

比如上面的User，我只需要firstName和lastName，首先定义一个子集，然后结果改成对应子集即可。

```java
public class NameTuple {
    @ColumnInfo(name="first_name")
    public String firstName;

    @ColumnInfo(name="last_name")
    public String lastName;
}

@Dao
public interface MyDao {
    @Query("SELECT first_name, last_name FROM user")
    public List<NameTuple> loadFullName();
}
```

### 支持集合参数

有个这样一个查询需求，比如要查询某两个地区的所有用户，直接用sql中的in即可，但是如果这个地区是程序指定的，个数不确定，那么改怎么办？

```
@Dao
public interface MyDao {
    @Query("SELECT first_name, last_name FROM user WHERE region IN (:regions)")
    public List<NameTuple> loadUsersFromRegions(List<String> regions);
}
```

### 支持Observable

前面提到了LiveData，可以异步的获取数据，那么我们的Room也是支持异步查询的。

```
@Dao
public interface MyDao {
    @Query("SELECT first_name, last_name FROM user WHERE region IN (:regions)")
    public LiveData<List<User>> loadUsersFromRegionsSync(List<String> regions);
}
```

### 支持RxJava

RxJava是另外一个异步操作库，同样也是支持的。

```java
@Dao
public interface MyDao {
    @Query("SELECT * from user where id = :id LIMIT 1")
    public Flowable<User> loadUserById(int id);
}
```

### 支持Cursor

原始的Android系统查询结果是通过Cursor来获取的，同样也支持。

```java
@Dao
public interface MyDao {
    @Query("SELECT * FROM user WHERE age > :minAge LIMIT 5")
    public Cursor loadRawUsersOlderThan(int minAge);
}
```

### 多表查询

有时候数据库存在范式相关，数据拆到了多个表中，那么就需要关联多个表进行查询，如果结果只是一个表的数据，那么很简单，直接用Entity定义的类型即可。

```java
@Dao
public interface MyDao {
    @Query("SELECT * FROM book "
           + "INNER JOIN loan ON loan.book_id = book.id "
           + "INNER JOIN user ON user.id = loan.user_id "
           + "WHERE user.name LIKE :userName")
   public List<Book> findBooksBorrowedByNameSync(String userName);
}
```

如果结果是部分字段，同上面一样，需要单独定义一个POJO，来接受数据。

```java
public class UserPet {
   public String userName;
   public String petName;
}

@Dao
public interface MyDao {
   @Query("SELECT user.name AS userName, pet.name AS petName "
          + "FROM user, pet "
          + "WHERE user.id = pet.user_id")
   public LiveData<List<UserPet>> loadUserAndPetNames();
}

```

### 类型转换

有时候Java定义的数据类型和数据库中存储的数据类型是不一样的，Room提供类型转换，这样在操作数据库的时候，可以自动转换。

比如在Java中，时间用Date表示，但是在数据库中类型确实long，这样有利于存储。

```java
public class Converters {
    @TypeConverter
    public static Date fromTimestamp(Long value) {
        return value == null ? null : new Date(value);
    }

    @TypeConverter
    public static Long dateToTimestamp(Date date) {
        return date == null ? null : date.getTime();
    }
}
```

定义数据库时候需要指定类型转换，同时定义好Entity和Dao。

```java
@Database(entities = {User.java}, version = 1)
@TypeConverters({Converter.class})
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}

@Entity
public class User {
    ...
    private Date birthday;
}

@Dao
public interface UserDao {
    ...
    @Query("SELECT * FROM user WHERE birthday BETWEEN :from AND :to")
    List<User> findUsersBornBetweenDates(Date from, Date to);
}
```

### 数据库升级

版本迭代中，我们不可避免的会遇到数据库升级问题，Room也为我们提供了数据库升级的处理接口。

```
Room.databaseBuilder(getApplicationContext(), MyDb.class, "database-name").addMigrations(MIGRATION_1_2, MIGRATION_2_3).build();

static final Migration MIGRATION_1_2 = new Migration(1, 2) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        database.execSQL("CREATE TABLE `Fruit` (`id` INTEGER, `name` TEXT, PRIMARY KEY(`id`))");
    }
};

static final Migration MIGRATION_2_3 = new Migration(2, 3) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        database.execSQL("ALTER TABLE Book ADD COLUMN pub_year INTEGER");
    }
};
```

迁移过程结束后，Room将验证架构以确保迁移正确发生。如果Room发现问题，则抛出包含不匹配信息的异常。

**警告：** 如果不提供必要的迁移，Room会重新构建数据库，这意味着将丢失数据库中的所有数据。


### 输出模式

可以在gradle中设置开启输出模式，便于我们调试，查看数据库表情况，以及做数据库迁移。

```java
android {
    ...
    defaultConfig {
        ...
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = ["room.schemaLocation":
                             "$projectDir/schemas".toString()]
            }
        }
    }
}
```

## Sample

这里是官方示例，本人自己参考官方文档和示例[Android Architecture Components samples](https://github.com/googlesamples/android-architecture-components)后，也写出了一个类似的示例项目[XiaoxiaZhihu_AAC](https://github.com/LiushuiXiaoxia/XiaoxiaZhihu_AAC)，还请多多指教。

## 总结

原先IO是在5月底已经结束，本来想尽快参照官方文档和示例，把API撸一遍，然后写个Demo和文章来介绍一下。代码示例早已经写出来了，但6月分工作太忙，然后又出差到北京，最后等到了6月底了，才把这篇文章给写出来了。中间可能有内容以及改变，如果有发现稳重有错误，请及时指出，不理赐教。同时以后做事一定要有始有终，确定的事一定要坚持。

## 相关资料

[Android Architecture Components](https://developer.android.com/topic/libraries/architecture/index.html)

[简单聊聊Android Architecture Componets](http://www.cocoachina.com/android/20170519/19309.html)

[Android Architecture Components samples](https://github.com/googlesamples/android-architecture-components)

[XiaoxiaZhihu_AAC](https://github.com/LiushuiXiaoxia/XiaoxiaZhihu_AAC)

[浅谈Android Architecture Components](https://github.com/LiushuiXiaoxia/AndroidArchitectureComponents)

[Room ORM 数据库框架](https://juejin.im/entry/591d41c70ce463006923f937)