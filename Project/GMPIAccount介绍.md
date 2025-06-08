# GMPIAccount 项目阐述

## 一、项目背景

    在车机Infotainment系统中，Android用户账户管理是系统的核心功能之一。随着用户需要使用的在线应用和服务越来越多。提供一个统一、安全且便捷的账户管理系统是必须的。GMPIAccount 系统应用的开发，针对GMPI Local Vehicle，提供集中式的账户管理解决方案。

## 二、架构设计

   GMPIAccount 项目应用采用分层架构设计，主要分为以下三层：

1. **表现层（Presentation Layer）**
   
   负责与用户进行交互，呈现账户管理界面。这一层使用Android 的视图系统，包括 Activity、Fragment 等组件。例如，账户列表界面使用 RecyclerView 展示已添加的账户账户图标、名称等信息。登录账户界面则通过一系列 EditText 和 Button 组件，引导用户输入账户相关信息进行登录操作。同时遵循 Material Design 设计规范。

2. **业务逻辑层（Business Logic Layer）**
   
   处理账户管理的核心业务逻辑。它接收来自表现层的用户操作请求，如添加账户、删除账户、修改账户信息等，并调用数据访问层的接口进行数据处理。同时，负责处理认证逻辑，如验证用户输入的账户密码，账户验证码是否正确，通过与SGM认证服务器进行交互完成认证过程。此外，还管理账户数据的同步任务，决定何时以及如何将本地账户数据同步到SGM服务器。

3. **数据访问层（Data Access Layer）**
   
   负责与数据存储进行交互。在本地，使用Android的SQLite数据库存储账户的基本信息。如账户名称、类型、登录凭证（UserId, token）等。对于一些敏感信息，考虑安全性，不做存储，只存储相关凭证信息。
   
   在与服务器交互方面，通过 HTTP/HTTPS 协议与账户服务器进行通信，获取或更新账户数据。例如，在登录/注册账户时，将用户录入的信息发送到服务器进行注册，并接收服务器返回的注册结果。

## 三、技术实现

### （一）开发框架和技术栈

   主要分为三个模块（gmpisdk，authtokenservice，GMPIAccount）

1. **gmpisdk模块：负责 AIDL 接口定义**
   
   对于自定义的数据类型，需要实现 `Parcelable` 接口。当传递复杂的数据结构（如嵌套的列表、映射等）时，要确保其在 `Parcel` 中的序列化和反序列化过程准确无误。
   
   **1）自定义 Parcelable 类型 `UserInfo.java`**
   
   ```java
   import android.os.Parcel;
   import android.os.Parcelable;
   
   public class UserInfo implements Parcelable {
       private String name;
       private int age;
   
       public UserInfo(String name, int age) {
           this.name = name;
           this.age = age;
       }
   
       protected UserInfo(Parcel in) {
           name = in.readString();
           age = in.readInt();
       }
   
       public static final Creator<UserInfo> CREATOR = new Creator<UserInfo>() {
           @Override
           public UserInfo createFromParcel(Parcel in) {
               return new UserInfo(in);
           }
   
           @Override
           public UserInfo[] newArray(int size) {
               return new UserInfo[size];
           }
       };
   
       public String getName() {
           return name;
       }
   
       public int getAge() {
           return age;
       }
   
       @Override
       public int describeContents() {
           return 0;
       }
   
       @Override
       public void writeToParcel(Parcel dest, int flags) {
           dest.writeString(name);
           dest.writeInt(age);
       }
   }
   ```
   
   **2）定义 AIDL 接口 IAuthTokenService.aidl**
   
   ```aidl
   // 在 aidl 文件中导入自定义 Parcelable 类型
   import com.example.gmpisdk.UserInfo;
   
   package com.example.gmpisdk;
   import android.os.Bundle;
   
   interface IAuthTokenService {
       // 定义带有基础类型和自定义Parcelable类型参数的方法，返回值带有Bundle
       Bundle getToken(String username, UserInfo userInfo);
   }
   ```

