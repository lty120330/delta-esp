name: build-apk-v13

# ============================================================================
# 一文件工程 v13 —— 免 root / 免 adb，MediaProjection 采屏 + ONNX 推理 + 悬浮窗绘制
#
# v13 修掉的三处致命 bug：
#   1) V12 的 HUD 用 16ms 定时 invalidate() 全屏重绘 → 叠加层 60fps 刷新 + 屏幕
#      同时被 MediaProjection 镜像 = 合成器负载爆掉，进游戏掉到个位数帧率。
#      v13：数据驱动重绘（ver 变化才画），上限 250ms 一次，静止时 4fps。
#   2) V12 的 metrics() 把宽高各自 coerceAtMost(1920/1080) → 竖屏 1080x2400 被压成
#      1080x1080，画面拉伸、框全偏。v13：长边等比缩放到 1280，比例严格保持。
#   3) V12 在 Service.onCreate 同步建 ORT session → 2~10s 阻塞主线程 = ANR。
#      v13：loadAsync 到工作线程，推理侧 sess==null 时返回空，加载完自动接管。
#   另：三槽 Bitmap 池（零分配 + 无空槽即丢帧的背压）、去掉 FLAG_LAYOUT_NO_LIMITS、
#       切片默认关、物资扫描默认关、通知栏可停。
#
# 用法：GitHub 新建仓库 → .github/workflows/build.yml 粘本文件 → Commit
#       6~10 分钟后 Actions → Artifacts → apk.zip → 解压装 app-debug.apk
# ============================================================================
on:
  workflow_dispatch:
  push:
    branches: [ main, master ]

