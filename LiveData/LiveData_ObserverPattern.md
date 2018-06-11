# LiveData and Observer Pattern

According to the documentation, LiveData is an observable data holder class. LiveData sits between our database and our UI. LiveData is 
able to monitor changes in the database and notify the observers when data changes.

![liveData intro](/images/livedata_intro.png)

Similar to the Singleton Pattern, **Observer Pattern** is one of the most common **design patterns** in software development. The classes 
called Observers subscribe to what we call, the subject. The subject, which in our case is the LiveData object which will keep a list of 
all the Observers that are subsribed to it and notify all of them when there is any relevant change

![observer Pattern](/images/observer_pattern.png)

When our data changes, the set value method on our LiveData object will be called. That will trigger a call to a method in each of the 
Observers. Then the Observers will use to do whatever function they need to do, but in this case to use the updated data to update the UI.

### Resources
- [Live Data](https://developer.android.com/topic/libraries/architecture/livedata)
- [Observer Pattern](https://en.wikipedia.org/wiki/Observer_pattern)

## Adding LiveData

To add LiveData and ViewModel, we would need to add relevant dependencies and annotation processors.
```java
    implementation "android.arch.lifecycle:extensions:1.1.0"
    annotationProcessor "android.arch.lifecycle:compiler:1.1.0"
```
Now lets open our TaskDao interface. The `loadAllTasks()` method returns a list of taskEntry objects. In our current app, this method is called
everytime we need to check if there is any change in the database. It will be more efficient if we get notified when there is a change in 
the database which is achieved by returning the object with LiveData

**Before**                                                   
```java                                                     
@Query("SELECT * FROM task ORDER BY priority")           
List<TaskEntry> loadAllTasks();             
```                                                        
**After**
```java
@Query("SELECT * FROM task ORDER BY priority")           
LiveData<List<TaskEntry>> loadAllTasks(); 
```
We will need to make a similar change in our `retrieveTasks()` method which makes this method call. LiveData simplifies the code as promised.

The first simplification comes from the fact that LiveData will run by default outside of the main thread. Because of that we can avoid 
using the executor.

**Before**
```java
    private void retrieveTasks() {
        AppExecutors.getInstance().diskIO().execute(new Runnable() {
            @Override
            public void run() {
               Log.d(TAG, "Actively retrieving the tasks from the DataBase");
               final List<TaskEntry> tasks = mDb.taskDao().loadAllTasks();
                 runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        mAdapter.setTasks(tasks);
                    }
                });
            }
        });
    }
```
We also will not need `runOnUI()` method for updating the UI. Our task variable is of type LiveData, so we can call its `observe` method.
This method requires 2 parameters, a lifecycle owner(something that has a lifecycle) and an observer. Lifecycle owner here will be our 
activity referred as this in code and we will create a new observer. 

This is an interface where we need to implement the onChanged method.The `onChanged()` method will take one parameter which is list
of task entry objects which we are wrapping inside our LiveData object. This `onChanged` method can access the views  so we can use it for 
the logic that we currently have on `runOnUI` method.

To sum up, we are calling the LiveData object and call its observe method. This happens out of the main thread by default. We have our logic
to update the UI in onChange method of the observer which runs on the main thread by default.

**After**
```java
    private void retrieveTasks() {
             
    Log.d(TAG, "Actively retrieving the tasks from the DataBase");

    final LiveData<List<TaskEntry>> tasks = mDb.taskDao().loadAllTasks();

    tasks.observe(this, new Observer<List<TaskEntry>>() {
    @Override
    public void onChanged(@Nullable List<TaskEntry> taskEntries) {
       mAdapter.setTasks(taskEntries);
     }
   }); 
 }
```
We make changes only to Reading Database because we want LiveData to observe changes in the database. For other operations such as INSERT,
UPDATE or DELETE , we do not need to observe changes in the database. For them we will not use LiveData and we will keep executors as is.

Every change in the database will trigger the onChanged method of the observer so we wont need to call retrieveTask method after
deleting a task. We can also move retrieveTask method call to onCreate from onResume method and delete the onResume() method from activity.
   
   

