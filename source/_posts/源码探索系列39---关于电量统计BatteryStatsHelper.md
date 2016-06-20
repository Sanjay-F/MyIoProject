title:  源码探索系列39---关于电量统计BatteryStatsHelper
date: 2016-06-14 12:55:46
tags: [android,BatteryStatsHelper]
categories: android

------------------------------------------

关于APP的优化，有一点就是关于电量的内容。
关于我们的app耗电，在前几年，各类的XXX管家等软件都充斥各种耗电排行榜和各种所谓的节能省电设置。PM看了说我们得优化啊，让开发不得不花点心事在这事情上。那时觉得，因为你用户用得多，耗费的电自然多啊，排在前面很正常啊，应该开心才对啊！！
不过到好，前段时间看到一份“互联网女皇”玛丽·米克尔发布的最新版《全球互联网趋势报告》，BAT的统治了71%的互联网流量，相应的手机电量估计也不弱。

说了那么多，我们先来看下到底系统是怎么多电量的统计的，知道他是怎么算的，我们才能更好的对症下药。

<!--more-->

为了研究，我们需要找到一个切入口，在我们的设置里面，有一个关于电量的选项，里面一般会分“软件消耗”和“硬件消耗”两栏。
是的，这里就是我们的突破口，既然他能够得到这部分数据，理论我们可以跟踪关于计算的信息，于是我们就去了对应的代码看下
# 起航

API:23

在`/packages/apps/Settings/src/com/android/settings/fuelgauge`目录下，经过一轮查找，在`PowerUsageSummary`的`refreshStats()`函数，我们看到它调用了父类的`refreshStats()`，在这个函数内容如下：

	 protected void refreshStats() {
	        mStatsHelper.refreshStats(BatteryStats.STATS_SINCE_CHARGED, 
							          mUm.getUserProfiles());
	 }
这里的`mStatsHelper`变量是`BatteryStatsHelper.class`，这样我们知道了我们比较关心的内容。

## BatteryStatsHelper.refreshStats()

	public void refreshStats(int statsType, int asUser) {
        SparseArray<UserHandle> users = new SparseArray<>(1);
        users.put(asUser, new UserHandle(asUser));
        refreshStats(statsType, users);
    }
    
	public void refreshStats(int statsType, SparseArray<UserHandle> asUsers) {
        refreshStats(statsType, asUsers, SystemClock.elapsedRealtime() * 1000,
                SystemClock.uptimeMillis() * 1000);
    }
    
	 public void refreshStats(int statsType, SparseArray<UserHandle> asUsers, 
							 long rawRealtimeUs,long rawUptimeUs) {
							 
		...//省略一堆内容
		
		 if (mCpuPowerCalculator == null) {
            mCpuPowerCalculator = new CpuPowerCalculator(mPowerProfile);
        }
        mCpuPowerCalculator.reset();

		...//各种XXXCalculator初始化

        if (mFlashlightPowerCalculator == null) {
            mFlashlightPowerCalculator = new FlashlightPowerCalculator(mPowerProfile);
        }
        mFlashlightPowerCalculator.reset();

       mTypeBatteryRealtime = mStats.computeBatteryRealtime(rawRealtimeUs, mStatsType);
		mRawUptime = rawUptimeUs;
        mRawRealtime = rawRealtimeUs;
		              
        mMinDrainedPower = (mStats.getLowDischargeAmountSinceCharge()
                * mPowerProfile.getBatteryCapacity()) / 100;
        mMaxDrainedPower = (mStats.getHighDischargeAmountSinceCharge()
                * mPowerProfile.getBatteryCapacity()) / 100;
       
	       //1. 我们关心的一个函数，这个是app的耗电的
        processAppUsage(asUsers);
        
        //我也是挺好奇，为何不把排序挪到别的地方去
        // Before aggregating apps in to users, collect all apps to sort by their ms per packet.
        for (int i=0; i<mUsageList.size(); i++) {
            BatterySipper bs = mUsageList.get(i);
            bs.computeMobilemspp();
            if (bs.mobilemspp != 0) {
                mMobilemsppList.add(bs);
            }
        }

        for (int i=0; i<mUserSippers.size(); i++) {
            List<BatterySipper> user = mUserSippers.valueAt(i);
            for (int j=0; j<user.size(); j++) {
                BatterySipper bs = user.get(j);
                bs.computeMobilemspp();
                if (bs.mobilemspp != 0) {
                    mMobilemsppList.add(bs);
                }
            }
        }
        
        Collections.sort(mMobilemsppList, new Comparator<BatterySipper>() {
            @Override
            public int compare(BatterySipper lhs, BatterySipper rhs) {
                return Double.compare(rhs.mobilemspp, lhs.mobilemspp);
            }
        });
       
       //又一个我们关心点的函数，就是硬件耗电的统计函数
        processMiscUsage();

        Collections.sort(mUsageList);

        // At this point, we've sorted the list so we are guaranteed the max values are at the top.
        // We have only added real powers so far.
        if (!mUsageList.isEmpty()) {
            mMaxRealPower = mMaxPower = mUsageList.get(0).totalPowerMah;
            final int usageListCount = mUsageList.size();
            for (int i = 0; i < usageListCount; i++) {
                mComputedPower += mUsageList.get(i).totalPowerMah;
            }
        }
        
        ...
    }
        
