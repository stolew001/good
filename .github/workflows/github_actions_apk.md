name: Build A4 Dewarp Scanner APK & AAB

on:
  push:
    branches: [ main, master ]
  workflow_dispatch:

jobs:
  build-android:
    runs-on: ubuntu-latest

    steps:
      - name: 1. 檢出儲存庫 (Checkout)
        uses: actions/checkout@v4

      - name: 2. 設定 Java 17 環境 (Setup JDK 17)
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: 3. 設定 Gradle 編譯工具 (Setup Gradle)
        uses: gradle/actions/setup-gradle@v3
        with:
          gradle-version: '8.7'

      - name: 4. 自動生成金鑰與 Android 專案原始碼
        run: |
          mkdir -p app/src/main/java/com/a4scanner/dewarp
          mkdir -p app/src/main/res/xml
          mkdir -p app/src/main/res/values

          # 產生簽名金鑰 (讓編譯出的 APK 可直接安裝於手機，AAB 可用於測試上架)
          keytool -genkey -v \
            -keystore release-key.jks \
            -alias a4scanner \
            -keyalg RSA \
            -keysize 2048 \
            -validity 10000 \
            -storepass android123 \
            -keypass android123 \
            -dname "CN=A4Scanner, OU=Mobile, O=A4Scanner, L=HK, S=HK, C=HK"

          cat << 'EOF' > settings.gradle
          pluginManagement {
              repositories {
                  google()
                  mavenCentral()
                  gradlePluginPortal()
              }
          }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories {
                  google()
                  mavenCentral()
              }
          }
          rootProject.name = "A4DewarpScanner"
          include(":app")
          EOF

          cat << 'EOF' > gradle.properties
          org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
          android.useAndroidX=true
          kotlin.code.style=official
          EOF

          cat << 'EOF' > build.gradle
          plugins {
              id 'com.android.application' version '8.4.0' apply false
              id 'org.jetbrains.kotlin.android' version '1.9.24' apply false
          }
          EOF

          cat << 'EOF' > app/build.gradle
          plugins {
              id 'com.android.application'
              id 'org.jetbrains.kotlin.android'
          }

          android {
              namespace 'com.a4scanner.dewarp'
              compileSdk 34

              defaultConfig {
                  applicationId "com.a4scanner.dewarp"
                  minSdk 24
                  targetSdk 34
                  versionCode 1
                  versionName "1.0.0"
              }

              signingConfigs {
                  release {
                      storeFile file("../release-key.jks")
                      storePassword "android123"
                      keyAlias "a4scanner"
                      keyPassword "android123"
                  }
              }

              buildTypes {
                  release {
                      signingConfig signingConfigs.release
                      minifyEnabled false
                  }
              }

              compileOptions {
                  sourceCompatibility JavaVersion.VERSION_17
                  targetCompatibility JavaVersion.VERSION_17
              }

              kotlinOptions {
                  jvmTarget = '17'
              }
          }

          dependencies {
              implementation 'androidx.core:core-ktx:1.12.0'
          }
          EOF

          cat << 'EOF' > app/src/main/res/xml/file_paths.xml
          <?xml version="1.0" encoding="utf-8"?>
          <paths>
              <cache-path name="camera_photos" path="." />
          </paths>
          EOF

          cat << 'EOF' > app/src/main/AndroidManifest.xml
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">

              <application
                  android:allowBackup="true"
                  android:label="A4 捲曲拉平掃描器"
                  android:supportsRtl="true"
                  android:theme="@android:style/Theme.DeviceDefault.NoActionBar">

                  <activity
                      android:name=".MainActivity"
                      android:exported="true"
                      android:screenOrientation="portrait">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>

                  <provider
                      android:name="androidx.core.content.FileProvider"
                      android:authorities="${applicationId}.fileprovider"
                      android:exported="false"
                      android:grantUriPermissions="true">
                      <meta-data
                          android:name="android.support.FILE_PROVIDER_PATHS"
                          android:resource="@xml/file_paths" />
                  </provider>
              </application>
          </manifest>
          EOF

          cat << 'EOF' > app/src/main/java/com/a4scanner/dewarp/MainActivity.kt
          package com.a4scanner.dewarp

          import android.app.Activity
          import android.content.Context
          import android.content.Intent
          import android.graphics.*
          import android.graphics.pdf.PdfDocument
          import android.net.Uri
          import android.os.Bundle
          import android.provider.MediaStore
          import android.view.Gravity
          import android.view.MotionEvent
          import android.view.View
          import android.widget.*
          import androidx.core.content.FileProvider
          import java.io.File
          import kotlin.math.hypot
          import kotlin.math.max
          import kotlin.math.min
          import kotlin.math.roundToInt

          class MainActivity : Activity() {

              private lateinit var cropView: A4CurveOverlayView
              private lateinit var statusText: TextView
              private lateinit var btnPreviewToggle: Button
              private lateinit var btnFilterToggle: Button
              private lateinit var btnAxisToggle: Button

              private var sourceBitmap: Bitmap? = null
              private var dewarpedBitmap: Bitmap? = null
              private var cameraImageUri: Uri? = null

              private var isPreviewMode = false
              // 0 = 原色, 1 = 文件增亮, 2 = 黑白高對比
              private var filterMode = 1

              companion object {
                  private const val REQ_PICK_IMAGE = 1001
                  private const val REQ_TAKE_PHOTO = 1002
                  private const val REQ_SAVE_PDF = 1003

                  // 標準 A4 比例 (1 : 1.4142) 輸出解析度
                  private const val A4_WIDTH = 1414
                  private const val A4_HEIGHT = 2000
              }

              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)

                  val root = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL
                      setBackgroundColor(Color.parseColor("#121212"))
                  }

                  // 頂部標題與提示列
                  statusText = TextView(this).apply {
                      text = "請拍照或選擇相片，拖曳 4 個角點與 2 個弧線中點貼合 A4 紙邊緣"
                      setTextColor(Color.WHITE)
                      textSize = 14f
                      setPadding(32, 28, 32, 20)
                      setBackgroundColor(Color.parseColor("#1E1E1E"))
                  }
                  root.addView(statusText, LinearLayout.LayoutParams(
                      LinearLayout.LayoutParams.MATCH_PARENT,
                      LinearLayout.LayoutParams.WRAP_CONTENT
                  ))

                  // 中央 A4 6點弧線校正畫布
                  cropView = A4CurveOverlayView(this)
                  root.addView(cropView, LinearLayout.LayoutParams(
                      LinearLayout.LayoutParams.MATCH_PARENT,
                      0,
                      1f
                  ))

                  // 第一排工具按鈕：旋轉、切換捲曲方向、切換色彩增強、預覽拉平效果
                  val toolRow = LinearLayout(this).apply {
                      orientation = LinearLayout.HORIZONTAL
                      setPadding(16, 12, 16, 4)
                      setBackgroundColor(Color.parseColor("#1E1E1E"))
                  }

                  val btnRotate = createButton("旋轉 90°", "#333333") {
                      sourceBitmap?.let { bmp ->
                          val matrix = Matrix().apply { postRotate(90f) }
                          sourceBitmap = Bitmap.createBitmap(bmp, 0, 0, bmp.width, bmp.height, matrix, true)
                          isPreviewMode = false
                          btnPreviewToggle.text = "預覽拉平"
                          cropView.setBitmap(sourceBitmap, resetPoints = true)
                      }
                  }

                  btnAxisToggle = createButton("弧邊：上下", "#333333") {
                      if (!isPreviewMode) {
                          val vertical = cropView.toggleCurveAxis()
                          btnAxisToggle.text = if (vertical) "弧邊：上下" else "弧邊：左右"
                      }
                  }

                  btnFilterToggle = createButton("模式：增亮", "#333333") {
                      filterMode = (filterMode + 1) % 3
                      btnFilterToggle.text = when (filterMode) {
                          0 -> "模式：原色"
                          1 -> "模式：增亮"
                          else -> "模式：黑白"
                      }
                      if (isPreviewMode) updateDewarpPreview()
                  }

                  btnPreviewToggle = createButton("預覽拉平", "#0277BD") {
                      if (sourceBitmap == null) return@createButton
                      isPreviewMode = !isPreviewMode
                      if (isPreviewMode) {
                          updateDewarpPreview()
                          btnPreviewToggle.text = "返回調整點"
                          statusText.text = "預覽標準 A4 拉平結果（比例 1 : 1.414），確認無誤即可匯出 PDF"
                      } else {
                          cropView.showPreviewBitmap(null)
                          btnPreviewToggle.text = "預覽拉平"
                          statusText.text = "拖曳 4 個角點與 2 個圓形弧度控制點，對齊彎曲紙面"
                      }
                  }

                  val lp = LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1f).apply {
                      setMargins(8, 0, 8, 0)
                  }
                  toolRow.addView(btnRotate, lp)
                  toolRow.addView(btnAxisToggle, lp)
                  toolRow.addView(btnFilterToggle, lp)
                  toolRow.addView(btnPreviewToggle, lp)
                  root.addView(toolRow)

                  // 第二排主要操作按鈕：拍照、相簿、匯出 A4 PDF
                  val actionRow = LinearLayout(this).apply {
                      orientation = LinearLayout.HORIZONTAL
                      setPadding(16, 8, 16, 24)
                      setBackgroundColor(Color.parseColor("#1E1E1E"))
                  }

                  val btnCamera = createButton("拍照", "#2E7D32") { takePhoto() }
                  val btnGallery = createButton("相簿選圖", "#455A64") { pickFromGallery() }
                  val btnExportPdf = createButton("匯出 A4 PDF", "#D84315") { startExportPdf() }

                  actionRow.addView(btnCamera, lp)
                  actionRow.addView(btnGallery, lp)
                  actionRow.addView(btnExportPdf, lp)
                  root.addView(actionRow)

                  setContentView(root)

                  // 建立預設示範紙張，打開 App 即可直接體驗拉平效果
                  loadDemoCurvedSheet()
              }

              private fun createButton(label: String, bgHex: String, onClick: () -> Unit): Button {
                  return Button(this).apply {
                      text = label
                      textSize = 13f
                      setTextColor(Color.WHITE)
                      setBackgroundColor(Color.parseColor(bgHex))
                      setOnClickListener { onClick() }
                  }
              }

              private fun loadDemoCurvedSheet() {
                  val w = 1000
                  val h = 1400
                  val bmp = Bitmap.createBitmap(w, h, Bitmap.Config.ARGB_8888)
                  val canvas = Canvas(bmp)
                  canvas.drawColor(Color.parseColor("#2B2B2B"))

                  val paint = Paint(Paint.ANTI_ALIAS_FLAG)
                  // 繪製一張模擬單向拱起彎曲的 A4 紙張
                  val path = Path().apply {
                      moveTo(150f, 220f)
                      quadTo(500f, 90f, 850f, 220f)
                      lineTo(850f, 1180f)
                      quadTo(500f, 1050f, 150f, 1180f)
                      close()
                  }
                  paint.color = Color.parseColor("#F5F5F0")
                  canvas.drawPath(path, paint)

                  // 繪製紙張上的彎曲文字行幫助測試拉平效果
                  paint.color = Color.parseColor("#222222")
                  paint.style = Paint.Style.STROKE
                  paint.strokeWidth = 6f
                  for (i in 1..10) {
                      val y = 220f + i * 80f
                      val linePath = Path().apply {
                          moveTo(210f, y)
                          quadTo(500f, y - 110f, 790f, y)
                      }
                      canvas.drawPath(linePath, paint)
                  }

                  sourceBitmap = bmp
                  cropView.setBitmap(bmp, resetPoints = true)
                  cropView.setDemoPoints()
              }

              private fun updateDewarpPreview() {
                  val src = sourceBitmap ?: return
                  dewarpedBitmap = performSingleAxisDewarp(
                      src,
                      cropView.getNormalizedPoints(),
                      cropView.isTopBottomCurved(),
                      filterMode
                  )
                  cropView.showPreviewBitmap(dewarpedBitmap)
              }

              /**
               * 核心單向 A4 柱面展開演算法：
               * 利用二次貝茲曲線擬合彎曲邊緣，並按真實弧長等距展開至 1 : 1.414 標準 A4 平面
               */
              private fun performSingleAxisDewarp(
                  src: Bitmap,
                  pts: Array<PointF>,
                  topBottomCurved: Boolean,
                  mode: Int
              ): Bitmap {
                  val srcW = src.width
                  val srcH = src.height
                  val srcPixels = IntArray(srcW * srcH)
                  src.getPixels(srcPixels, 0, srcW, 0, 0, srcW, srcH)

                  val outW = A4_WIDTH
                  val outH = A4_HEIGHT
                  val outPixels = IntArray(outW * outH)

                  // pts 順序: 0:TL, 1:TR, 2:BR, 3:BL, 4:弧線控制點1(上或左), 5:弧線控制點2(下或右)
                  val tl = PointF(pts[0].x * srcW, pts[0].y * srcH)
                  val tr = PointF(pts[1].x * srcW, pts[1].y * srcH)
                  val br = PointF(pts[2].x * srcW, pts[2].y * srcH)
                  val bl = PointF(pts[3].x * srcW, pts[3].y * srcH)
                  val m1 = PointF(pts[4].x * srcW, pts[4].y * srcH)
                  val m2 = PointF(pts[5].x * srcW, pts[5].y * srcH)

                  if (topBottomCurved) {
                      // 上下邊為彎曲弧線，左右邊為直線
                      // 求通過中點 m1, m2 的二次貝茲控制點 Pc = 2*M - 0.5*(P0 + P2)
                      val ctrlTop = PointF(2f * m1.x - 0.5f * (tl.x + tr.x), 2f * m1.y - 0.5f * (tl.y + tr.y))
                      val ctrlBot = PointF(2f * m2.x - 0.5f * (bl.x + br.x), 2f * m2.y - 0.5f * (bl.y + br.y))

                      val topX = FloatArray(outW)
                      val topY = FloatArray(outW)
                      val botX = FloatArray(outW)
                      val botY = FloatArray(outW)

                      // 依弧長等距採樣，消除紙張捲曲造成的橫向字距壓縮
                      val uTable = buildArcLengthLUT(tl, ctrlTop, tr, outW)

                      for (x in 0 until outW) {
                          val u = uTable[x]
                          val uInv = 1f - u
                          topX[x] = uInv * uInv * tl.x + 2f * uInv * u * ctrlTop.x + u * u * tr.x
                          topY[x] = uInv * uInv * tl.y + 2f * uInv * u * ctrlTop.y + u * u * tr.y
                          botX[x] = uInv * uInv * bl.x + 2f * uInv * u * ctrlBot.x + u * u * br.x
                          botY[x] = uInv * uInv * bl.y + 2f * uInv * u * ctrlBot.y + u * u * br.y
                      }

                      for (y in 0 until outH) {
                          val v = y.toFloat() / (outH - 1)
                          val rowOffset = y * outW
                          for (x in 0 until outW) {
                              val sx = (topX[x] + v * (botX[x] - topX[x])).roundToInt().coerceIn(0, srcW - 1)
                              val sy = (topY[x] + v * (botY[x] - topY[x])).roundToInt().coerceIn(0, srcH - 1)
                              outPixels[rowOffset + x] = enhancePixel(srcPixels[sy * srcW + sx], mode)
                          }
                      }
                  } else {
                      // 左右邊為彎曲弧線，上下邊為直線
                      val ctrlLeft = PointF(2f * m1.x - 0.5f * (tl.x + bl.x), 2f * m1.y - 0.5f * (tl.y + bl.y))
                      val ctrlRight = PointF(2f * m2.x - 0.5f * (tr.x + br.x), 2f * m2.y - 0.5f * (tr.y + br.y))

                      val vTable = buildArcLengthLUT(tl, ctrlLeft, bl, outH)

                      for (y in 0 until outH) {
                          val v = vTable[y]
                          val vInv = 1f - v
                          val lx = vInv * vInv * tl.x + 2f * vInv * v * ctrlLeft.x + v * v * bl.x
                          val ly = vInv * vInv * tl.y + 2f * vInv * v * ctrlLeft.y + v * v * bl.y
                          val rx = vInv * vInv * tr.x + 2f * vInv * v * ctrlRight.x + v * v * br.x
                          val ry = vInv * vInv * tr.y + 2f * vInv * v * ctrlRight.y + v * v * br.y

                          val rowOffset = y * outW
                          for (x in 0 until outW) {
                              val u = x.toFloat() / (outW - 1)
                              val sx = (lx + u * (rx - lx)).roundToInt().coerceIn(0, srcW - 1)
                              val sy = (ly + u * (ry - ly)).roundToInt().coerceIn(0, srcH - 1)
                              outPixels[rowOffset + x] = enhancePixel(srcPixels[sy * srcW + sx], mode)
                          }
                      }
                  }

                  val result = Bitmap.createBitmap(outW, outH, Bitmap.Config.ARGB_8888)
                  result.setPixels(outPixels, 0, outW, 0, 0, outW, outH)
                  return result
              }

              // 計算二次貝茲曲線真實弧長對照表，確保彎曲處拉平後比例均勻
              private fun buildArcLengthLUT(p0: PointF, pc: PointF, p2: PointF, steps: Int): FloatArray {
                  val samples = 500
                  val arcLengths = FloatArray(samples + 1)
                  var prevX = p0.x
                  var prevY = p0.y
                  var totalLen = 0f

                  for (i in 1..samples) {
                      val t = i.toFloat() / samples
                      val inv = 1f - t
                      val cx = inv * inv * p0.x + 2f * inv * t * pc.x + t * t * p2.x
                      val cy = inv * inv * p0.y + 2f * inv * t * pc.y + t * t * p2.y
                      totalLen += hypot(cx - prevX, cy - prevY)
                      arcLengths[i] = totalLen
                      prevX = cx
                      prevY = cy
                  }

                  val lut = FloatArray(steps)
                  if (totalLen <= 0.001f) {
                      for (i in 0 until steps) lut[i] = i.toFloat() / (steps - 1)
                      return lut
                  }

                  var idx = 0
                  for (i in 0 until steps) {
                      val target = (i.toFloat() / (steps - 1)) * totalLen
                      while (idx < samples && arcLengths[idx + 1] < target) {
                          idx++
                      }
                      val segStart = arcLengths[idx]
                      val segEnd = arcLengths[min(idx + 1, samples)]
                      val frac = if (segEnd > segStart) (target - segStart) / (segEnd - segStart) else 0f
                      lut[i] = (idx + frac) / samples
                  }
                  return lut
              }

              private fun enhancePixel(color: Int, mode: Int): Int {
                  if (mode == 0) return color
                  val r = (color shr 16) and 0xFF
                  val g = (color shr 8) and 0xFF
                  val b = color and 0xFF

                  return if (mode == 1) {
                      // 文件增亮：拉高紙張白底並加深字跡對比
                      val nr = ((r - 128) * 1.35f + 148).toInt().coerceIn(0, 255)
                      val ng = ((g - 128) * 1.35f + 148).toInt().coerceIn(0, 255)
                      val nb = ((b - 128) * 1.35f + 148).toInt().coerceIn(0, 255)
                      (0xFF shl 24) or (nr shl 16) or (ng shl 8) or nb
                  } else {
                      // 黑白高對比模式
                      val lum = (0.299f * r + 0.587f * g + 0.114f * b).toInt()
                      val bw = if (lum > 150) 255 else max(0, (lum - 50) * 2)
                      (0xFF shl 24) or (bw shl 16) or (bw shl 8) or bw
                  }
              }

              private fun takePhoto() {
                  try {
                      val photoFile = File(cacheDir, "a4_capture.jpg")
                      cameraImageUri = FileProvider.getUriForFile(
                          this,
                          "$packageName.fileprovider",
                          photoFile
                      )
                      val intent = Intent(MediaStore.ACTION_IMAGE_CAPTURE).apply {
                          putExtra(MediaStore.EXTRA_OUTPUT, cameraImageUri)
                      }
                      startActivityForResult(intent, REQ_TAKE_PHOTO)
                  } catch (e: Exception) {
                      Toast.makeText(this, "無法啟動相機：${e.message}", Toast.LENGTH_SHORT).show()
                  }
              }

              private fun pickFromGallery() {
                  val intent = Intent(Intent.ACTION_GET_CONTENT).apply {
                      type = "image/*"
                  }
                  startActivityForResult(Intent.createChooser(intent, "選擇 A4 紙張相片"), REQ_PICK_IMAGE)
              }

              private fun startExportPdf() {
                  val src = sourceBitmap ?: return
                  if (dewarpedBitmap == null || !isPreviewMode) {
                      dewarpedBitmap = performSingleAxisDewarp(
                          src,
                          cropView.getNormalizedPoints(),
                          cropView.isTopBottomCurved(),
                          filterMode
                      )
                  }
                  val intent = Intent(Intent.ACTION_CREATE_DOCUMENT).apply {
                      addCategory(Intent.CATEGORY_OPENABLE)
                      type = "application/pdf"
                      putExtra(Intent.EXTRA_TITLE, "A4_Scan_${System.currentTimeMillis() / 1000}.pdf")
                  }
                  startActivityForResult(intent, REQ_SAVE_PDF)
              }

              override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
                  super.onActivityResult(requestCode, resultCode, data)
                  if (resultCode != RESULT_OK) return

                  when (requestCode) {
                      REQ_PICK_IMAGE -> {
                          data?.data?.let { loadUriBitmap(it) }
                      }
                      REQ_TAKE_PHOTO -> {
                          cameraImageUri?.let { loadUriBitmap(it) }
                      }
                      REQ_SAVE_PDF -> {
                          data?.data?.let { uri ->
                              writeBitmapToPdf(uri)
                          }
                      }
                  }
              }

              private fun loadUriBitmap(uri: Uri) {
                  try {
                      contentResolver.openInputStream(uri)?.use { input ->
                          val raw = BitmapFactory.decodeStream(input)
                          if (raw != null) {
                              // 限制最大邊長避免記憶體溢出，同時保留高解析度細節
                              val maxDim = max(raw.width, raw.height)
                              sourceBitmap = if (maxDim > 2400) {
                                  val scale = 2400f / maxDim
                                  Bitmap.createScaledBitmap(
                                      raw,
                                      (raw.width * scale).roundToInt(),
                                      (raw.height * scale).roundToInt(),
                                      true
                                  )
                              } else {
                                  raw
                              }
                              isPreviewMode = false
                              btnPreviewToggle.text = "預覽拉平"
                              cropView.setBitmap(sourceBitmap, resetPoints = true)
                              statusText.text = "請拖曳 4 個角點與 2 個圓形弧線控制點貼合紙張邊緣"
                          }
                      }
                  } catch (e: Exception) {
                      Toast.makeText(this, "讀取圖片失敗：${e.message}", Toast.LENGTH_SHORT).show()
                  }
              }

              private fun writeBitmapToPdf(uri: Uri) {
                  val bmp = dewarpedBitmap ?: return
                  try {
                      val pdfDoc = PdfDocument()
                      val pageInfo = PdfDocument.PageInfo.Builder(A4_WIDTH, A4_HEIGHT, 1).create()
                      val page = pdfDoc.startPage(pageInfo)
                      page.canvas.drawBitmap(bmp, 0f, 0f, null)
                      pdfDoc.finishPage(page)

                      contentResolver.openOutputStream(uri)?.use { out ->
                          pdfDoc.writeTo(out)
                      }
                      pdfDoc.close()
                      Toast.makeText(this, "A4 PDF 儲存成功！", Toast.LENGTH_LONG).show()
                  } catch (e: Exception) {
                      Toast.makeText(this, "儲存 PDF 失敗：${e.message}", Toast.LENGTH_LONG).show()
                  }
              }
          }

          /**
           * 自訂 6 點單向弧線校正視圖：
           * 4 個方形角點 (TL, TR, BR, BL) + 2 個圓形弧度控制點 (對應單向彎曲的兩條對邊中點)
           */
          class A4CurveOverlayView(context: Context) : View(context) {

              private var bitmap: Bitmap? = null
              private var previewBitmap: Bitmap? = null
              private val imgRect = RectF()

              // 歸一化座標 (0.0 ~ 1.0): 0:TL, 1:TR, 2:BR, 3:BL, 4:Mid1, 5:Mid2
              private val pts = Array(6) { PointF() }
              private var activePointIndex = -1
              private var topBottomCurved = true

              private val edgePaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
                  color = Color.parseColor("#00E5FF")
                  style = Paint.Style.STROKE
                  strokeWidth = 6f
              }

              private val gridPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
                  color = Color.parseColor("#6600E5FF")
                  style = Paint.Style.STROKE
                  strokeWidth = 2.5f
              }

              private val cornerPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
                  color = Color.parseColor("#00E5FF")
                  style = Paint.Style.FILL
              }

              private val curveHandlePaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
                  color = Color.parseColor("#FFAB00")
                  style = Paint.Style.FILL
              }

              private val strokeWhite = Paint(Paint.ANTI_ALIAS_FLAG).apply {
                  color = Color.WHITE
                  style = Paint.Style.STROKE
                  strokeWidth = 4f
              }

              init {
                  resetDefaultPoints()
              }

              fun setBitmap(bmp: Bitmap?, resetPoints: Boolean) {
                  bitmap = bmp
                  previewBitmap = null
                  if (resetPoints) resetDefaultPoints()
                  invalidate()
              }

              fun showPreviewBitmap(bmp: Bitmap?) {
                  previewBitmap = bmp
                  invalidate()
              }

              fun isTopBottomCurved(): Boolean = topBottomCurved

              fun toggleCurveAxis(): Boolean {
                  topBottomCurved = !topBottomCurved
                  if (topBottomCurved) {
                      pts[4].set((pts[0].x + pts[1].x) / 2f, (pts[0].y + pts[1].y) / 2f - 0.05f)
                      pts[5].set((pts[3].x + pts[2].x) / 2f, (pts[3].y + pts[2].y) / 2f - 0.05f)
                  } else {
                      pts[4].set((pts[0].x + pts[3].x) / 2f - 0.05f, (pts[0].y + pts[3].y) / 2f)
                      pts[5].set((pts[1].x + pts[2].x) / 2f - 0.05f, (pts[1].y + pts[2].y) / 2f)
                  }
                  invalidate()
                  return topBottomCurved
              }

              fun setDemoPoints() {
                  topBottomCurved = true
                  pts[0].set(0.15f, 0.157f)
                  pts[1].set(0.85f, 0.157f)
                  pts[2].set(0.85f, 0.843f)
                  pts[3].set(0.15f, 0.843f)
                  pts[4].set(0.50f, 0.110f)
                  pts[5].set(0.50f, 0.796f)
                  invalidate()
              }

              private fun resetDefaultPoints() {
                  pts[0].set(0.15f, 0.15f)
                  pts[1].set(0.85f, 0.15f)
                  pts[2].set(0.85f, 0.85f)
                  pts[3].set(0.15f, 0.85f)
                  if (topBottomCurved) {
                      pts[4].set(0.50f, 0.12f)
                      pts[5].set(0.50f, 0.82f)
                  } else {
                      pts[4].set(0.12f, 0.50f)
                      pts[5].set(0.82f, 0.50f)
                  }
              }

              fun getNormalizedPoints(): Array<PointF> {
                  return Array(6) { i -> PointF(pts[i].x, pts[i].y) }
              }

              override fun onDraw(canvas: Canvas) {
                  super.onDraw(canvas)
                  val activeBmp = previewBitmap ?: bitmap ?: return

                  val vw = width.toFloat()
                  val vh = height.toFloat()
                  val pad = 36f
                  val availW = vw - pad * 2
                  val availH = vh - pad * 2

                  val scale = min(availW / activeBmp.width, availH / activeBmp.height)
                  val dw = activeBmp.width * scale
                  val dh = activeBmp.height * scale
                  val left = (vw - dw) / 2f
                  val top = (vh - dh) / 2f
                  imgRect.set(left, top, left + dw, top + dh)

                  canvas.drawBitmap(activeBmp, null, imgRect, null)

                  // 若在預覽模式，只顯示拉平後的 A4 成品，不畫控制點
                  if (previewBitmap != null) return

                  val sPts = Array(6) { i ->
                      PointF(imgRect.left + pts[i].x * imgRect.width(), imgRect.top + pts[i].y * imgRect.height())
                  }

                  val path = Path()
                  if (topBottomCurved) {
                      val cTopX = 2f * sPts[4].x - 0.5f * (sPts[0].x + sPts[1].x)
                      val cTopY = 2f * sPts[4].y - 0.5f * (sPts[0].y + sPts[1].y)
                      val cBotX = 2f * sPts[5].x - 0.5f * (sPts[3].x + sPts[2].x)
                      val cBotY = 2f * sPts[5].y - 0.5f * (sPts[3].y + sPts[2].y)

                      // 繪製內部參考輔助網格線
                      for (i in 1..3) {
                          val v = i / 4f
                          val p0x = sPts[0].x + v * (sPts[3].x - sPts[0].x)
                          val p0y = sPts[0].y + v * (sPts[3].y - sPts[0].y)
                          val p2x = sPts[1].x + v * (sPts[2].x - sPts[1].x)
                          val p2y = sPts[1].y + v * (sPts[2].y - sPts[1].y)
                          val pcx = cTopX + v * (cBotX - cTopX)
                          val pcy = cTopY + v * (cBotY - cTopY)
                          val gPath = Path().apply {
                              moveTo(p0x, p0y)
                              quadTo(pcx, pcy, p2x, p2y)
                          }
                          canvas.drawPath(gPath, gridPaint)
                      }

                      path.moveTo(sPts[0].x, sPts[0].y)
                      path.quadTo(cTopX, cTopY, sPts[1].x, sPts[1].y)
                      path.lineTo(sPts[2].x, sPts[2].y)
                      path.quadTo(cBotX, cBotY, sPts[3].x, sPts[3].y)
                      path.close()
                  } else {
                      val cLeftX = 2f * sPts[4].x - 0.5f * (sPts[0].x + sPts[3].x)
                      val cLeftY = 2f * sPts[4].y - 0.5f * (sPts[0].y + sPts[3].y)
                      val cRightX = 2f * sPts[5].x - 0.5f * (sPts[1].x + sPts[2].x)
                      val cRightY = 2f * sPts[5].y - 0.5f * (sPts[1].y + sPts[2].y)

                      path.moveTo(sPts[0].x, sPts[0].y)
                      path.lineTo(sPts[1].x, sPts[1].y)
                      path.quadTo(cRightX, cRightY, sPts[2].x, sPts[2].y)
                      path.lineTo(sPts[3].x, sPts[3].y)
                      path.quadTo(cLeftX, cLeftY, sPts[0].x, sPts[0].y)
                      path.close()
                  }

                  canvas.drawPath(path, edgePaint)

                  // 繪製 4 個角落控制點 (藍色圓點)
                  for (i in 0..3) {
                      canvas.drawCircle(sPts[i].x, sPts[i].y, 24f, cornerPaint)
                      canvas.drawCircle(sPts[i].x, sPts[i].y, 24f, strokeWhite)
                  }

                  // 繪製 2 個弧度控制點 (橘黃色圓點)
                  for (i in 4..5) {
                      canvas.drawCircle(sPts[i].x, sPts[i].y, 26f, curveHandlePaint)
                      canvas.drawCircle(sPts[i].x, sPts[i].y, 26f, strokeWhite)
                  }
              }

              override fun onTouchEvent(event: MotionEvent): Boolean {
                  if (previewBitmap != null || imgRect.width() <= 0) return false

                  val tx = event.x
                  val ty = event.y

                  when (event.actionMasked) {
                      MotionEvent.ACTION_DOWN -> {
                          var minDist = 90f
                          activePointIndex = -1
                          for (i in 0..5) {
                              val px = imgRect.left + pts[i].x * imgRect.width()
                              val py = imgRect.top + pts[i].y * imgRect.height()
                              val d = hypot(tx - px, ty - py)
                              if (d < minDist) {
                                  minDist = d
                                  activePointIndex = i
                              }
                          }
                          return activePointIndex != -1
                      }
                      MotionEvent.ACTION_MOVE -> {
                          if (activePointIndex != -1) {
                              val nx = ((tx - imgRect.left) / imgRect.width()).coerceIn(0.01f, 0.99f)
                              val ny = ((ty - imgRect.top) / imgRect.height()).coerceIn(0.01f, 0.99f)
                              pts[activePointIndex].set(nx, ny)
                              invalidate()
                              return true
                          }
                      }
                      MotionEvent.ACTION_UP, MotionEvent.ACTION_CANCEL -> {
                          activePointIndex = -1
                      }
                  }
                  return super.onTouchEvent(event)
              }
          }
          EOF

      - name: 5. 執行 Gradle 編譯 APK 與 Google Play AAB
        run: gradle assembleRelease bundleRelease --no-daemon

      - name: 6. 上傳可直接安裝的手機 APK 檔案
        uses: actions/upload-artifact@v4
        with:
          name: A4-Dewarp-Scanner-APK
          path: app/build/outputs/apk/release/app-release.apk

      - name: 7. 上傳可上架 Google Play 的 AAB 檔案
        uses: actions/upload-artifact@v4
        with:
          name: A4-Dewarp-Scanner-GooglePlay-AAB
          path: app/build/outputs/bundle/release/app-release.aab