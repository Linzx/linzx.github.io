---
layout: post
title:  "binderDied()过程分析"
date:   2016-10-02 20:12:50
catalog:  true
tags:
    - android
    - 组件

---

## 一. 概述

[理解Android进程启动之全过程](http://gityuan.com/2016/10/09/app-process-create-2/)介绍了进程是如何从AMS.startProcessLocked一步步创建的; 当进程不再需要时便会有杀进程的过程; [理解杀进程的实现原理](http://gityuan.com/2016/04/16/kill-signal/)介绍了Process.killProcess()如何一步步地将进程杀死.

当系统内存不足时,会触发[lmk](http://gityuan.com/2016/09/17/android-lowmemorykiller/)杀进程; 以及系统本身通过AMS也会控制系统中各个状态的进程个数上限. 当进程真正的被杀死之后,通过binder死亡回调后系统需要清理四大组件和进程信息.

### 1.1 死亡监听

当进程死亡后, 系统是如何知道的呢, 答案就在进程创建之后,会调用AMS.attachApplicationLocked()

    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
        ...
        try {
            //绑定死亡通知,此处thread真实数据类型为ApplicationThreadProxy
            AppDeathRecipient adr = new AppDeathRecipient(app, pid, thread);
            thread.asBinder().linkToDeath(adr, 0);
            app.deathRecipient = adr;
        } catch (RemoteException e) {
            ...
        }
        ...
    }

在这个过程中,会创建AppDeathRecipient死亡通告对象,通过binder机制绑定, 当新创建的应用进程死亡后,便会回调binderDied()方法. 关于[binder](http://gityuan.com/2015/10/31/binder-prepare/)前面有大量文章深入介绍过. 其中关于binder死亡回调,是指binder server端挂了之后通过binder driver会通知binder client端. 那么对于进程死亡过程, binder server端是指应用进程的ApplicationThread, binder client端是指system_server进程中的ApplicationThreadProxy对象. 接下来从binderDied()方法说起.


## 二. BinderDied流程

### 2.1 binderDied
[-> ActivityManagerService.java]

    private final class AppDeathRecipient implements IBinder.DeathRecipient {
        final ProcessRecord mApp;
        final int mPid;
        final IApplicationThread mAppThread;

        AppDeathRecipient(ProcessRecord app, int pid,
                IApplicationThread thread) {
            mApp = app;
            mPid = pid;
            mAppThread = thread;
        }

        public void binderDied() {
            synchronized(ActivityManagerService.this) {
                //[见流程2.2]
                appDiedLocked(mApp, mPid, mAppThread, true);
            }
        }
    }

当进程死亡后便会回调binderDied()方法.该方法是由ActivityManagerService对象锁保护.

### 2.2 AMS.appDiedLocked
[-> ActivityManagerService.java]

    final void appDiedLocked(ProcessRecord app, int pid, IApplicationThread thread,
            boolean fromBinderDied) {
        //检查pid与app是否匹配,不匹配则直接返回
        synchronized (mPidsSelfLocked) {
            ProcessRecord curProc = mPidsSelfLocked.get(pid);
            if (curProc != app) {
                return;
            }
        }
        ...
        //当进程还没有设置已被杀的标记,则进入该分支杀掉相应进程
        if (!app.killed) {
            //非binder死亡回调,而是上层直接调用该方法,则进入该分支
            if (!fromBinderDied) {
                Process.killProcessQuiet(pid);
            }
            killProcessGroup(app.info.uid, pid);
            app.killed = true;
        }

        if (app.pid == pid && app.thread != null &&
                app.thread.asBinder() == thread.asBinder()) {
            //一般来说为true
            boolean doLowMem = app.instrumentationClass == null;
            boolean doOomAdj = doLowMem;
            boolean homeRestart = false;
            if (!app.killedByAm) {
                //当app不是由am所杀,则往往都是lmk所杀
                if (mHomeProcessName != null && app.processName.equals(mHomeProcessName)) {
                    mHomeKilled = true;
                    homeRestart = true;
                }
                //既然是由lmk所杀,说明当时内存比较紧张,这时希望能
                mAllowLowerMemLevel = true;
            } else {
                mAllowLowerMemLevel = false;
                doLowMem = false;
            }

            //从ams移除该进程以及connections [见流程2.3]
            handleAppDiedLocked(app, false, true);

            //一般为true，则需要更新各个进程的adj
            if (doOomAdj) {
                updateOomAdjLocked();
            }

            //当进程是由lmkd所杀,则进入该分支
            if (doLowMem) {
                //只有当mLruProcesses中所有进程都运行在前台,才报告内存信息
                doLowMemReportIfNeededLocked(app);
            }
            if (mHomeKilled && homeRestart) {
                Intent intent = getHomeIntent();
                //根据intent解析相应的home activity信息
                ActivityInfo aInfo = mStackSupervisor.resolveActivity(intent, null, 0, null, 0);
                //当桌面被杀,则立马再次启动桌面进程
                startProcessLocked(aInfo.processName, aInfo.applicationInfo, true, 0,
                        "activity", null, false, false, true);
                homeRestart = false;
            }
        }
    }


- `mPidsSelfLocked`: 数据类型为SparseArray<ProcessRecord>, 以pid为key, ProcessRecord为value的结构体.
- app.killed: 当进程是由AMS杀掉的则killed=true, 当进程是由底层lmkd所杀或者是底层signal 9信号所杀则killed=false;
- fromBinderDied: 当appDiedLocked()是由binderDied()回调的则fromBinderDied=true;否则是由启动activity/service/provider过程中遇到DeadObjectException或者RemoteException等情况下也会直接调用appDiedLocked(),那么此时fromBinderDied=false.
- `mLruProcesses`是一个lru队列的进程信息, 队列中第一个成员便是最近最少使用的进程.

### 2.3 AMS.handleAppDiedLocked
[-> ActivityManagerService.java]

    // restarting = false,  allowRestart = true
    private final void handleAppDiedLocked(ProcessRecord app,
            boolean restarting, boolean allowRestart) {
        int pid = app.pid;
        //清理应用程序service, BroadcastReceiver, ContentProvider相关信息【见小节2.4】
        boolean kept = cleanUpApplicationRecordLocked(app, restarting, allowRestart, -1);

        //当应用不需要保持(即不需要重启则不保持), 且不处于正在启动中的状态,
        //则从mLruProcesses移除该应用,以及告诉lmk该pid被移除的信息
        if (!kept && !restarting) {
            removeLruProcessLocked(app);
            if (pid > 0) {
                ProcessList.remove(pid);
            }
        }

        //清理activity相关信息, 当应用存在可见的activity则返回true [见小节2.5]
        boolean hasVisibleActivities = mStackSupervisor.handleAppDiedLocked(app);
        app.activities.clear();
        ...
        //当死亡的app存在可见的Activity, 则恢复栈顶第一个非finish的activity
        if (!restarting && hasVisibleActivities && !mStackSupervisor.resumeTopActivitiesLocked()) {
           //恢复top activity失败,则再次确保有可见的activity
           mStackSupervisor.ensureActivitiesVisibleLocked(null, 0);
       }
    }

**主要功能:**

- AMS.cleanUpApplicationRecordLocked: 清理应用程序service, BroadcastReceiver, ContentProvider
- ASS.handleAppDiedLocked: 清理activity相关信息, 当应用存在可见的activity则返回true
- 当并没有处于正在启动,且死亡的app存在可见的Activity时, 则调用resumeTopActivitiesLocked恢复栈顶第一个非finish的activity,如果还失败,再调用ensureActivitiesVisibleLocked再次确保有可见的activity. 关于这里的方法,可查看文章[startActivity启动过程分析](http://gityuan.com/2016/03/12/start-activity/)的小节[2.11].


另外,handleAppDiedLocked()只在AMS的如下方法调用:

(1)allowRestart = true的调用情况

- attachApplicationLocked
- startProcessLocked
- appDiedLocked
- removeProcessLocked
    - killAllBackgroundProcesses
    - killPackageProcessesLocked
    - processContentProviderPublishTimedOutLocked

(2)allowRestart = false的调用情况

- removeProcessLocked
    - handleAppCrashLocked
    - systemReady

也就是说只有在发生app crash或者是系统刚启动时systemReady的过程调用handleAppDiedLocked()是不允许重启的,其他case下都是运行重启的.

### 2.4 cleanUpApplicationRecordLocked

该方法清理应用程序service, BroadcastReceiver, ContentProvider，process相关信息，为了便于说明将该方法划分为4个部分讲解.
参数:restarting = false,  allowRestart =true,  index =-1

#### part 1 清理service


    private final boolean cleanUpApplicationRecordLocked(ProcessRecord app,
            boolean restarting, boolean allowRestart, int index) {
        ...
        mProcessesToGc.remove(app);
        mPendingPssProcesses.remove(app);

        //如果存在，则清除crash/anr/wait对话框
        if (app.crashDialog != null && !app.forceCrashReport) {
            app.crashDialog.dismiss();
            app.crashDialog = null;
        }
        if (app.anrDialog != null) {
            app.anrDialog.dismiss();
            app.anrDialog = null;
        }
        if (app.waitDialog != null) {
            app.waitDialog.dismiss();
            app.waitDialog = null;
        }

        app.crashing = false;
        app.notResponding = false;

        app.resetPackageList(mProcessStats); //重置该进程下的所有包名列表
        app.unlinkDeathRecipient(); //解除app的死亡通告
        app.makeInactive(mProcessStats);
        app.waitingToKill = null;
        app.forcingToForeground = null;

        updateProcessForegroundLocked(app, false, false); //将app移除前台进程
        app.foregroundActivities = false;
        app.hasShownUi = false;
        app.treatLikeActivity = false;
        app.hasAboveClient = false;
        app.hasClientActivities = false;
        //清理service信息，这个过程也比较复杂，后续再展开
        mServices.killServicesLocked(app, allowRestart);
        boolean restart = false;
    }

- mProcessesToGc：记录着需要尽快执行gc的进程列表
- mPendingPssProcesses：记录着需要收集内存信息的进程列表

#### part 1-1 ActiveServices.killServicesLocked
[-> ActiveServices.java]

    final void killServicesLocked(ProcessRecord app, boolean allowRestart) {
          //移除应用跟其他service的所有连接对象
         for (int i = app.connections.size() - 1; i >= 0; i--) {
             ConnectionRecord r = app.connections.valueAt(i);
             removeConnectionLocked(r, app, null);
         }
         updateServiceConnectionActivitiesLocked(app);
         app.connections.clear();

         //清空应用service状态
         for (int i = app.services.size() - 1; i >= 0; i--) {
             ServiceRecord sr = app.services.valueAt(i);
             ...
             if (sr.app != app && sr.app != null && !sr.app.persistent) {
                 sr.app.services.remove(sr);
             }
             sr.app = null;
             sr.isolatedProc = null;
             sr.executeNesting = 0;
             sr.forceClearTracker();
             mDestroyingServices.remove(sr);

             final int numClients = sr.bindings.size();
             for (int bindingi=numClients-1; bindingi>=0; bindingi--) {
                 IntentBindRecord b = sr.bindings.valueAt(bindingi);

                 b.binder = null;
                 b.requested = b.received = b.hasBound = false;

                 for (int appi=b.apps.size()-1; appi>=0; appi--) {

                     if (proc.killedByAm || proc.thread == null) {
                         continue;
                     }

                     final AppBindRecord abind = b.apps.valueAt(appi);
                     boolean hasCreate = false;
                     for (int conni=abind.connections.size()-1; conni>=0; conni--) {
                         ConnectionRecord conn = abind.connections.valueAt(conni);
                         if ((conn.flags&(Context.BIND_AUTO_CREATE|Context.BIND_ALLOW_OOM_MANAGEMENT
                                 |Context.BIND_WAIVE_PRIORITY)) == Context.BIND_AUTO_CREATE) {
                             hasCreate = true;
                             break;
                         }
                     }
                     if (!hasCreate) {
                         continue;
                     }

                 }
             }
         }

         ServiceMap smap = getServiceMap(app.userId);

         for (int i=app.services.size()-1; i>=0; i--) {
             ServiceRecord sr = app.services.valueAt(i);

             if (!app.persistent) {
                 app.services.removeAt(i);
             }

             final ServiceRecord curRec = smap.mServicesByName.get(sr.name);
             if (curRec != sr) {
                 continue;
             }

             if (allowRestart && sr.crashCount >= 2 && (sr.serviceInfo.applicationInfo.flags
                     &ApplicationInfo.FLAG_PERSISTENT) == 0) {
                 bringDownServiceLocked(sr);
             } else if (!allowRestart || !mAm.isUserRunningLocked(sr.userId, false)) {
                 bringDownServiceLocked(sr);
             } else {
                 //allowRestart=true 则会触发重启启动service
                 boolean canceled = scheduleServiceRestartLocked(sr, true);

                 if (sr.startRequested && (sr.stopIfKilled || canceled)) {
                     if (sr.pendingStarts.size() == 0) {
                         sr.startRequested = false;
                         if (sr.tracker != null) {
                             sr.tracker.setStarted(false, mAm.mProcessStats.getMemFactorLocked(),
                                     SystemClock.uptimeMillis());
                         }
                         if (!sr.hasAutoCreateConnections()) {
                             bringDownServiceLocked(sr);
                         }
                     }
                 }
             }
         }

         if (!allowRestart) {
             app.services.clear();

             for (int i=mRestartingServices.size()-1; i>=0; i--) {
                 ServiceRecord r = mRestartingServices.get(i);
                 if (r.processName.equals(app.processName) &&
                         r.serviceInfo.applicationInfo.uid == app.info.uid) {
                     mRestartingServices.remove(i);
                     clearRestartingIfNeededLocked(r);
                 }
             }
             for (int i=mPendingServices.size()-1; i>=0; i--) {
                 ServiceRecord r = mPendingServices.get(i);
                 if (r.processName.equals(app.processName) &&
                         r.serviceInfo.applicationInfo.uid == app.info.uid) {
                     mPendingServices.remove(i);
                 }
             }
         }

         int i = mDestroyingServices.size();
         while (i > 0) {
             i--;
             ServiceRecord sr = mDestroyingServices.get(i);
             if (sr.app == app) {
                 sr.forceClearTracker();
                 mDestroyingServices.remove(i);
             }
         }

         app.executingServices.clear();
     }


#### part 2 清理ContentProvider

    private final boolean cleanUpApplicationRecordLocked(...) {
        ...
        for (int i = app.pubProviders.size() - 1; i >= 0; i--) {
            //获取该进程已发表的ContentProvider
            ContentProviderRecord cpr = app.pubProviders.valueAt(i);
            // allowRestart=true，一般地always=false
            final boolean always = app.bad || !allowRestart;
            //ContentProvider服务端被杀，则client端进程也会被杀
            boolean inLaunching = removeDyingProviderLocked(app, cpr, always);
            if ((inLaunching || always) && cpr.hasConnectionOrHandle()) {
                restart = true; //需要重启
            }

            cpr.provider = null;
            cpr.proc = null;
        }
        app.pubProviders.clear();

        //处理正在启动并且是有client端正在等待的ContentProvider
        if (cleanupAppInLaunchingProvidersLocked(app, false)) {
            restart = true;
        }

        //取消已连接的ContentProvider的注册
        if (!app.conProviders.isEmpty()) {
            for (int i = app.conProviders.size() - 1; i >= 0; i--) {
                ContentProviderConnection conn = app.conProviders.get(i);
                conn.provider.connections.remove(conn);

                stopAssociationLocked(app.uid, app.processName, conn.provider.uid,
                        conn.provider.name);
            }
            app.conProviders.clear();
        }

#### part 3 清理BroadcastReceiver

    private final boolean cleanUpApplicationRecordLocked(...) {
        ...
        skipCurrentReceiverLocked(app);

        // 取消注册的广播接收者
        for (int i = app.receivers.size() - 1; i >= 0; i--) {
            removeReceiverLocked(app.receivers.valueAt(i));
        }
        app.receivers.clear();
    }

#### part 4 清理Process

    private final boolean cleanUpApplicationRecordLocked(...) {
        ...
        //当app正在备份时的处理方式
        if (mBackupTarget != null && app.pid == mBackupTarget.app.pid) {
            IBackupManager bm = IBackupManager.Stub.asInterface(
                    ServiceManager.getService(Context.BACKUP_SERVICE));
            bm.agentDisconnected(app.info.packageName);
        }

        for (int i = mPendingProcessChanges.size() - 1; i >= 0; i--) {
            ProcessChangeItem item = mPendingProcessChanges.get(i);
            if (item.pid == app.pid) {
                mPendingProcessChanges.remove(i);
                mAvailProcessChanges.add(item);
            }
        }
        mUiHandler.obtainMessage(DISPATCH_PROCESS_DIED, app.pid, app.info.uid, null).sendToTarget();
        //此时为false，不进入该分支
        if (restarting) {
            return false;
        }

        if (!app.persistent || app.isolated) {
            removeProcessNameLocked(app.processName, app.uid);
            if (mHeavyWeightProcess == app) {
                mHandler.sendMessage(mHandler.obtainMessage(CANCEL_HEAVY_NOTIFICATION_MSG,
                        mHeavyWeightProcess.userId, 0));
                mHeavyWeightProcess = null;
            }
        } else if (!app.removed) {
            //对于persistent应用，则需要重启
            if (mPersistentStartingProcesses.indexOf(app) < 0) {
                mPersistentStartingProcesses.add(app);
                restart = true;
            }
        }

        //mProcessesOnHold：记录着试图在系统ready之前就启动的进程。
        //在那时并不启动这些进程，先记录下来,等系统启动完成则启动这些进程。
        mProcessesOnHold.remove(app);

        if (app == mHomeProcess) {
            mHomeProcess = null;
        }
        if (app == mPreviousProcess) {
            mPreviousProcess = null;
        }

        if (restart && !app.isolated) {
            //仍有组件需要运行在该进程中，因此重启该进程
            if (index < 0) {
                ProcessList.remove(app.pid);
            }
            addProcessNameLocked(app);
            startProcessLocked(app, "restart", app.processName);
            return true;
        } else if (app.pid > 0 && app.pid != MY_PID) {
            //移除该进程相关信息
            boolean removed;
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.remove(app.pid);
                mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
            }
            app.setPid(0);
        }
        return false;
    }

### 2.5  ASS.handleAppDiedLocked

    boolean handleAppDiedLocked(ProcessRecord app) {
        boolean hasVisibleActivities = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                //[见流程2.6]
                hasVisibleActivities |= stacks.get(stackNdx).handleAppDiedLocked(app);
            }
        }
        return hasVisibleActivities;
    }

### 2.6 AS.handleAppDiedLocked

    boolean handleAppDiedLocked(ProcessRecord app) {
        //Activity暂停的过程中进程已死则无需走暂停流程
        if (mPausingActivity != null && mPausingActivity.app == app) {
            mPausingActivity = null;
        }
        //上次暂停activity,如果运行在该app则也清空
        if (mLastPausedActivity != null && mLastPausedActivity.app == app) {
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }
        //[见流程2.7]
        return removeHistoryRecordsForAppLocked(app);
    }

当以下Activity运行在app进程,则置空.

- AS.mLastPausedActivity
- AS.mLastNoHistoryActivity
- AS.mPausingActivity

### 2.7 AS.removeHistoryRecordsForAppLocked

    boolean removeHistoryRecordsForAppLocked(ProcessRecord app) {
      removeHistoryRecordsForAppLocked(mLRUActivities, app, "mLRUActivities");
      removeHistoryRecordsForAppLocked(mStackSupervisor.mStoppingActivities, app,
              "mStoppingActivities");
      removeHistoryRecordsForAppLocked(mStackSupervisor.mGoingToSleepActivities, app,
              "mGoingToSleepActivities");
      removeHistoryRecordsForAppLocked(mStackSupervisor.mWaitingVisibleActivities, app,
              "mWaitingVisibleActivities");
      removeHistoryRecordsForAppLocked(mStackSupervisor.mFinishingActivities, app,
              "mFinishingActivities");

      boolean hasVisibleActivities = false;

      int i = numActivities();
      for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
          final ArrayList<ActivityRecord> activities = mTaskHistory.get(taskNdx).mActivities;
          for (int activityNdx = activities.size() - 1; activityNdx >= 0; --activityNdx) {
              final ActivityRecord r = activities.get(activityNdx);
              --i;

              if (r.app == app) {
                  //当该activity可见,则设置该标识
                  if (r.visible) {
                      hasVisibleActivities = true;
                  }
                  final boolean remove;
                  if ((!r.haveState && !r.stateNotNeeded) || r.finishing) {
                      //当r没有状态 或者正在结束,则需要rmove
                      remove = true;
                  } else if (r.launchCount > 2 &&
                          r.lastLaunchTime > (SystemClock.uptimeMillis()-60000)) {
                      remove = true;
                  } else {
                      remove = false;
                  }
                  if (remove) {
                      if (!r.finishing) {
                          if (r.state == ActivityState.RESUMED) {
                              mService.updateUsageStats(r, false);
                          }
                      }
                  } else {
                      r.app = null;
                      r.nowVisible = false;
                      if (!r.haveState) {
                          r.icicle = null;
                      }
                  }
                  //清理Activity信息[见流程2.7.1]
                  cleanUpActivityLocked(r, true, true);
                  if (remove) {
                      //移除Activity [见流量2.7.2]
                      removeActivityFromHistoryLocked(r, "appDied");
                  }
              }
          }
      }

      return hasVisibleActivities;
    }


