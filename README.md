// Android ANPR App - Core Classes Overview
// Language: Java (can also be done in Kotlin if preferred)

// MainActivity.java
// Highlight flagged plates in the camera feed UI

import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.Rect;
import android.view.SurfaceView;
import android.view.View;

// Inside MainActivity.java after CameraX setup:
overlay = new SurfaceView(this);
overlay.setZOrderOnTop(true);
overlay.getHolder().setFormat(android.graphics.PixelFormat.TRANSPARENT);
((ViewGroup) viewFinder.getParent()).addView(overlay);

import androidx.biometric.BiometricManager;
import androidx.biometric.BiometricPrompt;
import androidx.core.content.ContextCompat;

// Combined settings panel with expandable menu
ImageButton settingsToggle = new ImageButton(this);
settingsToggle.setImageResource(android.R.drawable.ic_menu_manage);
settingsToggle.setBackgroundColor(Color.TRANSPARENT);

LinearLayout settingsPanel = new LinearLayout(this);
settingsPanel.setOrientation(LinearLayout.VERTICAL);
settingsPanel.setPadding(16, 16, 16, 16);
settingsPanel.setBackgroundColor(Color.parseColor("#AA000000"));
settingsPanel.setVisibility(View.GONE);

Button toggleHighlightBtn = new Button(this);
toggleHighlightBtn.setText("Disable Highlight");
toggleHighlightBtn.setOnClickListener(v -> {
    highlightEnabled = !highlightEnabled;
    toggleHighlightBtn.setText(highlightEnabled ? "Disable Highlight" : "Enable Highlight");
});
settingsPanel.addView(toggleHighlightBtn);

Button toggleMuteBtn = new Button(this);
toggleMuteBtn.setText("Mute Alerts");
toggleMuteBtn.setOnClickListener(v -> {
    muteAlerts = !muteAlerts;
    toggleMuteBtn.setText(muteAlerts ? "Unmute Alerts" : "Mute Alerts");
});
settingsPanel.addView(toggleMuteBtn);

// Additional settings
TextView passcodeLabel = new TextView(this);
passcodeLabel.setText("Admin Passcode:");
passcodeLabel.setTextColor(Color.WHITE);
settingsPanel.addView(passcodeLabel);

EditText passcodeInput = new EditText(this);
passcodeInput.setInputType(android.text.InputType.TYPE_CLASS_TEXT | android.text.InputType.TYPE_TEXT_VARIATION_PASSWORD);
CheckBox showPassCheckbox = new CheckBox(this);
showPassCheckbox.setText("Show Passcode");
showPassCheckbox.setTextColor(Color.WHITE);
showPassCheckbox.setOnCheckedChangeListener((buttonView, isChecked) -> {
    if (isChecked) {
        passcodeInput.setInputType(android.text.InputType.TYPE_CLASS_TEXT);
    } else {
        passcodeInput.setInputType(android.text.InputType.TYPE_CLASS_TEXT | android.text.InputType.TYPE_TEXT_VARIATION_PASSWORD);
    }
    passcodeInput.setSelection(passcodeInput.getText().length());
});
settingsPanel.addView(showPassCheckbox);
passcodeInput.setText(PlateHistory.getAdminPasscode());
passcodeInput.setBackgroundColor(Color.WHITE);
passcodeInput.setTextColor(Color.BLACK);
settingsPanel.addView(passcodeInput);