2. **authtokenservice模块：自定义系统服务**  
   
   **1）实现 AIDL 接口**
   
   在 `authtokenservice` 模块中实现 `IAuthTokenService` 接口的Stub。
   
   **实现接口 `AuthTokenServiceImpl.java`**
   
   ```java
   import android.os.Binder;
   import android.os.IBinder;
   import android.os.RemoteException;
   import android.util.Log;
   
   import com.example.gmpisdk.IAuthTokenService;
   import com.example.gmpisdk.UserInfo;
   
   public class AuthTokenServiceImpl extends IAuthTokenService.Stub {
       private static final String TAG = "AuthTokenServiceImpl";
       // 大致逻辑如下伪代码
       // 1.发起网络请求
       private OkHttpClient client = new OkHttpClient();
       private String fetchUserData() throws IOException {
           Request request = new Request.Builder()
                 .url("https://example.com/api/user")
                 .build();
           try (Response response = client.newCall(request).execute()) {
               return response.body().string();
           }
       }
       // 2. 将网络数据转换为 Bundle, 获取到的是JSON数据，使用解析库Gson或org.json包。这里以 Gson 为例，先添加 Gson 依赖：
       private Bundle convertToBundle(String jsonData) {
           Gson gson = new Gson();
           JsonObject jsonObject = gson.fromJson(jsonData, JsonObject.class);
           Bundle bundle = new Bundle();
           bundle.putString("username", jsonObject.get("name").getAsString());
           bundle.putInt("userage", jsonObject.get("age").getAsInt());
           return bundle;
       }
   
       // 3. aidl getToken 返回
       @Override
       public Bundle getToken(String username, UserInfo userInfo) throws RemoteException {
           Log.d(TAG, "Received request for user: " + username + ", age: " + userInfo.getAge());
           try {
               String jsonData = fetchUserData();
               return convertToBundle(jsonData);
           } catch (IOException e) {
               e.printStackTrace();
               return null;
           }
       }
   }
   ```
   
   **2）在 Application 中添加服务**
   
   **创建自定义Application类并添加服务 , 关键方法 ServiceManager.addService()**
   
   ```java
   import android.app.Application;
   import android.os.ServiceManager;
   import android.util.Log;
   
   import com.example.authtokenservice.AuthTokenServiceImpl;
   
   public class CustomApp extends Application {
       private static final String TAG = "CustomApp";
   
       @Override
       public void onCreate() {
           super.onCreate();
           registerCustomService();
       }
   
       private void registerCustomService() {
           AuthTokenServiceImpl service = new AuthTokenServiceImpl();
           boolean result = ServiceManager.addService("auth_token_service", service);
           if (result) {
               Log.d(TAG, "Auth Token Service registered successfully");
           } else {
               Log.e(TAG, "Failed to register Auth Token Service");
           }
       }
   }
   ```
   
   **3）配置 AndroidManifest.xml**
   
   ```xml
   <manifest xmlns:android="http://schemas.android.com/apk/res/android"
       package="com.example.authtokenservice">
   
       <uses-permission android:name="android.permission.INTERACT_ACROSS_USERS_FULL" />
       <uses-permission android:name="android.permission.BIND_QUICK_SETTINGS_TILE" />
       <!-- 根据实际需求添加其他权限 -->
   
       <application
           android:name=".CustomApp"
           android:sharedUserId="android.uid.system"
           android:persistent="true"
           android:allowBackup="true"
           android:icon="@mipmap/ic_launcher"
           android:label="@string/app_name"
           android:roundIcon="@mipmap/ic_launcher_round"
           android:supportsRtl="true"
           android:theme="@style/AppTheme">
   
       </application>
   
   </manifest>
   ```
   
   **`android:sharedUserId="android.uid.system"` 的作用**
   
   **权限共享与系统级访问**：当一个应用设置了，它就与系统框架运行在同一个 Linux 用户 ID 下，即 `android.uid.system`。这意味着该应用可以访问系统级别的资源和权限，而这些资源和权限通常是普通应用无法获取的。例如，它可以访问系统设置数据库，直接操作一些系统级的服务接口等。这在开发系统应用或需要与系统深度集成的应用时非常有用。
   
   **进程间通信与数据共享**：在进程间通信（IPC）方面，具有相同 `sharedUserId` 的应用可以共享数据和内存空间，方便进行进程间的数据传递和交互。例如，一个系统应用需要与某个系统服务频繁交换数据，通过设置相同的 `sharedUserId`，可以简化 IPC 机制，提高通信效率，减少因权限限制导致的通信问题。
   
   **`android:persistent="true"` 的作用**
   
   **系统启动时自动启动**：系统在启动过程中会自动启动该应用及其相关的组件，确保了应用提供的功能在系统启动后立即可用。例如，一些系统关键服务，如电源管理服务、网络管理服务等。在AMS 走 systemReady方法时，会从拿到PMS里拿带有这个属性的应用，然后AMS启动这个应用，这个启动方式远早于startHomeActivty的，比开机广播快很多。
   
   **常驻内存与资源分配**：系统会将设置为 `persistent` 的应用视为重要的、需要持续运行的组件，会倾向于为其分配更多的系统资源，包括内存、CPU 时间片等。在系统资源紧张时，系统会尽量避免终止这类应用，以保证其功能的持续性。例如，音乐播放应用如果设置为持久化，在用户切换应用或设备内存不足时，系统会优先保留音乐播放应用的进程，确保音乐播放不会中断。
   
   **应用生命周期管理**：对于具有持久化属性的应用，其生命周期管理也有所不同。即使应用的某些组件（如 Activity）被用户关闭或由于内存不足被系统回收，系统也会尝试在适当的时候重新启动这些组件，以维持应用的正常运行。
   
   **4）SeLinux 权限**
   
   在上述 ServiceManager addService 时遇到日志提示 SELinux : avc: denied 问题，该如何解决？
   
   修改 service_contexts，一般修改`/system/sepolicy/service_contexts`， 作用将服务名映射到对应的 SELinux 类型（Type）
   
   ```plaintext
   /system/sepolicy/service_contexts  # 系统预置的服务上下文配置
   /vendor/etc/selinux/service_contexts  # 厂商自定义的服务上下文配置
   ```
   
   修改 service.te，一般修改`system/sepolicy/private/service.te`，定义服务的 SELinux 类型（Type）及其属性，控制服务之间的访问规则
   
   ```plaintext
   system/sepolicy/private/service.te  # 系统核心服务策略
   device/<厂商>/sepolicy/common/service.te  # 厂商通用策略
   ```

