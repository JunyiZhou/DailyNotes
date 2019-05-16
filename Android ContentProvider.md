# ContentProvider
> 四大组价之一，用于面向 App 提供 content，支持跨进程

```java
public List<Map<String, Object>> getContacts() {
    List<Map<String, Object>> contacts = new ArrayList<>();
    Cursor cursor = getContentResolver().query(
            Uri.parse("content://com.android.contacts/raw_contacts"),
            null,
            null,
            null,
            null);
    if (cursor == null) {
        return contacts;
    }

    while (cursor.moveToNext()) {
        Map<String, Object> contact = new HashMap<>();
        int id = cursor.getInt(cursor.getColumnIndex("_id"));
        String displayName = cursor.getString(cursor.getColumnIndex("display_name"));
        contact.put("names", displayName);

        //根据联系人获取联系人数据
        Cursor cursor2 = getContentResolver().query(
                Uri.parse("content://com.android.contacts/raw_contacts/" + id + "/data"),
                null,
                null,
                null,
                null);
        if (cursor2 == null) {
            return contacts;
        }

        while (cursor2.moveToNext()) {
            String type = cursor2.getString(cursor2.getColumnIndex("mimetype"));
            String data1 = null;
            if ("vnd.android.cursor.item/phone_v2".equals(type)) {
                data1 = cursor2.getString(cursor2.getColumnIndex("data1"));
                contact.put("phones", data1);
            }
        }
        cursor2.close();
        contacts.add(contact);
    } //通知适配器发生改变
    cursor.close();

    return contacts;
}
```
这段代码是通过 ``ContentProvider`` 获取联系人信息的基本示例，主要描述了 ``ContentProvider`` 的基本用法：
1.  通过 ``getContentResolver`` 的 ``query()`` 方法，获取 ``Cursor`` 实例，该 ``Cursor`` 就是 ``ContentProvider`` 提供过来的用于进行后续操作的工具
2.  ``Cursor`` 可能是 ``null``，需要先判空后使用
3.  此 ``Cursor`` 和 SQLite 的 ``Cursor`` 类似，可以 ``moveToNext`` 换行，可以获取列号获取对应位置的数据，同时也提供了通过列名获取列号的方法等
4.  ``Cursor`` 使用完后需要及时 ``close``

##  调用流程
基本用法如上，接下来看一下 ``ContentProvider`` 是如何被调用起来的。

对 ``ContentProvider`` provide 的 content 进行 CURD 操作依赖于 ``ContentResolver``，比如 demo 里面的 query 方法，就是用来查数据用的。query 方法的调用链是 ``context.getContentResolver().query()``，其中 ``Context`` 的具体实现在 ``ContextImpl`` 中，就从这开始：

```java
API 27:ContextImpl.java

private final ApplicationContentResolver mContentResolver;

private ContextImpl(@Nullable ContextImpl container, @NonNull ActivityThread mainThread,
        @NonNull LoadedApk packageInfo, @Nullable String splitName,
        @Nullable IBinder activityToken, @Nullable UserHandle user, int flags,
        @Nullable ClassLoader classLoader) {
    ...
    mContentResolver = new ApplicationContentResolver(this, mainThread, user);
}

@Override
public ContentResolver getContentResolver() {
    return mContentResolver;
}
```

可以看到 ``getContentResolver`` 最终是一个叫 ``ApplicationContentResolver`` 的实现类被扔了出来，``ApplicationContentResolver`` 是 ``ContentResolver`` 的一个实现类，粗略看了一下 ``ApplicationContentResolver`` 的代码，发现没有 ``query()`` 的实现，因此先来看一下 ``ContentResolver`` 里面看一下：

