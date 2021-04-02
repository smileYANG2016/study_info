## packageManagerSerivce疑问



1. apk文件的结构？
2. apk安装的概念，何为安装？
3. 系统如何扫描解析apk文件以及解析出来的数据如何保存、相应的类结构图。
4. 第三方apk安装流程。
5. android UID/GIDS的概念
6. split apk,https://developer.android.com/studio/build/configure-apk-splits

### apk文件的结构

​	参考资料如下

apk结构：https://blog.csdn.net/bupt073114/article/details/42298337?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromBaidu-1.control&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromBaidu-1.control

解析AXML：

https://blog.csdn.net/beyond702/article/details/51830108

Android APK文件结构 完整打包编译的流程 APK安装过程 详解：

https://blog.csdn.net/aha_jasper/article/details/104944929

apk的打包：

https://juejin.cn/post/6844903910734299149

### 系统如何扫描解析apk文件以及解析出来的数据如何保存



- 扫描目录的顺序

  1. 先扫描/vendor/product/system下面的overlay
  
     ```
     // Collect vendor/product/product_services overlay packages. (Do this before scanning
     // any apps.)
     
     
         private static final String VENDOR_OVERLAY_DIR = "/vendor/overlay";
     
         private static final String PRODUCT_OVERLAY_DIR = "/product/overlay";
     
         private static final String PRODUCT_SERVICES_OVERLAY_DIR = "/product_services/overlay";
     
         private static final String ODM_OVERLAY_DIR = "/odm/overlay";
     
         private static final String OEM_OVERLAY_DIR = "/oem/overlay";
     ```
  
  2. 扫描/system/framework/
  
     ```
     // Find base frameworks (resource packages without code).
     File frameworkDir = new File(Environment.getRootDirectory(), "framework");
     ```
  
  3. 扫描/system分区的priv-app/和app/目录
  
   ```
     // Collect privileged system packages.
   final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
     
     // Collect ordinary system packages.
                 final File systemAppDir = new File(Environment.getRootDirectory(), "app");
     
   ```
  
  4. 扫描/vendor分区的priv-app/和app/目录
  
     ```
     // Collect privileged vendor packages.
     File privilegedVendorAppDir = new File(Environment.getVendorDirectory(), "priv-app");
     
      // Collect ordinary vendor packages.
                 File vendorAppDir = new File(Environment.getVendorDirectory(), "app");
     ```
  
  5. 扫描/odm分区的priv-app/和app/目录
  
     ```
     // Collect privileged odm packages. /odm is another vendor partition
     // other than /vendor.
     File privilegedOdmAppDir = new File(Environment.getOdmDirectory(),
                 "priv-app");
                 
     // Collect ordinary odm packages. /odm is another vendor partition
                 // other than /vendor.
                 File odmAppDir = new File(Environment.getOdmDirectory(), "app");
     ```
  
  6. 扫描/oem分区的app目录
  
     ```
     // Collect all OEM packages.
     final File oemAppDir = new File(Environment.getOemDirectory(), "app");
     
     
     ```
  
  7. 扫描/product分区的priv-app/和app/目录
  
     ```
     // Collected privileged /product packages.
     File privilegedProductAppDir = new File(Environment.getProductDirectory(), "priv-app");
     
     // Collect ordinary /product packages.
                 File productAppDir = new File(Environment.getProductDirectory(), "app");
     ```
  
  8. 扫描/product_services的priv-app/和app/目录
  
     ```
     // Collected privileged /product_services packages.
     File privilegedProductServicesAppDir =
             new File(Environment.getProductServicesDirectory(), "priv-app");
             
     // Collect ordinary /product_services packages.
     File productServicesAppDir = new File(Environment.getProductServicesDirectory(), "app");
     
          public static @NonNull File getProductServicesDirectory() {
     290          return getDirectory("PRODUCT_SERVICES_ROOT", "/product_services");
     291      }
     
     7      static File getDirectory(String variableName, String defaultPath) {
     1368          String path = System.getenv(variableName);
     1369          return path == null ? new File(defaultPath) : new File(path);
     1370      }
     ```
  
  9. 扫描installed apk(/data/app/目录)
  
     ```
     /** Directory where installed applications are stored */
     private static final File sAppInstallDir =
             new File(Environment.getDataDirectory(), "app");
             
     scanDirTracedLI(sAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);
     ```
  
  10. 
  
  
  