从以下5个队列中移除该activity的信息:

- AS.mLRUActivities
- ASS.mStoppingActivities
- ASS.mGoingToSleepActivities
- ASS.mWaitingVisibleActivities
- ASS.mFinishingActivities

#### 2.7.1 AS.cleanUpActivityLocked

    final void cleanUpActivityLocked(ActivityRecord r, boolean cleanServices,
            boolean setState) {
        //重置resume和pause信息
        if (mResumedActivity == r) {
            mResumedActivity = null;
        }
        if (mPausingActivity == r) {
            mPausingActivity = null;
        }
        //重置mFocusedActivity
        mService.clearFocusedActivity(r);

        r.configDestroy = false;
        r.frozenBeforeDestroy = false;

        if (setState) {
            //设置r状态为destroyed
            r.state = ActivityState.DESTROYED;
            r.app = null;
        }

        mStackSupervisor.mFinishingActivities.remove(r);
        mStackSupervisor.mWaitingVisibleActivities.remove(r);

        if (r.finishing && r.pendingResults != null) {
            for (WeakReference<PendingIntentRecord> apr : r.pendingResults) {
                PendingIntentRecord rec = apr.get();
                if (rec != null) {
                    mService.cancelIntentSenderLocked(rec, false);
                }
            }
            r.pendingResults = null;
        }

        if (cleanServices) {
            //清除跟该activity的service connections
            cleanUpActivityServicesLocked(r);
        }
        //移除超时消息
        removeTimeoutsForActivityLocked(r);
        if (getVisibleBehindActivity() == r) {
            mStackSupervisor.requestVisibleBehindLocked(r, false);
        }
    }

