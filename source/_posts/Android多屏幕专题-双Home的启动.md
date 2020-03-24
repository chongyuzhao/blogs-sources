---
title: Android多屏幕专题--双Home的启动
date: 2020-03-22 10:54:36
tags:
- Android
- AMS
---

# Android多屏幕专题 -- 双Home的启动

​	在一个偶然的机会下，发现AndroidQ是支持两个Launcher的，在两个显示设备的情况下，可以分别启动两个Launcher，在这两个Launcher中都可以操作，如打开应用，虚拟导航栏等，也就是说，使用这种模式，是可以同时打开两个应用分别操作。

​	启动双Home的方式，首先是这个Android系统不能是低内存设备(low ram device)，其次可通过`settings put global force_desktop_mode_on_external_displays 1`，重启后两个显示器上就有不同的Home了。

​	下面我们就来看看这个settings字段是如何控制双Home的启动。前面的代码追溯我们就不一一展开，直接从SystemServer启动完成后就会调用AMS的systemReady方法，而在AMS的systemReady中就会去启动我们的Launcher:

```java
 8960     public void systemReady(final Runnable goingCallback, TimingsTraceLog traceLog) {
     	……
 9076             mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");
     	……
 9136     }

6691         @Override
6692         public boolean startHomeOnAllDisplays(int userId, String reason) {
6693             synchronized (mGlobalLock) {
6694                 return mRootActivityContainer.startHomeOnAllDisplays(userId, reason);
6695             }
6696         }

 332     boolean startHomeOnAllDisplays(int userId, String reason) {
 333         boolean homeStarted = false;
 334         for (int i = mActivityDisplays.size() - 1; i >= 0; i--) {
 335             final int displayId = mActivityDisplays.get(i).mDisplayId;
 336             homeStarted |= startHomeOnDisplay(userId, reason, displayId);
 337         }
 338         return homeStarted;
 339     }
```

## startHomeOnDisplay

​	mActivityDisplays中记录了启动过程中所注册进来的display，在AMS中，使用ActivityDisplay来描述显示屏，这里的`startHomeOnAllDisplays`就是去为所有的显示屏去启动一个Home，当然啦，并不是一定会为每个屏幕都启动一个Home.

```java
 350     boolean startHomeOnDisplay(int userId, String reason, int displayId) {
 351         return startHomeOnDisplay(userId, reason, displayId, false /* allowInstrumenting */,
 352                 false /* fromHomeKey */);
 353     }
 366     boolean startHomeOnDisplay(int userId, String reason, int displayId, boolean allowInstrumenting,
 367             boolean fromHomeKey) {
 368         // Fallback to top focused display if the displayId is invalid.
 369         if (displayId == INVALID_DISPLAY) {
 370             displayId = getTopDisplayFocusedStack().mDisplayId;
 371         }
 372
 373         Intent homeIntent = null;
 374         ActivityInfo aInfo = null;
 375         if (displayId == DEFAULT_DISPLAY) {
 376             homeIntent = mService.getHomeIntent();
 377             aInfo = resolveHomeActivity(userId, homeIntent);
 378         } else if (shouldPlaceSecondaryHomeOnDisplay(displayId)) {
 379             Pair<ActivityInfo, Intent> info = resolveSecondaryHomeActivity(userId, displayId);
 380             aInfo = info.first;
 381             homeIntent = info.second;
 382         }
 383         if (aInfo == null || homeIntent == null) {
 384             return false;
 385         }
 386
 387         if (!canStartHomeOnDisplay(aInfo, displayId, allowInstrumenting)) {
 388             return false;
 389         }
 390
 391         // Updates the home component of the intent.
 392         homeIntent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
 393         homeIntent.setFlags(homeIntent.getFlags() | FLAG_ACTIVITY_NEW_TASK);
 394         // Updates the extra information of the intent.
 395         if (fromHomeKey) {
 396             homeIntent.putExtra(WindowManagerPolicy.EXTRA_FROM_HOME_KEY, true);
 397         }
 398         // Update the reason for ANR debugging to verify if the user activity is the one that
 399         // actually launched.
 400         final String myReason = reason + ":" + userId + ":" + UserHandle.getUserId(
 401                 aInfo.applicationInfo.uid) + ":" + displayId;
 402         mService.getActivityStartController().startHomeActivity(homeIntent, aInfo, myReason,
 403                 displayId);
 404         return true;
 405     }
```