```java
API 27:ContentResolver.java

public final @Nullable Cursor query(final @RequiresPermission.Read @NonNull Uri uri,
        @Nullable String[] projection, @Nullable Bundle queryArgs,
        @Nullable CancellationSignal cancellationSignal) {
    Preconditions.checkNotNull(uri, "uri");

    // 获取一个 unstableProvider
    // Q1：unstable 代表什么？
    IContentProvider unstableProvider = acquireUnstableProvider(uri);
    if (unstableProvider == null) {
        // 获取不到 unstableProvider 直接 return null
        // 这步说明 unstableProvider 获取成功与否可以体现出当前系统处于某种不可用状态
        // Q2：什么情况下 unstableProvider 会返回 null？
        return null;
    }
    IContentProvider stableProvider = null;
    Cursor qCursor = null;
    try {
        long startTime = SystemClock.uptimeMillis();
        ICancellationSignal remoteCancellationSignal = null;
        if (cancellationSignal != null) {
            cancellationSignal.throwIfCanceled();
            remoteCancellationSignal = unstableProvider.createCancellationSignal();
            cancellationSignal.setRemote(remoteCancellationSignal);
        }
        try {
            // 尝试 query 一下
            qCursor = unstableProvider.query(mPackageName, uri, projection,
                    queryArgs, remoteCancellationSignal);
        } catch (DeadObjectException e) {
            // The remote process has died...  but we only hold an unstable
            // reference though, so we might recover!!!  Let's try!!!!
            // This is exciting!!1!!1!!!!1

            // 这段沙雕的注释主要想告诉我们的是“兄弟，远端虽然跪了，但是因为我们机智的用了
            // unstable 的资源，所以没毛事，我们还有机会，燥起来吧”
            // 虽然沙雕一点，但是我们还是可以 get 到一些不寻常的信息：
            // 1.首先基本就可以回答 Q1 了，unstable 表明获取到的资源是不稳定的
            // 2.client 持有 unstable 资源，即便远端 server 跪了，也没事，还有机会重试
            // Q3：持有 stable 资源，如果远端 server 跪了会有什么事？

            // 一套 release unstable 资源的广播体操，具体怎么跳不重要
            unstableProviderDied(unstableProvider);
            // 所谓的重试在这体现出来了，但不同的是这次 acquire 是 stableProvider
            stableProvider = acquireProvider(uri);
            if (stableProvider == null) {
                return null;
            }

            // 再次尝试 query 一下
            // 注意！这里没有在 catch DeadObjectException，stable 和 unstable 的区别在这略有体现了
            // 我们可以初步猜测一下，不处理一般是因为没必要，一旦抛异常了，client 连带着也跪了
            // Q4：为什么在这不 catch DeadObjectException 了？就是希望 client 跟着一起跪？
            qCursor = stableProvider.query(
                    mPackageName, uri, projection, queryArgs, remoteCancellationSignal);
        }
        if (qCursor == null) {
            return null;
        }
        // Force query execution.  Might fail and throw a runtime exception here.
        qCursor.getCount();
        long durationMillis = SystemClock.uptimeMillis() - startTime;
        maybeLogQueryToEventLog(durationMillis, uri, projection, queryArgs);
        // Wrap the cursor object into CursorWrapperInner object.
        final IContentProvider provider = (stableProvider != null) ? stableProvider
                : acquireProvider(uri);
        final CursorWrapperInner wrapper = new CursorWrapperInner(qCursor, provider);
        stableProvider = null;
        qCursor = null;
        return wrapper;
    } catch (RemoteException e) {
        // Arbitrary and not worth documenting, as Activity
        // Manager will kill this process shortly anyway.

        // 到这终于明白为啥不再 catch DeadObjectException 了，AM 会很快干掉 client 进程
        // 所以处理不处理没啥，甚至于 trace 都不想打出来。。。
        // 所以可以回答 Q3&4，是的，就是希望 client 跟着跪，不但希望，AM 还强行让 client 一起跪
        // Q5：why？？？
        return null;
    } finally {
        // 释放各种资源
        if (qCursor != null) {
            qCursor.close();
        }
        if (cancellationSignal != null) {
            cancellationSignal.setRemote(null);
        }
        if (unstableProvider != null) {
            releaseUnstableProvider(unstableProvider);
        }
        if (stableProvider != null) {
            releaseProvider(stableProvider);
        }
    }
}
```

总结一下：
1.  优先 acquire unstableProvider，如果可以用就对付用，如果发现远端跪了就 acquire stableProvider
2.  stableProvider acquire 到之后同样是可以用就对付用，如果远端跪了也无所谓，反正 AM 也不让你活

目前的问题主要有两个，一个是 provider 的 acquire 过程，另一个就是为什么 AM 会主动杀死 client，连带着就是如何避免被杀

接下来到 ``ApplicationContentResolver`` 看一下 ``acquireUnstableProvider`` 和 ``acquireProvider``：