Button savePassBtn = new Button(this);
savePassBtn.setText("Save Passcode");
savePassBtn.setOnClickListener(v -> {
    String newPass = passcodeInput.getText().toString();
    if (newPass.length() < 4) {
        Toast.makeText(this, "Passcode must be at least 4 characters", Toast.LENGTH_SHORT).show();
        return;
    }
    PlateHistory.setAdminPasscode(newPass);
    Toast.makeText(this, "Passcode updated", Toast.LENGTH_SHORT).show();
});
settingsPanel.addView(savePassBtn);
CheckBox autoDeleteLogsCheckbox = new CheckBox(this);
autoDeleteLogsCheckbox.setText("Auto-Delete Old Audit Logs");
autoDeleteLogsCheckbox.setTextColor(Color.WHITE);
autoDeleteLogsCheckbox.setChecked(PlateHistory.isAutoDeleteLogsEnabled());
autoDeleteLogsCheckbox.setOnCheckedChangeListener((buttonView, isChecked) -> {
    PlateHistory.setAutoDeleteLogsEnabled(isChecked);
    Toast.makeText(this, isChecked ? "Auto-delete logs enabled" : "Auto-delete logs disabled", Toast.LENGTH_SHORT).show();
});
settingsPanel.addView(autoDeleteLogsCheckbox);

CheckBox autoClearCheckbox = new CheckBox(this);
autoClearCheckbox.setText("Enable Auto-Clear History");
autoClearCheckbox.setTextColor(Color.WHITE);
autoClearCheckbox.setChecked(PlateHistory.isAutoClearEnabled());
autoClearCheckbox.setOnCheckedChangeListener((buttonView, isChecked) -> {
    PlateHistory.setAutoClearEnabled(isChecked);
    Toast.makeText(this, isChecked ? "Auto-clear enabled" : "Auto-clear disabled", Toast.LENGTH_SHORT).show();
});
settingsPanel.addView(autoClearCheckbox);
TextView historyLimitLabel = new TextView(this);
historyLimitLabel.setText("History Limit (Auto Clear):");
historyLimitLabel.setTextColor(Color.WHITE);
settingsPanel.addView(historyLimitLabel);

EditText historyLimitInput = new EditText(this);
historyLimitInput.setInputType(android.text.InputType.TYPE_CLASS_NUMBER);
historyLimitInput.setText(String.valueOf(PlateHistory.getLimit()));
historyLimitInput.setBackgroundColor(Color.WHITE);
historyLimitInput.setTextColor(Color.BLACK);
settingsPanel.addView(historyLimitInput);

// Log storage size
TextView logSizeLabel = new TextView(this);
long logBytes = PlateHistory.getLogStorageSize();
logSizeLabel.setText("Audit Log Storage: " + (logBytes / 1024) + " KB");
logSizeLabel.setTextColor(Color.WHITE);
settingsPanel.addView(logSizeLabel);

// Manual Clear History Button
Button clearHistoryBtn = new Button(this);
clearHistoryBtn.setText("Clear All History");
clearHistoryBtn.setOnClickListener(v -> {
    BiometricManager biometricManager = BiometricManager.from(this);
    if (biometricManager.canAuthenticate(BiometricManager.Authenticators.BIOMETRIC_STRONG) == BiometricManager.BIOMETRIC_SUCCESS) {
        BiometricPrompt biometricPrompt = new BiometricPrompt(this, ContextCompat.getMainExecutor(this), new BiometricPrompt.AuthenticationCallback() {
            @Override
            public void onAuthenticationSucceeded(BiometricPrompt.AuthenticationResult result) {
                super.onAuthenticationSucceeded(result);
                runOnUiThread(() -> {
                    new AlertDialog.Builder(MainActivity.this)
                        .setTitle("Clear History")
                        .setMessage("Are you sure you want to delete all stored plates?")
                        .setPositiveButton("Yes", (dialog, which) -> {
                            PlateHistory.clearAll();
                            Toast.makeText(MainActivity.this, "History cleared", Toast.LENGTH_SHORT).show();
                        })
                        .setNegativeButton("No", null)
                        .show();
                });
            }
        });
        BiometricPrompt.PromptInfo promptInfo = new BiometricPrompt.PromptInfo.Builder()
            .setTitle("Biometric Authentication")
            .setSubtitle("Confirm to clear plate history")
            .setNegativeButtonText("Cancel")
            .build();
        biometricPrompt.authenticate(promptInfo);
    } else {
        Toast.makeText(this, "Biometric authentication not available", Toast.LENGTH_SHORT).show();
    }
});
        })
        .setNegativeButton("No", null)
        .show();
});
settingsPanel.addView(clearHistoryBtn);