在初始化各类的calculaotr的时候，传入的参数是`mPowerProfile`，他是存储部件电流数值的。因为在计算电量的时候，是需要结合具体的设备型号的耗电量情况来做计算的，因此在`frameworks/base/core/res/res/xml`有个`power_profile.xml`文件，给各类OEM做配置，因为具体硬件配置和对应的耗电情况，只有生成设备的OEM自己清楚。

另外，这个类有两个我们比较关注的入口，分别是负责统计软件`processAppUsage()`和负责统计硬件耗电的`processMiscUsage()`。我们先去看我们比较关心的processAppUsage()

### processAppUsage()

    private void processAppUsage(SparseArray<UserHandle> asUsers) {
	
	     final boolean forAllUsers = (asUsers.get(UserHandle.USER_ALL) != null);
	     mStatsPeriod = mTypeBatteryRealtime;
	
	     BatterySipper osSipper = null;
	     final SparseArray<? extends Uid> uidStats = mStats.getUidStats();
	     final int NU = uidStats.size();
	     for (int iu = 0; iu < NU; iu++) {
	         final Uid u = uidStats.valueAt(iu);
	         final BatterySipper app = new BatterySipper(BatterySipper.DrainType.APP, u, 0);
	
	         mCpuPowerCalculator.calculateApp(app, u, mRawRealtime, mRawUptime, mStatsType);
	         mWakelockPowerCalculator.calculateApp(app, u, mRawRealtime, mRawUptime, mStatsType);
	         mMobileRadioPowerCalculator.calculateApp(app, u, mRawRealtime, mRawUptime, mStatsType);
	         mWifiPowerCalculator.calculateApp(app, u, mRawRealtime, mRawUptime, mStatsType);
	         mBluetoothPowerCalculator.calculateApp(app, u, mRawRealtime, mRawUptime, mStatsType);
	         mSensorPowerCalculator.calculateApp(app, u, mRawRealtime, mRawUptime, mStatsType);
	         mCameraPowerCalculator.calculateApp(app, u, mRawRealtime, mRawUptime, mStatsType);
	         mFlashlightPowerCalculator.calculateApp(app, u, mRawRealtime, mRawUptime, mStatsType);
	
	         final double totalPower = app.sumPower();
	         if (DEBUG && totalPower != 0) {
	             Log.d(TAG, String.format("UID %d: total power=%s", u.getUid(),
	                     makemAh(totalPower)));
	         }
	
	         // Add the app to the list if it is consuming power.
	         if (totalPower != 0 || u.getUid() == 0) {
	             //
	             // Add the app to the app list, WiFi, Bluetooth, etc, or into "Other Users" list.
	             //
	             final int uid = app.getUid();
	             final int userId = UserHandle.getUserId(uid);
	             if (uid == Process.WIFI_UID) {
	                 mWifiSippers.add(app);
	             } else if (uid == Process.BLUETOOTH_UID) {
	                 mBluetoothSippers.add(app);
	             } else if (!forAllUsers && asUsers.get(userId) == null
	                     && UserHandle.getAppId(uid) >= Process.FIRST_APPLICATION_UID) {
	                 // We are told to just report this user's apps as one large entry.
	                 List<BatterySipper> list = mUserSippers.get(userId);
	                 if (list == null) {
	                     list = new ArrayList<>();
	                     mUserSippers.put(userId, list);
	                 }
	                 list.add(app);
	             } else {
	                 mUsageList.add(app);
	             }
	
	             if (uid == 0) {
	                 osSipper = app;
	             }
	         }
	     }
	
	     if (osSipper != null) {
	         // The device has probably been awake for longer than the screen on
	         // time and application wake lock time would account for.  Assign
	         // this remainder to the OS, if possible.
	         mWakelockPowerCalculator.calculateRemaining(osSipper, mStats, mRawRealtime,
	                                                     mRawUptime, mStatsType);
	         osSipper.sumPower();
	     }
    }

对于电量统计，我们看到他是以UID来做一句的，一般情况是一个APP对应一个UID咯。不过如果是相同的签名，共同的ShareUserId的话，就会有相同的UID。但这还是挺少出现的，就算某些公司的全家桶，也不会是相同签名。

主要是靠各种XXXCalculator去计算，目前有8个，这8个耗电计算器都继承自PowerCalculator抽象类。
 
    PowerCalculator mCpuPowerCalculator;
    PowerCalculator mWakelockPowerCalculator;
    MobileRadioPowerCalculator mMobileRadioPowerCalculator;
    PowerCalculator mWifiPowerCalculator;
    PowerCalculator mBluetoothPowerCalculator;
    PowerCalculator mSensorPowerCalculator;
    PowerCalculator mCameraPowerCalculator;
    PowerCalculator mFlashlightPowerCalculator;
    