​	当displayId等于DEFAULT_DISPLAY，即主显的时候，必然会给其找一个Launcher进行启动。当displayId不是DEFAULT_DISPLAY，即非主显时，就要过三关了：

​	第一关，系统是否支持SecondaryHomeOnDisplay--`shouldPlaceSecondaryHomeOnDisplay`，如果系统没有配置打开，那么AMS就不会启动双Home了。

​	第二关，系统中有定义SecondaryHomeActivity--`resolveSecondaryHomeActivity`，如果没有第二个Launcher，那么也就没有可以使用的Activity，AMS根本就没法启动。

​	第三关，检查是否允许在对应的display中启动Home--`canStartHomeOnDisplay`，如果是AMS在启动的时候，是不会有问题的。

### shouldPlaceSecondaryHomeOnDisplay

​	先看第一关考验的内容：

```java
 540     boolean shouldPlaceSecondaryHomeOnDisplay(int displayId) {
 541         if (displayId == DEFAULT_DISPLAY) {
 542             throw new IllegalArgumentException(
 543                     "shouldPlaceSecondaryHomeOnDisplay: Should not be DEFAULT_DISPLAY");
 544         } else if (displayId == INVALID_DISPLAY) {
 545             return false;
 546         }
 547
 548         if (!mService.mSupportsMultiDisplay) {
 549             // Can't launch home on secondary display if device does not support multi-display.
 550             return false;
 551         }
 552
 553         final boolean deviceProvisioned = Settings.Global.getInt(
 554                 mService.mContext.getContentResolver(),
 555                 Settings.Global.DEVICE_PROVISIONED, 0) != 0;
 556         if (!deviceProvisioned) {
 557             // Can't launch home on secondary display before device is provisioned.
 558             return false;
 559         }
 560
 561         if (!StorageManager.isUserKeyUnlocked(mCurrentUser)) {
 562             // Can't launch home on secondary displays if device is still locked.
 563             return false;
 564         }
 565
 566         final ActivityDisplay display = getActivityDisplay(displayId);
 567         if (display == null || display.isRemoved() || !display.supportsSystemDecorations()) {
 568             // Can't launch home on display that doesn't support system decorations.
 569             return false;
 570         }
 571
 572         return true;
 573     }
```


​	`mService.mSupportsMultiDisplay`是指系统是否支持多屏幕，其决定因素有：

```java
 721     public void retrieveSettings(ContentResolver resolver) {
 722         final boolean freeformWindowManagement =
 723                 mContext.getPackageManager().hasSystemFeature(FEATURE_FREEFORM_WINDOW_MANAGEMENT)
 724                         || Settings.Global.getInt(
 725                         resolver, DEVELOPMENT_ENABLE_FREEFORM_WINDOWS_SUPPORT, 0) != 0;
 726
 727         final boolean supportsMultiWindow = ActivityTaskManager.supportsMultiWindow(mContext);
 728         final boolean supportsPictureInPicture = supportsMultiWindow &&
 729                 mContext.getPackageManager().hasSystemFeature(FEATURE_PICTURE_IN_PICTURE);
 730         final boolean supportsSplitScreenMultiWindow =
 731                 ActivityTaskManager.supportsSplitScreenMultiWindow(mContext);
 732         final boolean supportsMultiDisplay = mContext.getPackageManager()
 733                 .hasSystemFeature(FEATURE_ACTIVITIES_ON_SECONDARY_DISPLAYS);
 734         final boolean forceRtl = Settings.Global.getInt(resolver, DEVELOPMENT_FORCE_RTL, 0) != 0;
 735         final boolean forceResizable = Settings.Global.getInt(
 736                 resolver, DEVELOPMENT_FORCE_RESIZABLE_ACTIVITIES, 0) != 0;
 737         final boolean isPc = mContext.getPackageManager().hasSystemFeature(FEATURE_PC);
 738
 739         // Transfer any global setting for forcing RTL layout, into a System Property
 740         DisplayProperties.debug_force_rtl(forceRtl);
 741
 742         final Configuration configuration = new Configuration();
 743         Settings.System.getConfiguration(resolver, configuration);
 744         if (forceRtl) {
 745             // This will take care of setting the correct layout direction flags
 746             configuration.setLayoutDirection(configuration.locale);
 747         }
 748
 749         synchronized (mGlobalLock) {
 750             mForceResizableActivities = forceResizable;
 751             final boolean multiWindowFormEnabled = freeformWindowManagement
 752                     || supportsSplitScreenMultiWindow
 753                     || supportsPictureInPicture
 754                     || supportsMultiDisplay;
 755             if ((supportsMultiWindow || forceResizable) && multiWindowFormEnabled) {
 756                 mSupportsMultiWindow = true;
 757                 mSupportsFreeformWindowManagement = freeformWindowManagement;
 758                 mSupportsSplitScreenMultiWindow = supportsSplitScreenMultiWindow;
 759                 mSupportsPictureInPicture = supportsPictureInPicture;
 760                 mSupportsMultiDisplay = supportsMultiDisplay;
 761             } else {
 762                 mSupportsMultiWindow = false;
 763                 mSupportsFreeformWindowManagement = false;
 764                 mSupportsSplitScreenMultiWindow = false;
 765                 mSupportsPictureInPicture = false;
 766                 mSupportsMultiDisplay = false;
 767             }
     		………
```