historyLimitInput.setOnEditorActionListener((v, actionId, event) -> {
    try {
        int limit = Integer.parseInt(v.getText().toString());
        PlateHistory.setLimit(limit);
        Toast.makeText(this, "History limit set to " + limit, Toast.LENGTH_SHORT).show();
    } catch (NumberFormatException e) {
        Toast.makeText(this, "Invalid number", Toast.LENGTH_SHORT).show();
    }
    return true;
});

settingsToggle.setOnClickListener(v -> {
    settingsPanel.setVisibility(settingsPanel.getVisibility() == View.VISIBLE ? View.GONE : View.VISIBLE);
});

FrameLayout.LayoutParams iconParams = new FrameLayout.LayoutParams(
    FrameLayout.LayoutParams.WRAP_CONTENT,
    FrameLayout.LayoutParams.WRAP_CONTENT
);
iconParams.setMargins(16, 16, 16, 16);
iconParams.gravity = Gravity.TOP | Gravity.END;

FrameLayout.LayoutParams panelParams = new FrameLayout.LayoutParams(
    FrameLayout.LayoutParams.WRAP_CONTENT,
    FrameLayout.LayoutParams.WRAP_CONTENT
);
panelParams.setMargins(16, 100, 16, 16);
panelParams.gravity = Gravity.TOP | Gravity.END;

((ViewGroup) viewFinder.getParent()).addView(settingsToggle, iconParams);
((ViewGroup) viewFinder.getParent()).addView(settingsPanel, panelParams);
});
((ViewGroup) viewFinder.getParent()).addView(toggleHighlightBtn);

// Inside the analyzer in imageAnalysis.setAnalyzer(...):
imageAnalysis.setAnalyzer(cameraExecutor, image -> {
    Bitmap bitmap = ImageUtils.imageToBitmap(image);
    String plateText = PlateRecognizer.runInference(bitmap, plateDetectionModel, ocrModel);

    runOnUiThread(() -> {
        resultText.setText(plateText);
        if (PlateHistory.isFlagged(plateText)) triggerFlagAlert();
        PlateHistory.add(plateText);
        if (highlightEnabled) drawHighlightIfFlagged(plateText);
    });
    image.close();
});

// Sound and vibration alert for flagged plates
private boolean muteAlerts = false;
private void triggerFlagAlert() {
    if (muteAlerts) return;
    android.os.VibrationEffect effect = android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.O ?
        android.os.VibrationEffect.createOneShot(300, android.os.VibrationEffect.DEFAULT_AMPLITUDE) : null;
    android.os.Vibrator vibrator = (android.os.Vibrator) getSystemService(Context.VIBRATOR_SERVICE);
    if (vibrator != null && vibrator.hasVibrator()) {
        if (effect != null) vibrator.vibrate(effect);
        else vibrator.vibrate(300);
    }
    android.media.ToneGenerator toneGen = new android.media.ToneGenerator(android.media.AudioManager.STREAM_MUSIC, 100);
    toneGen.startTone(android.media.ToneGenerator.TONE_CDMA_ALERT_CALL_GUARD, 300);
}
    android.media.ToneGenerator toneGen = new android.media.ToneGenerator(android.media.AudioManager.STREAM_MUSIC, 100);
    toneGen.startTone(android.media.ToneGenerator.TONE_CDMA_ALERT_CALL_GUARD, 300);
}

// New method in MainActivity
private void drawHighlightIfFlagged(String plate) {
    Canvas canvas = overlay.getHolder().lockCanvas();
    if (canvas != null) {
        canvas.drawColor(Color.TRANSPARENT, android.graphics.PorterDuff.Mode.CLEAR);
        if (PlateHistory.isFlagged(plate)) {
            Paint paint = new Paint();
            paint.setColor(Color.RED);
            paint.setStyle(Paint.Style.STROKE);
            paint.setStrokeWidth(8);
            Rect rect = new Rect(100, 100, viewFinder.getWidth() - 100, viewFinder.getHeight() - 100);
            canvas.drawRect(rect, paint);
        }
        overlay.getHolder().unlockCanvasAndPost(canvas);
    }
}