- 扫描各个分区下面的apk

  ```
   // Collect privileged system packages.  路径为/system/priv-app
              final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
              scanDirTracedLI(privilegedAppDir,
                      mDefParseFlags
                      | PackageParser.PARSE_IS_SYSTEM_DIR,
                      scanFlags
                      | SCAN_AS_SYSTEM
                      | SCAN_AS_PRIVILEGED,
                      0);
        
        
        //扫描不同的分区传入不同的scanDir和scanflags,从sytem/odm/product/vendor分区传入的参数看，扫描系统apk传入的parseFlags=mDefParseFlags|PackageParser.PARSE_IS_SYSTEM_DIR,
                      
      public void scanDirTracedLI(File scanDir, final int parseFlags, int scanFlags,
              long currentTime) {
          Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "scanDir [" + scanDir.getAbsolutePath() + "]");
          try {
              scanDirLI(scanDir, parseFlags, scanFlags, currentTime);
          } finally {
              Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
          }
      }
      
      
      
  ```

  解析的工作会交个packageparser进行处理，解析出来的pkg信息会存储在ParseResult。

  ```
             // Submit files for parsing in parallel
              int fileCount = 0;
              for (File file : files) {
                  final boolean isPackage = (isApkFile(file) || file.isDirectory())
                          && !PackageInstallerService.isStageName(file.getName());
                  if (!isPackage) {
                      // Ignore entries which are not packages
                      continue;
                  }
                  parallelPackageParser.submit(file, parseFlags);
                  fileCount++;
              }
              
              public void submit(File scanFile, int parseFlags) {
          mService.submit(() -> {
              ParseResult pr = new ParseResult();
              Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parallel parsePackage [" + scanFile + "]");
              try {
                  PackageParser pp = new PackageParser();
  
  				、、、、、
  				、、、、、
  				、、、、、
                  pr.scanFile = scanFile;
                  pr.pkg = parsePackage(pp, scanFile, parseFlags);
              } catch (Throwable e) {
                  pr.throwable = e;
              } finally {
                  Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
              }
              try {
                  mQueue.put(pr);
              } catch (InterruptedException e) {
                 
              }
          });
      }
      //前面传入的packageFile是各分区的priv-app/app路径。按照手机的目录结构应该是通过parseClusterPackage进行解析的
       public Package parsePackage(File packageFile, int flags, boolean useCaches)
              throws PackageParserException {
          Package parsed = useCaches ? getCachedResult(packageFile, flags) : null;
          if (parsed != null) {
              return parsed;
          }
  
          long parseTime = LOG_PARSE_TIMINGS ? SystemClock.uptimeMillis() : 0;
          if (packageFile.isDirectory()) {
              parsed = parseClusterPackage(packageFile, flags);
          } else {
              parsed = parseMonolithicPackage(packageFile, flags);
          }
  
          long cacheTime = LOG_PARSE_TIMINGS ? SystemClock.uptimeMillis() : 0;
          cacheResult(packageFile, flags, parsed);
          if (LOG_PARSE_TIMINGS) {
              parseTime = cacheTime - parseTime;
              cacheTime = SystemClock.uptimeMillis() - cacheTime;
              if (parseTime + cacheTime > LOG_PARSE_TIMINGS_THRESHOLD_MS) {
                  Slog.i(TAG, "Parse times for '" + packageFile + "': parse=" + parseTime
                          + "ms, update_cache=" + cacheTime + " ms");
              }
          }
          return parsed;
      }
      
      
      //分析parseClusterPackage的具体逻辑。该方法返回的Package对象是由parseBaseApk来完成的，而parseClusterPackageLite获取一些辅助信息，比如：splitCodePath,cpu的架构等。
      private Package parseClusterPackage(File packageDir, int flags) throws PackageParserException {
      	//第一步：parseClusterPackageLite获取PackageLite对象lite
          final PackageLite lite = parseClusterPackageLite(packageDir, 0);
          if (mOnlyCoreApps && !lite.coreApp) {
              throw new PackageParserException(INSTALL_PARSE_FAILED_MANIFEST_MALFORMED,
                      "Not a coreApp: " + packageDir);
          }
  
          // Build the split dependency tree.
          SparseArray<int[]> splitDependencies = null;
          final SplitAssetLoader assetLoader;
          if (lite.isolatedSplits && !ArrayUtils.isEmpty(lite.splitNames)) {
              try {
                  splitDependencies = SplitAssetDependencyLoader.createDependenciesFromPackage(lite);
                  assetLoader = new SplitAssetDependencyLoader(lite, splitDependencies, flags);
              } catch (SplitAssetDependencyLoader.IllegalDependencyException e) {
                  throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST, e.getMessage());
              }
          } else {
              assetLoader = new DefaultSplitAssetLoader(lite, flags);
          }
  
          try {
              final AssetManager assets = assetLoader.getBaseAssetManager();
              //第二步：parseBaseApk获取Package对象pkg
              final File baseApk = new File(lite.baseCodePath);
              final Package pkg = parseBaseApk(baseApk, assets, flags);
              if (pkg == null) {
                  throw new PackageParserException(INSTALL_PARSE_FAILED_NOT_APK,
                          "Failed to parse base APK: " + baseApk);
              }
  
              if (!ArrayUtils.isEmpty(lite.splitNames)) {
                  final int num = lite.splitNames.length;
                  pkg.splitNames = lite.splitNames;
                  pkg.splitCodePaths = lite.splitCodePaths;
                  pkg.splitRevisionCodes = lite.splitRevisionCodes;
                  pkg.splitFlags = new int[num];
                  pkg.splitPrivateFlags = new int[num];
                  pkg.applicationInfo.splitNames = pkg.splitNames;
                  pkg.applicationInfo.splitDependencies = splitDependencies;
                  pkg.applicationInfo.splitClassLoaderNames = new String[num];
  
                  for (int i = 0; i < num; i++) {
                      final AssetManager splitAssets = assetLoader.getSplitAssetManager(i);
                      //第三步：parseSplitApk
                      parseSplitApk(pkg, i, splitAssets, flags);
                  }
              }
  
              pkg.setCodePath(packageDir.getCanonicalPath());
              pkg.setUse32bitAbi(lite.use32bitAbi);
              return pkg;
          } catch (IOException e) {
              throw new PackageParserException(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                      "Failed to get path: " + lite.baseCodePath, e);
          } finally {
              IoUtils.closeQuietly(assetLoader);
          }
      }
  ```

  

