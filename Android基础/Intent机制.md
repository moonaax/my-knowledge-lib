# Intent 机制

## 一、Intent 是什么？

Intent 是 Android 中组件之间通信的"信使"。你要启动一个 Activity、启动一个 Service、发送一个广播，都需要通过 Intent 来传递信息。

通俗理解：Intent 就是一封"信"，里面写了你要找谁（目标组件）、要干什么（Action）、带了什么数据（Extras）。

---

## 二、显式 Intent vs 隐式 Intent

### 2.1 显式 Intent

明确指定目标组件的类名，通常用于 App 内部跳转。

```java
// 指定目标 Activity
Intent intent = new Intent(this, DetailActivity.class);
intent.putExtra("id", 123);
startActivity(intent);

// 指定目标 Service
Intent intent = new Intent(this, MyService.class);
startService(intent);
```

### 2.2 隐式 Intent

不指定具体组件，而是声明一个 Action，系统根据 IntentFilter 匹配合适的组件。通常用于跨 App 调用。

```java
// 打开浏览器
Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse("https://www.example.com"));
startActivity(intent);

// 打电话
Intent dialIntent = new Intent(Intent.ACTION_DIAL, Uri.parse("tel:10086"));
startActivity(dialIntent);

// 分享文本
Intent shareIntent = new Intent(Intent.ACTION_SEND);
shareIntent.setType("text/plain");
shareIntent.putExtra(Intent.EXTRA_TEXT, "分享内容");
startActivity(Intent.createChooser(shareIntent, "分享到"));

// 拍照
Intent captureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
startActivityForResult(captureIntent, REQUEST_CODE);
```

### 2.3 对比

| 对比项 | 显式 Intent | 隐式 Intent |
|--------|-----------|-----------|
| 指定目标 | 明确指定类名 | 通过 Action/Category/Data 匹配 |
| 使用场景 | App 内部跳转 | 跨 App 调用、系统功能 |
| 安全性 | 高（明确目标） | 低（可能被其他 App 拦截） |
| 是否需要 IntentFilter | 不需要 | 需要目标组件声明 IntentFilter |

---

## 三、IntentFilter 匹配规则

当你发送一个隐式 Intent，系统会拿它和所有组件声明的 IntentFilter 进行匹配。必须同时满足 Action、Category、Data 三个条件才算匹配成功。

### 3.1 Action 匹配

- IntentFilter 中可以声明多个 Action
- Intent 中的 Action 必须和 IntentFilter 中的**某一个**匹配（完全相等）
- Intent 没有设置 Action → 只要 IntentFilter 有至少一个 Action 就匹配成功

```xml
<intent-filter>
    <action android:name="com.example.ACTION_A" />
    <action android:name="com.example.ACTION_B" />
    <!-- Intent 的 Action 是 A 或 B 都能匹配 -->
</intent-filter>
```

### 3.2 Category 匹配

- Intent 中的**所有** Category 都必须在 IntentFilter 中找到对应的
- `startActivity()` 会自动添加 `CATEGORY_DEFAULT`，所以 IntentFilter 必须包含它

```xml
<intent-filter>
    <action android:name="com.example.ACTION_A" />
    <category android:name="android.intent.category.DEFAULT" />
    <!-- 必须有 DEFAULT，否则隐式 Intent 匹配不到 -->
</intent-filter>
```

### 3.3 Data 匹配

Data 由 URI 和 MIME Type 组成：

```
scheme://host:port/path
```

```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:scheme="https"
          android:host="www.example.com"
          android:pathPrefix="/article" />
</intent-filter>
```

匹配规则：
- 如果 IntentFilter 只有 MIME Type → Intent 也只能有 MIME Type
- 如果 IntentFilter 有 URI + MIME Type → Intent 两者都要匹配
- URI 匹配是从 scheme → host → port → path 逐级匹配

### 3.4 匹配安全

```java
// 隐式 Intent 启动前应该检查是否有匹配的组件
Intent intent = new Intent("com.example.ACTION_A");
if (intent.resolveActivity(getPackageManager()) != null) {
    startActivity(intent);
} else {
    // 没有匹配的组件，提示用户
}
```

