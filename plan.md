# 기능 명세서

## - Android Studio Ladybug 호환성 체크

| 항목 | 권장 값 | 비고 |
| --- | --- | --- |
| Android Studio | Ladybug 2024.2.1  |  |
| AGP (Android Gradle Plugin) | **8.7.x** | Ladybug는 8.7 기본 |
| Gradle | **8.9** | gradle-wrapper.properties |
| JDK | **17** (Ladybug 번들) | JAVA_HOME 자동 설정 |
| Kotlin | **2.0.20** 이상 |  |
| compileSdk / targetSdk | **34** (또는 35) | Play Store는 35 권장 |
| minSdk | **24** (Android 7.0) | 시장 점유율 99%+ |

```kotlin
// build.gradle.kts (app)
android {
    compileSdk = 34
    defaultConfig {
        minSdk = 24
        targetSdk = 34
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions { jvmTarget = "17" }
    buildFeatures { viewBinding = true }
}
```

---

## 1️⃣ Coroutine

| 사용처 | Dispatcher | 스코프 |
| --- | --- | --- |
| Retrofit 호출 (네이버 뉴스/사람인/OpenAI) | IO | viewModelScope |
| Firestore 저장·조회 (`.await()`) | IO | viewModelScope |
| Firebase Storage 업로드 (증명사진) | IO | viewModelScope |
| PdfDocument 생성 | IO | viewModelScope |
| TFLite 추론 | Default | viewModelScope |

```kotlin
fun saveResume(resume: Resume) = viewModelScope.launch {
    runCatching {
        withContext(Dispatchers.IO) { repo.saveToFirestore(resume) }
    }.onSuccess { _state.value = UiState.Success }
     .onFailure { _state.value = UiState.Error(it.message) }
}
```

---

## 2️⃣ 이력서 PDF 생성 — Android 내장 `PdfDocument`

### 개요

- **A4 = 595×842 pt** (72dpi 기준)
- **한글 폰트** 필수 임베딩: `assets/fonts/NotoSansKR-Regular.ttf` 다운로드해서 배치
- 자동 줄바꿈은 `StaticLayout` 사용
- 저장: `MediaStore.Downloads` (Android 10+ 권한 불필요)

### 구현 흐름

1. `ResumeWriteActivity`에서 입력 → "PDF 저장" 버튼
2. ViewModel이 `Dispatchers.IO`에서 `PdfDocument` 생성
3. 한글 Typeface 로드 → `Paint`에 적용 → `Canvas`에 텍스트 그리기
4. `MediaStore.Downloads`에 `resume_YYYYMMDD.pdf`로 저장
5. 토스트 + `Intent.ACTION_VIEW`로 외부 뷰어 열기 옵션

### 코드 골격

```kotlin
class ResumePdfGenerator(private val ctx: Context) {

    suspend fun build(resume: Resume): Uri = withContext(Dispatchers.IO) {
        val doc = PdfDocument()
        val pageInfo = PdfDocument.PageInfo.Builder(595, 842, 1).create()
        val page = doc.startPage(pageInfo)
        val canvas = page.canvas

        val typeface = Typeface.createFromAsset(ctx.assets, "fonts/NotoSansKR-Regular.ttf")
        val title = TextPaint().apply {
            this.typeface = typeface; textSize = 22f; isAntiAlias = true
        }
        val body = TextPaint().apply {
            this.typeface = typeface; textSize = 12f; isAntiAlias = true
        }

        canvas.drawText(resume.name, 40f, 60f, title)
        drawWrapped(canvas, resume.skills.joinToString(", "), body, 40, 100, 515)
        drawWrapped(canvas, resume.projects, body, 40, 200, 515)
        // ... awards, gpa, certificates, highlight

        doc.finishPage(page)

        // MediaStore.Downloads에 저장
        val cv = ContentValues().apply {
            put(MediaStore.MediaColumns.DISPLAY_NAME, "resume_${System.currentTimeMillis()}.pdf")
            put(MediaStore.MediaColumns.MIME_TYPE, "application/pdf")
            put(MediaStore.MediaColumns.RELATIVE_PATH, Environment.DIRECTORY_DOWNLOADS)
        }
        val uri = ctx.contentResolver.insert(MediaStore.Downloads.EXTERNAL_CONTENT_URI, cv)!!
        ctx.contentResolver.openOutputStream(uri)!!.use { doc.writeTo(it) }
        doc.close()
        uri
    }

    private fun drawWrapped(c: Canvas, text: String, p: TextPaint, x: Int, y: Int, w: Int) {
        val layout = StaticLayout.Builder.obtain(text, 0, text.length, p, w).build()
        c.save(); c.translate(x.toFloat(), y.toFloat()); layout.draw(c); c.restore()
    }
}
```