​	从代码中可以看到，supportsMultiDisplay要为true，那么就需要满足以下的条件：①.系统有配置`android.software.activities_on_secondary_displays`这个feature，如果是LowRaw设备，那么在配置中必须要加上`notLowRaw=true`这个属性。②.支持多窗口模式--supportsMultiWindow(此次成立的条件是非LowRam设备或者是watch设备，且配置了`config_supportsSplitScreenMultiWindow`)或者系统支持窗口大小调整--forceResizable(settings global中设置`force_resizable_activities`为1).③.使能多窗口模式 -- multiWindowFormEnabled，其实这条件在前面的条件成立后，自然也就成立。

​	在supportsMultiDisplay为true，后第二步就是要deviceProvisioned为true，这个值是从settings global中获取，而这个值是在开机向导(Provisions)结束后就会设置为true，这条件一般也会成立，我们无需在意。

​	第三步就是系统处于非锁屏状态，这也很好理解，当处于锁屏状态，Home是还没起来的。

​	第四步就是所传进来的Display是有效的且没有被移除，然后满足`isplay.supportsSystemDecorations()`返回true：

```java
1209     boolean supportsSystemDecorations() {
1210         return mDisplayContent.supportsSystemDecorations();
1211     }
5023     boolean supportsSystemDecorations() {
5024         return (mWmService.mDisplayWindowSettings.shouldShowSystemDecorsLocked(this)
5025                 || (mDisplay.getFlags() & FLAG_SHOULD_SHOW_SYSTEM_DECORATIONS) != 0
5026                 || (mWmService.mForceDesktopModeOnExternalDisplays && !isUntrustedVirtualDisplay()))
5027                 // VR virtual display will be used to run and render 2D app within a VR experience.
5028                 && mDisplayId != mWmService.mVr2dDisplayId;
5029     }
```

​	supportsSystemDecorations()返回true满足的条件：①.该Display不是2D的VR屏。②.满足以下条件之一：

​	一.mWmService.mDisplayWindowSettings.shouldShowSystemDecorsLocked()返回true：

```java
343     boolean shouldShowSystemDecorsLocked(DisplayContent dc) {
344         if (dc.getDisplayId() == Display.DEFAULT_DISPLAY) {
345             // For default display should show system decors.
346             return true;
347         }
348
349         final DisplayInfo displayInfo = dc.getDisplayInfo();
350         final Entry entry = getEntry(displayInfo);
351         if (entry == null) {
352             return false;
353         }
354         return entry.mShouldShowSystemDecors;
355     }
```

​	目前从Android的源码中没有看过设置这个值的地方，我们先跳过。

​	二.Display具备FLAG_SHOULD_SHOW_SYSTEM_DECORATIONS这个flags，也没有从代码中看到相关的设置，也跳过。

​	三.`mWmService.mForceDesktopModeOnExternalDisplays`为true，且非不可信的虚拟屏，后者一般为true，前者就是我们最开始说的`settings put global force_desktop_mode_on_external_displays 1`：

```java
 999     private WindowManagerService(Context context, InputManagerService inputManager,
1000             boolean showBootMsgs, boolean onlyCore, WindowManagerPolicy policy,
1001             ActivityTaskManagerService atm, TransactionFactory transactionFactory) {
		……
1119         mForceDesktopModeOnExternalDisplays = Settings.Global.getInt(resolver,
1120                 DEVELOPMENT_FORCE_DESKTOP_MODE_ON_EXTERNAL_DISPLAYS, 0) != 0;
     	……
```

