

/ModLoader/app/src/main/java/com/modloader/plugin/PluginAPI.java

// File: PluginAPI.java - API Interface for Plugins to Use (200+ lines)
// Path: /app/src/main/java/com/modloader/plugin/PluginAPI.java

package com.modloader.plugin;

import android.app.Activity;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.view.View;
import android.widget.Toast;
import androidx.appcompat.app.AlertDialog;

import com.modloader.util.LogUtils;
import com.modloader.util.FileUtils;
import com.modloader.util.PathManager;
import com.modloader.loader.ModManager;
import com.modloader.loader.MelonLoaderManager;

import java.io.File;
import java.util.*;

/**
 * Plugin API Interface - Provides core TerrariaLoader functionality to plugins
 * This is the main interface that plugins use to interact with the core application
 */
public class PluginAPI {
    private static final String TAG = "PluginAPI";
    private final Context context;
    private final PluginContext pluginContext;
    
    public PluginAPI(Context context, PluginContext pluginContext) {
        this.context = context;
        this.pluginContext = pluginContext;
    }
    
    // ===== LOGGING API =====
    
    /**
     * Log a message with the plugin's name as tag
     */
    public void log(String message) {
        LogUtils.logUser("[" + pluginContext.getPlugin().getName() + "] " + message);
    }
    
    /**
     * Log a debug message
     */
    public void logDebug(String message) {
        LogUtils.logDebug("[" + pluginContext.getPlugin().getName() + "] " + message);
    }
    
    /**
     * Log an error message
     */
    public void logError(String message) {
        LogUtils.logError("[" + pluginContext.getPlugin().getName() + "] " + message);
    }
    
    /**
     * Log a warning message
     */
    public void logWarning(String message) {
        LogUtils.logWarning("[" + pluginContext.getPlugin().getName() + "] " + message);
    }
    
    // ===== UI API =====
    
    /**
     * Show a toast message
     */
    public void showToast(String message) {
        Toast.makeText(context, message, Toast.LENGTH_SHORT).show();
    }
    
    /**
     * Show a long toast message
     */
    public void showLongToast(String message) {
        Toast.makeText(context, message, Toast.LENGTH_LONG).show();
    }
    
    /**
     * Show an alert dialog
     */
    public void showAlert(String title, String message) {
        showAlert(title, message, null);
    }
    
    /**
     * Show an alert dialog with callback
     */
    public void showAlert(String title, String message, Runnable onOk) {
        if (!(context instanceof Activity)) {
            showToast(title + ": " + message);
            return;
        }
        
        AlertDialog.Builder builder = new AlertDialog.Builder(context);
        builder.setTitle(title)
               .setMessage(message)
               .setPositiveButton("OK", (dialog, which) -> {
                   if (onOk != null) onOk.run();
               })
               .show();
    }
    
    /**
     * Show a confirmation dialog
     */
    public void showConfirmation(String title, String message, Runnable onYes, Runnable onNo) {
        if (!(context instanceof Activity)) {
            showToast(title + ": " + message);
            return;
        }
        
        AlertDialog.Builder builder = new AlertDialog.Builder(context);
        builder.setTitle(title)
               .setMessage(message)
               .setPositiveButton("Yes", (dialog, which) -> {
                   if (onYes != null) onYes.run();
               })
               .setNegativeButton("No", (dialog, which) -> {
                   if (onNo != null) onNo.run();
               })
               .show();
    }
    
    /**
     * Get current activity (may be null)
     */
    public Activity getCurrentActivity() {
        return (context instanceof Activity) ? (Activity) context : null;
    }
    
    // ===== FILE SYSTEM API =====
    
    /**
     * Get plugin's private data directory
     */
    public File getPluginDataDir() {
        return pluginContext.getDataDir();
    }
    
    /**
     * Get plugin's config directory
     */
    public File getPluginConfigDir() {
        return pluginContext.getConfigDir();
    }
    
    /**
     * Get TerrariaLoader's base directory
     */
    public File getTerrariaLoaderBaseDir() {
        return PathManager.getGameBaseDir(context, MelonLoaderManager.TERRARIA_PACKAGE);
    }
    
    /**
     * Get DEX mods directory
     */
    public File getDexModsDir() {
        return PathManager.getDexModsDir(context, MelonLoaderManager.TERRARIA_PACKAGE);
    }
    
    /**
     * Get DLL mods directory
     */
    public File getDllModsDir() {
        return PathManager.getDllModsDir(context, MelonLoaderManager.TERRARIA_PACKAGE);
    }
    
    /**
     * Get application logs directory
     */
    public File getAppLogsDir() {
        File baseDir = PathManager.getGameBaseDir(context, MelonLoaderManager.TERRARIA_PACKAGE);
        return new File(baseDir, "AppLogs");
    }
    
    /**
     * Copy a file safely
     */
    public boolean copyFile(File source, File destination) {
        return FileUtils.copyFile(source, destination);
    }
    