- parseClusterPackageLite

  ```
  static PackageLite parseClusterPackageLite(File packageDir, int flags)
              throws PackageParserException {
          final File[] files = packageDir.listFiles();
          if (ArrayUtils.isEmpty(files)) {
              throw new PackageParserException(INSTALL_PARSE_FAILED_NOT_APK,
                      "No packages found in split");
          }
  
          String packageName = null;
          int versionCode = 0;
  
          Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parseApkLite");
          final ArrayMap<String, ApkLite> apks = new ArrayMap<>();
          for (File file : files) {
              if (isApkFile(file)) {
              	//ApkLite的数据结构？
                  final ApkLite lite = parseApkLite(file, flags);
  
                  // Assert that all package names and version codes are
                  // consistent with the first one we encounter.
                  if (packageName == null) {
                      packageName = lite.packageName;
                      versionCode = lite.versionCode;
                  } else {
                      if (!packageName.equals(lite.packageName)) {
                          throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                                  "Inconsistent package " + lite.packageName + " in " + file
                                  + "; expected " + packageName);
                      }
                      if (versionCode != lite.versionCode) {
                          throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                                  "Inconsistent version " + lite.versionCode + " in " + file
                                  + "; expected " + versionCode);
                      }
                  }
  
                  // Assert that each split is defined only once
                  if (apks.put(lite.splitName, lite) != null) {
                      throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                              "Split name " + lite.splitName
                              + " defined more than once; most recent was " + file);
                  }
              }
          }
          Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
  
          final ApkLite baseApk = apks.remove(null);
          if (baseApk == null) {
              throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                      "Missing base APK in " + packageDir);
          }
  
          // Always apply deterministic ordering based on splitName
          final int size = apks.size();
  
          String[] splitNames = null;
          boolean[] isFeatureSplits = null;
          String[] usesSplitNames = null;
          String[] configForSplits = null;
          String[] splitCodePaths = null;
          int[] splitRevisionCodes = null;
          String[] splitClassLoaderNames = null;
          if (size > 0) {
              splitNames = new String[size];
              isFeatureSplits = new boolean[size];
              usesSplitNames = new String[size];
              configForSplits = new String[size];
              splitCodePaths = new String[size];
              splitRevisionCodes = new int[size];
  
              splitNames = apks.keySet().toArray(splitNames);
              Arrays.sort(splitNames, sSplitNameComparator);
  
              for (int i = 0; i < size; i++) {
                  final ApkLite apk = apks.get(splitNames[i]);
                  usesSplitNames[i] = apk.usesSplitName;
                  isFeatureSplits[i] = apk.isFeatureSplit;
                  configForSplits[i] = apk.configForSplit;
                  splitCodePaths[i] = apk.codePath;
                  splitRevisionCodes[i] = apk.revisionCode;
              }
          }
  
          final String codePath = packageDir.getAbsolutePath();
          return new PackageLite(codePath, baseApk, splitNames, isFeatureSplits, usesSplitNames,
                  configForSplits, splitCodePaths, splitRevisionCodes);
      }
  
  ```

  设计的数据结构PackageLite,ApkLite

- parseApkLite--->parseApkLiteInner--->