​	至此第一关就闯关成功。

### resolveSecondaryHomeActivity

​	第二关就准备开始了。

```
 443     @VisibleForTesting
 444     Pair<ActivityInfo, Intent> resolveSecondaryHomeActivity(int userId, int displayId) {
 445         if (displayId == DEFAULT_DISPLAY) {
 446             throw new IllegalArgumentException(
 447                     "resolveSecondaryHomeActivity: Should not be DEFAULT_DISPLAY");
 448         }
 449         // Resolve activities in the same package as currently selected primary home activity.
 450         Intent homeIntent = mService.getHomeIntent();
 451         ActivityInfo aInfo = resolveHomeActivity(userId, homeIntent);
 452         if (aInfo != null) {
 453             if (ResolverActivity.class.getName().equals(aInfo.name)) {
 454                 // Always fallback to secondary home component if default home is not set.
 455                 aInfo = null;
 456             } else {
 457                 // Look for secondary home activities in the currently selected default home
 458                 // package.
 459                 homeIntent = mService.getSecondaryHomeIntent(aInfo.applicationInfo.packageName);
 460                 final List<ResolveInfo> resolutions = resolveActivities(userId, homeIntent);
 461                 final int size = resolutions.size();
 462                 final String targetName = aInfo.name;
 463                 aInfo = null;
 464                 for (int i = 0; i < size; i++) {
 465                     ResolveInfo resolveInfo = resolutions.get(i);
 466                     // We need to traverse all resolutions to check if the currently selected
 467                     // default home activity is present.
 468                     if (resolveInfo.activityInfo.name.equals(targetName)) {
 469                         aInfo = resolveInfo.activityInfo;
 470                         break;
 471                     }
 472                 }
 473                 if (aInfo == null && size > 0) {
 474                     // First one is the best.
 475                     aInfo = resolutions.get(0).activityInfo;
 476                 }
 477             }
 478         }
 479
 480         if (aInfo != null) {
 481             if (!canStartHomeOnDisplay(aInfo, displayId, false /* allowInstrumenting */)) {
 482                 aInfo = null;
 483             }
 484         }
 485
 486         // Fallback to secondary home component.
 487         if (aInfo == null) {
 488             homeIntent = mService.getSecondaryHomeIntent(null);
 489             aInfo = resolveHomeActivity(userId, homeIntent);
 490         }
 491         return Pair.create(aInfo, homeIntent);
 492     }
```

​	第二关的核心方法就是`mService.getSecondaryHomeIntent`，其中从上面的代码中看到，有两种情况，传入了两个不同的参数，第一个是已经启动的Home所在的包名，第二个是null.

```
5869     Intent getSecondaryHomeIntent(String preferredPackage) {
5870         final Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
5871         final boolean useSystemProvidedLauncher = mContext.getResources().getBoolean(
5872                 com.android.internal.R.bool.config_useSystemProvidedLauncherForSecondary);
5873         if (preferredPackage == null || useSystemProvidedLauncher) {
5874             // Using the component stored in config if no package name or forced.
5875             final String secondaryHomeComponent = mContext.getResources().getString(
5876                     com.android.internal.R.string.config_secondaryHomeComponent);
5877             intent.setComponent(ComponentName.unflattenFromString(secondaryHomeComponent));
5878         } else {
5879             intent.setPackage(preferredPackage);
5880         }
5881         intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
5882         if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
5883             intent.addCategory(Intent.CATEGORY_SECONDARY_HOME);
5884         }
5885         return intent;
5886     }
```

​	如果将`config_useSystemProvidedLauncherForSecondary`配置为true，那么就会使用`config_secondaryHomeComponent`所配置的Home，这种做法可用于指定特定的SecondaryHome。但`config_useSystemProvidedLauncherForSecondary`默认是false，也就是会根据已启动的Home所在的包中的`CATEGORY_SECONDARY_HOME`定义的Home，因为一些Launcher会有支持上SecondaryHome，当launcher中找不到SecondaryHome，才会去使用默认的SecondaryHome。第二种情况是主显的launcher没有启动时走的流程，也就是使用默认的SecondaryHome。

​	至此就完成了查找SecondaryHome的任务。