这样我们优化算是有了一个大方向，明白要优化那些了，就是上面八巨头！     
计算的结果保存到app变量里面，然后我们调用`app.sumPower()`获得总耗电量结果。

	/**
     * Sum all the powers and store the value into `value`.
     * @return the sum of all the power in this BatterySipper.
     */
    public double sumPower() {
        return totalPowerMah = usagePowerMah + wifiPowerMah + gpsPowerMah + cpuPowerMah 
        + sensorPowerMah + mobileRadioPowerMah + wakeLockPowerMah + cameraPowerMah 
        + flashlightPowerMah;
    }
    
最后，关于计算值添加到的列表上，分了UID为WIFI，蓝牙，ROOT用户和APP四大类。

知道这样的大概流程后，我们分各个计算器，看下里面具体怎么处理的。
这对于指导我们写出**绿色环保**的代码  还是有一定的知道意义的！对吧？

### CpuPowerCalculator
 
	 public CpuPowerCalculator(PowerProfile profile) {
        final int speedSteps = profile.getNumSpeedSteps();
        //Returns the number of speeds that the CPU can be run at.
        
        mPowerCpuNormal = new double[speedSteps];
        //后面用来记录不同频率下的功耗
        
        mSpeedStepTimes = new long[speedSteps];
        for (int p = 0; p < speedSteps; p++) {
            mPowerCpuNormal[p] = profile.getAveragePower(PowerProfile.POWER_CPU_ACTIVE, p);
        }
    }
这里的`getNumSpeedSteps()`返回的是cpu支持的主频数目。为了控制耗电，很多手机在设置有提供运行的模式，什么高性能，平衡，节能的选项。
还有些手机厂商用的CPU什么8频12模的，4个核能运行在1.5GHz的，节能时候可以跑另外4个运行在1.0GHz的。例如三星的猎户座5420。
	

我们先来看下关于CPU的计算部分 

	@Override
    public void calculateApp(BatterySipper app, BatteryStats.Uid u, long rawRealtimeUs,
                             long rawUptimeUs, int statsType) {
        final int speedSteps = mSpeedStepTimes.length;

        long totalTimeAtSpeeds = 0;
        
        //1. 计算不同频率下的运行时间
        for (int step = 0; step < speedSteps; step++) {
            mSpeedStepTimes[step] = u.getTimeAtCpuSpeed(step, statsType);
            totalTimeAtSpeeds += mSpeedStepTimes[step];
        }
        //getTimeAtCpuSpeed(）返回的数据是毫秒级别的，
        //居然和1做比较，也是怪怪的感觉，虽然CPU可以是纳秒的时间单位
        totalTimeAtSpeeds = Math.max(totalTimeAtSpeeds, 1);

        app.cpuTimeMs = (u.getUserCpuTimeUs(statsType) + u.getSystemCpuTimeUs(statsType)) / 1000;
         
		//2.统计CPU的耗电量
		//根据这个公司，我们的一句废话结论是，不用CPU是节能的最高境界
        double cpuPowerMaMs = 0;
        for (int step = 0; step < speedSteps; step++) {
            final double ratio = (double) mSpeedStepTimes[step] / totalTimeAtSpeeds;
            final double cpuSpeedStepPower = ratio * app.cpuTimeMs * mPowerCpuNormal[step];             
            cpuPowerMaMs += cpuSpeedStepPower; 
        }
         
        // Keep track of the package with highest drain.
        double highestDrain = 0;

        app.cpuFgTimeMs = 0;
        final ArrayMap<String, ? extends BatteryStats.Uid.Proc> processStats = u.getProcessStats();
        final int processStatsCount = processStats.size();
        
         //3. 计算同一个uid的不同进程耗电情况
        for (int i = 0; i < processStatsCount; i++) {
            final BatteryStats.Uid.Proc ps = processStats.valueAt(i);
            final String processName = processStats.keyAt(i);
            app.cpuFgTimeMs += ps.getForegroundTime(statsType);

            final long costValue = ps.getUserTime(statsType) + ps.getSystemTime(statsType)
                    + ps.getForegroundTime(statsType);

            // Each App can have multiple packages and with multiple running processes.
            // Keep track of the package who's process has the highest drain.
            if (app.packageWithHighestDrain == null ||
                    app.packageWithHighestDrain.startsWith("*")) {
                highestDrain = costValue;
                app.packageWithHighestDrain = processName;
            } else if (highestDrain < costValue && !processName.startsWith("*")) {
                highestDrain = costValue;
                app.packageWithHighestDrain = processName;
            }
        }

        // Ensure that the CPU times make sense.
        if (app.cpuFgTimeMs > app.cpuTimeMs) {           
            // Statistics may not have been gathered yet.
            app.cpuTimeMs = app.cpuFgTimeMs;
        }

        // Convert the CPU power to mAh
        app.cpuPowerMah = cpuPowerMaMs / (60 * 60 * 1000);
    }