---

## 3️⃣ 갤러리 연동 : 증명사진 업로드

### 핵심

- **Photo Picker** 사용 (Android 13+, 백포트 동작) → 권한 불필요
- 선택한 이미지를 Firebase Storage에 업로드 → URL을 Firestore 유저 문서에 저장

### 코드 (Activity 또는 Fragment)

```kotlin
private val pickPhoto = registerForActivityResult(
    ActivityResultContracts.PickVisualMedia()
) { uri ->
    uri ?: return@registerForActivityResult
    viewModel.uploadProfilePhoto(uri)
}

binding.btnPickPhoto.setOnClickListener {
    pickPhoto.launch(
        PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
    )
}
```

```kotlin
// ViewModel
fun uploadProfilePhoto(uri: Uri) = viewModelScope.launch {
    val uid = Firebase.auth.currentUser?.uid ?: return@launch
    val ref = Firebase.storage.reference.child("profile/$uid.jpg")
    ref.putFile(uri).await()
    val url = ref.downloadUrl.await().toString()
    Firebase.firestore.collection("users").document(uid)
        .update("photoUrl", url).await()
}
```

### 의존성

```kotlin
implementation("androidx.activity:activity-ktx:1.9.3")
implementation("com.google.firebase:firebase-storage-ktx")
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-play-services:1.8.1")
```

표시용 ImageView: `ImageView.setImageURI(uri)` 또는 `BitmapFactory.decodeStream()`

---

## 4️⃣ Firebase — 저장 항목

### Authentication

- 방식: 이메일/비밀번호 (`signInWithEmailAndPassword`)
- 회원가입 시: `createUserWithEmailAndPassword` → 성공하면 `Firebase.auth.currentUser.uid` 받음
- "id"는 UID(자동) 또는 별도 닉네임 필드로 관리

### Firestore 구조

```
users/{uid}
  ├─ email: String
  ├─ nickname: String              // 사용자 지정 id
  ├─ photoUrl: String?             // 증명사진 URL
  └─ createdAt: Timestamp

users/{uid}/resumes/{resumeId}
  ├─ skills: [String]
  ├─ projects: String
  ├─ awards: String
  ├─ gpa: String
  ├─ certificates: String
  ├─ highlight: String
  ├─ recommendedKeywords: [String] // ML 결과 5개
  └─ updatedAt: Timestamp

users/{uid}/interviews/{sessionId}
  ├─ jobKeyword: String
  ├─ createdAt: Timestamp
  └─ messages: [
        { role: "user"|"assistant", content: String, ts: Timestamp }
     ]
```

### 보안 규칙 (firestore.rules)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    match /users/{uid}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

### 의존성

```kotlin
implementation(platform("com.google.firebase:firebase-bom:33.4.0"))
implementation("com.google.firebase:firebase-auth-ktx")
implementation("com.google.firebase:firebase-firestore-ktx")
implementation("com.google.firebase:firebase-storage-ktx")
```

---

## 5️⃣ 외부 API — 뉴스 + 채용공고

### 5-A. 네이버 뉴스 : **네이버 검색 OpenAPI**

**준비**

1. https://developers.naver.com → 애플리케이션 등록
2. "검색" 서비스 선택 → Client ID/Secret 발급
3. `local.properties`에 키 저장 → BuildConfig로 노출

```
# local.properties
NAVER_CLIENT_ID="xxxxxxx"
NAVER_CLIENT_SECRET="xxxxxxx"
```

```kotlin
// build.gradle.kts (app)
val props = Properties().apply { load(rootProject.file("local.properties").inputStream()) }
defaultConfig {
    buildConfigField("String", "NAVER_ID", "\"${props["NAVER_CLIENT_ID"]}\"")
    buildConfigField("String", "NAVER_SECRET", "\"${props["NAVER_CLIENT_SECRET"]}\"")
}
buildFeatures { buildConfig = true }
```

