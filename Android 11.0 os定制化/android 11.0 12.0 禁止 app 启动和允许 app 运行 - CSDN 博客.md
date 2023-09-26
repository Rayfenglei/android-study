> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124889195)

在 11.0 12.0 产品开发中, 可以有需求要实现禁止 app 启动和允许 app 运行的接口，禁用后 app 后已安装的应用从桌面消失，只存在于系统设置内的应用列表里，无法调用。启用后，恢复正常使用，在桌面显示。对于 app 管理的都是由 PackageManager 来负责的，PackageManger 的主要职责是管理应用程序包，通过它可以获取应用程序信息 所以就查看 PackManager 的相关源码看  
能不能实现需求:  
一、PackageManager 的功能：

1、安装，卸载应用  
2、查询 permission 相关信息  
3、查询 Application 相关信息 (application，activity，receiver，service，provider 及相应属性等）  
4、查询已安装应用  
5、增加，删除 permission  
6、清除用户数据、缓存，代码段等

二、PackageManager 相关类和方法介绍：

1、PackageManager 类

说明： 获得已安装的应用程序信息 。可以通过 getPackageManager() 方法获得。

常用方法：

public abstract PackageManager getPackageManager()

```
1

```

功能：获得一个 PackageManger 对象

public abstract Drawable getApplicationIcon(String packageName)

```
1

```

参数： packageName 包名  
功能：返回给定包名的图标，否则返回 null

public abstract ApplicationInfo getApplicationInfo(String packageName, int flags)

```
1

```

参数：  
　　packagename 包名  
　　flags 该 ApplicationInfo 是此 flags 标记，通常可以直接赋予常数 0 即可  
功能：返回该 ApplicationInfo 对象

public abstract List getInstalledApplications(int flags)

```
1

```

参数：  
　　flag 为一般为 GET_UNINSTALLED_PACKAGES，那么此时会返回所有 ApplicationInfo。我们可以对 ApplicationInfo  
　　的 flags 过滤, 得到我们需要的。  
功能：返回给定条件的所有 PackageInfo

public abstract List getInstalledPackages(int flags)

```
1

```

参数如上  
功能：返回给定条件的所有 PackageInfo

public abstract ResolveInfo resolveActivity(Intent intent, int flags)

```
1

```

参数：  
　　intent 查寻条件，Activity 所配置的 action 和 category  
　　flags： MATCH_DEFAULT_ONLY ：Category 必须带有 CATEGORY_DEFAULT 的 Activity，才匹配  
　　　　　　 GET_INTENT_FILTERS ：匹配 Intent 条件即可  
　　　　　　 GET_RESOLVED_FILTER ：匹配 Intent 条件即可  
功能 ：返回给定条件的 ResolveInfo 对象 (本质上是 Activity)

public abstract List queryIntentActivities(Intent intent, int flags)  
参数同上  
功能 ：返回给定条件的所有 ResolveInfo 对象 (本质上是 Activity)，集合对象

public abstract ResolveInfo resolveService(Intent intent, int flags)  
参数同上  
功能 ：返回给定条件的 ResolveInfo 对象 (本质上是 Service)

public abstract List queryIntentServices(Intent intent, int flags)  
参数同上  
功能 ：返回给定条件的所有 ResolveInfo 对象 (本质上是 Service)，集合对象

2、PackageItemInfo 类

说明： AndroidManifest.xml 文件中所有节点的基类，提供了这些节点的基本信息：label、icon、 meta-data。它并不直接使用，而是由子[类继承](https://so.csdn.net/so/search?q=%E7%B1%BB%E7%BB%A7%E6%89%BF&spm=1001.2101.3001.7020)然后调用相应方法。

3、ApplicationInfo 类 继承自 PackageItemInfo 类

说明：获取一个特定引用程序中节点的信息。

字段说明：  
flags 字段： FLAG_SYSTEM　系统应用程序  
FLAG_EXTERNAL_STORAGE　表示该应用安装在 sdcard 中

常用方法继承至 PackageItemInfo 类中的 loadIcon() 和 loadLabel()

4、ActivityInfo 类 继承自 PackageItemInfo 类

说明： 获得应用程序中或者 节点的信息 。我们可以通过它来获取我们设置的任何属性，包括 theme 、launchMode、launchmode 等

常用方法继承至 PackageItemInfo 类中的 loadIcon() 和 loadLabel()

5、ServiceInfo 类 继承自 PackageItemInfo 类

说明：与 ActivityInfo 类似，代表节点信息

6、ResolveInfo 类

说明：根据节点来获取其上一层目录的信息，通常是、、节点信息。  
常用方法有 loadIcon(PackageManager pm) 和 loadLabel(PackageManager pm)

PackageManager 类中一些变量和方法的介绍：

int COMPONENT_ENABLED_STATE_DEFAULT：

可以在方法 setApplicationEnabledSetting(String,int,int) 和 setComponentEnabledSetting(ComponentName,int,int) 中使用，该组件或应用程序处于默认开启状态（其在清单指定）。

```
1
2
3

```

int COMPONENT_ENABLED_STATE_DISABLED：

可以在方法 setApplicationEnabledSetting(String,int,int) 和 setComponentEnabledSetting(ComponentName,int,int) 中使用，该组件或者应用程序被禁用，不管你是否在清单文件中指定。

```
1

```

int COMPONENT_ENABLED_STATE_DISABLED_UNTIL_USED：

只在方法 setApplicationEnabledSetting(String,int,int) 中使用，用户实际上使用它，这个应用程序才会被启动。

```
1

```

int COMPONENT_ENABLED_STATE_DISABLED_USER：

只在方法 setApplicationEnabledSetting(String,int,int) 中使用，用户禁止启动该应用程序，不管是否在清单文件中指定。

```
1

```

int COMPONENT_ENABLED_STATE_ENABLED：

可以在方法 setApplicationEnabledSetting(String,int,int) 和 setComponentEnabledSetting(ComponentName,int,int) 中使用，该组件或者应用程序启动，不管你是否在清单文件中指定。

```
1

```

int DONT_KILL_APP：

setComponentEnabledSetting(ComponentName,int,int) 方法中的标志参数，表明您不想杀死包含该组件的应用程序。

```
1

```

通过上面的源码可以看出 通过 setApplicationEnabledSetting 的包名和变量值可以实现禁止或运行 app 运行  
解决思路:  
调用 PackageManager 的 setApplicationEnabledSetting 来实现  
1. 禁止 app 运行

```
pm.setApplicationEnabledSetting(pkg, PackageManager.COMPONENT_ENABLED_STATE_DISABLED, 0);

```

2. 允许 app 运行

```
pm.setApplicationEnabledSetting(str_packagename, PackageManager.COMPONENT_ENABLED_STATE_ENABLED, 0);

```

实现案例如下:

```
PackageManager pm = mContext.getPackageManager();
        if (packageNames==null || packageNames.size()==0){
            Log.d(TAG,"DisallowedRunningApp is null");
			String disableRunapps = Settings.System.getString(mContext.getContentResolver(), "DisallowedRunningApp");
            if(!TextUtils.isEmpty(disableRunapps)){
               String [] applist = disableRunapps.split(",");
				if(applist!=null&&applist.length>0){
					for(String str_packagename:applist){
    // 运行app运行
						pm.setApplicationEnabledSetting(str_packagename, PackageManager.COMPONENT_ENABLED_STATE_ENABLED, 0);
					}
				}
            }
            Settings.System.putString(mContext.getContentResolver(), "DisallowedRunningApp", "");
            return;
        }
        String s="";
        for (String str:packageNames){
            s=s+str+",";
        }
        Settings.System.putString(mContext.getContentResolver(), "DisallowedRunningApp", s);
        try {
            disable_packageNames = packageNames;
            if (disable_packageNames != null && disable_packageNames.size() > 0) {
                for (String pkg : disable_packageNames) {

                    ApplicationInfo applicationInfo = pm.getApplicationInfo(pkg, 0);
                    if (applicationInfo == null) {
                        continue;
                    }
                    if (!applicationInfo.enabled) {
                        continue;
                    }
                    if ("com.android.settings".equals(applicationInfo.packageName)) {
                        continue;
                    }
                   //禁止app运行
                    pm.setApplicationEnabledSetting(pkg, PackageManager.COMPONENT_ENABLED_STATE_DISABLED, 0);
                }
            } else {
                List<ApplicationInfo> installedApplications = pm.getInstalledApplications(PackageManager.MATCH_ALL);
                for (ApplicationInfo info : installedApplications) {
                    if (!info.enabled) {
						if ("com.android.settings".equals(info.packageName)) {
                            continue;
                        }
                        pm.setApplicationEnabledSetting(info.packageName, PackageManager.COMPONENT_ENABLED_STATE_ENABLED, 0);
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

```