看完上面一长串，我们可能比较在意的一条公式是：       

>final double cpuSpeedStepPower = ratio * app.cpuTimeMs * mPowerCpuNormal[step];




### WakelockPowerCalculator
 
	 public WakelockPowerCalculator(PowerProfile profile) {
        mPowerWakelock = profile.getAveragePower(PowerProfile.POWER_CPU_AWAKE);
    }

    @Override
    public void calculateApp(BatterySipper app, BatteryStats.Uid u, long rawRealtimeUs,
                             long rawUptimeUs, int statsType) {
                             
        long wakeLockTimeUs = 0;
        final ArrayMap<String, ? extends BatteryStats.Uid.Wakelock> wakelockStats =
                u.getWakelockStats();
        final int wakelockStatsCount = wakelockStats.size();
        
        for (int i = 0; i < wakelockStatsCount; i++) {
            final BatteryStats.Uid.Wakelock wakelock = wakelockStats.valueAt(i);

            // Only care about partial wake locks since full wake locks
            // are canceled when the user turns the screen off.
            BatteryStats.Timer timer = wakelock.getWakeTime(
							           BatteryStats.WAKE_TYPE_PARTIAL);
            if (timer != null) {
                wakeLockTimeUs += timer.getTotalTimeLocked(rawRealtimeUs, statsType);
            }
        }
        
        app.wakeLockTimeMs = wakeLockTimeUs / 1000; // convert to millis
        mTotalAppWakelockTimeMs += app.wakeLockTimeMs;
        // Add cost of holding a wake lock.
        app.wakeLockPowerMah = (app.wakeLockTimeMs * mPowerWakelock) / (1000*60*60);        
	　　
	}

我们看这个`wakeLock`的计算还是相对简单的，没什么好说的，就是统计下时间，然后乘以功耗。
关于这个wakeLock，其实用的还是挺少的，除了在OTA和以前的语音聊天时候用过之后，好像也没用过了。


###  MobileRadioPowerCalculator
手机的无线耗电就是通过这个来计算的。现在的手机应该它算是耗电大户了，开个4G网络耗电超快，比WIFI还耗电

	 public MobileRadioPowerCalculator(PowerProfile profile, BatteryStats stats) {
        mPowerRadioOn = profile.getAveragePower(PowerProfile.POWER_RADIO_ACTIVE);
        for (int i = 0; i < mPowerBins.length; i++) {
            mPowerBins[i] = profile.getAveragePower(PowerProfile.POWER_RADIO_ON, i);
        }
        mPowerScan = profile.getAveragePower(PowerProfile.POWER_RADIO_SCANNING);
        mStats = stats;
    }

    @Override
    public void calculateApp(BatterySipper app, BatteryStats.Uid u, long rawRealtimeUs,
                             long rawUptimeUs, int statsType) {
                             
        // Add cost of mobile traffic.
        app.mobileRxPackets = u.getNetworkActivityPackets(
			        BatteryStats.NETWORK_MOBILE_RX_DATA,statsType);	        
        app.mobileTxPackets = u.getNetworkActivityPackets(
			        BatteryStats.NETWORK_MOBILE_TX_DATA,statsType);
	        
        app.mobileActive = u.getMobileRadioActiveTime(statsType) / 1000;
        app.mobileActiveCount = u.getMobileRadioActiveCount(statsType);
        app.mobileRxBytes = u.getNetworkActivityBytes(
			        BatteryStats.NETWORK_MOBILE_RX_DATA,statsType);
        app.mobileTxBytes = u.getNetworkActivityBytes(
			        BatteryStats.NETWORK_MOBILE_TX_DATA,statsType);

        if (app.mobileActive > 0) {
            // We are tracking when the radio is up, so can use the active time to
            // determine power use.
            mTotalAppMobileActiveMs += app.mobileActive;
            app.mobileRadioPowerMah = (app.mobileActive * mPowerRadioOn) / (1000*60*60);
            
        } else {
            // We are not tracking when the radio is up, so must approximate power use
            // based on the number of packets.
            
            app.mobileRadioPowerMah = (app.mobileRxPackets + app.mobileTxPackets)            
							                                                * getMobilePowerPerPacket(rawRealtimeUs, statsType);
        }
     }
	
	/**
     * Return estimated power (in mAs) of sending or receiving a packet with the mobile radio.
     */
    private double getMobilePowerPerPacket(long rawRealtimeUs, int statsType) {
        final long MOBILE_BPS = 200000; // TODO: Extract average bit rates from system
        final double MOBILE_POWER = mPowerRadioOn / 3600;

        final long mobileRx = mStats.getNetworkActivityPackets(BatteryStats.NETWORK_MOBILE_RX_DATA,
                statsType);
        final long mobileTx = mStats.getNetworkActivityPackets(BatteryStats.NETWORK_MOBILE_TX_DATA,
                statsType);
        final long mobileData = mobileRx + mobileTx;

        final long radioDataUptimeMs =
                mStats.getMobileRadioActiveTime(rawRealtimeUs, statsType) / 1000;
        final double mobilePps = (mobileData != 0 && radioDataUptimeMs != 0)
                ? (mobileData / (double)radioDataUptimeMs)
                : (((double)MOBILE_BPS) / 8 / 2048);
        return (MOBILE_POWER / mobilePps) / (60*60);
    }