> Android 11+ 需要在 Manifest 中声明 `<queries>` 才能查询到其他 App 的组件（包可见性限制）。

```xml
<!-- AndroidManifest.xml -->
<queries>
    <intent>
        <action android:name="android.intent.action.VIEW" />
        <data android:scheme="https" />
    </intent>
</queries>
```

---

## 四、PendingIntent

PendingIntent 是对 Intent 的一层包装，它代表一个"将来要执行的 Intent"。你把它交给别人（系统、其他 App），别人在合适的时机代替你执行。

### 4.1 使用场景

| 场景 | 说明 |
|------|------|
| 通知点击 | 用户点击通知时打开指定页面 |
| AlarmManager | 定时触发某个操作 |
| Widget | 桌面小组件的点击事件 |
| 地理围栏 | 进入/离开某个区域时触发 |

### 4.2 三种类型

```java
// 启动 Activity
PendingIntent pendingIntent = PendingIntent.getActivity(
    context, requestCode, intent,
    PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE
);

// 启动 Service
PendingIntent pendingIntent = PendingIntent.getService(
    context, requestCode, intent,
    PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE
);

// 发送广播
PendingIntent pendingIntent = PendingIntent.getBroadcast(
    context, requestCode, intent,
    PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE
);
```

### 4.3 Flags

| Flag | 含义 |
|------|------|
| `FLAG_UPDATE_CURRENT` | 如果已存在，更新其中的 extras |
| `FLAG_CANCEL_CURRENT` | 如果已存在，取消旧的创建新的 |
| `FLAG_NO_CREATE` | 如果不存在，返回 null（不创建） |
| `FLAG_ONE_SHOT` | 只能使用一次 |
| `FLAG_IMMUTABLE` | PendingIntent 不可被修改（Android 12+ 必须指定 IMMUTABLE 或 MUTABLE） |
| `FLAG_MUTABLE` | PendingIntent 可被修改 |

```java
// Android 12+ 必须明确指定可变性
// 通知点击通常用 IMMUTABLE
int flags = PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE;
```

### 4.4 PendingIntent 的匹配规则

两个 PendingIntent 被认为是"相同的"条件：
- requestCode 相同
- Intent 的 ComponentName 和 Action 相同
- **注意：extras 不参与匹配！**

所以如果你用不同的 extras 创建多个通知的 PendingIntent，必须用不同的 requestCode，否则会互相覆盖。

```java
// ❌ 错误：requestCode 相同，extras 不同但会被覆盖
PendingIntent.getActivity(ctx, 0, intent1, flags);  // intent1 有 extra "id"=1
PendingIntent.getActivity(ctx, 0, intent2, flags);  // intent2 有 extra "id"=2
// 结果：两个通知点击都会打开 id=2

// ✅ 正确：用不同的 requestCode
PendingIntent.getActivity(ctx, 1, intent1, flags);
PendingIntent.getActivity(ctx, 2, intent2, flags);
```

---

## 五、Deep Link 与 App Links

### 5.1 Deep Link

通过 URI 直接打开 App 的某个页面。

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".DetailActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp"
              android:host="detail" />
    </intent-filter>
</activity>
```

```java
// 通过 URI 打开
// myapp://detail?id=123
Uri uri = getIntent().getData();
String id = uri != null ? uri.getQueryParameter("id") : null;
```

**Deep Link 的问题：** 点击链接时系统会弹出选择器（让用户选择用哪个 App 打开），体验不好。

### 5.2 App Links（Android 6.0+）

App Links 是 Deep Link 的升级版，通过域名验证实现**免弹窗直接打开**。

```xml
<activity android:name=".DetailActivity">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https"
              android:host="www.example.com"
              android:pathPrefix="/detail" />
    </intent-filter>
