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

#### Resources
- [ViewModel Overview](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [Serialization](https://en.wikipedia.org/wiki/Serialization)
- [Parcelable vs Serializable](https://android.jlelse.eu/parcelable-vs-serializable-6a2556d51538)

### Steps to add a ViewModel to MainActivity

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
Similary we will add ViewModel to our update method in **AddTask Activity**. But for updating we need to send task ID parameter to ViewModel, for that we need to create a ViewModel factory.

So we need to create a new class called AddTaskViewModelFactory which will extend from ViewModel provider NewInstanceFactory. This class will have 2 member variables, instance of AppDatabase and Id of the task we want to update and initialize them in class constructor.

We will override the create method to return a new AddTaskViewModel(a new class which we will create similar to MainViewModel class) that uses parameters in its constructor. This is a standard code that you can re-use with minor modifications.

```java
public class AddTaskViewModelFactory extends ViewModelProvider.NewInstanceFactory {

    private final AppDatabase mAppDatabase;
    private final int mTaskId;

    public AddTaskViewModelFactory(AppDatabase appDatabase, int taskId) {
        mAppDatabase = appDatabase;
        mTaskId = taskId;
    }
    // Note: This can be reused with minor modifications
    @Override
    public <T extends ViewModel> T create(Class<T> modelClass) {
        //noinspection unchecked
        return (T) new AddTaskViewModel(mAppDatabase, mTaskId);
    }
}
```
Now we will create AddTaskViewModel class but since we are using ViewModelFactory we will extend this class from ViewModel instead. Similary as in MainViewModel class , this class will have a private member variable for live data and a public getter. We will initialize our task variable in the constructor with a relevant call to our database.
```java
//since we using a factory, we will extend from ViewModel instead
public class AddTaskViewModel extends ViewModel{

    private LiveData<TaskEntry> task;

   // Note: The constructor should receive the database and the taskId
     public AddTaskViewModel(AppDatabase mDb, int taskId) {
        task = mDb.taskDao().loadTaskById(taskId);
    }

    public LiveData<TaskEntry> getTask() {
        return task;
    }
}
```
Now coming back to AddTaskActivity, we do not need live data object in the onCreate method now since the call to load task Id is now done on the ViewModel, so we can remove from the code.
**Before**
```java
        Intent intent = getIntent();
        if (intent != null && intent.hasExtra(EXTRA_TASK_ID)) {
            mButton.setText(R.string.update_button);
            if (mTaskId == DEFAULT_TASK_ID) {
                // populate the UI
                mTaskId = intent.getIntExtra(EXTRA_TASK_ID, DEFAULT_TASK_ID);

                final LiveData<TaskEntry> task = mDb.taskDao().loadTaskById(mTaskId);
                task.observe(this, new Observer<TaskEntry>() {
                    @Override
                    public void onChanged(@Nullable TaskEntry taskEntry) {
                        task.removeObserver(this);
                        Log.d(TAG, "Receiving database update from LiveData");
                        populateUI(taskEntry);
                    }
                });
            }
        }
```
We will create an instance of our factory by passing database and task id to its constructor. ViewModel will be created similar t what we created in MainViewModel with the difference that factory will be added as provider to it. We will use model `getTask()` method to retrieve the live data that we want to observe
**After**
```java
        Intent intent = getIntent();
        if (intent != null && intent.hasExtra(EXTRA_TASK_ID)) {
            mButton.setText(R.string.update_button);
            if (mTaskId == DEFAULT_TASK_ID) {
                // populate the UI
                mTaskId = intent.getIntExtra(EXTRA_TASK_ID, DEFAULT_TASK_ID);

                AddTaskViewModelFactory factory = new AddTaskViewModelFactory(mDb, mTaskId);
  
                final AddTaskViewModel viewModel = ViewModelProviders.of(this, factory).get(AddTaskViewModel.class);
                viewModel.getTask().observe(this, new Observer<TaskEntry>() {
                    @Override
                    public void onChanged(@Nullable TaskEntry taskEntry) {
                        viewModel.getTask().removeObserver(this);
                        Log.d(TAG, "Receiving database update from LiveData");
                        populateUI(taskEntry);
                    }
                });
            }
        }
```
Now , live data object is cast in the ViewModel. So, we can retrieve it again after rotation without needing to requery our database.

## Lifecycle Surprise
Lets discuss the lifecycle of our activities and see how ViewModel takes care of everything

We are in our AddTaskActivity and we click the add button to create a new task, the flow of data is like below
1. Task is saved in our db
2. Our LiveData in our main view model will be updated 
3. That will notify the observer in the main activity via the onChange method
4. Finally our UI will be updated
![Expected Flow](/images/expectedFlow.PNG)

But what happens if we are not in the activity that receives the update ? This is not uncommon and can happen because of various reasons
but we will force it to happen in this case, in our `onSaveButtonClicked()` of the activity we will comment the code `finish()` at the end and run the app. We will see the app stays on the AddTask page and activity is not closed. What will be the flow in this case?

![on deleting Finish code](/images/onFinishAdd.PNG)

As per the expected flow
1. The task should have been saved in the db
2. The LiveData in our main view model should have been updated
3. A call to the onChange method of the observer, that we have in our main activity **but** code in our activity cannot be executed if we are not in the activity *Activities cannot have their code run in the background*
##### If we are still in the AddTask Activity, how does this effect us?

![changed flow](/images/changedFlow.PNG)

If LiveData has tried to call the onChange method of the observer in main activity, that code has not been executed and so the UI in main activity has also not been updated. But if we press the back button, surprisingly *we see the task is added* (this also happens if phone is shut down accidently, because my emulator did) **How is this possible ?**

![On navigating back](/images/onHomeClick.PNG)

This is possible because of another element of Android Architecture components called lifecycle. **How you may ask?** 
We are using LiveData and it already supports lifecycle out of the box. LiveData is a lifecycle aware component, live data is able to know the state of its associated component which in our case is the activity.

If the activity is started or resumed, then its considered active, and only in that case its observers will be notified. When we our in Add Task Activity when the database is updated, then main activity is not active, that means then live data will not notify the observers in main activity. Once we press the back button, main activity becomes active again and live data notifies its observers so the UI is updated.

Another benefit of using live data is it also know when the state of the activity is destroyed, and when that happens it will automatically unsubscribe the observers for us to avoid memory leaks.

So *How did LiveData manage to get these amazing super powers?*

As we mentioned lifecycle is one of the architecture components and lifecycle will allow non-lifecycle objects to be lifecycle aware and that the different Android architecture components will decide to fit together as building blocks.

Lifecycle has two interfaces; Lifecycle Owner and Lifecycle Observers

![Lifecycle interfaces](/images/lifecycle_interfaces.PNG)

**Lifecycle Owners**
They are objects that have a lifecycle like activities and fragments

**Lifecycle Observers**
They can observe Lifecycle Owners, and get notified on lifecycle changes. LiveData is lifecycle aware because it is LifecycleObserver. If you remember when we added live data in the class we called its observe method which took two parameters, we let it know which LifecycleOwner it should observe by passing it as a parameter.

You can also implement your own LifecycleObservers to match your needs



