**Retrofit 인터페이스**

```kotlin
interface NaverApi {
    @Headers(
        "X-Naver-Client-Id: ${BuildConfig.NAVER_ID}",
        "X-Naver-Client-Secret: ${BuildConfig.NAVER_SECRET}"
    )
    @GET("v1/search/news.json")
    suspend fun searchNews(
        @Query("query") query: String,
        @Query("display") display: Int = 10,
        @Query("sort") sort: String = "date"
    ): NewsResponse
}

data class NewsResponse(val items: List<NewsItem>)
data class NewsItem(val title: String, val link: String, val description: String, val pubDate: String)
```

**흐름**: ML이 추천한 키워드 5개를 각각 query로 5번 호출 → RecyclerView에 합쳐서 표시 → 클릭 시 `Intent.ACTION_VIEW`로 원문 열기. `<b>` 태그가 결과에 섞여 오니 `Html.fromHtml()`로 정리.

### 5-B. 채용 정보 1안 : 사람인 OpenAPI

- 포털: https://oapi.saramin.co.kr
- 엔드포인트: `https://oapi.saramin.co.kr/job-search`
- 주요 파라미터: `access-key`, `keywords`, `count`, `sr`(소팅), `loc_cd`(지역코드), `job_type`(고용형태)

```kotlin
interface SaraminApi {
    @GET("job-search")
    suspend fun search(
        @Query("access-key") key: String = BuildConfig.SARAMIN_KEY,
        @Query("keywords") keywords: String,
        @Query("count") count: Int = 20
    ): SaraminResponse
}
```

`jobs.job[]` 안에 공고 객체가 들어있는 구조

### 5-C. 채용 정보 2안 : 잡코리아 캘린더 WebView

```xml
<!-- activity_jobcalendar.xml -->
<WebView
    android:id="@+id/webview"
    android:layout_width="match_parent"
    android:layout_height="match_parent"/>
```

```kotlin
class JobCalendarActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val webView = WebView(this).apply {
            settings.javaScriptEnabled = true
            settings.domStorageEnabled = true
            webViewClient = WebViewClient()
            loadUrl("https://www.jobkorea.co.kr/Starter/calendar/sub/month")
        }
        setContentView(webView)
    }
}
```

`AndroidManifest.xml`에 인터넷 권한 필요:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

---

## 6️⃣ ML 이력서 → 직군 키워드 5개 (TFLite)

### 모델 설계 (간단 버전)

- **입력**: 이력서 텍스트 (skills + gpa + projects + awards + 등을 이어붙임)
- **전처리**: 화이트스페이스 토큰화 + 사전(vocab) 매핑을 통해 고정 길이 벡터(예: 64)로 변환
- **모델**: Embedding(8000, 16) → GlobalAvgPool → Dense(32, relu) → Dense(N, softmax)
- **출력 N개 직군**: 프론트엔드, 백엔드, 인프라, AI/ML, 데이터, 보안, 게임, QA, 임베디드, 연구 등
- **점수제로 상위 5개 라벨**을 키워드로 반환

### 학습 (PC, Python)

가상 이력서 데이터 800~1000개 정도 GPT로 생성 → 라벨링 → Keras로 학습 → `model.tflite`로 변환 → `vocab.txt`와 함께 `assets/`에 배치

```python
import tensorflow as tf
converter = tf.lite.TFLiteConverter.from_keras_model(model)
open("model.tflite", "wb").write(converter.convert())
```

### 안드로이드 추론

```kotlin
class JobClassifier(ctx: Context) {
    private val interpreter: Interpreter
    private val vocab: Map<String, Int>
    private val labels = listOf(jobLists)

    init {
        val afd = ctx.assets.openFd("model.tflite")
        val fis = FileInputStream(afd.fileDescriptor)
        val bb = fis.channel.map(FileChannel.MapMode.READ_ONLY, afd.startOffset, afd.declaredLength)
        interpreter = Interpreter(bb)
        vocab = ctx.assets.open("vocab.txt").bufferedReader()
            .readLines().mapIndexed { i, w -> w to i }.toMap()
    }

    suspend fun top5(text: String): List<String> = withContext(Dispatchers.Default) {
        val tokens = text.split(Regex("\\s+")).take(64)
        val input = IntArray(64) { i -> vocab[tokens.getOrNull(i)] ?: 0 }
        val inputArr = arrayOf(input)
        val output = Array(1) { FloatArray(labels.size) }
        interpreter.run(inputArr, output)
        output[0].withIndex()
            .sortedByDescending { it.value }
            .take(5)
            .map { labels[it.index] }
    }
}
```