### canStartHomeOnDisplay

```java
 582     boolean canStartHomeOnDisplay(ActivityInfo homeInfo, int displayId,
 583             boolean allowInstrumenting) {
 584         if (mService.mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL
 585                 && mService.mTopAction == null) {
 586             // We are running in factory test mode, but unable to find the factory test app, so
 587             // just sit around displaying the error message and don't try to start anything.
 588             return false;
 589         }
 590
 591         final WindowProcessController app =
 592                 mService.getProcessController(homeInfo.processName, homeInfo.applicationInfo.uid);
 593         if (!allowInstrumenting && app != null && app.isInstrumenting()) {
 594             // Don't do this if the home app is currently being instrumented.
 595             return false;
 596         }
 597
 598         if (displayId == DEFAULT_DISPLAY || (displayId != INVALID_DISPLAY
 599                 && displayId == mService.mVr2dDisplayId)) {
 600             // No restrictions to default display or vr 2d display.
 601             return true;
 602         }
 603
 604         if (!shouldPlaceSecondaryHomeOnDisplay(displayId)) {
 605             return false;
 606         }
 607
 608         final boolean supportMultipleInstance = homeInfo.launchMode != LAUNCH_SINGLE_TASK
 609                 && homeInfo.launchMode != LAUNCH_SINGLE_INSTANCE;
 610         if (!supportMultipleInstance) {
 611             // Can't launch home on secondary displays if it requested to be single instance.
 612             return false;
 613         }
 614
 615         return true;
 616     }
```

​	`canStartHomeOnDisplay`的核心是`shouldPlaceSecondaryHomeOnDisplay`，而这个就是我们第一关的内容，因此毫无疑问，这关轻松闯过。

### startHomeActivity

​	闯完三关后，就到了真正的启动，从前面的参数可以知道，HomeActivity的启动必定加上`FLAG_ACTIVITY_NEW_TASK`这个flag参数，表示HomeActivity是要在新的Task中启动的，这很容易理解。

​	我们继续看`startHomeActivity`是如何将Home启动到副屏上的：

```java
171     void startHomeActivity(Intent intent, ActivityInfo aInfo, String reason, int displayId) {
172         final ActivityOptions options = ActivityOptions.makeBasic();
173         options.setLaunchWindowingMode(WINDOWING_MODE_FULLSCREEN);
174         if (!ActivityRecord.isResolverActivity(aInfo.name)) {
175             // The resolver activity shouldn't be put in home stack because when the foreground is
176             // standard type activity, the resolver activity should be put on the top of current
177             // foreground instead of bring home stack to front.
178             options.setLaunchActivityType(ACTIVITY_TYPE_HOME);
179         }
180         options.setLaunchDisplayId(displayId);
181         mLastHomeActivityStartResult = obtainStarter(intent, "startHomeActivity: " + reason)
182                 .setOutActivity(tmpOutRecord)
183                 .setCallingUid(0)
184                 .setActivityInfo(aInfo)
185                 .setActivityOptions(options.toBundle())
186                 .execute();
187         mLastHomeActivityStartRecord = tmpOutRecord[0];
188         final ActivityDisplay display =
189                 mService.mRootActivityContainer.getActivityDisplay(displayId);
190         final ActivityStack homeStack = display != null ? display.getHomeStack() : null;
191         if (homeStack != null && homeStack.mInResumeTopActivity) {
192             // If we are in resume section already, home activity will be initialized, but not
193             // resumed (to avoid recursive resume) and will stay that way until something pokes it
194             // again. We need to schedule another resume.
195             mSupervisor.scheduleResumeTopActivities();
196         }
197     }
```

​	显示到副屏的关键是--`options.setLaunchDisplayId(displayId)`，ActivityOptions是Activity的启动参数，setLaunchDisplayId就是设置了Activity在哪个display上进行启动。而这个详细的过程可以看am命令在副屏上启动Activity的详细分析。



## 总结

​	总的来说，AndroidQ已经支持得十分好，如果需要打开双Home启动，那么可以通过以下步骤去进行。

​	1.最好将设备设置为非LowRam设备。

​	2.打开多窗口模式，如果是非LowRam设备，默认也是打开的。

​	3.准备一个SecondaryHome应用，Android提供的Launcher3其实已经支持，但也可以自行配置。

​	4.通过`settings put global force_desktop_mode_on_external_displays 1`去打开双Home。