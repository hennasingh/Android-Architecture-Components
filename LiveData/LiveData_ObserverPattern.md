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
- [Adding Components to Project](https://developer.android.com/topic/libraries/architecture/adding-components)

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

### Adding LiveData to AddTaskActivity

Now we will also make changes in AddTaskActivity and apply LiveData to `loadTaskById()` method when we update a task entry object. Now in TaskDao interface method signature will change like below

**Before**
```java
    @Query("SELECT * FROM task WHERE id = :id")
    TaskEntry loadTaskById(int id);
```
**After**
```java
    @Query("SELECT * FROM task WHERE id = :id")
    LiveData<TaskEntry> loadTaskById(int id);
```
Now when you come back to AddTaskActivity, there is compiler error and we change the method sginature there as well and remove the exxecutor from place since LiveData run out of the main thread by default. We will update the UI in `onChanged()` method of the task observer. But in this case we do not want to receive updates, so we will remove the observer from our LiveData object.

**Before**
```java
     Intent intent = getIntent();
        if (intent != null && intent.hasExtra(EXTRA_TASK_ID)) {
            mButton.setText(R.string.update_button);
            if (mTaskId == DEFAULT_TASK_ID) {
      
                mTaskId = intent.getIntExtra(EXTRA_TASK_ID, DEFAULT_TASK_ID);

                AppExecutors.getInstance().diskIO().execute(new Runnable() {
                    @Override
                    public void run() {
                        final TaskEntry task = mDb.taskDao().loadTaskById(mTaskId);
                        runOnUiThread(new Runnable() {
                            @Override
                            public void run() {
                              populateUI(task);
                            }
                        });
                    }
                });
            }
```
**After**
```java
        Intent intent = getIntent();
        if (intent != null && intent.hasExtra(EXTRA_TASK_ID)) {
            mButton.setText(R.string.update_button);
            if (mTaskId == DEFAULT_TASK_ID) {
                mTaskId = intent.getIntExtra(EXTRA_TASK_ID, DEFAULT_TASK_ID);

                final LiveData<TaskEntry> task = mDb.taskDao().loadTaskById(mTaskId);
                task.observe(this, new Observer<TaskEntry>() {
                    @Override
                    public void onChanged(@Nullable TaskEntry taskEntry) {
                        task.removeObserver(this);
                        populateUI(taskEntry);
                    }
                });
            }
        }
```

Now when we run the app and update a task it works as before but when we press the home button and come back to it, we cans see that we are not activity querying the database again, the screen stays.

![Screen Stays At Update](/images/screenstays_atupdate.PNG)

Usually, we do not need to remove the observer as we want LiveData to reflect the state of the underlying data. In our case we are doing a one-time load, and we don’t want to listen to changes in the database.

Then, why are we using LiveData? Why don’t we keep the executor as it is instead?

When we progress in the lesson we will move this logic to the ViewModel. There we will benefit from the rest of advantages of LiveData, even if we have used it for a one-time load.
   