// Add this new Activity: HistoryActivity.java
// This screen displays and searches through detected plate history
package com.example.anprapp;

import android.content.Context;
import android.content.SharedPreferences;
import android.os.Bundle;
import android.text.Editable;
import android.text.TextWatcher;
import android.view.View;
import android.view.ViewGroup;
import android.widget.AdapterView;
import android.widget.ArrayAdapter;
import android.widget.Button;
import android.widget.EditText;
import android.widget.ListView;
import android.widget.TextView;
import android.widget.Toast;

import androidx.appcompat.app.AlertDialog;
import androidx.appcompat.app.AppCompatActivity;

import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

import android.content.Intent;
import android.net.Uri;
import java.io.File;

public class HistoryActivity extends AppCompatActivity {
    private ListView listView;
    private EditText searchInput;
    private Button toggleFilterBtn;
    private ArrayAdapter<String> adapter;
    private List<String> historyList;
    private boolean filterFlagged = false;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_history);

        listView = findViewById(R.id.listView);
        searchInput = findViewById(R.id.searchInput);
        toggleFilterBtn = findViewById(R.id.toggleFlagFilter);

        historyList = PlateHistory.getAll();
        adapter = new ArrayAdapter<String>(this, android.R.layout.simple_list_item_1, new ArrayList<>(historyList)) {
            @Override
            public View getView(int position, View convertView, ViewGroup parent) {
                View view = super.getView(position, convertView, parent);
                String plate = getItem(position);
                TextView textView = view.findViewById(android.R.id.text1);
                if (PlateHistory.isFlagged(plate)) {
                    textView.setText("🚩 " + plate);
                    textView.setTextColor(getResources().getColor(android.R.color.holo_red_light));
                } else {
                    textView.setText(plate);
                    textView.setTextColor(getResources().getColor(android.R.color.black));
                }
                return view;
            }
        };
        listView.setAdapter(adapter);

        searchInput.addTextChangedListener(new TextWatcher() {
            @Override public void beforeTextChanged(CharSequence s, int start, int count, int after) {}
            @Override public void onTextChanged(CharSequence s, int start, int before, int count) {
                filter(s.toString());
            }
            @Override public void afterTextChanged(Editable s) {}
        });

        listView.setOnItemLongClickListener((parent, view, position, id) -> {
            String selectedPlate = adapter.getItem(position);
            showActionDialog(selectedPlate);
            return true;
        });

        toggleFilterBtn.setOnClickListener(v -> {
    filterFlagged = !filterFlagged;
    refreshList();
    toggleFilterBtn.setText(filterFlagged ? "Show All" : "Show Flagged Only");
});