</activity>
```

需要在你的域名下放一个验证文件：
```
https://www.example.com/.well-known/assetlinks.json
```

```json
[{
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
        "namespace": "android_app",
        "package_name": "com.example.app",
        "sha256_cert_fingerprints": ["签名指纹"]
    }
}]
```

### 5.3 Deep Link vs App Links

| 对比项 | Deep Link | App Links |
|--------|-----------|-----------|
| scheme | 自定义（myapp://） | 必须 https |
| 域名验证 | 不需要 | 需要（assetlinks.json） |
| 弹窗选择 | 会弹 | 不弹，直接打开 |
| 最低版本 | 所有版本 | Android 6.0+ |

---

## 六、Intent 传递数据的限制

### 6.1 Bundle 大小限制

Intent 通过 Bundle 传递数据，底层走 Binder。Binder 的事务缓冲区大小约 **1MB**（所有进程共享），单次传输建议不超过 **500KB**。

超过限制会抛 `TransactionTooLargeException`。

```java
// ❌ 不要通过 Intent 传大数据
intent.putExtra("bitmap", largeBitmap);  // 可能崩溃

// ✅ 传 URI 或文件路径，让目标组件自己加载
intent.putExtra("image_uri", imageUri);

// ✅ 或者用 ViewModel / 数据库 / 文件共享
```

### 6.2 Serializable vs Parcelable

| 对比项 | Serializable | Parcelable |
|--------|-------------|------------|
| 实现方式 | Java 接口，自动序列化 | Android 接口，手动实现 |
| 性能 | 慢（用反射） | 快（直接写入内存） |
| 使用场景 | 简单场景、持久化 | Android 组件间传递 |

```java
// Java 中手动实现 Parcelable
public class User implements Parcelable {
    private int id;
    private String name;

    protected User(Parcel in) {
        id = in.readInt();
        name = in.readString();
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(id);
        dest.writeString(name);
    }

    @Override
    public int describeContents() { return 0; }

