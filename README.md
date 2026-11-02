name: Build APK
on:
  push:
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: 8.7
      - name: Create project files
        run: |
          mkdir -p app/src/main/java/com/kalima/app

          cat > settings.gradle <<'EOF'
          pluginManagement {
              repositories {
                  google()
                  mavenCentral()
                  gradlePluginPortal()
              }
          }
          dependencyResolutionManagement {
              repositories {
                  google()
                  mavenCentral()
              }
          }
          rootProject.name = 'kalima'
          include ':app'
          EOF

          cat > build.gradle <<'EOF'
          plugins {
              id 'com.android.application' version '8.5.2' apply false
          }
          EOF

          cat > app/build.gradle <<'EOF'
          plugins {
              id 'com.android.application'
          }
          android {
              namespace 'com.kalima.app'
              compileSdk 34
              defaultConfig {
                  applicationId 'com.kalima.app'
                  minSdk 26
                  targetSdk 34
                  versionCode 1
                  versionName '1.0'
              }
              compileOptions {
                  sourceCompatibility JavaVersion.VERSION_17
                  targetCompatibility JavaVersion.VERSION_17
              }
          }
          tasks.withType(JavaCompile).configureEach {
              options.encoding = 'UTF-8'
          }
          EOF

          cat > app/src/main/AndroidManifest.xml <<'EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
              <uses-permission android:name="android.permission.USE_EXACT_ALARM" />
              <uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" android:maxSdkVersion="32" />
              <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
              <uses-permission android:name="android.permission.WAKE_LOCK" />
              <queries>
                  <intent>
                      <action android:name="android.intent.action.TTS_SERVICE" />
                  </intent>
              </queries>
              <application
                  android:label="كلمة"
                  android:supportsRtl="true"
                  android:theme="@android:style/Theme.Material.NoActionBar">
                  <activity android:name=".MainActivity" android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>
                  <receiver android:name=".AlarmReceiver" android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.BOOT_COMPLETED" />
                      </intent-filter>
                  </receiver>
              </application>
          </manifest>
          EOF

          cat > app/src/main/java/com/kalima/app/AlarmReceiver.java <<'EOF'
          package com.kalima.app;

          import android.app.AlarmManager;
          import android.app.Notification;
          import android.app.NotificationChannel;
          import android.app.NotificationManager;
          import android.app.PendingIntent;
          import android.content.BroadcastReceiver;
          import android.content.Context;
          import android.content.Intent;
          import android.content.SharedPreferences;
          import android.media.AudioAttributes;
          import android.os.Build;
          import android.os.Handler;
          import android.os.Looper;
          import android.os.PowerManager;
          import android.speech.tts.TextToSpeech;
          import android.speech.tts.UtteranceProgressListener;
          import org.json.JSONArray;
          import org.json.JSONObject;
          import java.util.Locale;

          public class AlarmReceiver extends BroadcastReceiver {
              static final String CH = "words";
              static final String ACT = "kalima.ALARM";

              static SharedPreferences prefs(Context c) {
                  return c.getSharedPreferences("kalima", Context.MODE_PRIVATE);
              }

              static JSONArray words(Context c) {
                  try {
                      return new JSONArray(prefs(c).getString("words", "[]"));
                  } catch (Exception e) {
                      return new JSONArray();
                  }
              }

              static void schedule(Context c) {
                  AlarmManager am = (AlarmManager) c.getSystemService(Context.ALARM_SERVICE);
                  Intent i = new Intent(c, AlarmReceiver.class);
                  i.setAction(ACT);
                  PendingIntent pi = PendingIntent.getBroadcast(c, 1, i,
                          PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE);
                  am.cancel(pi);
                  int min = prefs(c).getInt("interval", 30);
                  if (min <= 0 || words(c).length() == 0) return;
                  long at = System.currentTimeMillis() + min * 60000L;
                  if (Build.VERSION.SDK_INT >= 31 && !am.canScheduleExactAlarms()) {
                      am.setAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, at, pi);
                  } else {
                      am.setExactAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, at, pi);
                  }
              }

              static void showNotification(Context c, String w, String m) {
                  NotificationManager nm = (NotificationManager) c.getSystemService(Context.NOTIFICATION_SERVICE);
                  NotificationChannel ch = new NotificationChannel(CH, "Words", NotificationManager.IMPORTANCE_HIGH);
                  ch.setSound(null, null);
                  nm.createNotificationChannel(ch);
                  PendingIntent open = PendingIntent.getActivity(c, 0,
                          new Intent(c, MainActivity.class), PendingIntent.FLAG_IMMUTABLE);
                  Notification n = new Notification.Builder(c, CH)
                          .setSmallIcon(android.R.drawable.ic_dialog_info)
                          .setContentTitle(w)
                          .setContentText(m)
                          .setContentIntent(open)
                          .setAutoCancel(true)
                          .build();
                  nm.notify(1, n);
              }

              @Override
              public void onReceive(Context ctx, Intent intent) {
                  final Context c = ctx.getApplicationContext();
                  String action = intent.getAction();
                  if (Intent.ACTION_BOOT_COMPLETED.equals(action)) {
                      schedule(c);
                      return;
                  }
                  if (!ACT.equals(action)) return;

                  schedule(c);

                  JSONArray arr = words(c);
                  if (arr.length() == 0) return;
                  int idx = prefs(c).getInt("idx", 0) % arr.length();
                  prefs(c).edit().putInt("idx", idx + 1).apply();
                  JSONObject o = arr.optJSONObject(idx);
                  if (o == null) return;
                  final String w = o.optString("w");
                  final String m = o.optString("m");
                  final String lang = o.optString("l", "el-GR");

                  showNotification(c, w, m);

                  final PendingResult pr = goAsync();
                  final PowerManager.WakeLock wl = ((PowerManager) c.getSystemService(Context.POWER_SERVICE))
                          .newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "kalima:tts");
                  wl.acquire(15000);
                  final Handler h = new Handler(Looper.getMainLooper());
                  final TextToSpeech[] holder = new TextToSpeech[1];
                  final Runnable finish = new Runnable() {
                      boolean done = false;
                      @Override
                      public void run() {
                          if (done) return;
                          done = true;
                          try {
                              if (holder[0] != null) holder[0].shutdown();
                          } catch (Exception e) {
                          }
                          if (wl.isHeld()) wl.release();
                          pr.finish();
                      }
                  };
                  h.postDelayed(finish, 9000);

                  holder[0] = new TextToSpeech(c, status -> {
                      TextToSpeech t = holder[0];
                      if (status != TextToSpeech.SUCCESS || t == null) {
                          h.post(finish);
                          return;
                      }
                      int r = t.setLanguage(Locale.forLanguageTag(lang));
                      if (r == TextToSpeech.LANG_MISSING_DATA || r == TextToSpeech.LANG_NOT_SUPPORTED) {
                          h.post(finish);
                          return;
                      }
                      t.setAudioAttributes(new AudioAttributes.Builder()
                              .setUsage(AudioAttributes.USAGE_MEDIA)
                              .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH)
                              .build());
                      t.setOnUtteranceProgressListener(new UtteranceProgressListener() {
                          @Override
                          public void onStart(String id) {
                          }
                          @Override
                          public void onDone(String id) {
                              h.post(finish);
                          }
                          @Override
                          public void onError(String id) {
                              h.post(finish);
                          }
                      });
                      t.speak(w, TextToSpeech.QUEUE_FLUSH, null, "kalima");
                  });
              }
          }
          EOF

          cat > app/src/main/java/com/kalima/app/MainActivity.java <<'EOF'
          package com.kalima.app;

          import android.app.Activity;
          import android.content.Intent;
          import android.os.Build;
          import android.os.Bundle;
          import android.provider.Settings;
          import android.speech.tts.TextToSpeech;
          import android.view.Gravity;
          import android.view.View;
          import android.view.ViewGroup;
          import android.widget.AdapterView;
          import android.widget.ArrayAdapter;
          import android.widget.Button;
          import android.widget.EditText;
          import android.widget.LinearLayout;
          import android.widget.ScrollView;
          import android.widget.Spinner;
          import android.widget.TextView;
          import android.widget.Toast;
          import org.json.JSONArray;
          import org.json.JSONObject;
          import java.util.Locale;

          public class MainActivity extends Activity {
              static final String[] LN = {"اليونانية", "English", "Español", "Français", "Deutsch", "Italiano", "Türkçe"};
              static final String[] LC = {"el-GR", "en-US", "es-ES", "fr-FR", "de-DE", "it-IT", "tr-TR"};
              static final String[] IN = {"كل دقيقة", "كل 30 دقيقة", "كل 4 ساعات", "إيقاف"};
              static final int[] IM = {1, 30, 240, 0};

              TextToSpeech tts;
              boolean ttsReady = false;
              boolean firstSel = true;
              LinearLayout list;

              int dp(int v) {
                  return (int) (v * getResources().getDisplayMetrics().density);
              }

              Button btn(String text, View.OnClickListener l) {
                  Button b = new Button(this);
                  b.setText(text);
                  b.setOnClickListener(l);
                  return b;
              }

              void save(JSONArray arr) {
                  AlarmReceiver.prefs(this).edit().putString("words", arr.toString()).apply();
              }

              void speak(String w, String lang) {
                  if (!ttsReady) {
                      Toast.makeText(this, "الصوت لسه بيتجهز", Toast.LENGTH_SHORT).show();
                      return;
                  }
                  int r = tts.setLanguage(Locale.forLanguageTag(lang));
                  if (r == TextToSpeech.LANG_MISSING_DATA || r == TextToSpeech.LANG_NOT_SUPPORTED) {
                      Toast.makeText(this, "نزّل صوت اللغة من إعدادات النطق", Toast.LENGTH_LONG).show();
                      return;
                  }
                  tts.speak(w, TextToSpeech.QUEUE_FLUSH, null, "w");
              }

              void refresh() {
                  list.removeAllViews();
                  final JSONArray arr = AlarmReceiver.words(this);
                  for (int i = 0; i < arr.length(); i++) {
                      final int pos = i;
                      final JSONObject o = arr.optJSONObject(i);
                      if (o == null) continue;
                      LinearLayout row = new LinearLayout(this);
                      row.setOrientation(LinearLayout.HORIZONTAL);
                      row.setGravity(Gravity.CENTER_VERTICAL);
                      TextView t = new TextView(this);
                      t.setText(o.optString("w") + "  -  " + o.optString("m"));
                      t.setTextSize(18);
                      row.addView(t, new LinearLayout.LayoutParams(0, ViewGroup.LayoutParams.WRAP_CONTENT, 1f));
                      row.addView(btn("\uD83D\uDD0A", v -> speak(o.optString("w"), o.optString("l", "el-GR"))));
                      row.addView(btn("\u2715", v -> {
                          arr.remove(pos);
                          save(arr);
                          AlarmReceiver.schedule(this);
                          refresh();
                      }));
                      list.addView(row);
                  }
              }

              @Override
              protected void onCreate(Bundle b) {
                  super.onCreate(b);
                  if (Build.VERSION.SDK_INT >= 33) {
                      requestPermissions(new String[]{"android.permission.POST_NOTIFICATIONS"}, 1);
                  }
                  tts = new TextToSpeech(this, s -> ttsReady = (s == TextToSpeech.SUCCESS));

                  ScrollView sv = new ScrollView(this);
                  LinearLayout root = new LinearLayout(this);
                  root.setOrientation(LinearLayout.VERTICAL);
                  root.setPadding(dp(16), dp(24), dp(16), dp(24));
                  root.setLayoutDirection(View.LAYOUT_DIRECTION_RTL);
                  sv.addView(root);
                  setContentView(sv);

                  TextView title = new TextView(this);
                  title.setText("كلمة - حافظ الكلمات");
                  title.setTextSize(24);
                  root.addView(title);

                  final EditText etW = new EditText(this);
                  etW.setHint("الكلمة");
                  root.addView(etW);
                  final EditText etM = new EditText(this);
                  etM.setHint("المعنى بالعربي");
                  root.addView(etM);

                  final Spinner spL = new Spinner(this);
                  spL.setAdapter(new ArrayAdapter<String>(this, android.R.layout.simple_spinner_dropdown_item, LN));
                  root.addView(spL);

                  root.addView(btn("إضافة كلمة", v -> {
                      String w = etW.getText().toString().trim();
                      String m = etM.getText().toString().trim();
                      if (w.isEmpty()) return;
                      try {
                          JSONArray arr = AlarmReceiver.words(this);
                          JSONObject o = new JSONObject();
                          o.put("w", w);
                          o.put("m", m);
                          o.put("l", LC[spL.getSelectedItemPosition()]);
                          arr.put(o);
                          save(arr);
                      } catch (Exception e) {
                      }
                      etW.setText("");
                      etM.setText("");
                      AlarmReceiver.schedule(this);
                      refresh();
                  }));

                  TextView lbl = new TextView(this);
                  lbl.setText("التذكير والنطق:");
                  lbl.setPadding(0, dp(16), 0, 0);
                  root.addView(lbl);

                  Spinner spI = new Spinner(this);
                  spI.setAdapter(new ArrayAdapter<String>(this, android.R.layout.simple_spinner_dropdown_item, IN));
                  int cur = AlarmReceiver.prefs(this).getInt("interval", 30);
                  for (int i = 0; i < IM.length; i++) {
                      if (IM[i] == cur) spI.setSelection(i);
                  }
                  spI.setOnItemSelectedListener(new AdapterView.OnItemSelectedListener() {
                      @Override
                      public void onItemSelected(AdapterView<?> p, View v, int pos, long id) {
                          if (firstSel) {
                              firstSel = false;
                              return;
                          }
                          AlarmReceiver.prefs(MainActivity.this).edit().putInt("interval", IM[pos]).apply();
                          AlarmReceiver.schedule(MainActivity.this);
                      }
                      @Override
                      public void onNothingSelected(AdapterView<?> p) {
                      }
                  });
                  root.addView(spI);

                  root.addView(btn("جرّب الآن (إشعار + نطق)", v -> {
                      Intent i = new Intent(this, AlarmReceiver.class);
                      i.setAction(AlarmReceiver.ACT);
                      sendBroadcast(i);
                  }));
                  root.addView(btn("إعدادات النطق (تنزيل صوت اليونانية)", v -> {
                      try {
                          startActivity(new Intent("com.android.settings.TTS_SETTINGS"));
                      } catch (Exception e) {
                          startActivity(new Intent(Settings.ACTION_SETTINGS));
                      }
                  }));
                  root.addView(btn("استثناء من توفير البطارية", v -> {
                      try {
                          startActivity(new Intent(Settings.ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS));
                      } catch (Exception e) {
                          startActivity(new Intent(Settings.ACTION_SETTINGS));
                      }
                  }));

                  list = new LinearLayout(this);
                  list.setOrientation(LinearLayout.VERTICAL);
                  list.setPadding(0, dp(16), 0, 0);
                  root.addView(list);
                  refresh();
              }

              @Override
              protected void onDestroy() {
                  if (tts != null) tts.shutdown();
                  super.onDestroy();
              }
          }
          EOF

      - name: Build
        run: gradle assembleDebug --no-daemon

      - uses: softprops/action-gh-release@v2
        with:
          tag_name: v${{ github.run_number }}
          files: app/build/outputs/apk/debug/app-debug.apk
