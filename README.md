# Stellar API FORKED
## 이 API는 FileTreeSize 프로젝트 의존성에 포함될 목적으로 수정된 코드 전체를 포함하고 있습니다.
원본 프로젝트는 [Stellar-API](https://github.com/roro2239/Stellar-API) 를 참조하십시오.

# How to use
Stellar-API-FullPackage 는 원본 프로젝트에 포함되어있지 않은 userservice 를 함께 빌드할 수 있도록 수정된 소스코드들을 포함하고 있습니다.
Release 탭에 있는 빌드 결과물들은 각각 aidl, api, provider, shared, userservice 로 나뉘어져있으며 필요한 경우 개별로 사용하고,
전체가 필요한 경우 모든 빌드들을 implementation 하십시오.

프로젝트 app 폴더 내에 libs 폴더를 생성 후 stellar 라이브러리를 이동하십시오.
그리고 아래 코드를 app 단 build.gradle.kts 에 붙여넣으십시오.
```kotlin
implementation(files("libs/stellar-aidl.aar"))
implementation(files("libs/stellar-userservice.aar"))
implementation(files("libs/stellar-api.aar"))
implementation(files("libs/stellar-provider.aar"))
implementation(files("libs/stellar-shared.aar"))
```

# Stellar API 통합 가이드
Stellar는 Shizuku의 포크 프로젝트로, ADB 또는 Root 권한으로 특권 작업을 실행할 수 있는 특권 API 프레임워크입니다. 이 문서는 Stellar API를 애플리케이션에 통합하는 방법과 Shizuku에서 Stellar로 마이그레이션하는 방법을 안내합니다.

---

## 목차

1. [빠른 시작](#빠른-시작)
2. [통합 단계](#통합-단계)
3. [API 참조](#api-참조)
4. [코드 예제](#코드-예제)
5. [Shizuku에서 마이그레이션](#shizuku에서-마이그레이션)
6. [자주 묻는 질문](#자주-묻는-질문)

---

## 빠른 시작

### 사전 조건

- Android 최소 버전: API 26 (Android 8.0)
- Stellar 관리자 설치 완료
- Stellar 서비스 시작 완료 (ADB 또는 Root 사용)

---

## 통합 단계

### 1. 의존성 추가

`settings.gradle`에 JitPack 저장소를 추가하세요:

```gradle
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

在 `build.gradle` 中添加依赖：

```gradle
dependencies {
    implementation 'com.github.roro2239:Stellar-API:Tag'
}
```

> `<버전번호>`를 [![JitPack](https://jitpack.io/v/roro2239/Stellar-API.svg)](https://jitpack.io/#roro2239/Stellar-API)에 표시된 최신 버전 번호로 교체하세요

### 2. AndroidManifest 구성

`AndroidManifest.xml`에 `StellarProvider`를 추가하세요:

```xml
<application>
    <!-- 기타 컴포넌트 -->

    <provider
        android:name="roro.stellar.StellarProvider"
        android:authorities="${applicationId}.stellar"
        android:exported="true"
        android:multiprocess="false"
        android:permission="android.permission.INTERACT_ACROSS_USERS_FULL" />

    <meta-data
        android:name="roro.stellar.permissions"
        android:value="stellar" />
</application>
```

**구성 설명:**
- `android:exported="true"` - Stellar 서비스가 Provider에 접근할 수 있도록 반드시 true로 설정해야 합니다
- `android:multiprocess="false"` - Stellar 서비스가 앱 시작 시에만 UID를 가져오므로 반드시 false로 설정해야 합니다
- `android:permission="android.permission.INTERACT_ACROSS_USERS_FULL"` - Shell과 앱 자체만 접근할 수 있도록 제한합니다
- `android:authorities` - 반드시 `${applicationId}.stellar` 형식을 사용해야 합니다

**선택 권한:**

Stellar 서비스 시작 시 앱을 자동으로 시작하려면 다음 권한을 추가할 수 있습니다:

```xml
<meta-data
    android:name="roro.stellar.permissions"
    android:value="stellar,follow_stellar_startup" />
```

권한 설명:
- `stellar` - 기본 Stellar API 접근 권한 (필수)
- `follow_stellar_startup` - Stellar 서비스 시작 시 함께 시작

### 3. Stellar 초기화

Activity 또는 Application에 Stellar 리스너를 추가하세요:

```kotlin
import roro.stellar.Stellar

class MainActivity : ComponentActivity() {

    private val binderReceivedListener = Stellar.OnBinderReceivedListener {
        Log.i("MyApp", "Stellar 서비스 연결됨")
        // 서비스 연결 성공, API 사용 가능
        checkServiceStatus()
    }

    private val binderDeadListener = Stellar.OnBinderDeadListener {
        Log.w("MyApp", "Stellar 서비스 연결 해제됨")
        // 서비스 연결 해제, UI 상태 업데이트
    }

    private val permissionResultListener =
        Stellar.OnRequestPermissionResultListener { requestCode, allowed, onetime ->
            if (allowed) {
                Log.i("MyApp", "권한 부여됨")
                // 권한 부여됨, 특권 작업 실행 가능
                checkServiceStatus()
            } else {
                Log.w("MyApp", "권한 거부됨")
                // 권한 거부됨, 사용자에게 알림
            }
        }

    private val serviceStartedListener = Stellar.OnServiceStartedListener {
        Log.i("MyApp", "Stellar 서비스 시작됨")
        // 서비스 시작 시 작업 실행 (예: 시작 추적 명령 실행)
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 리스너 추가
        // Sticky 버전 사용: 서비스가 이미 연결되어 있으면 즉시 콜백 발생, 아니면 연결될 때까지 대기
        Stellar.addBinderReceivedListenerSticky(binderReceivedListener)
        Stellar.addBinderDeadListener(binderDeadListener)
        Stellar.addRequestPermissionResultListener(permissionResultListener)
        Stellar.addServiceStartedListener(serviceStartedListener)
    }

    override fun onDestroy() {
        super.onDestroy()
        // 리스너 제거
        Stellar.removeBinderReceivedListener(binderReceivedListener)
        Stellar.removeBinderDeadListener(binderDeadListener)
        Stellar.removeRequestPermissionResultListener(permissionResultListener)
        Stellar.removeServiceStartedListener(serviceStartedListener)
    }

    private fun checkServiceStatus() {
        // 서비스 실행 여부 확인
        if (!Stellar.pingBinder()) {
            Log.e("MyApp", "서비스 실행 중이 아님")
            return
        }

        // 권한 확인
        if (!Stellar.checkSelfPermission()) {
            // 권한 요청
            Stellar.requestPermission(requestCode = 1)
            return
        }

        // 서비스 연결 및 권한 부여 완료, API 사용 가능
        Log.i("MyApp", "서비스 버전: ${Stellar.version}")
        Log.i("MyApp", "서비스 UID: ${Stellar.uid}")
    }
}
```

---

## API 참조

### 핵심 클래스: `Stellar`

#### 서비스 상태 확인

```kotlin
// 서비스 실행 여부 확인
Stellar.pingBinder(): Boolean

// 서비스 UID 가져오기 (0 = root, 2000 = adb)
Stellar.uid: Int

// 서비스 API 버전 가져오기
Stellar.version: Int

// 지원되는 최신 버전 가져오기
Stellar.latestServiceVersion: Int

// SELinux 컨텍스트 가져오기
Stellar.sELinuxContext: String?

// 관리자 버전 가져오기
Stellar.versionName: String?
Stellar.versionCode: Int
```

#### 권한 관리

```kotlin
// 권한 부여 여부 확인
Stellar.checkSelfPermission(permission: String = "stellar"): Boolean

// 권한 요청
Stellar.requestPermission(permission: String = "stellar", requestCode: Int)

// 권한 설명 표시 여부 확인
Stellar.shouldShowRequestPermissionRationale(): Boolean

// Stellar 서비스 자체가 지정된 권한을 보유하는지 확인
Stellar.checkRemotePermission(permission: String): Int

// 지원되는 권한 목록 가져오기
Stellar.supportedPermissions: Array<String>
```

#### 프로세스 실행

```kotlin
// 특권 프로세스 생성 (Stellar 서비스 신원으로 실행)
Stellar.newProcess(
    cmd: Array<String?>,      // 명령 및 인수
    env: Array<String?>?,     // 환경 변수 (선택)
    dir: String?              // 작업 디렉터리 (선택)
): StellarRemoteProcess
```

#### 시스템 서비스 접근

```kotlin
// 시스템 서비스 Binder 가져오기 (시스템 프로세스 권한으로)
Stellar.getSystemService(name: String): IBinder?

// 예제: DisplayManager 가져오기
val displayBinder = Stellar.getSystemService("display")
val displayManager = IDisplayManager.Stub.asInterface(displayBinder)
```

#### 고급 기능

```kotlin
// 다른 앱에 런타임 권한 부여/취소
Stellar.grantRuntimePermission(
    packageName: String,
    permissionName: String,
    userId: Int
)

Stellar.revokeRuntimePermission(
    packageName: String,
    permissionName: String,
    userId: Int
)

// Binder 트랜잭션 래퍼
Stellar.transactRemote(data: Parcel, reply: Parcel?, flags: Int)
```

### 보조 클래스: `StellarHelper`

```kotlin
// 관리자 설치 여부 확인
StellarHelper.isManagerInstalled(context: Context): Boolean

// 관리자 열기
StellarHelper.openManager(context: Context): Boolean

// 서비스 정보 가져오기
val serviceInfo = StellarHelper.serviceInfo
serviceInfo?.let {
    val uid = it.uid
    val version = it.version
    val seContext = it.seLinuxContext
    val isRoot = it.isRoot      // uid == 0
    val isAdb = it.isAdb        // uid == 2000
}
```

### 시스템 속성: `StellarSystemProperties`

```kotlin
// 시스템 속성 읽기
StellarSystemProperties.get(key: String): String
StellarSystemProperties.get(key: String, def: String): String
StellarSystemProperties.getInt(key: String, def: Int): Int
StellarSystemProperties.getLong(key: String, def: Long): Long
StellarSystemProperties.getBoolean(key: String, def: Boolean): Boolean

// 시스템 속성 쓰기 (해당 권한 필요)
StellarSystemProperties.set(key: String, value: String)
```

**쓰기 권한 설명:**
- **ADB 모드 (uid=2000):** `debug.*`, `persist.debug.*`, `log.*`, `vendor.debug.*` 쓰기 가능
- **Root 모드 (uid=0):** 대부분 속성 쓰기 가능 (`ro.*` 읽기 전용 속성 제외)

### 특권 프로세스: `StellarRemoteProcess`

```kotlin
val process = Stellar.newProcess(arrayOf("ls", "-la", "/sdcard"), null, null)

// 표준 프로세스 메서드
process.getInputStream(): InputStream
process.getOutputStream(): OutputStream
process.getErrorStream(): InputStream
process.waitFor(): Int
process.exitValue(): Int
process.destroy()

// 추가 메서드
process.alive(): Boolean
process.waitForTimeout(timeout: Long, unit: TimeUnit): Boolean
```

### Binder 래퍼: `StellarBinderWrapper`

시스템 서비스 Binder를 래핑하여 특권 접근을 얻습니다:

```kotlin
val binder = StellarBinderWrapper.getSystemService("package")
val pm = IPackageManager.Stub.asInterface(StellarBinderWrapper(binder))
// 이제 특권 PackageManager API를 사용할 수 있습니다
```

### 사용자 서비스: `StellarUserService`

사용자 서비스를 사용하면 Stellar 서비스 프로세스 내에서 커스텀 Binder 서비스를 실행하여 특권 신원으로 작업을 수행할 수 있습니다.

#### 핵심 메서드

```kotlin
// 사용자 서비스 바인딩
StellarUserService.bindUserService(
    args: UserServiceArgs,           // 서비스 매개변수 설정
    callback: ServiceCallback,       // 서비스 콜백
    handler: Handler? = mainHandler  // 콜백 실행 Handler (선택)
)

// 사용자 서비스 언바인딩
StellarUserService.unbindUserService(args: UserServiceArgs)

// 바인딩된 서비스 Binder 가져오기 (존재하고 활성 상태인 경우)
StellarUserService.peekUserService(args: UserServiceArgs): IBinder?

// 현재 활성 사용자 서비스 수 가져오기
StellarUserService.getUserServiceCount(): Int
```

#### 서비스 콜백 인터페이스

```kotlin
interface ServiceCallback {
    // 서비스 연결 성공
    fun onServiceConnected(service: IBinder)

    // 서비스 연결 해제
    fun onServiceDisconnected()

    // 서비스 시작 실패 (선택 구현)
    fun onServiceStartFailed(errorCode: Int, message: String) {}
}
```

### 사용자 서비스 매개변수: `UserServiceArgs`

Builder 패턴으로 사용자 서비스 매개변수를 구성합니다:

```kotlin
val args = UserServiceArgs.Builder(MyUserService::class.java)
    .processNameSuffix("myservice")  // 프로세스명 접미사, 기본값 "userservice"
    .debug(BuildConfig.DEBUG)        // 디버그 모드 활성화 여부
    .versionCode(BuildConfig.VERSION_CODE.toLong())  // 버전 번호
    .tag("my-tag")                   // 선택 태그
    .serviceMode(ServiceMode.DAEMON) // 서비스 모드
    .build()
```

#### 매개변수 설명

| 매개변수 | 유형 | 기본값 | 설명 |
|------|------|--------|------|
| `className` | String | 필수 | 서비스 클래스의 완전한 클래스명 |
| `processNameSuffix` | String | `"userservice"` | 프로세스명 접미사 |
| `debug` | Boolean | `false` | 디버그 모드 활성화 여부 |
| `use32Bit` | Boolean | `false` | 32비트 프로세스 사용 여부 |
| `versionCode` | Long | `0` | 서비스 버전 번호 |
| `tag` | String? | `null` | 선택 태그 |
| `serviceMode` | ServiceMode | `ONE_TIME` | 서비스 실행 모드 |
| `useStandaloneDex` | Boolean | `false` | 독립 DEX 사용 여부 |

#### 독립 DEX 모드 구성 (임시 삭제됨)

> **참고:** 독립 DEX 모드 기능은 임시 삭제되었으며, 아래 문서는 참고용입니다.

독립 DEX 모드는 사용자 서비스 클래스를 독립 DEX 파일로 컴파일하여 전체 APK 로드를 피할 수 있습니다.

**단계 1: 빌드 스크립트 참조**~~

앱의 `build.gradle`에서 Stellar가 제공하는 빌드 스크립트를 참조하세요:~~

```gradle
plugins {
    id('com.android.application')
    // ...
}

// apply from: project(':userservice').file('userservice-standalone.gradle')
```

**단계 2: stellarUserService 구성**~~

```gradle
// stellarUserService {
//     enabled = true                                    // 启用独立 DEX 模式
//     serviceClass = 'com.example.MyUserService'        // 必须指定服务类的完整类名
//     extraClasses = ['com.example.MyHelper']           // 可选：额外需要包含的类
// }
```

| 구성 항목 | 유형 | 기본값 | 설명 |
|--------|------|--------|------|
| `enabled` | Boolean | `false` | 독립 DEX 모드 활성화 여부 |
| `serviceClass` | String | `null` | 서비스 클래스의 완전한 클래스명 (활성화 시 필수) |
| `extraClasses` | List | `[]` | 추가로 포함할 클래스 목록 |

**단계 3: 코드에서 BuildConfig 사용**~~

활성화 후 빌드 스크립트가 자동으로 `BuildConfig.STELLAR_USE_STANDALONE_DEX` 필드를 생성합니다:~~

```kotlin
// 已临时删除
// val args = UserServiceArgs.Builder(MyUserService::class.java)
//     .useStandaloneDex(BuildConfig.STELLAR_USE_STANDALONE_DEX)
//     .build()
```

**독립 DEX 모드를 사용하지 않는 경우**~~

독립 DEX 모드가 필요하지 않으면 `enabled = false`로 설정하거나 구성을 생략할 수 있습니다:~~

```gradle
// 已临时删除
// stellarUserService {
//     enabled = false  // 此时 BuildConfig.STELLAR_USE_STANDALONE_DEX = false
// }
```

**명명 규칙:** 해당 AIDL 인터페이스는 `I<ServiceName>`으로 명명해야 합니다 (서비스 클래스가 `MyUserService`인 경우 인터페이스는 `IMyUserService`).

### 서비스 모드: `ServiceMode`

```kotlin
enum class ServiceMode {
    ONE_TIME,  // 일회성 서비스: 클라이언트 연결 해제 시 서비스 자동 중지
    DAEMON     // 데몬 프로세스: 명시적으로 중지할 때까지 서비스 지속 실행
}
```

### 사용자 서비스 보조 클래스: `UserServiceHelper`

사용자 서비스 Binder와 상호작용하는 보조 메서드를 제공합니다:

```kotlin
// 사용자 서비스 소멸
UserServiceHelper.destroy(binder: IBinder)

// 서비스 활성 상태 확인
UserServiceHelper.isAlive(binder: IBinder): Boolean

// 서비스 프로세스 UID 가져오기
UserServiceHelper.getUid(binder: IBinder): Int

// 서비스 프로세스 PID 가져오기
UserServiceHelper.getPid(binder: IBinder): Int
```

---

## 코드 예제

### 예제 1: 서비스 상태 확인

```kotlin
fun checkServiceStatus() {
    if (!Stellar.pingBinder()) {
        println("서비스 실행 중이 아님")
        return
    }

    if (!Stellar.checkSelfPermission()) {
        println("권한이 부여되지 않음")
        return
    }

    val version = Stellar.version
    println("서비스 버전: $version")

    val uid = Stellar.uid
    val mode = when (uid) {
        0 -> "Root"
        2000 -> "ADB"
        else -> "기타 (UID=$uid)"
    }
    println("실행 모드: $mode")

    val seContext = Stellar.sELinuxContext
    println("SELinux 컨텍스트: $seContext")
}
```

### 예제 2: Shell 명령 실행

```kotlin
fun executeCommand() {
    thread {
        try {
            println("$ ls -la /sdcard")

            val process = Stellar.newProcess(
                arrayOf("ls", "-la", "/sdcard"),
                null,
                null
            )

            val reader = BufferedReader(InputStreamReader(process.inputStream))
            reader.lineSequence().forEach { line ->
                println(line)
            }

            val exitCode = process.waitFor()
            println("종료 코드: $exitCode")

            process.destroy()
        } catch (e: Exception) {
            println("오류: ${e.message}")
        }
    }
}
```

### 예제 3: 시스템 속성 읽기

```kotlin
fun readSystemProperties() {
    try {
        val brand = StellarSystemProperties.get("ro.product.brand")
        println("브랜드: $brand")

        val model = StellarSystemProperties.get("ro.product.model")
        println("모델: $model")

        val androidVersion = StellarSystemProperties.get("ro.build.version.release")
        println("Android 버전: $androidVersion")

        val sdkInt = StellarSystemProperties.getInt("ro.build.version.sdk", 0)
        println("SDK 버전: $sdkInt")
    } catch (e: Exception) {
        println("오류: ${e.message}")
    }
}
```

### 예제 4: Stellar 서비스 권한 확인

```kotlin
fun checkStellarServicePermissions() {
    val permissions = arrayOf(
        "android.permission.WRITE_SECURE_SETTINGS",
        "android.permission.READ_LOGS",
        "android.permission.DUMP",
        "android.permission.PACKAGE_USAGE_STATS"
    )

    permissions.forEach { permission ->
        val result = Stellar.checkRemotePermission(permission)
        val status = if (result == PackageManager.PERMISSION_GRANTED) "부여됨" else "부여되지 않음"
        val name = permission.substringAfterLast(".")
        println("$name: $status")
    }
}
```

### 예제 5: 다른 앱에 권한 부여

```kotlin
fun grantPermissionToApp(packageName: String) {
    try {
        Stellar.grantRuntimePermission(
            packageName = packageName,
            permissionName = "android.permission.WRITE_EXTERNAL_STORAGE",
            userId = 0
        )
        println("권한이 부여됨")
    } catch (e: Exception) {
        println("권한 부여 실패: ${e.message}")
    }
}
```

### 예제 6: Stellar 시작 추적

`AndroidManifest.xml`에 `follow_stellar_startup` 권한을 선언한 경우 BroadcastReceiver를 생성하여 시작 알림을 수신할 수 있습니다:

```kotlin
class FollowStellarStartup : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        Log.i("MyApp", "Stellar 시작됨: ${intent?.action}")

        // Stellar 시작 시 작업 실행
        try {
            Stellar.newProcess(
                arrayOf("touch", "/sdcard/stellar_started.log"),
                null,
                null
            )
        } catch (e: Exception) {
            Log.e("MyApp", "명령 실행 실패", e)
        }
    }
}
```

`AndroidManifest.xml`에 등록:

```xml
<receiver
    android:name=".FollowStellarStartup"
    android:exported="false">
    <intent-filter>
        <action android:name="roro.stellar.action.STELLAR_STARTED" />
    </intent-filter>
</receiver>
```

### 예제 7: 사용자 서비스 생성

먼저 AIDL 인터페이스를 정의합니다:

```aidl
// src/main/aidl/com/example/IMyUserService.aidl
package com.example;

interface IMyUserService {
    String executeCommand(String command) = 1;
    String getSystemProperty(String name) = 2;
}
```

그런 다음 서비스 클래스를 구현합니다:

```kotlin
// src/main/java/com/example/MyUserService.kt
package com.example

class MyUserService : IMyUserService.Stub() {

    override fun executeCommand(command: String): String {
        return try {
            val process = Runtime.getRuntime().exec(arrayOf("sh", "-c", command))
            val reader = java.io.BufferedReader(
                java.io.InputStreamReader(process.inputStream)
            )
            val output = StringBuilder()
            var line: String?
            while (reader.readLine().also { line = it } != null) {
                output.append(line).append("\n")
            }
            process.waitFor()
            output.toString().trim()
        } catch (e: Exception) {
            "Error: ${e.message}"
        }
    }

    override fun getSystemProperty(name: String): String {
        return try {
            val process = Runtime.getRuntime().exec(arrayOf("getprop", name))
            val reader = java.io.BufferedReader(
                java.io.InputStreamReader(process.inputStream)
            )
            reader.readLine() ?: ""
        } catch (e: Exception) {
            ""
        }
    }
}
```

---

### 예제 8: 사용자 서비스 바인딩 및 사용

```kotlin
import roro.stellar.userservice.StellarUserService
import roro.stellar.userservice.UserServiceArgs
import roro.stellar.userservice.ServiceMode

class MainActivity : ComponentActivity() {

    private var userService: IMyUserService? = null

    private val serviceCallback = object : StellarUserService.ServiceCallback {
        override fun onServiceConnected(service: IBinder) {
            Log.i("MyApp", "사용자 서비스 연결됨")
            userService = IMyUserService.Stub.asInterface(service)
            // 이제 서비스를 사용할 수 있습니다
            executeCommandViaService()
        }

        override fun onServiceDisconnected() {
            Log.w("MyApp", "사용자 서비스 연결 해제됨")
            userService = null
        }

        override fun onServiceStartFailed(errorCode: Int, message: String) {
            Log.e("MyApp", "사용자 서비스 시작 실패: $errorCode - $message")
        }
    }

    private fun bindUserService() {
        val args = UserServiceArgs.Builder(MyUserService::class.java)
            .processNameSuffix("myservice")
            .debug(BuildConfig.DEBUG)
            .versionCode(BuildConfig.VERSION_CODE.toLong())
            .serviceMode(ServiceMode.ONE_TIME)
            .build()

        StellarUserService.bindUserService(args, serviceCallback)
    }

    private fun executeCommandViaService() {
        thread {
            try {
                val result = userService?.executeCommand("ls -la /sdcard")
                Log.i("MyApp", "명령 결과: $result")
            } catch (e: Exception) {
                Log.e("MyApp", "명령 실행 실패", e)
            }
        }
    }
}
```

---

### 예제 9: 데몬 프로세스 모드의 사용자 서비스

```kotlin
// DAEMON 모드로 지속 실행되는 서비스 생성
fun bindDaemonService() {
    val args = UserServiceArgs.Builder(MyUserService::class.java)
        .processNameSuffix("daemon")
        .serviceMode(ServiceMode.DAEMON)  // 데몬 프로세스 모드
        .versionCode(BuildConfig.VERSION_CODE.toLong())
        .build()

    StellarUserService.bindUserService(args, object : StellarUserService.ServiceCallback {
        override fun onServiceConnected(service: IBinder) {
            Log.i("MyApp", "데몬 서비스 시작됨")
            // 명시적으로 unbindUserService를 호출할 때까지 서비스가 지속 실행됩니다
        }

        override fun onServiceDisconnected() {
            Log.i("MyApp", "데몬 서비스 중지됨")
        }
    })
}

// 데몬 서비스 중지
fun stopDaemonService() {
    val args = UserServiceArgs.Builder(MyUserService::class.java)
        .processNameSuffix("daemon")
        .serviceMode(ServiceMode.DAEMON)
        .build()

    StellarUserService.unbindUserService(args)
}
```

---

## Shizuku에서 마이그레이션

Stellar는 Shizuku의 포크 프로젝트이므로 API 디자인이 매우 유사하며 마이그레이션 과정이 비교적 간단합니다. 아래는 자세한 마이그레이션 단계입니다.

### Stellar vs Shizuku 비교

| 기능 | Stellar | Shizuku |
|------|---------|---------|
| **패키지명** | `roro.stellar.manager` | `moe.shizuku.privileged.api` |
| **API 네임스페이스** | `roro.stellar.*` | `rikka.shizuku.*` |
| **권한 시스템** | 다중 권한: `stellar`, `follow_stellar_startup` | 단일 권한 모델 |
| **시작 훅** | 서비스 시작 추적 내장 지원 | 내장 지원 없음 |
| **Provider Authority** | `${applicationId}.stellar` | `${applicationId}.shizuku` |

### 마이그레이션 단계

#### 단계 1: 의존성 업데이트

```gradle
// Shizuku 의존성 제거
// implementation 'dev.rikka.shizuku:api:13.1.5'
// implementation 'dev.rikka.shizuku:provider:13.1.5'

// Stellar 의존성 추가
implementation 'com.github.roro2239:Stellar-API:Tag'
```

동시에 `settings.gradle`에 JitPack 저장소를 추가하세요:

```gradle
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

#### 단계 2: AndroidManifest 업데이트

```xml
<!-- Shizuku Provider 제거 -->
<!--
<provider
    android:name="rikka.shizuku.ShizukuProvider"
    android:authorities="${applicationId}.shizuku"
    android:exported="true"
    android:multiprocess="false"
    android:permission="android.permission.INTERACT_ACROSS_USERS_FULL" />
-->

<!-- Stellar Provider 추가 -->
<provider
    android:name="roro.stellar.StellarProvider"
    android:authorities="${applicationId}.stellar"
    android:exported="true"
    android:multiprocess="false"
    android:permission="android.permission.INTERACT_ACROSS_USERS_FULL" />

<meta-data
    android:name="roro.stellar.permissions"
    android:value="stellar" />
```

#### 단계 3: 임포트 구문 업데이트

```kotlin
// 교체
// import rikka.shizuku.Shizuku
// import rikka.shizuku.ShizukuProvider

// 로
import roro.stellar.Stellar
import roro.stellar.StellarProvider
import roro.stellar.StellarHelper
```

#### 단계 4: API 호출 업데이트

코드의 모든 Shizuku API 호출을 해당 Stellar API로 교체하세요. 대부분의 API는 클래스명을 `Shizuku`에서 `Stellar`로 변경하기만 하면 되며, 일부 API는 사소한 차이가 있습니다 (예: `getUid()`가 속성 `uid`로 변경).

| 기능 설명 | Stellar API | Shizuku API |
|----------|-------------|-------------|
| 서비스 실행 여부 확인 | `Stellar.pingBinder()` | `Shizuku.pingBinder()` |
| 서비스 UID 가져오기 (0=root, 2000=adb) | `Stellar.uid` | `Shizuku.getUid()` |
| 서비스 API 버전 가져오기 | `Stellar.version` | `Shizuku.getVersion()` |
| 앱 권한 부여 여부 확인 | `Stellar.checkSelfPermission()` | `Shizuku.checkSelfPermission()` |
| 사용자에게 권한 요청 | `Stellar.requestPermission(requestCode = code)` | `Shizuku.requestPermission(code)` |
| 서비스 연결 리스너 추가 | `Stellar.addBinderReceivedListener()` | `Shizuku.addBinderReceivedListener()` |
| 서비스 연결 리스너 추가 (즉시 트리거) | `Stellar.addBinderReceivedListenerSticky()` | `Shizuku.addBinderReceivedListenerSticky()` |
| 서비스 연결 해제 리스너 추가 | `Stellar.addBinderDeadListener()` | `Shizuku.addBinderDeadListener()` |
| 권한 요청 결과 리스너 추가 | `Stellar.addRequestPermissionResultListener()` | `Shizuku.addRequestPermissionResultListener()` |
| 서비스 시작 리스너 추가 | `Stellar.addServiceStartedListener()` | 해당 API 없음 |
| 서비스 연결 리스너 제거 | `Stellar.removeBinderReceivedListener()` | `Shizuku.removeBinderReceivedListener()` |
| 서비스 연결 해제 리스너 제거 | `Stellar.removeBinderDeadListener()` | `Shizuku.removeBinderDeadListener()` |
| 권한 요청 결과 리스너 제거 | `Stellar.removeRequestPermissionResultListener()` | `Shizuku.removeRequestPermissionResultListener()` |
| 서비스 시작 리스너 제거 | `Stellar.removeServiceStartedListener()` | 해당 API 없음 |
| 특권 프로세스 생성하여 명령 실행 | `Stellar.newProcess()` | `Shizuku.newProcess()` (최신 API에서 폐기됨) |
| 사용자 서비스 바인딩 | `StellarUserService.bindUserService()` | `Shizuku.bindUserService()` |
| 사용자 서비스 언바인딩 | `StellarUserService.unbindUserService()` | `Shizuku.unbindUserService()` |
| 사용자 서비스 가져오기 | `StellarUserService.peekUserService()` | `Shizuku.peekUserService()` |

#### 단계 5: 리스너 인터페이스 업데이트

```kotlin
// 교체
// Shizuku.OnBinderReceivedListener { }
// Shizuku.OnBinderDeadListener { }
// Shizuku.OnRequestPermissionResultListener { requestCode, grantResult -> }

// 로
Stellar.OnBinderReceivedListener { }
Stellar.OnBinderDeadListener { }
Stellar.OnRequestPermissionResultListener { requestCode, allowed, onetime ->
    // 주의: 매개변수가 grantResult에서 allowed 및 onetime으로 변경됨
}
```

#### 단계 6: 보조 메서드 업데이트

```kotlin
// 교체
// ShizukuProvider.isShizukuInstalled(context)
// ShizukuProvider.openShizuku(context)

// 로
StellarHelper.isManagerInstalled(context)
StellarHelper.openManager(context)
```

#### 단계 7: 마이그레이션 테스트

1. Stellar 관리자 설치
2. Stellar 서비스 시작 (ADB 또는 Root)
3. 권한 요청 흐름 테스트
4. 특권 프로세스 실행 테스트
5. 서비스 재시작 후 재연결 테스트

### 마이그레이션 예제

**마이그레이션 전 (Shizuku):**

```kotlin
import rikka.shizuku.Shizuku
import rikka.shizuku.ShizukuProvider

class MainActivity : AppCompatActivity() {

    private val binderReceivedListener = Shizuku.OnBinderReceivedListener {
        checkStatus()
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        Shizuku.addBinderReceivedListenerSticky(binderReceivedListener)
    }

    private fun checkStatus() {
        if (!Shizuku.pingBinder()) return

        if (Shizuku.checkSelfPermission() != PackageManager.PERMISSION_GRANTED) {
            Shizuku.requestPermission(1)
            return
        }

        val uid = Shizuku.getUid()
        println("UID: $uid")
    }
}
```

**마이그레이션 후 (Stellar):**

```kotlin
import roro.stellar.Stellar
import roro.stellar.StellarProvider

class MainActivity : AppCompatActivity() {

    private val binderReceivedListener = Stellar.OnBinderReceivedListener {
        checkStatus()
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        Stellar.addBinderReceivedListenerSticky(binderReceivedListener)
    }

    private fun checkStatus() {
        if (!Stellar.pingBinder()) return

        if (!Stellar.checkSelfPermission()) {
            Stellar.requestPermission(requestCode = 1)
            return
        }

        val uid = Stellar.uid
        println("UID: $uid")
    }
}
```

---

## 자주 묻는 질문

### Q1: 서비스가 Root 모드인지 ADB 모드인지 어떻게 판단하나요?

```kotlin
val uid = Stellar.uid
when (uid) {
    0 -> println("Root 모드")
    2000 -> println("ADB 모드")
    else -> println("알 수 없는 모드: $uid")
}

// 또는 StellarHelper 사용
val serviceInfo = StellarHelper.serviceInfo
if (serviceInfo?.isRoot == true) {
    println("Root 모드")
} else if (serviceInfo?.isAdb == true) {
    println("ADB 모드")
}
```

### Q2: 서비스 연결 해제를 어떻게 처리하나요?

```kotlin
private val binderDeadListener = Stellar.OnBinderDeadListener {
    // 서비스 연결 해제, UI 업데이트
    runOnUiThread {
        Toast.makeText(this, "Stellar 서비스 연결이 해제되었습니다", Toast.LENGTH_SHORT).show()
        // Stellar가 필요한 기능 비활성화
        updateUIForDisconnectedState()
    }
}
```

### Q3: 권한이 거부된 경우 어떻게 처리하나요?

```kotlin
private val permissionResultListener =
    Stellar.OnRequestPermissionResultListener { requestCode, allowed, onetime ->
        if (!allowed) {
            // 권한 거부됨
            if (Stellar.shouldShowRequestPermissionRationale()) {
                // 권한 설명 대화상자 표시
                showPermissionRationaleDialog()
            } else {
                // 사용자가 "다시 묻지 않음"을 선택함, 관리자에서 수동으로 권한 부여하도록 안내
                StellarHelper.openManager(this)
            }
        }
    }
```

### Q4: 다중 프로세스 앱에서 Stellar를 어떻게 사용하나요?

앱이 다중 프로세스를 사용하는 경우 Application에서 다중 프로세스 지원을 활성화해야 합니다:

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // 현재 프로세스가 Provider 프로세스인지 확인
        val isProviderProcess = // 판단 로직
        StellarProvider.enableMultiProcessSupport(isProviderProcess)

        // Provider 프로세스가 아닌 경우 Binder 요청
        if (!isProviderProcess) {
            StellarProvider.requestBinderForNonProviderProcess(this)
        }
    }
}
```

### Q5: 명령 실행 시 타임아웃을 어떻게 처리하나요?

```kotlin
fun executeCommandWithTimeout() {
    thread {
        try {
            val process = Stellar.newProcess(
                arrayOf("sleep", "10"),
                null,
                null
            )

            // 최대 5초 대기
            val finished = process.waitForTimeout(5, TimeUnit.SECONDS)

            if (!finished) {
                println("명령 타임아웃, 강제 종료")
                process.destroy()
            } else {
                println("명령 완료, 종료 코드: ${process.exitValue()}")
            }
        } catch (e: Exception) {
            println("오류: ${e.message}")
        }
    }
}
```

### Q6: 관리자가 설치되었는지 어떻게 확인하나요?

```kotlin
if (!StellarHelper.isManagerInstalled(context)) {
    // 관리자 미설치, 사용자에게 설치 안내
    AlertDialog.Builder(context)
        .setTitle("Stellar 관리자 필요")
        .setMessage("이 기능을 사용하려면 Stellar 관리자를 설치해야 합니다")
        .show()
}
```

### Q7: 명령의 오류 출력을 어떻게 읽나요?

```kotlin
fun executeCommandWithErrorHandling() {
    thread {
        try {
            val process = Stellar.newProcess(
                arrayOf("ls", "/nonexistent"),
                null,
                null
            )

            // 표준 출력 읽기
            val outputReader = BufferedReader(InputStreamReader(process.inputStream))
            outputReader.lineSequence().forEach { line ->
                println("출력: $line")
            }

            // 오류 출력 읽기
            val errorReader = BufferedReader(InputStreamReader(process.errorStream))
            errorReader.lineSequence().forEach { line ->
                println("오류: $line")
            }

            val exitCode = process.waitFor()
            println("종료 코드: $exitCode")

            process.destroy()
        } catch (e: Exception) {
            println("예외: ${e.message}")
        }
    }
}
```

### Q8: Stellar와 Shizuku를 동시에 사용할 수 있나요?

가능합니다. Stellar와 Shizuku는 서로 다른 패키지명과 Provider Authority를 사용하므로 동일한 기기에서 동시에 설치 및 실행할 수 있습니다. 다만 혼란을 피하기 위해 앱에서는 하나만 통합하는 것을 권장합니다.

---

## 더 많은 자료

- **예제 앱:** `demo` 모듈에서 완전한 사용 예제를 확인하세요
- **문제 제보:** GitHub Issues에 문제를 제출하세요

---

## 라이선스

본 프로젝트의 수정 부분은 [Mozilla Public License 2.0](LICENSE)이 적용됩니다.

원본 Shizuku 코드는 Apache License 2.0 라이선스를 유지합니다.

| 구성 요소 | 라이선스 |
|------|--------|
| Stellar 수정 부분 | Mozilla Public License 2.0 |
| [Shizuku](https://github.com/RikkaApps/Shizuku) 원본 코드 | Apache License 2.0 |
