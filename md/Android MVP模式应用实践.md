# Android MVP 模式应用实践

## 0x00 说在前头
MVP模式从提出到现在已经有很长时间了，不管是个人还是公司，对于MVP模式的实现方式都不尽相同，俨然有“百家争鸣”之势，我们公司也借着MVP模式的兴起，实现了一套MVP模式的框架，对本公司的代码进行了重构，最终达到的效果还是令人满意的，所以在此将我们的实现方式分享出来，供大家参考。  
虽然MVP模式的实现方式不尽相同，但是不管实现方案有多么不同，都脱离不了MVP的本质，下文就会从MVP模式的核心思想到实现进行阐述。  

---
## 0x01 MVP模式简介
* M，V，P分别代表什么  
M即为Model，是模型层，主要解决的是数据。  
V即为View，是展示层，也叫UI层，用于展示数据。  
P即为Presenter，是主持层，负责在Model和View之间，从Model里取出数据，格式化后在View上展示，是View与Model之间的纽带，负责用户交互与业务逻辑。

* 基本结构图  
![mvp结构图]()  
以上就是MVP模式的基本结构图，可以看出，View持有Presenter的引用，Presenter持有View的引用，当然它们之间是接口的形式，Presenter持有Model的引用，从Model获取数据，从这样的一个结构，也能很明白的看出基本的时序：  
![mvp时序图]()  
* 基本约束规则  
 1. Model与View不能直接通信，只能通过Presenter；  
 2. Presenter类似于中间人的角色进行协调和调度；  
 3. Model和View是接口，Presenter持有的是一个Model接口和一个View接口；  
 4. Model和View都应该是被动的，一切都由Presenter来主导；  
 5. View的逻辑应该尽可能的简单，不应该有状态。当事件发生时，调用Presenter来处理，Presenter处理后再调用View的方法来获取。  
---
## 0x02 应用实践  

* Google官方的实现  
Google官方分别写了View和Presenter的基本接口：  
```
public interface BaseView<T> {  
      void setPresenter(T presenter);
}
```
```
public interface BasePresenter {  
      void start();
}
```  
用一个Contract接口将View和Presenter组合在一起：  
```
public interface AddEditTaskContract {  

      interface View extends BaseView<Presenter> {

          void showEmptyTaskError();

          void showTasksList();

          void setTitle(String title);

          void setDescription(String description);

          boolean isActive();
      }

      interface Presenter extends BasePresenter {

          void saveTask(String title, String description);

          void populateTask();

          boolean isDataMissing();
      }
}
```  
将Activity和Fragment当做View来对待，并且把Activity作为容器，让Fragment去实现View接口并且持有Presenter的引用：  
```
public class AddEditTaskFragment extends Fragment implements AddEditTaskContract.View {
    private AddEditTaskContract.Presenter mPresenter;
}
```  
在Presenter中会有一个Model的引用（TasksDataSource）：  
```
public class AddEditTaskPresenter implements AddEditTaskContract.Presenter,
        TasksDataSource.GetTaskCallback {

    @NonNull
    private final TasksDataSource mTasksRepository;

    @NonNull
    private final AddEditTaskContract.View mAddTaskView;
}
```  
把TasksDataSource作为一种数据源的概念，分为Remote和Local：  
```
public class TasksRepository implements TasksDataSource {

    private static TasksRepository INSTANCE = null;

    private final TasksDataSource mTasksRemoteDataSource;

    private final TasksDataSource mTasksLocalDataSource;
}
```  

* 参考后的实现思路  
参考过Google官方的实现之后，我们决定使用Contract（契约类）这种方式，这种方式可以很好地将一对View和Presenter绑定在一起，并且也把Activity和Fragment当做View来处理，但是不采用Activity为容器的做法，也把Model当做一种数据源，按照目前代码的情况，分为NetworkModel和DatabaseModel，让Presenter持有Model的引用。  

* M，V，P分别的实现方式  
 1. M的实现方式  
 把Model当成数据源来对待，这样每对它做一次访问，就发送一个Request请求，相当于创建了一次连接、做了一次调用，所以抽象出来一个Call接口，让Model来管理Call，每次调用通过异步的Callback回调：
 ```
 public interface Request {
 }
 ```
 ```
 public interface Model<T extends Request, R> {

          void call(T request, Callback<T, R> callback);

          void cancelAll();
 }
 ```
 ```
 public interface Callback<T extends Request, R> {

          void onStart(T request);

          void onSuccess(T request, R data);

          void onError(T request, Throwable t);

          void onCancel(T request);
 }
 ```
 ```
 public interface Call {

          void execute();

          void cancel();
 }
 ```
 2. V的实现方式  
 和Google的方式一样，有一个基本的接口BaseView，不过不一样的是，我们把泛型放到了P层：  
 ```
 public interface BaseView {
 }
 ```
 3. P的实现方式  
 我们在P层上的设计和Google略有不同，我们采用的是抽象类，并且加上了View的泛型，并且有一个protected的成员变量mView，更是加到了构造函数的参数里，这样的话对子类就有一个规范性的约束：  
 ```
 public abstract class BasePresenter<T extends BaseView> {

         protected final T mView;

         protected BasePresenter(T view) {
             mView = view;
         }
 }
 ```
 4. Contract的实现方式  
 Contract的实现方式和Google官方的基本一致，不同的是，由于P层设计的关系，里面放的是抽象类而不是接口：
 ```
 public interface BizContract {

          interface View extends BaseView {

              void showLoading()；

              void cancelLoading();

              void showBizInfo(String info);

              void closeView();
          }

          abstract class Presenter extends BasePresenter<View> {

              public Presenter(View view) {
                  super(view);
              }

              public abstract void getBizInfo();
          }
 }
 ```  

---
## 0x03 总结  
我们把所有的页面（Activity）都按照MVP模式进行了改造，改造之后明显的感受到了以下的几点好处：  
* View层很清晰，很“薄”  
改造之前，所有的业务和一些数据操作的代码都放在Activity中，这样就造成Activity特别臃肿（可以想象到），改造之后，Activity只被当做View来对待，只负责展示，这样的话，Activity就变得非常“薄”，所有涉及到的方法就只有“set”和“show”，用来操纵View展示：  
```
public class BizActivity extends Activity implements BizContract.View {

      private BizContract.Presenter mPresenter;

      @Override
      protected void onCreate(Bundle savedInstanceState){
          super.onCreate(savedInstanceState);
          mPresenter = new BizPresenter(this);
          mPresenter.getBizInfo();
      }

      @Override
      public void showLoading(){

      }

      @Override
      public void cancelLoading(){

      }

      @Override
      public void showBizInfo(String info){

      }

      @Override
      public void closeView(){

      }
}
```  
* 代码更容易移植  
这样拆完之后，都是针对接口编程，业务都在Presenter，而不在Android系统的组件里（Activity等），所以只要保持接口不变，View层是可以随意替换、移植的：  
```
public class BizPresenter extends BizContract.Presenter {

      public void BizPresenter(BizContract.View view){
          super(view);
      }

      @Override
      public void getBizInfo(){

      }
}
```  
如上所示，业务都在Presenter，而Presenter持有的也是View的接口，所以View层可以随意的替换。
* 更容易做单元测试  
因为业务都抽到了Presenter，所有的调用都是接口，那么做单元测试会非常方便。

MVP的实现方式各不相同，各家有个家的想法，我们只是给出了其中的一种实现思路，如有雷同，纯属巧合。