// Add Email Flagged Plates button
Button emailFlaggedBtn = new Button(this);
emailFlaggedBtn.setText("Email Flagged Plates");
emailFlaggedBtn.setOnClickListener(v -> {
    List<String> flagged = PlateHistory.getFlagged();
    if (flagged.isEmpty()) {
        Toast.makeText(this, "No flagged plates to email", Toast.LENGTH_SHORT).show();
        return;
    }
    StringBuilder message = new StringBuilder();
    message.append("Flagged Plates Report:

");
    for (String plate : flagged) {
        message.append("🚩 ").append(plate).append("
");
    }

    Intent emailIntent = new Intent(Intent.ACTION_SEND);
    emailIntent.setType("message/rfc822");
    emailIntent.putExtra(Intent.EXTRA_SUBJECT, "Flagged ANPR Plates Report");
    emailIntent.putExtra(Intent.EXTRA_TEXT, message.toString());

    try {
        startActivity(Intent.createChooser(emailIntent, "Send email via..."));
    } catch (android.content.ActivityNotFoundException ex) {
        Toast.makeText(this, "No email clients installed.", Toast.LENGTH_SHORT).show();
    }
});
((ViewGroup) toggleFilterBtn.getParent()).addView(emailFlaggedBtn);

// Add Share Log button
Button shareLogBtn = new Button(this);
shareLogBtn.setText("Share Audit Logs");
shareLogBtn.setOnClickListener(v -> {
    File dir = getExternalFilesDir(null);
    if (dir != null && dir.exists()) {
        File[] files = dir.listFiles((d, name) -> name.startsWith("clear_history_log_"));
        if (files != null && files.length > 0) {
            File latest = files[files.length - 1];
            Uri uri = androidx.core.content.FileProvider.getUriForFile(this, getPackageName() + ".fileprovider", latest);
            Intent shareIntent = new Intent(Intent.ACTION_SEND);
            shareIntent.setType("text/plain");
            shareIntent.putExtra(Intent.EXTRA_STREAM, uri);
            shareIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
            startActivity(Intent.createChooser(shareIntent, "Share audit log via"));
        } else {
            Toast.makeText(this, "No audit logs found", Toast.LENGTH_SHORT).show();
        }
    }
});
((ViewGroup) toggleFilterBtn.getParent()).addView(shareLogBtn);

// Add Delete Logs button
Button deleteLogsBtn = new Button(this);
deleteLogsBtn.setText("Delete All Audit Logs");
deleteLogsBtn.setOnClickListener(v -> {
    EditText passInput = new EditText(this);
    passInput.setInputType(android.text.InputType.TYPE_CLASS_TEXT | android.text.InputType.TYPE_TEXT_VARIATION_PASSWORD);

    new AlertDialog.Builder(this)
        .setTitle("Admin Passcode")
        .setMessage("Enter passcode to delete audit logs:")
        .setView(passInput)
        .setPositiveButton("Confirm", (dialog, which) -> {
            String entered = passInput.getText().toString();
            SharedPreferences prefs = getSharedPreferences("PlateHistoryPrefs", MODE_PRIVATE);
            String savedPasscode = prefs.getString("admin_passcode", "admin123");
            if (savedPasscode.equals(entered)) {
                File dir = getExternalFilesDir(null);
                if (dir != null && dir.exists()) {
                    File[] files = dir.listFiles((d, name) -> name.startsWith("clear_history_log_"));
                    if (files != null) {
                        for (File file : files) file.delete();
                        Toast.makeText(this, "All audit logs deleted", Toast.LENGTH_SHORT).show();
                    }
                }
            } else {
                Toast.makeText(this, "Incorrect passcode", Toast.LENGTH_SHORT).show();
            }
        })
        .setNegativeButton("Cancel", null)
        .show();
                }
            }
        })
        .setNegativeButton("No", null)
        .show();
});
((ViewGroup) toggleFilterBtn.getParent()).addView(deleteLogsBtn);
((ViewGroup) toggleFilterBtn.getParent()).addView(shareLogBtn);
    }

    private void filter(String text) {
        List<String> filtered = new ArrayList<>();
        for (String item : historyList) {
            if (item.toLowerCase().contains(text.toLowerCase())) {
                filtered.add(item);
            }
        }
        adapter.clear();
        adapter.addAll(filtered);
        adapter.notifyDataSetChanged();
    }

    private void showActionDialog(String plate) {
    AlertDialog.Builder builder = new AlertDialog.Builder(this);
    builder.setTitle("Select Action for " + plate);
    builder.setItems(new CharSequence[]{"Delete", "Flag"}, (dialog, which) -> {
        if (which == 0) {
    new AlertDialog.Builder(this)
        .setTitle("Confirm Delete")
        .setMessage("Are you sure you want to delete " + plate + "?")
        .setPositiveButton("Yes", (confirmDialog, i) -> {
            PlateHistory.remove(plate);
            refreshList();
            Toast.makeText(this, "Plate deleted", Toast.LENGTH_SHORT).show();
        })
        .setNegativeButton("No", null)
        .show();
} else {
    new AlertDialog.Builder(this)
        .setTitle("Confirm Flag")
        .setMessage("Are you sure you want to flag " + plate + "?")
        .setPositiveButton("Yes", (confirmDialog, i) -> {
            PlateHistory.flag(plate);
            refreshList();
            Toast.makeText(this, "Plate flagged", Toast.LENGTH_SHORT).show();
        })
        .setNegativeButton("No", null)
        .show();
                })
                .setNegativeButton("No", null)
                .show();
        } else {
            PlateHistory.flag(plate);
            refreshList();
            Toast.makeText(this, "Plate flagged", Toast.LENGTH_SHORT).show();
        }
    });
    builder.show();
    }

    private void refreshList() {
        historyList = filterFlagged ? PlateHistory.getFlagged() : PlateHistory.getAll();
        filter(searchInput.getText().toString());
    }
}