我们看到，他最后在计算`app.mobileRadioPowerMah` 的时候，是分两种情况的，一种是我们跟踪网络开的时候，用的是`mobileActive时间` * `功耗`。看来开网络的是否，每秒的功耗是相对固定的。另外一种，对于没有跟踪信号开关的，我们就根据收发包来做估算，拿(Rx+Tx)*getMobilePowerPerPacket()每个包的耗电，这样得到总和，看来我们收发都是一样的。不过也挺有趣的，都知道收发包的多少，难道不知道时间？

### WifiPowerCalculator

关于wifi的统计，目前是分两种的，当有Wifi电源报告的时候，采用WifiPowerCalculator，这个是在BatteryStats支持能源报告的时候。
没有的话采用WifiPowerEstimator，这个是基于Timer来做处理的。

	final boolean hasWifiPowerReporting = checkHasWifiPowerReporting(mStats, mPowerProfile);
        if (mWifiPowerCalculator == null || hasWifiPowerReporting != mHasWifiPowerReporting) {
            mWifiPowerCalculator = hasWifiPowerReporting ?
                    new WifiPowerCalculator(mPowerProfile) :
                    new WifiPowerEstimator(mPowerProfile);
            mHasWifiPowerReporting = hasWifiPowerReporting;
        }

我们先来看下前者

####  WifiPowerCalculator

	public WifiPowerCalculator(PowerProfile profile) {
        mIdleCurrentMa = profile.getAveragePower(PowerProfile.POWER_WIFI_CONTROLLER_IDLE);
        mTxCurrentMa = profile.getAveragePower(PowerProfile.POWER_WIFI_CONTROLLER_TX);
        mRxCurrentMa = profile.getAveragePower(PowerProfile.POWER_WIFI_CONTROLLER_RX);
    }

分了三类，接收，发送和空闲时候的电流情况。单位看起来就是mA毫安。

	 public void calculateApp(BatterySipper app, BatteryStats.Uid u, long rawRealtimeUs,
                             long rawUptimeUs, int statsType) {
                             
        final long idleTime = u.getWifiControllerActivity(
				        BatteryStats.CONTROLLER_IDLE_TIME,statsType);
        final long txTime = u.getWifiControllerActivity(
					    BatteryStats.CONTROLLER_TX_TIME, statsType);
        final long rxTime = u.getWifiControllerActivity(
				        BatteryStats.CONTROLLER_RX_TIME, statsType);
				        
        app.wifiRunningTimeMs = idleTime + rxTime + txTime;
        
        app.wifiPowerMah =((idleTime * mIdleCurrentMa) + (txTime * mTxCurrentMa) +
					         (rxTime * mRxCurrentMa))  / (1000*60*60);
					         
        mTotalAppPowerDrain += app.wifiPowerMah;

        app.wifiRxPackets = u.getNetworkActivityPackets(
					        BatteryStats.NETWORK_WIFI_RX_DATA,statsType);
        app.wifiTxPackets = u.getNetworkActivityPackets(
					        BatteryStats.NETWORK_WIFI_TX_DATA,statsType);
        app.wifiRxBytes = u.getNetworkActivityBytes(
					        BatteryStats.NETWORK_WIFI_RX_DATA,statsType);
        app.wifiTxBytes = u.getNetworkActivityBytes(
					        BatteryStats.NETWORK_WIFI_TX_DATA,statsType);         
    }

计算公式还是挺简洁的

	app.wifiPowerMah =((idleTime * mIdleCurrentMa) + (txTime * mTxCurrentMa) +
					         (rxTime * mRxCurrentMa))  / (1000*60*60);
分别计算空间idle,Tx,Rx的时间乘以对应的消耗mA，就得到总的值。

#### WifiPowerEstimator

接着我们来看下另外一种方式的计算

	public WifiPowerEstimator(PowerProfile profile) {
        mWifiPowerPerPacket = getWifiPowerPerPacket(profile);
        mWifiPowerOn = profile.getAveragePower(PowerProfile.POWER_WIFI_ON);
        mWifiPowerScan = profile.getAveragePower(PowerProfile.POWER_WIFI_SCAN);
        mWifiPowerBatchScan = profile.getAveragePower(PowerProfile.POWER_WIFI_BATCHED_SCAN);
    }
