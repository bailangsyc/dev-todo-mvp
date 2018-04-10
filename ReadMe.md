官方mvp架构学习
[Android MVP 架构一 View与Presenter](https://blog.csdn.net/qiyei2009/article/details/68943089)这篇博客介绍的不错。

## 目录结构

```bash
.
├── addedittask 添加界面
│   ├── AddEditTaskActivity.java
│   ├── AddEditTaskContract.java
│   ├── AddEditTaskFragment.java
│   └── AddEditTaskPresenter.java
├── BasePresenter.java
├── BaseView.java
├── data
│   ├── source
│   │   ├── local
│   │   ├── remote
│   │   ├── TasksDataSource.java
│   │   └── TasksRepository.java
│   └── Task.java
├── statistics  //详情界面
│   ├── StatisticsActivity.java
│   ├── StatisticsContract.java
│   ├── StatisticsFragment.java
│   └── StatisticsPresenter.java
├── taskdetail  //详情界面
│   ├── TaskDetailActivity.java
│   ├── TaskDetailContract.java
│   ├── TaskDetailFragment.java
│   └── TaskDetailPresenter.java
├── tasks  //列表界面
│   ├── ScrollChildSwipeRefreshLayout.java
│   ├── TasksActivity.java
│   ├── TasksContract.java
│   ├── TasksFilterType.java
│   ├── TasksFragment.java
│   └── TasksPresenter.java
└── util  //工具类
    ├── ActivityUtils.java
    ├── AppExecutors.java
    ├── DiskIOThreadExecutor.java
    ├── EspressoIdlingResource.java
    └── SimpleCountingIdlingResource.java
```


androidTest（UI层测试）、androidTestMock（UI层测试mock数据支持）
test（业务层单元测试）、mock（业务层单元测试mock数据支持）

##  TasksActivity
1. TasksActivity的oncreat（）中执行的操作：
根据布局中的id获取到一个 tasksFragment 对象，
```java
tasksFragment =
 (TasksFragment) getSupportFragmentManager().findFragmentById(R.id.contentFrame);
```

然后将该Fragment添加到activity中
```java
 ActivityUtils.addFragmentToActivity(
                    getSupportFragmentManager(), tasksFragment, R.id.contentFrame);
```

同时在onCreat（）中创建了一个Presenter对象
```java
  mTasksPresenter = new TasksPresenter(
                Injection.provideTasksRepository(getApplicationContext()), tasksFragment);
```

可以看到通过该构造，mTasksPresenter持有了数据源与view（即tasksFragment）

## TasksPresenter
主要负责联系view层与model层
TasksPresenter通过构造持有了数据源tasksRepository和显示UI的tasksView



## TasksFragment

在TasksPresenter的构造中，调用如下代码后，tasksFragment就持有了TasksPresenter
```java
mTasksView.setPresenter(this);
```

onCreateView()中的逻辑
给添加任务的图标`fab`添加监听，
```java
    fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mPresenter.addNewTask();
            }
        });
```
点击添加任务的图标，mPresenter调用addNewTask()，添加任务

在mPresenter调用addNewTask（）中，tasksFragment调用showAddTask()，
```java
 public void addNewTask() {
        mTasksView.showAddTask();
    }
```

tasksFragment的showAddTask()中跳转到添加任务的页面 AddEditTaskActivity

---

## AddEditTaskActivity (添加任务的页面)

AddEditTaskActivity和TasksActivity逻辑类似，首先根据xml布局中的id来获取一个addEditTaskFragment（view层）实例，
然后将它添加到AddEditTaskActivity中。然后new一个 mAddEditTaskPresenter,通过构造方法，presenter持有了
数据源与view层的对象 addEditTaskFragment。

## AddEditTaskPresenter
通过构造方法，presenter持有了数据源与view层的对象addEditTaskFragment。同时在构造中将mIsDataMissing设置为true，表示远程数据丢失，
从本地数据库中取出数据。

构造中，调用如下代码,view 层（addEditTaskFragment）就持有了AddEditTaskPresenter
```java
mAddTaskView.setPresenter(this);
```

## AddEditTaskFragment
在AddEditTaskPresenter构造中调用setPresenter（this）持有了AddEditTaskPresenter。

在onActivityCreated(Bundle savedInstanceState)中设置添加任务的监听，
```java
   fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mPresenter.saveTask(mTitle.getText().toString(), mDescription.getText().toString());
            }
        });
```


 AddEditTaskPresenter中调用 saveTask（）,如果是创建任务的话就调用createTask，如果是更新任务的话就updateTask
 ```java
 if (isNewTask()) {
             createTask(title, description);
         } else {
             updateTask(title, description);
         }
 ```

 在creatTask中AddEditTaskFragment调用showTasksList（）来保存任务或者showEmptyTaskError显示空任务的错误。
 ```java

    private void createTask(String title, String description) {
        Task newTask = new Task(title, description);
        if (newTask.isEmpty()) {
            mAddTaskView.showEmptyTaskError();
        } else {
            mTasksRepository.saveTask(newTask);
            mAddTaskView.showTasksList();
        }
    }
 ```

 AddEditTaskFragment的 showTasksList中关闭页面,此时TasksFragment重新获取焦点，
 在TasksFragment的onResume中调用
 ```java
  mPresenter.start();
 ```

在taskPresenter中调用  loadTasks(false);
```java
   public void start() {
        loadTasks(false);
    }
```

判断是否是第一次加载，如果是第一次加载则请求网络，如果不是第一次加载，则从本地取出数据
```java
    public void loadTasks(boolean forceUpdate) {
        // Simplification for sample: a network reload will be forced on first load.
        loadTasks(forceUpdate || mFirstLoad, true);
        mFirstLoad = false;
    }

 ```

 然后在loadTasks(boolean forceUpdate, final boolean showLoadingUI)中执行了哪些操作呢
 首先显示加载的动画，然后根据需要刷新数据库中的task，接下来就是重点， 通过mTasksRepository提供的
 接口getTasks(@NonNull final LoadTasksCallback callback)采用回调的方式来获取task列表。
这里就是presenter层与mode层进行交互

 ```java
在taskPresenter中调用
   mTasksRepository.getTasks(new TasksDataSource.LoadTasksCallback() {
             @Override
             public void onTasksLoaded(List<Task> tasks) {
                ......
                 }
                 // The view may not be able to handle UI updates anymore
                 if (!mTasksView.isActive()) {
                     return;
                 }
                 if (showLoadingUI) {
                     mTasksView.setLoadingIndicator(false);
                 }
             }

 ```

  在  onTasksLoaded(List<Task> tasks)的回调方法中获取到任务列表，然后根据不同情况（所有，未完成，
  已完成）通过mTasksView 显示不同的列表
  ```java
      private void processTasks(List<Task> tasks) {
          if (tasks.isEmpty()) {
              // Show a message indicating there are no tasks for that filter type.
              processEmptyTasks();
          } else {
              // Show the list of tasks
              mTasksView.showTasks(tasks);
              // Set the filter label's text.
              showFilterLabel();
          }
      }

  ```


  --------

 ## 接下来分析mode层的相关操作
分析 TasksRepository 中的方法 getTasks(@NonNull final LoadTasksCallback callback)

// todo 以下待整理  如何模拟从远程获取数据，和如何从数据库中取出数据，这里有待明天研究还有单元测试的问题
这个方法中的逻辑如下：
这里的逻辑还是有待梳理

首先判断本地的缓存数据是否有效，如果有效的话直接取出
```java
      if (mCachedTasks != null && !mCacheIsDirty) {
            callback.onTasksLoaded(new ArrayList<>(mCachedTasks.values()));
            return;
        }
```
如果没有效，那么获取远程数据
如果有效的话，从本地数据库中获取
```java

if (mCacheIsDirty) {
            //如果本地数据是无效的那么请求远程数据
            // If the cache is dirty we need to fetch new data from the network.
            getTasksFromRemoteDataSource(callback);
        } else {
            // Query the local storage if available. If not, query the network.
            //从本地存储的数据库中查询
            mTasksLocalDataSource.getTasks(new LoadTasksCallback()
```