    public static final Creator<User> CREATOR = new Creator<User>() {
        @Override
        public User createFromParcel(Parcel in) { return new User(in); }
        @Override
        public User[] newArray(int size) { return new User[size]; }
    };
}
```

---

## 七、面试题库

### Q1：显式 Intent 和隐式 Intent 的区别？什么时候用哪个？

> 显式 Intent 直接指定目标组件的类名，用于 App 内部跳转，安全可控。隐式 Intent 通过 Action、Category、Data 描述意图，由系统匹配合适的组件，用于跨 App 调用或调用系统功能。实际开发中，App 内部一律用显式，调用外部能力（分享、拍照、打开浏览器）用隐式。隐式 Intent 启动前一定要做 resolveActivity 检查，避免找不到匹配组件导致 ActivityNotFoundException。

### Q2：IntentFilter 的匹配规则是什么？

> 隐式 Intent 需要同时匹配 Action、Category、Data 三项。Action 匹配：Intent 的 Action 必须和 IntentFilter 中声明的某一个完全相等。Category 匹配：Intent 中所有 Category 都必须在 IntentFilter 中存在，而且 startActivity 会自动加 CATEGORY_DEFAULT，所以 IntentFilter 必须声明它。Data 匹配：按 scheme → host → port → path 逐级匹配，URI 和 MIME Type 都要满足。三项全部通过才算匹配成功，任何一项不满足都会失败。

### Q3：PendingIntent 是什么？和 Intent 有什么区别？

> PendingIntent 是对 Intent 的一层封装，代表一个延迟执行的意图。核心区别在于执行权的转移——你把 PendingIntent 交给系统或其他 App，它们可以在未来某个时刻以你的身份和权限执行这个 Intent。典型场景是通知点击、AlarmManager 定时任务、桌面 Widget。Intent 是立即执行的，PendingIntent 是委托执行的。另外 Android 12 之后创建 PendingIntent 必须明确指定 FLAG_IMMUTABLE 或 FLAG_MUTABLE，否则会崩溃。

### Q4：PendingIntent 的匹配规则？为什么多个通知的点击事件会互相覆盖？

> PendingIntent 的匹配只看 requestCode、ComponentName 和 Action，不看 extras。所以如果你用相同的 requestCode 和相同的目标 Activity 创建多个 PendingIntent，即使 extras 不同，系统也认为是同一个。后创建的会覆盖前面的（FLAG_UPDATE_CURRENT）或者直接复用旧的。解决方案是给每个 PendingIntent 设置不同的 requestCode，比如用通知 ID 或数据库主键作为 requestCode。

### Q5：Deep Link 和 App Links 的区别？

> Deep Link 用自定义 scheme（如 myapp://），不需要域名验证，但点击时会弹出应用选择器让用户选。App Links 是 Android 6.0 引入的增强版，必须用 https scheme，需要在域名下放 assetlinks.json 做验证，验证通过后点击链接直接打开 App 不弹窗。实际项目中我们通常两者都配置——App Links 覆盖 6.0+ 设备实现无缝跳转，Deep Link 作为兜底方案覆盖低版本。

### Q6：Intent 传递数据有大小限制吗？怎么解决？

> 有。Intent 底层走 Binder 传输，Binder 事务缓冲区大约 1MB 且所有进程共享，单次传输建议控制在 500KB 以内，超过会抛 TransactionTooLargeException。实际开发中绝对不要通过 Intent 传 Bitmap 或大列表。解决方案：传 URI 或文件路径让目标组件自己加载；用 ViewModel 在同进程内共享数据；用数据库或文件做中转；用 EventBus 或 LiveData 做进程内通信。

### Q7：Serializable 和 Parcelable 的区别？为什么 Android 推荐 Parcelable？

> Serializable 是 Java 标准接口，实现简单但性能差，因为底层用反射和大量临时对象，会触发频繁 GC。Parcelable 是 Android 专门设计的序列化接口，直接在内存中读写，没有反射开销，速度比 Serializable 快 10 倍以上。Android 组件间传递数据（Intent、Bundle）推荐用 Parcelable。Kotlin 中用 @Parcelize 注解可以自动生成实现代码，开发体验和 Serializable 一样简单。如果是持久化到磁盘的场景，Serializable 或 JSON 序列化更合适。

### Q8：Android 11 的包可见性限制对 Intent 有什么影响？

> Android 11 引入了包可见性限制，App 默认无法查询到设备上其他 App 的信息。这意味着 resolveActivity、queryIntentActivities 这些方法可能返回空结果。解决方案是在 Manifest 中声明 `<queries>` 标签，明确列出需要交互的 Intent 或包名。如果你的 App 需要和很多 App 交互（比如启动器），可以声明 QUERY_ALL_PACKAGES 权限，但 Google Play 审核会比较严格。

### Q9：startActivityForResult 被废弃了，现在怎么获取 Activity 返回结果？

> 现在用 Activity Result API（registerForActivityResult）。旧的 startActivityForResult 有几个问题：requestCode 容易冲突、在 Fragment 中嵌套调用混乱、代码分散在 onActivityResult 中不好维护。新 API 基于 ActivityResultContract，类型安全，代码内聚，而且支持在 Fragment 和 Activity 中统一使用。常用的 Contract 有 StartActivityForResult、RequestPermission、TakePicture、GetContent 等。

```java
// 新方式
ActivityResultLauncher<Intent> launcher = registerForActivityResult(
    new ActivityResultContracts.StartActivityForResult(),
    result -> {
        if (result.getResultCode() == RESULT_OK && result.getData() != null) {
            String data = result.getData().getStringExtra("key");
        }
    }
);

// 启动
launcher.launch(new Intent(this, SecondActivity.class));
```

### Q10：Intent 的 Flag 有哪些常用的？分别什么效果？

> 最常用的几个：FLAG_ACTIVITY_NEW_TASK 在新任务栈中启动，从非 Activity 上下文（Service、BroadcastReceiver）启动 Activity 时必须加这个 Flag。FLAG_ACTIVITY_CLEAR_TOP 如果目标 Activity 已在栈中，清除其上方所有 Activity。FLAG_ACTIVITY_SINGLE_TOP 等同于 singleTop 启动模式，栈顶复用。FLAG_ACTIVITY_CLEAR_TASK 配合 NEW_TASK 使用，清空整个任务栈再启动。实际项目中最常见的组合是 CLEAR_TOP + SINGLE_TOP，用于"回到首页"的场景，效果等同于 singleTask。