3. **GMPIAccount模块：自定义系统应用**
   
   **1）获取服务接口**
   
   ```java
   import android.os.IBinder;
   import android.os.ServiceManager;
   import android.os.RemoteException;
   import android.util.Log;
   
   import com.example.IAutoTokenService;
   
   public class GMPIAccountUtil {
       private static final String TAG = "GMPIAccountUtil";
       private static IAutoTokenService getAutoTokenService() {
           IBinder binder = ServiceManager.getService("auth_token_service");
           if (binder == null) {
               Log.e(TAG, "Failed to get autotokenservice binder");
               return null;
           }
           return IAuthTokenService.Stub.asInterface(binder);
       }
   }
   ```
   
   **2）调用服务方法 （调用时必须注意异常的处理，Null和RemoteException）**
   
   ```java
   public void requestToken(String username, UserInfo userInfo) {
      IAuthTokenService authTokenService = GMPIAccountUtil.getAutoTokenService();
      if (authTokenService!= null) {
         try {
           String token = authTokenService.getToken(username, userInfo);
           Log.d(TAG, "Received token: " + token);
         } catch (RemoteException e) {
           Log.e(TAG, "Remote exception while calling getToken", e);
         }
      }
   }
   ```

### （二）关键技术点与解决方案

#### 实现车机端在不同屏幕，应用不同皮肤的效果