主要功能:

1. 当mResumedActivity,mPausingActivity,mFocusedActivity中与等待清理的Activity相同则置空.
2. 移除跟该Acitivity相关的以下超时消息:IDLE_TIMEOUT_MSG, PAUSE_TIMEOUT_MSG, STOP_TIMEOUT_MSG, DESTROY_TIMEOUT_MSG, LAUNCH_TICK_MSG, 这些常量定义在ActivityStack文件.


#### 2.7.2 AS.removeActivityFromHistoryLocked

    private void removeActivityFromHistoryLocked(ActivityRecord r, String reason) {
         mStackSupervisor.removeChildActivityContainers(r);
         finishActivityResultsLocked(r, Activity.RESULT_CANCELED, null);
         //设置activity状态为finishing
         r.makeFinishingLocked();

         r.takeFromHistory();
         removeTimeoutsForActivityLocked(r);
         //设置为destroyed状态
         r.state = ActivityState.DESTROYED;

         r.app = null;
         mWindowManager.removeAppToken(r.appToken);

         final TaskRecord task = r.task;
         if (task != null && task.removeActivity(r)) {
             if (mStackSupervisor.isFrontStack(this) && task == topTask() &&
                     task.isOverHomeStack()) {
                 mStackSupervisor.moveHomeStackTaskToTop(task.getTaskToReturnTo(), reason);
             }
             //从mTaskHistory中移除该task, 之后当mTaskHistory为空,则将home stack移至前台
             removeTask(task, reason);
         }
         cleanUpActivityServicesLocked(r);
         r.removeUriPermissionsLocked();
     }

- 设置Activity的finishing = true, 状态为destroyed, 并且从mTaskHistory中移除该task.
- 此时当mTaskHistory为空,则将home所在栈移至前台.


## 三. 总结

当进程被AMS或者lmk所杀时, system_server则会收到死亡回调,并进入binderDied()方法. 整个过程主要是移除各种系统相关的信息. 整个过程图:

![binder_died](/images/process/binder_died.jpg)


这个过程中进入AMS.handleAppDiedLocked后: 左边分支`cleanUpApplicationRecordLocked`用于清理死亡进程中运行的四大组件service, BroadcastReceiver, ContentProvider相关信息; 右边分支`ASS.handleAppDiedLocked`清理死亡进程中运行的activity相关信息.

另外,就是维护framework的各个进程队列mProcessesToGc, mPendingPssProcesses, mPidsSelfLocked, mLruProcesses以及ProcessList.remove()方法.