通过这个构造函数和前面的经验，我们大搞知道了一个套路，在计算功耗的时候，可能是根据wifi的不同状态来区分的，根据打开了wifi，在扫描scan和BatchScan来划分计算的。
 
    @Override
    public void calculateApp(BatterySipper app, BatteryStats.Uid u, long rawRealtimeUs,
                             long rawUptimeUs, int statsType) {
                             
        app.wifiRxPackets = u.getNetworkActivityPackets(BatteryStats.NETWORK_WIFI_RX_DATA,
                statsType);
        app.wifiTxPackets = u.getNetworkActivityPackets(BatteryStats.NETWORK_WIFI_TX_DATA,
                statsType);
        app.wifiRxBytes = u.getNetworkActivityBytes(BatteryStats.NETWORK_WIFI_RX_DATA,
                statsType);
        app.wifiTxBytes = u.getNetworkActivityBytes(BatteryStats.NETWORK_WIFI_TX_DATA,
                statsType);

        final double wifiPacketPower = (app.wifiRxPackets + app.wifiTxPackets)
                * mWifiPowerPerPacket;

        app.wifiRunningTimeMs = u.getWifiRunningTime(rawRealtimeUs, statsType) / 1000;
        mTotalAppWifiRunningTimeMs += app.wifiRunningTimeMs;        
        final double wifiLockPower = (app.wifiRunningTimeMs * mWifiPowerOn) / (1000*60*60);

        final long wifiScanTimeMs = u.getWifiScanTime(rawRealtimeUs, statsType) / 1000;
        final double wifiScanPower = (wifiScanTimeMs * mWifiPowerScan) / (1000*60*60);

        double wifiBatchScanPower = 0;
        for (int bin = 0; bin < BatteryStats.Uid.NUM_WIFI_BATCHED_SCAN_BINS; bin++) {
            final long batchScanTimeMs =
                    u.getWifiBatchedScanTime(bin, rawRealtimeUs, statsType) / 1000;
            final double batchScanPower = (batchScanTimeMs * mWifiPowerBatchScan) / (1000*60*60);
            
            wifiBatchScanPower += batchScanPower;
        }

        app.wifiPowerMah = wifiPacketPower + wifiLockPower +
									    wifiScanPower + wifiBatchScanPower; 
    }
    
	/**
     * Return estimated power (in mAs) of sending a byte with the Wi-Fi radio.
     */
    private static double getWifiPowerPerPacket(PowerProfile profile) {
        final long WIFI_BPS = 1000000; // TODO: Extract average bit rates from system
        final double WIFI_POWER = profile.getAveragePower(PowerProfile.POWER_WIFI_ACTIVE)
                / 3600;
        return (WIFI_POWER / (((double)WIFI_BPS) / 8 / 2048)) / (60*60);
    }
    
看完代码，我们验证了一开始的猜想，看来关于这部分，都是类似的套路，只需要先看下构造函数，就可以知道怎么计算功耗的

	app.wifiPowerMah = wifiPacketPower + wifiLockPower + 
						wifiScanPower + wifiBatchScanPower;
不过有点奇怪的是，为何有多一个wifiLockPower，根据计算公式，他是wifiRunning时候的消耗值，我们在看MobileRadio的时候，他是分开来处理计算的，这里就合并起来了。


### BluetoothPowerCalculator

	 public BluetoothPowerCalculator(PowerProfile profile) {
        mIdleMa = profile.getAveragePower(PowerProfile.POWER_BLUETOOTH_CONTROLLER_IDLE);
        mRxMa = profile.getAveragePower(PowerProfile.POWER_BLUETOOTH_CONTROLLER_RX);
        mTxMa = profile.getAveragePower(PowerProfile.POWER_BLUETOOTH_CONTROLLER_TX);
    }

    @Override
    public void calculateApp(BatterySipper app, BatteryStats.Uid u, long rawRealtimeUs,
                             long rawUptimeUs, int statsType) {
        // No per-app distribution yet.
    }

这也是逗我？居然是空的？我看了下还真的就是没有...还没写。
搜索了下，这个类是从API23加进来的，也就是说，在Android6以前，是没有统计蓝牙的，虽然现在加了也和没加一样...我以为还是分idle，tx，rx三个来算的。。

### mSensorPowerCalculator

	public SensorPowerCalculator(PowerProfile profile, SensorManager sensorManager) {
        mSensors = sensorManager.getSensorList(Sensor.TYPE_ALL);
        mGpsPowerOn = profile.getAveragePower(PowerProfile.POWER_GPS_ON);
    }
    