    /**
     * Delete a file safely
     */
    public boolean deleteFile(File file) {
        try {
            return file.delete();
        } catch (Exception e) {
            logError("Failed to delete file: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Create directory if it doesn't exist
     */
    public boolean createDirectory(File dir) {
        return PathManager.ensureDirectoryExists(dir);
    }
    
    /**
     * Get file size in human readable format
     */
    public String formatFileSize(long bytes) {
        return FileUtils.formatFileSize(bytes);
    }
    
    // ===== MOD MANAGEMENT API =====
    
    /**
     * Get list of installed DEX/JAR mods
     */
    public List<File> getDexMods() {
        try {
            File dexDir = getDexModsDir();
            if (!dexDir.exists()) return new ArrayList<>();
            
            File[] files = dexDir.listFiles((dir, name) -> {
                String lower = name.toLowerCase();
                return lower.endsWith(".dex") || lower.endsWith(".jar");
            });
            
            return files != null ? Arrays.asList(files) : new ArrayList<>();
        } catch (Exception e) {
            logError("Failed to get DEX mods: " + e.getMessage());
            return new ArrayList<>();
        }
    }
    
    /**
     * Get list of installed DLL mods
     */
    public List<File> getDllMods() {
        try {
            File dllDir = getDllModsDir();
            if (!dllDir.exists()) return new ArrayList<>();
            
            File[] files = dllDir.listFiles((dir, name) -> 
                name.toLowerCase().endsWith(".dll"));
            
            return files != null ? Arrays.asList(files) : new ArrayList<>();
        } catch (Exception e) {
            logError("Failed to get DLL mods: " + e.getMessage());
            return new ArrayList<>();
        }
    }
    
    /**
     * Check if a mod is enabled
     */
    public boolean isModEnabled(File modFile) {
        return !modFile.getName().toLowerCase().endsWith(".disabled");
    }
    
    /**
     * Enable/disable a mod
     */
    public boolean toggleMod(File modFile, boolean enabled) {
        try {
            String currentName = modFile.getName();
            String newName;
            
            if (enabled && currentName.endsWith(".disabled")) {
                // Enable mod - remove .disabled extension
                newName = currentName.substring(0, currentName.lastIndexOf(".disabled"));
            } else if (!enabled && !currentName.endsWith(".disabled")) {
                // Disable mod - add .disabled extension
                newName = currentName + ".disabled";
            } else {
                // Already in desired state
                return true;
            }
            
            File newFile = new File(modFile.getParent(), newName);
            boolean success = modFile.renameTo(newFile);
            
            if (success) {
                log((enabled ? "Enabled" : "Disabled") + " mod: " + currentName);
            }
            
            return success;
        } catch (Exception e) {
            logError("Failed to toggle mod: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Install a mod from file
     */
    public boolean installMod(File modFile, String targetName) {
        try {
            String fileName = targetName != null ? targetName : modFile.getName();
            File targetDir;
            
            // Determine target directory based on file extension
            if (fileName.toLowerCase().endsWith(".dll")) {
                targetDir = getDllModsDir();
            } else if (fileName.toLowerCase().endsWith(".dex") || fileName.toLowerCase().endsWith(".jar")) {
                targetDir = getDexModsDir();
            } else {
                logError("Unsupported mod file type: " + fileName);
                return false;
            }
            
            if (!createDirectory(targetDir)) {
                logError("Failed to create mod directory: " + targetDir.getAbsolutePath());
                return false;
            }
            
            File targetFile = new File(targetDir, fileName);
            boolean success = copyFile(modFile, targetFile);
            
            if (success) {
                log("Installed mod: " + fileName);
                // Trigger hook for mod installation
                triggerHook(PluginHook.Hooks.MOD_AFTER_LOAD, this, "mod_file", targetFile);
            }
            
            return success;
        } catch (Exception e) {
            logError("Failed to install mod: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Uninstall a mod
     */
    public boolean uninstallMod(File modFile) {
        try {
            // Trigger hook before uninstalling
            triggerHook(PluginHook.Hooks.MOD_BEFORE_UNLOAD, this, "mod_file", modFile);
            
            boolean success = deleteFile(modFile);
            
            if (success) {
                log("Uninstalled mod: " + modFile.getName());
                // Trigger hook after uninstalling
                triggerHook(PluginHook.Hooks.MOD_AFTER_UNLOAD, this, "mod_file", modFile);
            }
            
            return success;
        } catch (Exception e) {
            logError("Failed to uninstall mod: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Get mod count statistics
     */
    public ModStats getModStats() {
        ModStats stats = new ModStats();
        
        // Count DEX mods
        List<File> dexMods = getDexMods();
        File dexDir = getDexModsDir();
        if (dexDir.exists()) {
            File[] allDexFiles = dexDir.listFiles((dir, name) -> {
                String lower = name.toLowerCase();
                return lower.endsWith(".dex") || lower.endsWith(".jar") || 
                       lower.endsWith(".dex.disabled") || lower.endsWith(".jar.disabled");
            });
            
            if (allDexFiles != null) {
                stats.totalDexMods = allDexFiles.length;
                for (File mod : allDexFiles) {
                    if (isModEnabled(mod)) {
                        stats.enabledDexMods++;
                    }
                }
            }
        }
        
        // Count DLL mods
        File dllDir = getDllModsDir();
        if (dllDir.exists()) {
            File[] allDllFiles = dllDir.listFiles((dir, name) -> {
                String lower = name.toLowerCase();
                return lower.endsWith(".dll") || lower.endsWith(".dll.disabled");
            });
            
            if (allDllFiles != null) {
                stats.totalDllMods = allDllFiles.length;
                for (File mod : allDllFiles) {
                    if (isModEnabled(mod)) {
                        stats.enabledDllMods++;
                    }
                }
            }
        }
        
        return stats;
    }
    
    // ===== CONFIGURATION API =====
    
    /**
     * Get plugin's shared preferences
     */
    public SharedPreferences getPluginPreferences() {
        return context.getSharedPreferences("plugin_" + pluginContext.getPlugin().getId(), 
                                           Context.MODE_PRIVATE);
    }
    
    /**
     * Save a plugin setting
     */
    public void setSetting(String key, String value) {
        getPluginPreferences().edit().putString(key, value).apply();
        triggerHook(PluginHook.Hooks.SETTINGS_CHANGE, this, "key", key);
    }
    
    /**
     * Save a plugin setting (boolean)
     */
    public void setSetting(String key, boolean value) {
        getPluginPreferences().edit().putBoolean(key, value).apply();
        triggerHook(PluginHook.Hooks.SETTINGS_CHANGE, this, "key", key);
    }
    
    /**
     * Save a plugin setting (int)
     */
    public void setSetting(String key, int value) {
        getPluginPreferences().edit().putInt(key, value).apply();
        triggerHook(PluginHook.Hooks.SETTINGS_CHANGE, this, "key", key);
    }
    
    /**
     * Save a plugin setting (float)
     */
    public void setSetting(String key, float value) {
        getPluginPreferences().edit().putFloat(key, value).apply();
        triggerHook(PluginHook.Hooks.SETTINGS_CHANGE, this, "key", key);
    }
    
    /**
     * Get a plugin setting
     */
    public String getSetting(String key, String defaultValue) {
        return getPluginPreferences().getString(key, defaultValue);
    }
    
    /**
     * Get a plugin setting (boolean)
     */
    public boolean getSetting(String key, boolean defaultValue) {
        return getPluginPreferences().getBoolean(key, defaultValue);
    }
    
    /**
     * Get a plugin setting (int)
     */
    public int getSetting(String key, int defaultValue) {
        return getPluginPreferences().getInt(key, defaultValue);
    }
    
    /**
     * Get a plugin setting (float)
     */
    public float getSetting(String key, float defaultValue) {
        return getPluginPreferences().getFloat(key, defaultValue);
    }
    
    /**
     * Clear all plugin settings
     */
    public void clearAllSettings() {
        getPluginPreferences().edit().clear().apply();
        triggerHook(PluginHook.Hooks.SETTINGS_RESET, this);
    }
    
    // ===== HOOK SYSTEM API =====
    
    /**
     * Register a hook listener
     */
    public void registerHook(String hookName, PluginHook.HookListener listener) {
        PluginHook.getInstance().registerHook(hookName, listener);
        pluginContext.addHookListener(hookName, listener);
    }
    
    /**
     * Trigger a hook
     */
    public void triggerHook(String hookName, Object source) {
        PluginHook.getInstance().triggerHook(hookName, source);
    }
    
    /**
     * Trigger a hook with data
     */
    public void triggerHook(String hookName, Object source, String key, Object value) {
        PluginHook.trigger(hookName, source, key, value);
    }
    
    /**
     * Trigger a hook with multiple data items
     */
    public void triggerHook(String hookName, Object source, Map<String, Object> data) {
        PluginHook.getInstance().triggerHook(hookName, source, data);
    }
    
    // ===== SYSTEM INFORMATION API =====
    
    /**
     * Check if MelonLoader is installed
     */
    public boolean isMelonLoaderInstalled() {
        return MelonLoaderManager.isMelonLoaderInstalled(context, MelonLoaderManager.TERRARIA_PACKAGE);
    }
    
    /**
     * Check if LemonLoader is installed
     */
    public boolean isLemonLoaderInstalled() {
        return MelonLoaderManager.isLemonLoaderInstalled(context, MelonLoaderManager.TERRARIA_PACKAGE);
    }
    
    /**
     * Get TerrariaLoader version
     */
    public String getTerrariaLoaderVersion() {
        try {
            return context.getPackageManager()
                         .getPackageInfo(context.getPackageName(), 0)
                         .versionName;
        } catch (Exception e) {
            return "Unknown";
        }
    }
    
    /**
     * Get Android version
     */
    public String getAndroidVersion() {
        return android.os.Build.VERSION.RELEASE;
    }
    
    /**
     * Get device information
     */
    public String getDeviceInfo() {
        return android.os.Build.MANUFACTURER + " " + android.os.Build.MODEL;
    }
    
    /**
     * Get plugin system status
     */
    public String getPluginSystemStatus() {
        StringBuilder status = new StringBuilder();
        status.append("=== Plugin System Status ===\n");
        status.append("Plugin: ").append(pluginContext.getPlugin().getName()).append("\n");
        status.append("Version: ").append(pluginContext.getPlugin().getVersion()).append("\n");
        status.append("TerrariaLoader Version: ").append(getTerrariaLoaderVersion()).append("\n");
        status.append("Android Version: ").append(getAndroidVersion()).append("\n");
        status.append("Device: ").append(getDeviceInfo()).append("\n");
        status.append("MelonLoader: ").append(isMelonLoaderInstalled() ? "Installed" : "Not Installed").append("\n");
        
        ModStats stats = getModStats();
        status.append("DEX Mods: ").append(stats.enabledDexMods).append("/").append(stats.totalDexMods).append("\n");
        status.append("DLL Mods: ").append(stats.enabledDllMods).append("/").append(stats.totalDllMods).append("\n");
        
        return status.toString();
    }
    
    // ===== ACTIVITY INTEGRATION API =====
    
    /**
     * Start an activity
     */
    public void startActivity(Class<?> activityClass) {
        try {
            Intent intent = new Intent(context, activityClass);
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            context.startActivity(intent);
        } catch (Exception e) {
            logError("Failed to start activity: " + e.getMessage());
        }
    }
    
    /**
     * Start an activity with extras
     */
    public void startActivity(Class<?> activityClass, Map<String, String> extras) {
        try {
            Intent intent = new Intent(context, activityClass);
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            if (extras != null) {
                for (Map.Entry<String, String> entry : extras.entrySet()) {
                    intent.putExtra(entry.getKey(), entry.getValue());
                }
            }
            context.startActivity(intent);
        } catch (Exception e) {
            logError("Failed to start activity: " + e.getMessage());
        }
    }
    
    // ===== UTILITY METHODS =====
    
    /**
     * Execute a task in background thread
     */
    public void executeAsync(Runnable task) {
        new Thread(() -> {
            try {
                task.run();
            } catch (Exception e) {
                logError("Async task failed: " + e.getMessage());
            }
        }).start();
    }
    
    /**
     * Execute a task on UI thread
     */
    public void executeOnUI(Runnable task) {
        Activity activity = getCurrentActivity();
        if (activity != null) {
            activity.runOnUiThread(task);
        }
    }
    
    /**
     * Schedule a delayed task
     */
    public void scheduleTask(Runnable task, long delayMs) {
        new Thread(() -> {
            try {
                Thread.sleep(delayMs);
                task.run();
            } catch (Exception e) {
                logError("Scheduled task failed: " + e.getMessage());
            }
        }).start();
    }
    
    // ===== HELPER CLASSES =====
    
    /**
     * Mod statistics container
     */
    public static class ModStats {
        public int totalDexMods = 0;
        public int enabledDexMods = 0;
        public int totalDllMods = 0;
        public int enabledDllMods = 0;
        
        public int getTotalMods() {
            return totalDexMods + totalDllMods;
        }
        
        public int getTotalEnabled() {
            return enabledDexMods + enabledDllMods;
        }
        
        public int getTotalDisabled() {
            return getTotalMods() - getTotalEnabled();
        }
        
        @Override
        public String toString() {
            return String.format("ModStats[Total: %d, Enabled: %d, Disabled: %d, DEX: %d/%d, DLL: %d/%d]",
                getTotalMods(), getTotalEnabled(), getTotalDisabled(),
                enabledDexMods, totalDexMods, enabledDllMods, totalDllMods);
        }
    }
}


/ModLoader/app/src/main/java/com/modloader/plugin/PluginContext.java

// File: PluginContext.java - Context and Utilities for Plugins with Hook Management (250+ lines)
// Path: /app/src/main/java/com/modloader/plugin/PluginContext.java

package com.modloader.plugin;

import android.content.Context;
import com.modloader.util.LogUtils;
import com.modloader.util.PathManager;

import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * Plugin Context - Provides context and utilities for plugins
 * Manages plugin lifecycle, configuration, and hook registration
 */
public class PluginContext {
    private static final String TAG = "PluginContext";
    
    private final Plugin plugin;
    private final Context androidContext;
    private final PluginManager pluginManager;
    private final PluginAPI api;
    
    // Plugin directories
    private File dataDir;
    private File configDir;
    private File cacheDir;
    private File logsDir;
    
    // Plugin state
    private boolean initialized = false;
    private boolean enabled = true;
    private long loadTime;
    private final Map<String, Object> contextData = new ConcurrentHashMap<>();
    
    // Hook management
    private final Map<String, List<PluginHook.HookListener>> registeredHooks = new ConcurrentHashMap<>();
    private final List<String> subscribedHooks = new CopyOnWriteArrayList<>();
    
    // Plugin resources
    private final Map<String, Object> resources = new ConcurrentHashMap<>();
    private final List<AutoCloseable> closeableResources = new CopyOnWriteArrayList<>();
    
    // Event listeners
    private final List<PluginEventListener> eventListeners = new CopyOnWriteArrayList<>();
    
    /**
     * Plugin event listener interface
     */
    public interface PluginEventListener {
        void onPluginEvent(PluginEvent event);
    }
    
    /**
     * Plugin event types
     */
    public enum PluginEventType {
        LOADING,
        LOADED,
        ENABLING,
        ENABLED,
        DISABLING,
        DISABLED,
        UNLOADING,
        UNLOADED,
        ERROR,
        HOOK_REGISTERED,
        HOOK_UNREGISTERED,
        CONFIG_CHANGED
    }
    
    /**
     * Plugin event data
     */
    public static class PluginEvent {
        private final PluginEventType type;
        private final PluginContext context;
        private final String message;
        private final Object data;
        private final long timestamp;
        
        public PluginEvent(PluginEventType type, PluginContext context, String message, Object data) {
            this.type = type;
            this.context = context;
            this.message = message;
            this.data = data;
            this.timestamp = System.currentTimeMillis();
        }
        
        public PluginEventType getType() { return type; }
        public PluginContext getContext() { return context; }
        public String getMessage() { return message; }
        public Object getData() { return data; }
        public long getTimestamp() { return timestamp; }
    }
    
    public PluginContext(Plugin plugin, Context androidContext, PluginManager pluginManager) {
        this.plugin = plugin;
        this.androidContext = androidContext;
        this.pluginManager = pluginManager;
        this.api = new PluginAPI(androidContext, this);
        this.loadTime = System.currentTimeMillis();
        
        initializeDirectories();
        fireEvent(PluginEventType.LOADING, "Plugin context created", null);
        
        LogUtils.logDebug("Created plugin context for: " + plugin.getName());
    }
    
    /**
     * Initialize plugin directories
     */
    private void initializeDirectories() {
        try {
            File pluginsBaseDir = getPluginsBaseDirectory();
            String pluginDirName = sanitizeFileName(plugin.getId());
            
            // Create main plugin directory
            File pluginMainDir = new File(pluginsBaseDir, pluginDirName);
            PathManager.ensureDirectoryExists(pluginMainDir);
            
            // Create subdirectories
            dataDir = new File(pluginMainDir, "data");
            configDir = new File(pluginMainDir, "config");
            cacheDir = new File(pluginMainDir, "cache");
            logsDir = new File(pluginMainDir, "logs");
            
            PathManager.ensureDirectoryExists(dataDir);
            PathManager.ensureDirectoryExists(configDir);
            PathManager.ensureDirectoryExists(cacheDir);
            PathManager.ensureDirectoryExists(logsDir);
            
            LogUtils.logDebug("Initialized directories for plugin: " + plugin.getId());
            
        } catch (Exception e) {
            LogUtils.logError("Failed to initialize plugin directories: " + e.getMessage());
        }
    }
    
    /**
     * Get base plugins directory
     */
    private File getPluginsBaseDirectory() {
        File baseDir = PathManager.getGameBaseDir(androidContext, "com.and.games505.TerrariaPaid");
        return new File(baseDir, "Plugins");
    }
    
    /**
     * Sanitize filename for directory creation
     */
    private String sanitizeFileName(String name) {
        return name.replaceAll("[^a-zA-Z0-9._-]", "_").toLowerCase();
    }
    
    // ===== GETTERS =====
    
    public Plugin getPlugin() {
        return plugin;
    }
    
    public Context getAndroidContext() {
        return androidContext;
    }
    
    public PluginManager getPluginManager() {
        return pluginManager;
    }
    
    public PluginAPI getAPI() {
        return api;
    }
    
    public File getDataDir() {
        return dataDir;
    }
    
    public File getConfigDir() {
        return configDir;
    }
    
    public File getCacheDir() {
        return cacheDir;
    }
    
    public File getLogsDir() {
        return logsDir;
    }
    
    public boolean isInitialized() {
        return initialized;
    }
    
    public boolean isEnabled() {
        return enabled;
    }
    
    public long getLoadTime() {
        return loadTime;
    }
    
    // ===== PLUGIN STATE MANAGEMENT =====
    
    /**
     * Initialize the plugin
     */
    public boolean initialize() {
        if (initialized) {
            return true;
        }
        
        try {
            fireEvent(PluginEventType.LOADING, "Initializing plugin", null);
            
            // Call plugin's onLoad method
            plugin.onLoad(this);
            
            initialized = true;
            fireEvent(PluginEventType.LOADED, "Plugin loaded successfully", null);
            
            LogUtils.logUser("Plugin loaded: " + plugin.getName() + " v" + plugin.getVersion());
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Failed to initialize plugin " + plugin.getName() + ": " + e.getMessage());
            fireEvent(PluginEventType.ERROR, "Initialization failed", e);
            return false;
        }
    }
    
    /**
     * Enable the plugin
     */
    public boolean enable() {
        if (!initialized) {
            LogUtils.logError("Cannot enable uninitialized plugin: " + plugin.getName());
            return false;
        }
        
        if (enabled) {
            return true;
        }
        
        try {
            fireEvent(PluginEventType.ENABLING, "Enabling plugin", null);
            
            // Call plugin's onEnable method
            plugin.onEnable(this);
            
            enabled = true;
            fireEvent(PluginEventType.ENABLED, "Plugin enabled successfully", null);
            
            LogUtils.logUser("Plugin enabled: " + plugin.getName());
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Failed to enable plugin " + plugin.getName() + ": " + e.getMessage());
            fireEvent(PluginEventType.ERROR, "Enable failed", e);
            return false;
        }
    }
    
    /**
     * Disable the plugin
     */
    public boolean disable() {
        if (!enabled) {
            return true;
        }
        
        try {
            fireEvent(PluginEventType.DISABLING, "Disabling plugin", null);
            
            // Call plugin's onDisable method
            plugin.onDisable(this);
            
            enabled = false;
            fireEvent(PluginEventType.DISABLED, "Plugin disabled successfully", null);
            
            LogUtils.logUser("Plugin disabled: " + plugin.getName());
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Failed to disable plugin " + plugin.getName() + ": " + e.getMessage());
            fireEvent(PluginEventType.ERROR, "Disable failed", e);
            return false;
        }
    }
    
    /**
     * Unload the plugin
     */
    public boolean unload() {
        try {
            fireEvent(PluginEventType.UNLOADING, "Unloading plugin", null);
            
            // Disable first if enabled
            if (enabled) {
                disable();
            }
            
            // Clean up hooks
            unregisterAllHooks();
            
            // Close resources
            closeAllResources();
            
            // Call plugin's onUnload method
            if (initialized) {
                plugin.onUnload(this);
            }
            
            initialized = false;
            fireEvent(PluginEventType.UNLOADED, "Plugin unloaded successfully", null);
            
            LogUtils.logUser("Plugin unloaded: " + plugin.getName());
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Failed to unload plugin " + plugin.getName() + ": " + e.getMessage());
            fireEvent(PluginEventType.ERROR, "Unload failed", e);
            return false;
        }
    }
    
    // ===== HOOK MANAGEMENT =====
    
    /**
     * Add a hook listener (called by PluginAPI)
     */
    public void addHookListener(String hookName, PluginHook.HookListener listener) {
        registeredHooks.computeIfAbsent(hookName, k -> new CopyOnWriteArrayList<>()).add(listener);
        if (!subscribedHooks.contains(hookName)) {
            subscribedHooks.add(hookName);
        }
        fireEvent(PluginEventType.HOOK_REGISTERED, "Registered hook: " + hookName, hookName);
        LogUtils.logDebug("Plugin " + plugin.getName() + " registered hook: " + hookName);
    }
    
    /**
     * Remove a hook listener
     */
    public void removeHookListener(String hookName, PluginHook.HookListener listener) {
        List<PluginHook.HookListener> listeners = registeredHooks.get(hookName);
        if (listeners != null) {
            listeners.remove(listener);
            if (listeners.isEmpty()) {
                registeredHooks.remove(hookName);
                subscribedHooks.remove(hookName);
            }
            PluginHook.getInstance().unregisterHook(hookName, listener);
            fireEvent(PluginEventType.HOOK_UNREGISTERED, "Unregistered hook: " + hookName, hookName);
            LogUtils.logDebug("Plugin " + plugin.getName() + " unregistered hook: " + hookName);
        }
    }
    
    /**
     * Unregister all hooks for this plugin
     */
    public void unregisterAllHooks() {
        for (Map.Entry<String, List<PluginHook.HookListener>> entry : registeredHooks.entrySet()) {
            String hookName = entry.getKey();
            for (PluginHook.HookListener listener : entry.getValue()) {
                PluginHook.getInstance().unregisterHook(hookName, listener);
            }
        }
        registeredHooks.clear();
        subscribedHooks.clear();
        LogUtils.logDebug("Unregistered all hooks for plugin: " + plugin.getName());
    }
    
    /**
     * Get registered hooks for this plugin
     */
    public Set<String> getRegisteredHooks() {
        return new HashSet<>(subscribedHooks);
    }
    
    // ===== CONTEXT DATA MANAGEMENT =====
    
    /**
     * Store context data
     */
    public void setData(String key, Object value) {
        contextData.put(key, value);
    }
    
    /**
     * Get context data
     */
    @SuppressWarnings("unchecked")
    public <T> T getData(String key, Class<T> type) {
        Object value = contextData.get(key);
        if (value != null && type.isInstance(value)) {
            return (T) value;
        }
        return null;
    }
    
    /**
     * Get context data with default value
     */
    @SuppressWarnings("unchecked")
    public <T> T getData(String key, Class<T> type, T defaultValue) {
        T value = getData(key, type);
        return value != null ? value : defaultValue;
    }
    
    /**
     * Check if context data exists
     */
    public boolean hasData(String key) {
        return contextData.containsKey(key);
    }
    
    /**
     * Remove context data
     */
    public void removeData(String key) {
        contextData.remove(key);
    }
    
    /**
     * Clear all context data
     */
    public void clearData() {
        contextData.clear();
    }
    
    // ===== RESOURCE MANAGEMENT =====
    
    /**
     * Store a resource
     */
    public void setResource(String key, Object resource) {
        resources.put(key, resource);
        if (resource instanceof AutoCloseable) {
            closeableResources.add((AutoCloseable) resource);
        }
    }
    
    /**
     * Get a resource
     */
    @SuppressWarnings("unchecked")
    public <T> T getResource(String key, Class<T> type) {
        Object resource = resources.get(key);
        if (resource != null && type.isInstance(resource)) {
            return (T) resource;
        }
        return null;
    }
    
    /**
     * Remove a resource
     */
    public void removeResource(String key) {
        Object resource = resources.remove(key);
        if (resource instanceof AutoCloseable) {
            closeableResources.remove(resource);
            try {
                ((AutoCloseable) resource).close();
            } catch (Exception e) {
                LogUtils.logDebug("Error closing resource: " + e.getMessage());
            }
        }
    }
    
    /**
     * Close all resources
     */
    private void closeAllResources() {
        for (AutoCloseable resource : closeableResources) {
            try {
                resource.close();
            } catch (Exception e) {
                LogUtils.logDebug("Error closing resource: " + e.getMessage());
            }
        }
        closeableResources.clear();
        resources.clear();
    }
    
    // ===== EVENT SYSTEM =====
    
    /**
     * Add event listener
     */
    public void addEventListener(PluginEventListener listener) {
        eventListeners.add(listener);
    }
    
    /**
     * Remove event listener
     */
    public void removeEventListener(PluginEventListener listener) {
        eventListeners.remove(listener);
    }
    
    /**
     * Fire plugin event
     */
    private void fireEvent(PluginEventType type, String message, Object data) {
        PluginEvent event = new PluginEvent(type, this, message, data);
        
        // Notify internal listeners
        for (PluginEventListener listener : eventListeners) {
            try {
                listener.onPluginEvent(event);
            } catch (Exception e) {
                LogUtils.logDebug("Error in plugin event listener: " + e.getMessage());
            }
        }
        
        // Notify plugin manager
        if (pluginManager != null) {
            pluginManager.onPluginEvent(event);
        }
    }
    
    // ===== LOGGING UTILITIES =====
    
    /**
     * Write to plugin log file
     */
    public void writeToLogFile(String message) {
        try {
            File logFile = new File(logsDir, "plugin.log");
            try (FileWriter writer = new FileWriter(logFile, true)) {
                writer.write(new Date().toString() + " - " + message + "\n");
            }
        } catch (IOException e) {
            LogUtils.logDebug("Failed to write to plugin log: " + e.getMessage());
        }
    }
    
    /**
     * Create plugin-specific log file
     */
    public File createLogFile(String filename) {
        File logFile = new File(logsDir, filename);
        try {
            logFile.createNewFile();
            return logFile;
        } catch (IOException e) {
            LogUtils.logDebug("Failed to create log file: " + e.getMessage());
            return null;
        }
    }
    
    // ===== CONFIGURATION UTILITIES =====
    
    /**
     * Create config file
     */
    public File createConfigFile(String filename) {
        File configFile = new File(configDir, filename);
        try {
            configFile.createNewFile();
            fireEvent(PluginEventType.CONFIG_CHANGED, "Config file created: " + filename, configFile);
            return configFile;
        } catch (IOException e) {
            LogUtils.logDebug("Failed to create config file: " + e.getMessage());
            return null;
        }
    }
    
    /**
     * Get config file
     */
    public File getConfigFile(String filename) {
        return new File(configDir, filename);
    }
    
    /**
     * Check if config file exists
     */
    public boolean hasConfigFile(String filename) {
        return new File(configDir, filename).exists();
    }
    
    // ===== STATUS AND DEBUGGING =====
    
    /**
     * Get plugin context status
     */
    public String getContextStatus() {
        StringBuilder status = new StringBuilder();
        status.append("=== Plugin Context Status ===\n");
        status.append("Plugin: ").append(plugin.getName()).append(" (").append(plugin.getId()).append(")\n");
        status.append("Version: ").append(plugin.getVersion()).append("\n");
        status.append("Initialized: ").append(initialized).append("\n");
        status.append("Enabled: ").append(enabled).append("\n");
        status.append("Load Time: ").append(new Date(loadTime)).append("\n");
        status.append("Uptime: ").append((System.currentTimeMillis() - loadTime) / 1000).append("s\n");
        status.append("Data Directory: ").append(dataDir.getAbsolutePath()).append("\n");
        status.append("Registered Hooks: ").append(subscribedHooks.size()).append("\n");
        status.append("Context Data Items: ").append(contextData.size()).append("\n");
        status.append("Resources: ").append(resources.size()).append("\n");
        status.append("Event Listeners: ").append(eventListeners.size()).append("\n");
        
        if (!subscribedHooks.isEmpty()) {
            status.append("Active Hooks: ").append(String.join(", ", subscribedHooks)).append("\n");
        }
        
        return status.toString();
    }
    
    /**
     * Get plugin statistics
     */
    public Map<String, Object> getStatistics() {
        Map<String, Object> stats = new HashMap<>();
        stats.put("initialized", initialized);
        stats.put("enabled", enabled);
        stats.put("loadTime", loadTime);
        stats.put("uptime", System.currentTimeMillis() - loadTime);
        stats.put("hookCount", subscribedHooks.size());
        stats.put("dataCount", contextData.size());
        stats.put("resourceCount", resources.size());
        stats.put("listenerCount", eventListeners.size());
        return stats;
    }
    
    /**
     * Cleanup method called during plugin unload
     */
    public void cleanup() {
        try {
            unregisterAllHooks();
            closeAllResources();
            contextData.clear();
            eventListeners.clear();
            LogUtils.logDebug("Cleaned up plugin context for: " + plugin.getName());
        } catch (Exception e) {
            LogUtils.logError("Error during plugin context cleanup: " + e.getMessage());
        }
    }
}


/ModLoader/app/src/main/java/com/modloader/plugin/PluginHook.java

// File: PluginHook.java - Hook System for Core App Integration (150+ lines)
// Path: /app/src/main/java/com/modloader/plugin/PluginHook.java

package com.modloader.plugin;

import android.content.Context;
import com.modloader.util.LogUtils;
import java.lang.reflect.Method;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * Plugin Hook System for Core App Integration
 * Provides hooks into TerrariaLoader's core functionality for plugins
 */
public class PluginHook {
    private static final String TAG = "PluginHook";
    
    // Hook priority levels
    public enum Priority {
        LOWEST(0),
        LOW(25),
        NORMAL(50),
        HIGH(75),
        HIGHEST(100);
        
        private final int value;
        Priority(int value) { this.value = value; }
        public int getValue() { return value; }
    }
    
    // Hook listener interface
    public interface HookListener {
        void onEvent(HookEvent event);
        default Priority getPriority() { return Priority.NORMAL; }
    }
    
    // Hook event data container
    public static class HookEvent {
        private final String name;
        private final Object source;
        private final Map<String, Object> data;
        private boolean cancelled = false;
        
        public HookEvent(String name, Object source) {
            this.name = name;
            this.source = source;
            this.data = new HashMap<>();
        }
        
        public String getName() { return name; }
        public Object getSource() { return source; }
        public Map<String, Object> getData() { return data; }
        
        public void putData(String key, Object value) { data.put(key, value); }
        public Object getData(String key) { return data.get(key); }
        
        public boolean isCancelled() { return cancelled; }
        public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
        
        @SuppressWarnings("unchecked")
        public <T> T getData(String key, Class<T> type) {
            Object value = data.get(key);
            if (value != null && type.isInstance(value)) {
                return (T) value;
            }
            return null;
        }
    }
    
    // Hook manager - singleton instance
    private static PluginHook instance;
    private final Map<String, List<HookListener>> hooks = new ConcurrentHashMap<>();
    private final Map<String, Integer> hookCallCounts = new ConcurrentHashMap<>();
    private boolean debugMode = false;
    
    // Core hook names - these are the hooks available to plugins
    public static final class Hooks {
        // Application lifecycle hooks
        public static final String APP_STARTUP = "app.startup";
        public static final String APP_SHUTDOWN = "app.shutdown";
        public static final String ACTIVITY_CREATE = "activity.create";
        public static final String ACTIVITY_RESUME = "activity.resume";
        public static final String ACTIVITY_PAUSE = "activity.pause";
        
        // Mod loading hooks
        public static final String MOD_BEFORE_LOAD = "mod.before_load";
        public static final String MOD_AFTER_LOAD = "mod.after_load";
        public static final String MOD_BEFORE_UNLOAD = "mod.before_unload";
        public static final String MOD_AFTER_UNLOAD = "mod.after_unload";
        public static final String MOD_ENABLE = "mod.enable";
        public static final String MOD_DISABLE = "mod.disable";
        
        // APK processing hooks
        public static final String APK_BEFORE_PATCH = "apk.before_patch";
        public static final String APK_AFTER_PATCH = "apk.after_patch";
        public static final String APK_BEFORE_INSTALL = "apk.before_install";
        public static final String APK_AFTER_INSTALL = "apk.after_install";
        public static final String APK_VALIDATION = "apk.validation";
        
        // Loader system hooks
        public static final String LOADER_BEFORE_INSTALL = "loader.before_install";
        public static final String LOADER_AFTER_INSTALL = "loader.after_install";
        public static final String LOADER_CONFIG_CHANGE = "loader.config_change";
        
        // UI hooks
        public static final String UI_MENU_CREATE = "ui.menu_create";
        public static final String UI_BUTTON_CLICK = "ui.button_click";
        public static final String UI_DIALOG_SHOW = "ui.dialog_show";
        public static final String UI_THEME_CHANGE = "ui.theme_change";
        
        // File system hooks
        public static final String FILE_BEFORE_COPY = "file.before_copy";
        public static final String FILE_AFTER_COPY = "file.after_copy";
        public static final String FILE_BEFORE_DELETE = "file.before_delete";
        public static final String FILE_AFTER_DELETE = "file.after_delete";
        
        // Log system hooks
        public static final String LOG_MESSAGE = "log.message";
        public static final String LOG_ERROR = "log.error";
        public static final String LOG_DEBUG = "log.debug";
        
        // Settings hooks
        public static final String SETTINGS_CHANGE = "settings.change";
        public static final String SETTINGS_RESET = "settings.reset";
        public static final String PERMISSION_CHANGE = "permission.change";
    }
    
    private PluginHook() {
        LogUtils.logDebug("PluginHook system initialized");
    }
    
    public static synchronized PluginHook getInstance() {
        if (instance == null) {
            instance = new PluginHook();
        }
        return instance;
    }
    
    /**
     * Register a hook listener for specific hook
     */
    public void registerHook(String hookName, HookListener listener) {
        if (hookName == null || listener == null) {
            LogUtils.logDebug("Cannot register null hook or listener");
            return;
        }
        
        hooks.computeIfAbsent(hookName, k -> new CopyOnWriteArrayList<>()).add(listener);
        
        // Sort by priority (highest first)
        hooks.get(hookName).sort((a, b) -> 
            Integer.compare(b.getPriority().getValue(), a.getPriority().getValue()));
        
        if (debugMode) {
            LogUtils.logDebug("Registered hook listener for: " + hookName + 
                " (Priority: " + listener.getPriority() + ")");
        }
    }
    
    /**
     * Unregister a hook listener
     */
    public void unregisterHook(String hookName, HookListener listener) {
        List<HookListener> listeners = hooks.get(hookName);
        if (listeners != null) {
            listeners.remove(listener);
            if (listeners.isEmpty()) {
                hooks.remove(hookName);
            }
            if (debugMode) {
                LogUtils.logDebug("Unregistered hook listener for: " + hookName);
            }
        }
    }
    
    /**
     * Trigger a hook with event data
     */
    public HookEvent triggerHook(String hookName, Object source, Map<String, Object> data) {
        HookEvent event = new HookEvent(hookName, source);
        if (data != null) {
            event.getData().putAll(data);
        }
        
        return triggerHook(event);
    }
    
    /**
     * Trigger a hook with just hook name and source
     */
    public HookEvent triggerHook(String hookName, Object source) {
        return triggerHook(hookName, source, null);
    }
    
    /**
     * Trigger a hook with existing event
     */
    public HookEvent triggerHook(HookEvent event) {
        String hookName = event.getName();
        List<HookListener> listeners = hooks.get(hookName);
        
        if (listeners == null || listeners.isEmpty()) {
            if (debugMode) {
                LogUtils.logDebug("No listeners for hook: " + hookName);
            }
            return event;
        }
        
        // Increment call count
        hookCallCounts.put(hookName, hookCallCounts.getOrDefault(hookName, 0) + 1);
        
        if (debugMode) {
            LogUtils.logDebug("Triggering hook: " + hookName + " (" + listeners.size() + " listeners)");
        }
        
        // Call all listeners in priority order
        for (HookListener listener : listeners) {
            try {
                listener.onEvent(event);
                
                // Stop processing if event was cancelled
                if (event.isCancelled()) {
                    if (debugMode) {
                        LogUtils.logDebug("Hook cancelled by listener: " + hookName);
                    }
                    break;
                }
            } catch (Exception e) {
                LogUtils.logError("Hook listener error for " + hookName + ": " + e.getMessage());
            }
        }
        
        return event;
    }
    
    /**
     * Check if a hook has any listeners
     */
    public boolean hasListeners(String hookName) {
        List<HookListener> listeners = hooks.get(hookName);
        return listeners != null && !listeners.isEmpty();
    }
    
    /**
     * Get number of listeners for a hook
     */
    public int getListenerCount(String hookName) {
        List<HookListener> listeners = hooks.get(hookName);
        return listeners != null ? listeners.size() : 0;
    }
    
    /**
     * Get all registered hook names
     */
    public Set<String> getRegisteredHooks() {
        return new HashSet<>(hooks.keySet());
    }
    
    /**
     * Get hook call statistics
     */
    public Map<String, Integer> getHookStats() {
        return new HashMap<>(hookCallCounts);
    }
    
    /**
     * Enable/disable debug mode
     */
    public void setDebugMode(boolean debug) {
        this.debugMode = debug;
        LogUtils.logDebug("PluginHook debug mode: " + debug);
    }
    
    /**
     * Clear all hooks (useful for plugin unloading)
     */
    public void clearAllHooks() {
        hooks.clear();
        hookCallCounts.clear();
        LogUtils.logDebug("Cleared all plugin hooks");
    }
    
    /**
     * Clear hooks for a specific plugin (by listener class)
     */
    public void clearHooksForPlugin(Class<?> pluginClass) {
        int removed = 0;
        for (Map.Entry<String, List<HookListener>> entry : hooks.entrySet()) {
            List<HookListener> listeners = entry.getValue();
            listeners.removeIf(listener -> listener.getClass().equals(pluginClass));
            if (listeners.isEmpty()) {
                hooks.remove(entry.getKey());
            } else {
                removed += listeners.size();
            }
        }
        LogUtils.logDebug("Removed " + removed + " hooks for plugin: " + pluginClass.getSimpleName());
    }
    
    /**
     * Utility method to create hook event with single data item
     */
    public static HookEvent createEvent(String hookName, Object source, String key, Object value) {
        HookEvent event = new HookEvent(hookName, source);
        event.putData(key, value);
        return event;
    }
    
    /**
     * Quick hook trigger for simple events
     */
    public static void trigger(String hookName, Object source) {
        getInstance().triggerHook(hookName, source);
    }
    
    /**
     * Quick hook trigger with one data item
     */
    public static void trigger(String hookName, Object source, String key, Object value) {
        getInstance().triggerHook(createEvent(hookName, source, key, value));
    }
    
    /**
     * Get hook system status
     */
    public String getHookSystemStatus() {
        StringBuilder status = new StringBuilder();
        status.append("=== Plugin Hook System Status ===\n");
        status.append("Total Hooks: ").append(hooks.size()).append("\n");
        status.append("Debug Mode: ").append(debugMode).append("\n");
        status.append("Total Calls: ").append(hookCallCounts.values().stream()
            .mapToInt(Integer::intValue).sum()).append("\n\n");
        
        status.append("Active Hooks:\n");
        for (Map.Entry<String, List<HookListener>> entry : hooks.entrySet()) {
            status.append("• ").append(entry.getKey())
                  .append(" (").append(entry.getValue().size()).append(" listeners, ")
                  .append(hookCallCounts.getOrDefault(entry.getKey(), 0)).append(" calls)\n");
        }
        
        return status.toString();
    }
}


/ModLoader/app/src/main/java/com/modloader/plugin/PluginLoader.java

// File: PluginLoader.java - Part 1 (Core Loading)
// Path: /app/src/main/java/com/modloader/plugin/PluginLoader.java
package com.modloader.plugin;

import android.content.Context;
import com.modloader.util.LogUtils;
import com.modloader.util.FileUtils;

import java.io.*;
import java.lang.reflect.Constructor;
import java.util.*;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;
import java.util.jar.Manifest;
import java.util.zip.ZipEntry;
import java.util.zip.ZipFile;

/**
 * Plugin Loading System for TerrariaLoader
 * Handles dynamic loading of plugin files (.jar, .dex, .plugin)
 */
public class PluginLoader {
    private static final String TAG = "PluginLoader";
    
    private final Context context;
    private final Map<String, ClassLoader> pluginClassLoaders = new HashMap<>();
    private final Set<String> loadedPluginFiles = new HashSet<>();
    
    // Plugin manifest keys
    private static final String MANIFEST_PLUGIN_NAME = "Plugin-Name";
    private static final String MANIFEST_PLUGIN_VERSION = "Plugin-Version";
    private static final String MANIFEST_PLUGIN_MAIN = "Plugin-Main";
    private static final String MANIFEST_PLUGIN_AUTHOR = "Plugin-Author";
    private static final String MANIFEST_PLUGIN_DESCRIPTION = "Plugin-Description";
    private static final String MANIFEST_PLUGIN_DEPENDENCIES = "Plugin-Dependencies";
    private static final String MANIFEST_PLUGIN_PERMISSIONS = "Plugin-Permissions";
    private static final String MANIFEST_PLUGIN_CATEGORY = "Plugin-Category";
    
    public PluginLoader(Context context) {
        this.context = context;
        LogUtils.logDebug("PluginLoader initialized");
    }
    
    /**
     * Analyze a plugin file and extract metadata
     */
    public PluginInfo analyzePlugin(File pluginFile) throws Exception {
        if (pluginFile == null || !pluginFile.exists()) {
            throw new IllegalArgumentException("Plugin file does not exist");
        }
        
        LogUtils.logDebug("Analyzing plugin: " + pluginFile.getName());
        
        String fileName = pluginFile.getName().toLowerCase();
        PluginInfo info = new PluginInfo();
        
        // Set basic file information
        info.setFilePath(pluginFile.getAbsolutePath());
        info.setFileSize(pluginFile.length());
        info.setLastModified(pluginFile.lastModified());
        
        // Determine plugin type based on file extension
        PluginInfo.PluginType type = determinePluginType(fileName);
        info.setType(type);
        
        // Extract metadata based on plugin type
        switch (type) {
            case JAVA_PLUGIN:
                return analyzeJarPlugin(pluginFile, info);
            case DEX_PLUGIN:
                return analyzeDexPlugin(pluginFile, info);
            case CONFIG_PLUGIN:
                return analyzeConfigPlugin(pluginFile, info);
            default:
                return analyzeGenericPlugin(pluginFile, info);
        }
    }
    
    /**
     * Load a plugin from PluginInfo
     */
    public Plugin loadPlugin(PluginInfo info, PluginContext context) throws Exception {
        if (info == null) {
            throw new IllegalArgumentException("PluginInfo cannot be null");
        }
        
        LogUtils.logDebug("Loading plugin: " + info.getName());
        
        File pluginFile = new File(info.getFilePath());
        if (!pluginFile.exists()) {
            throw new FileNotFoundException("Plugin file not found: " + info.getFilePath());
        }
        
        // Check if already loaded
        if (loadedPluginFiles.contains(info.getFilePath())) {
            LogUtils.logDebug("Plugin file already loaded: " + info.getName());
        }
        
        Plugin plugin = null;
        
        switch (info.getType()) {
            case JAVA_PLUGIN:
                plugin = loadJarPlugin(pluginFile, info, context);
                break;
            case DEX_PLUGIN:
                plugin = loadDexPlugin(pluginFile, info, context);
                break;
            case CONFIG_PLUGIN:
                plugin = loadConfigPlugin(pluginFile, info, context);
                break;
            default:
                throw new UnsupportedOperationException("Unsupported plugin type: " + info.getType());
        }
        
        if (plugin != null) {
            loadedPluginFiles.add(info.getFilePath());
            LogUtils.logDebug("Successfully loaded plugin: " + info.getName());
        }
        
        return plugin;
    }
    
    /**
     * Determine plugin type from filename
     */
    private PluginInfo.PluginType determinePluginType(String fileName) {
        if (fileName.endsWith(".jar") || fileName.endsWith(".jar.disabled")) {
            return PluginInfo.PluginType.JAVA_PLUGIN;
        } else if (fileName.endsWith(".dex") || fileName.endsWith(".dex.disabled")) {
            return PluginInfo.PluginType.DEX_PLUGIN;
        } else if (fileName.endsWith(".plugin") || fileName.endsWith(".plugin.disabled")) {
            return PluginInfo.PluginType.CONFIG_PLUGIN;
        } else if (fileName.endsWith(".theme") || fileName.endsWith(".theme.disabled")) {
            return PluginInfo.PluginType.THEME_PLUGIN;
        } else if (fileName.endsWith(".util") || fileName.endsWith(".util.disabled")) {
            return PluginInfo.PluginType.UTILITY_PLUGIN;
        }
        return PluginInfo.PluginType.JAVA_PLUGIN; // Default
    }
    
    /**
     * Analyze JAR plugin file
     */
    private PluginInfo analyzeJarPlugin(File jarFile, PluginInfo info) throws Exception {
        try (JarFile jar = new JarFile(jarFile)) {
            // Read manifest
            Manifest manifest = jar.getManifest();
            if (manifest != null) {
                extractManifestInfo(manifest, info);
            }
            
            // If no manifest info, use filename
            if (info.getName() == null) {
                String baseName = getBaseName(jarFile);
                info.setName(baseName);
                info.setId(baseName.toLowerCase().replaceAll("[^a-z0-9_]", "_"));
            }
            
            // Set defaults if missing
            if (info.getVersion() == null) info.setVersion("1.0.0");
            if (info.getDescription() == null) info.setDescription("TerrariaLoader Plugin");
            if (info.getAuthor() == null) info.setAuthor("Unknown");
            if (info.getCategory() == null) info.setCategory("General");
            
            // Find main class if not specified
            if (info.getMainClass() == null) {
                String mainClass = findMainClass(jar);
                info.setMainClass(mainClass);
            }
            
            LogUtils.logDebug("JAR plugin analyzed: " + info.getName() + " v" + info.getVersion());
            return info;
        }
    }
    
    /**
     * Analyze DEX plugin file
     */
    private PluginInfo analyzeDexPlugin(File dexFile, PluginInfo info) throws Exception {
        String baseName = getBaseName(dexFile);
        
        info.setName(baseName);
        info.setId(baseName.toLowerCase().replaceAll("[^a-z0-9_]", "_"));
        info.setVersion("1.0.0");
        info.setDescription("Android DEX Plugin for TerrariaLoader");
        info.setAuthor("Unknown");
        info.setCategory("Android");
        
        // Try to find main class by analyzing DEX file
        String mainClass = analyzeDexFile(dexFile);
        info.setMainClass(mainClass);
        
        LogUtils.logDebug("DEX plugin analyzed: " + info.getName());
        return info;
    }
    
    /**
     * Analyze configuration-based plugin file
     */
    private PluginInfo analyzeConfigPlugin(File configFile, PluginInfo info) throws Exception {
        Properties props = new Properties();
        try (FileInputStream fis = new FileInputStream(configFile)) {
            props.load(fis);
        }
        
        info.setName(props.getProperty("name", getBaseName(configFile)));
        info.setId(props.getProperty("id", info.getName().toLowerCase().replaceAll("[^a-z0-9_]", "_")));
        info.setVersion(props.getProperty("version", "1.0.0"));
        info.setDescription(props.getProperty("description", "Configuration-based plugin"));
        info.setAuthor(props.getProperty("author", "Unknown"));
        info.setCategory(props.getProperty("category", "Configuration"));
        info.setMainClass(props.getProperty("main_class", "com.modloader.plugin.ConfigPlugin"));
        
        // Parse dependencies
        String deps = props.getProperty("dependencies", "");
        if (!deps.isEmpty()) {
            info.setDependencies(deps.split(","));
        }
        
        // Parse permissions
        String perms = props.getProperty("permissions", "");
        if (!perms.isEmpty()) {
            info.setPermissions(perms.split(","));
        }
        
        LogUtils.logDebug("Config plugin analyzed: " + info.getName());
        return info;
    }
    
    /**
     * Analyze generic plugin file
     */
    private PluginInfo analyzeGenericPlugin(File pluginFile, PluginInfo info) throws Exception {
        String baseName = getBaseName(pluginFile);
        
        info.setName(baseName);
        info.setId(baseName.toLowerCase().replaceAll("[^a-z0-9_]", "_"));
        info.setVersion("1.0.0");
        info.setDescription("Generic TerrariaLoader Plugin");
        info.setAuthor("Unknown");
        info.setCategory("Generic");
        
        LogUtils.logDebug("Generic plugin analyzed: " + info.getName());
        return info;
    }
    
    /**
     * Load JAR plugin
     */
    private Plugin loadJarPlugin(File jarFile, PluginInfo info, PluginContext context) throws Exception {
        LogUtils.logDebug("Loading JAR plugin: " + info.getName());
        
        // Create plugin class loader
        PluginClassLoader classLoader = new PluginClassLoader(jarFile, getClass().getClassLoader());
        pluginClassLoaders.put(info.getId(), classLoader);
        
        // Load main class
        String mainClassName = info.getMainClass();
        if (mainClassName == null) {
            throw new Exception("No main class specified for plugin: " + info.getName());
        }
        
        try {
            Class<?> pluginClass = classLoader.loadClass(mainClassName);
            
            // Check if class implements Plugin interface
            if (!Plugin.class.isAssignableFrom(pluginClass)) {
                throw new Exception("Main class does not implement Plugin interface: " + mainClassName);
            }
            
            // Create plugin instance
            Constructor<?> constructor = pluginClass.getDeclaredConstructor();
            constructor.setAccessible(true);
            Plugin plugin = (Plugin) constructor.newInstance();
            
            LogUtils.logDebug("JAR plugin instance created: " + info.getName());
            return plugin;
            
        } catch (ClassNotFoundException e) {
            throw new Exception("Main class not found: " + mainClassName, e);
        } catch (Exception e) {
            throw new Exception("Failed to instantiate plugin: " + e.getMessage(), e);
        }
    }
    
    /**
     * Load DEX plugin
     */
    private Plugin loadDexPlugin(File dexFile, PluginInfo info, PluginContext context) throws Exception {
        LogUtils.logDebug("Loading DEX plugin: " + info.getName());
        
        // Create DexClassLoader for Android DEX files
        String dexPath = dexFile.getAbsolutePath();
        File optimizedDir = new File(context.getPluginDataDirectory(), info.getId() + "_dex");
        optimizedDir.mkdirs();
        
        dalvik.system.DexClassLoader dexLoader = new dalvik.system.DexClassLoader(
            dexPath,
            optimizedDir.getAbsolutePath(),
            null,
            getClass().getClassLoader()
        );
        
        pluginClassLoaders.put(info.getId(), dexLoader);
        
        // Try to find and load main class
        String mainClassName = info.getMainClass();
        if (mainClassName == null) {
            // Try common plugin class names
            String[] possibleClasses = {
                "com.modloader.plugin.MainPlugin",
                "com.modloader.plugin.Plugin",
                "com.plugin.Main",
                "Plugin",
                "MainPlugin"
            };
            
            for (String className : possibleClasses) {
                try {
                    Class<?> testClass = dexLoader.loadClass(className);
                    if (Plugin.class.isAssignableFrom(testClass)) {
                        mainClassName = className;
                        info.setMainClass(className);
                        break;
                    }
                } catch (ClassNotFoundException e) {
                    // Continue searching
                }
            }
        }
        
        if (mainClassName == null) {
            throw new Exception("No valid main class found for DEX plugin: " + info.getName());
        }
        
        try {
            Class<?> pluginClass = dexLoader.loadClass(mainClassName);
            Constructor<?> constructor = pluginClass.getDeclaredConstructor();
            constructor.setAccessible(true);
            Plugin plugin = (Plugin) constructor.newInstance();
            
            LogUtils.logDebug("DEX plugin instance created: " + info.getName());
            return plugin;
            
        } catch (Exception e) {
            throw new Exception("Failed to instantiate DEX plugin: " + e.getMessage(), e);
        }
    }
    
    /**
     * Load configuration-based plugin
     */
    private Plugin loadConfigPlugin(File configFile, PluginInfo info, PluginContext context) throws Exception {
        LogUtils.logDebug("Loading config plugin: " + info.getName());
        
        // Create a configuration-based plugin wrapper
        return new ConfigBasedPlugin(configFile, info, context);
    }
    
    /**
     * Extract manifest information from JAR file
     */
    private void extractManifestInfo(Manifest manifest, PluginInfo info) {
        if (manifest.getMainAttributes() == null) {
            return;
        }
        
        var attributes = manifest.getMainAttributes();
        
        String name = attributes.getValue(MANIFEST_PLUGIN_NAME);
        if (name != null) {
            info.setName(name);
            info.setId(name.toLowerCase().replaceAll("[^a-z0-9_]", "_"));
        }
        
        String version = attributes.getValue(MANIFEST_PLUGIN_VERSION);
        if (version != null) {
            info.setVersion(version);
        }
        
        String mainClass = attributes.getValue(MANIFEST_PLUGIN_MAIN);
        if (mainClass != null) {
            info.setMainClass(mainClass);
        }
        
        String author = attributes.getValue(MANIFEST_PLUGIN_AUTHOR);
        if (author != null) {
            info.setAuthor(author);
        }
        
        String description = attributes.getValue(MANIFEST_PLUGIN_DESCRIPTION);
        if (description != null) {
            info.setDescription(description);
        }
        
        String category = attributes.getValue(MANIFEST_PLUGIN_CATEGORY);
        if (category != null) {
            info.setCategory(category);
        }
        
        String dependencies = attributes.getValue(MANIFEST_PLUGIN_DEPENDENCIES);
        if (dependencies != null && !dependencies.trim().isEmpty()) {
            info.setDependencies(dependencies.split(","));
        }
        
        String permissions = attributes.getValue(MANIFEST_PLUGIN_PERMISSIONS);
        if (permissions != null && !permissions.trim().isEmpty()) {
            info.setPermissions(permissions.split(","));
        }
    }
    
    /**
     * Find main class in JAR file
     */
    private String findMainClass(JarFile jar) {
        try {
            Enumeration<JarEntry> entries = jar.entries();
            
            // Look for classes that might be the main plugin class
            List<String> candidates = new ArrayList<>();
            
            while (entries.hasMoreElements()) {
                JarEntry entry = entries.nextElement();
                String entryName = entry.getName();
                
                if (entryName.endsWith(".class")) {
                    String className = entryName.substring(0, entryName.length() - 6)
                                              .replace('/', '.');
                    
                    // Skip inner classes and common non-plugin classes
                    if (className.contains("$") || 
                        className.startsWith("android.") ||
                        className.startsWith("java.") ||
                        className.startsWith("javax.")) {
                        continue;
                    }
                    
                    // Prioritize classes with "Plugin" in the name
                    if (className.toLowerCase().contains("plugin")) {
                        candidates.add(0, className); // Add to front
                    } else if (className.toLowerCase().contains("main")) {
                        candidates.add(className);
                    } else {
                        candidates.add(className);
                    }
                }
            }
            
            // Return first candidate
            if (!candidates.isEmpty()) {
                String mainClass = candidates.get(0);
                LogUtils.logDebug("Found potential main class: " + mainClass);
                return mainClass;
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Error finding main class: " + e.getMessage());
        }
        
        return null;
    }
    
    /**
     * Analyze DEX file to find main class
     */
    private String analyzeDexFile(File dexFile) {
        try {
            String baseName = getBaseName(dexFile);
            
            // Generate possible main class names
            String[] possibleNames = {
                "com.modloader.plugin." + capitalize(baseName) + "Plugin",
                "com.plugin." + capitalize(baseName),
                "com.modloader." + baseName + ".MainPlugin",
                "Plugin",
                "MainPlugin",
                capitalize(baseName) + "Plugin"
            };
            
            // Return the first reasonable option
            for (String name : possibleNames) {
                if (name.matches("^[a-zA-Z_][a-zA-Z0-9_.]*$")) {
                    LogUtils.logDebug("Using main class for DEX: " + name);
                    return name;
                }
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Error analyzing DEX file: " + e.getMessage());
        }
        
        return "com.modloader.plugin.MainPlugin"; // Default
    }
    
    /**
     * Get base name from file (without extension)
     */
    private String getBaseName(File file) {
        String name = file.getName();
        
        // Remove .disabled suffix if present
        if (name.endsWith(".disabled")) {
            name = name.substring(0, name.length() - 9);
        }
        
        // Remove file extension
        int lastDot = name.lastIndexOf('.');
        if (lastDot > 0) {
            name = name.substring(0, lastDot);
        }
        
        return name;
    }
    
    /**
     * Capitalize first letter of string
     */
    private String capitalize(String str) {
        if (str == null || str.isEmpty()) {
            return str;
        }
        return str.substring(0, 1).toUpperCase() + str.substring(1);
    }
    
    /**
     * Unload a plugin by ID
     */
    public void unloadPlugin(String pluginId) {
        ClassLoader loader = pluginClassLoaders.get(pluginId);
        if (loader != null) {
            // Close class loader if it's closeable
            if (loader instanceof Closeable) {
                try {
                    ((Closeable) loader).close();
                } catch (IOException e) {
                    LogUtils.logDebug("Error closing plugin class loader: " + e.getMessage());
                }
            }
            
            pluginClassLoaders.remove(pluginId);
            LogUtils.logDebug("Plugin class loader removed: " + pluginId);
        }
        
        // Remove from loaded files tracking
        loadedPluginFiles.removeIf(path -> path.contains(pluginId));
    }
    
    /**
     * Get plugin class loader
     */
    public ClassLoader getPluginClassLoader(String pluginId) {
        return pluginClassLoaders.get(pluginId);
    }
    
    /**
     * Check if plugin file is loaded
     */
    public boolean isPluginFileLoaded(String filePath) {
        return loadedPluginFiles.contains(filePath);
    }


// File: PluginLoader.java - Part 2 (Utilities & Validation)
// Continuation of PluginLoader.java

    /**
     * Custom ClassLoader for JAR plugins
     */
    private static class PluginClassLoader extends ClassLoader implements Closeable {
        private final File jarFile;
        private final Map<String, Class<?>> loadedClasses = new HashMap<>();
        
        public PluginClassLoader(File jarFile, ClassLoader parent) {
            super(parent);
            this.jarFile = jarFile;
        }
        
        @Override
        protected Class<?> findClass(String name) throws ClassNotFoundException {
            // Check if already loaded
            Class<?> loadedClass = loadedClasses.get(name);
            if (loadedClass != null) {
                return loadedClass;
            }
            
            // Try to load from JAR
            try (JarFile jar = new JarFile(jarFile)) {
                String classPath = name.replace('.', '/') + ".class";
                JarEntry entry = jar.getJarEntry(classPath);
                
                if (entry != null) {
                    try (InputStream is = jar.getInputStream(entry);
                         ByteArrayOutputStream baos = new ByteArrayOutputStream()) {
                        
                        byte[] buffer = new byte[8192];
                        int bytesRead;
                        while ((bytesRead = is.read(buffer)) != -1) {
                            baos.write(buffer, 0, bytesRead);
                        }
                        
                        byte[] classBytes = baos.toByteArray();
                        Class<?> clazz = defineClass(name, classBytes, 0, classBytes.length);
                        loadedClasses.put(name, clazz);
                        
                        LogUtils.logDebug("Loaded class from plugin: " + name);
                        return clazz;
                    }
                }
            } catch (IOException e) {
                throw new ClassNotFoundException("Failed to load class from plugin: " + name, e);
            }
            
            // Delegate to parent if not found in plugin
            return super.findClass(name);
        }
        
        @Override
        public void close() throws IOException {
            loadedClasses.clear();
            LogUtils.logDebug("Plugin class loader closed for: " + jarFile.getName());
        }
    }
    
    /**
     * Configuration-based plugin implementation
     */
    private static class ConfigBasedPlugin extends BasePlugin {
        private final File configFile;
        private final Properties config;
        private final PluginInfo pluginInfo;
        
        public ConfigBasedPlugin(File configFile, PluginInfo info, PluginContext context) {
            this.configFile = configFile;
            this.pluginInfo = info;
            this.config = new Properties();
            
            try (FileInputStream fis = new FileInputStream(configFile)) {
                config.load(fis);
            } catch (IOException e) {
                LogUtils.logError("Failed to load config plugin: " + e.getMessage());
            }
        }
        
        @Override
        public PluginInfo getPluginInfo() {
            return pluginInfo;
        }
        
        @Override
        protected void onPluginLoad() {
            logUser("Configuration-based plugin loaded");
            
            // Execute load actions from config
            String loadActions = config.getProperty("load_actions", "");
            if (!loadActions.isEmpty()) {
                executeConfigActions(loadActions);
            }
        }
        
        @Override
        protected void onPluginUnload() {
            logUser("Configuration-based plugin unloaded");
            
            // Execute unload actions from config
            String unloadActions = config.getProperty("unload_actions", "");
            if (!unloadActions.isEmpty()) {
                executeConfigActions(unloadActions);
            }
        }
        
        @Override
        protected void onPluginEnable() {
            logUser("Configuration-based plugin enabled");
            
            // Execute enable actions from config
            String enableActions = config.getProperty("enable_actions", "");
            if (!enableActions.isEmpty()) {
                executeConfigActions(enableActions);
            }
        }
        
        @Override
        protected void onPluginDisable() {
            logUser("Configuration-based plugin disabled");
            
            // Execute disable actions from config
            String disableActions = config.getProperty("disable_actions", "");
            if (!disableActions.isEmpty()) {
                executeConfigActions(disableActions);
            }
        }
        
        private void executeConfigActions(String actions) {
            try {
                String[] actionList = actions.split(";");
                for (String action : actionList) {
                    action = action.trim();
                    if (!action.isEmpty()) {
                        executeConfigAction(action);
                    }
                }
            } catch (Exception e) {
                logError("Failed to execute config actions: " + e.getMessage());
            }
        }
        
        private void executeConfigAction(String action) {
            try {
                if (action.startsWith("log:")) {
                    String message = action.substring(4);
                    logUser(message);
                } else if (action.startsWith("hook:")) {
                    String hookInfo = action.substring(5);
                    String[] parts = hookInfo.split(":");
                    if (parts.length >= 2) {
                        registerConfigHook(parts[0], parts[1]);
                    }
                } else if (action.startsWith("file:")) {
                    String fileOp = action.substring(5);
                    executeFileOperation(fileOp);
                } else {
                    logUser("Unknown config action: " + action);
                }
            } catch (Exception e) {
                logError("Config action failed: " + action + " - " + e.getMessage());
            }
        }
        
        private void registerConfigHook(String hookPoint, String hookAction) {
            registerHook(hookPoint, new PluginHook() {
                @Override
                public boolean onHookCalled(String hookPoint, Object... args) {
                    logUser("Hook triggered: " + hookPoint + " -> " + hookAction);
                    executeConfigAction(hookAction);
                    return true;
                }
                
                @Override
                public String getHookName() {
                    return pluginInfo.getName() + "_" + hookPoint;
                }
            });
        }
        
        private void executeFileOperation(String fileOp) {
            if (fileOp.startsWith("create:")) {
                String fileName = fileOp.substring(7);
                try {
                    File file = new File(getPluginDataDirectory(), fileName);
                    if (!file.exists()) {
                        file.createNewFile();
                        logUser("Created file: " + fileName);
                    }
                } catch (IOException e) {
                    logError("Failed to create file: " + e.getMessage());
                }
            } else if (fileOp.startsWith("delete:")) {
                String fileName = fileOp.substring(7);
                try {
                    File file = new File(getPluginDataDirectory(), fileName);
                    if (file.exists() && file.delete()) {
                        logUser("Deleted file: " + fileName);
                    }
                } catch (Exception e) {
                    logError("Failed to delete file: " + e.getMessage());
                }
            }
        }
    }
    
    /**
     * Plugin Validation Result class
     */
    public static class PluginValidationResult {
        public boolean isValid = false;
        public String errorMessage = null;
        public String successMessage = null;
        public List<String> warnings = new ArrayList<>();
        
        @Override
        public String toString() {
            if (isValid) {
                String msg = successMessage != null ? successMessage : "Valid plugin";
                if (!warnings.isEmpty()) {
                    msg += " (Warnings: " + warnings.size() + ")";
                }
                return msg;
            } else {
                return "Invalid plugin: " + (errorMessage != null ? errorMessage : "Unknown error");
            }
        }
    }
    
    /**
     * Get plugin file validation info
     */
    public PluginValidationResult validatePluginFile(File pluginFile) {
        PluginValidationResult result = new PluginValidationResult();
        result.isValid = false;
        
        try {
            if (!pluginFile.exists()) {
                result.errorMessage = "Plugin file does not exist";
                return result;
            }
            
            if (!pluginFile.canRead()) {
                result.errorMessage = "Plugin file is not readable";
                return result;
            }
            
            if (pluginFile.length() == 0) {
                result.errorMessage = "Plugin file is empty";
                return result;
            }
            
            if (pluginFile.length() > 50 * 1024 * 1024) {
                result.errorMessage = "Plugin file too large (max 50MB)";
                return result;
            }
            
            // Type-specific validation
            String fileName = pluginFile.getName().toLowerCase();
            if (fileName.endsWith(".jar") || fileName.endsWith(".jar.disabled")) {
                result = validateJarFile(pluginFile);
            } else if (fileName.endsWith(".dex") || fileName.endsWith(".dex.disabled")) {
                result = validateDexFile(pluginFile);
            } else if (fileName.endsWith(".plugin") || fileName.endsWith(".plugin.disabled")) {
                result = validateConfigFile(pluginFile);
            } else {
                result.errorMessage = "Unsupported plugin file type";
                return result;
            }
            
        } catch (Exception e) {
            result.errorMessage = "Validation error: " + e.getMessage();
        }
        
        return result;
    }
    
    private PluginValidationResult validateJarFile(File jarFile) {
        PluginValidationResult result = new PluginValidationResult();
        
        try (JarFile jar = new JarFile(jarFile)) {
            result.isValid = true;
            result.successMessage = "Valid JAR plugin file";
            
            // Check for manifest
            if (jar.getManifest() == null) {
                result.warnings.add("No manifest found - using filename for metadata");
            }
            
            // Check for classes
            boolean hasClasses = false;
            Enumeration<JarEntry> entries = jar.entries();
            while (entries.hasMoreElements()) {
                JarEntry entry = entries.nextElement();
                if (entry.getName().endsWith(".class")) {
                    hasClasses = true;
                    break;
                }
            }
            
            if (!hasClasses) {
                result.warnings.add("No .class files found in JAR");
            }
            
        } catch (Exception e) {
            result.isValid = false;
            result.errorMessage = "Invalid JAR file: " + e.getMessage();
        }
        
        return result;
    }
    
    private PluginValidationResult validateDexFile(File dexFile) {
        PluginValidationResult result = new PluginValidationResult();
        
        try (FileInputStream fis = new FileInputStream(dexFile)) {
            byte[] header = new byte[8];
            if (fis.read(header) >= 8) {
                String magic = new String(header, 0, 4);
                if ("dex\n".equals(magic)) {
                    result.isValid = true;
                    result.successMessage = "Valid DEX plugin file";
                } else {
                    result.isValid = false;
                    result.errorMessage = "Invalid DEX magic number";
                }
            } else {
                result.isValid = false;
                result.errorMessage = "DEX file too small to be valid";
            }
            
        } catch (Exception e) {
            result.isValid = false;
            result.errorMessage = "Error reading DEX file: " + e.getMessage();
        }
        
        return result;
    }
    
    private PluginValidationResult validateConfigFile(File configFile) {
        PluginValidationResult result = new PluginValidationResult();
        
        try {
            Properties props = new Properties();
            try (FileInputStream fis = new FileInputStream(configFile)) {
                props.load(fis);
            }
            
            result.isValid = true;
            result.successMessage = "Valid configuration plugin file";
            
            // Check required properties
            if (props.getProperty("name") == null) {
                result.warnings.add("No 'name' property found");
            }
            
            if (props.getProperty("version") == null) {
                result.warnings.add("No 'version' property found");
            }
            
        } catch (Exception e) {
            result.isValid = false;
            result.errorMessage = "Error reading config file: " + e.getMessage();
        }
        
        return result;
    }
    
    /**
     * Get loader statistics
     */
    public LoaderStatistics getStatistics() {
        LoaderStatistics stats = new LoaderStatistics();
        stats.totalClassLoaders = pluginClassLoaders.size();
        stats.loadedPluginFiles = loadedPluginFiles.size();
        
        // Calculate memory usage (approximate)
        stats.estimatedMemoryUsage = pluginClassLoaders.size() * 1024 * 1024; // Rough estimate
        
        return stats;
    }
    
    /**
     * Loader Statistics class
     */
    public static class LoaderStatistics {
        public int totalClassLoaders = 0;
        public int loadedPluginFiles = 0;
        public long estimatedMemoryUsage = 0;
        
        @Override
        public String toString() {
            return String.format("PluginLoader Stats: %d ClassLoaders, %d Files, %s Memory",
                totalClassLoaders, loadedPluginFiles, FileUtils.formatFileSize(estimatedMemoryUsage));
        }
    }
    
    /**
     * Check plugin compatibility
     */
    public boolean checkPluginCompatibility(PluginInfo pluginInfo) {
        try {
            // Check Android version compatibility
            if (android.os.Build.VERSION.SDK_INT < 21) {
                LogUtils.logDebug("Plugin may not be compatible with API level < 21");
                return false;
            }
            
            // Check file type compatibility
            PluginInfo.PluginType type = pluginInfo.getType();
            if (type == PluginInfo.PluginType.DEX_PLUGIN && android.os.Build.VERSION.SDK_INT < 14) {
                LogUtils.logDebug("DEX plugins require API level 14+");
                return false;
            }
            
            // Check dependencies
            String[] dependencies = pluginInfo.getDependencies();
            if (dependencies != null && dependencies.length > 0) {
                LogUtils.logDebug("Plugin has dependencies - compatibility check needed");
            }
            
            return true;
            
        } catch (Exception e) {
            LogUtils.logDebug("Compatibility check failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Get supported plugin types
     */
    public List<PluginInfo.PluginType> getSupportedPluginTypes() {
        List<PluginInfo.PluginType> supported = new ArrayList<>();
        supported.add(PluginInfo.PluginType.JAVA_PLUGIN);
        supported.add(PluginInfo.PluginType.CONFIG_PLUGIN);
        
        // DEX support depends on Android version
        if (android.os.Build.VERSION.SDK_INT >= 14) {
            supported.add(PluginInfo.PluginType.DEX_PLUGIN);
        }
        
        supported.add(PluginInfo.PluginType.THEME_PLUGIN);
        supported.add(PluginInfo.PluginType.UTILITY_PLUGIN);
        
        return supported;
    }
    
    /**
     * Create example plugin template
     */
    public String generatePluginTemplate(String pluginName, PluginInfo.PluginType type) {
        StringBuilder template = new StringBuilder();
        
        switch (type) {
            case JAVA_PLUGIN:
                template.append("// Example Java Plugin Template\n");
                template.append("package com.modloader.plugin;\n\n");
                template.append("public class ").append(capitalize(pluginName)).append("Plugin extends BasePlugin {\n");
                template.append("    \n");
                template.append("    @Override\n");
                template.append("    public PluginInfo getPluginInfo() {\n");
                template.append("        PluginInfo info = new PluginInfo();\n");
                template.append("        info.setName(\"").append(pluginName).append("\");\n");
                template.append("        info.setVersion(\"1.0.0\");\n");
                template.append("        info.setAuthor(\"Your Name\");\n");
                template.append("        info.setDescription(\"Example plugin for TerrariaLoader\");\n");
                template.append("        return info;\n");
                template.append("    }\n");
                template.append("    \n");
                template.append("    @Override\n");
                template.append("    protected void onPluginLoad() {\n");
                template.append("        logUser(\"").append(pluginName).append(" plugin loaded!\");\n");
                template.append("    }\n");
                template.append("    \n");
                template.append("    @Override\n");
                template.append("    protected void onPluginEnable() {\n");
                template.append("        logUser(\"").append(pluginName).append(" plugin enabled!\");\n");
                template.append("    }\n");
                template.append("    \n");
                template.append("    @Override\n");
                template.append("    protected void onPluginDisable() {\n");
                template.append("        logUser(\"").append(pluginName).append(" plugin disabled!\");\n");
                template.append("    }\n");
                template.append("    \n");
                template.append("    @Override\n");
                template.append("    protected void onPluginUnload() {\n");
                template.append("        logUser(\"").append(pluginName).append(" plugin unloaded!\");\n");
                template.append("    }\n");
                template.append("}\n");
                break;
                
            case CONFIG_PLUGIN:
                template.append("# Example Configuration Plugin Template\n");
                template.append("name=").append(pluginName).append("\n");
                template.append("version=1.0.0\n");
                template.append("author=Your Name\n");
                template.append("description=Example configuration-based plugin\n");
                template.append("category=Configuration\n");
                template.append("main_class=com.modloader.plugin.ConfigPlugin\n");
                template.append("\n");
                template.append("# Actions to execute on load\n");
                template.append("load_actions=log:").append(pluginName).append(" plugin loaded!\n");
                template.append("\n");
                template.append("# Actions to execute on enable\n");
                template.append("enable_actions=log:").append(pluginName).append(" plugin enabled!;file:create:enabled.txt\n");
                template.append("\n");
                template.append("# Actions to execute on disable\n");
                template.append("disable_actions=log:").append(pluginName).append(" plugin disabled!;file:delete:enabled.txt\n");
                template.append("\n");
                template.append("# Actions to execute on unload\n");
                template.append("unload_actions=log:").append(pluginName).append(" plugin unloaded!\n");
                break;
                
            default:
                template.append("# Generic plugin template for ").append(pluginName).append("\n");
                template.append("# Plugin type: ").append(type.getDisplayName()).append("\n");
                template.append("# Please refer to documentation for specific implementation details.\n");
                break;
        }
        
        return template.toString();
    }
    
    /**
     * Cleanup plugin loader
     */
    public void cleanup() {
        LogUtils.logDebug("Cleaning up PluginLoader...");
        
        // Close all class loaders
        for (Map.Entry<String, ClassLoader> entry : pluginClassLoaders.entrySet()) {
            ClassLoader loader = entry.getValue();
            if (loader instanceof Closeable) {
                try {
                    ((Closeable) loader).close();
                } catch (IOException e) {
                    LogUtils.logDebug("Error closing class loader: " + e.getMessage());
                }
            }
        }
        
        pluginClassLoaders.clear();
        loadedPluginFiles.clear();
        
        LogUtils.logDebug("PluginLoader cleanup completed");
    }
    
    /**
     * Get loaded plugin files count
     */
    public int getLoadedPluginCount() {
        return loadedPluginFiles.size();
    }
    
    /**
     * Get active class loaders count
     */
    public int getActiveClassLoadersCount() {
        return pluginClassLoaders.size();
    }
    
    /**
     * Check if plugin loader has any plugins loaded
     */
    public boolean hasLoadedPlugins() {
        return !loadedPluginFiles.isEmpty() || !pluginClassLoaders.isEmpty();
    }
    
    /**
     * Get memory usage estimate
     */
    public long getEstimatedMemoryUsage() {
        // Rough estimate: each class loader uses about 1MB
        return pluginClassLoaders.size() * 1024 * 1024;
    }
    
    /**
     * Force garbage collection for plugin resources
     */
    public void forceGarbageCollection() {
        LogUtils.logDebug("Forcing garbage collection for plugin resources");
        System.gc();
    }
    
    /**
     * Get debug information about loaded plugins
     */
    public String getDebugInfo() {
        StringBuilder debug = new StringBuilder();
        debug.append("=== PluginLoader Debug Info ===\n");
        debug.append("Loaded plugin files: ").append(loadedPluginFiles.size()).append("\n");
        debug.append("Active class loaders: ").append(pluginClassLoaders.size()).append("\n");
        debug.append("Estimated memory usage: ").append(FileUtils.formatFileSize(getEstimatedMemoryUsage())).append("\n");
        
        debug.append("\nLoaded files:\n");
        for (String filePath : loadedPluginFiles) {
            debug.append("- ").append(filePath).append("\n");
        }
        
        debug.append("\nClass loaders:\n");
        for (String pluginId : pluginClassLoaders.keySet()) {
            debug.append("- ").append(pluginId).append(" (").append(pluginClassLoaders.get(pluginId).getClass().getSimpleName()).append(")\n");
        }
        
        return debug.toString();
    }
}


/ModLoader/app/src/main/java/com/modloader/plugin/PluginManager.java

// File: PluginManager.java - Part 1 (Core Management)
// Path: /app/src/main/java/com/modloader/plugin/PluginManager.java
package com.modloader.plugin;

import android.content.Context;
import android.content.SharedPreferences;
import com.modloader.util.LogUtils;
import com.modloader.util.PathManager;
import com.modloader.util.FileUtils;

import java.io.File;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

/**
 * Central Plugin Management System for ModLoader
 * Handles plugin discovery, loading, lifecycle management, and API provisioning
 */
public class PluginManager {
    private static final String TAG = "PluginManager";
    private static final String PREFS_NAME = "plugin_manager_prefs";
    private static final String PREF_PLUGINS_ENABLED = "plugins_enabled";
    private static final String PREF_AUTO_DISCOVER = "auto_discover_plugins";
    private static final String PREF_SAFE_MODE = "plugin_safe_mode";
    
    // Plugin file extensions
    private static final String[] PLUGIN_EXTENSIONS = {".jar", ".dex", ".plugin"};
    private static final long MAX_PLUGIN_SIZE = 50 * 1024 * 1024; // 50MB max
    
    // Singleton instance
    private static volatile PluginManager instance;
    private static final Object instanceLock = new Object();
    
    // Core components
    private final Context context;
    private final PluginLoader pluginLoader;
    private final PluginRegistry pluginRegistry;
    private final PluginValidator pluginValidator;
    private final PluginStorage pluginStorage;
    private final ExecutorService executorService;
    
    // Plugin state management
    private final Map<String, Plugin> loadedPlugins = new ConcurrentHashMap<>();
    private final Map<String, PluginInfo> availablePlugins = new ConcurrentHashMap<>();
    private final Map<String, PluginContext> pluginContexts = new ConcurrentHashMap<>();
    private final Set<String> enabledPlugins = ConcurrentHashMap.newKeySet();
    private final Set<String> failedPlugins = ConcurrentHashMap.newKeySet();
    
    // Configuration
    private boolean pluginsEnabled = true;
    private boolean autoDiscoverEnabled = true;
    private boolean safeMode = false;
    private boolean initialized = false;
    
    // Listeners
    private final Set<PluginManagerListener> listeners = ConcurrentHashMap.newKeySet();
    
    /**
     * Plugin Manager Listener Interface
     */
    public interface PluginManagerListener {
        void onPluginLoaded(String pluginId, Plugin plugin);
        void onPluginUnloaded(String pluginId);
        void onPluginEnabled(String pluginId);
        void onPluginDisabled(String pluginId);
        void onPluginError(String pluginId, String error);
        void onPluginDiscovered(String pluginId, PluginInfo info);
    }
    
    private PluginManager(Context context) {
        this.context = context.getApplicationContext();
        this.pluginLoader = new PluginLoader(context);
        this.pluginRegistry = new PluginRegistry(context);
        this.pluginValidator = new PluginValidator(context);
        this.pluginStorage = new PluginStorage(context);
        this.executorService = Executors.newCachedThreadPool();
        
        loadConfiguration();
        LogUtils.logDebug("PluginManager initialized");
    }
    
    /**
     * Get singleton instance of PluginManager
     */
    public static PluginManager getInstance(Context context) {
        if (instance == null) {
            synchronized (instanceLock) {
                if (instance == null) {
                    instance = new PluginManager(context);
                }
            }
        }
        return instance;
    }
    
    /**
     * Initialize the plugin system
     */
    public void initialize() {
        if (initialized) {
            LogUtils.logDebug("PluginManager already initialized");
            return;
        }
        
        LogUtils.logUser("🔌 Initializing Plugin System...");
        
        try {
            // Create plugin directories
            initializePluginDirectories();
            
            // Initialize components
            pluginRegistry.initialize();
            pluginStorage.initialize();
            
            // Load plugin configurations
            loadPluginConfigurations();
            
            // Discover available plugins
            if (autoDiscoverEnabled) {
                discoverPlugins();
            }
            
            // Load enabled plugins
            if (pluginsEnabled) {
                loadEnabledPlugins();
            }
            
            initialized = true;
            LogUtils.logUser("✅ Plugin System initialized successfully");
            LogUtils.logUser("📊 Available plugins: " + availablePlugins.size());
            LogUtils.logUser("📊 Loaded plugins: " + loadedPlugins.size());
            
        } catch (Exception e) {
            LogUtils.logError("Failed to initialize Plugin System: " + e.getMessage());
            LogUtils.logDebug("Plugin initialization error: " + e.toString());
        }
    }
    
    /**
     * Create plugin directory structure
     */
    private void initializePluginDirectories() {
        try {
            File pluginsDir = getPluginsDirectory();
            File configDir = getPluginConfigDirectory();
            File dataDir = getPluginDataDirectory();
            File tempDir = getPluginTempDirectory();
            
            // Create directories
            PathManager.ensureDirectoryExists(pluginsDir);
            PathManager.ensureDirectoryExists(configDir);
            PathManager.ensureDirectoryExists(dataDir);
            PathManager.ensureDirectoryExists(tempDir);
            
            // Create info files
            createPluginDirectoryReadmes();
            
            LogUtils.logDebug("Plugin directories initialized");
            LogUtils.logDebug("Plugins: " + pluginsDir.getAbsolutePath());
            LogUtils.logDebug("Config: " + configDir.getAbsolutePath());
            LogUtils.logDebug("Data: " + dataDir.getAbsolutePath());
            
        } catch (Exception e) {
            LogUtils.logError("Failed to create plugin directories: " + e.getMessage());
        }
    }
    
    /**
     * Create README files for plugin directories
     */
    private void createPluginDirectoryReadmes() {
        try {
            // Main plugins directory README
            File pluginsReadme = new File(getPluginsDirectory(), "README.txt");
            if (!pluginsReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(pluginsReadme)) {
                    writer.write("=== ModLoader Plugins Directory ===\n\n");
                    writer.write("Place your plugin files here:\n");
                    writer.write("• .jar files (Java plugins)\n");
                    writer.write("• .dex files (Android plugins)\n");
                    writer.write("• .plugin files (Configuration-based plugins)\n\n");
                    writer.write("Plugin files can be:\n");
                    writer.write("• plugin_name.jar (enabled)\n");
                    writer.write("• plugin_name.jar.disabled (disabled)\n\n");
                    writer.write("Plugins provide additional functionality like:\n");
                    writer.write("• Custom UI themes\n");
                    writer.write("• New mod formats\n");
                    writer.write("• Enhanced features\n");
                    writer.write("• Bug fixes and patches\n\n");
                    writer.write("Path: " + pluginsReadme.getAbsolutePath() + "\n");
                }
            }
            
            // Config directory README
            File configReadme = new File(getPluginConfigDirectory(), "README.txt");
            if (!configReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(configReadme)) {
                    writer.write("=== Plugin Configuration Directory ===\n\n");
                    writer.write("This directory contains plugin configuration files:\n");
                    writer.write("• plugin_name.json (plugin settings)\n");
                    writer.write("• plugin_name.properties (legacy format)\n");
                    writer.write("• plugin_name.xml (advanced configuration)\n\n");
                    writer.write("Configuration files are automatically created when\n");
                    writer.write("plugins are installed and configured.\n\n");
                    writer.write("Path: " + configReadme.getAbsolutePath() + "\n");
                }
            }
            
            // Data directory README
            File dataReadme = new File(getPluginDataDirectory(), "README.txt");
            if (!dataReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(dataReadme)) {
                    writer.write("=== Plugin Data Directory ===\n\n");
                    writer.write("This directory contains plugin runtime data:\n");
                    writer.write("• Cached files\n");
                    writer.write("• Downloaded resources\n");
                    writer.write("• Temporary files\n");
                    writer.write("• Plugin-specific databases\n\n");
                    writer.write("Files here are managed by individual plugins.\n\n");
                    writer.write("Path: " + dataReadme.getAbsolutePath() + "\n");
                }
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Error creating plugin READMEs: " + e.getMessage());
        }
    }
    
    /**
     * Discover available plugins in the plugins directory
     */
    public void discoverPlugins() {
        LogUtils.logUser("🔍 Discovering plugins...");
        
        try {
            File pluginsDir = getPluginsDirectory();
            if (!pluginsDir.exists()) {
                LogUtils.logDebug("Plugins directory doesn't exist yet");
                return;
            }
            
            File[] pluginFiles = pluginsDir.listFiles(this::isValidPluginFile);
            if (pluginFiles == null || pluginFiles.length == 0) {
                LogUtils.logUser("📋 No plugins found in directory");
                return;
            }
            
            int discoveredCount = 0;
            int validCount = 0;
            
            for (File pluginFile : pluginFiles) {
                try {
                    PluginInfo info = analyzePluginFile(pluginFile);
                    if (info != null) {
                        availablePlugins.put(info.getId(), info);
                        discoveredCount++;
                        
                        // Validate plugin
                        if (pluginValidator.validatePlugin(info)) {
                            validCount++;
                            notifyPluginDiscovered(info.getId(), info);
                        } else {
                            LogUtils.logDebug("Plugin validation failed: " + info.getId());
                        }
                    }
                } catch (Exception e) {
                    LogUtils.logDebug("Error analyzing plugin " + pluginFile.getName() + ": " + e.getMessage());
                }
            }
            
            LogUtils.logUser("🔍 Plugin discovery completed");
            LogUtils.logUser("📊 Discovered: " + discoveredCount + " plugins");
            LogUtils.logUser("📊 Valid: " + validCount + " plugins");
            
        } catch (Exception e) {
            LogUtils.logError("Plugin discovery failed: " + e.getMessage());
        }
    }
    
    /**
     * Load all enabled plugins
     */
    public void loadEnabledPlugins() {
        if (!pluginsEnabled) {
            LogUtils.logDebug("Plugins are disabled globally");
            return;
        }
        
        LogUtils.logUser("🔌 Loading enabled plugins...");
        
        List<Future<Boolean>> loadTasks = new ArrayList<>();
        
        for (PluginInfo info : availablePlugins.values()) {
            if (isPluginEnabled(info.getId()) && !isPluginLoaded(info.getId())) {
                Future<Boolean> task = executorService.submit(() -> loadPlugin(info.getId()));
                loadTasks.add(task);
            }
        }
        
        // Wait for all plugins to load
        int successCount = 0;
        for (Future<Boolean> task : loadTasks) {
            try {
                if (task.get()) {
                    successCount++;
                }
            } catch (Exception e) {
                LogUtils.logDebug("Plugin load task failed: " + e.getMessage());
            }
        }
        
        LogUtils.logUser("🔌 Plugin loading completed");
        LogUtils.logUser("📊 Successfully loaded: " + successCount + "/" + loadTasks.size() + " plugins");
    }
    
    /**
     * Load a specific plugin by ID
     */
    public boolean loadPlugin(String pluginId) {
        if (!pluginsEnabled) {
            LogUtils.logDebug("Plugins are disabled globally");
            return false;
        }
        
        if (isPluginLoaded(pluginId)) {
            LogUtils.logDebug("Plugin already loaded: " + pluginId);
            return true;
        }
        
        PluginInfo info = availablePlugins.get(pluginId);
        if (info == null) {
            LogUtils.logError("Plugin not found: " + pluginId);
            return false;
        }
        
        LogUtils.logUser("🔌 Loading plugin: " + info.getName());
        
        try {
            // Validate plugin before loading
            if (!pluginValidator.validatePlugin(info)) {
                LogUtils.logError("Plugin validation failed: " + pluginId);
                failedPlugins.add(pluginId);
                notifyPluginError(pluginId, "Plugin validation failed");
                return false;
            }
            
            // Create plugin context
            PluginContext pluginContext = new PluginContext(context, info, this);
            pluginContexts.put(pluginId, pluginContext);
            
            // Load plugin using PluginLoader
            Plugin plugin = pluginLoader.loadPlugin(info, pluginContext);
            if (plugin == null) {
                LogUtils.logError("Failed to load plugin: " + pluginId);
                failedPlugins.add(pluginId);
                notifyPluginError(pluginId, "Plugin loading failed");
                return false;
            }
            
            // Initialize plugin
            plugin.onLoad(pluginContext);
            
            // Register plugin
            loadedPlugins.put(pluginId, plugin);
            pluginRegistry.registerPlugin(pluginId, plugin, info);
            enabledPlugins.add(pluginId);
            
            LogUtils.logUser("✅ Plugin loaded successfully: " + info.getName());
            LogUtils.logDebug("Plugin class: " + plugin.getClass().getName());
            
            notifyPluginLoaded(pluginId, plugin);
            notifyPluginEnabled(pluginId);
            
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Plugin loading error for " + pluginId + ": " + e.getMessage());
            LogUtils.logDebug("Plugin loading exception: " + e.toString());
            failedPlugins.add(pluginId);
            notifyPluginError(pluginId, e.getMessage());
            return false;
        }
    }
    
    /**
     * Unload a specific plugin by ID
     */
    public boolean unloadPlugin(String pluginId) {
        Plugin plugin = loadedPlugins.get(pluginId);
        if (plugin == null) {
            LogUtils.logDebug("Plugin not loaded: " + pluginId);
            return false;
        }
        
        LogUtils.logUser("🔌 Unloading plugin: " + pluginId);
        
        try {
            // Call plugin's unload method
            plugin.onUnload();
            
            // Remove from registrations
            loadedPlugins.remove(pluginId);
            pluginRegistry.unregisterPlugin(pluginId);
            pluginContexts.remove(pluginId);
            enabledPlugins.remove(pluginId);
            
            LogUtils.logUser("✅ Plugin unloaded: " + pluginId);
            notifyPluginUnloaded(pluginId);
            
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Plugin unloading error: " + e.getMessage());
            notifyPluginError(pluginId, "Unload failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Enable a plugin (load if not already loaded)
     */
    public boolean enablePlugin(String pluginId) {
        PluginInfo info = availablePlugins.get(pluginId);
        if (info == null) {
            LogUtils.logError("Plugin not found: " + pluginId);
            return false;
        }
        
        // Remove from failed list if present
        failedPlugins.remove(pluginId);
        
        // Enable in preferences
        setPluginEnabled(pluginId, true);
        
        // Load if not already loaded
        if (!isPluginLoaded(pluginId)) {
            return loadPlugin(pluginId);
        } else {
            // Already loaded, just mark as enabled
            enabledPlugins.add(pluginId);
            notifyPluginEnabled(pluginId);
            return true;
        }
    }
    
    /**
     * Disable a plugin (unload if loaded)
     */
    public boolean disablePlugin(String pluginId) {
        // Disable in preferences
        setPluginEnabled(pluginId, false);
        
        // Unload if loaded
        if (isPluginLoaded(pluginId)) {
            return unloadPlugin(pluginId);
        } else {
            // Not loaded, just mark as disabled
            enabledPlugins.remove(pluginId);
            notifyPluginDisabled(pluginId);
            return true;
        }
    }
    
    /**
     * Install a plugin from file
     */
    public boolean installPlugin(File pluginFile) {
        if (pluginFile == null || !pluginFile.exists()) {
            LogUtils.logError("Plugin file does not exist");
            return false;
        }
        
        LogUtils.logUser("📥 Installing plugin: " + pluginFile.getName());
        
        try {
            // Validate file
            if (!isValidPluginFile(pluginFile)) {
                LogUtils.logError("Invalid plugin file: " + pluginFile.getName());
                return false;
            }
            
            // Analyze plugin
            PluginInfo info = analyzePluginFile(pluginFile);
            if (info == null) {
                LogUtils.logError("Failed to analyze plugin file");
                return false;
            }
            
            // Check for conflicts
            if (availablePlugins.containsKey(info.getId())) {
                LogUtils.logUser("⚠️ Plugin already exists: " + info.getId());
                // Create backup of existing plugin
                backupExistingPlugin(info.getId());
            }
            
            // Copy to plugins directory
            File targetFile = new File(getPluginsDirectory(), pluginFile.getName());
            if (!FileUtils.copyFile(pluginFile, targetFile)) {
                LogUtils.logError("Failed to copy plugin file");
                return false;
            }
            
            // Update plugin info with new path
            info.setFilePath(targetFile.getAbsolutePath());
            availablePlugins.put(info.getId(), info);
            
            // Enable by default
            setPluginEnabled(info.getId(), true);
            
            LogUtils.logUser("✅ Plugin installed: " + info.getName());
            LogUtils.logUser("📁 Location: " + targetFile.getAbsolutePath());
            
            notifyPluginDiscovered(info.getId(), info);
            
            // Auto-load if plugins are enabled
            if (pluginsEnabled) {
                loadPlugin(info.getId());
            }
            
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Plugin installation failed: " + e.getMessage());
            return false;
        }
    }
    
    // Directory getters
    public File getPluginsDirectory() {
        return new File(context.getExternalFilesDir(null), "ModLoader/Plugins");
    }
    
    public File getPluginConfigDirectory() {
        return new File(context.getExternalFilesDir(null), "ModLoader/PluginConfig");
    }
    
    public File getPluginDataDirectory() {
        return new File(context.getExternalFilesDir(null), "ModLoader/PluginData");
    }
    
    public File getPluginTempDirectory() {
        return new File(context.getExternalFilesDir(null), "ModLoader/PluginTemp");
    }
    
    // Utility methods
    private boolean isValidPluginFile(File file) {
        if (!file.isFile()) return false;
        if (file.length() == 0 || file.length() > MAX_PLUGIN_SIZE) return false;
        
        String name = file.getName().toLowerCase();
        for (String ext : PLUGIN_EXTENSIONS) {
            if (name.endsWith(ext) || name.endsWith(ext + ".disabled")) {
                return true;
            }
        }
        return false;
    }
    
    private PluginInfo analyzePluginFile(File pluginFile) {
        try {
            return pluginLoader.analyzePlugin(pluginFile);
        } catch (Exception e) {
            LogUtils.logDebug("Failed to analyze plugin: " + e.getMessage());
            return null;
        }
    }
    
    private void loadConfiguration() {
        SharedPreferences prefs = context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE);
        pluginsEnabled = prefs.getBoolean(PREF_PLUGINS_ENABLED, true);
        autoDiscoverEnabled = prefs.getBoolean(PREF_AUTO_DISCOVER, true);
        safeMode = prefs.getBoolean(PREF_SAFE_MODE, false);
        
        LogUtils.logDebug("Plugin configuration loaded - Enabled: " + pluginsEnabled + 
                         ", Auto-discover: " + autoDiscoverEnabled + ", Safe mode: " + safeMode);
    }
    
    // Configuration getters/setters
    public boolean arePluginsEnabled() { return pluginsEnabled; }
    public boolean isAutoDiscoverEnabled() { return autoDiscoverEnabled; }
    public boolean isSafeMode() { return safeMode; }
    public boolean isInitialized() { return initialized; }
    
    public void setPluginsEnabled(boolean enabled) {
        this.pluginsEnabled = enabled;
        SharedPreferences prefs = context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE);
        prefs.edit().putBoolean(PREF_PLUGINS_ENABLED, enabled).apply();
        LogUtils.logUser("Plugin system " + (enabled ? "enabled" : "disabled"));
    }
    
    public void setAutoDiscoverEnabled(boolean enabled) {
        this.autoDiscoverEnabled = enabled;
        SharedPreferences prefs = context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE);
        prefs.edit().putBoolean(PREF_AUTO_DISCOVER, enabled).apply();
    }
    
    public void setSafeMode(boolean enabled) {
        this.safeMode = enabled;
        SharedPreferences prefs = context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE);
        prefs.edit().putBoolean(PREF_SAFE_MODE, enabled).apply();
        LogUtils.logUser("Plugin safe mode " + (enabled ? "enabled" : "disabled"));
    }

// File: PluginManager.java - Part 2 (Operations & Utilities)
// Continuation of PluginManager.java

    /**
     * Uninstall a plugin
     */
    public boolean uninstallPlugin(String pluginId) {
        PluginInfo info = availablePlugins.get(pluginId);
        if (info == null) {
            LogUtils.logError("Plugin not found: " + pluginId);
            return false;
        }
        
        LogUtils.logUser("🗑️ Uninstalling plugin: " + info.getName());
        
        try {
            // Unload if loaded
            if (isPluginLoaded(pluginId)) {
                unloadPlugin(pluginId);
            }
            
            // Remove plugin file
            File pluginFile = new File(info.getFilePath());
            if (pluginFile.exists() && !pluginFile.delete()) {
                LogUtils.logError("Failed to delete plugin file: " + pluginFile.getName());
                return false;
            }
            
            // Remove configuration
            File configFile = new File(getPluginConfigDirectory(), pluginId + ".json");
            if (configFile.exists()) {
                configFile.delete();
            }
            
            // Remove data directory
            File dataDir = new File(getPluginDataDirectory(), pluginId);
            if (dataDir.exists()) {
                FileUtils.deleteDirectory(dataDir);
            }
            
            // Remove from tracking
            availablePlugins.remove(pluginId);
            setPluginEnabled(pluginId, false);
            failedPlugins.remove(pluginId);
            
            LogUtils.logUser("✅ Plugin uninstalled: " + info.getName());
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Plugin uninstallation failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Reload a plugin (unload and load again)
     */
    public boolean reloadPlugin(String pluginId) {
        LogUtils.logUser("🔄 Reloading plugin: " + pluginId);
        
        boolean wasLoaded = isPluginLoaded(pluginId);
        if (wasLoaded) {
            if (!unloadPlugin(pluginId)) {
                LogUtils.logError("Failed to unload plugin for reload: " + pluginId);
                return false;
            }
        }
        
        // Re-discover and re-analyze the plugin
        PluginInfo info = availablePlugins.get(pluginId);
        if (info != null) {
            File pluginFile = new File(info.getFilePath());
            if (pluginFile.exists()) {
                PluginInfo newInfo = analyzePluginFile(pluginFile);
                if (newInfo != null) {
                    availablePlugins.put(pluginId, newInfo);
                }
            }
        }
        
        if (wasLoaded) {
            return loadPlugin(pluginId);
        }
        
        return true;
    }
    
    /**
     * Refresh plugin system (re-discover and reload)
     */
    public void refreshPlugins() {
        LogUtils.logUser("🔄 Refreshing plugin system...");
        
        try {
            // Clear current state
            availablePlugins.clear();
            failedPlugins.clear();
            
            // Re-discover plugins
            discoverPlugins();
            
            // Reload enabled plugins
            loadEnabledPlugins();
            
            LogUtils.logUser("✅ Plugin system refreshed");
            
        } catch (Exception e) {
            LogUtils.logError("Plugin refresh failed: " + e.getMessage());
        }
    }
    
    /**
     * Get plugin context by ID
     */
    public PluginContext getPluginContext(String pluginId) {
        return pluginContexts.get(pluginId);
    }
    
    /**
     * Check if plugin is enabled
     */
    public boolean isPluginEnabled(String pluginId) {
        SharedPreferences prefs = context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE);
        return prefs.getBoolean("plugin_enabled_" + pluginId, false);
    }
    
    /**
     * Set plugin enabled state
     */
    private void setPluginEnabled(String pluginId, boolean enabled) {
        SharedPreferences prefs = context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE);
        prefs.edit().putBoolean("plugin_enabled_" + pluginId, enabled).apply();
    }
    
    /**
     * Get plugin statistics
     */
    public PluginStatistics getStatistics() {
        PluginStatistics stats = new PluginStatistics();
        stats.totalPlugins = availablePlugins.size();
        stats.loadedPlugins = loadedPlugins.size();
        stats.enabledPlugins = enabledPlugins.size();
        stats.failedPlugins = failedPlugins.size();
        stats.disabledPlugins = availablePlugins.size() - enabledPlugins.size();
        stats.pluginStorageUsage = calculatePluginStorageUsage();
        return stats;
    }
    
    /**
     * Plugin Statistics class
     */
    public static class PluginStatistics {
        public int totalPlugins = 0;
        public int loadedPlugins = 0;
        public int enabledPlugins = 0;
        public int disabledPlugins = 0;
        public int failedPlugins = 0;
        public long pluginStorageUsage = 0;
        
        public String getFormattedReport() {
            StringBuilder sb = new StringBuilder();
            sb.append("=== Plugin Statistics ===\n");
            sb.append("Total Plugins: ").append(totalPlugins).append("\n");
            sb.append("Loaded: ").append(loadedPlugins).append("\n");
            sb.append("Enabled: ").append(enabledPlugins).append("\n");
            sb.append("Disabled: ").append(disabledPlugins).append("\n");
            sb.append("Failed: ").append(failedPlugins).append("\n");
            sb.append("Storage Usage: ").append(FileUtils.formatFileSize(pluginStorageUsage)).append("\n");
            return sb.toString();
        }
    }
    
    /**
     * Get failed plugins
     */
    public Set<String> getFailedPlugins() {
        return new HashSet<>(failedPlugins);
    }
    
    /**
     * Clear failed plugins list
     */
    public void clearFailedPlugins() {
        failedPlugins.clear();
        LogUtils.logUser("Cleared failed plugins list");
    }
    
    /**
     * Get plugins by type
     */
    public List<PluginInfo> getPluginsByType(PluginInfo.PluginType type) {
        List<PluginInfo> result = new ArrayList<>();
        for (PluginInfo info : availablePlugins.values()) {
            if (info.getType() == type) {
                result.add(info);
            }
        }
        return result;
    }
    
    /**
     * Get plugins by category
     */
    public List<PluginInfo> getPluginsByCategory(String category) {
        List<PluginInfo> result = new ArrayList<>();
        for (PluginInfo info : availablePlugins.values()) {
            if (category.equals(info.getCategory())) {
                result.add(info);
            }
        }
        return result;
    }
    
    /**
     * Search plugins by name or description
     */
    public List<PluginInfo> searchPlugins(String query) {
        List<PluginInfo> result = new ArrayList<>();
        String lowerQuery = query.toLowerCase();
        
        for (PluginInfo info : availablePlugins.values()) {
            if (info.getName().toLowerCase().contains(lowerQuery) ||
                info.getDescription().toLowerCase().contains(lowerQuery) ||
                info.getId().toLowerCase().contains(lowerQuery)) {
                result.add(info);
            }
        }
        
        return result;
    }
    
    /**
     * Export plugin list to file
     */
    public boolean exportPluginList(File outputFile) {
        try {
            LogUtils.logUser("📤 Exporting plugin list...");
            
            try (java.io.FileWriter writer = new java.io.FileWriter(outputFile)) {
                writer.write("=== TerrariaLoader Plugin List ===\n");
                writer.write("Export Date: " + new java.util.Date().toString() + "\n");
                writer.write("Total Plugins: " + availablePlugins.size() + "\n");
                writer.write("Loaded Plugins: " + loadedPlugins.size() + "\n\n");
                
                for (PluginInfo info : availablePlugins.values()) {
                    writer.write("Plugin: " + info.getName() + "\n");
                    writer.write("  ID: " + info.getId() + "\n");
                    writer.write("  Version: " + info.getVersion() + "\n");
                    writer.write("  Type: " + info.getType() + "\n");
                    writer.write("  Category: " + info.getCategory() + "\n");
                    writer.write("  Status: " + (isPluginLoaded(info.getId()) ? "Loaded" : 
                                 isPluginEnabled(info.getId()) ? "Enabled" : "Disabled") + "\n");
                    writer.write("  File: " + info.getFilePath() + "\n");
                    writer.write("  Description: " + info.getDescription() + "\n\n");
                }
            }
            
            LogUtils.logUser("✅ Plugin list exported: " + outputFile.getName());
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Plugin list export failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Cleanup plugin system
     */
    public void cleanup() {
        LogUtils.logUser("🧹 Cleaning up plugin system...");
        
        try {
            // Unload all plugins
            for (String pluginId : new ArrayList<>(loadedPlugins.keySet())) {
                unloadPlugin(pluginId);
            }
            
            // Clear temporary files
            File tempDir = getPluginTempDirectory();
            if (tempDir.exists()) {
                FileUtils.deleteDirectory(tempDir);
                tempDir.mkdirs();
            }
            
            // Shutdown executor
            executorService.shutdown();
            
            // Clear state
            loadedPlugins.clear();
            pluginContexts.clear();
            enabledPlugins.clear();
            
            LogUtils.logUser("✅ Plugin system cleanup completed");
            
        } catch (Exception e) {
            LogUtils.logError("Plugin cleanup failed: " + e.getMessage());
        }
    }
    
    // Utility methods
    private void loadPluginConfigurations() {
        for (PluginInfo info : availablePlugins.values()) {
            try {
                pluginStorage.loadPluginConfig(info.getId());
            } catch (Exception e) {
                LogUtils.logDebug("Failed to load config for plugin " + info.getId() + ": " + e.getMessage());
            }
        }
    }
    
    private void backupExistingPlugin(String pluginId) {
        try {
            PluginInfo existingInfo = availablePlugins.get(pluginId);
            if (existingInfo != null) {
                File existingFile = new File(existingInfo.getFilePath());
                if (existingFile.exists()) {
                    File backupFile = new File(existingFile.getParent(), 
                                             existingFile.getName() + ".backup." + System.currentTimeMillis());
                    if (existingFile.renameTo(backupFile)) {
                        LogUtils.logUser("📦 Created backup: " + backupFile.getName());
                    }
                }
            }
        } catch (Exception e) {
            LogUtils.logDebug("Plugin backup failed: " + e.getMessage());
        }
    }
    
    private long calculatePluginStorageUsage() {
        long usage = 0;
        try {
            usage += FileUtils.getDirectorySize(getPluginsDirectory());
            usage += FileUtils.getDirectorySize(getPluginConfigDirectory());
            usage += FileUtils.getDirectorySize(getPluginDataDirectory());
        } catch (Exception e) {
            LogUtils.logDebug("Error calculating plugin storage: " + e.getMessage());
        }
        return usage;
    }
    
    // Listener management
    public void addListener(PluginManagerListener listener) {
        listeners.add(listener);
    }
    
    public void removeListener(PluginManagerListener listener) {
        listeners.remove(listener);
    }
    
    // Notification methods
    private void notifyPluginLoaded(String pluginId, Plugin plugin) {
        for (PluginManagerListener listener : listeners) {
            try {
                listener.onPluginLoaded(pluginId, plugin);
            } catch (Exception e) {
                LogUtils.logDebug("Listener notification error: " + e.getMessage());
            }
        }
    }
    
    private void notifyPluginUnloaded(String pluginId) {
        for (PluginManagerListener listener : listeners) {
            try {
                listener.onPluginUnloaded(pluginId);
            } catch (Exception e) {
                LogUtils.logDebug("Listener notification error: " + e.getMessage());
            }
        }
    }
    
    private void notifyPluginEnabled(String pluginId) {
        for (PluginManagerListener listener : listeners) {
            try {
                listener.onPluginEnabled(pluginId);
            } catch (Exception e) {
                LogUtils.logDebug("Listener notification error: " + e.getMessage());
            }
        }
    }
    
    private void notifyPluginDisabled(String pluginId) {
        for (PluginManagerListener listener : listeners) {
            try {
                listener.onPluginDisabled(pluginId);
            } catch (Exception e) {
                LogUtils.logDebug("Listener notification error: " + e.getMessage());
            }
        }
    }
    
    private void notifyPluginError(String pluginId, String error) {
        for (PluginManagerListener listener : listeners) {
            try {
                listener.onPluginError(pluginId, error);
            } catch (Exception e) {
                LogUtils.logDebug("Listener notification error: " + e.getMessage());
            }
        }
    }
    
    private void notifyPluginDiscovered(String pluginId, PluginInfo info) {
        for (PluginManagerListener listener : listeners) {
            try {
                listener.onPluginDiscovered(pluginId, info);
            } catch (Exception e) {
                LogUtils.logDebug("Listener notification error: " + e.getMessage());
            }
        }
    }
    
    /**
     * Get plugins by type
     */
    public List<PluginInfo> getPluginsByType(PluginInfo.PluginType type) {
        List<PluginInfo> result = new ArrayList<>();
        for (PluginInfo info : availablePlugins.values()) {
            if (info.getType() == type) {
                result.add(info);
            }
        }
        return result;
    }
    
    /**
     * Get plugins by category
     */
    public List<PluginInfo> getPluginsByCategory(String category) {
        List<PluginInfo> result = new ArrayList<>();
        for (PluginInfo info : availablePlugins.values()) {
            if (category.equals(info.getCategory())) {
                result.add(info);
            }
        }
        return result;
    }
    
    /**
     * Search plugins by name or description
     */
    public List<PluginInfo> searchPlugins(String query) {
        List<PluginInfo> result = new ArrayList<>();
        String lowerQuery = query.toLowerCase();
        
        for (PluginInfo info : availablePlugins.values()) {
            if (info.getName().toLowerCase().contains(lowerQuery) ||
                info.getDescription().toLowerCase().contains(lowerQuery) ||
                info.getId().toLowerCase().contains(lowerQuery)) {
                result.add(info);
            }
        }
        
        return result;
    }
    
    /**
     * Export plugin list to file
     */
    public boolean exportPluginList(File outputFile) {
        try {
            LogUtils.logUser("📤 Exporting plugin list...");
            
            try (java.io.FileWriter writer = new java.io.FileWriter(outputFile)) {
                writer.write("=== TerrariaLoader Plugin List ===\n");
                writer.write("Export Date: " + new java.util.Date().toString() + "\n");
                writer.write("Total Plugins: " + availablePlugins.size() + "\n");
                writer.write("Loaded Plugins: " + loadedPlugins.size() + "\n\n");
                
                for (PluginInfo info : availablePlugins.values()) {
                    writer.write("Plugin: " + info.getName() + "\n");
                    writer.write("  ID: " + info.getId() + "\n");
                    writer.write("  Version: " + info.getVersion() + "\n");
                    writer.write("  Type: " + info.getType() + "\n");
                    writer.write("  Category: " + info.getCategory() + "\n");
                    writer.write("  Status: " + (isPluginLoaded(info.getId()) ? "Loaded" : 
                                 isPluginEnabled(info.getId()) ? "Enabled" : "Disabled") + "\n");
                    writer.write("  File: " + info.getFilePath() + "\n");
                    writer.write("  Description: " + info.getDescription() + "\n\n");
                }
            }
            
            LogUtils.logUser("✅ Plugin list exported: " + outputFile.getName());
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Plugin list export failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Cleanup plugin system
     */
    public void cleanup() {
        LogUtils.logUser("🧹 Cleaning up plugin system...");
        
        try {
            // Unload all plugins
            for (String pluginId : new ArrayList<>(loadedPlugins.keySet())) {
                unloadPlugin(pluginId);
            }
            
            // Clear temporary files
            File tempDir = getPluginTempDirectory();
            if (tempDir.exists()) {
                FileUtils.deleteDirectory(tempDir);
                tempDir.mkdirs();
            }
            
            // Shutdown executor
            executorService.shutdown();
            
            // Clear state
            loadedPlugins.clear();
            pluginContexts.clear();
            enabledPlugins.clear();
            
            LogUtils.logUser("✅ Plugin system cleanup completed");
            
        } catch (Exception e) {
            LogUtils.logError("Plugin cleanup failed: " + e.getMessage());
        }
    }
    
    /**
     * Get failed plugins
     */
    public Set<String> getFailedPlugins() {
        return new HashSet<>(failedPlugins);
    }
    
    /**
     * Clear failed plugins list
     */
    public void clearFailedPlugins() {
        failedPlugins.clear();
        LogUtils.logUser("Cleared failed plugins list");
    }
    
    /**
     * Get plugin registry
     */
    public PluginRegistry getPluginRegistry() {
        return pluginRegistry;
    }
    
    /**
     * Get plugin storage
     */
    public PluginStorage getPluginStorage() {
        return pluginStorage;
    }
    
    /**
     * Get supported plugin extensions
     */
    public String[] getSupportedExtensions() {
        return PLUGIN_EXTENSIONS.clone();
    }
    
    /**
     * Get maximum plugin file size
     */
    public long getMaxPluginSize() {
        return MAX_PLUGIN_SIZE;
    }
    
    private void loadPluginConfigurations() {
        for (PluginInfo info : availablePlugins.values()) {
            try {
                pluginStorage.loadPluginConfig(info.getId());
            } catch (Exception e) {
                LogUtils.logDebug("Failed to load config for plugin " + info.getId() + ": " + e.getMessage());
            }
        }
    }
    
    private void backupExistingPlugin(String pluginId) {
        try {
            PluginInfo existingInfo = availablePlugins.get(pluginId);
            if (existingInfo != null) {
                File existingFile = new File(existingInfo.getFilePath());
                if (existingFile.exists()) {
                    File backupFile = new File(existingFile.getParent(), 
                                             existingFile.getName() + ".backup." + System.currentTimeMillis());
                    if (existingFile.renameTo(backupFile)) {
                        LogUtils.logUser("📦 Created backup: " + backupFile.getName());
                    }
                }
            }
        } catch (Exception e) {
            LogUtils.logDebug("Plugin backup failed: " + e.getMessage());
        }
    }
    
    private long calculatePluginStorageUsage() {
        long usage = 0;
        try {
            usage += FileUtils.getDirectorySize(getPluginsDirectory());
            usage += FileUtils.getDirectorySize(getPluginConfigDirectory());
            usage += FileUtils.getDirectorySize(getPluginDataDirectory());
        } catch (Exception e) {
            LogUtils.logDebug("Error calculating plugin storage: " + e.getMessage());
        }
        return usage;
    }
    
    /**
     * Validate plugin dependencies
     */
    public boolean checkPluginDependencies(String pluginId) {
        PluginInfo info = getPluginInfo(pluginId);
        if (info == null) {
            return false;
        }
        
        for (String dependency : info.getDependencies()) {
            if (!isPluginLoaded(dependency)) {
                LogUtils.logDebug("Missing dependency for " + pluginId + ": " + dependency);
                return false;
            }
        }
        
        return true;
    }
    
    /**
     * Get plugin dependency tree
     */
    public List<String> getPluginDependencyTree(String pluginId) {
        List<String> dependencies = new ArrayList<>();
        collectDependencies(pluginId, dependencies, new HashSet<>());
        return dependencies;
    }
    
    private void collectDependencies(String pluginId, List<String> dependencies, Set<String> visited) {
        if (visited.contains(pluginId)) {
            return; // Circular dependency protection
        }
        
        visited.add(pluginId);
        PluginInfo info = getPluginInfo(pluginId);
        if (info != null) {
            for (String dependency : info.getDependencies()) {
                if (!dependencies.contains(dependency)) {
                    dependencies.add(dependency);
                    collectDependencies(dependency, dependencies, visited);
                }
            }
        }
    }
    
    /**
     * Shutdown plugin system gracefully
     */
    public void shutdown() {
        LogUtils.logUser("🔌 Shutting down plugin system...");
        
        try {
            // Save all plugin configurations
            for (String pluginId : loadedPlugins.keySet()) {
                try {
                    pluginStorage.savePluginConfig(pluginId);
                } catch (Exception e) {
                    LogUtils.logDebug("Failed to save config for plugin " + pluginId + ": " + e.getMessage());
                }
            }
            
            // Cleanup
            cleanup();
            
            // Clear singleton
            synchronized (instanceLock) {
                instance = null;
            }
            
            LogUtils.logUser("✅ Plugin system shut down successfully");
            
        } catch (Exception e) {
            LogUtils.logError("Plugin shutdown error: " + e.getMessage());
        }
    }
}


/ModLoader/app/src/main/java/com/modloader/plugin/PluginRegistry.java

// File: PluginRegistry.java - Plugin Registration and Lifecycle
// Path: /app/src/main/java/com/modloader/plugin/PluginRegistry.java
package com.modloader.plugin;

import android.content.Context;
import com.modloader.util.LogUtils;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Plugin Registry manages plugin registration, dependencies, and lifecycle coordination
 */
public class PluginRegistry {
    private static final String TAG = "PluginRegistry";
    
    private final Context context;
    private final Map<String, RegistryEntry> registeredPlugins = new ConcurrentHashMap<>();
    private final Map<String, Set<String>> dependencyGraph = new ConcurrentHashMap<>();
    private final Map<String, Set<String>> reverseDependencyGraph = new ConcurrentHashMap<>();
    private final Set<PluginRegistryListener> listeners = ConcurrentHashMap.newKeySet();
    
    // Plugin categories for organization
    private final Map<String, Set<String>> categorizedPlugins = new ConcurrentHashMap<>();
    
    // Plugin lifecycle tracking
    private final Map<String, Plugin.PluginStatus> pluginStatuses = new ConcurrentHashMap<>();
    private final Map<String, Long> pluginLoadTimes = new ConcurrentHashMap<>();
    private final Map<String, String> pluginErrors = new ConcurrentHashMap<>();
    
    /**
     * Registry entry containing plugin information and metadata
     */
    public static class RegistryEntry {
        private final String pluginId;
        private final Plugin plugin;
        private final PluginInfo pluginInfo;
        private final long registrationTime;
        private int priority = 0;
        private Map<String, Object> metadata = new HashMap<>();
        
        public RegistryEntry(String pluginId, Plugin plugin, PluginInfo pluginInfo) {
            this.pluginId = pluginId;
            this.plugin = plugin;
            this.pluginInfo = pluginInfo;
            this.registrationTime = System.currentTimeMillis();
        }
        
        // Getters
        public String getPluginId() { return pluginId; }
        public Plugin getPlugin() { return plugin; }
        public PluginInfo getPluginInfo() { return pluginInfo; }
        public long getRegistrationTime() { return registrationTime; }
        public int getPriority() { return priority; }
        public Map<String, Object> getMetadata() { return metadata; }
        
        // Setters
        public void setPriority(int priority) { this.priority = priority; }
        public void setMetadata(Map<String, Object> metadata) { this.metadata = metadata; }
        public void addMetadata(String key, Object value) { this.metadata.put(key, value); }
        
        @Override
        public String toString() {
            return String.format("RegistryEntry[%s - %s v%s]", 
                pluginId, pluginInfo.getName(), pluginInfo.getVersion());
        }
    }
    
    /**
     * Plugin Registry Listener Interface
     */
    public interface PluginRegistryListener {
        void onPluginRegistered(String pluginId, PluginInfo pluginInfo);
        void onPluginUnregistered(String pluginId);
        void onDependencyResolved(String pluginId, String dependencyId);
        void onDependencyFailed(String pluginId, String dependencyId, String reason);
        void onPluginStatusChanged(String pluginId, Plugin.PluginStatus oldStatus, Plugin.PluginStatus newStatus);
    }
    
    public PluginRegistry(Context context) {
        this.context = context;
        LogUtils.logDebug("PluginRegistry initialized");
    }
    
    /**
     * Initialize the plugin registry
     */
    public void initialize() {
        LogUtils.logDebug("Initializing PluginRegistry");
        
        // Clear any existing state
        registeredPlugins.clear();
        dependencyGraph.clear();
        reverseDependencyGraph.clear();
        categorizedPlugins.clear();
        pluginStatuses.clear();
        pluginLoadTimes.clear();
        pluginErrors.clear();
        
        LogUtils.logDebug("PluginRegistry initialized successfully");
    }
    
    /**
     * Register a plugin in the registry
     */
    public boolean registerPlugin(String pluginId, Plugin plugin, PluginInfo pluginInfo) {
        if (pluginId == null || plugin == null || pluginInfo == null) {
            LogUtils.logError("Cannot register plugin with null parameters");
            return false;
        }
        
        LogUtils.logDebug("Registering plugin: " + pluginId);
        
        try {
            // Check if already registered
            if (registeredPlugins.containsKey(pluginId)) {
                LogUtils.logUser("⚠️ Plugin already registered: " + pluginId);
                return false;
            }
            
            // Create registry entry
            RegistryEntry entry = new RegistryEntry(pluginId, plugin, pluginInfo);
            
            // Register the plugin
            registeredPlugins.put(pluginId, entry);
            
            // Update status tracking
            pluginStatuses.put(pluginId, plugin.getStatus());
            pluginLoadTimes.put(pluginId, System.currentTimeMillis());
            
            // Add to category
            String category = pluginInfo.getCategory();
            if (category != null) {
                categorizedPlugins.computeIfAbsent(category, k -> ConcurrentHashMap.newKeySet()).add(pluginId);
            }
            
            // Process dependencies
            processDependencies(pluginId, pluginInfo);
            
            // Notify listeners
            notifyPluginRegistered(pluginId, pluginInfo);
            
            LogUtils.logUser("✅ Plugin registered: " + pluginInfo.getName());
            LogUtils.logDebug("Plugin ID: " + pluginId);
            LogUtils.logDebug("Plugin category: " + category);
            LogUtils.logDebug("Plugin dependencies: " + Arrays.toString(pluginInfo.getDependencies()));
            
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Plugin registration failed for " + pluginId + ": " + e.getMessage());
            pluginErrors.put(pluginId, e.getMessage());
            return false;
        }
    }
    
    /**
     * Unregister a plugin from the registry
     */
    public boolean unregisterPlugin(String pluginId) {
        if (pluginId == null) {
            LogUtils.logError("Cannot unregister plugin with null ID");
            return false;
        }
        
        LogUtils.logDebug("Unregistering plugin: " + pluginId);
        
        try {
            RegistryEntry entry = registeredPlugins.get(pluginId);
            if (entry == null) {
                LogUtils.logDebug("Plugin not found in registry: " + pluginId);
                return false;
            }
            
            // Check for dependent plugins
            Set<String> dependents = reverseDependencyGraph.get(pluginId);
            if (dependents != null && !dependents.isEmpty()) {
                LogUtils.logUser("⚠️ Plugin has dependents, unregistering may cause issues: " + pluginId);
                for (String dependent : dependents) {
                    LogUtils.logDebug("Dependent plugin: " + dependent);
                }
            }
            
            // Remove from registry
            registeredPlugins.remove(pluginId);
            
            // Clean up tracking
            pluginStatuses.remove(pluginId);
            pluginLoadTimes.remove(pluginId);
            pluginErrors.remove(pluginId);
            
            // Remove from category
            String category = entry.getPluginInfo().getCategory();
            if (category != null) {
                Set<String> categoryPlugins = categorizedPlugins.get(category);
                if (categoryPlugins != null) {
                    categoryPlugins.remove(pluginId);
                    if (categoryPlugins.isEmpty()) {
                        categorizedPlugins.remove(category);
                    }
                }
            }
            
            // Clean up dependency tracking
            cleanupDependencies(pluginId);
            
            // Notify listeners
            notifyPluginUnregistered(pluginId);
            
            LogUtils.logUser("✅ Plugin unregistered: " + pluginId);
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Plugin unregistration failed for " + pluginId + ": " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Get registered plugin by ID
     */
    public Plugin getPlugin(String pluginId) {
        RegistryEntry entry = registeredPlugins.get(pluginId);
        return entry != null ? entry.getPlugin() : null;
    }
    
    /**
     * Get plugin info by ID
     */
    public PluginInfo getPluginInfo(String pluginId) {
        RegistryEntry entry = registeredPlugins.get(pluginId);
        return entry != null ? entry.getPluginInfo() : null;
    }
    
    /**
     * Get registry entry by ID
     */
    public RegistryEntry getRegistryEntry(String pluginId) {
        return registeredPlugins.get(pluginId);
    }
    
    /**
     * Check if plugin is registered
     */
    public boolean isPluginRegistered(String pluginId) {
        return registeredPlugins.containsKey(pluginId);
    }
    
    /**
     * Get all registered plugin IDs
     */
    public Set<String> getRegisteredPluginIds() {
        return new HashSet<>(registeredPlugins.keySet());
    }
    
    /**
     * Get all registered plugins
     */
    public Map<String, Plugin> getAllPlugins() {
        Map<String, Plugin> plugins = new HashMap<>();
        for (Map.Entry<String, RegistryEntry> entry : registeredPlugins.entrySet()) {
            plugins.put(entry.getKey(), entry.getValue().getPlugin());
        }
        return plugins;
    }
    
    /**
     * Get plugins by category
     */
    public Set<String> getPluginsByCategory(String category) {
        Set<String> categoryPlugins = categorizedPlugins.get(category);
        return categoryPlugins != null ? new HashSet<>(categoryPlugins) : new HashSet<>();
    }
    
    /**
     * Get all categories
     */
    public Set<String> getAllCategories() {
        return new HashSet<>(categorizedPlugins.keySet());
    }
    
    /**
     * Process plugin dependencies
     */
    private void processDependencies(String pluginId, PluginInfo pluginInfo) {
        String[] dependencies = pluginInfo.getDependencies();
        if (dependencies == null || dependencies.length == 0) {
            return;
        }
        
        LogUtils.logDebug("Processing dependencies for plugin: " + pluginId);
        
        Set<String> pluginDeps = new HashSet<>();
        for (String dependency : dependencies) {
            dependency = dependency.trim();
            if (!dependency.isEmpty()) {
                pluginDeps.add(dependency);
                
                // Add to reverse dependency graph
                reverseDependencyGraph.computeIfAbsent(dependency, k -> ConcurrentHashMap.newKeySet()).add(pluginId);
                
                // Check if dependency is available
                if (registeredPlugins.containsKey(dependency)) {
                    notifyDependencyResolved(pluginId, dependency);
                    LogUtils.logDebug("Dependency resolved: " + pluginId + " -> " + dependency);
                } else {
                    notifyDependencyFailed(pluginId, dependency, "Dependency not registered");
                    LogUtils.logDebug("Dependency missing: " + pluginId + " -> " + dependency);
                }
            }
        }
        
        if (!pluginDeps.isEmpty()) {
            dependencyGraph.put(pluginId, pluginDeps);
        }
    }
    
    /**
     * Clean up dependencies for a plugin
     */
    private void cleanupDependencies(String pluginId) {
        // Remove from dependency graph
        dependencyGraph.remove(pluginId);
        
        // Remove from reverse dependency graph
        reverseDependencyGraph.remove(pluginId);
        
        // Remove references in other plugins' reverse dependencies
        for (Set<String> dependents : reverseDependencyGraph.values()) {
            dependents.remove(pluginId);
        }
    }
    
    /**
     * Check if plugin dependencies are satisfied
     */
    public boolean areDependenciesSatisfied(String pluginId) {
        Set<String> dependencies = dependencyGraph.get(pluginId);
        if (dependencies == null || dependencies.isEmpty()) {
            return true; // No dependencies
        }
        
        for (String dependency : dependencies) {
            if (!registeredPlugins.containsKey(dependency)) {
                return false;
            }
        }
        
        return true;
    }
    
    /**
     * Get missing dependencies for a plugin
     */
    public Set<String> getMissingDependencies(String pluginId) {
        Set<String> missing = new HashSet<>();
        Set<String> dependencies = dependencyGraph.get(pluginId);
        
        if (dependencies != null) {
            for (String dependency : dependencies) {
                if (!registeredPlugins.containsKey(dependency)) {
                    missing.add(dependency);
                }
            }
        }
        
        return missing;
    }
    
    /**
     * Get plugins that depend on the given plugin
     */
    public Set<String> getDependentPlugins(String pluginId) {
        Set<String> dependents = reverseDependencyGraph.get(pluginId);
        return dependents != null ? new HashSet<>(dependents) : new HashSet<>();
    }
    
    /**
     * Get load order for plugins based on dependencies
     */
    public List<String> getLoadOrder() {
        List<String> loadOrder = new ArrayList<>();
        Set<String> visited = new HashSet<>();
        Set<String> visiting = new HashSet<>();
        
        for (String pluginId : registeredPlugins.keySet()) {
            if (!visited.contains(pluginId)) {
                if (!topologicalSort(pluginId, visited, visiting, loadOrder)) {
                    LogUtils.logError("Circular dependency detected involving plugin: " + pluginId);
                    // Still add remaining plugins in arbitrary order
                    for (String remaining : registeredPlugins.keySet()) {
                        if (!loadOrder.contains(remaining)) {
                            loadOrder.add(remaining);
                        }
                    }
                    break;
                }
            }
        }
        
        return loadOrder;
    }
    
    /**
     * Topological sort for dependency resolution
     */
    private boolean topologicalSort(String pluginId, Set<String> visited, Set<String> visiting, List<String> result) {
        if (visiting.contains(pluginId)) {
            return false; // Circular dependency
        }
        
        if (visited.contains(pluginId)) {
            return true; // Already processed
        }
        
        visiting.add(pluginId);
        
        Set<String> dependencies = dependencyGraph.get(pluginId);
        if (dependencies != null) {
            for (String dependency : dependencies) {
                if (registeredPlugins.containsKey(dependency)) {
                    if (!topologicalSort(dependency, visited, visiting, result)) {
                        return false;
                    }
                }
            }
        }
        
        visiting.remove(pluginId);
        visited.add(pluginId);
        result.add(pluginId);
        
        return true;
    }
    
    /**
     * Update plugin status
     */
    public void updatePluginStatus(String pluginId, Plugin.PluginStatus newStatus) {
        Plugin.PluginStatus oldStatus = pluginStatuses.get(pluginId);
        if (oldStatus != newStatus) {
            pluginStatuses.put(pluginId, newStatus);
            notifyPluginStatusChanged(pluginId, oldStatus, newStatus);
            LogUtils.logDebug("Plugin status changed: " + pluginId + " " + oldStatus + " -> " + newStatus);
        }
    }
    
    /**
     * Get plugin status
     */
    public Plugin.PluginStatus getPluginStatus(String pluginId) {
        return pluginStatuses.get(pluginId);
    }
    
    /**
     * Get plugin load time
     */
    public Long getPluginLoadTime(String pluginId) {
        return pluginLoadTimes.get(pluginId);
    }
    
    /**
     * Set plugin error
     */
    public void setPluginError(String pluginId, String error) {
        pluginErrors.put(pluginId, error);
        LogUtils.logError("Plugin error recorded: " + pluginId + " - " + error);
    }
    
    /**
     * Get plugin error
     */
    public String getPluginError(String pluginId) {
        return pluginErrors.get(pluginId);
    }
    
    /**
     * Clear plugin error
     */
    public void clearPluginError(String pluginId) {
        pluginErrors.remove(pluginId);
    }
    
    /**
     * Get registry statistics
     */
    public RegistryStatistics getStatistics() {
        RegistryStatistics stats = new RegistryStatistics();
        stats.totalRegisteredPlugins = registeredPlugins.size();
        stats.totalCategories = categorizedPlugins.size();
        stats.totalDependencies = dependencyGraph.values().stream()
                .mapToInt(Set::size)
                .sum();
        stats.pluginsWithErrors = pluginErrors.size();
        
        // Count by status
        for (Plugin.PluginStatus status : pluginStatuses.values()) {
            switch (status) {
                case LOADED:
                    stats.loadedPlugins++;
                    break;
                case ENABLED:
                    stats.enabledPlugins++;
                    break;
                case DISABLED:
                    stats.disabledPlugins++;
                    break;
                case ERROR:
                    stats.errorPlugins++;
                    break;
                default:
                    stats.otherStatusPlugins++;
                    break;
            }
        }
        
        return stats;
    }
    
    /**
     * Registry Statistics class
     */
    public static class RegistryStatistics {
        public int totalRegisteredPlugins = 0;
        public int totalCategories = 0;
        public int totalDependencies = 0;
        public int pluginsWithErrors = 0;
        public int loadedPlugins = 0;
        public int enabledPlugins = 0;
        public int disabledPlugins = 0;
        public int errorPlugins = 0;
        public int otherStatusPlugins = 0;
        
        @Override
        public String toString() {
            return String.format("Registry Stats: %d plugins, %d categories, %d dependencies, %d errors",
                totalRegisteredPlugins, totalCategories, totalDependencies, pluginsWithErrors);
        }
        
        public String getDetailedReport() {
            StringBuilder sb = new StringBuilder();
            sb.append("=== Plugin Registry Statistics ===\n");
            sb.append("Total Registered Plugins: ").append(totalRegisteredPlugins).append("\n");
            sb.append("Categories: ").append(totalCategories).append("\n");
            sb.append("Dependencies: ").append(totalDependencies).append("\n");
            sb.append("Plugins with Errors: ").append(pluginsWithErrors).append("\n");
            sb.append("\nPlugin Status Breakdown:\n");
            sb.append("- Loaded: ").append(loadedPlugins).append("\n");
            sb.append("- Enabled: ").append(enabledPlugins).append("\n");
            sb.append("- Disabled: ").append(disabledPlugins).append("\n");
            sb.append("- Error: ").append(errorPlugins).append("\n");
            sb.append("- Other: ").append(otherStatusPlugins).append("\n");
            return sb.toString();
        }
    }
    
    /**
     * Export registry information
     */
    public String exportRegistryInfo() {
        StringBuilder info = new StringBuilder();
        info.append("=== Plugin Registry Export ===\n");
        info.append("Export Time: ").append(new Date().toString()).append("\n");
        info.append("Total Plugins: ").append(registeredPlugins.size()).append("\n\n");
        
        for (Map.Entry<String, RegistryEntry> entry : registeredPlugins.entrySet()) {
            String pluginId = entry.getKey();
            RegistryEntry regEntry = entry.getValue();
            PluginInfo info_inner = regEntry.getPluginInfo();
            
            info.append("Plugin ID: ").append(pluginId).append("\n");
            info.append("  Name: ").append(info_inner.getName()).append("\n");
            info.append("  Version: ").append(info_inner.getVersion()).append("\n");
            info.append("  Author: ").append(info_inner.getAuthor()).append("\n");
            info.append("  Category: ").append(info_inner.getCategory()).append("\n");
            info.append("  Status: ").append(pluginStatuses.get(pluginId)).append("\n");
            info.append("  Registration Time: ").append(new Date(regEntry.getRegistrationTime())).append("\n");
            
            Set<String> dependencies = dependencyGraph.get(pluginId);
            if (dependencies != null && !dependencies.isEmpty()) {
                info.append("  Dependencies: ").append(dependencies).append("\n");
            }
            
            Set<String> dependents = reverseDependencyGraph.get(pluginId);
            if (dependents != null && !dependents.isEmpty()) {
                info.append("  Dependents: ").append(dependents).append("\n");
            }
            
            String error = pluginErrors.get(pluginId);
            if (error != null) {
                info.append("  Error: ").append(error).append("\n");
            }
            
            info.append("\n");
        }
        
        return info.toString();
    }
    
    /**
     * Validate registry consistency
     */
    public List<String> validateRegistry() {
        List<String> issues = new ArrayList<>();
        
        // Check for orphaned dependencies
        for (Map.Entry<String, Set<String>> entry : dependencyGraph.entrySet()) {
            String pluginId = entry.getKey();
            Set<String> dependencies = entry.getValue();
            
            if (!registeredPlugins.containsKey(pluginId)) {
                issues.add("Dependency graph contains unregistered plugin: " + pluginId);
                continue;
            }
            
            for (String dependency : dependencies) {
                if (!registeredPlugins.containsKey(dependency)) {
                    issues.add("Plugin " + pluginId + " depends on unregistered plugin: " + dependency);
                }
            }
        }
        
        // Check for orphaned reverse dependencies
        for (Map.Entry<String, Set<String>> entry : reverseDependencyGraph.entrySet()) {
            String pluginId = entry.getKey();
            Set<String> dependents = entry.getValue();
            
            for (String dependent : dependents) {
                if (!registeredPlugins.containsKey(dependent)) {
                    issues.add("Reverse dependency graph contains unregistered plugin: " + dependent);
                }
            }
        }
        
        // Check for circular dependencies
        try {
            getLoadOrder();
        } catch (Exception e) {
            issues.add("Circular dependency detected in registry");
        }
        
        return issues;
    }
    
    // Listener management
    public void addListener(PluginRegistryListener listener) {
        listeners.add(listener);
    }
    
    public void removeListener(PluginRegistryListener listener) {
        listeners.remove(listener);
    }
    
    // Notification methods
    private void notifyPluginRegistered(String pluginId, PluginInfo pluginInfo) {
        for (PluginRegistryListener listener : listeners) {
            try {
                listener.onPluginRegistered(pluginId, pluginInfo);
            } catch (Exception e) {
                LogUtils.logDebug("Registry listener error: " + e.getMessage());
            }
        }
    }
    
    private void notifyPluginUnregistered(String pluginId) {
        for (PluginRegistryListener listener : listeners) {
            try {
                listener.onPluginUnregistered(pluginId);
            } catch (Exception e) {
                LogUtils.logDebug("Registry listener error: " + e.getMessage());
            }
        }
    }
    
    private void notifyDependencyResolved(String pluginId, String dependencyId) {
        for (PluginRegistryListener listener : listeners) {
            try {
                listener.onDependencyResolved(pluginId, dependencyId);
            } catch (Exception e) {
                LogUtils.logDebug("Registry listener error: " + e.getMessage());
            }
        }
    }
    
    private void notifyDependencyFailed(String pluginId, String dependencyId, String reason) {
        for (PluginRegistryListener listener : listeners) {
            try {
                listener.onDependencyFailed(pluginId, dependencyId, reason);
            } catch (Exception e) {
                LogUtils.logDebug("Registry listener error: " + e.getMessage());
            }
        }
    }
    
    private void notifyPluginStatusChanged(String pluginId, Plugin.PluginStatus oldStatus, Plugin.PluginStatus newStatus) {
        for (PluginRegistryListener listener : listeners) {
            try {
                listener.onPluginStatusChanged(pluginId, oldStatus, newStatus);
            } catch (Exception e) {
                LogUtils.logDebug("Registry listener error: " + e.getMessage());
            }
        }
    }
    
    /**
     * Cleanup registry
     */
    public void cleanup() {
        LogUtils.logDebug("Cleaning up PluginRegistry");
        
        registeredPlugins.clear();
        dependencyGraph.clear();
        reverseDependencyGraph.clear();
        categorizedPlugins.clear();
        pluginStatuses.clear();
        pluginLoadTimes.clear();
        pluginErrors.clear();
        listeners.clear();
        
        LogUtils.logDebug("PluginRegistry cleanup completed");
    }
}


/ModLoader/app/src/main/java/com/modloader/plugin/PluginStorage.java

// File: PluginStorage.java - Plugin Data Persistence (150+ lines)
// Path: /app/src/main/java/com/modloader/plugin/PluginStorage.java

package com.modloader.plugin;

import android.content.Context;
import com.modloader.util.LogUtils;
import com.modloader.util.PathManager;

import java.io.*;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Plugin Storage - Manages plugin data persistence
 * Provides database-like storage for plugin data with JSON serialization
 */
public class PluginStorage {
    private static final String TAG = "PluginStorage";
    private static final String STORAGE_FILE_EXTENSION = ".json";
    private static final String BACKUP_EXTENSION = ".backup";
    
    private final Context context;
    private final String pluginId;
    private final File storageDir;
    private final File storageFile;
    private final Object lock = new Object();
    
    // In-memory cache for better performance
    private final Map<String, Object> dataCache = new ConcurrentHashMap<>();
    private boolean cacheLoaded = false;
    private long lastSaveTime = 0;
    
    public PluginStorage(Context context, String pluginId) {
        this.context = context;
        this.pluginId = pluginId;
        
        // Create storage directory
        File baseDir = PathManager.getGameBaseDir(context, "com.and.games505.TerrariaPaid");
        storageDir = new File(baseDir, "Plugins/" + sanitizePluginId(pluginId) + "/storage");
        PathManager.ensureDirectoryExists(storageDir);
        
        // Main storage file
        storageFile = new File(storageDir, "data" + STORAGE_FILE_EXTENSION);
        
        LogUtils.logDebug("Created plugin storage for: " + pluginId);
    }
    
    /**
     * Sanitize plugin ID for use in file paths
     */
    private String sanitizePluginId(String id) {
        return id.replaceAll("[^a-zA-Z0-9._-]", "_").toLowerCase();
    }
    
    /**
     * Store a value with a key
     */
    public void put(String key, Object value) {
        synchronized (lock) {
            ensureCacheLoaded();
            
            if (value == null) {
                dataCache.remove(key);
            } else {
                dataCache.put(key, value);
            }
            
            // Auto-save after modifications
            scheduleSave();
        }
    }
    
    /**
     * Store multiple values at once
     */
    public void putAll(Map<String, Object> data) {
        synchronized (lock) {
            ensureCacheLoaded();
            
            for (Map.Entry<String, Object> entry : data.entrySet()) {
                if (entry.getValue() == null) {
                    dataCache.remove(entry.getKey());
                } else {
                    dataCache.put(entry.getKey(), entry.getValue());
                }
            }
            
            scheduleSave();
        }
    }
    
    /**
     * Get a value by key
     */
    @SuppressWarnings("unchecked")
    public <T> T get(String key, Class<T> type) {
        synchronized (lock) {
            ensureCacheLoaded();
            Object value = dataCache.get(key);
            
            if (value != null && type.isInstance(value)) {
                return (T) value;
            }
            return null;
        }
    }
    
    /**
     * Get a value with default fallback
     */
    public <T> T get(String key, Class<T> type, T defaultValue) {
        T value = get(key, type);
        return value != null ? value : defaultValue;
    }
    
    /**
     * Get string value
     */
    public String getString(String key) {
        return get(key, String.class);
    }
    
    /**
     * Get string value with default
     */
    public String getString(String key, String defaultValue) {
        return get(key, String.class, defaultValue);
    }
    
    /**
     * Get integer value
     */
    public int getInt(String key, int defaultValue) {
        Integer value = get(key, Integer.class);
        return value != null ? value : defaultValue;
    }
    
    /**
     * Get boolean value
     */
    public boolean getBoolean(String key, boolean defaultValue) {
        Boolean value = get(key, Boolean.class);
        return value != null ? value : defaultValue;
    }
    
    /**
     * Get long value
     */
    public long getLong(String key, long defaultValue) {
        Long value = get(key, Long.class);
        return value != null ? value : defaultValue;
    }
    
    /**
     * Get double value
     */
    public double getDouble(String key, double defaultValue) {
        Double value = get(key, Double.class);
        return value != null ? value : defaultValue;
    }
    
    /**
     * Check if key exists
     */
    public boolean contains(String key) {
        synchronized (lock) {
            ensureCacheLoaded();
            return dataCache.containsKey(key);
        }
    }
    
    /**
     * Remove a key
     */
    public void remove(String key) {
        synchronized (lock) {
            ensureCacheLoaded();
            dataCache.remove(key);
            scheduleSave();
        }
    }
    
    /**
     * Clear all data
     */
    public void clear() {
        synchronized (lock) {
            dataCache.clear();
            scheduleSave();
        }
    }
    
    /**
     * Get all keys
     */
    public Set<String> getKeys() {
        synchronized (lock) {
            ensureCacheLoaded();
            return new HashSet<>(dataCache.keySet());
        }
    }
    
    /**
     * Get all data as a map
     */
    public Map<String, Object> getAllData() {
        synchronized (lock) {
            ensureCacheLoaded();
            return new HashMap<>(dataCache);
        }
    }
    
    /**
     * Get data size
     */
    public int size() {
        synchronized (lock) {
            ensureCacheLoaded();
            return dataCache.size();
        }
    }
    
    /**
     * Check if storage is empty
     */
    public boolean isEmpty() {
        synchronized (lock) {
            ensureCacheLoaded();
            return dataCache.isEmpty();
        }
    }
    
    /**
     * Force save to disk
     */
    public boolean save() {
        synchronized (lock) {
            return saveToFile();
        }
    }
    
    /**
     * Force reload from disk
     */
    public boolean reload() {
        synchronized (lock) {
            cacheLoaded = false;
            dataCache.clear();
            return loadFromFile();
        }
    }
    
    /**
     * Create a backup of current data
     */
    public boolean backup() {
        synchronized (lock) {
            try {
                if (!storageFile.exists()) {
                    return true; // Nothing to backup
                }
                
                File backupFile = new File(storageDir, "data_" + System.currentTimeMillis() + BACKUP_EXTENSION);
                return copyFile(storageFile, backupFile);
                
            } catch (Exception e) {
                LogUtils.logError("Failed to create backup for plugin " + pluginId + ": " + e.getMessage());
                return false;
            }
        }
    }
    
    /**
     * Restore from a backup file
     */
    public boolean restoreFromBackup(File backupFile) {
        synchronized (lock) {
            try {
                if (!backupFile.exists()) {
                    return false;
                }
                
                if (copyFile(backupFile, storageFile)) {
                    return reload();
                }
                return false;
                
            } catch (Exception e) {
                LogUtils.logError("Failed to restore backup for plugin " + pluginId + ": " + e.getMessage());
                return false;
            }
        }
    }
    
    /**
     * Get available backup files
     */
    public List<File> getBackupFiles() {
        File[] files = storageDir.listFiles((dir, name) -> name.endsWith(BACKUP_EXTENSION));
        if (files == null) {
            return new ArrayList<>();
        }
        
        List<File> backups = Arrays.asList(files);
        backups.sort((a, b) -> Long.compare(b.lastModified(), a.lastModified())); // Newest first
        return backups;
    }
    
    /**
     * Clean up old backup files (keep only the newest N backups)
     */
    public void cleanupOldBackups(int keepCount) {
        List<File> backups = getBackupFiles();
        if (backups.size() <= keepCount) {
            return;
        }
        
        for (int i = keepCount; i < backups.size(); i++) {
            File backup = backups.get(i);
            if (backup.delete()) {
                LogUtils.logDebug("Deleted old backup: " + backup.getName());
            }
        }
    }
    
    /**
     * Ensure cache is loaded from file
     */
    private void ensureCacheLoaded() {
        if (!cacheLoaded) {
            loadFromFile();
            cacheLoaded = true;
        }
    }
    
    /**
     * Schedule a save operation (debounced)
     */
    private void scheduleSave() {
        // Simple debouncing - only save if it's been more than 1 second since last save
        long currentTime = System.currentTimeMillis();
        if (currentTime - lastSaveTime > 1000) {
            saveToFile();
        }
    }
    
    /**
     * Load data from file
     */
    private boolean loadFromFile() {
        try {
            if (!storageFile.exists()) {
                return true; // No file means empty data, which is valid
            }
            
            StringBuilder content = new StringBuilder();
            try (BufferedReader reader = new BufferedReader(new FileReader(storageFile))) {
                String line;
                while ((line = reader.readLine()) != null) {
                    content.append(line);
                }
            }
            
            if (content.length() > 0) {
                Map<String, Object> loadedData = parseJsonData(content.toString());
                dataCache.clear();
                dataCache.putAll(loadedData);
            }
            
            LogUtils.logDebug("Loaded " + dataCache.size() + " items for plugin: " + pluginId);
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Failed to load data for plugin " + pluginId + ": " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Save data to file
     */
    private boolean saveToFile() {
        try {
            // Create backup before saving
            if (storageFile.exists()) {
                File tempBackup = new File(storageDir, "temp" + BACKUP_EXTENSION);
                copyFile(storageFile, tempBackup);
            }
            
            String jsonData = generateJsonData(dataCache);
            
            try (FileWriter writer = new FileWriter(storageFile)) {
                writer.write(jsonData);
            }
            
            lastSaveTime = System.currentTimeMillis();
            LogUtils.logDebug("Saved " + dataCache.size() + " items for plugin: " + pluginId);
            return true;
            
        } catch (Exception e) {
            LogUtils.logError("Failed to save data for plugin " + pluginId + ": " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Simple JSON data parsing (basic implementation)
     */
    private Map<String, Object> parseJsonData(String jsonContent) {
        Map<String, Object> result = new HashMap<>();
        
        try {
            // This is a simplified JSON parser for basic data types
            // In a production implementation, you would use a proper JSON library
            jsonContent = jsonContent.trim();
            if (jsonContent.startsWith("{") && jsonContent.endsWith("}")) {
                jsonContent = jsonContent.substring(1, jsonContent.length() - 1);
                
                String[] pairs = jsonContent.split(",");
                for (String pair : pairs) {
                    String[] keyValue = pair.split(":", 2);
                    if (keyValue.length == 2) {
                        String key = keyValue[0].trim().replaceAll("\"", "");
                        String value = keyValue[1].trim();
                        
                        // Parse value type
                        Object parsedValue = parseJsonValue(value);
                        result.put(key, parsedValue);
                    }
                }
            }
        } catch (Exception e) {
            LogUtils.logDebug("JSON parsing error: " + e.getMessage());
        }
        
        return result;
    }
    
    /**
     * Parse individual JSON values
     */
    private Object parseJsonValue(String value) {
        value = value.trim();
        
        // String
        if (value.startsWith("\"") && value.endsWith("\"")) {
            return value.substring(1, value.length() - 1);
        }
        
        // Boolean
        if ("true".equals(value)) return true;
        if ("false".equals(value)) return false;
        
        // Null
        if ("null".equals(value)) return null;
        
        // Number
        try {
            if (value.contains(".")) {
                return Double.parseDouble(value);
            } else {
                return Integer.parseInt(value);
            }
        } catch (NumberFormatException e) {
            // Fall back to string
            return value;
        }
    }
    
    /**
     * Generate JSON data (basic implementation)
     */
    private String generateJsonData(Map<String, Object> data) {
        StringBuilder json = new StringBuilder();
        json.append("{");
        
        boolean first = true;
        for (Map.Entry<String, Object> entry : data.entrySet()) {
            if (!first) {
                json.append(",");
            }
            first = false;
            
            json.append("\"").append(entry.getKey()).append("\":");
            json.append(formatJsonValue(entry.getValue()));
        }
        
        json.append("}");
        return json.toString();
    }
    
    /**
     * Format individual values for JSON
     */
    private String formatJsonValue(Object value) {
        if (value == null) {
            return "null";
        } else if (value instanceof String) {
            return "\"" + value.toString().replace("\"", "\\\"") + "\"";
        } else if (value instanceof Boolean) {
            return value.toString();
        } else if (value instanceof Number) {
            return value.toString();
        } else {
            // Convert to string as fallback
            return "\"" + value.toString().replace("\"", "\\\"") + "\"";
        }
    }
    
    /**
     * Copy file utility
     */
    private boolean copyFile(File source, File target) {
        try (FileInputStream in = new FileInputStream(source);
             FileOutputStream out = new FileOutputStream(target)) {
            
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = in.read(buffer)) != -1) {
                out.write(buffer, 0, bytesRead);
            }
            return true;
            
        } catch (Exception e) {
            LogUtils.logDebug("File copy error: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Get storage statistics
     */
    public StorageStats getStats() {
        synchronized (lock) {
            ensureCacheLoaded();
            
            StorageStats stats = new StorageStats();
            stats.pluginId = pluginId;
            stats.itemCount = dataCache.size();
            stats.fileSize = storageFile.exists() ? storageFile.length() : 0;
            stats.lastModified = storageFile.exists() ? storageFile.lastModified() : 0;
            stats.backupCount = getBackupFiles().size();
            
            return stats;
        }
    }
    
    /**
     * Storage statistics container
     */
    public static class StorageStats {
        public String pluginId;
        public int itemCount;
        public long fileSize;
        public long lastModified;
        public int backupCount;
        
        @Override
        public String toString() {
            return String.format("StorageStats[%s]: %d items, %d bytes, %d backups", 
                pluginId, itemCount, fileSize, backupCount);
        }
    }
}


/ModLoader/app/src/main/java/com/modloader/plugin/PluginValidator.java

// File: PluginValidator.java - Plugin Validation and Security System
// Path: /app/src/main/java/com/modloader/plugin/PluginValidator.java

package com.modloader.plugin;

import android.content.Context;
import com.modloader.util.LogUtils;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.HashMap;
import java.util.Set;
import java.util.HashSet;
import java.util.zip.ZipEntry;
import java.util.zip.ZipInputStream;
import java.lang.reflect.Method;
import java.util.regex.Pattern;

/**
 * Plugin Validator - Comprehensive plugin validation and security checking system
 * Validates plugin files, checks signatures, scans for malicious code, and enforces security policies
 */
public class PluginValidator {
    private static final String TAG = "PluginValidator";

    // Security settings
    private final Context context;
    private boolean strictValidationEnabled = true;
    private boolean sandboxModeEnabled = true;
    private boolean scanForMaliciousCode = true;
    private final Set<String> trustedPluginHashes = new HashSet<>();
    private final Set<String> blockedPluginHashes = new HashSet<>();
    
    // Validation rules
    private static final long MAX_PLUGIN_SIZE = 50 * 1024 * 1024; // 50MB
    private static final int MAX_METHODS_PER_CLASS = 1000;
    private static final String[] ALLOWED_PLUGIN_EXTENSIONS = {".jar", ".dex", ".apk"};
    
    // Dangerous patterns to scan for
    private static final String[] DANGEROUS_PATTERNS = {
        "java.lang.Runtime.exec",
        "android.os.SystemClock.sleep", 
        "java.net.HttpURLConnection",
        "java.io.FileOutputStream",
        "android.content.Context.openFileOutput",
        "dalvik.system.DexClassLoader",
        "java.lang.Class.forName",
        "java.lang.reflect.Method.invoke"
    };
    
    // Required plugin manifest entries
    private static final String[] REQUIRED_MANIFEST_ENTRIES = {
        "Plugin-Id",
        "Plugin-Name", 
        "Plugin-Version",
        "Plugin-Author"
    };

    public PluginValidator(Context context) {
        this.context = context;
        loadTrustedHashes();
        loadBlockedHashes();
    }

    /**
     * Comprehensive plugin validation
     */
    public ValidationResult validatePlugin(File pluginFile) {
        LogUtils.logDebug("Starting comprehensive validation for: " + pluginFile.getName());
        
        ValidationResult result = new ValidationResult();
        result.pluginFile = pluginFile;
        result.pluginName = pluginFile.getName();
        
        try {
            // Phase 1: Basic file validation
            if (!validateBasicFile(pluginFile, result)) {
                return result;
            }
            
            // Phase 2: Security validation
            if (!validateSecurity(pluginFile, result)) {
                return result;
            }
            
            // Phase 3: Plugin structure validation
            if (!validatePluginStructure(pluginFile, result)) {
                return result;
            }
            
            // Phase 4: Code analysis (if enabled)
            if (scanForMaliciousCode) {
                if (!scanPluginCode(pluginFile, result)) {
                    return result;
                }
            }
            
            // Phase 5: Permission validation
            if (!validatePermissions(pluginFile, result)) {
                return result;
            }
            
            // If all validations pass
            result.isValid = true;
            result.validationMessage = "Plugin validation completed successfully";
            LogUtils.logDebug("Plugin validation successful: " + pluginFile.getName());
            
        } catch (Exception e) {
            result.isValid = false;
            result.validationMessage = "Validation error: " + e.getMessage();
            result.securityIssues.add("Validation exception: " + e.getMessage());
            LogUtils.logDebug("Plugin validation failed with exception: " + e.getMessage());
        }
        
        return result;
    }

    /**
     * Basic file validation - size, format, accessibility
     */
    private boolean validateBasicFile(File pluginFile, ValidationResult result) {
        // Check if file exists
        if (!pluginFile.exists()) {
            result.validationMessage = "Plugin file does not exist";
            result.issues.add("File not found: " + pluginFile.getAbsolutePath());
            return false;
        }
        
        // Check if file is readable
        if (!pluginFile.canRead()) {
            result.validationMessage = "Plugin file is not readable";
            result.issues.add("File permissions deny read access");
            return false;
        }
        
        // Check file size
        long fileSize = pluginFile.length();
        if (fileSize == 0) {
            result.validationMessage = "Plugin file is empty";
            result.issues.add("File size is 0 bytes");
            return false;
        }
        
        if (fileSize > MAX_PLUGIN_SIZE) {
            result.validationMessage = "Plugin file is too large";
            result.issues.add("File size exceeds maximum: " + formatFileSize(fileSize));
            return false;
        }
        
        // Check file extension
        String fileName = pluginFile.getName().toLowerCase();
        boolean validExtension = false;
        for (String ext : ALLOWED_PLUGIN_EXTENSIONS) {
            if (fileName.endsWith(ext)) {
                validExtension = true;
                break;
            }
        }
        
        if (!validExtension) {
            result.validationMessage = "Invalid plugin file extension";
            result.issues.add("File extension not allowed: " + fileName);
            return false;
        }
        
        result.fileSize = fileSize;
        return true;
    }

    /**
     * Security validation - hash checking, signature verification
     */
    private boolean validateSecurity(File pluginFile, ValidationResult result) {
        try {
            // Calculate file hash
            String fileHash = calculateFileHash(pluginFile);
            result.fileHash = fileHash;
            
            // Check if plugin is in blocked list
            if (blockedPluginHashes.contains(fileHash)) {
                result.validationMessage = "Plugin is blocked (security risk)";
                result.securityIssues.add("Plugin hash matches blocked list");
                return false;
            }
            
            // Check if plugin is trusted
            boolean isTrusted = trustedPluginHashes.contains(fileHash);
            result.isTrusted = isTrusted;
            
            if (strictValidationEnabled && !isTrusted) {
                result.warnings.add("Plugin is not in trusted list - proceed with caution");
            }
            
            // Validate file header/magic numbers
            if (!validateFileHeader(pluginFile, result)) {
                return false;
            }
            
            return true;
            
        } catch (Exception e) {
            result.validationMessage = "Security validation failed: " + e.getMessage();
            result.securityIssues.add("Hash calculation error: " + e.getMessage());
            return false;
        }
    }

    /**
     * Validate plugin structure and manifest
     */
    private boolean validatePluginStructure(File pluginFile, ValidationResult result) {
        try {
            if (pluginFile.getName().toLowerCase().endsWith(".jar")) {
                return validateJarStructure(pluginFile, result);
            } else if (pluginFile.getName().toLowerCase().endsWith(".dex")) {
                return validateDexStructure(pluginFile, result);
            } else if (pluginFile.getName().toLowerCase().endsWith(".apk")) {
                return validateApkStructure(pluginFile, result);
            }
            
            result.warnings.add("Unknown plugin format - skipping structure validation");
            return true;
            
        } catch (Exception e) {
            result.validationMessage = "Structure validation failed: " + e.getMessage();
            result.issues.add("Structure validation error: " + e.getMessage());
            return false;
        }
    }

    /**
     * Validate JAR plugin structure
     */
    private boolean validateJarStructure(File pluginFile, ValidationResult result) {
        try (ZipInputStream zis = new ZipInputStream(new FileInputStream(pluginFile))) {
            ZipEntry entry;
            boolean hasManifest = false;
            boolean hasPluginClass = false;
            int classCount = 0;
            
            while ((entry = zis.getNextEntry()) != null) {
                String entryName = entry.getName();
                
                if ("META-INF/MANIFEST.MF".equals(entryName)) {
                    hasManifest = true;
                    // Could validate manifest content here
                }
                
                if (entryName.endsWith(".class")) {
                    classCount++;
                    if (entryName.contains("Plugin")) {
                        hasPluginClass = true;
                    }
                }
            }
            
            if (!hasManifest) {
                result.warnings.add("JAR missing manifest file");
            }
            
            if (!hasPluginClass) {
                result.warnings.add("No plugin class found in JAR");
            }
            
            if (classCount == 0) {
                result.validationMessage = "No class files found in plugin JAR";
                result.issues.add("JAR contains no executable classes");
                return false;
            }
            
            result.classCount = classCount;
            return true;
            
        } catch (IOException e) {
            result.validationMessage = "Failed to read JAR structure: " + e.getMessage();
            result.issues.add("JAR reading error: " + e.getMessage());
            return false;
        }
    }

    /**
     * Validate DEX plugin structure
     */
    private boolean validateDexStructure(File pluginFile, ValidationResult result) {
        try (FileInputStream fis = new FileInputStream(pluginFile)) {
            // Check DEX magic number
            byte[] header = new byte[8];
            if (fis.read(header) != 8) {
                result.validationMessage = "Invalid DEX file header";
                result.issues.add("Could not read DEX header");
                return false;
            }
            
            // Verify DEX magic ("dex\n" followed by version)
            String magic = new String(header, 0, 4);
            if (!"dex\n".equals(magic)) {
                result.validationMessage = "Invalid DEX magic number";
                result.issues.add("File does not have valid DEX signature");
                return false;
            }
            
            // Basic DEX validation passed
            result.dexVersion = new String(header, 4, 4).trim();
            return true;
            
        } catch (IOException e) {
            result.validationMessage = "DEX structure validation failed: " + e.getMessage();
            result.issues.add("DEX reading error: " + e.getMessage());
            return false;
        }
    }

    /**
     * Validate APK plugin structure
     */
    private boolean validateApkStructure(File pluginFile, ValidationResult result) {
        try (ZipInputStream zis = new ZipInputStream(new FileInputStream(pluginFile))) {
            ZipEntry entry;
            boolean hasManifest = false;
            boolean hasDex = false;
            
            while ((entry = zis.getNextEntry()) != null) {
                String entryName = entry.getName();
                
                if ("AndroidManifest.xml".equals(entryName)) {
                    hasManifest = true;
                }
                
                if (entryName.endsWith(".dex")) {
                    hasDex = true;
                }
            }
            
            if (!hasManifest) {
                result.validationMessage = "APK missing AndroidManifest.xml";
                result.issues.add("Invalid APK structure - no manifest");
                return false;
            }
            
            if (!hasDex) {
                result.validationMessage = "APK missing DEX files";
                result.issues.add("Invalid APK structure - no executable code");
                return false;
            }
            
            return true;
            
        } catch (IOException e) {
            result.validationMessage = "APK structure validation failed: " + e.getMessage();
            result.issues.add("APK reading error: " + e.getMessage());
            return false;
        }
    }

    /**
     * Scan plugin code for malicious patterns
     */
    private boolean scanPluginCode(File pluginFile, ValidationResult result) {
        try {
            List<String> suspiciousPatterns = new ArrayList<>();
            
            // Simple pattern scanning in file content
            try (FileInputStream fis = new FileInputStream(pluginFile)) {
                byte[] buffer = new byte[8192];
                StringBuilder content = new StringBuilder();
                int bytesRead;
                
                while ((bytesRead = fis.read(buffer)) != -1) {
                    content.append(new String(buffer, 0, bytesRead));
                    
                    // Don't let content get too large
                    if (content.length() > 1024 * 1024) { // 1MB limit
                        break;
                    }
                }
                
                String pluginContent = content.toString();
                
                // Check for dangerous patterns
                for (String pattern : DANGEROUS_PATTERNS) {
                    if (pluginContent.contains(pattern)) {
                        suspiciousPatterns.add(pattern);
                    }
                }
            }
            
            if (!suspiciousPatterns.isEmpty()) {
                result.suspiciousPatterns.addAll(suspiciousPatterns);
                
                if (strictValidationEnabled) {
                    result.validationMessage = "Plugin contains suspicious code patterns";
                    result.securityIssues.add("Detected patterns: " + String.join(", ", suspiciousPatterns));
                    return false;
                } else {
                    result.warnings.add("Plugin contains suspicious patterns (allowed in non-strict mode)");
                }
            }
            
            return true;
            
        } catch (Exception e) {
            result.warnings.add("Code scanning failed: " + e.getMessage());
            return true; // Don't fail validation if scanning fails
        }
    }

    /**
     * Validate plugin permissions
     */
    private boolean validatePermissions(File pluginFile, ValidationResult result) {
        // For now, just check if sandbox mode should be enforced
        if (sandboxModeEnabled) {
            result.requiresSandbox = true;
        }
        
        // Could check plugin manifest for declared permissions
        // and validate against allowed permissions list
        
        return true;
    }

    /**
     * Validate file header/magic numbers
     */
    private boolean validateFileHeader(File pluginFile, ValidationResult result) {
        try (FileInputStream fis = new FileInputStream(pluginFile)) {
            byte[] header = new byte[4);
            if (fis.read(header) != 4) {
                result.warnings.add("Could not read file header");
                return true; // Don't fail validation
            }
            
            String fileName = pluginFile.getName().toLowerCase();
            
            if (fileName.endsWith(".jar") || fileName.endsWith(".apk")) {
                // Check for ZIP signature (PK)
                if (header[0] == 0x50 && header[1] == 0x4B) {
                    return true;
                }
                result.validationMessage = "Invalid ZIP/JAR/APK file signature";
                result.issues.add("File header does not match expected format");
                return false;
                
            } else if (fileName.endsWith(".dex")) {
                // Check for DEX signature
                String magic = new String(header);
                if (magic.startsWith("dex\n")) {
                    return true;
                }
                result.validationMessage = "Invalid DEX file signature";
                result.issues.add("File header does not match DEX format");
                return false;
            }
            
            return true;
            
        } catch (IOException e) {
            result.warnings.add("Header validation failed: " + e.getMessage());
            return true; // Don't fail validation
        }
    }

    /**
     * Calculate SHA-256 hash of file
     */
    private String calculateFileHash(File file) throws IOException, NoSuchAlgorithmException {
        MessageDigest md = MessageDigest.getInstance("SHA-256");
        
        try (FileInputStream fis = new FileInputStream(file)) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            
            while ((bytesRead = fis.read(buffer)) != -1) {
                md.update(buffer, 0, bytesRead);
            }
        }
        
        byte[] hashBytes = md.digest();
        StringBuilder sb = new StringBuilder();
        for (byte b : hashBytes) {
            sb.append(String.format("%02x", b));
        }
        
        return sb.toString();
    }

    /**
     * Load trusted plugin hashes from storage
     */
    private void loadTrustedHashes() {
        // Could load from file or shared preferences
        // For now, add some example trusted hashes
        trustedPluginHashes.add("example_trusted_hash_1");
        trustedPluginHashes.add("example_trusted_hash_2");
    }

    /**
     * Load blocked plugin hashes from storage  
     */
    private void loadBlockedHashes() {
        // Could load from file or shared preferences
        // For now, add some example blocked hashes
        blockedPluginHashes.add("example_malicious_hash_1");
        blockedPluginHashes.add("example_malicious_hash_2");
    }

    /**
     * Format file size for display
     */
    private String formatFileSize(long bytes) {
        if (bytes < 1024) return bytes + " B";
        if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
        return String.format("%.1f MB", bytes / (1024.0 * 1024.0));
    }

    // ===== Configuration Methods =====

    public void setStrictValidationEnabled(boolean enabled) {
        this.strictValidationEnabled = enabled;
        LogUtils.logDebug("Strict validation " + (enabled ? "enabled" : "disabled"));
    }

    public void setSandboxModeEnabled(boolean enabled) {
        this.sandboxModeEnabled = enabled;
        LogUtils.logDebug("Sandbox mode " + (enabled ? "enabled" : "disabled"));
    }

    public void setScanForMaliciousCode(boolean enabled) {
        this.scanForMaliciousCode = enabled;
        LogUtils.logDebug("Malicious code scanning " + (enabled ? "enabled" : "disabled"));
    }

    public void addTrustedPluginHash(String hash) {
        trustedPluginHashes.add(hash);
        LogUtils.logDebug("Added trusted plugin hash");
    }

    public void addBlockedPluginHash(String hash) {
        blockedPluginHashes.add(hash);
        LogUtils.logDebug("Added blocked plugin hash");
    }

    // ===== Validation Result Class =====

    public static class ValidationResult {
        public boolean isValid = false;
        public String validationMessage = "";
        public File pluginFile;
        public String pluginName;
        public long fileSize;
        public String fileHash;
        public boolean isTrusted = false;
        public boolean requiresSandbox = false;
        public int classCount = 0;
        public String dexVersion = "";
        
        public List<String> issues = new ArrayList<>();
        public List<String> warnings = new ArrayList<>();
        public List<String> securityIssues = new ArrayList<>();
        public List<String> suspiciousPatterns = new ArrayList<>();
        
        public boolean hasIssues() {
            return !issues.isEmpty();
        }
        
        public boolean hasWarnings() {
            return !warnings.isEmpty();
        }
        
        public boolean hasSecurityIssues() {
            return !securityIssues.isEmpty();
        }
        
        public String getDetailedReport() {
            StringBuilder report = new StringBuilder();
            report.append("=== Plugin Validation Report ===\n");
            report.append("Plugin: ").append(pluginName).append("\n");
            report.append("Valid: ").append(isValid ? "✅ YES" : "❌ NO").append("\n");
            report.append("Size: ").append(formatFileSize(fileSize)).append("\n");
            
            if (fileHash != null) {
                report.append("Hash: ").append(fileHash.substring(0, 16)).append("...\n");
            }
            
            report.append("Trusted: ").append(isTrusted ? "✅ YES" : "❌ NO").append("\n");
            report.append("Requires Sandbox: ").append(requiresSandbox ? "✅ YES" : "❌ NO").append("\n");
            
            if (classCount > 0) {
                report.append("Classes: ").append(classCount).append("\n");
            }
            
            if (!dexVersion.isEmpty()) {
                report.append("DEX Version: ").append(dexVersion).append("\n");
            }
            
            if (!validationMessage.isEmpty()) {
                report.append("Message: ").append(validationMessage).append("\n");
            }
            
            if (hasIssues()) {
                report.append("\n❌ Issues:\n");
                for (String issue : issues) {
                    report.append("• ").append(issue).append("\n");
                }
            }
            
            if (hasSecurityIssues()) {
                report.append("\n🔒 Security Issues:\n");
                for (String issue : securityIssues) {
                    report.append("• ").append(issue).append("\n");
                }
            }
            
            if (hasWarnings()) {
                report.append("\n⚠️ Warnings:\n");
                for (String warning : warnings) {
                    report.append("• ").append(warning).append("\n");
                }
            }
            
            if (!suspiciousPatterns.isEmpty()) {
                report.append("\n🚨 Suspicious Patterns:\n");
                for (String pattern : suspiciousPatterns) {
                    report.append("• ").append(pattern).append("\n");
                }
            }
            
            return report.toString();
        }
        
        private String formatFileSize(long bytes) {
            if (bytes < 1024) return bytes + " B";
            if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
            return String.format("%.1f MB", bytes / (1024.0 * 1024.0));
        }
    }

    // ===== Quick Validation Methods =====

    /**
     * Quick validation for basic plugin acceptance
     */
    public boolean isPluginAcceptable(File pluginFile) {
        ValidationResult result = validatePlugin(pluginFile);
        return result.isValid && !result.hasSecurityIssues();
    }

    /**
     * Get validation summary
     */
    public String getValidationSummary(File pluginFile) {
        ValidationResult result = validatePlugin(pluginFile);
        
        if (result.isValid) {
            return "✅ Valid plugin" + (result.hasWarnings() ? " (with warnings)" : "");
        } else {
            return "❌ Invalid plugin: " + result.validationMessage;
        }
    }

    /**
     * Check if plugin requires special handling
     */
    public boolean requiresSpecialHandling(File pluginFile) {
        ValidationResult result = validatePlugin(pluginFile);
        return result.requiresSandbox || result.hasSecurityIssues() || !result.suspiciousPatterns.isEmpty();
    }
}



/ModLoader/app/src/main/java/com/modloader/plugin/examples/ExampleModificationPlugin.java

// File: ExampleModificationPlugin.java - Sample Game Modification Plugin
// Path: /app/src/main/java/com/modloader/plugin/examples/ExampleModificationPlugin.java

package com.modloader.plugin.examples;

import android.content.Context;
import android.widget.Toast;

import com.modloader.plugin.Plugin;
import com.modloader.plugin.PluginAPI;
import com.modloader.plugin.PluginContext;
import com.modloader.plugin.PluginHook;
import com.modloader.util.LogUtils;

import java.util.Map;
import java.util.HashMap;
import java.util.List;
import java.util.ArrayList;

/**
 * Example Game Modification Plugin - Demonstrates game modification capabilities
 * This plugin shows how to modify game behavior, patch methods, and interact with game systems
 */
public class ExampleModificationPlugin extends Plugin {
    private static final String PLUGIN_ID = "example_modification_plugin";
    private static final String PLUGIN_NAME = "Example Game Modification";
    private static final String PLUGIN_VERSION = "1.0.0";
    private static final String PLUGIN_AUTHOR = "Plugin System";
    private static final String PLUGIN_DESCRIPTION = "Demonstrates game modification capabilities including method patching, value changes, and hook implementations.";

    // Plugin state
    private boolean gameHooksInstalled = false;
    private Map<String, Object> originalValues = new HashMap<>();
    private List<String> installedPatches = new ArrayList<>();

    // Configuration
    private boolean enableDamageMultiplier = true;
    private float damageMultiplier = 2.0f;
    private boolean enableSpeedBoost = true;
    private float speedMultiplier = 1.5f;
    private boolean enableResourceMultiplier = true;
    private int resourceMultiplier = 3;

    @Override
    public String getId() {
        return PLUGIN_ID;
    }

    @Override
    public String getName() {
        return PLUGIN_NAME;
    }

    @Override
    public String getVersion() {
        return PLUGIN_VERSION;
    }

    @Override
    public String getAuthor() {
        return PLUGIN_AUTHOR;
    }

    @Override
    public String getDescription() {
        return PLUGIN_DESCRIPTION;
    }

    @Override
    public void initialize(PluginContext context) {
        super.initialize(context);
        LogUtils.logUser("🔧 Initializing Game Modification Plugin");
        
        loadConfiguration();
        setupGameHooks();
        
        LogUtils.logUser("✅ Game Modification Plugin initialized successfully");
    }

    @Override
    public void onEnable() {
        super.onEnable();
        LogUtils.logUser("🎮 Enabling game modifications...");
        
        try {
            installGameHooks();
            applyGamePatches();
            registerEventHandlers();
            
            showToast("Game modifications enabled! Damage: " + damageMultiplier + "x, Speed: " + speedMultiplier + "x");
            LogUtils.logUser("✅ Game modifications enabled successfully");
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to enable game modifications: " + e.getMessage());
            showToast("Failed to enable game modifications: " + e.getMessage());
        }
    }

    @Override
    public void onDisable() {
        super.onDisable();
        LogUtils.logUser("🎮 Disabling game modifications...");
        
        try {
            removeGameHooks();
            revertGamePatches();
            unregisterEventHandlers();
            
            showToast("Game modifications disabled - game restored to normal");
            LogUtils.logUser("✅ Game modifications disabled successfully");
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to disable game modifications: " + e.getMessage());
            showToast("Error disabling modifications: " + e.getMessage());
        }
    }

    @Override
    public void onGameStart() {
        LogUtils.logUser("🎮 Game starting - applying runtime modifications");
        
        if (isEnabled()) {
            applyRuntimeModifications();
            showToast("Game modifications active!");
        }
    }

    @Override
    public void onGameStop() {
        LogUtils.logUser("🎮 Game stopping - cleaning up modifications");
        
        if (isEnabled()) {
            cleanupRuntimeModifications();
        }
    }

    @Override
    public Map<String, Object> getConfigurationOptions() {
        Map<String, Object> config = new HashMap<>();
        config.put("enableDamageMultiplier", enableDamageMultiplier);
        config.put("damageMultiplier", damageMultiplier);
        config.put("enableSpeedBoost", enableSpeedBoost);
        config.put("speedMultiplier", speedMultiplier);
        config.put("enableResourceMultiplier", enableResourceMultiplier);
        config.put("resourceMultiplier", resourceMultiplier);
        return config;
    }

    @Override
    public void setConfigurationOptions(Map<String, Object> config) {
        if (config.containsKey("enableDamageMultiplier")) {
            enableDamageMultiplier = (Boolean) config.get("enableDamageMultiplier");
        }
        if (config.containsKey("damageMultiplier")) {
            damageMultiplier = ((Number) config.get("damageMultiplier")).floatValue();
        }
        if (config.containsKey("enableSpeedBoost")) {
            enableSpeedBoost = (Boolean) config.get("enableSpeedBoost");
        }
        if (config.containsKey("speedMultiplier")) {
            speedMultiplier = ((Number) config.get("speedMultiplier")).floatValue();
        }
        if (config.containsKey("enableResourceMultiplier")) {
            enableResourceMultiplier = (Boolean) config.get("enableResourceMultiplier");
        }
        if (config.containsKey("resourceMultiplier")) {
            resourceMultiplier = ((Number) config.get("resourceMultiplier")).intValue();
        }
        
        LogUtils.logDebug("Configuration updated: damage=" + damageMultiplier + ", speed=" + speedMultiplier);
        
        // Apply changes immediately if plugin is enabled
        if (isEnabled()) {
            updateGameModifications();
        }
    }

    // ===== Game Modification Methods =====

    private void loadConfiguration() {
        try {
            // Load configuration from plugin storage
            Map<String, Object> saved = getPluginContext().getStorage().loadData("config");
            if (saved != null && !saved.isEmpty()) {
                setConfigurationOptions(saved);
                LogUtils.logDebug("Loaded configuration from storage");
            }
        } catch (Exception e) {
            LogUtils.logDebug("Failed to load configuration: " + e.getMessage());
        }
    }

    private void saveConfiguration() {
        try {
            getPluginContext().getStorage().saveData("config", getConfigurationOptions());
            LogUtils.logDebug("Configuration saved successfully");
        } catch (Exception e) {
            LogUtils.logDebug("Failed to save configuration: " + e.getMessage());
        }
    }

    private void setupGameHooks() {
        PluginAPI api = getPluginContext().getAPI();
        
        // Register hook for player damage calculation
        api.registerHook("player.calculateDamage", new PluginHook.HookCallback() {
            @Override
            public Object onHookCalled(String hookName, Object[] args) {
                if (enableDamageMultiplier && args.length > 0) {
                    try {
                        float originalDamage = ((Number) args[0]).floatValue();
                        float modifiedDamage = originalDamage * damageMultiplier;
                        LogUtils.logDebug("Damage modified: " + originalDamage + " -> " + modifiedDamage);
                        return modifiedDamage;
                    } catch (Exception e) {
                        LogUtils.logDebug("Damage hook error: " + e.getMessage());
                        return args[0];
                    }
                }
                return args[0];
            }
        });

        // Register hook for player movement speed
        api.registerHook("player.getMovementSpeed", new PluginHook.HookCallback() {
            @Override
            public Object onHookCalled(String hookName, Object[] args) {
                if (enableSpeedBoost && args.length > 0) {
                    try {
                        float originalSpeed = ((Number) args[0]).floatValue();
                        float modifiedSpeed = originalSpeed * speedMultiplier;
                        LogUtils.logDebug("Speed modified: " + originalSpeed + " -> " + modifiedSpeed);
                        return modifiedSpeed;
                    } catch (Exception e) {
                        LogUtils.logDebug("Speed hook error: " + e.getMessage());
                        return args[0];
                    }
                }
                return args[0];
            }
        });

        // Register hook for resource collection
        api.registerHook("game.collectResource", new PluginHook.HookCallback() {
            @Override
            public Object onHookCalled(String hookName, Object[] args) {
                if (enableResourceMultiplier && args.length > 1) {
                    try {
                        int originalAmount = ((Number) args[1]).intValue();
                        int modifiedAmount = originalAmount * resourceMultiplier;
                        LogUtils.logDebug("Resource collection modified: " + originalAmount + " -> " + modifiedAmount);
                        return modifiedAmount;
                    } catch (Exception e) {
                        LogUtils.logDebug("Resource hook error: " + e.getMessage());
                        return args[1];
                    }
                }
                return args[1];
            }
        });

        LogUtils.logDebug("Game hooks registered successfully");
    }

    private void installGameHooks() {
        if (gameHooksInstalled) {
            return;
        }

        try {
            PluginAPI api = getPluginContext().getAPI();
            
            // Install damage modification hook
            if (enableDamageMultiplier) {
                api.installHook("player.calculateDamage");
                installedPatches.add("damage_multiplier");
            }

            // Install speed modification hook
            if (enableSpeedBoost) {
                api.installHook("player.getMovementSpeed");
                installedPatches.add("speed_boost");
            }

            // Install resource collection hook
            if (enableResourceMultiplier) {
                api.installHook("game.collectResource");
                installedPatches.add("resource_multiplier");
            }

            gameHooksInstalled = true;
            LogUtils.logUser("🔧 Installed " + installedPatches.size() + " game modification hooks");

        } catch (Exception e) {
            LogUtils.logDebug("Failed to install game hooks: " + e.getMessage());
            throw new RuntimeException("Game hook installation failed", e);
        }
    }

    private void removeGameHooks() {
        if (!gameHooksInstalled) {
            return;
        }

        try {
            PluginAPI api = getPluginContext().getAPI();
            
            // Remove all installed hooks
            for (String patch : installedPatches) {
                try {
                    switch (patch) {
                        case "damage_multiplier":
                            api.uninstallHook("player.calculateDamage");
                            break;
                        case "speed_boost":
                            api.uninstallHook("player.getMovementSpeed");
                            break;
                        case "resource_multiplier":
                            api.uninstallHook("game.collectResource");
                            break;
                    }
                } catch (Exception e) {
                    LogUtils.logDebug("Failed to remove hook " + patch + ": " + e.getMessage());
                }
            }

            installedPatches.clear();
            gameHooksInstalled = false;
            LogUtils.logUser("🔧 Removed all game modification hooks");

        } catch (Exception e) {
            LogUtils.logDebug("Failed to remove game hooks: " + e.getMessage());
        }
    }

    private void applyGamePatches() {
        try {
            PluginAPI api = getPluginContext().getAPI();
            
            // Example: Patch game configuration values
            if (api.hasGameAccess()) {
                // Store original values before modifying
                Object originalMaxHealth = api.getGameValue("player.maxHealth");
                if (originalMaxHealth != null) {
                    originalValues.put("maxHealth", originalHealth);
                    api.setGameValue("player.maxHealth", ((Number) originalMaxHealth).intValue() * 2);
                }

                Object originalMaxMana = api.getGameValue("player.maxMana");
                if (originalMaxMana != null) {
                    originalValues.put("maxMana", originalMaxMana);
                    api.setGameValue("player.maxMana", ((Number) originalMaxMana).intValue() * 2);
                }

                LogUtils.logDebug("Game patches applied successfully");
            }

        } catch (Exception e) {
            LogUtils.logDebug("Failed to apply game patches: " + e.getMessage());
        }
    }

    private void revertGamePatches() {
        try {
            PluginAPI api = getPluginContext().getAPI();
            
            // Restore original values
            if (api.hasGameAccess()) {
                for (Map.Entry<String, Object> entry : originalValues.entrySet()) {
                    try {
                        switch (entry.getKey()) {
                            case "maxHealth":
                                api.setGameValue("player.maxHealth", entry.getValue());
                                break;
                            case "maxMana":
                                api.setGameValue("player.maxMana", entry.getValue());
                                break;
                        }
                    } catch (Exception e) {
                        LogUtils.logDebug("Failed to revert " + entry.getKey() + ": " + e.getMessage());
                    }
                }

                originalValues.clear();
                LogUtils.logDebug("Game patches reverted successfully");
            }

        } catch (Exception e) {
            LogUtils.logDebug("Failed to revert game patches: " + e.getMessage());
        }
    }

    private void registerEventHandlers() {
        PluginAPI api = getPluginContext().getAPI();
        
        // Register for game events
        api.registerEventHandler("game.playerLevelUp", this::onPlayerLevelUp);
        api.registerEventHandler("game.playerDeath", this::onPlayerDeath);
        api.registerEventHandler("game.itemPickup", this::onItemPickup);
        
        LogUtils.logDebug("Event handlers registered");
    }

    private void unregisterEventHandlers() {
        PluginAPI api = getPluginContext().getAPI();
        
        api.unregisterEventHandler("game.playerLevelUp", this::onPlayerLevelUp);
        api.unregisterEventHandler("game.playerDeath", this::onPlayerDeath);
        api.unregisterEventHandler("game.itemPickup", this::onItemPickup);
        
        LogUtils.logDebug("Event handlers unregistered");
    }

    private void applyRuntimeModifications() {
        try {
            // Apply modifications that need to be active during gameplay
            LogUtils.logDebug("Applying runtime modifications");
            
            // Example: Modify game timers, physics, etc.
            showToast("Runtime modifications applied!");
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to apply runtime modifications: " + e.getMessage());
        }
    }

    private void cleanupRuntimeModifications() {
        try {
            // Clean up runtime modifications
            LogUtils.logDebug("Cleaning up runtime modifications");
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to cleanup runtime modifications: " + e.getMessage());
        }
    }

    private void updateGameModifications() {
        if (isEnabled() && gameHooksInstalled) {
            LogUtils.logDebug("Updating game modifications with new configuration");
            
            // Save new configuration
            saveConfiguration();
            
            // Reinstall hooks with new values
            removeGameHooks();
            installGameHooks();
            
            showToast("Game modifications updated!");
        }
    }

    // ===== Event Handlers =====

    private void onPlayerLevelUp(Object... args) {
        LogUtils.logDebug("Player leveled up - applying level-based modifications");
        showToast("Level up! Modifications enhanced!");
    }

    private void onPlayerDeath(Object... args) {
        LogUtils.logDebug("Player died - modifications remain active");
        showToast("Don't worry, modifications are still active!");
    }

    private void onItemPickup(Object... args) {
        if (enableResourceMultiplier && args.length > 0) {
            LogUtils.logDebug("Item pickup enhanced by resource multiplier");
        }
    }

    // ===== Utility Methods =====

    private void showToast(String message) {
        Context context = getPluginContext().getApplicationContext();
        if (context != null) {
            Toast.makeText(context, "[Game Mod] " + message, Toast.LENGTH_SHORT).show();
        }
    }

    @Override
    public String getStatusReport() {
        StringBuilder report = new StringBuilder();
        report.append("=== Game Modification Plugin Status ===\n");
        report.append("Plugin Enabled: ").append(isEnabled()).append("\n");
        report.append("Game Hooks Installed: ").append(gameHooksInstalled).append("\n");
        report.append("Active Patches: ").append(installedPatches.size()).append("\n");
        
        report.append("\nModifications:\n");
        report.append("• Damage Multiplier: ").append(enableDamageMultiplier ? damageMultiplier + "x" : "Disabled").append("\n");
        report.append("• Speed Boost: ").append(enableSpeedBoost ? speedMultiplier + "x" : "Disabled").append("\n");
        report.append("• Resource Multiplier: ").append(enableResourceMultiplier ? resourceMultiplier + "x" : "Disabled").append("\n");
        
        report.append("\nInstalled Patches:\n");
        for (String patch : installedPatches) {
            report.append("• ").append(patch).append("\n");
        }
        
        report.append("\nOriginal Values Stored: ").append(originalValues.size()).append("\n");
        
        return report.toString();
    }

    @Override
    public void cleanup() {
        LogUtils.logUser("🧹 Cleaning up Game Modification Plugin");
        
        try {
            // Ensure all modifications are removed
            if (isEnabled()) {
                onDisable();
            }
            
            // Clear all stored data
            originalValues.clear();
            installedPatches.clear();
            gameHooksInstalled = false;
            
            LogUtils.logUser("✅ Game Modification Plugin cleanup completed");
            
        } catch (Exception e) {
            LogUtils.logDebug("Error during plugin cleanup: " + e.getMessage());
        }
        
        super.cleanup();
    }

    // ===== Diagnostic and Testing Methods =====

    public void testModifications() {
        LogUtils.logUser("🧪 Testing game modifications...");
        
        try {
            // Test damage multiplier
            if (enableDamageMultiplier) {
                LogUtils.logUser("Testing damage multiplier: " + damageMultiplier + "x");
            }
            
            // Test speed boost
            if (enableSpeedBoost) {
                LogUtils.logUser("Testing speed boost: " + speedMultiplier + "x");
            }
            
            // Test resource multiplier
            if (enableResourceMultiplier) {
                LogUtils.logUser("Testing resource multiplier: " + resourceMultiplier + "x");
            }
            
            showToast("Modification test completed - check logs for details");
            
        } catch (Exception e) {
            LogUtils.logDebug("Modification test failed: " + e.getMessage());
            showToast("Modification test failed: " + e.getMessage());
        }
    }

    public void resetToDefaults() {
        LogUtils.logUser("🔄 Resetting game modifications to defaults");
        
        enableDamageMultiplier = true;
        damageMultiplier = 2.0f;
        enableSpeedBoost = true;
        speedMultiplier = 1.5f;
        enableResourceMultiplier = true;
        resourceMultiplier = 3;
        
        saveConfiguration();
        
        if (isEnabled()) {
            updateGameModifications();
        }
        
        showToast("Game modifications reset to defaults");
    }
}


/ModLoader/app/src/main/java/com/modloader/plugin/examples/ExampleThemePlugin.java

// File: ExampleThemePlugin.java - Sample Theme Plugin (150+ lines)
// Path: /app/src/main/java/com/modloader/plugin/examples/ExampleThemePlugin.java

package com.modloader.plugin.examples;

import android.graphics.Color;
import android.view.View;
import android.widget.Toast;

import com.modloader.plugin.Plugin;
import com.modloader.plugin.PluginContext;
import com.modloader.plugin.PluginHook;
import com.modloader.plugin.PluginAPI;

/**
 * Example Theme Plugin - Demonstrates UI theming capabilities
 * Shows how plugins can modify the app's appearance and behavior
 */
public class ExampleThemePlugin implements Plugin {
    private static final String PLUGIN_ID = "example_theme_plugin";
    private static final String PLUGIN_NAME = "Example Theme Plugin";
    private static final String VERSION = "1.0.0";
    private static final String AUTHOR = "TerrariaLoader Team";
    private static final String DESCRIPTION = "An example plugin that demonstrates theme customization and UI modifications";
    
    private PluginContext context;
    private PluginAPI api;
    
    // Theme configuration
    private String currentTheme = "default";
    private boolean darkModeEnabled = false;
    private int primaryColor = Color.parseColor("#4CAF50");
    private int accentColor = Color.parseColor("#FF9800");
    
    // Hook listeners
    private PluginHook.HookListener uiCreateListener;
    private PluginHook.HookListener themeChangeListener;
    private PluginHook.HookListener settingsChangeListener;
    
    @Override
    public String getId() {
        return PLUGIN_ID;
    }
    
    @Override
    public String getName() {
        return PLUGIN_NAME;
    }
    
    @Override
    public String getVersion() {
        return VERSION;
    }
    
    @Override
    public String getAuthor() {
        return AUTHOR;
    }
    
    @Override
    public String getDescription() {
        return DESCRIPTION;
    }
    
    @Override
    public void onLoad(PluginContext context) {
        this.context = context;
        this.api = context.getAPI();
        
        api.log("Theme plugin loading...");
        
        // Load theme configuration
        loadThemeConfiguration();
        
        // Initialize theme system
        initializeThemeSystem();
        
        // Setup hook listeners
        setupHookListeners();
        
        api.log("Theme plugin loaded successfully");
    }
    
    @Override
    public void onEnable(PluginContext context) {
        api.log("Theme plugin enabled");
        
        // Register hook listeners
        registerHookListeners();
        
        // Apply current theme
        applyCurrentTheme();
        
        // Show welcome message
        api.showToast("🎨 Theme Plugin: " + currentTheme + " theme activated!");
        
        // Schedule theme updates
        scheduleThemeUpdates();
    }
    
    @Override
    public void onDisable(PluginContext context) {
        api.log("Theme plugin disabled");
        
        // Restore default theme
        restoreDefaultTheme();
        
        // Show farewell message
        api.showToast("🎨 Theme Plugin: Restored default theme");
    }
    
    @Override
    public void onUnload(PluginContext context) {
        api.log("Theme plugin unloading...");
        
        // Clean up resources
        cleanup();
        
        api.log("Theme plugin unloaded");
    }
    
    /**
     * Load theme configuration from preferences
     */
    private void loadThemeConfiguration() {
        currentTheme = api.getSetting("theme_name", "default");
        darkModeEnabled = api.getSetting("dark_mode", false);
        
        // Load colors
        String primaryColorHex = api.getSetting("primary_color", "#4CAF50");
        String accentColorHex = api.getSetting("accent_color", "#FF9800");
        
        try {
            primaryColor = Color.parseColor(primaryColorHex);
            accentColor = Color.parseColor(accentColorHex);
        } catch (IllegalArgumentException e) {
            api.logWarning("Invalid color format, using defaults");
            primaryColor = Color.parseColor("#4CAF50");
            accentColor = Color.parseColor("#FF9800");
        }
        
        api.logDebug("Loaded theme configuration: " + currentTheme + 
                    " (Dark: " + darkModeEnabled + ")");
    }
    
    /**
     * Initialize the theme system
     */
    private void initializeThemeSystem() {
        // Create theme data directory
        context.setData("theme_applied", false);
        context.setData("original_colors", null);
        context.setData("theme_start_time", System.currentTimeMillis());
        
        // Store theme configurations
        context.setResource("available_themes", new String[]{
            "default", "dark", "nature", "ocean", "sunset", "neon"
        });
        
        context.setResource("theme_colors", createThemeColorMap());
    }
    
    /**
     * Create theme color mapping
     */
    private java.util.Map<String, ThemeColors> createThemeColorMap() {
        java.util.Map<String, ThemeColors> themeMap = new java.util.HashMap<>();
        
        themeMap.put("default", new ThemeColors(
            Color.parseColor("#4CAF50"), // primary
            Color.parseColor("#FF9800"), // accent
            Color.parseColor("#FFFFFF"), // background
            Color.parseColor("#333333")  // text
        ));
        
        themeMap.put("dark", new ThemeColors(
            Color.parseColor("#BB86FC"), // primary
            Color.parseColor("#03DAC6"), // accent
            Color.parseColor("#121212"), // background
            Color.parseColor("#FFFFFF")  // text
        ));
        
        themeMap.put("nature", new ThemeColors(
            Color.parseColor("#4CAF50"), // primary
            Color.parseColor("#8BC34A"), // accent
            Color.parseColor("#F1F8E9"), // background
            Color.parseColor("#2E7D32")  // text
        ));
        
        themeMap.put("ocean", new ThemeColors(
            Color.parseColor("#2196F3"), // primary
            Color.parseColor("#00BCD4"), // accent
            Color.parseColor("#E3F2FD"), // background
            Color.parseColor("#0D47A1")  // text
        ));
        
        themeMap.put("sunset", new ThemeColors(
            Color.parseColor("#FF5722"), // primary
            Color.parseColor("#FF9800"), // accent
            Color.parseColor("#FFF3E0"), // background
            Color.parseColor("#BF360C")  // text
        ));
        
        themeMap.put("neon", new ThemeColors(
            Color.parseColor("#E91E63"), // primary
            Color.parseColor("#9C27B0"), // accent
            Color.parseColor("#000000"), // background
            Color.parseColor("#00FF00")  // text
        ));
        
        return themeMap;
    }
    
    /**
     * Setup hook listeners
     */
    private void setupHookListeners() {
        // UI Creation Hook - Apply theme when UI elements are created
        uiCreateListener = new PluginHook.HookListener() {
            @Override
            public void onEvent(PluginHook.HookEvent event) {
                View view = event.getData("view", View.class);
                if (view != null) {
                    applyThemeToView(view);
                }
            }
            
            @Override
            public PluginHook.Priority getPriority() {
                return PluginHook.Priority.HIGH;
            }
        };
        
        // Theme Change Hook - Respond to theme changes
        themeChangeListener = new PluginHook.HookListener() {
            @Override
            public void onEvent(PluginHook.HookEvent event) {
                String newTheme = event.getData("theme", String.class);
                if (newTheme != null && !newTheme.equals(currentTheme)) {
                    switchTheme(newTheme);
                }
            }
        };
        
        // Settings Change Hook - Update theme when settings change
        settingsChangeListener = new PluginHook.HookListener() {
            @Override
            public void onEvent(PluginHook.HookEvent event) {
                String settingKey = event.getData("key", String.class);
                if (settingKey != null && settingKey.startsWith("theme_")) {
                    loadThemeConfiguration();
                    applyCurrentTheme();
                }
            }
        };
    }
    
    /**
     * Register hook listeners
     */
    private void registerHookListeners() {
        api.registerHook(PluginHook.Hooks.UI_MENU_CREATE, uiCreateListener);
        api.registerHook(PluginHook.Hooks.UI_THEME_CHANGE, themeChangeListener);
        api.registerHook(PluginHook.Hooks.SETTINGS_CHANGE, settingsChangeListener);
    }
    
    /**
     * Apply current theme
     */
    private void applyCurrentTheme() {
        @SuppressWarnings("unchecked")
        java.util.Map<String, ThemeColors> themeColors = context.getResource("theme_colors", java.util.Map.class);
        
        if (themeColors != null && themeColors.containsKey(currentTheme)) {
            ThemeColors colors = themeColors.get(currentTheme);
            primaryColor = colors.primary;
            accentColor = colors.accent;
            
            // Store current colors
            context.setData("current_primary", primaryColor);
            context.setData("current_accent", accentColor);
            context.setData("theme_applied", true);
            
            api.logDebug("Applied theme: " + currentTheme);
            
            // Trigger theme change event for other components
            api.triggerHook(PluginHook.Hooks.UI_THEME_CHANGE, this, "theme", currentTheme);
        }
    }
    
    /**
     * Apply theme to a specific view
     */
    private void applyThemeToView(View view) {
        if (!context.getData("theme_applied", Boolean.class, false)) {
            return;
        }
        
        try {
            // This is a simplified example - in a real implementation,
            // you would apply theme colors to specific view types
            view.setBackgroundColor(primaryColor);
            
            // Log the theming action
            api.logDebug("Applied theme to view: " + view.getClass().getSimpleName());
            
        } catch (Exception e) {
            api.logWarning("Failed to apply theme to view: " + e.getMessage());
        }
    }
    
    /**
     * Switch to a different theme
     */
    private void switchTheme(String themeName) {
        String oldTheme = currentTheme;
        currentTheme = themeName;
        
        // Save new theme setting
        api.setSetting("theme_name", currentTheme);
        
        // Apply the new theme
        applyCurrentTheme();
        
        api.log("Switched theme from " + oldTheme + " to " + currentTheme);
        api.showToast("🎨 Theme changed to: " + currentTheme);
    }
    
    /**
     * Restore default theme
     */
    private void restoreDefaultTheme() {
        currentTheme = "default";
        darkModeEnabled = false;
        applyCurrentTheme();
        
        api.logDebug("Restored default theme");
    }
    
    /**
     * Schedule periodic theme updates
     */
    private void scheduleThemeUpdates() {
        // Example: Auto-switch themes based on time of day
        api.executeAsync(() -> {
            while (context.isEnabled()) {
                try {
                    Thread.sleep(300000); // Check every 5 minutes
                    
                    if (api.getSetting("auto_theme_switch", false)) {
                        updateThemeBasedOnTime();
                    }
                    
                } catch (InterruptedException e) {
                    break;
                }
            }
        });
    }
    
    /**
     * Update theme based on current time
     */
    private void updateThemeBasedOnTime() {
        java.util.Calendar calendar = java.util.Calendar.getInstance();
        int hour = calendar.get(java.util.Calendar.HOUR_OF_DAY);
        
        String newTheme;
        if (hour >= 6 && hour < 12) {
            newTheme = "nature"; // Morning
        } else if (hour >= 12 && hour < 17) {
            newTheme = "ocean"; // Afternoon
        } else if (hour >= 17 && hour < 20) {
            newTheme = "sunset"; // Evening
        } else {
            newTheme = "dark"; // Night
        }
        
        if (!newTheme.equals(currentTheme)) {
            switchTheme(newTheme);
            api.log("Auto-switched to " + newTheme + " theme based on time");
        }
    }
    
    /**
     * Clean up resources
     */
    private void cleanup() {
        // Cancel any scheduled tasks
        context.setData("theme_applied", false);
        
        // Clear theme data
        context.removeData("current_primary");
        context.removeData("current_accent");
        context.removeData("theme_start_time");
    }
    
    // Helper class for theme colors
    private static class ThemeColors {
        final int primary;
        final int accent;
        final int background;
        final int text;
        
        public ThemeColors(int primary, int accent, int background, int text) {
            this.primary = primary;
            this.accent = accent;
            this.background = background;
            this.text = text;
        }
    }
}


/ModLoader/app/src/main/java/com/modloader/plugin/examples/ExampleUtilityPlugin.java

// File: ExampleUtilityPlugin.java - Sample Utility Plugin
// Path: /app/src/main/java/com/modloader/plugin/examples/ExampleUtilityPlugin.java

package com.modloader.plugin.examples;

import android.content.Context;
import android.widget.Toast;
import android.os.Handler;
import android.os.Looper;

import com.modloader.plugin.Plugin;
import com.modloader.plugin.PluginAPI;
import com.modloader.plugin.PluginContext;
import com.modloader.util.LogUtils;

import java.util.Map;
import java.util.HashMap;
import java.util.List;
import java.util.ArrayList;
import java.util.Timer;
import java.util.TimerTask;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;

/**
 * Example Utility Plugin - Demonstrates utility and helper functionality
 * This plugin provides various utility features like auto-save, performance monitoring,
 * system information display, and other helpful tools
 */
public class ExampleUtilityPlugin extends Plugin {
    private static final String PLUGIN_ID = "example_utility_plugin";
    private static final String PLUGIN_NAME = "Example Utility Tools";
    private static final String PLUGIN_VERSION = "1.0.0";
    private static final String PLUGIN_AUTHOR = "Plugin System";
    private static final String PLUGIN_DESCRIPTION = "Provides utility features including auto-save, performance monitoring, system info, and helpful tools.";

    // Plugin state
    private Timer autoSaveTimer;
    private Timer performanceMonitor;
    private Handler uiHandler;
    private List<String> performanceLog = new ArrayList<>();
    private long pluginStartTime;

    // Configuration
    private boolean enableAutoSave = true;
    private int autoSaveInterval = 300; // seconds
    private boolean enablePerformanceMonitoring = true;
    private int performanceCheckInterval = 60; // seconds
    private boolean enableSystemInfo = true;
    private boolean enableNotifications = true;
    private boolean enableDebugMode = false;

    // Statistics
    private int autoSaveCount = 0;
    private int performanceChecks = 0;
    private long totalMemoryUsed = 0;
    private long lastGameSaveTime = 0;

    @Override
    public String getId() {
        return PLUGIN_ID;
    }

    @Override
    public String getName() {
        return PLUGIN_NAME;
    }

    @Override
    public String getVersion() {
        return PLUGIN_VERSION;
    }

    @Override
    public String getAuthor() {
        return PLUGIN_AUTHOR;
    }

    @Override
    public String getDescription() {
        return PLUGIN_DESCRIPTION;
    }

    @Override
    public void initialize(PluginContext context) {
        super.initialize(context);
        LogUtils.logUser("🛠️ Initializing Utility Plugin");
        
        pluginStartTime = System.currentTimeMillis();
        uiHandler = new Handler(Looper.getMainLooper());
        
        loadConfiguration();
        initializeUtilities();
        
        LogUtils.logUser("✅ Utility Plugin initialized successfully");
    }

    @Override
    public void onEnable() {
        super.onEnable();
        LogUtils.logUser("🛠️ Enabling utility tools...");
        
        try {
            startAutoSave();
            startPerformanceMonitoring();
            registerUtilityCommands();
            
            if (enableSystemInfo) {
                displaySystemInfo();
            }
            
            showNotification("Utility tools enabled! Auto-save every " + autoSaveInterval + "s");
            LogUtils.logUser("✅ Utility tools enabled successfully");
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to enable utility tools: " + e.getMessage());
            showNotification("Failed to enable utility tools: " + e.getMessage());
        }
    }

    @Override
    public void onDisable() {
        super.onDisable();
        LogUtils.logUser("🛠️ Disabling utility tools...");
        
        try {
            stopAutoSave();
            stopPerformanceMonitoring();
            unregisterUtilityCommands();
            
            showNotification("Utility tools disabled - statistics saved");
            LogUtils.logUser("✅ Utility tools disabled successfully");
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to disable utility tools: " + e.getMessage());
        }
    }

    @Override
    public void onGameStart() {
        LogUtils.logUser("🎮 Game starting - activating utility monitoring");
        
        if (isEnabled()) {
            recordGameEvent("Game Started");
            if (enablePerformanceMonitoring) {
                startPerformanceBaseline();
            }
        }
    }

    @Override
    public void onGameStop() {
        LogUtils.logUser("🎮 Game stopping - saving utility data");
        
        if (isEnabled()) {
            recordGameEvent("Game Stopped");
            saveUtilityData();
        }
    }

    @Override
    public Map<String, Object> getConfigurationOptions() {
        Map<String, Object> config = new HashMap<>();
        config.put("enableAutoSave", enableAutoSave);
        config.put("autoSaveInterval", autoSaveInterval);
        config.put("enablePerformanceMonitoring", enablePerformanceMonitoring);
        config.put("performanceCheckInterval", performanceCheckInterval);
        config.put("enableSystemInfo", enableSystemInfo);
        config.put("enableNotifications", enableNotifications);
        config.put("enableDebugMode", enableDebugMode);
        return config;
    }

    @Override
    public void setConfigurationOptions(Map<String, Object> config) {
        if (config.containsKey("enableAutoSave")) {
            enableAutoSave = (Boolean) config.get("enableAutoSave");
        }
        if (config.containsKey("autoSaveInterval")) {
            autoSaveInterval = ((Number) config.get("autoSaveInterval")).intValue();
        }
        if (config.containsKey("enablePerformanceMonitoring")) {
            enablePerformanceMonitoring = (Boolean) config.get("enablePerformanceMonitoring");
        }
        if (config.containsKey("performanceCheckInterval")) {
            performanceCheckInterval = ((Number) config.get("performanceCheckInterval")).intValue();
        }
        if (config.containsKey("enableSystemInfo")) {
            enableSystemInfo = (Boolean) config.get("enableSystemInfo");
        }
        if (config.containsKey("enableNotifications")) {
            enableNotifications = (Boolean) config.get("enableNotifications");
        }
        if (config.containsKey("enableDebugMode")) {
            enableDebugMode = (Boolean) config.get("enableDebugMode");
        }
        
        LogUtils.logDebug("Configuration updated: auto-save=" + autoSaveInterval + "s, perf-check=" + performanceCheckInterval + "s");
        
        // Apply changes immediately if plugin is enabled
        if (isEnabled()) {
            restartUtilities();
        }
    }

    // ===== Utility Methods =====

    private void loadConfiguration() {
        try {
            Map<String, Object> saved = getPluginContext().getStorage().loadData("config");
            if (saved != null && !saved.isEmpty()) {
                setConfigurationOptions(saved);
                LogUtils.logDebug("Loaded utility configuration from storage");
            }
        } catch (Exception e) {
            LogUtils.logDebug("Failed to load utility configuration: " + e.getMessage());
        }
    }

    private void saveConfiguration() {
        try {
            getPluginContext().getStorage().saveData("config", getConfigurationOptions());
            LogUtils.logDebug("Utility configuration saved successfully");
        } catch (Exception e) {
            LogUtils.logDebug("Failed to save utility configuration: " + e.getMessage());
        }
    }

    private void initializeUtilities() {
        performanceLog.clear();
        autoSaveCount = 0;
        performanceChecks = 0;
        totalMemoryUsed = 0;
        
        LogUtils.logDebug("Utility components initialized");
    }

    private void restartUtilities() {
        LogUtils.logDebug("Restarting utilities with new configuration");
        
        stopAutoSave();
        stopPerformanceMonitoring();
        
        if (enableAutoSave) {
            startAutoSave();
        }
        if (enablePerformanceMonitoring) {
            startPerformanceMonitoring();
        }
        
        saveConfiguration();
        showNotification("Utility settings updated!");
    }

    // ===== Auto-Save Functionality =====

    private void startAutoSave() {
        if (!enableAutoSave || autoSaveTimer != null) {
            return;
        }

        autoSaveTimer = new Timer("AutoSaveTimer");
        autoSaveTimer.scheduleAtFixedRate(new TimerTask() {
            @Override
            public void run() {
                performAutoSave();
            }
        }, autoSaveInterval * 1000L, autoSaveInterval * 1000L);

        LogUtils.logDebug("Auto-save started: interval " + autoSaveInterval + " seconds");
    }

    private void stopAutoSave() {
        if (autoSaveTimer != null) {
            autoSaveTimer.cancel();
            autoSaveTimer = null;
            LogUtils.logDebug("Auto-save stopped");
        }
    }

    private void performAutoSave() {
        try {
            PluginAPI api = getPluginContext().getAPI();
            
            if (api.hasGameAccess()) {
                api.executeGameCommand("save");
                lastGameSaveTime = System.currentTimeMillis();
                autoSaveCount++;
                
                if (enableDebugMode) {
                    LogUtils.logDebug("Auto-save #" + autoSaveCount + " completed");
                }
                
                if (autoSaveCount % 10 == 0) {
                    uiHandler.post(() -> showNotification("Auto-saved! (" + autoSaveCount + " total saves)"));
                }
            }
            
            saveUtilityData();
            
        } catch (Exception e) {
            LogUtils.logDebug("Auto-save failed: " + e.getMessage());
        }
    }

    // ===== Performance Monitoring =====

    private void startPerformanceMonitoring() {
        if (!enablePerformanceMonitoring || performanceMonitor != null) {
            return;
        }

        performanceMonitor = new Timer("PerformanceMonitor");
        performanceMonitor.scheduleAtFixedRate(new TimerTask() {
            @Override
            public void run() {
                performPerformanceCheck();
            }
        }, performanceCheckInterval * 1000L, performanceCheckInterval * 1000L);

        LogUtils.logDebug("Performance monitoring started: interval " + performanceCheckInterval + " seconds");
    }

    private void stopPerformanceMonitoring() {
        if (performanceMonitor != null) {
            performanceMonitor.cancel();
            performanceMonitor = null;
            LogUtils.logDebug("Performance monitoring stopped");
        }
    }

    private void performPerformanceCheck() {
        try {
            Runtime runtime = Runtime.getRuntime();
            long totalMemory = runtime.totalMemory();
            long freeMemory = runtime.freeMemory();
            long usedMemory = totalMemory - freeMemory;
            
            totalMemoryUsed += usedMemory;
            performanceChecks++;
            
            String timestamp = new SimpleDateFormat("HH:mm:ss", Locale.getDefault()).format(new Date());
            String logEntry = timestamp + " - Memory: " + formatMemory(usedMemory) + 
                            " (Total: " + formatMemory(totalMemory) + ")";
            
            performanceLog.add(logEntry);
            
            if (performanceLog.size() > 100) {
                performanceLog.remove(0);
            }
            
            if (enableDebugMode) {
                LogUtils.logDebug("Performance check #" + performanceChecks + ": " + logEntry);
            }
            
            double memoryUsagePercent = (double) usedMemory / totalMemory * 100;
            if (memoryUsagePercent > 80) {
                uiHandler.post(() -> showNotification("High memory usage: " + String.format("%.1f%%", memoryUsagePercent)));
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Performance check failed: " + e.getMessage());
        }
    }

    private void startPerformanceBaseline() {
        LogUtils.logDebug("Starting performance baseline measurement");
        recordGameEvent("Performance Baseline Started");
    }

    // ===== System Information =====

    private void displaySystemInfo() {
        try {
            Runtime runtime = Runtime.getRuntime();
            String systemInfo = buildSystemInfoString(runtime);
            
            LogUtils.logUser("📊 System Information:\n" + systemInfo);
            
            if (enableNotifications) {
                showNotification("System info logged - check debug output");
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to display system info: " + e.getMessage());
        }
    }

    private String buildSystemInfoString(Runtime runtime) {
        StringBuilder info = new StringBuilder();
        info.append("=== System Information ===\n");
        info.append("Available Processors: ").append(runtime.availableProcessors()).append("\n");
        info.append("Max Memory: ").append(formatMemory(runtime.maxMemory())).append("\n");
        info.append("Total Memory: ").append(formatMemory(runtime.totalMemory())).append("\n");
        info.append("Free Memory: ").append(formatMemory(runtime.freeMemory())).append("\n");
        info.append("Used Memory: ").append(formatMemory(runtime.totalMemory() - runtime.freeMemory())).append("\n");
        info.append("Android Version: ").append(android.os.Build.VERSION.RELEASE).append("\n");
        info.append("SDK Level: ").append(android.os.Build.VERSION.SDK_INT).append("\n");
        info.append("Device Model: ").append(android.os.Build.MODEL).append("\n");
        info.append("Manufacturer: ").append(android.os.Build.MANUFACTURER).append("\n");
        return info.toString();
    }

    private String formatMemory(long bytes) {
        if (bytes < 1024) return bytes + " B";
        if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
        return String.format("%.1f MB", bytes / (1024.0 * 1024.0));
    }

    // ===== Utility Commands =====

    private void registerUtilityCommands() {
        PluginAPI api = getPluginContext().getAPI();
        
        // Register utility commands
        api.registerCommand("save", this::executeManualSave);
        api.registerCommand("perfcheck", this::executePerformanceCheck);
        api.registerCommand("sysinfo", this::executeSystemInfo);
        api.registerCommand("stats", this::executeShowStats);
        
        LogUtils.logDebug("Utility commands registered");
    }

    private void unregisterUtilityCommands() {
        PluginAPI api = getPluginContext().getAPI();
        
        api.unregisterCommand("save");
        api.unregisterCommand("perfcheck");
        api.unregisterCommand("sysinfo");
        api.unregisterCommand("stats");
        
        LogUtils.logDebug("Utility commands unregistered");
    }

    // Command implementations
    private void executeManualSave(String... args) {
        LogUtils.logUser("🔄 Manual save triggered");
        performAutoSave();
        showNotification("Manual save completed!");
    }

    private void executePerformanceCheck(String... args) {
        LogUtils.logUser("📊 Manual performance check");
        performPerformanceCheck();
        showNotification("Performance check completed - see logs");
    }

    private void executeSystemInfo(String... args) {
        LogUtils.logUser("📋 System information requested");
        displaySystemInfo();
    }

    private void executeShowStats(String... args) {
        String stats = getStatusReport();
        LogUtils.logUser("📈 Utility Plugin Statistics:\n" + stats);
        showNotification("Statistics displayed in logs");
    }

    // ===== Data Management =====

    private void saveUtilityData() {
        try {
            Map<String, Object> data = new HashMap<>();
            data.put("autoSaveCount", autoSaveCount);
            data.put("performanceChecks", performanceChecks);
            data.put("totalMemoryUsed", totalMemoryUsed);
            data.put("lastGameSaveTime", lastGameSaveTime);
            data.put("performanceLog", new ArrayList<>(performanceLog));
            data.put("pluginStartTime", pluginStartTime);
            
            getPluginContext().getStorage().saveData("statistics", data);
            
            if (enableDebugMode) {
                LogUtils.logDebug("Utility data saved successfully");
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to save utility data: " + e.getMessage());
        }
    }

    private void loadUtilityData() {
        try {
            @SuppressWarnings("unchecked")
            Map<String, Object> data = getPluginContext().getStorage().loadData("statistics");
            
            if (data != null) {
                if (data.containsKey("autoSaveCount")) {
                    autoSaveCount = ((Number) data.get("autoSaveCount")).intValue();
                }
                if (data.containsKey("performanceChecks")) {
                    performanceChecks = ((Number) data.get("performanceChecks")).intValue();
                }
                if (data.containsKey("totalMemoryUsed")) {
                    totalMemoryUsed = ((Number) data.get("totalMemoryUsed")).longValue();
                }
                if (data.containsKey("lastGameSaveTime")) {
                    lastGameSaveTime = ((Number) data.get("lastGameSaveTime")).longValue();
                }
                if (data.containsKey("performanceLog")) {
                    @SuppressWarnings("unchecked")
                    List<String> savedLog = (List<String>) data.get("performanceLog");
                    performanceLog.clear();
                    performanceLog.addAll(savedLog);
                }
                
                LogUtils.logDebug("Utility data loaded successfully");
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to load utility data: " + e.getMessage());
        }
    }

    private void recordGameEvent(String event) {
        String timestamp = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault()).format(new Date());
        String logEntry = timestamp + " - " + event;
        performanceLog.add(logEntry);
        
        if (enableDebugMode) {
            LogUtils.logDebug("Game event recorded: " + event);
        }
    }

    // ===== Utility Methods =====

    private void showNotification(String message) {
        if (enableNotifications) {
            Context context = getPluginContext().getApplicationContext();
            if (context != null) {
                Toast.makeText(context, "[Utility] " + message, Toast.LENGTH_SHORT).show();
            }
        }
    }

    @Override
    public String getStatusReport() {
        StringBuilder report = new StringBuilder();
        report.append("=== Utility Plugin Status ===\n");
        report.append("Plugin Enabled: ").append(isEnabled()).append("\n");
        report.append("Plugin Runtime: ").append(formatRuntime()).append("\n");
        
        report.append("\nAuto-Save:\n");
        report.append("• Enabled: ").append(enableAutoSave).append("\n");
        report.append("• Interval: ").append(autoSaveInterval).append("s\n");
        report.append("• Total Saves: ").append(autoSaveCount).append("\n");
        if (lastGameSaveTime > 0) {
            report.append("• Last Save: ").append(new Date(lastGameSaveTime)).append("\n");
        }
        
        report.append("\nPerformance Monitoring:\n");
        report.append("• Enabled: ").append(enablePerformanceMonitoring).append("\n");
        report.append("• Check Interval: ").append(performanceCheckInterval).append("s\n");
        report.append("• Total Checks: ").append(performanceChecks).append("\n");
        report.append("• Avg Memory Usage: ").append(getAverageMemoryUsage()).append("\n");
        
        report.append("\nSettings:\n");
        report.append("• System Info: ").append(enableSystemInfo).append("\n");
        report.append("• Notifications: ").append(enableNotifications).append("\n");
        report.append("• Debug Mode: ").append(enableDebugMode).append("\n");
        
        report.append("\nPerformance Log Entries: ").append(performanceLog.size()).append("\n");
        
        return report.toString();
    }

    private String formatRuntime() {
        long runtimeMs = System.currentTimeMillis() - pluginStartTime;
        long seconds = runtimeMs / 1000;
        long minutes = seconds / 60;
        long hours = minutes / 60;
        
        if (hours > 0) {
            return String.format("%dh %dm %ds", hours, minutes % 60, seconds % 60);
        } else if (minutes > 0) {
            return String.format("%dm %ds", minutes, seconds % 60);
        } else {
            return String.format("%ds", seconds);
        }
    }

    private String getAverageMemoryUsage() {
        if (performanceChecks == 0) {
            return "No data";
        }
        long avgMemory = totalMemoryUsed / performanceChecks;
        return formatMemory(avgMemory);
    }

    @Override
    public void cleanup() {
        LogUtils.logUser("🧹 Cleaning up Utility Plugin");
        
        try {
            if (isEnabled()) {
                onDisable();
            }
            
            saveUtilityData();
            saveConfiguration();
            
            performanceLog.clear();
            
            LogUtils.logUser("✅ Utility Plugin cleanup completed");
            
        } catch (Exception e) {
            LogUtils.logDebug("Error during utility plugin cleanup: " + e.getMessage());
        }
        
        super.cleanup();
    }

    // ===== Public Utility Methods for Other Plugins =====

    public List<String> getPerformanceHistory() {
        return new ArrayList<>(performanceLog);
    }

    public int getAutoSaveCount() {
        return autoSaveCount;
    }

    public void triggerManualSave() {
        if (isEnabled()) {
            performAutoSave();
        }
    }

    public boolean isPerformanceMonitoringActive() {
        return enablePerformanceMonitoring && performanceMonitor != null;
    }

    public void exportUtilityReport() {
        try {
            String report = getStatusReport() + "\n\nPerformance History:\n";
            for (String entry : performanceLog) {
                report += entry + "\n";
            }
            
            getPluginContext().getStorage().saveData("utility_report_" + System.currentTimeMillis(), report);
            showNotification("Utility report exported to plugin storage");
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to export utility report: " + e.getMessage());
        }
    }
}


/ModLoader/app/src/main/java/com/modloader/ui/BaseActivity.java

// File: BaseActivity.java (FIXED) - Compatible with PermissionManager
// Path: /app/src/main/java/com/modloader/ui/BaseActivity.java

package com.modloader.ui;

import android.content.Intent;
import android.os.Bundle;
import android.view.MenuItem;
import android.widget.Toast;
import androidx.annotation.NonNull;
import androidx.appcompat.app.AlertDialog;
import androidx.appcompat.app.AppCompatActivity;

import com.modloader.util.LogUtils;
import com.modloader.util.PermissionManager;
import com.modloader.util.ShizukuManager;
import com.modloader.util.RootManager;

/**
 * Base activity that provides common functionality for all activities
 * including permission management, error handling, and utility methods
 */
public abstract class BaseActivity extends AppCompatActivity implements PermissionManager.PermissionCallback {
    
    protected static final String TAG = "BaseActivity";
    
    // Common managers
    protected PermissionManager permissionManager;
    protected ShizukuManager shizukuManager;
    protected RootManager rootManager;
    
    // Activity state
    private boolean isResumed = false;
    private boolean hasInitialized = false;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        LogUtils.logDebug(getClass().getSimpleName() + " onCreate started");
        
        try {
            // Initialize managers
            initializeManagers();
            
            // Setup common UI elements
            setupCommonUI();
            
            // Auto-setup permissions if needed
            if (shouldAutoSetupPermissions()) {
                autoSetupPermissions();
            }
            
            hasInitialized = true;
            LogUtils.logDebug(getClass().getSimpleName() + " onCreate completed");
            
        } catch (Exception e) {
            LogUtils.logError("Error in BaseActivity onCreate: " + e.getMessage());
            handleStartupError(e);
        }
    }
    
    /**
     * Initialize common managers
     */
    private void initializeManagers() {
        try {
            permissionManager = new PermissionManager(this);
            permissionManager.setCallback(this);
            
            shizukuManager = new ShizukuManager(this);
            rootManager = new RootManager(this);
            
            LogUtils.logDebug("Managers initialized successfully");
        } catch (Exception e) {
            LogUtils.logError("Error initializing managers: " + e.getMessage());
        }
    }
    
    /**
     * Setup common UI elements
     */
    private void setupCommonUI() {
        // Enable up navigation if not main activity
        if (getSupportActionBar() != null && !isMainActivity()) {
            getSupportActionBar().setDisplayHomeAsUpEnabled(true);
        }
    }
    
    /**
     * Auto-setup permissions if the activity requires them
     */
    private void autoSetupPermissions() {
        if (permissionManager != null && requiresPermissions()) {
            permissionManager.autoSetupPermissions();
        }
    }
    
    /**
     * Check if this activity has all required permissions
     */
    protected boolean hasRequiredPermissions() {
        if (permissionManager != null) {
            return permissionManager.hasAllRequiredPermissions();
        }
        return true; // Assume true if manager not available
    }
    
    /**
     * Handle startup errors gracefully
     */
    private void handleStartupError(Exception e) {
        LogUtils.logError("Startup error in " + getClass().getSimpleName() + ": " + e.getMessage());
        
        new AlertDialog.Builder(this)
            .setTitle("❌ Startup Error")
            .setMessage("An error occurred while starting this activity:\n\n" + e.getMessage() + 
                "\n\nThe app may not function correctly.")
            .setPositiveButton("Continue", null)
            .setNegativeButton("Close App", (dialog, which) -> finishAffinity())
            .show();
    }
    
    // === Permission Management Methods ===
    
    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        
        LogUtils.logDebug("Permission result in " + getClass().getSimpleName() + 
            ": requestCode=" + requestCode);
        
        if (permissionManager != null) {
            permissionManager.handlePermissionResult(requestCode, permissions, grantResults);
        }
    }
    
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        
        LogUtils.logDebug("Activity result in " + getClass().getSimpleName() + 
            ": requestCode=" + requestCode + ", resultCode=" + resultCode);
        
        // Handle special permission results
        if (permissionManager != null) {
            switch (requestCode) {
                case PermissionManager.REQUEST_ALL_FILES_ACCESS:
                case PermissionManager.REQUEST_INSTALL_PERMISSION:
                    permissionManager.handlePermissionResult(requestCode, null, null);
                    break;
            }
        }
        
        // Let subclasses handle their specific results
        handleActivityResult(requestCode, resultCode, data);
    }
    
    /**
     * Subclasses can override this to handle activity results
     */
    protected void handleActivityResult(int requestCode, int resultCode, Intent data) {
        // Default implementation does nothing
    }
    
    // === PermissionManager.PermissionCallback Implementation ===
    
    @Override
    public void onPermissionGranted(String permission) {
        LogUtils.logUser("✅ Permission granted: " + permission);
        onPermissionStateChanged();
    }
    
    @Override
    public void onPermissionDenied(String permission) {
        LogUtils.logUser("❌ Permission denied: " + permission);
        handlePermissionDenied(permission);
        onPermissionStateChanged();
    }
    
    @Override
    public void onPermissionPermanentlyDenied(String permission) {
        LogUtils.logUser("⚠️ Permission permanently denied: " + permission);
        handlePermissionPermanentlyDenied(permission);
        onPermissionStateChanged();
    }
    
    @Override
    public void onAllPermissionsGranted() {
        LogUtils.logUser("✅ All permissions granted!");
        onPermissionStateChanged();
        onAllPermissionsReady();
    }
    
    @Override
    public void onPermissionRequestCompleted(boolean allGranted) {
        LogUtils.logDebug("Permission request completed: " + allGranted);
        onPermissionStateChanged();
    }
    
    /**
     * Called when permission state changes - subclasses can override
     */
    protected void onPermissionStateChanged() {
        // Update UI or refresh content based on new permission state
        refreshPermissionStatus();
    }
    
    /**
     * Called when all permissions are ready - subclasses can override
     */
    protected void onAllPermissionsReady() {
        // Subclasses can implement specific behavior when all permissions are granted
    }
    
    /**
     * Handle permission denial with user-friendly messaging
     */
    protected void handlePermissionDenied(String permission) {
        String message = getPermissionDenialMessage(permission);
        showToast(message);
    }
    
    /**
     * Handle permanently denied permissions
     */
    protected void handlePermissionPermanentlyDenied(String permission) {
        new AlertDialog.Builder(this)
            .setTitle("🔐 Permission Required")
            .setMessage("The permission for " + permission + " has been permanently denied.\n\n" +
                "To enable it, please:\n" +
                "1. Go to App Settings\n" +
                "2. Find Permissions\n" +
                "3. Enable the required permission")
            .setPositiveButton("Open Settings", (dialog, which) -> {
                if (permissionManager != null) {
                    permissionManager.openAppSettings();
                }
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    /**
     * Get user-friendly message for permission denial
     */
    private String getPermissionDenialMessage(String permission) {
        switch (permission) {
            case "MANAGE_EXTERNAL_STORAGE":
                return "Storage access is required to manage mod files";
            case "INSTALL_PACKAGES":
                return "Install permission is required to install modded APKs";
            default:
                return "This permission is required for the app to function properly";
        }
    }
    
    /**
     * Refresh permission status display
     */
    protected void refreshPermissionStatus() {
        // Subclasses can override to update permission-related UI
        if (permissionManager != null) {
            boolean hasAll = permissionManager.hasAllRequiredPermissions();
            LogUtils.logDebug("Permission status refreshed: " + (hasAll ? "All granted" : "Missing some"));
        }
    }
    
    // === Utility Methods ===
    
    /**
     * Show permission status dialog
     */
    protected void showPermissionStatus() {
        if (permissionManager != null) {
            String status = permissionManager.getPermissionStatusText();
            new AlertDialog.Builder(this)
                .setTitle("🔐 Permission Status")
                .setMessage(status)
                .setPositiveButton("OK", null)
                .setNeutralButton("Request Missing", (dialog, which) -> {
                    permissionManager.requestAllPermissions();
                })
                .show();
        }
    }
    
    /**
     * Show toast message
     */
    protected void showToast(String message) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show();
    }
    
    /**
     * Show long toast message
     */
    protected void showLongToast(String message) {
        Toast.makeText(this, message, Toast.LENGTH_LONG).show();
    }
    
    /**
     * Show error dialog
     */
    protected void showError(String title, String message) {
        new AlertDialog.Builder(this)
            .setTitle("❌ " + title)
            .setMessage(message)
            .setPositiveButton("OK", null)
            .show();
    }
    
    /**
     * Show confirmation dialog
     */
    protected void showConfirmation(String title, String message, Runnable onConfirm) {
        new AlertDialog.Builder(this)
            .setTitle(title)
            .setMessage(message)
            .setPositiveButton("Yes", (dialog, which) -> {
                if (onConfirm != null) {
                    onConfirm.run();
                }
            })
            .setNegativeButton("No", null)
            .show();
    }
    
    // === Navigation Methods ===
    
    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        if (item.getItemId() == android.R.id.home) {
            onBackPressed();
            return true;
        }
        return super.onOptionsItemSelected(item);
    }
    
    /**
     * Safe finish that handles edge cases
     */
    protected void safeFinish() {
        try {
            if (!isFinishing()) {
                finish();
            }
        } catch (Exception e) {
            LogUtils.logDebug("Error during safeFinish: " + e.getMessage());
        }
    }
    
    // === Lifecycle Methods ===
    
    @Override
    protected void onResume() {
        super.onResume();
        isResumed = true;
        
        // Refresh permission states when returning from other apps
        if (hasInitialized && permissionManager != null) {
            permissionManager.refreshPermissionStates();
            refreshPermissionStatus();
        }
        
        LogUtils.logDebug(getClass().getSimpleName() + " onResume");
    }
    
    @Override
    protected void onPause() {
        super.onPause();
        isResumed = false;
        LogUtils.logDebug(getClass().getSimpleName() + " onPause");
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        
        // Clean up managers
        if (permissionManager != null) {
            permissionManager.cleanup();
        }
        if (shizukuManager != null) {
            shizukuManager.cleanup();
        }
        if (rootManager != null) {
            rootManager.cleanup();
        }
        
        LogUtils.logDebug(getClass().getSimpleName() + " onDestroy");
    }
    
    // === Abstract/Virtual Methods for Subclasses ===
    
    /**
     * Whether this activity should auto-setup permissions on create
     */
    protected boolean shouldAutoSetupPermissions() {
        return requiresPermissions();
    }
    
    /**
     * Whether this activity requires permissions to function
     */
    protected boolean requiresPermissions() {
        return true; // Most activities require permissions
    }
    
    /**
     * Whether this is the main activity (affects navigation setup)
     */
    protected boolean isMainActivity() {
        return false; // Subclasses should override if they are main activities
    }
    
    // === State Query Methods ===
    
    protected boolean isActivityResumed() {
        return isResumed;
    }
    
    protected boolean isActivityInitialized() {
        return hasInitialized;
    }
    
    /**
     * Get the permission manager instance
     */
    protected PermissionManager getPermissionManager() {
        return permissionManager;
    }
    
    /**
     * Get the Shizuku manager instance
     */
    protected ShizukuManager getShizukuManager() {
        return shizukuManager;
    }
    
    /**
     * Get the root manager instance
     */
    protected RootManager getRootManager() {
        return rootManager;
    }
}


/ModLoader/app/src/main/java/com/modloader/ui/DllModActivity.java

// File: DllModActivity.java (Updated Activity) - Uses Controller Pattern
// Path: /storage/emulated/0/AndroidIDEProjects/main/java/com/terrarialoader/ui/DllModActivity.java

package com.modloader.ui;

import android.app.Activity;
import android.app.AlertDialog;
import android.content.Intent;
import android.database.Cursor;
import android.net.Uri;
import android.os.Bundle;
import android.provider.OpenableColumns;
import android.view.View;
import android.widget.Button;
import android.widget.LinearLayout;
import android.widget.TextView;
import android.widget.Toast;

import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;

import com.modloader.R;
import com.modloader.util.LogUtils;

import java.io.File;
import java.util.List;

/**
 * DllModActivity now delegates business logic to DllModController
 * This keeps the activity focused on UI management while the controller handles the logic
 */
public class DllModActivity extends Activity implements DllModController.DllModCallback {

    private static final int REQUEST_SELECT_APK = 1001;
    private static final int REQUEST_SELECT_DLL = 1002;
    
    // UI Components
    private TextView statusText;
    private TextView loaderStatusText;
    private RecyclerView dllModRecyclerView;
    private Button installLoaderBtn;
    private Button selectApkBtn;
    private Button installDllBtn;
    private Button viewLogsBtn;
    private Button refreshBtn;
    private LinearLayout loaderInfoSection;
    
    // Controller and Adapter
    private DllModController controller;
    private DllModAdapter adapter;
    private AlertDialog progressDialog;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_dll_mod);
        
        setTitle("DLL Mod Manager");
        
        // Initialize controller
        controller = new DllModController(this);
        controller.setCallback(this);
        
        initializeViews();
        setupUI();
        
        // Load initial data
        controller.loadDllMods();
        controller.updateLoaderStatus();
    }

    private void initializeViews() {
        statusText = findViewById(R.id.statusText);
        loaderStatusText = findViewById(R.id.loaderStatusText);
        dllModRecyclerView = findViewById(R.id.dllModRecyclerView);
        installLoaderBtn = findViewById(R.id.installLoaderBtn);
        selectApkBtn = findViewById(R.id.selectApkBtn);
        installDllBtn = findViewById(R.id.installDllBtn);
        viewLogsBtn = findViewById(R.id.viewLogsBtn);
        refreshBtn = findViewById(R.id.refreshBtn);
        loaderInfoSection = findViewById(R.id.loaderInfoSection);
    }

    private void setupUI() {
        // Setup RecyclerView
        dllModRecyclerView.setLayoutManager(new LinearLayoutManager(this));
        adapter = new DllModAdapter(this, controller);
        dllModRecyclerView.setAdapter(adapter);

        // Button listeners - delegate to controller
        installLoaderBtn.setOnClickListener(v -> controller.showLoaderInstallDialog());
        
        selectApkBtn.setOnClickListener(v -> {
            Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
            intent.setType("application/vnd.android.package-archive");
            intent.addCategory(Intent.CATEGORY_OPENABLE);
            startActivityForResult(intent, REQUEST_SELECT_APK);
        });
        
        installDllBtn.setOnClickListener(v -> {
            Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
            intent.setType("*/*"); // Allow all files, we'll filter by extension
            intent.addCategory(Intent.CATEGORY_OPENABLE);
            startActivityForResult(intent, REQUEST_SELECT_DLL);
        });
        
        viewLogsBtn.setOnClickListener(v -> {
            startActivity(new Intent(this, LogViewerActivity.class));
        });
        
        refreshBtn.setOnClickListener(v -> {
            controller.refresh();
            Toast.makeText(this, "Refreshed DLL mod list", Toast.LENGTH_SHORT).show();
        });
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        
        if (resultCode != RESULT_OK || data == null || data.getData() == null) {
            return;
        }
        
        Uri uri = data.getData();
        
        switch (requestCode) {
            case REQUEST_SELECT_APK:
                controller.handleApkSelection(uri);
                break;
                
            case REQUEST_SELECT_DLL:
                String filename = getFilenameFromUri(uri);
                controller.handleDllSelection(uri, filename);
                break;
        }
    }

    private String getFilenameFromUri(Uri uri) {
        String filename = null;
        
        try (Cursor cursor = getContentResolver().query(uri, null, null, null, null)) {
            if (cursor != null && cursor.moveToFirst()) {
                int nameIndex = cursor.getColumnIndex(OpenableColumns.DISPLAY_NAME);
                if (nameIndex >= 0) {
                    filename = cursor.getString(nameIndex);
                }
            }
        } catch (Exception e) {
            LogUtils.logDebug("Could not get filename from URI: " + e.getMessage());
        }
        
        return filename;
    }

    @Override
    protected void onResume() {
        super.onResume();
        controller.refresh();
    }

    // === DllModController.DllModCallback Implementation ===
    
    @Override
    public void onModsLoaded(List<File> mods) {
        runOnUiThread(() -> {
            adapter.updateMods(mods);
            statusText.setText(controller.getStatusText());
        });
    }

    @Override
    public void onLoaderStatusChanged(boolean installed, String statusText, int textColor) {
        runOnUiThread(() -> {
            loaderStatusText.setText(statusText);
            loaderStatusText.setTextColor(textColor);
            
            if (installed) {
                installLoaderBtn.setText("Reinstall Loader");
                loaderInfoSection.setVisibility(View.VISIBLE);
                installDllBtn.setEnabled(true);
                installDllBtn.setText("Install DLL Mod");
            } else {
                installLoaderBtn.setText("Install Loader");
                loaderInfoSection.setVisibility(View.GONE);
                installDllBtn.setEnabled(false);
                installDllBtn.setText("Install Loader First");
            }
        });
    }

    @Override
    public void onInstallationProgress(String message) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.dismiss();
            }
            
            if (message.contains("Please select")) {
                // This is a request for user action, not a progress message
                return;
            }
            
            progressDialog = new AlertDialog.Builder(this)
                .setTitle("Installing Loader")
                .setMessage(message)
                .setCancelable(false)
                .show();
        });
    }

    @Override
    public void onInstallationComplete(boolean success, String message) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.dismiss();
                progressDialog = null;
            }
            
            if (success) {
                Toast.makeText(this, message, Toast.LENGTH_SHORT).show();
            } else {
                Toast.makeText(this, message, Toast.LENGTH_LONG).show();
            }
        });
    }

    @Override
    public void onError(String error) {
        runOnUiThread(() -> {
            Toast.makeText(this, error, Toast.LENGTH_SHORT).show();
        });
    }

    // === Simple adapter class for DLL mods ===
    private static class DllModAdapter extends RecyclerView.Adapter<DllModAdapter.ViewHolder> {
        private final Activity activity;
        private final DllModController controller;
        private List<File> mods;

        public DllModAdapter(Activity activity, DllModController controller) {
            this.activity = activity;
            this.controller = controller;
            this.mods = controller.getDllMods();
        }

        public void updateMods(List<File> newMods) {
            this.mods = newMods;
            notifyDataSetChanged();
        }

        @Override
        public ViewHolder onCreateViewHolder(android.view.ViewGroup parent, int viewType) {
            View view = android.view.LayoutInflater.from(parent.getContext())
                .inflate(R.layout.item_mod, parent, false);
            return new ViewHolder(view);
        }

        @Override
        public void onBindViewHolder(ViewHolder holder, int position) {
            File mod = mods.get(position);
            String name = mod.getName();
            boolean isEnabled = !name.endsWith(".disabled");
            
            holder.modDescription.setText(name);
            holder.modSwitch.setChecked(isEnabled);
            
            holder.modSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
                controller.toggleMod(mod, isChecked);
            });
            
            holder.modDeleteButton.setOnClickListener(v -> {
                controller.deleteMod(mod, () -> {
                    mods.remove(position);
                    notifyItemRemoved(position);
                    notifyItemRangeChanged(position, mods.size());
                });
            });
        }

        @Override
        public int getItemCount() {
            return mods.size();
        }

        static class ViewHolder extends RecyclerView.ViewHolder {
            TextView modDescription;
            android.widget.Switch modSwitch;
            android.widget.ImageButton modDeleteButton;

            ViewHolder(View itemView) {
                super(itemView);
                modDescription = itemView.findViewById(R.id.modDescription);
                modSwitch = itemView.findViewById(R.id.modSwitch);
                modDeleteButton = itemView.findViewById(R.id.modDeleteButton);
            }
        }
    }
}


/ModLoader/app/src/main/java/com/modloader/ui/DllModController.java

// File: DllModController.java (Fixed) - Corrected method calls with Context parameter
// Path: /storage/emulated/0/AndroidIDEProjects/TerrariaML/app/src/main/java/com/terrarialoader/ui/DllModController.java

package com.modloader.ui;

import android.app.Activity;
import android.app.AlertDialog;
import android.content.Intent;
import android.net.Uri;
import com.modloader.loader.ModManager;
import com.modloader.loader.ModBase;
import com.modloader.loader.MelonLoaderManager;
import com.modloader.util.LogUtils;
import com.modloader.util.FileUtils;
import com.modloader.installer.ModInstaller;
import java.io.File;
import java.util.ArrayList;
import java.util.List;

public class DllModController {
    private final Activity activity;
    private List<File> dllMods = new ArrayList<>();
    
    // Callback interface for UI updates
    public interface DllModCallback {
        void onModsLoaded(List<File> mods);
        void onLoaderStatusChanged(boolean installed, String statusText, int textColor);
        void onInstallationProgress(String message);
        void onInstallationComplete(boolean success, String message);
        void onError(String error);
    }
    
    private DllModCallback callback;

    public DllModController(Activity activity) {
        this.activity = activity;
    }
    
    public void setCallback(DllModCallback callback) {
        this.callback = callback;
    }

    public void loadDllMods() {
        dllMods.clear();
        
        // Get DLL mods from ModManager
        List<File> allMods = ModManager.getAvailableMods();
        for (File mod : allMods) {
            ModBase.ModType type = ModBase.ModType.fromFileName(mod.getName());
            if (type == ModBase.ModType.DLL || type == ModBase.ModType.HYBRID) {
                dllMods.add(mod);
            }
        }
        
        LogUtils.logDebug("Loaded " + dllMods.size() + " DLL mods");
        
        if (callback != null) {
            callback.onModsLoaded(dllMods);
        }
    }

    public void updateLoaderStatus() {
        // FIXED: Pass activity context to MelonLoaderManager methods
        boolean melonInstalled = MelonLoaderManager.isMelonLoaderInstalled(activity);
        boolean lemonInstalled = MelonLoaderManager.isLemonLoaderInstalled(activity);
        
        if (melonInstalled) {
            String statusText = "✅ MelonLoader " + MelonLoaderManager.getInstalledLoaderVersion() + " installed";
            if (callback != null) {
                callback.onLoaderStatusChanged(true, statusText, 0xFF4CAF50); // Green
            }
        } else if (lemonInstalled) {
            String statusText = "✅ LemonLoader " + MelonLoaderManager.getInstalledLoaderVersion() + " installed";
            if (callback != null) {
                callback.onLoaderStatusChanged(true, statusText, 0xFF4CAF50); // Green
            }
        } else {
            String statusText = "❌ No loader installed - DLL mods will not work";
            if (callback != null) {
                callback.onLoaderStatusChanged(false, statusText, 0xFFF44336); // Red
            }
        }
    }

    public void showLoaderInstallDialog() {
        AlertDialog.Builder builder = new AlertDialog.Builder(activity);
        builder.setTitle("Choose Loader");
        builder.setMessage("Select which loader to install:\n\n" +
                          "• MelonLoader: Full-featured, works with most Unity games\n" +
                          "• LemonLoader: Lightweight, better for older devices\n\n" +
                          "You will need to select a Terraria APK file after choosing.");
        
        builder.setPositiveButton("MelonLoader", (dialog, which) -> {
            LogUtils.logUser("User selected MelonLoader installation");
            showApkInstructions("MelonLoader");
        });
        
        builder.setNegativeButton("LemonLoader", (dialog, which) -> {
            LogUtils.logUser("User selected LemonLoader installation");
            showApkInstructions("LemonLoader");
        });
        
        builder.setNeutralButton("Cancel", null);
        builder.show();
    }

    public void showApkInstructions(String loaderType) {
        AlertDialog.Builder builder = new AlertDialog.Builder(activity);
        builder.setTitle("APK Installation Instructions");
        builder.setMessage("To install " + loaderType + ":\n\n" +
                          "1. Select your Terraria APK file\n" +
                          "2. The loader will be injected into the APK\n" +
                          "3. Install the modified APK\n" +
                          "4. Your DLL mods will work in Terraria\n\n" +
                          "Ready to select APK?");
        
        builder.setPositiveButton("Select APK", (dialog, which) -> {
            // Store the selected loader type for later use
            activity.getSharedPreferences("dll_mod_prefs", Activity.MODE_PRIVATE)
                .edit()
                .putString("selected_loader", loaderType)
                .apply();
            
            // Trigger APK selection in the UI
            if (callback != null) {
                callback.onInstallationProgress("Please select Terraria APK file");
            }
        });
        
        builder.setNegativeButton("Cancel", null);
        builder.show();
    }

    public void handleApkSelection(Uri apkUri) {
        LogUtils.logUser("APK selected for loader installation");
        
        // Get the selected loader type
        String selectedLoader = activity.getSharedPreferences("dll_mod_prefs", Activity.MODE_PRIVATE)
            .getString("selected_loader", "MelonLoader");
        
        // Show confirmation dialog
        AlertDialog.Builder builder = new AlertDialog.Builder(activity);
        builder.setTitle("Confirm Installation");
        builder.setMessage("Install " + selectedLoader + " into the selected APK?\n\n" +
                          "This will create a new patched APK file that you can install.");
        
        builder.setPositiveButton("Install", (dialog, which) -> {
            if ("MelonLoader".equals(selectedLoader)) {
                installLoaderIntoApk(apkUri, MelonLoaderManager.LoaderType.MELONLOADER_NET8);
            } else {
                installLoaderIntoApk(apkUri, MelonLoaderManager.LoaderType.MELONLOADER_NET35);
            }
        });
        
        builder.setNegativeButton("Cancel", null);
        builder.show();
    }

    public void handleDllSelection(Uri dllUri, String filename) {
        LogUtils.logUser("DLL file selected for installation");
        
        if (filename == null) {
            filename = "mod_" + System.currentTimeMillis() + ".dll";
        }
        
        if (!filename.toLowerCase().endsWith(".dll")) {
            if (callback != null) {
                callback.onError("Please select a .dll file");
            }
            return;
        }
        
        // FIXED: Pass activity context to MelonLoaderManager methods
        if (!MelonLoaderManager.isMelonLoaderInstalled(activity) && !MelonLoaderManager.isLemonLoaderInstalled(activity)) {
            showLoaderRequiredDialog();
            return;
        }
        
        // Install DLL mod
        boolean success = ModInstaller.installMod(activity, dllUri, filename);
        
        if (success) {
            if (callback != null) {
                callback.onInstallationComplete(true, "DLL mod installed successfully");
            }
            loadDllMods(); // Refresh list
        } else {
            if (callback != null) {
                callback.onInstallationComplete(false, "Failed to install DLL mod");
            }
        }
    }

    public void showLoaderRequiredDialog() {
        AlertDialog.Builder builder = new AlertDialog.Builder(activity);
        builder.setTitle("Loader Required");
        builder.setMessage("DLL mods require MelonLoader or LemonLoader to be installed first.\n\n" +
                          "Would you like to install a loader now?");
        
        builder.setPositiveButton("Install Loader", (dialog, which) -> {
            showLoaderInstallDialog();
        });
        
        builder.setNegativeButton("Cancel", null);
        builder.show();
    }

    public void installLoaderIntoApk(Uri apkUri, MelonLoaderManager.LoaderType loaderType) {
        LogUtils.logUser("Installing " + loaderType.getDisplayName() + " into APK");
        
        if (callback != null) {
            callback.onInstallationProgress("Installing " + loaderType.getDisplayName() + " into APK...\nThis may take a few minutes.");
        }
        
        // Run installation in background thread
        new Thread(() -> {
            boolean success = false;
            File outputApk = null;
            
            try {
                // Create temporary files
                File tempApk = File.createTempFile("input_", ".apk", activity.getCacheDir());
                outputApk = new File(activity.getExternalFilesDir(null), "patched_terraria_" + 
                                        loaderType.name().toLowerCase() + "_" +
                                        System.currentTimeMillis() + ".apk");
                
                // Copy input APK to temp file
                FileUtils.copyUriToFile(activity, apkUri, tempApk);
                
                // Install loader
                if (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) {
                    success = MelonLoaderManager.installMelonLoader(activity, tempApk, outputApk);
                } else if (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET35) {
                    success = MelonLoaderManager.installLemonLoader(activity, tempApk, outputApk);
                }
                
                // Cleanup temp file
                tempApk.delete();
                
            } catch (Exception e) {
                LogUtils.logDebug("Loader installation error: " + e.getMessage());
                success = false;
            }
            
            // Update UI on main thread
            final boolean finalSuccess = success;
            final File finalOutputApk = outputApk;
            
            activity.runOnUiThread(() -> {
                if (finalSuccess) {
                    showInstallationSuccessDialog(finalOutputApk, loaderType);
                } else {
                    if (callback != null) {
                        callback.onInstallationComplete(false, "Loader installation failed");
                    }
                    LogUtils.logUser("❌ " + loaderType.getDisplayName() + " installation failed");
                }
                
                updateLoaderStatus();
            });
        }).start();
    }

    public void showInstallationSuccessDialog(File outputApk, MelonLoaderManager.LoaderType loaderType) {
        AlertDialog.Builder builder = new AlertDialog.Builder(activity);
        builder.setTitle("Installation Complete");
        builder.setMessage("✅ " + loaderType.getDisplayName() + " has been installed into the APK!\n\n" +
                          "Patched APK saved to:\n" + outputApk.getName() + "\n\n" +
                          "Next steps:\n" +
                          "1. Install the patched APK\n" +
                          "2. Install DLL mods\n" +
                          "3. Launch Terraria");
        
        builder.setPositiveButton("Install APK", (dialog, which) -> {
            installApk(outputApk);
        });
        
        builder.setNegativeButton("Later", null);
        builder.show();
        
        LogUtils.logUser("✅ " + loaderType.getDisplayName() + " installation completed successfully");
        
        if (callback != null) {
            callback.onInstallationComplete(true, loaderType.getDisplayName() + " installation completed successfully");
        }
    }

    public void installApk(File apkFile) {
        try {
            Intent intent = new Intent(Intent.ACTION_VIEW);
            intent.setDataAndType(
                androidx.core.content.FileProvider.getUriForFile(
                    activity, 
                    activity.getPackageName() + ".provider", 
                    apkFile
                ),
                "application/vnd.android.package-archive"
            );
            intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
            activity.startActivity(intent);
        } catch (Exception e) {
            if (callback != null) {
                callback.onError("Cannot install APK: " + e.getMessage());
            }
            LogUtils.logDebug("APK installation error: " + e.getMessage());
        }
    }

    public void toggleMod(File mod, boolean isChecked) {
        if (isChecked) {
            ModManager.enableMod(activity, mod);
        } else {
            ModManager.disableMod(activity, mod);
        }
    }

    public void deleteMod(File mod, Runnable onSuccess) {
        AlertDialog.Builder builder = new AlertDialog.Builder(activity);
        builder.setTitle("Delete Mod");
        builder.setMessage("Delete " + mod.getName() + "?");
        builder.setPositiveButton("Delete", (dialog, which) -> {
            if (ModManager.deleteMod(activity, mod)) {
                if (onSuccess != null) {
                    onSuccess.run();
                }
                loadDllMods(); // Refresh the list
            }
        });
        builder.setNegativeButton("Cancel", null);
        builder.show();
    }

    public List<File> getDllMods() {
        return new ArrayList<>(dllMods);
    }

    public String getStatusText() {
        int enabledCount = 0;
        for (File mod : dllMods) {
            if (!mod.getName().endsWith(".disabled")) {
                enabledCount++;
            }
        }
        
        return "DLL Mods: " + enabledCount + " enabled, " + 
               (dllMods.size() - enabledCount) + " disabled, " + 
               dllMods.size() + " total";
    }

    public boolean isLoaderInstalled() {
        // FIXED: Pass activity context to MelonLoaderManager methods
        return MelonLoaderManager.isMelonLoaderInstalled(activity) || MelonLoaderManager.isLemonLoaderInstalled(activity);
    }

    public void refresh() {
        loadDllMods();
        updateLoaderStatus();
    }
}


/ModLoader/app/src/main/java/com/modloader/ui/InstructionsActivity.java

// File: InstructionsActivity.java (Fixed) - Corrected method calls with Context parameter
// Path: /storage/emulated/0/AndroidIDEProjects/TerrariaML/app/src/main/java/com/terrarialoader/ui/InstructionsActivity.java

package com.modloader.ui;

import android.content.ClipData;
import android.content.ClipboardManager;
import android.content.Context;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.view.Menu;
import android.view.MenuItem;
import android.widget.TextView;
import android.widget.Toast;
import androidx.appcompat.app.AppCompatActivity;

import com.modloader.R;
import com.modloader.loader.MelonLoaderManager;
import com.modloader.util.LogUtils;

public class InstructionsActivity extends AppCompatActivity {

    private TextView tvInstructions;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_instructions);

        setTitle("Manual Installation Guide");

        tvInstructions = findViewById(R.id.tv_instructions);
        
        setupInstructions();
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        menu.add(0, 1, 0, "Copy GitHub URLs");
        menu.add(0, 2, 0, "Open GitHub");
        menu.add(0, 3, 0, "Check Installation");
        return true;
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch (item.getItemId()) {
            case 1:
                copyGitHubUrls();
                return true;
            case 2:
                openGitHubDialog();
                return true;
            case 3:
                checkInstallationStatus();
                return true;
            default:
                return super.onOptionsItemSelected(item);
        }
    }

    private void setupInstructions() {
        // Get the actual path that the app uses
        String actualBasePath = getExternalFilesDir(null) + "/TerrariaLoader/com.and.games505.TerrariaPaid";
        
        String manualInstructions = "📱 Manual MelonLoader/LemonLoader Installation Guide\n\n" +
                
                "🔗 STEP 1: Download Required Files\n" +
                "──────────────────────────────────\n" +
                "Visit GitHub and download ONE of these:\n\n" +
                
                "🔸 For MelonLoader (Full Features):\n" +
                "• Go to: github.com/LavaGang/MelonLoader/releases\n" +
                "• Download 'melon_data.zip' from latest release\n" +
                "• File size: ~40MB\n\n" +
                
                "🔸 For LemonLoader (Lightweight):\n" +
                "• Go to: github.com/LemonLoader/LemonLoader/releases\n" +
                "• Download 'lemon_data.zip' or installer APK\n" +
                "• File size: ~15MB\n\n" +
                
                "📁 STEP 2: Create Directory Structure\n" +
                "────────────────────────────────────\n" +
                "⚠️ IMPORTANT: Use the CORRECT path for your device!\n\n" +
                
                "Using a file manager, create this structure:\n" +
                actualBasePath + "/\n\n" +
                
                "Alternative path (if first doesn't work):\n" +
                "/storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/com.and.games505.TerrariaPaid/\n\n" +
                
                "Create these folders inside the above path:\n" +
                "├── Loaders/MelonLoader/\n" +
                "│   ├── net8/                    (for MelonLoader)\n" +
                "│   ├── net35/                   (for LemonLoader)\n" +
                "│   └── Dependencies/\n" +
                "│       ├── SupportModules/\n" +
                "│       ├── CompatibilityLayers/\n" +
                "│       └── Il2CppAssemblyGenerator/\n" +
                "│           ├── Cpp2IL/cpp2il_out/\n" +
                "│           ├── UnityDependencies/\n" +
                "│           └── Il2CppInterop/Il2CppAssemblies/\n" +
                "├── Mods/\n" +
                "│   ├── DLL/                     (for your DLL mods)\n" +
                "│   └── DEX/                     (for your DEX/JAR mods)\n" +
                "├── Logs/\n" +
                "├── Config/\n" +
                "└── Backups/\n\n" +
                
                "📦 STEP 3: Extract Files\n" +
                "───────────────────────\n" +
                "Extract the downloaded ZIP file:\n\n" +
                
                "🔸 Core Files (place in Loaders/MelonLoader/net8/ or net35/):\n" +
                "• MelonLoader.dll\n" +
                "• 0Harmony.dll\n" +
                "• MonoMod.RuntimeDetour.dll\n" +
                "• MonoMod.Utils.dll\n" +
                "• Il2CppInterop.Runtime.dll (MelonLoader only)\n\n" +
                
                "🔸 Dependencies (place in Loaders/MelonLoader/Dependencies/):\n" +
                "• All remaining DLL files go in appropriate subdirectories\n" +
                "• Unity assemblies go in UnityDependencies/\n" +
                "• Il2Cpp files go in Il2CppAssemblyGenerator/\n\n" +
                
                "⚠️ IMPORTANT FILE PLACEMENT:\n" +
                "────────────────────────────\n" +
                "• MelonLoader files → Loaders/MelonLoader/net8/\n" +
                "• LemonLoader files → Loaders/MelonLoader/net35/\n" +
                "• Support modules → Loaders/MelonLoader/Dependencies/SupportModules/\n" +
                "• Your mod DLLs → Mods/DLL/\n" +
                "• Your DEX/JAR mods → Mods/DEX/\n\n" +
                
                "✅ STEP 4: Verify Installation\n" +
                "─────────────────────────────\n" +
                "Return to TerrariaLoader and:\n" +
                "1. Go to 'DLL Mod Manager'\n" +
                "2. Check loader status\n" +
                "3. Should show '✅ Loader Installed'\n" +
                "4. If not, check file paths carefully\n\n" +
                
                "🎮 STEP 5: Use Your Loader\n" +
                "─────────────────────────\n" +
                "1. Place DLL mods in Mods/DLL/ folder\n" +
                "2. Select Terraria APK in DLL Manager\n" +
                "3. Patch APK with loader\n" +
                "4. Install patched Terraria\n" +
                "5. Launch and enjoy mods!\n\n" +
                
                "🔧 TROUBLESHOOTING:\n" +
                "──────────────────\n" +
                "• Loader not detected → Check file paths exactly\n" +
                "• Can't find directory → Try both paths mentioned in Step 2\n" +
                "• APK patch fails → Verify all DLL files present\n" +
                "• Mods don't load → Check Logs/ folder for errors\n" +
                "• Slow performance → Try LemonLoader instead\n" +
                "• Permission denied → Enable 'All files access' for file manager\n\n" +
                
                "📋 REQUIRED FILES CHECKLIST:\n" +
                "───────────────────────────\n" +
                "☐ MelonLoader.dll (in Loaders/MelonLoader/net8/ or net35/)\n" +
                "☐ 0Harmony.dll (in Loaders/MelonLoader/net8/ or net35/)\n" +
                "☐ MonoMod files (in Loaders/MelonLoader/net8/ or net35/)\n" +
                "☐ Il2CppInterop files (Dependencies/SupportModules/)\n" +
                "☐ Unity dependencies (Dependencies/Il2CppAssemblyGenerator/UnityDependencies/)\n" +
                "☐ Directory structure matches exactly\n" +
                "☐ Using correct base path for your device\n\n" +
                
                "💡 TIPS:\n" +
                "──────\n" +
                "• Use a good file manager (like Solid Explorer)\n" +
                "• Enable 'Show hidden files' in your file manager\n" +
                "• Grant 'All files access' permission to your file manager\n" +
                "• Double-check spelling of folder names\n" +
                "• Keep backup of original Terraria APK\n" +
                "• Start with LemonLoader if you have storage issues\n" +
                "• Copy the exact path from Step 2 to avoid typos\n\n" +
                
                "📍 PATH VERIFICATION:\n" +
                "────────────────────\n" +
                "Your device should use:\n" + actualBasePath + "\n\n" +
                
                "If that doesn't work, try:\n" +
                "/storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/com.and.games505.TerrariaPaid/\n\n" +
                
                "Need help? Use the menu (⋮) for quick actions or check the logs in TerrariaLoader for detailed error messages!";

        tvInstructions.setText(manualInstructions);
    }

    private void copyGitHubUrls() {
        String githubUrls = "MelonLoader: https://github.com/LavaGang/MelonLoader/releases\n" +
                           "LemonLoader: https://github.com/LemonLoader/LemonLoader/releases";
        
        ClipboardManager clipboard = (ClipboardManager) getSystemService(Context.CLIPBOARD_SERVICE);
        ClipData clip = ClipData.newPlainText("GitHub URLs", githubUrls);
        clipboard.setPrimaryClip(clip);
        
        Toast.makeText(this, "GitHub URLs copied to clipboard!", Toast.LENGTH_SHORT).show();
        LogUtils.logUser("User copied GitHub URLs to clipboard");
    }

    private void openGitHubDialog() {
        android.app.AlertDialog.Builder builder = new android.app.AlertDialog.Builder(this);
        builder.setTitle("Open GitHub Repository");
        builder.setMessage("Which repository would you like to visit?");
        
        builder.setPositiveButton("MelonLoader", (dialog, which) -> {
            openUrl("https://github.com/LavaGang/MelonLoader/releases");
        });
        
        builder.setNegativeButton("LemonLoader", (dialog, which) -> {
            openUrl("https://github.com/LemonLoader/LemonLoader/releases");
        });
        
        builder.setNeutralButton("Cancel", null);
        builder.show();
    }

    private void openUrl(String url) {
        try {
            Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse(url));
            startActivity(intent);
            LogUtils.logUser("User opened GitHub URL: " + url);
        } catch (Exception e) {
            Toast.makeText(this, "Cannot open URL. Please visit manually.", Toast.LENGTH_LONG).show();
            LogUtils.logDebug("Failed to open URL: " + e.getMessage());
        }
    }

    private void checkInstallationStatus() {
        LogUtils.logUser("User checking manual installation status");
        
        // FIXED: Pass context to MelonLoaderManager methods
        boolean melonInstalled = MelonLoaderManager.isMelonLoaderInstalled(this);
        boolean lemonInstalled = MelonLoaderManager.isMelonLoaderInstalled(this);
        
        android.app.AlertDialog.Builder builder = new android.app.AlertDialog.Builder(this);
        
        if (melonInstalled || lemonInstalled) {
            String loaderType = melonInstalled ? "MelonLoader" : "LemonLoader";
            String version = MelonLoaderManager.getInstalledLoaderVersion();
            
            builder.setTitle("✅ Installation Detected!");
            builder.setMessage("Great! " + loaderType + " v" + version + " is properly installed.\n\n" +
                              "You can now:\n" +
                              "• Go to DLL Mod Manager\n" +
                              "• Install DLL mods\n" +
                              "• Patch Terraria APK\n" +
                              "• Start modding!\n\n" +
                              "Installation path:\n" + 
                              MelonLoaderManager.getStatus(this, MelonLoaderManager.TERRARIA_PACKAGE).basePath);
            
            builder.setPositiveButton("Open DLL Manager", (dialog, which) -> {
                Intent intent = new Intent(this, DllModActivity.class);
                startActivity(intent);
            });
            
            builder.setNegativeButton("Stay Here", null);
            
        } else {
            builder.setTitle("❌ Installation Not Found");
            builder.setMessage("No loader installation detected.\n\n" +
                              "Please check:\n" +
                              "• Files are in correct directories\n" +
                              "• Directory names match exactly\n" +
                              "• Core DLL files are present\n" +
                              "• File permissions are correct\n\n" +
                              "Expected path:\n" +
                              "/storage/emulated/0/TerrariaLoader/com.and.games505.TerrariaPaid/");
            
            builder.setPositiveButton("View Debug Info", (dialog, which) -> {
                showDebugInfo();
            });
            
            builder.setNegativeButton("OK", null);
        }
        
        builder.show();
    }

    private void showDebugInfo() {
        // FIXED: Pass context to MelonLoaderManager.getDebugInfo method
        String debugInfo = MelonLoaderManager.getDebugInfo(this, MelonLoaderManager.TERRARIA_PACKAGE);
        
        android.app.AlertDialog.Builder builder = new android.app.AlertDialog.Builder(this);
        builder.setTitle("Debug Information");
        builder.setMessage(debugInfo);
        builder.setPositiveButton("Copy to Clipboard", (dialog, which) -> {
            ClipboardManager clipboard = (ClipboardManager) getSystemService(Context.CLIPBOARD_SERVICE);
            ClipData clip = ClipData.newPlainText("Debug Info", debugInfo);
            clipboard.setPrimaryClip(clip);
            Toast.makeText(this, "Debug info copied to clipboard", Toast.LENGTH_SHORT).show();
        });
        builder.setNegativeButton("Close", null);
        builder.show();
    }
}


/ModLoader/app/src/main/java/com/modloader/ui/LogCategoryAdapter.java

// File: LogCategoryAdapter.java - Advanced Log Display Adapter
// Path: /main/java/com/terrarialoader/ui/LogCategoryAdapter.java

package com.modloader.ui;

import android.content.Context;
import android.graphics.Color;
import android.graphics.Typeface;
import android.text.SpannableString;
import android.text.Spanned;
import android.text.style.BackgroundColorSpan;
import android.text.style.ForegroundColorSpan;
import android.text.style.StyleSpan;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;
import androidx.annotation.NonNull;
import androidx.recyclerview.widget.RecyclerView;
import com.modloader.R;
import java.util.ArrayList;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * Advanced RecyclerView adapter for displaying categorized log entries
 * with highlighting, filtering, and rich formatting
 */
public class LogCategoryAdapter extends RecyclerView.Adapter<LogCategoryAdapter.LogViewHolder> {
    
    private final Context context;
    private List<LogEntry> logEntries = new ArrayList<>();
    private String searchQuery = "";
    private boolean highlightMode = true;
    private LogEntry.LogLevel filterLevel = null;
    private LogEntry.LogType filterType = null;
    
    // Interface for item interactions
    public interface LogItemClickListener {
        void onLogItemClick(LogEntry logEntry);
        void onLogItemLongClick(LogEntry logEntry);
    }
    
    private LogItemClickListener clickListener;
    
    public LogCategoryAdapter(Context context) {
        this.context = context;
    }
    
    public void setClickListener(LogItemClickListener listener) {
        this.clickListener = listener;
    }
    
    public void updateEntries(List<LogEntry> entries) {
        this.logEntries = entries != null ? entries : new ArrayList<>();
        notifyDataSetChanged();
    }
    
    public void setSearchQuery(String query) {
        this.searchQuery = query != null ? query : "";
        notifyDataSetChanged();
    }
    
    public void setHighlightMode(boolean enabled) {
        this.highlightMode = enabled;
        notifyDataSetChanged();
    }
    
    public void setLevelFilter(LogEntry.LogLevel level) {
        this.filterLevel = level;
        notifyDataSetChanged();
    }
    
    public void setTypeFilter(LogEntry.LogType type) {
        this.filterType = type;
        notifyDataSetChanged();
    }
    
    @NonNull
    @Override
    public LogViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(context).inflate(R.layout.item_log_entry, parent, false);
        return new LogViewHolder(view);
    }
    
    @Override
    public void onBindViewHolder(@NonNull LogViewHolder holder, int position) {
        LogEntry entry = logEntries.get(position);
        holder.bind(entry);
    }
    
    @Override
    public int getItemCount() {
        return logEntries.size();
    }
    
    public class LogViewHolder extends RecyclerView.ViewHolder {
        private final TextView timestampText;
        private final TextView levelText;
        private final TextView tagText;
        private final TextView messageText;
        private final View levelIndicator;
        private final View rootView;
        
        public LogViewHolder(@NonNull View itemView) {
            super(itemView);
            timestampText = itemView.findViewById(R.id.logTimestamp);
            levelText = itemView.findViewById(R.id.logLevel);
            tagText = itemView.findViewById(R.id.logTag);
            messageText = itemView.findViewById(R.id.logMessage);
            levelIndicator = itemView.findViewById(R.id.levelIndicator);
            rootView = itemView;
            
            // Set click listeners
            itemView.setOnClickListener(v -> {
                if (clickListener != null) {
                    int pos = getAdapterPosition();
                    if (pos != RecyclerView.NO_POSITION) {
                        clickListener.onLogItemClick(logEntries.get(pos));
                    }
                }
            });
            
            itemView.setOnLongClickListener(v -> {
                if (clickListener != null) {
                    int pos = getAdapterPosition();
                    if (pos != RecyclerView.NO_POSITION) {
                        clickListener.onLogItemLongClick(logEntries.get(pos));
                        return true;
                    }
                }
                return false;
            });
        }
        
        public void bind(LogEntry entry) {
            // Set timestamp
            timestampText.setText(entry.getFormattedTimestamp());
            
            // Set level with color
            levelText.setText(entry.getLevel().getDisplayName());
            levelText.setTextColor(Color.parseColor(entry.getLevel().getColor()));
            
            // Set tag
            tagText.setText(entry.getTag());
            
            // Set message with highlighting
            if (highlightMode && !searchQuery.trim().isEmpty()) {
                messageText.setText(highlightText(entry.getMessage(), searchQuery));
            } else {
                messageText.setText(entry.getMessage());
            }
            
            // Set level indicator color
            levelIndicator.setBackgroundColor(Color.parseColor(entry.getLevel().getColor()));
            
            // Set background based on log type and importance
            setItemBackground(entry);
            
            // Set typography based on level
            setTypography(entry);
            
            // Add type badge for system/game logs
            addTypeBadge(entry);
        }
        
        private SpannableString highlightText(String text, String query) {
            SpannableString spannableString = new SpannableString(text);
            
            if (query.trim().isEmpty()) {
                return spannableString;
            }
            
            try {
                // Try regex highlighting first
                if (isValidRegex(query)) {
                    Pattern pattern = Pattern.compile(query, Pattern.CASE_INSENSITIVE);
                    Matcher matcher = pattern.matcher(text);
                    
                    while (matcher.find()) {
                        spannableString.setSpan(
                            new BackgroundColorSpan(Color.YELLOW),
                            matcher.start(),
                            matcher.end(),
                            Spanned.SPAN_EXCLUSIVE_EXCLUSIVE
                        );
                        spannableString.setSpan(
                            new ForegroundColorSpan(Color.BLACK),
                            matcher.start(),
                            matcher.end(),
                            Spanned.SPAN_EXCLUSIVE_EXCLUSIVE
                        );
                    }
                } else {
                    // Fall back to simple text highlighting
                    String lowerText = text.toLowerCase();
                    String lowerQuery = query.toLowerCase();
                    int index = lowerText.indexOf(lowerQuery);
                    
                    while (index >= 0) {
                        spannableString.setSpan(
                            new BackgroundColorSpan(Color.YELLOW),
                            index,
                            index + query.length(),
                            Spanned.SPAN_EXCLUSIVE_EXCLUSIVE
                        );
                        spannableString.setSpan(
                            new ForegroundColorSpan(Color.BLACK),
                            index,
                            index + query.length(),
                            Spanned.SPAN_EXCLUSIVE_EXCLUSIVE
                        );
                        
                        index = lowerText.indexOf(lowerQuery, index + 1);
                    }
                }
            } catch (Exception e) {
                // If highlighting fails, return original text
                return spannableString;
            }
            
            return spannableString;
        }
        
        private boolean isValidRegex(String regex) {
            try {
                Pattern.compile(regex);
                return true;
            } catch (Exception e) {
                return false;
            }
        }
        
        private void setItemBackground(LogEntry entry) {
            int backgroundColor;
            
            switch (entry.getLevel()) {
                case ERROR:
                    backgroundColor = Color.parseColor("#FFEBEE");
                    break;
                case WARN:
                    backgroundColor = Color.parseColor("#FFF8E1");
                    break;
                case USER:
                    backgroundColor = Color.parseColor("#E8F5E8");
                    break;
                case DEBUG:
                    backgroundColor = Color.parseColor("#F5F5F5");
                    break;
                default:
                    backgroundColor = Color.WHITE;
                    break;
            }
            
            rootView.setBackgroundColor(backgroundColor);
        }
        
        private void setTypography(LogEntry entry) {
            switch (entry.getLevel()) {
                case ERROR:
                    messageText.setTypeface(null, Typeface.BOLD);
                    break;
                case WARN:
                    messageText.setTypeface(null, Typeface.ITALIC);
                    break;
                case USER:
                    messageText.setTypeface(null, Typeface.BOLD);
                    break;
                default:
                    messageText.setTypeface(null, Typeface.NORMAL);
                    break;
            }
        }
        
        private void addTypeBadge(LogEntry entry) {
            if (entry.getType() != LogEntry.LogType.APP) {
                // Add type indicator to tag
                String tagWithType = "[" + entry.getType().getDisplayName() + "] " + entry.getTag();
                tagText.setText(tagWithType);
                
                // Color code the type
                switch (entry.getType()) {
                    case GAME:
                        tagText.setTextColor(Color.parseColor("#FF9800"));
                        break;
                    case SYSTEM:
                        tagText.setTextColor(Color.parseColor("#9C27B0"));
                        break;
                    default:
                        tagText.setTextColor(Color.parseColor("#666666"));
                        break;
                }
            } else {
                tagText.setTextColor(Color.parseColor("#666666"));
            }
        }
    }
    
    // Filter methods
    public List<LogEntry> getFilteredEntries() {
        List<LogEntry> filtered = new ArrayList<>();
        
        for (LogEntry entry : logEntries) {
            if (matchesFilters(entry)) {
                filtered.add(entry);
            }
        }
        
        return filtered;
    }
    
    private boolean matchesFilters(LogEntry entry) {
        // Level filter
        if (filterLevel != null && entry.getLevel() != filterLevel) {
            return false;
        }
        
        // Type filter
        if (filterType != null && entry.getType() != filterType) {
            return false;
        }
        
        // Search query filter
        if (!searchQuery.trim().isEmpty()) {
            return entry.matchesFilter(searchQuery);
        }
        
        return true;
    }
    
    // Statistics methods
    public int getErrorCount() {
        int count = 0;
        for (LogEntry entry : logEntries) {
            if (entry.getLevel() == LogEntry.LogLevel.ERROR) {
                count++;
            }
        }
        return count;
    }
    
    public int getWarningCount() {
        int count = 0;
        for (LogEntry entry : logEntries) {
            if (entry.getLevel() == LogEntry.LogLevel.WARN) {
                count++;
            }
        }
        return count;
    }
    
    public int getImportantCount() {
        int count = 0;
        for (LogEntry entry : logEntries) {
            if (entry.isImportant()) {
                count++;
            }
        }
        return count;
    }
    
    // Export filtered entries
    public String exportFilteredEntries() {
        StringBuilder export = new StringBuilder();
        export.append("=== Filtered Log Export ===\n");
        export.append("Total entries: ").append(getItemCount()).append("\n");
        export.append("Errors: ").append(getErrorCount()).append("\n");
        export.append("Warnings: ").append(getWarningCount()).append("\n");
        export.append("Important: ").append(getImportantCount()).append("\n");
        export.append("Export time: ").append(new java.util.Date().toString()).append("\n");
        export.append("=" + "=".repeat(50)).append("\n\n");
        
        for (LogEntry entry : getFilteredEntries()) {
            export.append(entry.toFormattedString()).append("\n");
        }
        
        return export.toString();
    }
    
    // Clear all entries
    public void clear() {
        logEntries.clear();
        notifyDataSetChanged();
    }
    
    // Add single entry
    public void addEntry(LogEntry entry) {
        if (entry != null) {
            logEntries.add(entry);
            notifyItemInserted(logEntries.size() - 1);
        }
    }
    
    // Add multiple entries
    public void addEntries(List<LogEntry> entries) {
        if (entries != null && !entries.isEmpty()) {
            int startPosition = logEntries.size();
            logEntries.addAll(entries);
            notifyItemRangeInserted(startPosition, entries.size());
        }
    }
}


/ModLoader/app/src/main/java/com/modloader/ui/LogEntry.java

// File: LogEntry.java - Enhanced Log Entry Model for Advanced Logging
// Path: /main/java/com/terrarialoader/ui/LogEntry.java

package com.modloader.ui;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;

/**
 * Enhanced log entry model for advanced logging system
 * Supports categorization, filtering, and rich metadata
 */
public class LogEntry {
    
    public enum LogLevel {
        DEBUG("DEBUG", "#808080"),
        INFO("INFO", "#2196F3"),
        WARN("WARN", "#FF9800"),
        ERROR("ERROR", "#F44336"),
        USER("USER", "#4CAF50");
        
        private final String displayName;
        private final String color;
        
        LogLevel(String displayName, String color) {
            this.displayName = displayName;
            this.color = color;
        }
        
        public String getDisplayName() { return displayName; }
        public String getColor() { return color; }
    }
    
    public enum LogType {
        APP("App", "Application logs"),
        GAME("Game", "Game/MelonLoader logs"),
        SYSTEM("System", "System information");
        
        private final String displayName;
        private final String description;
        
        LogType(String displayName, String description) {
            this.displayName = displayName;
            this.description = description;
        }
        
        public String getDisplayName() { return displayName; }
        public String getDescription() { return description; }
    }
    
    private final long timestamp;
    private final LogLevel level;
    private final LogType type;
    private final String tag;
    private final String message;
    private final String threadName;
    private final String className;
    
    // Primary constructor
    public LogEntry(long timestamp, LogLevel level, LogType type, String tag, String message) {
        this.timestamp = timestamp;
        this.level = level != null ? level : LogLevel.INFO;
        this.type = type != null ? type : LogType.APP;
        this.tag = tag != null ? tag : "UNKNOWN";
        this.message = message != null ? message : "";
        this.threadName = Thread.currentThread().getName();
        this.className = getCallingClassName();
    }
    
    // Convenience constructor for app logs
    public LogEntry(LogLevel level, String tag, String message) {
        this(System.currentTimeMillis(), level, LogType.APP, tag, message);
    }
    
    // Convenience constructor for simple messages
    public LogEntry(String message) {
        this(System.currentTimeMillis(), LogLevel.INFO, LogType.APP, "APP", message);
    }
    
    // Getters
    public long getTimestamp() { return timestamp; }
    public LogLevel getLevel() { return level; }
    public LogType getType() { return type; }
    public String getTag() { return tag; }
    public String getMessage() { return message; }
    public String getThreadName() { return threadName; }
    public String getClassName() { return className; }
    
    // Formatted timestamp
    public String getFormattedTimestamp() {
        return new SimpleDateFormat("HH:mm:ss.SSS", Locale.getDefault()).format(new Date(timestamp));
    }
    
    public String getFormattedDate() {
        return new SimpleDateFormat("yyyy-MM-dd", Locale.getDefault()).format(new Date(timestamp));
    }
    
    public String getFormattedDateTime() {
        return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault()).format(new Date(timestamp));
    }
    
    // Formatted string representation
    public String toFormattedString() {
        return String.format("[%s] [%s] [%s] %s: %s", 
            getFormattedTimestamp(), 
            level.getDisplayName(), 
            type.getDisplayName(),
            tag, 
            message);
    }
    
    // Detailed string representation
    public String toDetailedString() {
        StringBuilder sb = new StringBuilder();
        sb.append("Timestamp: ").append(getFormattedDateTime()).append("\n");
        sb.append("Level: ").append(level.getDisplayName()).append("\n");
        sb.append("Type: ").append(type.getDisplayName()).append("\n");
        sb.append("Tag: ").append(tag).append("\n");
        sb.append("Thread: ").append(threadName).append("\n");
        if (className != null && !className.isEmpty()) {
            sb.append("Class: ").append(className).append("\n");
        }
        sb.append("Message: ").append(message);
        return sb.toString();
    }
    
    // JSON representation
    public String toJson() {
        return String.format(
            "{\"timestamp\":%d,\"level\": %s\",\"type\":\"%s\",\"tag\":\"%s\",\"message\":\"%s\",\"thread\":\"%s\"}",
            timestamp,
            level.getDisplayName(),
            type.getDisplayName(),
            escapeJson(tag),
            escapeJson(message),
            escapeJson(threadName)
        );
    }
    
    // Filtering helpers
    public boolean matchesFilter(String searchQuery) {
        if (searchQuery == null || searchQuery.trim().isEmpty()) {
            return true;
        }
        
        String query = searchQuery.toLowerCase();
        return message.toLowerCase().contains(query) ||
               tag.toLowerCase().contains(query) ||
               level.getDisplayName().toLowerCase().contains(query) ||
               type.getDisplayName().toLowerCase().contains(query);
    }
    
    public boolean matchesLevel(LogLevel filterLevel) {
        return filterLevel == null || this.level == filterLevel;
    }
    
    public boolean matchesType(LogType filterType) {
        return filterType == null || this.type == filterType;
    }
    
    public boolean matchesTimeRange(long startTime, long endTime) {
        return timestamp >= startTime && timestamp <= endTime;
    }
    
    // Priority for sorting (higher = more important)
    public int getPriority() {
        switch (level) {
            case ERROR: return 4;
            case WARN: return 3;
            case USER: return 2;
            case INFO: return 1;
            case DEBUG: return 0;
            default: return 0;
        }
    }
    
    // Get severity color for UI display
    public String getSeverityColor() {
        return level.getColor();
    }
    
    // Check if this is an important log entry
    public boolean isImportant() {
        return level == LogLevel.ERROR || level == LogLevel.WARN || 
               message.toLowerCase().contains("fail") ||
               message.toLowerCase().contains("error") ||
               message.toLowerCase().contains("crash");
    }
    
    // Get calling class name for debugging
    private String getCallingClassName() {
        try {
            StackTraceElement[] stackTrace = Thread.currentThread().getStackTrace();
            // Skip the first few elements (Thread.getStackTrace, this constructor, etc.)
            for (int i = 3; i < stackTrace.length && i < 8; i++) {
                String className = stackTrace[i].getClassName();
                if (!className.startsWith("java.") && 
                    !className.startsWith("android.") &&
                    !className.equals(LogEntry.class.getName())) {
                    return className.substring(className.lastIndexOf('.') + 1);
                }
            }
        } catch (Exception e) {
            // Ignore exceptions during class name detection
        }
        return null;
    }
    
    // Helper method to escape JSON strings
    private String escapeJson(String str) {
        if (str == null) return "";
        return str.replace("\\", "\\\\")
                  .replace("\"", "\\\"")
                  .replace("\n", "\\n")
                  .replace("\r", "\\r")
                  .replace("\t", "\\t");
    }
    
    // Factory methods for common log types
    public static LogEntry debug(String tag, String message) {
        return new LogEntry(LogLevel.DEBUG, tag, message);
    }
    
    public static LogEntry info(String tag, String message) {
        return new LogEntry(LogLevel.INFO, tag, message);
    }
    
    public static LogEntry warn(String tag, String message) {
        return new LogEntry(LogLevel.WARN, tag, message);
    }
    
    public static LogEntry error(String tag, String message) {
        return new LogEntry(LogLevel.ERROR, tag, message);
    }
    
    public static LogEntry user(String tag, String message) {
        return new LogEntry(LogLevel.USER, tag, message);
    }
    
    public static LogEntry system(String message) {
        return new LogEntry(System.currentTimeMillis(), LogLevel.INFO, LogType.SYSTEM, "SYSTEM", message);
    }
    
    public static LogEntry game(String tag, String message) {
        return new LogEntry(System.currentTimeMillis(), LogLevel.INFO, LogType.GAME, tag, message);
    }
    
    @Override
    public String toString() {
        return toFormattedString();
    }
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        
        LogEntry logEntry = (LogEntry) obj;
        return timestamp == logEntry.timestamp &&
               level == logEntry.level &&
               type == logEntry.type &&
               tag.equals(logEntry.tag) &&
               message.equals(logEntry.message);
    }
    
    @Override
    public int hashCode() {
        int result = Long.hashCode(timestamp);
        result = 31 * result + level.hashCode();
        result = 31 * result + type.hashCode();
        result = 31 * result + tag.hashCode();
        result = 31 * result + message.hashCode();
        return result;
    }
}
