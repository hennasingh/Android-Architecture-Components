# Introduction to ViewModel

Thanks to LiveData we are notified of changes in our database without needing to be continuously re-querying. Unfortunately we 
are still re-querying the database everytime that we rotate our device. 

We know activities are destroyed and re-created on rotation, this means that `onCreate()` will be called again, a new livedata object 
will be created and for that there will be new call to the database.

The usual approach of using `onSaveInstanceState()` is meant for a small amounts of data that can be easily serialized or deserialized
but that does not happen in this case. The alternative is querying the database, but doing it everytime on configuration change is not 
efficient.

It is where **ViewModel** comes in. ViewModel allows data to survive configuration changes such as rotation. As depicted in the diagram 
below the lifecycle of ViewModel starts when the activity is created and lasts until it is finished.

![ViewModel lifecycle](/images/intro_viewModel.PNG)

Because of that we can cache complex data in our ViewModel. When the activity is recreated after rotation, it will use exact same
ViewModel, object where we already have our cache data. But it is common that we make asynchronous calls to retrieve our data. 
Without the ViewModel if the activity is destroyed if the call finishes, we could have memory leaks. Making sure we do not have memory 
leaks is extra overhead.

Instead we make this asynchronous call from the ViewModel, the result will be delivered back to ViewModel, and as it does not matter 
if the activity has been destroyed or not, there will be no memory leaks we need to worry about.

### Resources
- [ViewModel Overview](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [Serialization](https://en.wikipedia.org/wiki/Serialization)
- [Parcelable vs Serializable](https://android.jlelse.eu/parcelable-vs-serializable-6a2556d51538)

## Steps to add a ViewModel

1. Create a new class called MainViewModel or any name and extend from AndroidViewModel
2. You are required to implement a constructor which takes one parameter of type Application. Since this ViewModel is used to cache our
list of task entry objects wrap in a liveData object, we will create it as private variable `private LiveData<List<TaskEntry>> tasks` 
which will have a public getter and initialize this variable in our constructor getting an instance of AppDatabase.

```java
    public MainViewModel(@NonNull Application application) {
        super(application);
        tasks = AppDatabase.getInstance(this.getApplication()).taskDao().loadAllTasks();
    }
```
Now we will use this ViewModel in our MainActivity in `retriveTasks` method. Since we are querying the database in our ViewModel, we do
not need to call the `loadAlltask` method from here. We will remove this and get the ViewModel by calling  ViewModels's providers of
this activity and pass the ViewModel class as a parameter.
Now we can retrieve our livedata object using the getTasks method from the ViewModel.

**Before**
```java
    private void retrieveTasks() {
        LiveData<List<TaskEntry>> tasks = mDb.taskDao().loadAllTasks();
        tasks.observe(this, new Observer<List<TaskEntry>>() {
            @Override
            public void onChanged(@Nullable List<TaskEntry> taskEntries) {
                Log.d(TAG, "Receiving database update from LiveData");
                mAdapter.setTasks(taskEntries);
            }
        });
    }
```
**After**
```java
    private void setUpViewModel() {
        MainViewModel viewModel = ViewModelProviders.of(this).get(MainViewModel.class);
        viewModel.getTasks().observe(this, new Observer<List<TaskEntry>>() {
            @Override
            public void onChanged(@Nullable List<TaskEntry> taskEntries) {
                Log.d(TAG, "Updating lists of tasks from LiveData in ViewModel");
                mAdapter.setTasks(taskEntries);
            }
        });
    }
```