// PlateHistory.java — persistent flag/delete history manager using SharedPreferences
class PlateHistory {
    public static void setAdminPasscode(String passcode) {
        SharedPreferences prefs = getPrefs(App.getContext());
        prefs.edit().putString("admin_passcode", passcode).apply();
    }

    public static String getAdminPasscode() {
        SharedPreferences prefs = getPrefs(App.getContext());
        return prefs.getString("admin_passcode", "admin123");
    }
    public static long getLogStorageSize() {
        File dir = App.getContext().getExternalFilesDir(null);
        if (dir != null && dir.exists()) {
            File[] files = dir.listFiles((d, name) -> name.startsWith("clear_history_log_"));
            long total = 0;
            if (files != null) {
                for (File file : files) total += file.length();
            }
            return total;
        }
        return 0;
    }
    private static final String KEY_AUTO_CLEAR = "auto_clear";
private static final String KEY_AUTO_DELETE_LOGS = "auto_delete_logs";
    private static final String KEY_LIMIT = "limit";
    private static final String PREFS_NAME = "PlateHistoryPrefs";
    private static final String KEY_HISTORY = "history";
    private static final String KEY_FLAGGED = "flagged";

    private static SharedPreferences getPrefs(Context context) {
        return context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE);
    }

    public static void add(String plate) {
    SharedPreferences prefs = getPrefs(App.getContext());
    Set<String> history = new HashSet<>(prefs.getStringSet(KEY_HISTORY, new HashSet<>()));
    history.add(plate);

    if (isAutoClearEnabled()) {
        int limit = getLimit();
        if (history.size() > limit) {
            List<String> sorted = new ArrayList<>(history);
            while (sorted.size() > limit) {
                sorted.remove(0);
            }
            history = new HashSet<>(sorted);
        }
    }

    prefs.edit()
        .remove(KEY_HISTORY)
        .remove(KEY_FLAGGED)
        .apply();

    // Auto-delete logs older than 30 days
    File dir = App.getContext().getExternalFilesDir(null);
    if (dir != null && dir.exists()) {
        File[] files = dir.listFiles((d, name) -> name.startsWith("clear_history_log_"));
        long now = System.currentTimeMillis();
        if (files != null) {
            for (File file : files) {
                if (now - file.lastModified() > 30L * 24 * 60 * 60 * 1000) {
                    file.delete();
                }
            }
        }
    }
    }

    public static List<String> getAll() {
        SharedPreferences prefs = getPrefs(App.getContext());
        return new ArrayList<>(prefs.getStringSet(KEY_HISTORY, new HashSet<>()));
    }

    public static List<String> getFlagged() {
        SharedPreferences prefs = getPrefs(App.getContext());
        return new ArrayList<>(prefs.getStringSet(KEY_FLAGGED, new HashSet<>()));
    }

    public static void remove(String plate) {
        SharedPreferences prefs = getPrefs(App.getContext());
        Set<String> history = new HashSet<>(prefs.getStringSet(KEY_HISTORY, new HashSet<>()));
        Set<String> flagged = new HashSet<>(prefs.getStringSet(KEY_FLAGGED, new HashSet<>()));
        history.remove(plate);
        flagged.remove(plate);
        prefs.edit().putStringSet(KEY_HISTORY, history).putStringSet(KEY_FLAGGED, flagged).apply();
    }

    public static void flag(String plate) {
        SharedPreferences prefs = getPrefs(App.getContext());
        Set<String> flagged = new HashSet<>(prefs.getStringSet(KEY_FLAGGED, new HashSet<>()));
        flagged.add(plate);
        prefs.edit().putStringSet(KEY_FLAGGED, flagged).apply();
    }

    public static boolean isFlagged(String plate) {
        SharedPreferences prefs = getPrefs(App.getContext());
        return prefs.getStringSet(KEY_FLAGGED, new HashSet<>()).contains(plate);
    }

    public static int getLimit() {
        SharedPreferences prefs = getPrefs(App.getContext());
        return prefs.getInt(KEY_LIMIT, 50);
    }

    public static void setLimit(int limit) {
    SharedPreferences prefs = getPrefs(App.getContext());
    prefs.edit().putInt(KEY_LIMIT, limit).apply();
}

