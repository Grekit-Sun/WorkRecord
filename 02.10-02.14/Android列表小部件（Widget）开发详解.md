### 先看效果图
![image](https://img-blog.csdn.net/20180621192107794?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
这个页面如果是在App内部实现，相信只要有一点Android基础的童鞋都能很轻松写出来。但是如果放到Widget中可能就不是那么简单了。因为Widget并没有运行在我们App的进程中，而是运行在系统的SystemServer进程中。你可能会惊讶，Whf！竟然不在我们App进程中！那么是不是意味着我们也不能像在App中那样操作View控件了？答案确实如此。不过不必过于担心，为了我们能在远程进程中更新界面，Google爸爸专门为我们提供了一个RemoteViews类。从名字上看，可能会觉得RemoteViews就是一个View。但事实并非如此，RemoteViews仅仅表示的是一个View结构。它可以在远程进程中展示和更新界面。今天我们要实现的列表小部件就是基于RemoteVeiw实现的。
那么接下来我们来学习如何实现一个桌面Widget，我们先列出要实现Widget的几个核心步骤：

- widget页面布局
- 小部件配置信息
- 了解AppWidgetProvider
- RemoteViewsFactory实现列表适配
- 点击的事件处理

### 支持控件：
```
A RemoteViews object (and, consequently, an App Widget) can support the following layout classes:
FrameLayout
LinearLayout
RelativeLayout
GridLayout
And the following widget classes:
AnalogClock
Button
Chronometer
ImageButton
ImageView
ProgressBar
TextView
ViewFlipper
ListView
GridView
StackView
AdapterViewFlipper
Descendants of these classes are not supported.
```
除了上述列出的几个View，其它的包括Android原生View和自定义View是都不支持在Widget中运行的。因此基于Widget页面限制我们基本就可以告别炫酷的动画效果了。

### 小部件配置信息
配置信息主要是设定小部件的一些属性，比如宽高、缩放模式、更新时间间隔等。我们需要在res/xml目录下新建widget_provider.xml文件，文件名字可以任意取。文件内容如下（可做参考）：
```
<?xml version="1.0" encoding="utf-8"?>
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:minHeight="180dp"
    android:minWidth="300dp"
    android:previewImage="@drawable/ic_launcher_background"
    android:initialLayout="@layout/layout_widget"
    android:updatePeriodMillis="50000"
    android:resizeMode="horizontal|vertical"
    android:widgetCategory="home_screen"> 
    
</appwidget-provider>
```
针对上述文件中的配置信息来做下介绍。
- **minHeight、minWidth** 定义Widget的最小高度和最小宽度（Widget可以通过拉伸来调整尺寸大小）。
- **previewImage** 定义添加小部件时显示的图标。
- **initialLayout** 定义了小部件使用的布局。
- **updatePeriodMillis**定义小部件自动更新的周期，单位为毫秒。
- **resizeMode** 指定了 widget 的调整尺寸的规则。可取的值有: “horizontal”, “vertical”, “none”。“horizontal"意味着widget可以水平拉伸，“vertical”意味着widget可以竖值拉伸，“none”意味着widget不能拉伸；默认值是"none”。
- widgetCategory 指定了 widget 能显示的地方：能否显示在 home Screen 或 lock screen 或 两者都可以。它的取值包括：“home_screen” 和 “keyguard”。Android 4.2 引入。
最后，需要我们在AndroidManifest中注册AppWidgetProvider时引用该文件，使用如下：
```
<receiver android:name=".widget.ListWidgetProvider">
     ...
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/widget_provider" />
</receiver>
```
### 了解AppWidgetProvider类
我们来简单了解下AppWidgetProvider这个类。Widget的功能均是通过AppWidgetProvider来实现的。我们跟进源码可以发现它是继承自BroadcastReceiver类，也就是一个广播接收者。上面我们提到过RemoteViews是运行在SystemServer进程中的，再结合此处我们应该可以推测小部件的事件应该是通过广播来实现的。像小部件的添加、删除、更新、启用、禁用等均是在AppWidgetProvider中通过接受广播来完成的。看AppWidgetProvider中的几个方法：
- **onUpdate()** 当Widget被添加或者被更新时会调用该方法。上边我们提到通过配置updatePeriodMillis可以定期更新Widget。但是当我们在widget的配置文件中声明了android:configure的时候，添加Widget时则不会调用onUpdate方法。
- **onEnable()** 这个方法会在用户首次添加Widget时调用。
- **onAppWidgetOptionsChanged()** 这个方法会在添加Widget或者改变Widget的大小时候被调用。在这个方法中我们还可以根据Widget的大小来选择性的显示或隐藏某些控件。
- **onDeleted(Context, int[])** 当控件被删除的时候调用该方法
- **onEnabled(Context)** 当第一个Widget被添加的时候调用。如果用户添加了两个这个小部件，那么只有第一个添加时才会调用onEnabled.
- **onDisabled(Context)** 当最后一个Widget实例被移除的时候调用这个方法。在这个方法中我们可以做一些清除工作，例如删掉临时的数据库等。
- **onReceive(Context, Intent)** 当接收到广播的时候会被调用。
上述方法中，我们需要着重关心一下onUpdate()方法和onReceive()方法。因为onUpdate()方法会在Widget被添加时候调用，我们可以在此时为Widget添加一View的些交互事件，例如点击事件。由于本篇我们要实现的是一个列表小部件。因此我们还需要RemoteViewsFactory这个类来适配列表数据。
```
public class ListWidgetProvider extends AppWidgetProvider {

    private static final String TAG = "WIDGET";

    public static final String REFRESH_WIDGET = "com.oitsme.REFRESH_WIDGET";
    public static final String COLLECTION_VIEW_ACTION = "com.oitsme.COLLECTION_VIEW_ACTION";
    public static final String COLLECTION_VIEW_EXTRA = "com.oitsme.COLLECTION_VIEW_EXTRA";
    private static Handler mHandler=new Handler();
    private Runnable runnable=new Runnable() {
        @Override
        public void run() {
            hideLoading(Utils.getContext());
            Toast.makeText(Utils.getContext(), "刷新成功", Toast.LENGTH_SHORT).show();
        }
    };

    @Override
    public void onUpdate(Context context, AppWidgetManager appWidgetManager,
                         int[] appWidgetIds) {

        Log.d(TAG, "ListWidgetProvider onUpdate");
        for (int appWidgetId : appWidgetIds) {
            // 获取AppWidget对应的视图
            RemoteViews remoteViews = new RemoteViews(context.getPackageName(), R.layout.layout_widget);

            // 设置响应 “按钮(bt_refresh)” 的intent
            Intent btIntent = new Intent().setAction(REFRESH_WIDGET);
            PendingIntent btPendingIntent = PendingIntent.getBroadcast(context, 0, btIntent, PendingIntent.FLAG_UPDATE_CURRENT);
            remoteViews.setOnClickPendingIntent(R.id.tv_refresh, btPendingIntent);

            // 设置 “ListView” 的adapter。
            // (01) intent: 对应启动 ListWidgetService(RemoteViewsService) 的intent
            // (02) setRemoteAdapter: 设置 gridview的适配器
            //    通过setRemoteAdapter将ListView和ListWidgetService关联起来，
            //    以达到通过 ListWidgetService 更新 ListView的目的
            Intent serviceIntent = new Intent(context, ListWidgetService.class);
            remoteViews.setRemoteAdapter(R.id.lv_device, serviceIntent);


            // 设置响应 “ListView” 的intent模板
            // 说明：“集合控件(如GridView、ListView、StackView等)”中包含很多子元素，如GridView包含很多格子。
            //     它们不能像普通的按钮一样通过 setOnClickPendingIntent 设置点击事件，必须先通过两步。
            //        (01) 通过 setPendingIntentTemplate 设置 “intent模板”，这是比不可少的！
            //        (02) 然后在处理该“集合控件”的RemoteViewsFactory类的getViewAt()接口中 通过 setOnClickFillInIntent 设置“集合控件的某一项的数据”
            Intent gridIntent = new Intent();

            gridIntent.setAction(COLLECTION_VIEW_ACTION);
            gridIntent.putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, appWidgetId);
            PendingIntent pendingIntent = PendingIntent.getBroadcast(context, 0, gridIntent, PendingIntent.FLAG_UPDATE_CURRENT);
            // 设置intent模板
            remoteViews.setPendingIntentTemplate(R.id.lv_device, pendingIntent);
            // 调用集合管理器对集合进行更新
            appWidgetManager.updateAppWidget(appWidgetId, remoteViews);
        }
        super.onUpdate(context, appWidgetManager, appWidgetIds);
    }


    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        AppWidgetManager appWidgetManager = AppWidgetManager.getInstance(context);
        if (action.equals(COLLECTION_VIEW_ACTION)) {
            // 接受“ListView”的点击事件的广播
            int type = intent.getIntExtra("Type", 0);
            int appWidgetId = intent.getIntExtra(AppWidgetManager.EXTRA_APPWIDGET_ID,
                    AppWidgetManager.INVALID_APPWIDGET_ID);
            int index = intent.getIntExtra(COLLECTION_VIEW_EXTRA, 0);
            switch (type) {
                case 0:
                    Toast.makeText(context, "item" + index, Toast.LENGTH_SHORT).show();
                    break;
                case 1:
                    Toast.makeText(context, "lock"+index, Toast.LENGTH_SHORT).show();
                    break;
                case 2:
                    Toast.makeText(context, "unlock"+index, Toast.LENGTH_SHORT).show();
                    break;
            }
        } else if (action.equals(REFRESH_WIDGET)) {
            // 接受“bt_refresh”的点击事件的广播
            Toast.makeText(context, "刷新...", Toast.LENGTH_SHORT).show();
            final AppWidgetManager mgr = AppWidgetManager.getInstance(context);
            final ComponentName cn = new ComponentName(context,ListWidgetProvider.class);
            ListRemoteViewsFactory.refresh();
            mgr.notifyAppWidgetViewDataChanged(mgr.getAppWidgetIds(cn),R.id.lv_device);
            mHandler.postDelayed(runnable,2000);
            showLoading(context);
        }
        super.onReceive(context, intent);
    }

    /**
     * 显示加载loading
     *
     */
    private void showLoading(Context context) {
        RemoteViews remoteViews = new RemoteViews(context.getPackageName(), R.layout.layout_widget);
        remoteViews.setViewVisibility(R.id.tv_refresh, View.VISIBLE);
        remoteViews.setViewVisibility(R.id.progress_bar, View.VISIBLE);
        remoteViews.setTextViewText(R.id.tv_refresh, "正在刷新...");
        refreshWidget(context, remoteViews, false);
    }

    /**
     * 隐藏加载loading
     */
    private void hideLoading(Context context) {
        RemoteViews remoteViews = new RemoteViews(context.getPackageName(), R.layout.layout_widget);
        remoteViews.setViewVisibility(R.id.progress_bar, View.GONE);
        remoteViews.setTextViewText(R.id.tv_refresh, "刷新");
        refreshWidget(context, remoteViews, false);
    }



    /**
     * 刷新Widget
     */
    private void refreshWidget(Context context, RemoteViews remoteViews, boolean refreshList) {
        AppWidgetManager appWidgetManager = AppWidgetManager.getInstance(context);
        ComponentName componentName = new ComponentName(context, ListWidgetProvider.class);
        appWidgetManager.updateAppWidget(componentName, remoteViews);
        if (refreshList)
            appWidgetManager.notifyAppWidgetViewDataChanged(appWidgetManager.getAppWidgetIds(componentName), R.id.lv_device);
    }
}
```
针对以上代码，我们着重来看onUpdate()方法。在onUpdate()中我们主要实现了两个功能，第一个功能ListView以外的事件点击，例如点击“刷新”来更新小部件。第二个功能是适配ListView并实现ListView内部Item控件的点击事件。在这个方法中我们首先获取到了一个RemoteView的实例，这个RemoteView对应的就是我们Widget布局的View。关于点击事件的实现代码中注释写的也比较详细，在这里就不做过多解释了。重点是需要了解如何实现并适配ListView.
### RemoteViewsFactory实现列表适配
上面我们提到了RemoteViewsFactory，这个类其实可以类比为ListView的Adapter，该类存在的意义就是为了适配ListView的数据。只不过这里是把Adapter换成RemoteViews来实现的。看下ListRemoteViewsFactory中的代码：

```
class ListRemoteViewsFactory implements RemoteViewsService.RemoteViewsFactory {
    private final static String TAG="Widget";
    private Context mContext;
    private int mAppWidgetId;

    private static List<Device> mDevices;

    /**
     * 构造GridRemoteViewsFactory
     */
    public ListRemoteViewsFactory(Context context, Intent intent) {
        mContext = context;
        mAppWidgetId = intent.getIntExtra(AppWidgetManager.EXTRA_APPWIDGET_ID,
                AppWidgetManager.INVALID_APPWIDGET_ID);
    }

    @Override
    public RemoteViews getViewAt(int position) {
        //  HashMap<String, Object> map;

        // 获取 item_widget_device.xml 对应的RemoteViews
        RemoteViews rv = new RemoteViews(mContext.getPackageName(), R.layout.item_widget_device);

        // 设置 第position位的“视图”的数据
        Device device = mDevices.get(position);
        //  rv.setImageViewResource(R.id.iv_lock, ((Integer) map.get(IMAGE_ITEM)).intValue());
        rv.setTextViewText(R.id.tv_name, device.getName());

        // 设置 第position位的“视图”对应的响应事件
        Intent fillInIntent = new Intent();
        fillInIntent.putExtra("Type", 0);
        fillInIntent.putExtra(ListWidgetProvider.COLLECTION_VIEW_EXTRA, position);
        rv.setOnClickFillInIntent(R.id.rl_widget_device, fillInIntent);


        Intent lockIntent = new Intent();
        lockIntent.putExtra(ListWidgetProvider.COLLECTION_VIEW_EXTRA, position);
        lockIntent.putExtra("Type", 1);
        rv.setOnClickFillInIntent(R.id.iv_lock, lockIntent);

        Intent unlockIntent = new Intent();
        unlockIntent.putExtra("Type", 2);
        unlockIntent.putExtra(ListWidgetProvider.COLLECTION_VIEW_EXTRA, position);
        rv.setOnClickFillInIntent(R.id.iv_unlock, unlockIntent);

        return rv;
    }


    /**
     * 初始化ListView的数据
     */
    private void initListViewData() {
        mDevices = new ArrayList<>();
        mDevices.add(new Device("Hello", 0));
        mDevices.add(new Device("Oitsme", 1));
        mDevices.add(new Device("Hi", 0));
        mDevices.add(new Device("Hey", 1));
    }
    private static int i;
    public static void refresh(){
        i++;
        mDevices.add(new Device("Refresh"+i, 1));
    }

    @Override
    public void onCreate() {
        Log.e(TAG,"onCreate");
        // 初始化“集合视图”中的数据
        initListViewData();
    }

    @Override
    public int getCount() {
        // 返回“集合视图”中的数据的总数
        return mDevices.size();
    }

    @Override
    public long getItemId(int position) {
        // 返回当前项在“集合视图”中的位置
        return position;
    }

    @Override
    public RemoteViews getLoadingView() {
        return null;
    }

    @Override
    public int getViewTypeCount() {
        // 只有一类 ListView
        return 1;
    }

    @Override
    public boolean hasStableIds() {
        return true;
    }

    @Override
    public void onDataSetChanged() {
    }

    @Override
    public void onDestroy() {
        mDevices.clear();
    }
}
```
有了RemoteViewsFactory 还需要有RemoteViewsService才能与ListView关联起来。来看RemoteViewsService的实现类ListWidgetService，很简单，只重写了onGetViewFactory方法：
```
public class ListWidgetService extends RemoteViewsService {

    @Override
    public RemoteViewsService.RemoteViewsFactory onGetViewFactory(Intent intent) {
        return new ListRemoteViewsFactory(this, intent);
    }
}
```
至此我们可以再次回到ListWidgetProvider中的onUpdate()方法，来看ListWidgetService 是如何与ListView关联到一起的了。
```
//  设置 “ListView” 的adapter。
 // (01) intent: 对应启动 ListWidgetService(RemoteViewsService) 的intent
 // (02) setRemoteAdapter: 设置 ListView的适配器
 //  通过setRemoteAdapter将ListView和ListWidgetService关联起来，
 //  以达到通过 ListWidgetService 更新 ListView 的目的
  Intent serviceIntent = new Intent(context, ListWidgetService.class);
  remoteViews.setRemoteAdapter(R.id.lv_device, serviceIntent);
```
### 点击事件处理
Widget中事件点击以及适配ListView，想必大家都有所了解了。那么对于事件的处理我们还没有提到，例如在Widget中点击了刷新后我们不能像在App中那样给控件设置一个事件监听来在回掉方法中处理。在文章开头我们就提到了Widget是依赖广播来实现，因此我们点击了刷新后其实仅仅是发送出来一个广播。如果我们不去处理广播那么点击事件其实是没有任何意义的。因此，来看ListWidgetProvider中第二个比较重要的方法onReceive()。这个方法比较简单，只要我们对特定的广播来做相应的处理就可以了。
```
@Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        AppWidgetManager appWidgetManager = AppWidgetManager.getInstance(context);
	        if (action.equals(COLLECTION_VIEW_ACTION)) {//处理列表中的事件
            // 接受“ListView”的点击事件的广播
            int type = intent.getIntExtra("Type", 0);
            int appWidgetId = intent.getIntExtra(AppWidgetManager.EXTRA_APPWIDGET_ID,
                    AppWidgetManager.INVALID_APPWIDGET_ID);
            int index = intent.getIntExtra(COLLECTION_VIEW_EXTRA, 0);
            switch (type) {
                case 0:
                    Toast.makeText(context, "item" + index, Toast.LENGTH_SHORT).show();
                    break;
                case 1:
                    Toast.makeText(context, "lock"+index, Toast.LENGTH_SHORT).show();
                    break;
                case 2:
                    Toast.makeText(context, "unlock"+index, Toast.LENGTH_SHORT).show();
                    break;
            }
        } else if (action.equals(REFRESH_WIDGET)) {//处理刷新事件
            // 接受“bt_refresh”的点击事件的广播
            Toast.makeText(context, "刷新...", Toast.LENGTH_SHORT).show();
            final AppWidgetManager mgr = AppWidgetManager.getInstance(context);
            final ComponentName cn = new ComponentName(context,ListWidgetProvider.class);
            ListRemoteViewsFactory.refresh();
            mgr.notifyAppWidgetViewDataChanged(mgr.getAppWidgetIds(cn),R.id.lv_device);
            mHandler.postDelayed(runnable,2000);
            showLoading(context);
        }
        super.onReceive(context, intent);
    }
```
最后，别忘了ListWidgetProvider是广播，ListWidgetService是服务，都需要我们在AndroidManifest文件中来注册：
```
<receiver android:name=".widget.ListWidgetProvider">
            <intent-filter>
                <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />

                <!-- ListWidgetProvider接收点击ListView的响应事件 -->
                <action android:name="com.oitsme.COLLECTION_VIEW_ACTION" />
                <!-- ListWidgetProvider接收点击bt_refresh的响应事件 -->
                <action android:name="com.oitsme.REFRESH_WIDGET" />
                <action android:name="com.oitsme.LOCK_ACTION"/>
                <action android:name="com.oitsme.UNLOCK_ACTION"/>
            </intent-filter>
            <meta-data android:name="android.appwidget.provider"
                android:resource="@xml/widget_provider"/>
        </receiver>

        <service
            android:name=".widget.ListWidgetService"
            android:permission="android.permission.BIND_REMOTEVIEWS" />
```