构造函数很简单，只有gps的，别的传感器不理了？

    @Override
    public void calculateApp(BatterySipper app, BatteryStats.Uid u, long rawRealtimeUs,
                             long rawUptimeUs, int statsType) {
        // Process Sensor usage
        final SparseArray<? extends BatteryStats.Uid.Sensor> sensorStats = u.getSensorStats();
        final int NSE = sensorStats.size();
        for (int ise = 0; ise < NSE; ise++) {
            final BatteryStats.Uid.Sensor sensor = sensorStats.valueAt(ise);
            final int sensorHandle = sensorStats.keyAt(ise);
            final BatteryStats.Timer timer = sensor.getSensorTime();
            final long sensorTime = timer.getTotalTimeLocked(rawRealtimeUs, statsType) / 1000;
            switch (sensorHandle) {
                case BatteryStats.Uid.Sensor.GPS:
                    app.gpsTimeMs = sensorTime;
                    app.gpsPowerMah = (app.gpsTimeMs * mGpsPowerOn) / (1000*60*60);
                    break;
                default:
                    final int sensorsCount = mSensors.size();
                    for (int i = 0; i < sensorsCount; i++) {
                        final Sensor s = mSensors.get(i);
                        if (s.getHandle() == sensorHandle) {
                            app.sensorPowerMah += (sensorTime * s.getPower()) / (1000*60*60);
                            break;
                        }
                    }
                    break;
            }
        }
    }

看这个统计部分内容，是针对了GPS做单独处理，确实，目前手机耗电的比较厉害的传感器就是GPS了，开地图导航都挺费电的。其余的先对耗电还是挺小的。
这样我们看到的计算公式有两个：

	app.gpsPowerMah = (app.gpsTimeMs * mGpsPowerOn) / (1000*60*60);

和除GPS外的各个路人甲级别的传感器之和：
	
	app.sensorPowerMah += (sensorTime * s.getPower()) / (1000*60*60);


### mCameraPowerCalculator

	public CameraPowerCalculator(PowerProfile profile) {
        mCameraPowerOnAvg = profile.getAveragePower(PowerProfile.POWER_CAMERA);
    }
我们看这个拍照的构造函数，知道它在计算上，估计是不区分你拍照时候是用什么像素，多大的ISO等等信息的，一视同仁哈！

    @Override
    public void calculateApp(BatterySipper app, BatteryStats.Uid u, long rawRealtimeUs,
                             long rawUptimeUs, int statsType) {

        // Calculate camera power usage.  Right now, 
        // this is a (very) rough estimate based on the
        // average power usage for a typical camera application.
        
        final BatteryStats.Timer timer = u.getCameraTurnedOnTimer();
        if (timer != null) {
          final long totalTime =timer.getTotalTimeLocked(rawRealtimeUs,statsType)/ 1000;
          app.cameraTimeMs = totalTime;
          app.cameraPowerMah = (totalTime * mCameraPowerOnAvg) / (1000*60*60);
        } else {
            app.cameraTimeMs = 0;
            app.cameraPowerMah = 0;
        }
    }
    
真的是这样，没什么好说的，很简单

          app.cameraPowerMah = (totalTime * mCameraPowerOnAvg) / (1000*60*60);
想吐槽的是，从头到尾好多MagicNumber，就不能把后面的1000*60*60弄好一个Hour的常量放到他们的父类
`PowerCalculator`里面去嘛？



### FlashlightPowerCalculator

最后就是这个开闪光灯的，以前各类的手电筒应用泛滥啊，哈哈...自己看过一些处理方案，就是跑去调用摄像头的API，然后影藏画面，只开他的闪光灯为Torch的模式。

	public FlashlightPowerCalculator(PowerProfile profile) {
        mFlashlightPowerOnAvg = profile.getAveragePower(PowerProfile.POWER_FLASHLIGHT);
    }

    @Override
    public void calculateApp(BatterySipper app, BatteryStats.Uid u, long rawRealtimeUs,
                             long rawUptimeUs, int statsType) {
 
        final BatteryStats.Timer timer = u.getFlashlightTurnedOnTimer();
        if (timer != null) {
         final long totalTime=timer.getTotalTimeLocked(rawRealtimeUs,statsType)/1000;
            app.flashlightTimeMs = totalTime;
            app.flashlightPowerMah = (totalTime * mFlashlightPowerOnAvg) / (1000*60*60);
        } else {
            app.flashlightTimeMs = 0;
            app.flashlightPowerMah = 0;
        }
    }
也很简单，没什么好说的，时间*电量的公式

## 优化

看完这么长的关于电量统计部分的内容，那么怎么知道我们去做电量优化工作呢，写出绿色环保的代码？

写代码的最高境界就是不写啊，这样什么鬼消耗也没有。

开玩笑   ：)

在做优化，当然要知道自己都耗电在那个地方啊！是吧？要不然怎么知道优化的方向？

好在5.0后，安卓提供了查电量的指令，我们可以去获取我们的程序耗电信息，利用下面这条指令

	adb shell dumpsys batterystats   yourPackage.name