public static boolean isAutoClearEnabled() {
    SharedPreferences prefs = getPrefs(App.getContext());
    return prefs.getBoolean(KEY_AUTO_CLEAR, true);
}

public static void setAutoClearEnabled(boolean enabled) {
    SharedPreferences prefs = getPrefs(App.getContext());
    prefs.edit().putBoolean(KEY_AUTO_CLEAR, enabled).apply();
}

public static boolean isAutoDeleteLogsEnabled() {
    SharedPreferences prefs = getPrefs(App.getContext());
    return prefs.getBoolean(KEY_AUTO_DELETE_LOGS, true);
}

public static void setAutoDeleteLogsEnabled(boolean enabled) {
    SharedPreferences prefs = getPrefs(App.getContext());
    prefs.edit().putBoolean(KEY_AUTO_DELETE_LOGS, enabled).apply();
}

public static void clearAll() {
    SharedPreferences prefs = getPrefs(App.getContext());
    List<String> auditLog = getAll();
    long timestamp = System.currentTimeMillis();
    StringBuilder log = new StringBuilder();
    log.append("Cleared History at ").append(new java.util.Date(timestamp)).append("
");
    for (String plate : auditLog) {
        log.append(plate).append("
");
    }

    File logFile = new File(App.getContext().getExternalFilesDir(null), "clear_history_log_" + timestamp + ".txt");

    if (isAutoDeleteLogsEnabled()) {
        File dir = App.getContext().getExternalFilesDir(null);
        if (dir != null && dir.exists()) {
            File[] files = dir.listFiles((d, name) -> name.startsWith("clear_history_log_"));
            long now = System.currentTimeMillis();
            if (files != null) {
                for (File file : files) {
                    if (now - file.lastModified() > 30L * 24 * 60 * 60 * 1000) {
                        file.delete();
                    }
                }
            }
        }
    }

    try (FileWriter writer = new FileWriter(logFile)) {
        writer.write(log.toString());
    } catch (Exception e) {
        e.printStackTrace();
    }

    prefs.edit()
        .remove(KEY_HISTORY)
        .remove(KEY_FLAGGED)
        .apply();
}
}
    }
}

// App.java (global context provider for SharedPreferences)
package com.example.anprapp;

import android.app.Application;
import android.content.Context;

public class App extends Application {
    private static App instance;

    @Override
    public void onCreate() {
        super.onCreate();
        instance = this;
    }

    public static Context getContext() {
        return instance.getApplicationContext();
    }
}

// AndroidManifest.xml — register Application class
// <application
//    android:name=".App"
//    ... >

// res/layout/activity_history.xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <EditText
        android:id="@+id/searchInput"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Search plates..."
        android:inputType="text" />

    <Button
        android:id="@+id/toggleFlagFilter"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Show Flagged Only"
        android:layout_marginTop="8dp" />

    <ListView
        android:id="@+id/listView"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />
</LinearLayout>
