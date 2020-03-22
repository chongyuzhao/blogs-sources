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

