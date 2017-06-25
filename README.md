>最近在项目中研究计步模块，每天0点开始记录当天的步数，类似微信运动。碰到了不少坑今天有时间整理出来给大家看看。
做之前在google、baidu、github上搜了个遍没找到好的，大多数都是需要在后台存活，需要后台Service。
对于现在的各大手机厂商为了提高电池的续航里程（省电），基本上AlertManager、android.intent.action.BOOT_COMPLETED、后台Service都是被干掉的。
后台保活策略Service，基本上没什么用，被手机系统干掉只是时间问题，所以我认为最好也不要去做，就算后台存活了，用户看到这个app非常费电也会被删除的。

##### 目前android计步有两种方式
###### 系统计步芯片
>在Android4.4版本之后，部分机型实现了Sensor.TYPE_STEP_COUNTER传感器，用于纪录用户行走的步数。从手机开机开始纪录，手机关机时重置为0。 
这个记步芯片是系统级别的，相对之前老版本的传感器记步，性能有一些优化：
不会因为App单独用了记步的功能而额外耗电
系统芯片记步是持续的，能够优化部分机型后台不记步的问题。

###### 加速度传感器计算方式
>加速度传感器非常耗电，导致App的耗电量很高，影响用户体验。
需要后台实时运行才能实现记步的功能，如果App进程被系统或者安全软件杀死，导致记步功能没办法使用

>本代码使用Sensor.TYPE_STEP_COUNTER传感器和Sensor.TYPE_ACCELEROMETER传感器实现
如果支持Sensor.TYPE_STEP_COUNTER传感器app不在后台存活也可以计步
如果只支持Sensor.TYPE_ACCELEROMETER传感器app必须开启或者在后台存活才可以计步

简书地址：http://www.jianshu.com/p/ca1e1c3ac086

使用代码非常简单
```
public class MainActivity extends AppCompatActivity {

    private static String TAG = "MainActivity";

    //循环取当前时刻的步数中间的间隔时间
    private long TIME_INTERVAL_REFRESH = 500;

    private Messenger messenger = new Messenger(new Handler(new TodayStepCounterCall()));
    private Messenger mServiceMessenger = null;
    private Handler mDelayHandler = new Handler(new TodayStepCounterCall());
    private int mStepSum;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Intent intent = new Intent(this, VitalityStepService.class);
        startService(intent);
        bindService(intent, new ServiceConnection() {
            @Override
            public void onServiceConnected(ComponentName name, IBinder service) {
                try {
                    Logger.d(TAG, "send msg to fetch step count when onServiceConnected");
                    mServiceMessenger = new Messenger(service);
                    Message msg = Message.obtain(null, BaseConstantDef.MSG_FROM_CLIENT);
                    msg.replyTo = messenger;
                    mServiceMessenger.send(msg);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void onServiceDisconnected(ComponentName name) {

            }
        }, Context.BIND_AUTO_CREATE);
    }

    class TodayStepCounterCall implements Handler.Callback{

        @Override
        public boolean handleMessage(Message msg) {
            switch (msg.what) {
                case BaseConstantDef.MSG_FROM_SERVER:
                    // 更新界面上的步数
                    int type = msg.getData().getInt(VitalityStepService.VITALITY_STEP_TYPE);
                    if(VitalityStepService.VITALITY_STEP_TYPE_REFRESH_SHOW == type) {
                        int step = msg.getData().getInt("step");
                        if (mStepSum != step) {
                            mStepSum = step;
                            updateStepCount();
                        }
                    }
                    mDelayHandler.sendEmptyMessageDelayed(BaseConstantDef.REQUEST_SERVER, TIME_INTERVAL_REFRESH);
                    break;
                case BaseConstantDef.REQUEST_SERVER:
                    try {
                        Message msg1 = Message.obtain(null, BaseConstantDef.MSG_FROM_CLIENT);
                        msg1.replyTo = messenger;
                        mServiceMessenger.send(msg1);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }

                    break;
            }
            return false;
        }
    }

    private void updateStepCount() {
        Log.e(TAG,"updateStepCount : " + mStepSum);
        TextView stepTextView = (TextView)findViewById(R.id.stepTextView);
        stepTextView.setText(mStepSum + "步");
    }
}
```