### 의존성

```kotlin
implementation("org.tensorflow:tensorflow-lite:2.14.0")
implementation("org.tensorflow:tensorflow-lite-support:0.4.4")
```

---

## 7️⃣ Activity 구성 + Intent

| Activity | 역할 |
| --- | --- |
| `LoginActivity` | 로그인/회원가입 |
| `MainActivity` | 홈 + BottomNav (뉴스/채용/마이) |
| `ResumeWriteActivity` | 이력서 작성 → ML 키워드 추천 → PDF 저장 |
| `InterviewActivity` | ChatGPT 모의면접(RecyclerView) |
| `JobCalendarActivity` 또는 `JobListActivity` | 사람인 결과(RecyclerView) or 잡코리아 캘린더 WebView
뉴스는 (Fragment + ViewPager) |
|  `InformationActivity` | 로그인 유저의 id, pw, 이력서 내용 조회 가능 (Drawer Layout) |

### Intent 데이터 전달

**(a) 단방향**: Main → ResumeWrite

```kotlin
startActivity(Intent(this, ResumeWriteActivity::class.java).apply {
    putExtra("MODE", "EDIT")
    putExtra("RESUME_ID", id)
})
```

**(b) 양방향**: ResumeWrite → JobList (키워드 전달) → 다시 Main으로 결과 반환

```kotlin
// ResumeWriteActivity에서 ML 추론 후
val launcher = registerForActivityResult(StartActivityForResult()) { result ->
    if (result.resultCode == RESULT_OK) {
        val viewed = result.data?.getStringArrayListExtra("VIEWED_JOBS")
        // 처리
    }
}
launcher.launch(Intent(this, JobListActivity::class.java)
    .putStringArrayListExtra("KEYWORDS", ArrayList(top5)))

// JobListActivity 종료 시
setResult(RESULT_OK, Intent().putStringArrayListExtra("VIEWED_JOBS", arrayListOf(...)))
finish()
```

---

---

## 📦 최종 의존성 (build.gradle.kts)

```kotlin
dependencies {
    // AndroidX
    implementation("androidx.core:core-ktx:1.13.1")
    implementation("androidx.appcompat:appcompat:1.7.0")
    implementation("com.google.android.material:material:1.12.0")
    implementation("androidx.activity:activity-ktx:1.9.3")
    implementation("androidx.fragment:fragment-ktx:1.8.5")
    implementation("androidx.constraintlayout:constraintlayout:2.1.4")
    implementation("androidx.recyclerview:recyclerview:1.3.2")
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.7")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.7")

    // 코루틴
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-play-services:1.8.1")

    // Retrofit (네이버/사람인/OpenAI)
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.retrofit2:converter-moshi:2.11.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")

    // Firebase
    implementation(platform("com.google.firebase:firebase-bom:33.4.0"))
    implementation("com.google.firebase:firebase-auth-ktx")
    implementation("com.google.firebase:firebase-firestore-ktx")
    implementation("com.google.firebase:firebase-storage-ktx")

    // TFLite
    implementation("org.tensorflow:tensorflow-lite:2.14.0")
    implementation("org.tensorflow:tensorflow-lite-support:0.4.4")
}
```

---

## ✅ 구현 시작 체크리스트

- [ ]  `compileSdk=34`, `minSdk=24`, JDK 17 확인
- [ ]  Firebase 프로젝트 만들고 `google-services.json` `app/` 폴더에 추가
- [ ]  `firestore.rules` 등록
- [ ]  네이버 개발자 센터에서 검색 API 키 발급 → `local.properties`
- [ ]  사람인 OpenAPI 신청 (병행) — 거절/지연되면 2안(WebView)으로
- [ ]  `assets/fonts/NotoSansKR-Regular.ttf`, `assets/model.tflite`, `assets/vocab.txt` 준비
- [ ]  AndroidManifest에 `INTERNET` 권한 + FileProvider(필요 시) + Provider authorities

---