jobs:
  apk:
    runs-on: ubuntu-22.04
    timeout-minutes: 90
    env:
      ANDROID_SDK_ROOT: /home/runner/android-sdk
      ANDROID_HOME: /home/runner/android-sdk

    steps:
      - name: "1 装 JDK 17"
        run: |
          set -e
          sudo apt-get -qq update
          sudo apt-get -qq install -y openjdk-17-jdk-headless unzip curl
          echo "JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64" >> "$GITHUB_ENV"
          /usr/lib/jvm/java-17-openjdk-amd64/bin/java -version

      - name: "2 生成工程骨架"
        run: |
          set -e
          mkdir -p app/src/main/java/com/df/esp
          mkdir -p app/src/main/res/xml app/src/main/res/values app/src/main/assets

          cat > settings.gradle.kts <<'XEOF'
          pluginManagement {
              repositories { google(); mavenCentral(); gradlePluginPortal() }
          }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)
              repositories { google(); mavenCentral() }
          }
          rootProject.name = "df-esp"
          include(":app")
          XEOF

          cat > build.gradle.kts <<'XEOF'
          plugins {
              id("com.android.application") version "8.5.2" apply false
              kotlin("android") version "1.9.24" apply false
          }
          XEOF

          cat > gradle.properties <<'XEOF'
          android.useAndroidX=true
          android.nonTransitiveRClass=true
          org.gradle.jvmargs=-Xmx3g -Dfile.encoding=UTF-8
          org.gradle.parallel=false
          kotlin.code.style=official
          XEOF

          cat > app/build.gradle.kts <<'XEOF'
          plugins {
              id("com.android.application")
              kotlin("android")
          }

          android {
              namespace = "com.df.esp"
              compileSdk = 34

              defaultConfig {
                  applicationId = "com.df.esp"
                  minSdk = 26
                  targetSdk = 34
                  versionCode = 13
                  versionName = "1.9-v13"
              }

              buildTypes {
                  release { isMinifyEnabled = false }
              }

              androidResources {
                  noCompress += "onnx"
              }

              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
          }

          dependencies {
              implementation("com.microsoft.onnxruntime:onnxruntime-android:1.17.1")
          }
          XEOF

          cat > app/src/main/res/values/strings.xml <<'XEOF'
          <?xml version="1.0" encoding="utf-8"?>
          <resources>
              <string name="app_name">设备服务</string>
              <string name="acc_desc">用于显示屏幕叠加标记（不读取任何界面内容）</string>
          </resources>
          XEOF

          cat > app/src/main/res/xml/acc.xml <<'XEOF'
          <?xml version="1.0" encoding="utf-8"?>
          <accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
              android:accessibilityEventTypes="typeWindowStateChanged"
              android:accessibilityFeedbackType="feedbackGeneric"
              android:notificationTimeout="100"
              android:canRetrieveWindowContent="false"
              android:description="@string/acc_desc" />
          XEOF

          cat > app/src/main/AndroidManifest.xml <<'XEOF'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

              <application
                  android:allowBackup="false"
                  android:largeHeap="true"
                  android:label="@string/app_name">

                  <activity
                      android:name=".MainActivity"
                      android:exported="true"
                      android:configChanges="orientation|screenSize|keyboardHidden"
                      android:screenOrientation="portrait">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>

                  <service
                      android:name=".OverlayService"
                      android:exported="false"
                      android:foregroundServiceType="specialUse">
                      <property
                          android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
                          android:value="overlay_hud" />
                  </service>

                  <service
                      android:name=".CaptureService"
                      android:exported="false"
                      android:foregroundServiceType="mediaProjection" />

                  <service
                      android:name=".EspAccessibilityService"
                      android:exported="false"
                      android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
                      <intent-filter>
                          <action android:name="android.accessibilityservice.AccessibilityService" />
                      </intent-filter>
                      <meta-data
                          android:name="android.accessibilityservice"
                          android:resource="@xml/acc" />
                  </service>
              </application>
          </manifest>
          XEOF
          ls -R app/src/main | head -40

      - name: "3 写 State.kt（全局状态 / 盒子模型）"
        run: |
          set -e
          cat > app/src/main/java/com/df/esp/State.kt <<'XEOF'
          package com.df.esp

          import kotlin.math.max
          import kotlin.math.min

          const val VER = "1.9-v13"

          val LABELS = arrayOf("enemy", "head", "mate", "box", "loot", "exit")

          data class Blip(val dx: Float, val dy: Float, val dist: Float)

          data class Box(
              val cls: Int,
              var score: Float,
              var x1: Float,
              var y1: Float,
              var x2: Float,
              var y2: Float,
              var id: Int = -1,
              var vx: Float = 0f,
              var vy: Float = 0f,
              var frames: Int = 0
          ) {
              val w: Float get() = x2 - x1
              val h: Float get() = y2 - y1
              val cx: Float get() = (x1 + x2) * 0.5f
              val cy: Float get() = (y1 + y2) * 0.5f

              fun iou(o: Box): Float {
                  val iw = max(0f, min(x2, o.x2) - max(x1, o.x1))
                  val ih = max(0f, min(y2, o.y2) - max(y1, o.y1))
                  val inter = iw * ih
                  val u = w * h + o.w * o.h - inter
                  return if (u <= 0f) 0f else inter / u
              }
          }

          object RadarDiag {
              @Volatile var redPix = 0
              @Volatile var totalPix = 0
              @Volatile var avgRgb = "-"
              @Volatile var topColors = "-"
              @Volatile var suggest = "-"
              @Volatile var clusters = 0
          }

          object EspState {
              /** 每次检测结果更新 +1；HUD 只在它变化时重绘，避免 60fps 全屏刷 */
              @Volatile var ver = 0

              @Volatile var boxes: List<Box> = emptyList()
              @Volatile var blips: List<Blip> = emptyList()

              @Volatile var enabled = true
              @Volatile var showLoot = false
              @Volatile var hideStatic = false
              @Volatile var conf = 0.22f
              @Volatile var tile = false
              @Volatile var loot = false
              /** 防自反馈：浮层设 FLAG_SECURE，使自己不被 MediaProjection 采到 */
              @Volatile var secure = true

              @Volatile var inferMs = 0f
              @Volatile var capFps = 0f
              @Volatile var frames = 0
              @Volatile var capW = 0
              @Volatile var capH = 0

              @Volatile var mode = "启动中"
              @Volatile var model = ""
              @Volatile var assets = ""
              @Volatile var shape = ""
              @Volatile var err = ""
              @Volatile var capErr = ""
              @Volatile var projErr = ""

              @Volatile var mmOn = true
              @Volatile var mmCxR = 0.855f
              @Volatile var mmCyR = 0.130f
              @Volatile var mmRr = 0.062f
              @Volatile var mmWorldR = 120f
              @Volatile var mmArrows = true
              @Volatile var mmShowRoi = true
              @Volatile var pendingAutoLocate = false

              const val K_DIST = 520f

              @Synchronized
              fun push(b: List<Box>, bl: List<Blip>? = null) {
                  boxes = b
                  if (bl != null) blips = bl
                  ver++
              }
          }
          XEOF
          wc -l app/src/main/java/com/df/esp/State.kt

      - name: "4 写 Tracker.kt（抗闪烁跟踪）"
        run: |
          set -e
          cat > app/src/main/java/com/df/esp/Tracker.kt <<'XEOF'
          package com.df.esp

          /**
           * IoU 关联 + 速度外推 + EMA 平滑。
           *   maxAge 24 帧（约 0.4s）：检测断断续续时框不会一闪就没
           *   smooth 0.45：位置跟随更快，拖影更少
           *   分数衰减 0.94：丢失期间分数掉得慢，不会提前被阈值切掉
           */
          class Tracker(
              private val iouGate: Float = 0.25f,
              private val maxAge: Int = 24,
              private val smooth: Float = 0.45f
          ) {
              private class Track(var box: Box, var age: Int)

              private val tracks = ArrayList<Track>()
              private var nextId = 1

              fun update(det: List<Box>): List<Box> {
                  val used = BooleanArray(det.size)

                  for (t in tracks) {
                      var bi = -1
                      var bv = iouGate
                      for (i in det.indices) {
                          if (used[i] || det[i].cls != t.box.cls) continue
                          val v = t.box.iou(det[i])
                          if (v > bv) { bv = v; bi = i }
                      }
                      if (bi >= 0) {
                          used[bi] = true
                          val d = det[bi]
                          val px = t.box.cx
                          val py = t.box.cy
                          val nx = px + (d.cx - px) * (1 - smooth)
                          val ny = py + (d.cy - py) * (1 - smooth)
                          t.box.vx = nx - px
                          t.box.vy = ny - py
                          val hw = t.box.w + (d.w - t.box.w) * (1 - smooth)
                          val hh = t.box.h + (d.h - t.box.h) * (1 - smooth)
                          t.box.x1 = nx - hw / 2
                          t.box.x2 = nx + hw / 2
                          t.box.y1 = ny - hh / 2
                          t.box.y2 = ny + hh / 2
                          t.box.score += (d.score - t.box.score) * 0.35f
                          t.box.frames++
                          t.age = 0
                      } else {
                          t.age++
                          t.box.x1 += t.box.vx * 0.6f
                          t.box.x2 += t.box.vx * 0.6f
                          t.box.y1 += t.box.vy * 0.6f
                          t.box.y2 += t.box.vy * 0.6f
                          t.box.score *= 0.94f
                      }
                  }

                  for (i in det.indices) {
                      if (!used[i]) {
                          det[i].id = nextId++
                          det[i].frames = 1
                          tracks.add(Track(det[i], 0))
                      }
                  }

                  tracks.removeAll { it.age > maxAge }
                  return tracks.map { it.box }
              }

              fun reset() {
                  tracks.clear()
                  nextId = 1
              }
          }
          XEOF
          wc -l app/src/main/java/com/df/esp/Tracker.kt

      - name: "5 写 Detector.kt（ORT 推理 / letterbox / 解码 / NMS）"
        run: |
          set -e
          cat > app/src/main/java/com/df/esp/Detector.kt <<'XEOF'
          package com.df.esp

          import android.content.Context
          import android.graphics.Bitmap
          import ai.onnxruntime.OnnxTensor
          import ai.onnxruntime.OnnxTensorLike
          import ai.onnxruntime.OnnxValue
          import ai.onnxruntime.OrtEnvironment
          import ai.onnxruntime.OrtSession
          import ai.onnxruntime.TensorInfo
          import java.nio.FloatBuffer
          import kotlin.math.min

          class Detector(private val ctx: Context) {

              companion object {
                  val CANDIDATES = arrayOf(
                      "df_esp_int8.onnx", "df_esp.onnx", "yolov8n.onnx", "yolov8n_int8.onnx"
                  )
                  const val IN = 640
                  const val IOU = 0.45f
                  const val COCO_PERSON = 0

                  @Volatile
                  private var inst: Detector? = null

                  fun get(ctx: Context): Detector =
                      inst ?: synchronized(this) {
                          inst ?: Detector(ctx.applicationContext).also { inst = it }
                      }
              }

              @Volatile var ready = false
                  private set
              @Volatile var usingCpu = false
                  private set

              private var env: OrtEnvironment? = null
              private var sess: OrtSession? = null
              private var inputName = "images"
              private var modelName = ""

              private val tracker = Tracker()
              private val inputBuf = FloatArray(3 * IN * IN)
              private var lbSrc = IntArray(0)

              private var gScale = 1f
              private var gDx = 0f
              private var gDy = 0f

              /** 异步加载：绝不阻塞调用线程（V12 在 Service.onCreate 同步建 session 会 ANR） */
              fun loadAsync(onDone: ((Boolean) -> Unit)? = null) {
                  Thread {
                      EspState.assets = try {
                          (ctx.assets.list("") ?: emptyArray()).joinToString(",")
                      } catch (t: Throwable) { "?" }
                      var ok = false
                      for (name in CANDIDATES) {
                          val exist = try { ctx.assets.open(name).close(); true } catch (t: Throwable) { false }
                          if (!exist) continue
                          modelName = name
                          EspState.model = name
                          if (buildSession(true)) { ok = true; break }
                          if (buildSession(false)) { ok = true; break }
                      }
                      ready = ok
                      EspState.mode = if (ok) "就绪" else "无模型"
                      onDone?.invoke(ok)
                  }.apply { priority = Thread.NORM_PRIORITY - 1 }.start()
              }

              private fun buildSession(useNnapi: Boolean): Boolean {
                  try { sess?.close() } catch (t: Throwable) { }
                  sess = null
                  return try {
                      val e = env ?: OrtEnvironment.getEnvironment().also { env = it }
                      val o = OrtSession.SessionOptions()
                      o.setIntraOpNumThreads(4)
                      o.setOptimizationLevel(OrtSession.SessionOptions.OptLevel.ALL_OPT)
                      if (useNnapi) {
                          try { o.addNnapi() } catch (t: Throwable) { }
                      }
                      val bytes = ctx.assets.open(modelName).use { it.readBytes() }
                      val s = e.createSession(bytes, o)
                      sess = s
                      inputName = s.inputNames.first()
                      usingCpu = !useNnapi
                      true
                  } catch (t: Throwable) {
                      EspState.err = "建会话失败: " + t
                      false
                  }
              }

              private fun pickTensor(s: OrtSession, res: OrtSession.Result): OnnxTensor? {
                  try {
                      val v: Any? = res.get(0)
                      if (v is OnnxTensor) return v
                  } catch (t: Throwable) { }
                  try {
                      val name = s.outputNames.firstOrNull()
                      if (name != null) {
                          val v: Any? = res.get(name)
                          if (v is OnnxTensor) return v
                      }
                  } catch (t: Throwable) { }
                  try {
                      res.forEach { entry ->
                          val v: OnnxValue = entry.value
                          if (v is OnnxTensor) return v
                      }
                  } catch (t: Throwable) { }
                  return null
              }

              /** 手写 letterbox：源图 -> IN x IN，返回几何参数放在 gScale/gDx/gDy */
              private fun letterbox(bmp: Bitmap, out: FloatArray) {
                  val need = bmp.width * bmp.height
                  if (lbSrc.size < need) lbSrc = IntArray(need)
                  val src = lbSrc
                  bmp.getPixels(src, 0, bmp.width, 0, 0, bmp.width, bmp.height)

                  val scale = min(IN.toFloat() / bmp.width, IN.toFloat() / bmp.height)
                  val nw = (bmp.width * scale).toInt().coerceAtLeast(1)
                  val nh = (bmp.height * scale).toInt().coerceAtLeast(1)
                  val dx = (IN - nw) / 2
                  val dy = (IN - nh) / 2
                  gScale = scale
                  gDx = dx.toFloat()
                  gDy = dy.toFloat()

                  java.util.Arrays.fill(out, 114f / 255f)
                  val hw = IN * IN
                  val fx = bmp.width.toFloat() / nw
                  val fy = bmp.height.toFloat() / nh
                  for (y in 0 until nh) {
                      val sy = ((y + 0.5f) * fy).toInt().coerceIn(0, bmp.height - 1)
                      val rowOff = (y + dy) * IN + dx
                      val syOff = sy * bmp.width
                      for (x in 0 until nw) {
                          val sx = ((x + 0.5f) * fx).toInt().coerceIn(0, bmp.width - 1)
                          val p = src[syOff + sx]
                          val o = rowOff + x
                          out[o] = ((p shr 16) and 0xFF) / 255f
                          out[hw + o] = ((p shr 8) and 0xFF) / 255f
                          out[2 * hw + o] = (p and 0xFF) / 255f
                      }
                  }
              }

              fun detect(bmp: Bitmap): List<Box> {
                  if (sess == null) return emptyList()
                  return try {
                      tracker.update(nms(inferSmart(bmp)))
                  } catch (t: Throwable) {
                      EspState.err = "推理失败: " + t
                      if (!usingCpu && buildSession(false)) {
                          try {
                              tracker.update(nms(inferSmart(bmp)))
                          } catch (t2: Throwable) {
                              EspState.mode = "推理失败"
                              EspState.err = "CPU 也失败: " + t2
                              emptyList()
                          }
                      } else {
                          EspState.mode = "推理失败"
                          emptyList()
                      }
                  }
              }

              /** 横屏时左右切片各跑一次：等效输入放大 2x，远景小目标检出提升 */
              private fun inferSmart(bmp: Bitmap): List<Box> {
                  if (!EspState.tile || bmp.width < bmp.height) {
                      return inferRaw(bmp, 0, 0, bmp.width, bmp.height)
                  }
                  val half = bmp.width / 2
                  val a = ArrayList<Box>(64)
                  a.addAll(inferRaw(bmp, 0, 0, half, bmp.height))
                  a.addAll(inferRaw(bmp, half, 0, bmp.width - half, bmp.height))
                  return a
              }

              private fun inferRaw(bmp: Bitmap, tileX: Int, tileY: Int, tileW: Int, tileH: Int): List<Box> {
                  val e = env ?: return emptyList()
                  val s = sess ?: return emptyList()

                  val needTile = tileX != 0 || tileY != 0 || tileW != bmp.width || tileH != bmp.height
                  val tile = if (needTile) Bitmap.createBitmap(bmp, tileX, tileY, tileW, tileH) else bmp

                  var tn: OnnxTensor? = null
                  var res: OrtSession.Result? = null
                  try {
                      letterbox(tile, inputBuf)
                      val scale = gScale
                      val dx = gDx
                      val dy = gDy

                      val tensor = OnnxTensor.createTensor(
                          e, FloatBuffer.wrap(inputBuf),
                          longArrayOf(1, 3, IN.toLong(), IN.toLong())
                      )
                      tn = tensor
                      val feeds: Map<String, OnnxTensorLike> = mapOf(inputName to tensor)
                      val r = s.run(feeds)
                      res = r

                      val out = pickTensor(s, r) ?: run {
                          EspState.mode = "取不到输出"
                          return emptyList()
                      }
                      val dims = (out.info as TensorInfo).shape
                      if (dims.size < 3) {
                          EspState.mode = "输出维度异常"
                          return emptyList()
                      }

                      val d1 = dims[1].toInt()
                      val d2 = dims[2].toInt()
                      val chanFirst = d1 < d2
                      val chans = if (chanFirst) d1 else d2
                      val n = if (chanFirst) d2 else d1
                      val nc = chans - 4
                      if (nc <= 0) {
                          EspState.mode = "通道数异常"
                          return emptyList()
                      }

                      val fb = out.floatBuffer
                      val coco = nc == 80
                      val first = if (coco) COCO_PERSON else 0
                      val last = if (coco) COCO_PERSON else if (nc >= LABELS.size) nc - 1 else 0
                      val clsMode = when {
                          nc == LABELS.size -> "custom" + nc
                          coco -> "person"
                          nc == 1 -> "single"
                          else -> "generic" + nc
                      }
                      EspState.mode =
                          (if (EspState.tile && bmp.width >= bmp.height) "tile-" else "") +
                              (if (usingCpu) "cpu-" else "nnapi-") + clsMode
                      EspState.shape = "[" + d1 + "," + d2 + "]"
                      if (tileX == 0 && tileY == 0) EspState.err = ""

                      fun sc(c: Int, i: Int): Float =
                          if (chanFirst) fb.get(c * n + i) else fb.get(i * chans + c)

                      val conf = EspState.conf
                      val cand = ArrayList<Box>(48)
                      for (i in 0 until n) {
                          var best = -1
                          var bestS = conf
                          for (c in first..last) {
                              val v = sc(4 + c, i)
                              if (v > bestS) { bestS = v; best = c }
                          }
                          if (best < 0) continue
                          val cls = if (best < LABELS.size) best else 0
                          val cx = sc(0, i)
                          val cy = sc(1, i)
                          val w = sc(2, i)
                          val h = sc(3, i)
                          val bx1 = ((cx - w / 2) - dx) / scale + tileX
                          val by1 = ((cy - h / 2) - dy) / scale + tileY
                          val bx2 = ((cx + w / 2) - dx) / scale + tileX
                          val by2 = ((cy + h / 2) - dy) / scale + tileY
                          if (bx2 - bx1 < 4f || by2 - by1 < 6f) continue
                          cand.add(Box(cls, bestS, bx1, by1, bx2, by2))
                      }
                      return cand
                  } finally {
                      try { res?.close() } catch (t: Throwable) { }
                      try { tn?.close() } catch (t: Throwable) { }
                      if (needTile) {
                          try { tile.recycle() } catch (t: Throwable) { }
                      }
                  }
              }

              private fun nms(list: List<Box>): List<Box> {
                  val out = ArrayList<Box>(list.size)
                  for (c in LABELS.indices) {
                      val k = list.filter { it.cls == c }.sortedByDescending { it.score }
                      val keep = ArrayList<Box>()
                      for (b in k) if (keep.none { it.iou(b) > IOU }) keep.add(b)
                      out.addAll(keep)
                  }
                  return out
              }
          }
          XEOF
          wc -l app/src/main/java/com/df/esp/Detector.kt

      - name: "6 写 Radar.kt + Loot.kt（小地图红点雷达 / 颜色兜底）"
        run: |
          set -e
          cat > app/src/main/java/com/df/esp/Radar.kt <<'XEOF'
          package com.df.esp

          import android.graphics.Bitmap
          import kotlin.math.hypot
          import kotlin.math.max
          import kotlin.math.min

          class MiniMapRadar {

              fun scan(bmp: Bitmap): List<Blip> {
                  val w = bmp.width
                  val h = bmp.height
                  val r = (min(w, h) * EspState.mmRr).coerceAtLeast(6f)
                  val cx = EspState.mmCxR * w
                  val cy = EspState.mmCyR * h
                  val x0 = (cx - r).toInt().coerceAtLeast(0)
                  val y0 = (cy - r).toInt().coerceAtLeast(0)
                  val x1 = (cx + r).toInt().coerceAtMost(w)
                  val y1 = (cy + r).toInt().coerceAtMost(h)
                  val pw = x1 - x0
                  val ph = y1 - y0
                  if (pw < 6 || ph < 6) return emptyList()

                  val px = IntArray(pw * ph)
                  bmp.getPixels(px, 0, pw, x0, y0, pw, ph)
                  diagnose(px, pw * ph)

                  val cell = (r * 0.16f).coerceAtLeast(2f)
                  val grid = HashMap<Long, IntArray>()
                  for (y in 0 until ph) {
                      val row = y * pw
                      for (x in 0 until pw) {
                          if (!isEnemyDot(px[row + x])) continue
                          val k = ((x / cell).toLong() shl 20) or (y / cell).toLong()
                          val acc = grid[k]
                          if (acc == null) grid[k] = intArrayOf(x, y, 1)
                          else { acc[0] += x; acc[1] += y; acc[2]++ }
                      }
                  }

                  val out = ArrayList<Blip>(8)
                  for (acc in grid.values) {
                      if (acc[2] < 4) continue
                      if (acc[2] > r * r * 0.35f) continue
                      val gx = acc[0].toFloat() / acc[2]
                      val gy = acc[1].toFloat() / acc[2]
                      val dx = (gx - (cx - x0)) / r
                      val dy = (gy - (cy - y0)) / r
                      val len = hypot(dx, dy)
                      if (len > 1.05f) continue
                      if (len < 0.06f) continue
                      out.add(Blip(dx, dy, len * EspState.mmWorldR))
                  }
                  RadarDiag.clusters = out.size
                  return out
              }

              private fun diagnose(px: IntArray, n: Int) {
                  var red = 0
                  var rs = 0L
                  var gs = 0L
                  var bs = 0L
                  val hist = HashMap<Int, Int>()
                  val step = if (n > 40000) 3 else 1
                  var cnt = 0
                  var i = 0
                  while (i < n) {
                      val c = px[i]
                      val r = (c shr 16) and 0xFF
                      val g = (c shr 8) and 0xFF
                      val b = c and 0xFF
                      rs += r; gs += g; bs += b
                      cnt++
                      if (isEnemyDot(c)) red++
                      val q = ((r shr 3) shl 10) or ((g shr 3) shl 5) or (b shr 3)
                      hist[q] = (hist[q] ?: 0) + 1
                      i += step
                  }
                  RadarDiag.totalPix = cnt
                  RadarDiag.redPix = red
                  if (cnt > 0) {
                      RadarDiag.avgRgb = String.format(
                          "#%02X%02X%02X",
                          (rs / cnt).toInt(), (gs / cnt).toInt(), (bs / cnt).toInt()
                      )
                  }
                  RadarDiag.topColors = hist.entries
                      .sortedByDescending { it.value }
                      .take(3)
                      .joinToString(" ") { e ->
                          val q = e.key
                          String.format(
                              "#%02X%02X%02X(%d)",
                              ((q shr 10) and 31) shl 3,
                              ((q shr 5) and 31) shl 3,
                              (q and 31) shl 3,
                              e.value
                          )
                      }
              }

              /** 对准游戏画面点一下：全屏找红色像素最密集的方块，建议成小地图 ROI */
              fun autoLocate(bmp: Bitmap): String {
                  val w = bmp.width
                  val h = bmp.height
                  val step = 6
                  val sw = w / step
                  val sh = h / step
                  if (sw < 8 || sh < 8) return "画面太小"
                  val small = Bitmap.createScaledBitmap(bmp, sw, sh, false)
                  val px = IntArray(sw * sh)
                  small.getPixels(px, 0, sw, 0, 0, sw, sh)
                  small.recycle()

                  val cell = max(4, sw / 24)
                  val grid = HashMap<Long, Int>()
                  for (y in 0 until sh) {
                      val row = y * sw
                      for (x in 0 until sw) {
                          if (!isEnemyDot(px[row + x])) continue
                          val k = ((x / cell).toLong() shl 16) or (y / cell).toLong()
                          grid[k] = (grid[k] ?: 0) + 1
                      }
                  }
                  if (grid.isEmpty()) return "全屏没找到红色像素（小地图可能没有红点）"
                  val best = grid.entries.maxByOrNull { it.value } ?: return "无"
                  val k = best.key
                  val gx = ((k shr 16) and 0xFFFF).toInt() * cell + cell / 2
                  val gy = (k and 0xFFFF).toInt() * cell + cell / 2
                  val rRatio = (cell * 3f / min(sw, sh)).coerceIn(0.03f, 0.12f)
                  EspState.mmCxR = gx.toFloat() / sw
                  EspState.mmCyR = gy.toFloat() / sh
                  EspState.mmRr = rRatio
                  return String.format(
                      "建议 X=%.3f Y=%.3f R=%.3f（红点 %d）",
                      EspState.mmCxR, EspState.mmCyR, EspState.mmRr, best.value
                  )
              }

              private fun isEnemyDot(c: Int): Boolean {
                  val r = ((c shr 16) and 0xFF) / 255f
                  val g = ((c shr 8) and 0xFF) / 255f
                  val b = (c and 0xFF) / 255f
                  val mx = maxOf(r, g, b)
                  val mn = minOf(r, g, b)
                  if (mx < 0.45f) return false
                  val s = if (mx <= 0f) 0f else (mx - mn) / mx
                  if (s < 0.45f) return false
                  if (r < 0.45f) return false
                  if (g > 0.5f && b > 0.5f) return false
                  return r > g + 0.15f
              }
          }
          XEOF

          cat > app/src/main/java/com/df/esp/Loot.kt <<'XEOF'
          package com.df.esp

          import android.graphics.Bitmap

          /** HSV 连通域兜底：物资高亮/描边（默认关，模型靠谱时不需要它） */
          class LootScanner {
              var hMin = 35f
              var hMax = 62f
              var sMin = 0.18f
              var vMin = 0.62f
              var minPixels = 60

              private val dirs = arrayOf(
                  intArrayOf(1, 0), intArrayOf(-1, 0), intArrayOf(0, 1), intArrayOf(0, -1)
              )

              fun scan(bmp: Bitmap, step: Int = 4): List<Box> {
                  val sw = (bmp.width / step).coerceAtLeast(1)
                  val sh = (bmp.height / step).coerceAtLeast(1)
                  val small = Bitmap.createScaledBitmap(bmp, sw, sh, false)
                  val px = IntArray(sw * sh)
                  small.getPixels(px, 0, sw, 0, 0, sw, sh)
                  small.recycle()

                  val visited = BooleanArray(sw * sh)
                  val stack = IntArray(sw * sh)
                  val out = ArrayList<Box>(8)
                  for (i in px.indices) {
                      if (visited[i] || !hit(px[i])) continue
                      var top = 0
                      stack[top++] = i
                      visited[i] = true
                      var minX = sw
                      var maxX = 0
                      var minY = sh
                      var maxY = 0
                      var cnt = 0
                      while (top > 0) {
                          val q = stack[--top]
                          val x = q % sw
                          val y = q / sw
                          cnt++
                          if (x < minX) minX = x
                          if (x > maxX) maxX = x
                          if (y < minY) minY = y
                          if (y > maxY) maxY = y
                          for (d in dirs) {
                              val nx = x + d[0]
                              val ny = y + d[1]
                              if (nx < 0 || ny < 0 || nx >= sw || ny >= sh) continue
                              val n = ny * sw + nx
                              if (!visited[n] && hit(px[n])) {
                                  visited[n] = true
                                  stack[top++] = n
                              }
                          }
                      }
                      if (cnt < minPixels) continue
                      val bw = maxX - minX
                      val bh = maxY - minY
                      if (bw < 3 || bh < 3) continue
                      val ratio = bw.toFloat() / bh
                      if (ratio < 0.3f || ratio > 3.4f) continue
                      if (bw * bh > sw * sh / 6) continue
                      out.add(
                          Box(
                              4, 0.30f,
                              (minX * step).toFloat(), (minY * step).toFloat(),
                              ((maxX + 1) * step).toFloat(), ((maxY + 1) * step).toFloat()
                          )
                      )
                  }
                  return out
              }

              private fun hit(c: Int): Boolean {
                  val r = ((c shr 16) and 0xFF) / 255f
                  val g = ((c shr 8) and 0xFF) / 255f
                  val b = (c and 0xFF) / 255f
                  val mx = maxOf(r, g, b)
                  val mn = minOf(r, g, b)
                  if (mx < vMin) return false
                  val s = if (mx <= 0f) 0f else (mx - mn) / mx
                  if (s < sMin) return false
                  val d = mx - mn + 1e-6f
                  var h = when (mx) {
                      r -> 60f * ((g - b) / d)
                      g -> 60f * (2f + (b - r) / d)
                      else -> 60f * (4f + (r - g) / d)
                  }
                  if (h < 0) h += 360f
                  return h in hMin..hMax
              }
          }
          XEOF
          wc -l app/src/main/java/com/df/esp/Radar.kt app/src/main/java/com/df/esp/Loot.kt

      - name: "7 写 CaptureService.kt（采集 + 帧池 + 推理调度）"
        run: |
          set -e
          cat > app/src/main/java/com/df/esp/CaptureService.kt <<'XEOF'
          package com.df.esp

          import android.app.Notification
          import android.app.NotificationChannel
          import android.app.NotificationManager
          import android.app.PendingIntent
          import android.app.Service
          import android.content.Context
          import android.content.Intent
          import android.content.pm.ServiceInfo
          import android.graphics.Bitmap
          import android.graphics.PixelFormat
          import android.hardware.display.DisplayManager
          import android.hardware.display.VirtualDisplay
          import android.media.Image
          import android.media.ImageReader
          import android.media.projection.MediaProjection
          import android.media.projection.MediaProjectionManager
          import android.os.Build
          import android.os.Handler
          import android.os.HandlerThread
          import android.os.IBinder
          import android.util.DisplayMetrics
          import android.view.WindowManager
          import java.util.concurrent.ArrayBlockingQueue
          import java.util.concurrent.atomic.AtomicBoolean
          import kotlin.math.hypot
          import kotlin.math.max
          import kotlin.math.min

          class CaptureService : Service() {

              companion object {
                  @Volatile var alive = false
                      private set

                  const val EXTRA_CODE = "code"
                  const val EXTRA_DATA = "data"
                  const val ACTION_STOP = "com.df.esp.STOP"

                  private const val CH = "esp_cap"
                  private const val NID = 42

                  private var pendCode = 0
                  private var pendData: Intent? = null

                  fun start(ctx: Context, code: Int, data: Intent) {
                      pendCode = code
                      pendData = data
                      val i = Intent(ctx, CaptureService::class.java)
                          .putExtra(EXTRA_CODE, code)
                          .putExtra(EXTRA_DATA, data)
                      if (Build.VERSION.SDK_INT >= 26) ctx.startForegroundService(i)
                      else ctx.startService(i)
                  }

                  fun stop(ctx: Context) {
                      ctx.stopService(Intent(ctx, CaptureService::class.java))
                  }
              }

              private var projection: MediaProjection? = null
              private var vd: VirtualDisplay? = null
              private var reader: ImageReader? = null
              private var detector: Detector? = null
              private val lootScanner = LootScanner()
              private val radar = MiniMapRadar()

              /** 三槽 Bitmap 池：解码永不分配新内存；无空闲槽即丢帧（天然背压） */
              private var pool: Array<Bitmap?> = arrayOfNulls(3)
              private val poolLock = Object()

              private val frames = ArrayBlockingQueue<Bitmap>(1)
              private val running = AtomicBoolean(false)
              private val ht = HandlerThread("esp-cap").apply { start() }
              private val h by lazy { Handler(ht.looper) }

              private var rowBuf = ByteArray(0)
              private var pxBuf = IntArray(0)

              private var fpsCnt = 0
              private var fpsT = 0L

              private var lastBlips: List<Blip> = emptyList()
              private var blipMiss = 0

              private val disp by lazy { getSystemService(DisplayManager::class.java) }
              private val dispListener = object : DisplayManager.DisplayListener {
                  override fun onDisplayAdded(displayId: Int) {}
                  override fun onDisplayRemoved(displayId: Int) {}
                  override fun onDisplayChanged(displayId: Int) {
                      h.post { reconfigure() }
                  }
              }

              override fun onBind(i: Intent?): IBinder? = null

              override fun onCreate() {
                  super.onCreate()
                  fg()
                  val d = Detector.get(this)
                  detector = d
                  d.loadAsync()          // 工作线程里建 session，主线程零阻塞
              }

              private fun fg() {
                  val nm = getSystemService(NotificationManager::class.java)
                  if (Build.VERSION.SDK_INT >= 26 && nm.getNotificationChannel(CH) == null) {
                      nm.createNotificationChannel(
                          NotificationChannel(CH, "采集", NotificationManager.IMPORTANCE_MIN)
                      )
                  }
                  val stopPi = PendingIntent.getService(
                      this, 11,
                      Intent(this, CaptureService::class.java).setAction(ACTION_STOP),
                      PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
                  )
                  val b = if (Build.VERSION.SDK_INT >= 26) Notification.Builder(this, CH)
                  else Notification.Builder(this)
                  val n = b.setContentTitle("屏幕服务")
                      .setContentText("运行中")
                      .setSmallIcon(android.R.drawable.ic_menu_camera)
                      .setOngoing(true)
                      .addAction(0, "停止", stopPi)
                      .build()
                  if (Build.VERSION.SDK_INT >= 29) {
                      startForeground(NID, n, ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PROJECTION)
                  } else {
                      startForeground(NID, n)
                  }
              }

              override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
                  if (intent?.action == ACTION_STOP) {
                      stopSelf()
                      return START_NOT_STICKY
                  }
                  val code = intent?.getIntExtra(EXTRA_CODE, 0) ?: pendCode
                  val data: Intent? = extraData(intent) ?: pendData
                  if (data == null) {
                      EspState.projErr = "没拿到投屏授权数据"
                      return START_STICKY
                  }
                  if (!running.get()) {
                      try {
                          begin(code, data)
                      } catch (t: Throwable) {
                          EspState.projErr = "启动采集失败: " + t
                      }
                  }
                  return START_STICKY
              }

              private fun extraData(intent: Intent?): Intent? {
                  if (intent == null) return null
                  return if (Build.VERSION.SDK_INT >= 33) {
                      intent.getParcelableExtra(EXTRA_DATA, Intent::class.java)
                  } else {
                      @Suppress("DEPRECATION")
                      intent.getParcelableExtra(EXTRA_DATA)
                  }
              }

              /** 长边等比缩放到 1280 —— 保持宽高比，绝不各自截断（V12 的 bug） */
              @Suppress("DEPRECATION")
              private fun metrics(): Triple<Int, Int, Int> {
                  val wm = getSystemService(WindowManager::class.java)
                  val dm = DisplayMetrics()
                  wm.defaultDisplay.getRealMetrics(dm)
                  val sw = dm.widthPixels
                  val sh = dm.heightPixels
                  val longSide = max(sw, sh)
                  val maxLong = 1280
                  val k = if (longSide > maxLong) maxLong.toFloat() / longSide else 1f
                  var cw = (sw * k).toInt() and 0x7FFFFFFE
                  var chh = (sh * k).toInt() and 0x7FFFFFFE
                  if (cw < 2) cw = 2
                  if (chh < 2) chh = 2
                  return Triple(cw, chh, dm.densityDpi)
              }

              private fun begin(code: Int, data: Intent) {
                  val mpm = getSystemService(MediaProjectionManager::class.java)
                  projection = mpm.getMediaProjection(code, data)
                  if (projection == null) {
                      EspState.projErr = "getMediaProjection 返回 null"
                      return
                  }
                  EspState.projErr = ""
                  projection?.registerCallback(object : MediaProjection.Callback() {
                      override fun onStop() {
                          running.set(false)
                          stopSelf()
                      }
                  }, h)

                  val m = metrics()
                  EspState.capW = m.first
                  EspState.capH = m.second
                  buildDisplay(m.first, m.second, m.third)
                  running.set(true)
                  alive = true
                  try { disp.registerDisplayListener(dispListener, h) } catch (t: Throwable) { }
                  h.post(loop)
              }

              private fun buildDisplay(cw: Int, chh: Int, dpi: Int) {
                  val p = projection ?: return
                  reader = ImageReader.newInstance(cw, chh, PixelFormat.RGBA_8888, 3)
                  reader!!.setOnImageAvailableListener({ r -> grab(r, cw, chh) }, h)
                  vd = p.createVirtualDisplay(
                      "esp", cw, chh, dpi,
                      DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR,
                      reader!!.surface, null, h
                  )
              }

              private fun reconfigure() {
                  if (!running.get()) return
                  val m = metrics()
                  if (m.first == EspState.capW && m.second == EspState.capH) return
                  try { reader?.close() } catch (t: Throwable) { }
                  try { vd?.release() } catch (t: Throwable) { }
                  synchronized(poolLock) {
                      for (i in pool.indices) {
                          try { pool[i]?.recycle() } catch (t: Throwable) { }
                          pool[i] = null
                      }
                  }
                  frames.clear()
                  EspState.capW = m.first
                  EspState.capH = m.second
                  buildDisplay(m.first, m.second, m.third)
              }

              private fun acquire(w: Int, hh: Int): Bitmap? {
                  synchronized(poolLock) {
                      for (i in pool.indices) {
                          val b = pool[i]
                          if (b != null && !b.isRecycled && b.width == w && b.height == hh) {
                              pool[i] = null
                              return b
                          }
                      }
                  }
                  return Bitmap.createBitmap(w, hh, Bitmap.Config.ARGB_8888)
              }

              private fun recycle(b: Bitmap) {
                  synchronized(poolLock) {
                      if (!b.isRecycled) {
                          for (i in pool.indices) {
                              if (pool[i] == null) {
                                  pool[i] = b
                                  return
                              }
                          }
                      }
                  }
                  if (!b.isRecycled) b.recycle()
              }

              private fun grab(r: ImageReader, cw: Int, chh: Int) {
                  val img = try { r.acquireLatestImage() } catch (t: Throwable) {
                      if (EspState.capErr.isEmpty()) EspState.capErr = "取帧失败: " + t
                      null
                  }
                  if (img == null) return

                  var handedOff = false
                  val dst = acquire(cw, chh)
                  try {
                      if (dst == null) return          // 池里没空槽 → 直接丢这一帧
                      decode(img, dst, cw, chh)
                      EspState.frames++
                      frames.poll()?.let { recycle(it) }
                      if (frames.offer(dst)) handedOff = true
                  } catch (t: Throwable) {
                      if (EspState.capErr.isEmpty()) EspState.capErr = "解码失败: " + t
                  } finally {
                      try { img.close() } catch (t: Throwable) { }
                      if (!handedOff && dst != null) recycle(dst)
                  }
              }

              private fun decode(img: Image, dst: Bitmap, w: Int, hh: Int) {
                  val p = img.planes[0]
                  val b = p.buffer
                  val ps = p.pixelStride
                  val rs = p.rowStride
                  if (rs == w * ps) {
                      b.rewind()
                      dst.copyPixelsFromBuffer(b)
                      return
                  }
                  if (rowBuf.size != rs) rowBuf = ByteArray(rs)
                  if (pxBuf.size != w) pxBuf = IntArray(w)
                  val rb = rowBuf
                  val pb = pxBuf
                  for (y in 0 until hh) {
                      b.position(y * rs)
                      b.get(rb, 0, rs)
                      for (x in 0 until w) {
                          val o = x * ps
                          pb[x] = (0xFF shl 24) or
                              ((rb[o].toInt() and 0xFF) shl 16) or
                              ((rb[o + 1].toInt() and 0xFF) shl 8) or
                              (rb[o + 2].toInt() and 0xFF)
                      }
                      dst.setPixels(pb, 0, w, 0, y, w, 1)
                  }
              }

              private fun smoothBlips(cur: List<Blip>): List<Blip> {
                  if (cur.isEmpty()) {
                      blipMiss++
                      if (blipMiss <= 15) return lastBlips
                      lastBlips = emptyList()
                      return emptyList()
                  }
                  blipMiss = 0
                  val usedM = BooleanArray(lastBlips.size)
                  val out = ArrayList<Blip>(cur.size)
                  for (c in cur) {
                      var bi = -1
                      var bd = 0.12f
                      for (i in lastBlips.indices) {
                          if (usedM[i]) continue
                          val m = lastBlips[i]
                          val dd = hypot(m.dx - c.dx, m.dy - c.dy)
                          if (dd < bd) { bd = dd; bi = i }
                      }
                      if (bi >= 0) {
                          usedM[bi] = true
                          val m = lastBlips[bi]
                          out.add(
                              Blip(
                                  m.dx + (c.dx - m.dx) * 0.45f,
                                  m.dy + (c.dy - m.dy) * 0.45f,
                                  m.dist + (c.dist - m.dist) * 0.45f
                              )
                          )
                      } else {
                          out.add(c)
                      }
                  }
                  lastBlips = out
                  return out
              }

              private var lootTick = 0

              private val loop = object : Runnable {
                  override fun run() {
                      if (!running.get()) return
                      val b = frames.poll()
                      try {
                          if (b != null) {
                              val t0 = System.nanoTime()
                              val det = detector?.detect(b) ?: emptyList()

                              var extra: List<Box> = emptyList()
                              if (EspState.loot) {
                                  lootTick++
                                  if (lootTick % 2 == 0) extra = lootScanner.scan(b)
                              }
                              val merged = ArrayList<Box>(det.size + extra.size)
                              merged.addAll(det)
                              for (x in extra) if (merged.none { it.iou(x) > 0.3f }) merged.add(x)

                              val blips =
                                  if (EspState.mmOn) smoothBlips(radar.scan(b)) else emptyList()

                              if (EspState.pendingAutoLocate) {
                                  RadarDiag.suggest = radar.autoLocate(b)
                                  EspState.pendingAutoLocate = false
                              }

                              EspState.push(merged, blips)
                              EspState.inferMs = (System.nanoTime() - t0) / 1e6f

                              fpsCnt++
                              val now = System.nanoTime()
                              if (fpsT == 0L) fpsT = now
                              if (now - fpsT > 500000000L) {
                                  EspState.capFps = fpsCnt * 1e9f / (now - fpsT)
                                  fpsCnt = 0
                                  fpsT = now
                              }
                          }
                      } catch (t: Throwable) {
                          EspState.err = "主循环: " + t
                      } finally {
                          if (b != null) recycle(b)
                      }
                      h.postDelayed(this, 8)
                  }
              }

              override fun onDestroy() {
                  running.set(false)
                  alive = false
                  try { disp.unregisterDisplayListener(dispListener) } catch (t: Throwable) { }
                  h.removeCallbacksAndMessages(null)
                  try { reader?.close() } catch (t: Throwable) { }
                  try { vd?.release() } catch (t: Throwable) { }
                  try { projection?.stop() } catch (t: Throwable) { }
                  frames.clear()
                  synchronized(poolLock) {
                      for (i in pool.indices) {
                          try { pool[i]?.recycle() } catch (t: Throwable) { }
                          pool[i] = null
                      }
                  }
                  ht.quitSafely()
                  stopForeground(true)
                  super.onDestroy()
              }
          }
          XEOF
          wc -l app/src/main/java/com/df/esp/CaptureService.kt

      - name: "8 写 Overlay.kt（悬浮窗绘制 / 气泡 / 无障碍宿主）"
        run: |
          set -e
          cat > app/src/main/java/com/df/esp/Overlay.kt <<'XEOF'
          package com.df.esp

          import android.accessibilityservice.AccessibilityService
          import android.app.Notification
          import android.app.NotificationChannel
          import android.app.NotificationManager
          import android.app.Service
          import android.content.Context
          import android.content.Intent
          import android.content.pm.ServiceInfo
          import android.graphics.Canvas
          import android.graphics.Color
          import android.graphics.DashPathEffect
          import android.graphics.Paint
          import android.graphics.Path
          import android.graphics.PixelFormat
          import android.graphics.drawable.GradientDrawable
          import android.os.Build
          import android.os.IBinder
          import android.os.SystemClock
          import android.view.Gravity
          import android.view.MotionEvent
          import android.view.View
          import android.view.WindowManager
          import android.view.accessibility.AccessibilityEvent
          import android.widget.Toast
          import kotlin.math.hypot
          import kotlin.math.min

          class EspOverlayView(ctx: Context) : View(ctx) {

              private val stroke = Paint(Paint.ANTI_ALIAS_FLAG)
              private val fill = Paint(Paint.ANTI_ALIAS_FLAG)
              private val txt = Paint(Paint.ANTI_ALIAS_FLAG)
              private val small = Paint(Paint.ANTI_ALIAS_FLAG)
              private val bg = Paint()
              private val roi = Paint(Paint.ANTI_ALIAS_FLAG)
              private val d = resources.displayMetrics.density

              private val colors = intArrayOf(
                  Color.rgb(255, 60, 60), Color.rgb(255, 205, 40), Color.rgb(70, 200, 255),
                  Color.rgb(175, 100, 255), Color.rgb(60, 255, 140), Color.rgb(255, 235, 90)
              )

              private var lastVer = -1
              private var lastDraw = 0L

              /**
               * 数据驱动重绘：只有检测结果变化（ver 变）或超过 250ms 才 invalidate。
               * 这修掉了 V12 每 16ms 全屏重绘 → 和屏幕镜像叠加把合成器打满的问题。
               */
              private val ticker = object : Runnable {
                  override fun run() {
                      val now = SystemClock.uptimeMillis()
                      if (EspState.ver != lastVer || now - lastDraw > 250L) {
                          lastVer = EspState.ver
                          lastDraw = now
                          invalidate()
                      }
                      postDelayed(this, 80L)
                  }
              }

              init {
                  stroke.style = Paint.Style.STROKE
                  stroke.strokeWidth = 2.5f * d
                  fill.style = Paint.Style.FILL
                  txt.color = Color.WHITE
                  txt.textSize = 11f * d
                  txt.isFakeBoldText = true
                  small.color = Color.rgb(255, 210, 120)
                  small.textSize = 10f * d
                  bg.color = Color.argb(160, 0, 0, 0)
                  roi.style = Paint.Style.STROKE
                  roi.strokeWidth = 2f * d
                  roi.color = Color.argb(200, 255, 220, 60)
                  roi.pathEffect = DashPathEffect(floatArrayOf(8f * d, 6f * d), 0f)
                  setBackgroundColor(Color.TRANSPARENT)
              }

              override fun onAttachedToWindow() {
                  super.onAttachedToWindow()
                  postDelayed(ticker, 80L)
              }

              override fun onDetachedFromWindow() {
                  removeCallbacks(ticker)
                  super.onDetachedFromWindow()
              }

              override fun onDraw(c: Canvas) {
                  if (!EspState.enabled) return
                  drawBoxes(c)
                  drawRoi(c)
                  if (EspState.mmOn) drawRadar(c)
                  drawHud(c)
              }

              private fun drawBoxes(c: Canvas) {
                  val cw = EspState.capW
                  val chh = EspState.capH
                  if (cw <= 0 || chh <= 0 || width <= 0 || height <= 0) return
                  val sx = width.toFloat() / cw
                  val sy = height.toFloat() / chh
                  val list = EspState.boxes
                  for (b in list) {
                      if (!EspState.showLoot && (b.cls == 4 || b.cls == 3)) continue
                      if (EspState.hideStatic && b.frames > 20 && hypot(b.vx, b.vy) < 0.4f) continue

                      stroke.color = colors.getOrElse(b.cls) { Color.WHITE }
                      c.drawRect(b.x1 * sx, b.y1 * sy, b.x2 * sx, b.y2 * sy, stroke)

                      val label: String = if (b.cls <= 1) {
                          val dist = if (b.h > 4f) EspState.K_DIST / b.h else 0f
                          String.format("%.0fm %.2f", dist, b.score)
                      } else {
                          LABELS.getOrElse(b.cls) { "?" }
                      }
                      val tw = txt.measureText(label)
                      c.drawRect(
                          b.x1 * sx, b.y1 * sy - 12f * d,
                          b.x1 * sx + tw + 5f * d, b.y1 * sy, bg
                      )
                      c.drawText(label, b.x1 * sx + 2.5f * d, b.y1 * sy - 3f * d, txt)

                      val cxv = b.cx * sx
                      c.drawLine(cxv - 6f * d, b.y1 * sy, cxv + 6f * d, b.y1 * sy, stroke)
                  }
              }

              private fun drawRoi(c: Canvas) {
                  if (!EspState.mmShowRoi) return
                  if (width <= 0 || height <= 0) return
                  val cx = EspState.mmCxR * width
                  val cy = EspState.mmCyR * height
                  val r = min(width, height) * EspState.mmRr
                  c.drawCircle(cx, cy, r, roi)
                  c.drawLine(cx - r * 0.2f, cy, cx + r * 0.2f, cy, roi)
                  c.drawLine(cx, cy - r * 0.2f, cx, cy + r * 0.2f, roi)
                  val s = "红" + RadarDiag.redPix + "/" + RadarDiag.totalPix + " 点" + RadarDiag.clusters
                  small.color = Color.rgb(255, 220, 60)
                  c.drawRect(
                      cx - r, cy + r + 2f * d,
                      cx - r + small.measureText(s) + 8f * d, cy + r + 16f * d, bg
                  )
                  c.drawText(s, cx - r + 4f * d, cy + r + 12f * d, small)
              }

              private fun drawRadar(c: Canvas) {
                  if (height <= 0) return
                  val blips = EspState.blips
                  val cr = 44f * d
                  val ccx = cr + 14f * d
                  val ccy = height - cr - 14f * d

                  stroke.color = Color.argb(110, 220, 220, 220)
                  c.drawCircle(ccx, ccy, cr, stroke)
                  stroke.color = Color.argb(60, 220, 220, 220)
                  c.drawCircle(ccx, ccy, cr * 0.55f, stroke)
                  c.drawLine(ccx - cr, ccy, ccx + cr, ccy, stroke)
                  c.drawLine(ccx, ccy - cr, ccx, ccy + cr, stroke)

                  for (b in blips) {
                      val px = ccx + b.dx * cr
                      val py = ccy + b.dy * cr
                      fill.color = if (b.dist < 30f) Color.rgb(255, 60, 60) else Color.rgb(255, 140, 60)
                      c.drawCircle(px, py, 2.6f * d, fill)
                  }
                  fill.color = Color.WHITE
                  c.drawCircle(ccx, ccy, 2.4f * d, fill)
                  val cnt = String.format("%d", blips.size)
                  c.drawText(cnt, ccx - txt.measureText(cnt) / 2, ccy - cr - 3f * d, txt)

                  if (EspState.mmArrows) drawOffscreenArrows(c, blips)
              }

              private fun drawOffscreenArrows(c: Canvas, blips: List<Blip>) {
                  if (blips.isEmpty()) return
                  val cxm = width / 2f
                  val cym = height / 2f
                  val p = Path()
                  for (b in blips.sortedBy { it.dist }.take(6)) {
                      val len = hypot(b.dx, b.dy)
                      if (len < 0.02f) continue
                      val ux = b.dx / len
                      val uy = b.dy / len
                      val tx = if (ux > 0.0001f) (width - cxm) / ux
                      else if (ux < -0.0001f) -cxm / ux else 1e9f
                      val ty = if (uy > 0.0001f) (height - cym) / uy
                      else if (uy < -0.0001f) -cym / uy else 1e9f
                      val tt = min(tx, ty) * 0.93f
                      if (tt <= 0f || tt > 1e8f) continue
                      val ex = cxm + ux * tt
                      val ey = cym + uy * tt
                      val s = 7f * d
                      val nx = -uy
                      val ny = ux
                      fill.color = Color.argb(210, 255, 70, 70)
                      p.reset()
                      p.moveTo(ex + ux * s, ey + uy * s)
                      p.lineTo(ex - ux * s + nx * s * 0.8f, ey - uy * s + ny * s * 0.8f)
                      p.lineTo(ex - ux * s - nx * s * 0.8f, ey - uy * s - ny * s * 0.8f)
                      p.close()
                      c.drawPath(p, fill)
                  }
              }

              private fun drawHud(c: Canvas) {
                  val st = String.format(
                      "%s %s f%d b%d %dx%d c%.2f",
                      VER, EspState.mode, EspState.frames, EspState.boxes.size,
                      EspState.capW, EspState.capH, EspState.conf
                  )
                  c.drawRect(0f, 0f, txt.measureText(st) + 10f * d, 18f * d, bg)
                  c.drawText(st, 5f * d, 13f * d, txt)

                  val e = when {
                      EspState.model.isEmpty() -> "无模型 assets=[" + EspState.assets + "]"
                      EspState.projErr.isNotEmpty() -> "投屏: " + EspState.projErr
                      EspState.capErr.isNotEmpty() -> "取帧: " + EspState.capErr
                      EspState.err.isNotEmpty() -> "错误: " + EspState.err
                      else -> ""
                  }
                  if (e.isNotEmpty()) {
                      val t = e.take(72)
                      c.drawRect(0f, 19f * d, small.measureText(t) + 10f * d, 34f * d, bg)
                      c.drawText(t, 5f * d, 30f * d, small)
                  }
              }
          }

          class OverlayService : Service() {

              companion object {
                  @Volatile var alive = false
                      private set
                  private const val CH = "esp_ov"
                  private const val NID = 41
              }

              private lateinit var wm: WindowManager
              private var bubble: View? = null
              private var hud: EspOverlayView? = null
              private var lastSecure = true

              override fun onCreate() {
                  super.onCreate()
                  fg()
                  alive = true
                  wm = getSystemService(WindowManager::class.java)
                  addBubble()
                  attachHud()
                  startWatchdog()
              }

              private fun fg() {
                  val nm = getSystemService(NotificationManager::class.java)
                  if (Build.VERSION.SDK_INT >= 26 && nm.getNotificationChannel(CH) == null) {
                      nm.createNotificationChannel(
                          NotificationChannel(CH, "显示", NotificationManager.IMPORTANCE_MIN)
                      )
                  }
                  val b = if (Build.VERSION.SDK_INT >= 26) Notification.Builder(this, CH)
                  else Notification.Builder(this)
                  val n = b.setContentTitle("显示服务")
                      .setContentText("运行中")
                      .setSmallIcon(android.R.drawable.ic_menu_view)
                      .setOngoing(true)
                      .build()
                  if (Build.VERSION.SDK_INT >= 34) {
                      startForeground(NID, n, ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE)
                  } else {
                      startForeground(NID, n)
                  }
              }

              /** 无障碍连上了就用它的浮层（游戏「隐藏悬浮窗」屏蔽不掉），否则自己开 OVERLAY */
              private fun attachHud() {
                  if (hud != null) return
                  if (EspAccessibilityService.connected) return
                  val flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
                      WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE or
                      WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL or
                      WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN or
                      WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED or
                      (if (EspState.secure) WindowManager.LayoutParams.FLAG_SECURE else 0)
                  val p = WindowManager.LayoutParams(
                      WindowManager.LayoutParams.MATCH_PARENT,
                      WindowManager.LayoutParams.MATCH_PARENT,
                      WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
                      flags,
                      PixelFormat.TRANSLUCENT
                  )
                  p.gravity = Gravity.TOP or Gravity.START
                  p.title = "hud_layer"
                  val v = EspOverlayView(this)
                  try {
                      wm.addView(v, p)
                      hud = v
                      lastSecure = EspState.secure
                  } catch (t: Throwable) {
                      hud = null
                      Toast.makeText(this, "浮层添加失败: " + t, Toast.LENGTH_LONG).show()
                  }
              }

              private fun detachHud() {
                  hud?.let { try { wm.removeView(it) } catch (t: Throwable) { } }
                  hud = null
              }

              private fun startWatchdog() {
                  val h = android.os.Handler(mainLooper)
                  h.post(object : Runnable {
                      override fun run() {
                          val need = !EspAccessibilityService.connected
                          if (need && hud == null) attachHud()
                          if (!need && hud != null) detachHud()
                          if (hud != null && lastSecure != EspState.secure) {
                              detachHud()
                              attachHud()
                          }
                          h.postDelayed(this, 700)
                      }
                  })
              }

              private fun addBubble() {
                  val d = resources.displayMetrics.density
                  val size = (52 * d).toInt()
                  val v = View(this)
                  v.background = GradientDrawable().apply {
                      shape = GradientDrawable.OVAL
                      setColor(0xAA1B1B1B.toInt())
                      setStroke((2 * d).toInt(), 0xFF39D98A.toInt())
                  }
                  val p = WindowManager.LayoutParams(
                      size, size,
                      WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
                      WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
                      PixelFormat.TRANSLUCENT
                  )
                  p.gravity = Gravity.TOP or Gravity.START
                  p.x = 24
                  p.y = 260
                  p.title = "hud_bubble"

                  var dx = 0
                  var dy = 0
                  var downX = 0f
                  var downY = 0f
                  var moved = false
                  var lastTap = 0L

                  v.setOnTouchListener { _, e ->
                      when (e.actionMasked) {
                          MotionEvent.ACTION_DOWN -> {
                              dx = p.x; dy = p.y
                              downX = e.rawX; downY = e.rawY
                              moved = false
                          }
                          MotionEvent.ACTION_MOVE -> {
                              val mx = (e.rawX - downX).toInt()
                              val my = (e.rawY - downY).toInt()
                              if (Math.abs(mx) + Math.abs(my) > 14) moved = true
                              p.x = dx + mx
                              p.y = dy + my
                              try { wm.updateViewLayout(v, p) } catch (t: Throwable) { }
                          }
                          MotionEvent.ACTION_UP -> {
                              if (!moved) {
                                  val now = SystemClock.uptimeMillis()
                                  if (now - lastTap < 320) {
                                      EspState.conf = when {
                                          EspState.conf > 0.24f -> 0.18f
                                          EspState.conf > 0.15f -> 0.12f
                                          else -> 0.30f
                                      }
                                      Toast.makeText(
                                          this, "阈值 " + String.format("%.2f", EspState.conf),
                                          Toast.LENGTH_SHORT
                                      ).show()
                                      lastTap = 0L
                                  } else {
                                      EspState.enabled = !EspState.enabled
                                      Toast.makeText(
                                          this,
                                          if (EspState.enabled) "ESP 开" else "ESP 关",
                                          Toast.LENGTH_SHORT
                                      ).show()
                                      lastTap = now
                                  }
                              }
                          }
                      }
                      true
                  }

                  try {
                      wm.addView(v, p)
                      bubble = v
                  } catch (t: Throwable) {
                      Toast.makeText(this, "气泡添加失败: " + t, Toast.LENGTH_LONG).show()
                  }
              }

              override fun onDestroy() {
                  alive = false
                  bubble?.let { try { wm.removeView(it) } catch (t: Throwable) { } }
                  bubble = null
                  detachHud()
                  super.onDestroy()
              }

              override fun onBind(i: Intent?): IBinder? = null
          }

          class EspAccessibilityService : AccessibilityService() {

              companion object {
                  @Volatile var connected = false
                      private set
              }

              private var view: EspOverlayView? = null

              override fun onServiceConnected() {
                  super.onServiceConnected()
                  connected = true
                  attach()
              }

              private fun attach() {
                  if (view != null) return
                  val wm = getSystemService(WindowManager::class.java)
                  val flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
                      WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE or
                      WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL or
                      WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN or
                      WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED or
                      (if (EspState.secure) WindowManager.LayoutParams.FLAG_SECURE else 0)
                  val p = WindowManager.LayoutParams(
                      WindowManager.LayoutParams.MATCH_PARENT,
                      WindowManager.LayoutParams.MATCH_PARENT,
                      WindowManager.LayoutParams.TYPE_ACCESSIBILITY_OVERLAY,
                      flags,
                      PixelFormat.TRANSLUCENT
                  )
                  p.gravity = Gravity.TOP or Gravity.START
                  val v = EspOverlayView(this)
                  try {
                      wm.addView(v, p)
                      view = v
                  } catch (t: Throwable) {
                      view = null
                  }
              }

              private fun detach() {
                  val wm = getSystemService(WindowManager::class.java)
                  view?.let { try { wm.removeView(it) } catch (t: Throwable) { } }
                  view = null
              }

              override fun onAccessibilityEvent(event: AccessibilityEvent?) = Unit

              override fun onInterrupt() = Unit

              override fun onUnbind(intent: Intent?): Boolean {
                  connected = false
                  detach()
                  return super.onUnbind(intent)
              }

              override fun onDestroy() {
                  connected = false
                  detach()
                  super.onDestroy()
              }
          }
          XEOF
          wc -l app/src/main/java/com/df/esp/Overlay.kt

      - name: "9 写 MainActivity.kt（授权引导 / 调参控制台）"
        run: |
          set -e
          cat > app/src/main/java/com/df/esp/MainActivity.kt <<'XEOF'
          package com.df.esp

          import android.app.Activity
          import android.content.Intent
          import android.media.projection.MediaProjectionManager
          import android.net.Uri
          import android.os.Build
          import android.os.Bundle
          import android.provider.Settings
          import android.widget.Button
          import android.widget.LinearLayout
          import android.widget.ScrollView
          import android.widget.SeekBar
          import android.widget.TextView
          import android.widget.Toast

          class MainActivity : Activity() {

              private val REQ_CAP = 1001

              private lateinit var status: TextView
              private lateinit var diag: TextView
              private lateinit var sliders: LinearLayout

              override fun onCreate(s: Bundle?) {
                  super.onCreate(s)

                  val root = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL
                      setPadding(36, 92, 36, 36)
                  }
                  status = TextView(this).apply {
                      textSize = 12f
                      setPadding(0, 0, 0, 10)
                  }
                  diag = TextView(this).apply {
                      textSize = 11f
                      setTextColor(0xFF88FF99.toInt())
                      setPadding(0, 0, 0, 16)
                  }
                  root.addView(status)
                  root.addView(diag)

                  root.addView(btn("1. 授予悬浮窗权限") {
                      if (!Settings.canDrawOverlays(this)) {
                          startActivity(
                              Intent(
                                  Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                                  Uri.parse("package:" + packageName)
                              )
                          )
                      } else toast("已授权")
                      refresh()
                  })

                  root.addView(btn("2. 开无障碍（可选，防被游戏屏蔽浮窗）") {
                      toast("找到「设备服务」并打开")
                      try {
                          startActivity(Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS))
                      } catch (t: Throwable) {
                          toast("打不开: " + t)
                      }
                  })

                  root.addView(btn("3. 启动 / 关闭 气泡 + 浮层") {
                      if (!Settings.canDrawOverlays(this)) {
                          toast("先给悬浮窗权限")
                          return@btn
                      }
                      if (OverlayService.alive) {
                          stopService(Intent(this, OverlayService::class.java))
                      } else {
                          val i = Intent(this, OverlayService::class.java)
                          if (Build.VERSION.SDK_INT >= 26) startForegroundService(i)
                          else startService(i)
                      }
                      reload()
                  })

                  root.addView(btn("4. 开始透视（采集屏幕）") {
                      val mpm = getSystemService(MediaProjectionManager::class.java)
                      startActivityForResult(mpm.createScreenCaptureIntent(), REQ_CAP)
                  })

                  root.addView(btn("5. 停止采集") {
                      CaptureService.stop(this)
                      reload()
                  })

                  root.addView(btn("6. 自动定位小地图（对准游戏画面后点）") {
                      EspState.pendingAutoLocate = true
                      toast("下一帧全屏扫红色像素，看下方「定位」结果")
                      reload()
                  })

                  root.addView(btn("7. 切片推理 开 / 关") {
                      EspState.tile = !EspState.tile
                      toast(if (EspState.tile) "切片开（慢一倍、检出更高）" else "切片关（更快）")
                      refresh()
                  })

                  root.addView(btn("8. 雷达 开 / 关") {
                      EspState.mmOn = !EspState.mmOn
                      toast(if (EspState.mmOn) "雷达开" else "雷达关")
                      refresh()
                  })

                  root.addView(btn("9. 隐藏静止目标 开 / 关") {
                      EspState.hideStatic = !EspState.hideStatic
                      toast(
                          if (EspState.hideStatic) "只画会动的（蹲点真人也会被隐）"
                          else "全部绘制"
                      )
                      refresh()
                  })

                  root.addView(btn("10. 显示 / 隐藏 ROI 黄圈") {
                      EspState.mmShowRoi = !EspState.mmShowRoi
                      refresh()
                  })

                  root.addView(btn("11. 循环置信度（也可双击气泡）") {
                      EspState.conf = when {
                          EspState.conf > 0.24f -> 0.18f
                          EspState.conf > 0.15f -> 0.12f
                          else -> 0.30f
                      }
                      toast("阈值 " + String.format("%.2f", EspState.conf))
                      refresh()
                  })

                  root.addView(btn("12. 防自反馈 FLAG_SECURE 开 / 关") {
                      EspState.secure = !EspState.secure
                      toast(
                          if (EspState.secure) "开：浮层不被自己采集（推荐）"
                          else "关：若开时整屏黑屏就用这个"
                      )
                      refresh()
                  })

                  root.addView(btn("13. 物资颜色扫描 开 / 关") {
                      EspState.loot = !EspState.loot
                      EspState.showLoot = EspState.loot
                      toast(if (EspState.loot) "物资扫描开" else "物资扫描关")
                      refresh()
                  })

                  sliders = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL }
                  root.addView(sliders)
                  buildSliders()

                  root.addView(btn("刷新状态") { refresh() })

                  setContentView(ScrollView(this).apply { addView(root) })
                  refresh()
              }

              private fun buildSliders() {
                  addSlider("小地图 X", 0.02f, 0.98f, EspState.mmCxR) { v -> EspState.mmCxR = v }
                  addSlider("小地图 Y", 0.02f, 0.98f, EspState.mmCyR) { v -> EspState.mmCyR = v }
                  addSlider("小地图半径", 0.02f, 0.20f, EspState.mmRr) { v -> EspState.mmRr = v }
                  addSlider("雷达量程(米)", 20f, 300f, EspState.mmWorldR) { v -> EspState.mmWorldR = v }
              }

              private fun addSlider(
                  label: String, minV: Float, maxV: Float, cur: Float,
                  onChange: (Float) -> Unit
              ) {
                  val row = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL
                      setPadding(0, 2, 0, 2)
                  }
                  val title = TextView(this).apply {
                      textSize = 11f
                      text = "$label: ${String.format("%.3f", cur)}"
                  }
                  row.addView(title)
                  val sb = SeekBar(this).apply {
                      max = 1000
                      progress = (((cur - minV) / (maxV - minV)) * 1000f).toInt().coerceIn(0, 1000)
                  }
                  sb.setOnSeekBarChangeListener(object : SeekBar.OnSeekBarChangeListener {
                      override fun onProgressChanged(s: SeekBar?, p: Int, fromUser: Boolean) {
                          val v = minV + (maxV - minV) * (p / 1000f)
                          title.text = "$label: ${String.format("%.3f", v)}"
                          onChange(v)
                      }
                      override fun onStartTrackingTouch(s: SeekBar?) {}
                      override fun onStopTrackingTouch(s: SeekBar?) {}
                  })
                  row.addView(sb)
                  sliders.addView(row)
              }

              private fun btn(t: String, f: () -> Unit): Button =
                  Button(this).apply {
                      text = t
                      textSize = 13f
                      setOnClickListener { f() }
                  }

              private fun toast(t: String) = Toast.makeText(this, t, Toast.LENGTH_SHORT).show()

              private fun reload() {
                  status.postDelayed({ refresh() }, 500)
              }

              private fun refresh() {
                  status.text = buildString {
                      append("版本 ").append(VER).append("  模型: ")
                          .append(if (EspState.model.isEmpty()) "【无】" else EspState.model)
                          .append("  ").append(EspState.mode).append('\n')
                      append("无障碍 ").append(if (EspAccessibilityService.connected) "√" else "×")
                          .append(" 悬浮窗 ").append(if (Settings.canDrawOverlays(this@MainActivity)) "√" else "×")
                          .append(" 气泡 ").append(if (OverlayService.alive) "开" else "关")
                          .append(" 采集 ").append(if (CaptureService.alive) "开" else "关").append('\n')
                      append("画面 ").append(EspState.capW).append("x").append(EspState.capH)
                          .append("  帧 ").append(EspState.frames)
                          .append("  框 ").append(EspState.boxes.size)
                          .append("  切片 ").append(if (EspState.tile) "开" else "关")
                          .append("  静止过滤 ").append(if (EspState.hideStatic) "开" else "关").append('\n')
                      append("阈值 ").append(String.format("%.2f", EspState.conf))
                          .append("  ").append(String.format("%.1f", EspState.capFps)).append("fps")
                          .append("  ").append(String.format("%.1f", EspState.inferMs)).append("ms")
                  }
                  diag.text = buildString {
                      append("— 小地图 ROI 自检 —").append('\n')
                      append("红像素 ").append(RadarDiag.redPix).append(" / ")
                          .append(RadarDiag.totalPix)
                          .append("   聚类点 ").append(RadarDiag.clusters).append('\n')
                      append("平均色 ").append(RadarDiag.avgRgb).append('\n')
                      append("主色 ").append(RadarDiag.topColors).append('\n')
                      if (RadarDiag.suggest != "-") append("定位 ").append(RadarDiag.suggest).append('\n')
                      if (EspState.projErr.isNotEmpty()) append("投屏: ").append(EspState.projErr).append('\n')
                      if (EspState.capErr.isNotEmpty()) append("取帧: ").append(EspState.capErr).append('\n')
                      if (EspState.err.isNotEmpty()) append("错误: ").append(EspState.err.take(200))
                  }
                  status.postDelayed({ refresh() }, 1000)
              }

              @Deprecated("legacy api")
              override fun onActivityResult(req: Int, code: Int, data: Intent?) {
                  super.onActivityResult(req, code, data)
                  if (req == REQ_CAP) {
                      if (code == RESULT_OK && data != null) {
                          CaptureService.start(this, code, data)
                          toast("采集已启动，切回游戏")
                      } else {
                          toast("已取消采集授权")
                      }
                      reload()
                  }
              }

              override fun onResume() {
                  super.onResume()
                  refresh()
              }
          }
          XEOF
          wc -l app/src/main/java/com/df/esp/MainActivity.kt

      - name: "10 源码完整性校验"
        run: |
          set -e
          echo "== 生成结果 =="
          fail=0
          for f in settings.gradle.kts build.gradle.kts gradle.properties \
                   app/build.gradle.kts app/src/main/AndroidManifest.xml \
                   app/src/main/res/values/strings.xml app/src/main/res/xml/acc.xml \
                   app/src/main/java/com/df/esp/State.kt \
                   app/src/main/java/com/df/esp/Tracker.kt \
                   app/src/main/java/com/df/esp/Detector.kt \
                   app/src/main/java/com/df/esp/Radar.kt \
                   app/src/main/java/com/df/esp/Loot.kt \
                   app/src/main/java/com/df/esp/CaptureService.kt \
                   app/src/main/java/com/df/esp/Overlay.kt \
                   app/src/main/java/com/df/esp/MainActivity.kt; do
            if [ -f "$f" ]; then echo "$f  $(wc -l < "$f") 行"
            else echo "::error::缺少文件 $f"; fail=1; fi
          done
          chk() { n=$(wc -l < "$1" 2>/dev/null || echo 0); if [ "$n" -lt "$3" ]; then echo "::error::$1 只有 $n 行（应 >= $3），被截断"; fail=1; fi; }
          chk app/src/main/java/com/df/esp/State.kt 90
          chk app/src/main/java/com/df/esp/Tracker.kt 60
          chk app/src/main/java/com/df/esp/Detector.kt 240
          chk app/src/main/java/com/df/esp/CaptureService.kt 300
          chk app/src/main/java/com/df/esp/Overlay.kt 330
          chk app/src/main/java/com/df/esp/MainActivity.kt 240
          [ "$fail" = "0" ] || exit 1
          echo "全部文件就位"

      - name: "11 装 Android SDK"
        run: |
          set -e
          mkdir -p "$ANDROID_SDK_ROOT/cmdline-tools"
          curl -sSLo /tmp/cmdline.zip https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
          unzip -q /tmp/cmdline.zip -d /tmp/cl
          rm -rf "$ANDROID_SDK_ROOT/cmdline-tools/latest"
          mv /tmp/cl/cmdline-tools "$ANDROID_SDK_ROOT/cmdline-tools/latest"
          echo "$ANDROID_SDK_ROOT/cmdline-tools/latest/bin" >> "$GITHUB_PATH"
          export PATH="$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$PATH"
          yes | sdkmanager --licenses || true
          sdkmanager --install "platforms;android-34" "build-tools;34.0.0" 2>&1 | tail -6

      - name: "12 装 Gradle 8.7"
        run: |
          set -e
          curl -sSLo /tmp/gradle.zip https://services.gradle.org/distributions/gradle-8.7-bin.zip
          unzip -q /tmp/gradle.zip -d "$HOME"
          echo "$HOME/gradle-8.7/bin" >> "$GITHUB_PATH"
          echo "sdk.dir=$ANDROID_SDK_ROOT" > local.properties
          "$HOME/gradle-8.7/bin/gradle" --version

      - name: "13 导模型（COCO yolov8n 占位，先用它跑通链路）"
        run: |
          set -e
          mkdir -p app/src/main/assets
          python3 -m pip install --quiet --upgrade pip
          python3 -m pip install --quiet ultralytics onnx onnxruntime
          ok=0
          if python3 -c "from ultralytics import YOLO; YOLO('yolov8n.pt').export(format='onnx', imgsz=640, opset=12, simplify=False, dynamic=False)"; then ok=1; fi
          if [ "$ok" = "0" ]; then
            python3 -c "from ultralytics import YOLO; YOLO('yolov8n.pt').export(format='onnx', imgsz=640, opset=12, simplify=True, dynamic=False)" && ok=1 || true
          fi
          f=$(ls -1 *.onnx 2>/dev/null | head -1 || true)
          [ -n "$f" ] || { echo "::error::没拿到 onnx"; exit 1; }
          mv -f "$f" app/src/main/assets/yolov8n.onnx
          ls -lh app/src/main/assets

      - name: "14 编译并校验 APK"
        run: |
          export PATH="$HOME/gradle-8.7/bin:$PATH"
          export NO_COLOR=1
          echo "sdk.dir=$ANDROID_SDK_ROOT" > local.properties
          set +e
          gradle :app:assembleDebug --no-daemon --console=plain --stacktrace > /tmp/build.log 2>&1
          rc=$?
          cp -f /tmp/build.log ./build.log
          echo "===== exit=$rc ====="
          grep -n "^e: " /tmp/build.log | head -60
          grep -nE "What went wrong|Execution failed|Unresolved|FAILURE:" /tmp/build.log | head -30
          if [ "$rc" = "0" ]; then
            unzip -l app/build/outputs/apk/debug/app-debug.apk | grep "assets/" || true
            unzip -l app/build/outputs/apk/debug/app-debug.apk | grep -q "assets/yolov8n.onnx" \
              || { echo "::error::APK 内没有模型"; exit 1; }
          fi
          exit $rc

      - name: "15 上传 APK"
        if: success()
        uses: actions/upload-artifact@v4
        with:
          name: df-esp-apk
          path: app/build/outputs/apk/debug/app-debug.apk
          retention-days: 30
          if-no-files-found: error

      - name: "16 上传编译日志"
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: build-log
          path: build.log
          retention-days: 7
          if-no-files-found: warn