```java
API 27:ContextImpl.java -> ApplicationContentResolver

private final ActivityThread mMainThread;

@Override
protected IContentProvider acquireProvider(Context context, String auth) {
    return mMainThread.acquireProvider(context,
            ContentProvider.getAuthorityWithoutUserId(auth),
            resolveUserIdFromAuthority(auth), true);
}

@Override
protected IContentProvider acquireUnstableProvider(Context c, String auth) {
    return mMainThread.acquireProvider(c,
            ContentProvider.getAuthorityWithoutUserId(auth),
            resolveUserIdFromAuthority(auth), false);
}
```

``ApplicationContentResolver`` 里面没什么实际逻辑，就是调用 ``AndroidThread`` 的 `acquireProvider` 方法：

```java
API 27:AndroidThread.java

public final IContentProvider acquireProvider(
        Context c, String auth, int userId, boolean stable) {
    // AndroidThread 内部会对 acquire 过的 provider 做缓存，通过 acquireExistingProvider 可以获得
    final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
    if (provider != null) {
        return provider;
    }
    // There is a possible race here.  Another thread may try to acquire
    // the same provider at the same time.  When this happens, we want to ensure
    // that the first one wins.
    // Note that we cannot hold the lock while acquiring and installing the
    // provider since it might take a long time to run and it could also potentially
    // be re-entrant in the case where the provider is in the same process.

    // 这里 google 的大佬提出一个多线程 acquire 同一个 provider 的风险，说是会确保第一个赢
    // 但又说不能 hold lock。。。不知道咋搞
    ContentProviderHolder holder = null;
    try {
        // 重点！！！通过 AM 向 AMS 请求 ContentProvider
        holder = ActivityManager.getService().getContentProvider(
                getApplicationThread(), auth, userId, stable);
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
    if (holder == null) {
        Slog.e(TAG, "Failed to find provider info for " + auth);
        return null;
    }
    // Install provider will increment the reference count for us, and break
    // any ties in the race.

    // “安装” provider，新建实例，增加 reference 的数量，先放一放
    holder = installProvider(c, holder, holder.info,
            true /*noisy*/, holder.noReleaseNeeded, stable);
    return holder.provider;
}
```

走到这我们可以看到 client 端 `acquireProvider` 最终是通过 AM 走 binder 向 AMS 请求来的。
基于 AIDL 的基本规则，AMS 一定会对应有一个 `getContentProvider` 方法，接下来看一下：