- 通过Runtime Resource Overlay（RRO），Android的一种在运行时更改目标应用所使用资源值的机制。
  
  #### 1. 创建 RRO 模块
  
  1. **项目初始化**：
     - 在 Android 开发环境（如 Android Studio）中创建一个新的 Android 项目，该项目将作为 RRO 模块。
     - 在 `build.gradle` 文件中配置模块相关属性，例如设置模块类型为 `library` ，因为 RRO 模块本质上是一个资源库。
  
  ```groovy
  apply plugin: 'com.android.library'
  
  android {
     ...
  }
  
  dependencies {
      implementation 'androidx.appcompat:appcompat:1.6.1'
  }
  ```
  
  2. **配置 `AndroidManifest.xml`**：
     - 通过设置 `android:overlay="true"` 表明这是一个资源覆盖模块，
     - 使用 `android:targetPackage` 属性指定要覆盖资源的目标应用包名。
     - `android:isStatic="false"` 表示这是一个动态 RRO，可在运行时加载和卸载；若设置为 `true` 则为静态 RRO，会在系统启动时加载。
  
  ```xml
  <manifest xmlns:android="http://schemas.android.com/apk/res/android"
      package="com.example.rroproject">
      <overlay android:targetPackage="com.target.app"
               android:overlay="true"
               android:isStatic="false"/>
  </manifest>
  ```
  
  #### 2. 准备替换资源
  
  - 在 RRO 模块的 `res` 目录下，按照与目标应用相同的资源结构组织要替换的资源。例如，如果目标应用的某个布局文件路径为 `res/layout/main_activity.xml`，那么在 RRO 模块的 `res/layout` 目录下创建同名的 `main_activity.xml` 文件来替换它。
  - 对于其他类型资源（如 `drawable`、`values` 等）同理。例如，若要替换目标应用的颜色资源，可以在 RRO 模块的 `res/values/colors.xml` 文件中定义相同名称的颜色资源。
  
  #### 3. 编译与部署 RRO 模块
  
  1. **编译**：
     - 在开发环境中编译 RRO 模块，生成对应的 APK 文件。如果是在系统源码环境下开发，可能需要使用特定的编译命令（如 `mmm` 命令）进行编译。
  2. **部署**：
     - **动态 RRO**：可以通过 `adb install` 命令将生成的 APK 安装到设备上。在安装后，需要通过系统 API 或特定工具来加载 RRO。例如，在系统设置中有专门的 “主题” 或 “资源替换” 选项，用户可以选择启用已安装的 RRO 模块。
     - **静态 RRO**：将编译好的 APK push到系统镜像的 `/system/overlay` 目录下。在系统启动过程中，系统会自动加载 `/system/overlay` 目录下的所有静态 RRO 模块。
  
  #### 4. 加载与生效
  
  1. **针对动态 RRO**：
     - 系统提供了相关的 API 来动态加载 RRO 模块。应用或系统组件可以通过调用这些 API，在运行时加载指定的 RRO 模块。例如，通过 `PackageManager` 的相关方法来激活 RRO。一旦加载成功，RRO 中的资源就会按照资源加载优先级规则，在目标应用请求相应资源时生效。
  2. **针对静态 RRO**：
     - 当系统启动时，会扫描 `/system/overlay` 目录，加载其中的所有静态 RRO 模块。在目标应用启动或请求资源时，系统会优先从已加载的静态 RRO 模块中查找匹配的资源，若找到则使用 RRO 模块中的资源，从而实现资源的覆盖。

#### 百度SDK的接入

- 在GMPIAccount系统应用中，根据不同场景，实现百度账号的绑定，解绑，状态获取的API接口调用。必要时把对应状态通过Shareprefence或SQLite进行存储。一般第三方SDK在应用启动时一起初始化，在需要使用的地方调用对应API。

### （三）性能优化

