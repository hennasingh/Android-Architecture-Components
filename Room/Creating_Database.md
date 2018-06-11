# Introduction to Room
Room is an ORM - object relational mapping library. Room will map our database objects to Java objects. Using Room we will wrote less boilerplate and benefit from SQL validation at compile time.

![room features](http://github.com/hennasingh/Android-Architecture-Components/tree/master/images/room_features.png)

Room uses annotations and has 3 main components
1. @Entity - to define our database tables
2. @DAO(data access object) - to provide an API for reading and writing data
3. @Database - which represents a database holder

This will include a list of entities and DAO and will allow us to create a new database or to acquire a connection to our db at runtime

## Creating an Entity

Adding Room dependencies to gradle file
```java
implementation 'android.arch.persistence.room:runtime:1.0.0'
annotationProcessor 'android.arch.persistence.room:compiler:1.0.0'
```

### Plain Old Java Object

We will create a POJO class, here its called `TaskEntry.java`. The class has member variables, getters and setters, and constructors. The task here is to turn this class into an entity. This is done by adding @Entity annotation to our class, this way room will generate a table called task entry but we do not need to have java classes and database tables with the same name.

The way this is handled in Room is to optionally assign a specific table name to our entity. Having a look at the member variables- Id, description, priority, updatedAt; each of them will match a column in the associated table for this entity.
Sometimes we may also like to add extra fields to our class that are not part of the table. When that happens we need to let Room know by using the `ignore` annotation. One more requirement for Room is to define a primary key which will be id of task.
With this Id field will be unique, its preferrable to get database to do it for us, so we set autoGenrate value to true.

```java
@Entity (tableName = "task")
public class TaskEntry {

    @PrimaryKey (autoGenerate = true)
    private int id;
    private String description;
    private int priority;
    private Date updatedAt;

    @Ignore
    public TaskEntry(String description, int priority, Date updatedAt) {
        this.description = description;
        this.priority = priority;
        this.updatedAt = updatedAt;
    }

    public TaskEntry(int id, String description, int priority, Date updatedAt) {
        this.id = id;
        this.description = description;
        this.priority = priority;
        this.updatedAt = updatedAt;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public int getPriority() {
        return priority;
    }

    public void setPriority(int priority) {
        this.priority = priority;
    }

    public Date getUpdatedAt() {
        return updatedAt;
    }

    public void setUpdatedAt(Date updatedAt) {
        this.updatedAt = updatedAt;
    }
}
```
One of the limitations of room is that it can only use one constructor but we have 2, one with ID and one without it. The first constructor is used when we write to the database because the ID is auto-generated and is not known to us while second is needed when we read from the database, each entry will already have an ID. To let room know it has to use second constructor, we add ignore annotation to the first one.

### Additinal Resources

- [SQL Cheatsheet](https://d17h27t6h515a5.cloudfront.net/topher/2016/September/57ed880e_sql-sqlite-commands-cheat-sheet/sql-sqlite-commands-cheat-sheet.pdf)
- [W3School SQL Tutorial](https://www.w3schools.com/sql/)

## Creating a DAO

We will now create a Dao or data access object for each of our entities. This is our Task DAO

```java
@Dao
public interface TaskDao {

    @Query("SELECT * FROM task ORDER BY priority")
    List<TaskEntry> loadAllTasks();

    @Insert
    void insertTask(TaskEntry taskEntry);

    @Update(onConflict = OnConflictStrategy.REPLACE)
    void updateTask(TaskEntry taskEntry);

    @Delete
    void deleteTask(TaskEntry taskEntry);
}
```
Observations
1. It is an interface that has Dao annotation.
2. We have methods for each of CRUD operations and relevant annotations are added to each of these methods.
3. The first method includes the query annotation to run SQL command that will return a list of task entry objects.

The fact that we can request objects back, is what makes room an object relational mapping library. The rest of the methods take a TaskEntry object as a parameter. The update task method annotated with update is set to replace in case of conflict

##  Creating a Database

We learnt to create database table (entity) and DAO (queries to access the table ) and now we will create a database that used both of them. The standard code to create a database is as follows

```java
public abstract class AppDatabase extends RoomDatabase {

    private static final String LOG_TAG = AppDatabase.class.getSimpleName();
    private static final Object LOCK = new Object();
    private static final String DATABASE_NAME = "todolist";
    private static AppDatabase sInstance;

    public static AppDatabase getInstance(Context context) {
        if (sInstance == null) {
            synchronized (LOCK) {
                Log.d(LOG_TAG, "Creating new database instance");
                sInstance = Room.databaseBuilder(context.getApplicationContext(),
                        AppDatabase.class, AppDatabase.DATABASE_NAME)
                        .build();
            }
        }
        Log.d(LOG_TAG, "Getting the database instance");
        return sInstance;
    }

```

The getInstance method will return an AppDatabase using the Singleton pattern. The Singleton pattern is a software design pattern that restricts the instantiation of a class to one object. This is useful when we want to ensure that only one instance of a given class is created. 

The first call when sInstance is not null will instantiate a new value and being a static variable the value will be retained throughout app lifecycle and subsequent calls will only return sInstance object without creating a new instance.
The database is created if it does not already exists, if it exist then a connection to the database is established.

We annotate the class with database notation. In it we will provide the list of classes that we have annotated as entities for our database, in this case there is only 1 class.

 ```java
 @Database(entities = {TaskEntry.class}, version = 1, exportSchema = false)
public abstract class AppDatabase extends RoomDatabase {
```
The version will change if we update the db, export is not necessary in this case but is mentioned to avoid warnings. Since we had only one entity, we mentioned that.

To add Task DAO we will define an abstract method that will returns it `public abstract TaskDao taskDao();`

But if you try to build your project at this stage, you will find that it does not compile, it seems like Room cannot figure out how to save one of the fields in the database

![room date error](https://github.com/hennasingh/Android-Architecture-Components/tree/master/images/room_errordate.png)

On double clicking the error, we see the field that is causing the problem is updatedAt. Looking at documentation of different data types in SQL helps finding the exact prob and its solution .

- [Data Types](https://www.sqlite.org/datatype3.html)

From the documentation, we know date can be of 3 types 
- Real, 
- Text or 
- Integer
Room needs to match each of our fields to one of these datatypes, Strings and numbers are OK, but Room cannot automatically map more complex extractors like Dates. It is in cases like these when type converters come to play. The error msg also suggests to use typeconverters.
Below is DateConverter class 

```java
public class DateConverter {
    @TypeConverter
    public static Date toDate(Long timestamp) {
        return timestamp == null ? null : new Date(timestamp);
    }

    @TypeConverter
    public static Long toTimestamp(Date date) {
        return date == null ? null : date.getTime();
    }
}
```
Both methods in the class are annotated with TypeConverter
1. The first one receives a timestamp and converts it to a date object. Room will use this method when reading from the database
2. The second one receives a date object and converts it into a timestamp long. Room will use this method when writing into the database

For Room to use this TypeConverter , we will add TypeConverter annotation to our AppDatabase class created above

```java
@Database(entities = {TaskEntry.class}, version = 1, exportSchema = false)
@TypeConverters(DateConverter.class)
public abstract class AppDatabase extends RoomDatabase {
```
Now Room knows how to deal with the database. Now the code compiles without any problems.

### Saving a Task

Accessing the database in the main thread can be time consuming, and could lock the UI and throw an ANR (Application Not Responding) error. To avoid that, Room will, by default, throw an error if you attempt to access the database in the main thread.

In the finished app, we need to implement the database operations to run asynchronously, but we also need to validate that what we have done so far is working, so let’s temporarily enable the option to allow queries in the main thread.

For doing that we will call a method in our AppDatabase class when we create database instance.

```java
 sInstance = Room.databaseBuilder(context.getApplicationContext(),
             AppDatabase.class, AppDatabase.DATABASE_NAME)
             .allowMainThreadQueries()
             .build();
```
#### Steps to save data in the database
1. Open AddTaskActivity and create an instance of AppDatabase class
2. Initialize the variable in Activity onCreate method ` mAppDatabase = AppDatabase.getInstance(this);`
3. The `onSaveButtonClicked()` method will be called add Task is clicked from the UI
4. To insert the data in the database, we would need to get details what user input in the UI.
5. With these details we can call the constructor of TaskEntry class.
6. We will retrieve taskDao from our database instance and call the insertTask method using the task entry object that we created.

```java
public void onSaveButtonClicked() {
       
        String description = mEditText.getText().toString();
        int priority = getPriorityFromViews();
        Date date = new Date();

        TaskEntry taskEntry = new TaskEntry(description,priority,date);
         mAppDatabase.taskDao().insertTask(taskEntry);
         finish();
    }
```

### Reading the List of Tasks
Now that we have saved tasks in our database we would query the database to display all data in the UI

1. Add a AppDatabase variable and initialize it in the `onCreate()` method of the Activity
2. We can query the data in `onCreate()` but it will never be refreshed unless our activity is recreated. The alternate is to query the database in `onResume()` method
3. We call loadAllTasks method from our taskDao and resulting list is passed to the adapter using setTasks method

```java
    @Override
    protected void onResume() {
        super.onResume();
        mAdapter.setTasks(mAppDatabase.taskDao().loadAllTasks());
    }
```

### Threads and Runnables

Now that we know that our implementation works, we will disable queries in the main UI Thread. As we know certain operations can block the UI, we will run these operations asynchronously and in a separate thread.

One way to get off the main thread is to make a runnable in a separate thread, which will look like this

![runnable thread](https://github.com/hennasingh/Android-Architecture-Components/tree/master/images/runnable_thread.png)

We create a new thread that uses a runnable. The runnable will perform our database logic, if its needed it will update the UI and finally we start the thread to execute the logic. 

The issue with this approach is that UI cannot be modified from the newly created thread. But thanks to Architecture Components, we will handle UI in a completely different way. The temporary solution now would be to run on UI thread with a new runnable to update our UI which will look like below

![UI on runnable](https://github.com/hennasingh/Android-Architecture-Components/tree/master/images/runnable_UI.png)

### Executors

We discussed implementing database logic as below, this means we create a new thread every time that we need to use the database. Once the thread is done, it will be garbage collected.

```java
    Thread thread = new Thread(new Runnable(){
        @Override
        public void run() {
            //Database Logic
        }
    });
    thread.start();
```
This process will be repeated many times. We may create [race conditions](https://en.wikipedia.org/wiki/Race_condition) if various threads run on the same time. To avoid all these problems it would be great to have all our database calls in the same thread. That will ensure our calls are done sequentially and will avoid creating thread every time. Here is where [**Executors**](https://developer.android.com/reference/java/util/concurrent/Executor) come into play.

An Executor is an object that executes as submitted runnable tasks. It is normally used instead of explicitly creating threads for each of a set of tasks.

AppExecutors class look like below

```java
public class AppExecutors {

    // For Singleton instantiation
    private static final Object LOCK = new Object();
    private static AppExecutors sInstance;
    private final Executor diskIO;
    private final Executor mainThread;
    private final Executor networkIO;

    private AppExecutors(Executor diskIO, Executor networkIO, Executor mainThread) {
        this.diskIO = diskIO;
        this.networkIO = networkIO;
        this.mainThread = mainThread;
    }

    public static AppExecutors getInstance() {
        if (sInstance == null) {
            synchronized (LOCK) {
                sInstance = new AppExecutors(Executors.newSingleThreadExecutor(),
                        Executors.newFixedThreadPool(3),
                        new MainThreadExecutor());
            }
        }
        return sInstance;
    }

    public Executor diskIO() {
        return diskIO;
    }

    public Executor mainThread() {
        return mainThread;
    }

    public Executor networkIO() {
        return networkIO;
    }

    private static class MainThreadExecutor implements Executor {
        private Handler mainThreadHandler = new Handler(Looper.getMainLooper());

        @Override
        public void execute(@NonNull Runnable command) {
            mainThreadHandler.post(command);
        }
    }

}
```
This class contains 3 executors, but in our ToDO list app we will use only disk IO executor. The class constructor receives the three executors as parameters in this order - diskIO, networkIO, and mainThread. 

The **disk IO** is a single thread executor. This ensures that our database transactions are done in order, so we do not have race conditions.

The **network IO** is a pool of three threads. This allows us to run different network calls simultaneously while controlling the number of threads that we have.

Finally the **main thread** executor uses the main thread executor class which essentially will post runnables using a handle associated with the main looper. When we are in an activity we do not need this main thread executor because we can use the run on UI thread method. When we are in a different class and we do not have the run on UI thread method, we can access the main thread using this last executor. As its not possible to imagine a situation where this can be used, here is an [example](https://github.com/googlesamples/android-architecture-components/blob/b1a194c1ae267258cd002e2e1c102df7180be473/GithubBrowserSample/app/src/main/java/com/android/example/github/repository/NetworkBoundResource.java)

Now we will use diskIO executor to perform our database operations. We used `onSaveButtonClicked()` method to add tasks to our database, we can use the same method to get an instance of our app executors and access our diskIO executor to execute a new runnable and that will contain our database logic.

The newly created will look like this

```java
public void onSaveButtonClicked() {
        String description = mEditText.getText().toString();
        int priority = getPriorityFromViews();
        Date date = new Date();

      //final so that its accessible from inside run method
       final TaskEntry taskEntry = new TaskEntry(description, priority, date);
       AppExecutors.getInstance().diskIO().execute(new Runnable() {
            @Override
            public void run() {
                mDb.taskDao().insertTask(taskEntry);
                finish();
            }
        });
     }
```
Similarly in our mainActivity we will call the executor to load the tasks in our `onResume()` method.
1. We will get App executor instance and then execute a new runnable that will query the database to retrieve a list of task entry objects.
2. We wont be able to pass the list to the adapter from the thread in diskIO executor. We will need to wrap it inside a runOnUI() thread method call 

```java
    @Override
    protected void onResume() {
        super.onResume();

        AppExecutors.getInstance().diskIO().execute(new Runnable() {
            @Override
            public void run() {
                final List<TaskEntry> tasks = mDb.taskDao().loadAllTasks();
                //we will be able to simplify this when we learn about AAC
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        mAdapter.setTasks(mDb.taskDao().loadAllTasks());
                    }
                });
            }
        });
     }
```
### Deleting from Database

For now in the app we have an ItemTouchHelper() attached to our RecyclerView for deleting the task. But on swiping, it deletes the task but row still remains there and we relaunch the app, the data reappears. 

```java
new ItemTouchHelper(new ItemTouchHelper.SimpleCallback(0,
                ItemTouchHelper.LEFT | ItemTouchHelper.RIGHT) {
            @Override
            public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder,
                                  RecyclerView.ViewHolder target) {
                return false;
            }

            // Called when a user swipes left or right on a ViewHolder
            @Override
            public void onSwiped(RecyclerView.ViewHolder viewHolder, int swipeDir) {
                // Here is where you'll implement swipe to delete
               }
        }).attachToRecyclerView(mRecyclerView);
```
The OnSwipe() method of the call back will be called  when the user swipes. This method receives 2 parameters
- viewHolder for that row
- integer that describes the swipe direction. We may want to use swipe direction to implement different actions. But in our case we will delete the task from our database in any case.

#### Steps to delete
1. Get an instance of diskIO executor from the AppExecutors and use it to execute a new runnable
2. We need to find out which task to delete, activity does not hold any references to the list of tasks. The object that holds that know how is our adapter. To access the lists of tasks we will need to add a public getTasks method to our adapter

```java
   public List<TaskEntry> getTasks(){
        return mTaskEntries;
    }
```
using the method above we can retrieve list of task entry objects from our adapter. To get the exact task we need to know which position was swiped. Back in main activity we will get this result by getting position from viewHolder.
3. We can then call the delete method of task DAO and pass in the task entry object with that position. While we deleted the task from the database , we will need to update the UI
4. So instead of duplicating the code in onResume() method we will separate the code in a separate method and call that.

```java
@Override
            public void onSwiped(final RecyclerView.ViewHolder viewHolder, int swipeDir) {
 
                AppExecutors.getInstance().diskIO().execute(new Runnable() {
                    @Override
                    public void run() {
                        int position = viewHolder.getAdapterPosition();
                        List<TaskEntry> tasks = mAdapter.getTasks();
                        mDb.taskDao().deleteTask(tasks.get(position));
                        retrieveTasks();
                    }
                });
```
### Updating a Task

This is similarly implementented like all other tasks but when an item is clicked for update. In our app we have onItemClickListener() which receives item ID as a parameter
