```java
@Override
public final ContentProviderHolder getContentProvider(
        IApplicationThread caller, String name, int userId, boolean stable) {
    enforceNotIsolatedCaller("getContentProvider");
    if (caller == null) {
        String msg = "null IApplicationThread when getting content provider "
                + name;
        Slog.w(TAG, msg);
        throw new SecurityException(msg);
    }
    // The incoming user check is now handled in checkContentProviderPermissionLocked() to deal
    // with cross-user grant.
    // 进一步调用 getContentProviderImpl 实现具体逻辑
    return getContentProviderImpl(caller, name, null, stable, userId);
}

private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
        String name, IBinder token, boolean stable, int userId) {
    ContentProviderRecord cpr;
    ContentProviderConnection conn = null;
    ProviderInfo cpi = null;
    synchronized(this) {
        long startTime = SystemClock.uptimeMillis();
        ProcessRecord r = null;
        if (caller != null) {
            r = getRecordForAppLocked(caller);
            if (r == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                      + " (pid=" + Binder.getCallingPid()
                      + ") when getting content provider " + name);
            }
        }
        boolean checkCrossUser = true;
        checkTime(startTime, "getContentProviderImpl: getProviderByName");
        // First check if this content provider has been published...
        cpr = mProviderMap.getProviderByName(name, userId);
        // If that didn't work, check if it exists for user 0 and then
        // verify that it's a singleton provider before using it.
        if (cpr == null && userId != UserHandle.USER_SYSTEM) {
            cpr = mProviderMap.getProviderByName(name, UserHandle.USER_SYSTEM);
            if (cpr != null) {
                cpi = cpr.info;
                if (isSingleton(cpi.processName, cpi.applicationInfo,
                        cpi.name, cpi.flags)
                        && isValidSingletonCall(r.uid, cpi.applicationInfo.uid)) {
                    userId = UserHandle.USER_SYSTEM;
                    checkCrossUser = false;
                } else {
                    cpr = null;
                    cpi = null;
                }
            }
        }
        boolean providerRunning = cpr != null && cpr.proc != null && !cpr.proc.killed;
        if (providerRunning) {
            cpi = cpr.info;
            String msg;
            checkTime(startTime, "getContentProviderImpl: before checkContentProviderPermission");
            if ((msg = checkContentProviderPermissionLocked(cpi, r, userId, checkCrossUser))
                    != null) {
                throw new SecurityException(msg);
            }
            checkTime(startTime, "getContentProviderImpl: after checkContentProviderPermission");
            if (r != null && cpr.canRunHere(r)) {
                // This provider has been published or is in the process
                // of being published...  but it is also allowed to run
                // in the caller's process, so don't make a connection
                // and just let the caller instantiate its own instance.
                ContentProviderHolder holder = cpr.newHolder(null);
                // don't give caller the provider object, it needs
                // to make its own.
                holder.provider = null;
                return holder;
            }
            // Don't expose providers between normal apps and instant apps
            try {
                if (AppGlobals.getPackageManager()
                        .resolveContentProvider(name, 0 /*flags*/, userId) == null) {
                    return null;
                }
            } catch (RemoteException e) {
            }
            final long origId = Binder.clearCallingIdentity();
            checkTime(startTime, "getContentProviderImpl: incProviderCountLocked");
            // In this case the provider instance already exists, so we can
            // return it right away.
            conn = incProviderCountLocked(r, cpr, token, stable);
            if (conn != null && (conn.stableCount+conn.unstableCount) == 1) {
                if (cpr.proc != null && r.setAdj <= ProcessList.PERCEPTIBLE_APP_ADJ) {
                    // If this is a perceptible app accessing the provider,
                    // make sure to count it as being accessed and thus
                    // back up on the LRU list.  This is good because
                    // content providers are often expensive to start.
                    checkTime(startTime, "getContentProviderImpl: before updateLruProcess");
                    updateLruProcessLocked(cpr.proc, false, null);
                    checkTime(startTime, "getContentProviderImpl: after updateLruProcess");
                }
            }
            checkTime(startTime, "getContentProviderImpl: before updateOomAdj");
            final int verifiedAdj = cpr.proc.verifiedAdj;
            boolean success = updateOomAdjLocked(cpr.proc, true);
            // XXX things have changed so updateOomAdjLocked doesn't actually tell us
            // if the process has been successfully adjusted.  So to reduce races with
            // it, we will check whether the process still exists.  Note that this doesn't
            // completely get rid of races with LMK killing the process, but should make
            // them much smaller.
            if (success && verifiedAdj != cpr.proc.setAdj && !isProcessAliveLocked(cpr.proc)) {
                success = false;
            }
            maybeUpdateProviderUsageStatsLocked(r, cpr.info.packageName, name);
            checkTime(startTime, "getContentProviderImpl: after updateOomAdj");
            if (DEBUG_PROVIDER) Slog.i(TAG_PROVIDER, "Adjust success: " + success);
            // NOTE: there is still a race here where a signal could be
            // pending on the process even though we managed to update its
            // adj level.  Not sure what to do about this, but at least
            // the race is now smaller.
            if (!success) {
                // Uh oh...  it looks like the provider's process
                // has been killed on us.  We need to wait for a new
                // process to be started, and make sure its death
                // doesn't kill our process.
                Slog.i(TAG, "Existing provider " + cpr.name.flattenToShortString()
                        + " is crashing; detaching " + r);
                boolean lastRef = decProviderCountLocked(conn, cpr, token, stable);
                checkTime(startTime, "getContentProviderImpl: before appDied");
                appDiedLocked(cpr.proc);
                checkTime(startTime, "getContentProviderImpl: after appDied");
                if (!lastRef) {
                    // This wasn't the last ref our process had on
                    // the provider...  we have now been killed, bail.
                    return null;
                }
                providerRunning = false;
                conn = null;
            } else {
                cpr.proc.verifiedAdj = cpr.proc.setAdj;
            }
            Binder.restoreCallingIdentity(origId);
        }
        if (!providerRunning) {
            try {
                checkTime(startTime, "getContentProviderImpl: before resolveContentProvider");
                cpi = AppGlobals.getPackageManager().
                    resolveContentProvider(name,
                        STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS, userId);
                checkTime(startTime, "getContentProviderImpl: after resolveContentProvider");
            } catch (RemoteException ex) {
            }
            if (cpi == null) {
                return null;
            }
            // If the provider is a singleton AND
            // (it's a call within the same user || the provider is a
            // privileged app)
            // Then allow connecting to the singleton provider
            boolean singleton = isSingleton(cpi.processName, cpi.applicationInfo,
                    cpi.name, cpi.flags)
                    && isValidSingletonCall(r.uid, cpi.applicationInfo.uid);
            if (singleton) {
                userId = UserHandle.USER_SYSTEM;
            }
            cpi.applicationInfo = getAppInfoForUser(cpi.applicationInfo, userId);
            checkTime(startTime, "getContentProviderImpl: got app info for user");
            String msg;
            checkTime(startTime, "getContentProviderImpl: before checkContentProviderPermission");
            if ((msg = checkContentProviderPermissionLocked(cpi, r, userId, !singleton))
                    != null) {
                throw new SecurityException(msg);
            }
            checkTime(startTime, "getContentProviderImpl: after checkContentProviderPermission");
            if (!mProcessesReady
                    && !cpi.processName.equals("system")) {
                // If this content provider does not run in the system
                // process, and the system is not yet ready to run other
                // processes, then fail fast instead of hanging.
                throw new IllegalArgumentException(
                        "Attempt to launch content provider before system ready");
            }
            // Make sure that the user who owns this provider is running.  If not,
            // we don't want to allow it to run.
            if (!mUserController.isUserRunningLocked(userId, 0)) {
                Slog.w(TAG, "Unable to launch app "
                        + cpi.applicationInfo.packageName + "/"
                        + cpi.applicationInfo.uid + " for provider "
                        + name + ": user " + userId + " is stopped");
                return null;
            }
            ComponentName comp = new ComponentName(cpi.packageName, cpi.name);
            checkTime(startTime, "getContentProviderImpl: before getProviderByClass");
            cpr = mProviderMap.getProviderByClass(comp, userId);
            checkTime(startTime, "getContentProviderImpl: after getProviderByClass");
            final boolean firstClass = cpr == null;
            if (firstClass) {
                final long ident = Binder.clearCallingIdentity();
                // If permissions need a review before any of the app components can run,
                // we return no provider and launch a review activity if the calling app
                // is in the foreground.
                if (mPermissionReviewRequired) {
                    if (!requestTargetProviderPermissionsReviewIfNeededLocked(cpi, r, userId)) {
                        return null;
                    }
                }
                try {
                    checkTime(startTime, "getContentProviderImpl: before getApplicationInfo");
                    ApplicationInfo ai =
                        AppGlobals.getPackageManager().
                            getApplicationInfo(
                                    cpi.applicationInfo.packageName,
                                    STOCK_PM_FLAGS, userId);
                    checkTime(startTime, "getContentProviderImpl: after getApplicationInfo");
                    if (ai == null) {
                        Slog.w(TAG, "No package info for content provider "
                                + cpi.name);
                        return null;
                    }
                    ai = getAppInfoForUser(ai, userId);
                    cpr = new ContentProviderRecord(this, cpi, ai, comp, singleton);
                } catch (RemoteException ex) {
                    // pm is in same process, this will never happen.
                } finally {
                    Binder.restoreCallingIdentity(ident);
                }
            }
            checkTime(startTime, "getContentProviderImpl: now have ContentProviderRecord");
            if (r != null && cpr.canRunHere(r)) {
                // If this is a multiprocess provider, then just return its
                // info and allow the caller to instantiate it.  Only do
                // this if the provider is the same user as the caller's
                // process, or can run as root (so can be in any process).
                return cpr.newHolder(null);
            }
            if (DEBUG_PROVIDER) Slog.w(TAG_PROVIDER, "LAUNCHING REMOTE PROVIDER (myuid "
                        + (r != null ? r.uid : null) + " pruid " + cpr.appInfo.uid + "): "
                        + cpr.info.name + " callers=" + Debug.getCallers(6));
            // This is single process, and our app is now connecting to it.
            // See if we are already in the process of launching this
            // provider.
            final int N = mLaunchingProviders.size();
            int i;
            for (i = 0; i < N; i++) {
                if (mLaunchingProviders.get(i) == cpr) {
                    break;
                }
            }
            // If the provider is not already being launched, then get it
            // started.
            if (i >= N) {
                final long origId = Binder.clearCallingIdentity();
                try {
                    // Content provider is now in use, its package can't be stopped.
                    try {
                        checkTime(startTime, "getContentProviderImpl: before set stopped state");
                        AppGlobals.getPackageManager().setPackageStoppedState(
                                cpr.appInfo.packageName, false, userId);
                        checkTime(startTime, "getContentProviderImpl: after set stopped state");
                    } catch (RemoteException e) {
                    } catch (IllegalArgumentException e) {
                        Slog.w(TAG, "Failed trying to unstop package "
                                + cpr.appInfo.packageName + ": " + e);
                    }
                    // Use existing process if already started
                    checkTime(startTime, "getContentProviderImpl: looking for process record");
                    ProcessRecord proc = getProcessRecordLocked(
                            cpi.processName, cpr.appInfo.uid, false);
                    if (proc != null && proc.thread != null && !proc.killed) {
                        if (DEBUG_PROVIDER) Slog.d(TAG_PROVIDER,
                                "Installing in existing process " + proc);
                        if (!proc.pubProviders.containsKey(cpi.name)) {
                            checkTime(startTime, "getContentProviderImpl: scheduling install");
                            proc.pubProviders.put(cpi.name, cpr);
                            try {
                                proc.thread.scheduleInstallProvider(cpi);
                            } catch (RemoteException e) {
                            }
                        }
                    } else {
                        checkTime(startTime, "getContentProviderImpl: before start process");
                        proc = startProcessLocked(cpi.processName,
                                cpr.appInfo, false, 0, "content provider",
                                new ComponentName(cpi.applicationInfo.packageName,
                                        cpi.name), false, false, false);
                        checkTime(startTime, "getContentProviderImpl: after start process");
                        if (proc == null) {
                            Slog.w(TAG, "Unable to launch app "
                                    + cpi.applicationInfo.packageName + "/"
                                    + cpi.applicationInfo.uid + " for provider "
                                    + name + ": process is bad");
                            return null;
                        }
                    }
                    cpr.launchingApp = proc;
                    mLaunchingProviders.add(cpr);
                } finally {
                    Binder.restoreCallingIdentity(origId);
                }
            }
            checkTime(startTime, "getContentProviderImpl: updating data structures");
            // Make sure the provider is published (the same provider class
            // may be published under multiple names).
            if (firstClass) {
                mProviderMap.putProviderByClass(comp, cpr);
            }
            mProviderMap.putProviderByName(name, cpr);
            conn = incProviderCountLocked(r, cpr, token, stable);
            if (conn != null) {
                conn.waiting = true;
            }
        }
        checkTime(startTime, "getContentProviderImpl: done!");
        grantEphemeralAccessLocked(userId, null /*intent*/,
                cpi.applicationInfo.uid, UserHandle.getAppId(Binder.getCallingUid()));
    }
    // Wait for the provider to be published...
    synchronized (cpr) {
        while (cpr.provider == null) {
            if (cpr.launchingApp == null) {
                Slog.w(TAG, "Unable to launch app "
                        + cpi.applicationInfo.packageName + "/"
                        + cpi.applicationInfo.uid + " for provider "
                        + name + ": launching app became null");
                EventLog.writeEvent(EventLogTags.AM_PROVIDER_LOST_PROCESS,
                        UserHandle.getUserId(cpi.applicationInfo.uid),
                        cpi.applicationInfo.packageName,
                        cpi.applicationInfo.uid, name);
                return null;
            }
            try {
                if (DEBUG_MU) Slog.v(TAG_MU,
                        "Waiting to start provider " + cpr
                        + " launchingApp=" + cpr.launchingApp);
                if (conn != null) {
                    conn.waiting = true;
                }
                cpr.wait();
            } catch (InterruptedException ex) {
            } finally {
                if (conn != null) {
                    conn.waiting = false;
                }
            }
        }
    }
    return cpr != null ? cpr.newHolder(conn) : null;
}
```
##  启动
##  连坐诛杀