[frameworks](http://192.168.160.3:8000/source/xref/u520/android/frameworks/)/[base](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/)/[core](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/core/)/[java](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/core/java/)/[android](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/core/java/android/)/[content](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/core/java/android/content/)/[res](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/core/java/android/content/res/)/[ApkAssets.java](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/core/java/android/content/res/ApkAssets.java)[frameworks](http://192.168.160.3:8000/source/xref/u520/android/frameworks/)/[base](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/)/[core](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/core/)/[java](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/core/java/)/[android](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/core/java/android/)/[content](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/core/java/android/content/)/[res](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/core/java/android/content/res/)/[ApkAssets.java](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/core/java/android/content/res/ApkAssets.java)

[android](http://192.168.160.3:8000/source/xref/u520/android/)/[frameworks](http://192.168.160.3:8000/source/xref/u520/android/frameworks/)/[base](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/)/[libs](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/libs/)/[androidfw](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/libs/androidfw/)/[ApkAssets.cpp](http://192.168.160.3:8000/source/xref/u520/android/frameworks/base/libs/androidfw/ApkAssets.cpp)

会通过ApkAssets.cpp打开apk压缩包。

```
XmlResourceParser parser = null;
        ApkAssets apkAssets = null;
        
parser = apkAssets.openXml(ANDROID_MANIFEST_FILENAME)


```

- parseApkLite(String codePath, XmlPullParser parser, AttributeSet attrs,SigningDetails signingDetails)

  1. 获取应用的包名称parsePackageSplitNames(parser, attrs)
  2. 获取每个AndroidManifest的versioncode、installLocation、versionCodeMajor、revisionCode、coreApp、isolatedSplits、configForSplit、isFeatureSplit、isSplitRequired
  3. 解析<package-verifier><application><uses-split><uses-sdk>获取相应的属性值。
  4. 使用上面获取的信息new ApkLite实例。

- ApkLite baseApk = apks.remove(null);//获取split为null的base apk

  这里的splitName是由parsePackageSplitNames(XmlPullParser parser,AttributeSet attrs) throws IOException, XmlPullParserException, PackageParserException方法赋值的，如果没有split属性就会默认赋值为null

  ```
  <manifest xmlns:android="http://schemas.android.com/apk/res/android"
  3     xmlns:tools="http://schemas.android.com/tools"
  4     package="com.android.phone"
  5     split="com.android.phone.helper"
  6     android:sharedUserId="android.uid.phone" >
  ```

  这里涉及到一个新的概念**split apk.**

- parseBaseApk(baseApk, assets, flags)

  1. 通过baseApk获取apkPath

  2. 通过assets获取XmlResourceParser parser和Resource res实例

     ```
     XmlResourceParser parser = null;
     parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);
     
     final Resources res = new Resources(assets, mMetrics, null);
     ```

  3. 通过parseBaseApk(apkPath, res, parser, flags, outError)获取package

- parseBaseApk(apkPath, res, parser, flags, outError)

  1. new Package(pkgName)
  2. parseBaseApkCommon(pkg, null, res, parser, flags, outError)该方法主要解析<manifest>的子元素，比如<application><uses-sdk><key-sets><permission>
  3. 

- parseBaseApkCommon(pkg, null, res, parser, flags, outError)

  1. parseBaseApplication(pkg, res, parser, flags, outError)解析<application>标签的属性及子元素
  2. 

- parseBaseApplication

  1. 解析<Application>标签的属性信息会保存到ApplicationInfo对象中。
  2. "activity"和"receiver"都是通过parseActivity解析成Activity对象添加到package的activities/receivers集合中。
  3. "service"通过parseService解析成Service对象添加到package的services集合中。
  4. "provider"通过parseProvider解析成Provider对象添加到package的providers集合中。

- **parseActivity解析<activity>流程如下**：

  1. <activity>标签的属性信息会解析成ActivityInfo对象。
  2. 子元素<intent-filter>标签的信息会解析成ActivityIntentInfo对象，并且将AcitivtyIntentInfo对象保存到Activity对象的intents成员中。<intent-filter>子标签<action><category>等信息通过parseIntent进行解析。
  3. activity的<preferred>标签的信息解析成ActivityIntentInfo对象保存到package的preferredActivityFilters集合中。
  4. <meta-data>标签通过parseMetaData解析出来信息会保存到Activity对象的metaData成员中。
  5. <layout>标签通过parseLayout解析出来的信息会保存到ActivityInfo对象的windowLayout成员中。

- parseService解析<service>流程如下：

  1. <service>标签的属性信息会解析成ServiceInfo对象。
  2. 子元素<intent-filter>标签的信息会解析成ServiceIntentInfo对象，并且将ServiceIntentInfo对象保存到Service对象的intents成员中。<intent-filter>子标签<action><category>等信息通过parseIntent进行解析。
  3. <meta-data>标签通过parseMetaData解析出来信息会保存到Service对象的metaData成员中。

- parseProvider

  1. <provider>标签的属性信息会解析成ProviderInfo对象。
  2. <provider>的子标签（元素）<meta-data><grant-uri-permission><intent-filter><path-permission>通过parseProviderTags进行解析。

- 



扫描流程中createNewSetting的调用过程如下，PackageSettings对象的设计是为了packages.xml的<package>标签服务的。

scanPackageChildLI(parseResult.pkg, parseFlags, scanFlags,currentTime, null)--->addForInitLI(pkg, parseFlags, scanFlags, currentTime, user)----->scanPackageNewLI--->scanPackageOnlyLI---->createNewSetting

#### PackageManagerService的main函数

```
 public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        // Self-check for initial settings.
        PackageManagerServiceCompilerMapping.checkProperties();

        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        m.enableSystemUserPackages();
        ServiceManager.addService("package", m);// *注册pms服务*
        final PackageManagerNative pmn = m.new PackageManagerNative();
        ServiceManager.addService("package_native", pmn);
        return m;
    }
    
```



#### PackageManagerService的构造方法

```
// Create sub-components that provide services / data. Order here is important.
        synchronized (mInstallLock) {
        synchronized (mPackages) {
            // Expose private service for system components to use.
            LocalServices.addService(
                    PackageManagerInternal.class, new PackageManagerInternalImpl());
            //UserManagerService相关
            sUserManager = new UserManagerService(context, this,
                    new UserDataPreparer(mInstaller, mInstallLock, mContext, mOnlyCore), mPackages);
            mComponentResolver = new ComponentResolver(sUserManager,
                    LocalServices.getService(PackageManagerInternal.class),
                    mPackages);
            //PermissionManagerService相关
            mPermissionManager = PermissionManagerService.create(context,
                    mPackages /*externalLock*/);
            mDefaultPermissionPolicy = mPermissionManager.getDefaultPermissionGrantPolicy();
            //实例化Settings对象，传入“/data”路径，PermissionSettings对象，mPackages对象
            mSettings = new Settings(Environment.getDataDirectory(),
                    mPermissionManager.getPermissionSettings(), mPackages);
        }
        }
        
        .....
        .....
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "get system config");
        //扫描各个分区下etc/permissions、etc/sysconfig的配置信息
        SystemConfig systemConfig = SystemConfig.getInstance();
        mAvailableFeatures = systemConfig.getAvailableFeatures();
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        .....
        .....
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "read user settings");
        mFirstBoot = !mSettings.readLPw(sUserManager.getUsers(false));
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        
        
        
如下的方法从上往下依次调用


//通过scanDirTracedLI扫描apk，通过scanFlags区分是否是系统app
scanDirTracedLI(File scanDir, final int parseFlags, int scanFlags,
            long currentTime)


//扫描包后会将信息传给Settings
scanPackageLI--->scanPackageChildLI(pkg, parseFlags, scanFlags, currentTime, user);---->addForInitLI---->commitReconciledScanResultLocked----->commitPackageSettings----->mSettings.insertPackageSettingLPw(pkgSetting, pkg);


//将解析出来的信息写入到/data/system/目录下进行保存。
mSettings.writeLPr();
```





#### ./core/java/com/android/server/pm/Settings.java

用于存储系统package相关的信息，比如：

/data/system/packages.list

/data/system/packages.xml

/data/system/packages-backup.xml

/data/system/packages-stopped.xml

/data/system/packages-stopped-backup.xml



/config/sdcardfs用于记录apk分配的linux uid.

/data/system/users/user.id(默认为0)/package-restrictions.xml

##### WritePackageListLPrInternal写packages.list

```
packages.list中每条数据具体的信息含义如下：
// we store on each line the following information for now:
                //
                // pkgName    - package name
                // userId     - application-specific user id
                // debugFlag  - 0 or 1 if the package is debuggable.
                // dataPath   - path to package's data path
                // seinfo     - seinfo label for the app (assigned at install time)
                // gids       - supplementary gids this app launches with
                // profileableFromShellFlag  - 0 or 1 if the package is profileable from shell.
                // longVersionCode - integer version of the package.
                //
                // NOTE: We prefer not to expose all ApplicationInfo flags for now.
                //
                // DO NOT MODIFY THIS FORMAT UNLESS YOU CAN ALSO MODIFY ITS USERS
                // FROM NATIVE CODE. AT THE MOMENT, LOOK AT THE FOLLOWING SOURCES:
                //   system/core/libpackagelistparser
                //
```



#### WriteLPr()

```



serializer.startTag(null, "permission-trees");
mPermissions.writePermissionTrees(serializer);
serializer.endTag(null, "permission-trees");

serializer.startTag(null, "permissions");
mPermissions.writePermissions(serializer);
serializer.endTag(null, "permissions");
   

for (final PackageSetting pkg : mPackages.values()) {
    writePackageLPr(serializer, pkg);//通过writePackageLPr写packaes.xml文件的<package></package>
}

// List of replaced system applications
private final ArrayMap<String, PackageSetting> mDisabledSysPackages =
        new ArrayMap<String, PackageSetting>();
for (final PackageSetting pkg : mDisabledSysPackages.values()) {
                writeDisabledSysPackageLPr(serializer, pkg);//这里记录的是update的包信息。写<updated-package></updated-package>
}

.....
.....
.....


// New settings successfully written, old ones are no longer
// needed.
mBackupSettingsFilename.delete();//清除备份文件packages-backup.xml
//修改packages.xml的文件权限
FileUtils.setPermissions(mSettingsFilename.toString(),
           FileUtils.S_IRUSR|FileUtils.S_IWUSR
           |FileUtils.S_IRGRP|FileUtils.S_IWGRP,
           -1, -1);

///** The top level directory in configfs for sdcardfs to push the package->uid,userId mappings */
//private final File mKernelMappingFilename;
writeKernelMappingLPr();//在/config/sdcardfs下会记录KernelPackageState

writePackageListLPr();//写data/system/packages.list

writeAllUsersPackageRestrictionsLPr();//写入到/data/system/users/user.id(默认为0)/package-restrictions.xml

writeAllRuntimePermissionsLPr();

比如updated-package记录的信息如下：
packagesManagerService$ grep -nri -E "updated-package name=" packages.xml 
12662:    <updated-package name="com.amazon.mShop.android.shopping" codePath="/system/app/AmazonShopping" ft="11e8f7d4c00" it="11e8f7d4c00" ut="11e8f7d4c00" version="1241192011" nativeLibraryPath="/system/app/AmazonShopping/lib" primaryCpuAbi="arm64-v8a" userId="10129">
12693:    <updated-package name="com.google.android.youtube" codePath="/product/app/YouTube" ft="11e8f7d4c00" it="11e8f7d4c00" ut="11e8f7d4c00" version="1516625344" nativeLibraryPath="/product/app/YouTube/lib" primaryCpuAbi="arm64-v8a" userId="10155">

实际安装的位置为/data/app该路径说明该apk进行了在线更新：
packagesManagerService$ adb shell pm path com.amazon.mShop.android.shopping
package:/data/app/~~8LmCdWu_vluocl1S3JOaPw==/com.amazon.mShop.android.shopping-CvBS90dXqllKcnUy8wz_Tw==/base.apk

```



#### readLPw

```
boolean readLPw(@NonNull List<UserInfo> users) {
    FileInputStream str = null;
    //判断packages-backup.xml是否存在。
    if (mBackupSettingsFilename.exists()) {
        try {
            str = new FileInputStream(mBackupSettingsFilename);
            mReadMessages.append("Reading from backup settings file\n");
            PackageManagerService.reportSettingsProblem(Log.INFO,
                    "Need to read from backup settings file");
            if (mSettingsFilename.exists()) {
                // If both the backup and settings file exist, we
                // ignore the settings since it might have been
                // corrupted.
                Slog.w(PackageManagerService.TAG, "Cleaning up settings file "
                        + mSettingsFilename);
                mSettingsFilename.delete();
            }
        } catch (java.io.IOException e) {
            // We'll try for the normal settings file.
        }
    }

    mPendingPackages.clear();
    mPastSignatures.clear();
    mKeySetRefs.clear();
    mInstallerPackages.clear();

    try {
        if (str == null) {
            if (!mSettingsFilename.exists()) {
                mReadMessages.append("No settings file found\n");
                PackageManagerService.reportSettingsProblem(Log.INFO,
                        "No settings file; creating initial state");
                // It's enough to just touch version details to create them
                // with default values
                findOrCreateVersion(StorageManager.UUID_PRIVATE_INTERNAL).forceCurrent();
                findOrCreateVersion(StorageManager.UUID_PRIMARY_PHYSICAL).forceCurrent();
                return false;
            }
            str = new FileInputStream(mSettingsFilename);
        }
        XmlPullParser parser = Xml.newPullParser();
        parser.setInput(str, StandardCharsets.UTF_8.name());

        int type;
        while ((type = parser.next()) != XmlPullParser.START_TAG
                && type != XmlPullParser.END_DOCUMENT) {
            ;
        }

        if (type != XmlPullParser.START_TAG) {
            mReadMessages.append("No start tag found in settings file\n");
            PackageManagerService.reportSettingsProblem(Log.WARN,
                    "No start tag found in package manager settings");
            Slog.wtf(PackageManagerService.TAG,
                    "No start tag found in package manager settings");
            return false;
        }

        int outerDepth = parser.getDepth();
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("package")) {//解析package标签
                readPackageLPw(parser);
            } 、、、、
            、、、、、
            } else if (tagName.equals("shared-user")) {//解析shared-user标签
                readSharedUserLPw(parser);
            } 、、、
            、、、、
        }

        str.close();

    } catch (XmlPullParserException e) {
       
       、、、、
       、、、、
       
    }

	//处理mPendingPackages集合内的数据
   、、、、
   、、、、
   、、、、

    if (mBackupStoppedPackagesFilename.exists()
            || mStoppedPackagesFilename.exists()) {
        // Read old file
        readStoppedLPw();
        mBackupStoppedPackagesFilename.delete();
        mStoppedPackagesFilename.delete();
        // Migrate to new file format
        writePackageRestrictionsLPr(UserHandle.USER_SYSTEM);
    } else {
        for (UserInfo user : users) {
            readPackageRestrictionsLPr(user.id);
        }
    }

    for (UserInfo user : users) {
        mRuntimePermissionsPersistence.readStateForUserSyncLPr(user.id);
    }

    /*
     * Make sure all the updated system packages have their shared users
     * associated with them.
     */
    final Iterator<PackageSetting> disabledIt = mDisabledSysPackages.values().iterator();
    while (disabledIt.hasNext()) {
        final PackageSetting disabledPs = disabledIt.next();
        final Object id = getSettingLPr(disabledPs.appId);
        if (id != null && id instanceof SharedUserSetting) {
            disabledPs.sharedUser = (SharedUserSetting) id;
        }
    }

    mReadMessages.append("Read completed successfully: " + mPackages.size() + " packages, "
            + mSharedUsers.size() + " shared uids\n");

    writeKernelMappingLPr();

    return true;
}
```



##### **readPackageLPw**

```
private void readPackageLPw(XmlPullParser parser) throws XmlPullParserException, IOException {
   、、、、
   、、、、
   
    long versionCode = 0;
    String parentPackageName;
    try {
        name = parser.getAttributeValue(null, ATTR_NAME);
        realName = parser.getAttributeValue(null, "realName");
        idStr = parser.getAttributeValue(null, "userId");
        uidError = parser.getAttributeValue(null, "uidError");
        sharedIdStr = parser.getAttributeValue(null, "sharedUserId");
        
    、、、、
    、、、、
    
        
        final int userId = idStr != null ? Integer.parseInt(idStr) : 0;
        final int sharedUserId = sharedIdStr != null ? Integer.parseInt(sharedIdStr) : 0;
  
  	、、、、
  	、、、、
        } else if (userId > 0) {//如果有独立的linux uid
            packageSetting = addPackageLPw(name.intern(), realName, new File(codePathStr),
                    new File(resourcePathStr), legacyNativeLibraryPathStr, primaryCpuAbiString,
                    secondaryCpuAbiString, cpuAbiOverrideString, userId, versionCode, pkgFlags,
                    pkgPrivateFlags, parentPackageName, null /*childPackageNames*/,
                    null /*usesStaticLibraries*/, null /*usesStaticLibraryVersions*/);
            if (PackageManagerService.DEBUG_SETTINGS)
                Log.i(PackageManagerService.TAG, "Reading package " + name + ": userId="
                        + userId + " pkg=" + packageSetting);
            if (packageSetting == null) {
                PackageManagerService.reportSettingsProblem(Log.ERROR, "Failure adding uid "
                        + userId + " while parsing settings at "
                        + parser.getPositionDescription());
            } else {
                packageSetting.setTimeStamp(timeStamp);
                packageSetting.firstInstallTime = firstInstallTime;
                packageSetting.lastUpdateTime = lastUpdateTime;
            }
        } else if (sharedIdStr != null) {//上一次记录显示分配了共享的uid，但该共享uid的有效性需要验证。
            if (sharedUserId > 0) {
                packageSetting = new PackageSetting(name.intern(), realName, new File(
                        codePathStr), new File(resourcePathStr), legacyNativeLibraryPathStr,
                        primaryCpuAbiString, secondaryCpuAbiString, cpuAbiOverrideString,
                        versionCode, pkgFlags, pkgPrivateFlags, parentPackageName,
                        null /*childPackageNames*/, sharedUserId,
                        null /*usesStaticLibraries*/, null /*usesStaticLibraryVersions*/);
                packageSetting.setTimeStamp(timeStamp);
                packageSetting.firstInstallTime = firstInstallTime;
                packageSetting.lastUpdateTime = lastUpdateTime;
                mPendingPackages.add(packageSetting);//暂时Pending不分配uid
                if (PackageManagerService.DEBUG_SETTINGS)
                    Log.i(PackageManagerService.TAG, "Reading package " + name
                            + ": sharedUserId=" + sharedUserId + " pkg=" + packageSetting);
            } else {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Error in package manager settings: package " + name
                                + " has bad sharedId " + sharedIdStr + " at "
                                + parser.getPositionDescription());
            }
        } else {
            PackageManagerService.reportSettingsProblem(Log.WARN,
                    "Error in package manager settings: package " + name + " has bad userId "
                            + idStr + " at " + parser.getPositionDescription());
        }
    } catch (NumberFormatException e) {
        PackageManagerService.reportSettingsProblem(Log.WARN,
                "Error in package manager settings: package " + name + " has bad userId "
                        + idStr + " at " + parser.getPositionDescription());
    }
    if (packageSetting != null) {
        packageSetting.uidError = "true".equals(uidError);
        packageSetting.installerPackageName = installerPackageName;
        packageSetting.isOrphaned = "true".equals(isOrphaned);
        packageSetting.volumeUuid = volumeUuid;
        packageSetting.categoryHint = categoryHint;
        packageSetting.legacyNativeLibraryPathString = legacyNativeLibraryPathStr;
        packageSetting.primaryCpuAbiString = primaryCpuAbiString;
        packageSetting.secondaryCpuAbiString = secondaryCpuAbiString;
        packageSetting.updateAvailable = "true".equals(updateAvailable);
        // Handle legacy string here for single-user mode
        final String enabledStr = parser.getAttributeValue(null, ATTR_ENABLED);
        if (enabledStr != null) {
            try {
                packageSetting.setEnabled(Integer.parseInt(enabledStr), 0 /* userId */, null);
            } catch (NumberFormatException e) {
                、、、
                、、、
            }
        } else {
            packageSetting.setEnabled(COMPONENT_ENABLED_STATE_DEFAULT, 0, null);
        }
			、、、、
            、、、、
    } else {
        XmlUtils.skipCurrentTag(parser);
    }
}
```



###### **addPackageLPw**

```
PackageSetting addPackageLPw(String name, String realName, File codePath, File resourcePath,
        String legacyNativeLibraryPathString, String primaryCpuAbiString,
        String secondaryCpuAbiString, String cpuAbiOverrideString, int uid, long vc, int
        pkgFlags, int pkgPrivateFlags, String parentPackageName,
        List<String> childPackageNames, String[] usesStaticLibraries,
        long[] usesStaticLibraryNames) {
    PackageSetting p = mPackages.get(name);
    if (p != null) {
        if (p.appId == uid) {
            return p;
        }
        PackageManagerService.reportSettingsProblem(Log.ERROR,
                "Adding duplicate package, keeping first: " + name);
        return null;
    }
    p = new PackageSetting(name, realName, codePath, resourcePath,
            legacyNativeLibraryPathString, primaryCpuAbiString, secondaryCpuAbiString,
            cpuAbiOverrideString, vc, pkgFlags, pkgPrivateFlags, parentPackageName,
            childPackageNames, 0 /*userId*/, usesStaticLibraries, usesStaticLibraryNames);
    p.appId = uid;
    if (registerExistingAppIdLPw(uid, p, name)) {//
        mPackages.put(name, p);
        return p;
    }
    return null;
}
```

1. 先判断添加的packagename是否已经存在于mPackages，如果存在就不用分配uid了。

2. 如果mPackages中packagename不存在，将该package的信息封装成PackageSetting对象，并将uid分配给该package,然后将package存入mPackages中。

   ```
   /** Returns true if the requested AppID was valid and not already registered. */
   private boolean registerExistingAppIdLPw(int appId, SettingBase obj, Object name) {
       if (appId > Process.LAST_APPLICATION_UID) {// 10000 <= 安装apk uid <= 19999
           return false;
       }
   
       if (appId >= Process.FIRST_APPLICATION_UID) {
           int size = mAppIds.size();
           final int index = appId - Process.FIRST_APPLICATION_UID;
           // fill the array until our index becomes valid
           while (index >= size) {
               mAppIds.add(null);
               size++;
           }
           if (mAppIds.get(index) != null) {
               PackageManagerService.reportSettingsProblem(Log.ERROR,
                       "Adding duplicate app id: " + appId
                       + " name=" + name);
               return false;
           }
           mAppIds.set(index, obj);//mAppIds集合用来管理已经分配uid的package.
       } else {
           if (mOtherAppIds.get(appId) != null) {
               PackageManagerService.reportSettingsProblem(Log.ERROR,
                       "Adding duplicate shared id: " + appId
                               + " name=" + name);
               return false;
           }
           mOtherAppIds.put(appId, obj);//mOtherAppIds集合用来管理特权uid的package
       }
       return true;
   }
   ```

##### readSharedUserLPw

```
private void readSharedUserLPw(XmlPullParser parser) throws XmlPullParserException,IOException {
    String name = null;
    String idStr = null;
    int pkgFlags = 0;
    int pkgPrivateFlags = 0;
    SharedUserSetting su = null;
    try {
        name = parser.getAttributeValue(null, ATTR_NAME);
        idStr = parser.getAttributeValue(null, "userId");
        int userId = idStr != null ? Integer.parseInt(idStr) : 0;
        if ("true".equals(parser.getAttributeValue(null, "system"))) {
            pkgFlags |= ApplicationInfo.FLAG_SYSTEM;
        }
        if (name == null) {
            PackageManagerService.reportSettingsProblem(Log.WARN,
                    "Error in package manager settings: <shared-user> has no name at "
                            + parser.getPositionDescription());
        } else if (userId == 0) {
            PackageManagerService.reportSettingsProblem(Log.WARN,
                    "Error in package manager settings: shared-user " + name
                            + " has bad userId " + idStr + " at "
                            + parser.getPositionDescription());
        } else {
            if ((su = addSharedUserLPw(name.intern(), userId, pkgFlags, pkgPrivateFlags))
                    == null) {
                PackageManagerService
                        .reportSettingsProblem(Log.ERROR, "Occurred while parsing settings at "
                                + parser.getPositionDescription());
            }
        }
    } catch (NumberFormatException e) {
        PackageManagerService.reportSettingsProblem(Log.WARN,
                "Error in package manager settings: package " + name + " has bad userId "
                        + idStr + " at " + parser.getPositionDescription());
    }

	
    if (su != null) {
        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }
			//解析shared-user的子元素
            String tagName = parser.getName();
            if (tagName.equals("sigs")) {
                、、、、
            } else if (tagName.equals("perms")) {
                、、、、
            } else {
                、、、、
            }
        }
    } else {
        XmlUtils.skipCurrentTag(parser);
    }
}
```



###### addSharedUserLPw

```
SharedUserSetting addSharedUserLPw(String name, int uid, int pkgFlags, int pkgPrivateFlags) {
    //@note
    //查看相应的name是否存在，如果已经存在直接返回ShareUserSetting对象，如果不存在就创建相应name的SharedUserSetting并保留在集合里。
    SharedUserSetting s = mSharedUsers.get(name);
    if (s != null) {
        if (s.userId == uid) {
            return s;
        }
        PackageManagerService.reportSettingsProblem(Log.ERROR,
                "Adding duplicate shared user, keeping first: " + name);
        return null;
    }
    s = new SharedUserSetting(name, pkgFlags, pkgPrivateFlags);
    s.userId = uid;
    if (registerExistingAppIdLPw(uid, s, name)) {
        mSharedUsers.put(name, s);
        return s;
    }
    return null;
}
```



##### 处理readLPr（）->mPendingPackages集合内的数据

```
final int N = mPendingPackages.size();

for (int i = 0; i < N; i++) {
    final PackageSetting p = mPendingPackages.get(i);
    final int sharedUserId = p.getSharedUserId();
    final Object idObj = getSettingLPr(sharedUserId);
    if (idObj instanceof SharedUserSetting) {
        final SharedUserSetting sharedUser = (SharedUserSetting) idObj;
        p.sharedUser = sharedUser;
        p.appId = sharedUser.userId;
        addPackageSettingLPw(p, sharedUser);
    } else if (idObj != null) {
        String msg = "Bad package setting: package " + p.name + " has shared uid "
                + sharedUserId + " that is not a shared uid\n";
        mReadMessages.append(msg);
        PackageManagerService.reportSettingsProblem(Log.ERROR, msg);
    } else {
        String msg = "Bad package setting: package " + p.name + " has shared uid "
                + sharedUserId + " that is not defined\n";
        mReadMessages.append(msg);
        PackageManagerService.reportSettingsProblem(Log.ERROR, msg);
    }
}
mPendingPackages.clear();
```



#### 分配linux UID



```
/**
 * Registers a user ID with the system. Potentially allocates a new user ID.
 * @return {@code true} if a new app ID was created in the process. {@code false} can be
 *         returned in the case that a shared user ID already exists or the explicit app ID is
 *         already registered.
 * @throws PackageManagerException If a user ID could not be allocated.
 */
boolean registerAppIdLPw(PackageSetting p) throws PackageManagerException {
    final boolean createdNew;
    if (p.appId == 0) {
        // Assign new user ID
        p.appId = acquireAndRegisterNewAppIdLPw(p);
        createdNew = true;
    } else {
        // Add new setting to list of user IDs
        createdNew = registerExistingAppIdLPw(p.appId, p, p.name);
    }
    if (p.appId < 0) {
        PackageManagerService.reportSettingsProblem(Log.WARN,
                "Package " + p.name + " could not be assigned a valid UID");
        throw new PackageManagerException(INSTALL_FAILED_INSUFFICIENT_STORAGE,
                "Package " + p.name + " could not be assigned a valid UID");
    }
    return createdNew;
}
```











C++ 强制类型转换运算符的用法如下：

​	强制类型转换运算符 <要转换到的类型> (待转换的表达式)

### C++的四种强制转换类型

1. static_cast

   ```
   static_cast 用于进行比较“自然”和低风险的转换，如整型和浮点型、字符型之间的互相转换。
   
   ```

   

2. reinterpret_cast

   ```
   reinterpret_cast 用于进行各种不同类型的指针之间、不同类型的引用之间以及指针和能容纳指针的整数类型之间的转换。转换时，执行的是逐个比特复制的操作。
   这种转换提供了很强的灵活性，但转换的安全性只能由程序员的细心来保证了
   ```

   

3. const_cast

   ```
   const_cast 运算符仅用于进行去除 const 属性的转换，它也是四个强制类型转换运算符中唯一能够去除 const 属性的运算符。
   
   将 const 引用转换为同类型的非 const 引用，将 const 指针转换为同类型的非 const 指针时可以使用 const_cast 运算符。
   ```

   

4. dynamic_cast

   ```
   用 reinterpret_cast 可以将多态基类（包含虚函数的基类）的指针强制转换为派生类的指针，但是这种转换不检查安全性，即不检查转换后的指针是否确实指向一个派生类对象。dynamic_cast专门用于将多态基类的指针或引用强制转换为派生类的指针或引用，而且能够检查转换的安全性。对于不安全的指针转换，转换结果返回 NULL 指针。
   
   dynamic_cast 是通过“运行时类型检查”来保证安全性的。dynamic_cast 不能用于将非多态基类的指针或引用强制转换为派生类的指针或引用
   ```

   来源http://c.biancheng.net/view/410.html

   https://zhuanlan.zhihu.com/p/101493574

   https://www.cnblogs.com/linuxAndMcu/p/10387829.html

   

split apk参考资料

https://blog.csdn.net/feelinghappy/article/details/80744966



http://vanelst.site/2020/03/17/pms-install/

https://blog.csdn.net/yiranfeng/article/details/104073200

https://www.codenong.com/cs105232472/

https://blog.csdn.net/qq_31429205/article/details/105232472