1. **内存优化**：通过使用内存分析工具（如 Android Profiler），对应用的内存使用情况进行实时监测。避免内存泄漏，例如在 Activity 和 Fragment 销毁时，及时释放资源，取消注册的监听器等。对于频繁创建和销毁的对象，采用对象池技术进行复用，减少内存分配和垃圾回收的开销。
   
      **关于authtoken模块需要注意的点**
   
   - **网络请求的线程管理**：网络请求是耗时操作，不能在主线程中进行。可以使用线程池、AsyncTask 或 Kotlin 的协程来处理网络请求，避免阻塞服务的主线程。
   - **错误处理**：在获取网络数据、解析数据以及通过 AIDL 接口传递数据过程中，都可能出现错误。要对各种可能的异常进行捕获和处理，如网络连接失败、JSON 解析错误、远程调用异常等，确保服务的稳定性和健壮性。
   - **性能优化**：合理设置网络请求的超时时间，避免长时间等待无响应的请求。同时，对获取到的数据进行适当的缓存，减少不必要的网络请求。

2. **电量优化**：为了减少应用对设备电量的消耗，优化同步任务的执行策略。避免在不必要的情况下频繁同步数据，将多个同步任务合并执行，减少网络连接次数。同时，使用 JobScheduler 机制在设备处于充电、闲置等低功耗状态时执行同步任务，降低对用户正常使用设备的影响。例如，设置同步任务在设备充电且连接 Wi-Fi 时执行，既保证数据及时同步，又减少电量消耗。

3. **启动速度优化**：采用懒加载策略，在应用启动时，只加载必要的组件和数据，其他功能模块在需要时再进行加载。对初始化任务进行异步处理，避免阻塞主线程。例如，在应用启动时，先显示账户列表的骨架屏，同时在后台异步加载账户数据，待数据加载完成后再更新界面，提高应用的启动速度和用户体验。

### （四）单元测试

1. **配置测试环境**
   
   添加依赖**：在模块的 `build.gradle` 文件中添加测试依赖。JUnit 4。 Android 特定的测试，还需添加 `androidx.test` 等相关依赖

2. **编写测试用例**
   
   **Instrumented 单元测试**：在 `src/androidTest/java` 目录下编写依赖 Android 框架的测试用例。例如，测试一个 AndroidViewModel：
   
   ```java
   import androidx.arch.core.executor.testing.InstantTaskExecutorRule;
   import androidx.lifecycle.MutableLiveData;
   import androidx.lifecycle.Observer;
   import androidx.test.ext.junit.runners.AndroidJUnit4;
   import org.junit.Before;
   import org.junit.Rule;
   import org.junit.Test;
   import org.junit.runner.RunWith;
   import static org.junit.Assert.*;
   import static org.mockito.Mockito.mock;
   import static org.mockito.Mockito.when;
   
   @RunWith(AndroidJUnit4.class)
   public class MyViewModelTest {
   
       @Rule
       public InstantTaskExecutorRule instantExecutorRule = new InstantTaskExecutorRule();
   
       private MyViewModel viewModel;
   
       @Before
       public void setUp() {
           viewModel = new MyViewModel();
       }
   
       @Test
       public void testLiveDataValue() {
           MutableLiveData<String> liveData = viewModel.getLiveData();
           Observer<String> observer = mock(Observer.class);
           liveData.observeForever(observer);
           viewModel.setLiveDataValue("Test Value");
           verify(observer).onChanged("Test Value");
       }
   }
   ```

3. **运行测试**
   
   在 Android Studio 中，可以通过右键点击测试类或测试方法，选择 `Run 'ClassNameTest'` 来运行测试。也可以通过 Gradle 命令运行，在项目根目录下执行 `./gradlew test`。

4. **连接 Sonar 查看覆盖率**
   
   Jacoco 是一个常用的 Java 代码覆盖率工具。在模块的 `build.gradle` 文件中添加Jacoco 插件和配置。