然后你获得下面这么一大堆的信息

	 Daily stats:
	  Current start time: 2016-06-13-23-23-16
	  Next min deadline: 2016-06-14-01-00-00
	  Next max deadline: 2016-06-14-03-00-00
	  Current daily steps:
	
	//由于是一个demo，什么也没写，所以下面一大堆是0。
	Statistics since last charge:
	  System starts: 33, currently on battery: false
	  Time on battery: 0ms (0.0%) realtime, 0ms (0.0%) uptime
	  Time on battery screen off: 0ms (0.0%) realtime, 0ms (0.0%) uptime
	  Total run time: 1m 57s 828ms realtime, 1m 57s 827ms uptime
	  Start clock time: 2016-06-13-23-22-41
	  Screen on: 0ms (--%) 1x, Interactive: 0ms (--%)
	  Screen brightnesses: (no activity)
	  Connectivity changes: 1
	  Mobile total received: 0B, sent: 0B (packets received 0, sent 0)
	  Phone signal levels: (no activity)
	  Signal scanning time: 0ms
	  Radio types: (no activity)
	  Mobile radio active time: 0ms (--%) 0x
	  Wi-Fi total received: 0B, sent: 0B (packets received 0, sent 0)
	  Wifi on: 0ms (--%), Wifi running: 0ms (--%)
	  Wifi states: (no activity)
	  Wifi supplicant states: (no activity)
	  Wifi signal levels: (no activity)
	  WiFi Idle time: 0ms (--%)
	  WiFi Rx time:   0ms (--%)
	  WiFi Tx time:   0ms (--%)
	  WiFi Power drain: 0mAh
	  Bluetooth Idle time: 0ms (--%)
	  Bluetooth Rx time:   0ms (--%)
	  Bluetooth Tx time:   0ms (--%)
	  Bluetooth Power drain: 0mAh
	
	  Device battery use since last full charge
	    Amount discharged (lower bound): 0
	    Amount discharged (upper bound): 0
	    Amount discharged while screen on: 0
	    Amount discharged while screen off: 0
	
	  1000:
	    Wake lock ActivityManager-Sleep realtime
	    Wake lock WifiSuspend realtime
	    Wake lock WiredAccessoryManager realtime
	    Wake lock *alarm* realtime
	    Wake lock PhoneWindowManager.mPowerKeyWakeLock realtime
	    Wake lock AudioMix realtime
	    Wake lock NetworkStats realtime
	    Wake lock GpsLocationProvider realtime
	    Wake lock SyncLoopWakeLock realtime
	    Wake lock SCREEN_FROZEN realtime
	    Sensor 1: (not used)
	    Apk com.android.location.fused:
	      Service com.android.location.fused.FusedLocationService:
	        Created for: 0ms uptime
	        Starts: 0, launches: 1
	    Apk com.android.keychain:
	      (nothing executed)
	    Apk com.android.server.telecom:
	      Service com.android.server.telecom.components.TelecomService:
	        Created for: 0ms uptime
	        Starts: 0, launches: 1
	    Apk android:
	      Service android.hardware.location.GeofenceHardwareService:
	        Created for: 0ms uptime
	        Starts: 0, launches: 1
	      Service com.android.internal.backup.LocalTransportService:
	        Created for: 0ms uptime
	        Starts: 0, launches: 1
	  u0a60:
	    Wake lock *launch* realtime

查看一些项目的信息，我们可以对应的去做特定的优化。

实际上，我们也是知道个大概的情况，不是能很细致的告诉我，那几行代码的调用得比较多，耗费了较多的电量。这时候结合`MethodTracking`会比较好的帮助我们去看。

当然了，上面数据看起来还是挺枯燥无味的，为此谷歌在5.0引入了一个[`Battery Historian`](https://developer.android.com/about/versions/android-5.0.html#Power)来帮助我们视觉化的看堆内容


　
 adb shell dumpsys batterystats > com.package.name > xxx.txt
 //得到我们指定的app电量消耗信息到某个文件

通过上面这条指令，在我们的项目文件目录下，就生成了一份新的文件。
然后我们可以用谷歌写的一个Python脚本，把这个换成可以看的html页面。
对于需要安装python环境的，[看这个教程](http://jingyan.baidu.com/article/7908e85c78c743af491ad261.html)

项目地址如下：
https://github.com/google/battery-historian

 (谷歌在16年的I/O把这个升级到了2.0，,配置比较麻烦，[看这个教程](http://ph0b.com/battery-historian-2-0-windows/?utm_source=tuicool&utm_medium=referral) ，很逗逼的是没有旧版本的Tag，没有Release的。只有在日志看到一条更新信息，这样做一点都不友好！！我只是想静静的看下结果就好了)

接着我们允许下面的指令

	python historian.py xxx.txt > xxx.html


用游览器打开这个文件，我们收获一张类似下面的图片！

![enter image description here](http://www.41443.com/uploads/allimg/151012/1FA2TZ_0.png)

# 后记

对电量优化，做了一个初步了解，平时针对电量的优化操作很少去留意的，因为这个的优先级还不是很高哈。

原本应该继续写点关于dalvik的内存方面的，不过目前我知道还是理论上的比较多。
要从源码角度看还是得缓下，这几天继续看下先看有什么灵感，或者搜刮下有什么参考资料再写吧