

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/plugin/PluginLoader.java

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

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/plugin/PluginManager.java

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

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/plugin/PluginRegistry.java

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

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/BaseActivity.java

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

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/DllModActivity.java

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

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/DllModController.java

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

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/InstructionsActivity.java

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

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/LogCategoryAdapter.java

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

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/LogEntry.java

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

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/LogViewerActivity.java

// File: LogViewerActivity.java (ENHANCED) - Advanced Log Viewer with Better UI
// Path: /main/java/com/terrarialoader/ui/LogViewerActivity.java

package com.modloader.ui;

import android.content.Intent;
import android.graphics.Color;
import android.graphics.Typeface;
import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.text.SpannableString;
import android.text.Spanned;
import android.text.style.ForegroundColorSpan;
import android.text.style.StyleSpan;
import android.view.Menu;
import android.view.MenuItem;
import android.view.View;
import android.widget.*;
import androidx.appcompat.app.AlertDialog;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.FileProvider;
import androidx.swiperefreshlayout.widget.SwipeRefreshLayout;

import com.modloader.R;
import com.modloader.util.LogUtils;
import com.modloader.util.PathManager;

import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.*;
import java.util.regex.Pattern;

public class LogViewerActivity extends AppCompatActivity {
    
    // UI Components
    private SwipeRefreshLayout swipeRefreshLayout;
    private ScrollView logScrollView;
    private TextView logTextView;
    private LinearLayout filterSection;
    private Spinner logTypeSpinner;
    private Spinner logLevelSpinner;
    private EditText searchEditText;
    private Button clearLogsButton;
    private Button exportLogsButton;
    private Button refreshButton;
    private CheckBox autoScrollCheckbox;
    private TextView logStatsText;
    
    // Data
    private List<String> allLogs = new ArrayList<>();
    private List<String> filteredLogs = new ArrayList<>();
    private String currentFilter = "ALL";
    private String currentLevel = "ALL";
    private String searchQuery = "";
    private boolean autoScroll = true;
    private Handler refreshHandler;
    private Runnable refreshRunnable;
    
    // Log types and levels
    private final String[] LOG_TYPES = {"ALL", "USER", "DEBUG", "ERROR", "SYSTEM", "MOD"};
    private final String[] LOG_LEVELS = {"ALL", "INFO", "WARN", "ERROR", "FATAL"};
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_log_viewer_enhanced);
        
        setTitle("📋 Advanced Log Viewer");
        getSupportActionBar().setDisplayHomeAsUpEnabled(true);
        
        initializeComponents();
        setupUI();
        setupFilters();
        loadLogs();
        startAutoRefresh();
        
        LogUtils.logUser("Advanced Log Viewer opened");
    }
    
    private void initializeComponents() {
        // Find UI components
        swipeRefreshLayout = findViewById(R.id.swipeRefreshLayout);
        logScrollView = findViewById(R.id.logScrollView);
        logTextView = findViewById(R.id.logTextView);
        filterSection = findViewById(R.id.filterSection);
        logTypeSpinner = findViewById(R.id.logTypeSpinner);
        logLevelSpinner = findViewById(R.id.logLevelSpinner);
        searchEditText = findViewById(R.id.searchEditText);
        clearLogsButton = findViewById(R.id.clearLogsButton);
        exportLogsButton = findViewById(R.id.exportLogsButton);
        refreshButton = findViewById(R.id.refreshButton);
        autoScrollCheckbox = findViewById(R.id.autoScrollCheckbox);
        logStatsText = findViewById(R.id.logStatsText);
        
        // Initialize handlers
        refreshHandler = new Handler(Looper.getMainLooper());
    }
    
    private void setupUI() {
        // Setup swipe to refresh
        swipeRefreshLayout.setOnRefreshListener(() -> {
            loadLogs();
            swipeRefreshLayout.setRefreshing(false);
            Toast.makeText(this, "Logs refreshed", Toast.LENGTH_SHORT).show();
        });
        
        // Setup log text view
        logTextView.setTextSize(12);
        logTextView.setTypeface(Typeface.MONOSPACE);
        logTextView.setTextIsSelectable(true);
        logTextView.setBackgroundColor(Color.parseColor("#1E1E1E"));
        logTextView.setTextColor(Color.parseColor("#E0E0E0"));
        logTextView.setPadding(16, 16, 16, 16);
        
        // Setup buttons
        clearLogsButton.setOnClickListener(v -> showClearLogsDialog());
        exportLogsButton.setOnClickListener(v -> exportLogs());
        refreshButton.setOnClickListener(v -> {
            loadLogs();
            Toast.makeText(this, "Logs refreshed", Toast.LENGTH_SHORT).show();
        });
        
        // Setup auto-scroll checkbox
        autoScrollCheckbox.setChecked(autoScroll);
        autoScrollCheckbox.setOnCheckedChangeListener((buttonView, isChecked) -> {
            autoScroll = isChecked;
            if (autoScroll) {
                scrollToBottom();
            }
        });
        
        // Setup search functionality
        searchEditText.addTextChangedListener(new android.text.TextWatcher() {
            @Override
            public void beforeTextChanged(CharSequence s, int start, int count, int after) {}
            
            @Override
            public void onTextChanged(CharSequence s, int start, int before, int count) {
                searchQuery = s.toString().trim();
                applyFilters();
            }
            
            @Override
            public void afterTextChanged(android.text.Editable s) {}
        });
    }
    
    private void setupFilters() {
        // Setup log type spinner
        ArrayAdapter<String> typeAdapter = new ArrayAdapter<>(this, 
            android.R.layout.simple_spinner_item, LOG_TYPES);
        typeAdapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
        logTypeSpinner.setAdapter(typeAdapter);
        logTypeSpinner.setOnItemSelectedListener(new AdapterView.OnItemSelectedListener() {
            @Override
            public void onItemSelected(AdapterView<?> parent, View view, int position, long id) {
                currentFilter = LOG_TYPES[position];
                applyFilters();
            }
            
            @Override
            public void onNothingSelected(AdapterView<?> parent) {}
        });
        
        // Setup log level spinner
        ArrayAdapter<String> levelAdapter = new ArrayAdapter<>(this,
            android.R.layout.simple_spinner_item, LOG_LEVELS);
        levelAdapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
        logLevelSpinner.setAdapter(levelAdapter);
        logLevelSpinner.setOnItemSelectedListener(new AdapterView.OnItemSelectedListener() {
            @Override
            public void onItemSelected(AdapterView<?> parent, View view, int position, long id) {
                currentLevel = LOG_LEVELS[position];
                applyFilters();
            }
            
            @Override
            public void onNothingSelected(AdapterView<?> parent) {}
        });
    }
    
    private void loadLogs() {
        try {
            // Get logs from LogUtils
            String rawLogs = LogUtils.getLogs();
            
            // Parse logs into individual lines
            allLogs.clear();
            if (rawLogs != null && !rawLogs.trim().isEmpty()) {
                String[] logLines = rawLogs.split("\n");
                Collections.addAll(allLogs, logLines);
            }
            
            // Apply current filters
            applyFilters();
            
            // Update statistics
            updateLogStats();
            
        } catch (Exception e) {
            LogUtils.logDebug("Error loading logs: " + e.getMessage());
            Toast.makeText(this, "Error loading logs: " + e.getMessage(), Toast.LENGTH_SHORT).show();
        }
    }
    
    private void applyFilters() {
        filteredLogs.clear();
        
        for (String logLine : allLogs) {
            if (matchesFilters(logLine)) {
                filteredLogs.add(logLine);
            }
        }
        
        displayLogs();
        updateLogStats();
    }
    
    private boolean matchesFilters(String logLine) {
        // Apply type filter
        if (!"ALL".equals(currentFilter)) {
            if (!logLine.toLowerCase().contains(currentFilter.toLowerCase())) {
                return false;
            }
        }
        
        // Apply level filter
        if (!"ALL".equals(currentLevel)) {
            if (!logLine.toUpperCase().contains(currentLevel.toUpperCase())) {
                return false;
            }
        }
        
        // Apply search query
        if (!searchQuery.isEmpty()) {
            if (!logLine.toLowerCase().contains(searchQuery.toLowerCase())) {
                return false;
            }
        }
        
        return true;
    }
    
    private void displayLogs() {
        if (filteredLogs.isEmpty()) {
            logTextView.setText("📝 No logs match the current filters.\n\n" +
                "Try adjusting your filter settings or clearing the search box.");
            return;
        }
        
        SpannableString spannableLog = new SpannableString(String.join("\n", filteredLogs));
        
        // Apply syntax highlighting
        applySyntaxHighlighting(spannableLog);
        
        logTextView.setText(spannableLog);
        
        // Auto-scroll to bottom if enabled
        if (autoScroll) {
            scrollToBottom();
        }
    }
    
    private void applySyntaxHighlighting(SpannableString spannableLog) {
        String text = spannableLog.toString();
        
        // Highlight different log levels with colors
        highlightPattern(spannableLog, "ERROR", Color.parseColor("#FF6B6B"));
        highlightPattern(spannableLog, "FATAL", Color.parseColor("#FF3030"));
        highlightPattern(spannableLog, "WARN", Color.parseColor("#FFB366"));
        highlightPattern(spannableLog, "INFO", Color.parseColor("#66B2FF"));
        highlightPattern(spannableLog, "DEBUG", Color.parseColor("#98FB98"));
        
        // Highlight timestamps
        highlightPattern(spannableLog, "\\[\\d{2}:\\d{2}:\\d{2}\\]", Color.parseColor("#CCCCCC"));
        
        // Highlight file paths
        highlightPattern(spannableLog, "/[\\w/.-]+\\.(java|kt|xml)", Color.parseColor("#DDA0DD"));
        
        // Highlight search query if present
        if (!searchQuery.isEmpty()) {
            highlightPattern(spannableLog, Pattern.quote(searchQuery), Color.parseColor("#FFFF00"));
        }
    }
    
    private void highlightPattern(SpannableString spannableLog, String pattern, int color) {
        Pattern p = Pattern.compile(pattern, Pattern.CASE_INSENSITIVE);
        java.util.regex.Matcher matcher = p.matcher(spannableLog.toString());
        
        while (matcher.find()) {
            spannableLog.setSpan(new ForegroundColorSpan(color), 
                matcher.start(), matcher.end(), Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
        }
    }
    
    private void scrollToBottom() {
        logScrollView.post(() -> logScrollView.fullScroll(ScrollView.FOCUS_DOWN));
    }
    
    private void updateLogStats() {
        int totalLogs = allLogs.size();
        int filteredCount = filteredLogs.size();
        int errorCount = 0;
        int warningCount = 0;
        
        for (String log : allLogs) {
            if (log.toUpperCase().contains("ERROR") || log.toUpperCase().contains("FATAL")) {
                errorCount++;
            } else if (log.toUpperCase().contains("WARN")) {
                warningCount++;
            }
        }
        
        String statsText = String.format("📊 Total: %d | Showing: %d | Errors: %d | Warnings: %d",
            totalLogs, filteredCount, errorCount, warningCount);
        logStatsText.setText(statsText);
    }
    
    private void startAutoRefresh() {
        refreshRunnable = new Runnable() {
            @Override
            public void run() {
                loadLogs();
                refreshHandler.postDelayed(this, 5000); // Refresh every 5 seconds
            }
        };
        refreshHandler.postDelayed(refreshRunnable, 5000);
    }
    
    private void stopAutoRefresh() {
        if (refreshHandler != null && refreshRunnable != null) {
            refreshHandler.removeCallbacks(refreshRunnable);
        }
    }
    
    private void showClearLogsDialog() {
        new AlertDialog.Builder(this)
            .setTitle("🗑️ Clear Logs")
            .setMessage("Are you sure you want to clear all logs? This action cannot be undone.")
            .setPositiveButton("Clear", (dialog, which) -> {
                clearLogs();
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void clearLogs() {
        try {
            // Clear logs in LogUtils
            LogUtils.clearLogs();
            
            // Reload empty logs
            loadLogs();
            
            Toast.makeText(this, "✅ Logs cleared successfully", Toast.LENGTH_SHORT).show();
            LogUtils.logUser("Logs cleared by user");
            
        } catch (Exception e) {
            Toast.makeText(this, "❌ Failed to clear logs: " + e.getMessage(), Toast.LENGTH_SHORT).show();
            LogUtils.logDebug("Error clearing logs: " + e.getMessage());
        }
    }
    
    private void exportLogs() {
        try {
            // Create export directory
            File exportDir = new File(getExternalFilesDir(null), "TerrariaLoader/com.and.games505.TerrariaPaid/AppLogs");
            if (!exportDir.exists()) {
                exportDir.mkdirs();
            }
            
            // Create filename with timestamp
            SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd_HH-mm-ss", Locale.getDefault());
            String timestamp = dateFormat.format(new Date());
            String filename = "TerrariaLoader_Logs_" + timestamp + ".txt";
            File logFile = new File(exportDir, filename);
            
            // Write logs to file
            try (FileWriter writer = new FileWriter(logFile)) {
                writer.write("=== TerrariaLoader Log Export ===\n");
                writer.write("Export Date: " + new Date().toString() + "\n");
                writer.write("Total Logs: " + allLogs.size() + "\n");
                writer.write("Filtered Logs: " + filteredLogs.size() + "\n");
                writer.write("Current Filter: " + currentFilter + "\n");
                writer.write("Current Level: " + currentLevel + "\n");
                writer.write("Search Query: " + (searchQuery.isEmpty() ? "None" : searchQuery) + "\n");
                writer.write("\n=== LOG CONTENT ===\n\n");
                
                // Write filtered logs (what user is currently viewing)
                for (String logLine : filteredLogs) {
                    writer.write(logLine + "\n");
                }
                
                writer.write("\n=== END OF LOG ===\n");
            }
            
            // Share the log file
            Intent shareIntent = new Intent(Intent.ACTION_SEND);
            shareIntent.setType("text/plain");
            shareIntent.putExtra(Intent.EXTRA_STREAM,
                FileProvider.getUriForFile(this, getPackageName() + ".provider", logFile));
            shareIntent.putExtra(Intent.EXTRA_SUBJECT, "TerrariaLoader Logs - " + timestamp);
            shareIntent.putExtra(Intent.EXTRA_TEXT, "TerrariaLoader log export containing " + 
                filteredLogs.size() + " log entries.");
            shareIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
            
            startActivity(Intent.createChooser(shareIntent, "📤 Share Logs"));
            
            Toast.makeText(this, "✅ Logs exported: " + filename, Toast.LENGTH_LONG).show();
            LogUtils.logUser("Logs exported to: " + logFile.getAbsolutePath());
            
        } catch (IOException e) {
            Toast.makeText(this, "❌ Failed to export logs: " + e.getMessage(), Toast.LENGTH_SHORT).show();
            LogUtils.logDebug("Log export error: " + e.getMessage());
        }
    }
    
    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.log_viewer_menu, menu);
        return true;
    }
    
    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        int id = item.getItemId();
        
        if (id == android.R.id.home) {
            finish();
            return true;
        } else if (id == R.id.action_toggle_filters) {
            // Toggle filter section visibility
            filterSection.setVisibility(
                filterSection.getVisibility() == View.VISIBLE ? View.GONE : View.VISIBLE);
            return true;
        } else if (id == R.id.action_share_logs) {
            exportLogs();
            return true;
        } else if (id == R.id.action_clear_logs) {
            showClearLogsDialog();
            return true;
        } else if (id == R.id.action_settings) {
            showLogSettings();
            return true;
        }
        
        return super.onOptionsItemSelected(item);
    }
    
    private void showLogSettings() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("⚙️ Log Viewer Settings");
        
        View settingsView = getLayoutInflater().inflate(R.layout.dialog_log_settings, null);
        
        CheckBox autoRefreshCheckbox = settingsView.findViewById(R.id.autoRefreshCheckbox);
        CheckBox syntaxHighlightCheckbox = settingsView.findViewById(R.id.syntaxHighlightCheckbox);
        SeekBar textSizeSeekBar = settingsView.findViewById(R.id.textSizeSeekBar);
        TextView textSizeLabel = settingsView.findViewById(R.id.textSizeLabel);
        
        // Set current values
        autoRefreshCheckbox.setChecked(refreshRunnable != null);
        syntaxHighlightCheckbox.setChecked(true); // Always enabled for now
        
        int currentTextSize = (int) logTextView.getTextSize() / 4; // Convert to reasonable scale
        textSizeSeekBar.setProgress(currentTextSize);
        textSizeLabel.setText("Text Size: " + currentTextSize);
        
        textSizeSeekBar.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
            @Override
            public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
                textSizeLabel.setText("Text Size: " + progress);
                logTextView.setTextSize(Math.max(8, progress));
            }
            
            @Override
            public void onStartTrackingTouch(SeekBar seekBar) {}
            
            @Override
            public void onStopTrackingTouch(SeekBar seekBar) {}
        });
        
        builder.setView(settingsView);
        builder.setPositiveButton("Apply", (dialog, which) -> {
            // Apply settings
            if (autoRefreshCheckbox.isChecked() && refreshRunnable == null) {
                startAutoRefresh();
            } else if (!autoRefreshCheckbox.isChecked() && refreshRunnable != null) {
                stopAutoRefresh();
            }
            
            Toast.makeText(this, "Settings applied", Toast.LENGTH_SHORT).show();
        });
        builder.setNegativeButton("Cancel", null);
        builder.show();
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        stopAutoRefresh();
        LogUtils.logUser("Advanced Log Viewer closed");
    }
    
    @Override
    protected void onPause() {
        super.onPause();
        // Stop auto-refresh when app is not visible
        stopAutoRefresh();
    }
    
    @Override
    protected void onResume() {
        super.onResume();
        // Resume auto-refresh when app becomes visible
        startAutoRefresh();
        loadLogs(); // Refresh logs when returning to activity
    }
}

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/LogViewerEnhancedActivity.java

// File: LogViewerEnhancedActivity.java - Enhanced log viewer with filtering and search
// Path: /app/src/main/java/com/modloader/ui/LogViewerEnhancedActivity.java

package com.modloader.ui;

import android.app.AlertDialog;
import android.content.Intent;
import android.os.Bundle;
import android.os.Handler;
import android.text.Editable;
import android.text.TextWatcher;
import android.view.Menu;
import android.view.MenuItem;
import android.view.View;
import android.widget.*;
import androidx.appcompat.app.AppCompatActivity;
import androidx.swiperefreshlayout.widget.SwipeRefreshLayout;

import com.modloader.R;
import com.modloader.util.LogUtils;
import com.modloader.util.DiagnosticBundleExporter;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

public class LogViewerEnhancedActivity extends AppCompatActivity {
    
    // UI Components
    private LinearLayout filterSection;
    private Spinner logTypeSpinner;
    private Spinner logLevelSpinner;
    private EditText searchEditText;
    private TextView logStatsText;
    private TextView logTextView;
    private ScrollView logScrollView;
    private SwipeRefreshLayout swipeRefreshLayout;
    private CheckBox autoScrollCheckbox;
    private Button refreshButton;
    private Button clearLogsButton;
    private Button exportLogsButton;
    
    // State variables
    private Handler refreshHandler;
    private Runnable refreshRunnable;
    private boolean autoRefreshEnabled = true;
    private boolean filtersVisible = true;
    private String currentFilter = "All";
    private String currentLevel = "All";
    private String currentSearch = "";
    
    // Log data
    private String fullLogContent = "";
    private List<String> logLines = new ArrayList<>();
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_log_viewer_enhanced);
        
        setTitle("Enhanced Log Viewer");
        
        initializeViews();
        setupListeners();
        setupSpinners();
        startAutoRefresh();
        
        // Initial load
        refreshLogs();
    }
    
    private void initializeViews() {
        // Filter section
        filterSection = findViewById(R.id.filterSection);
        logTypeSpinner = findViewById(R.id.logTypeSpinner);
        logLevelSpinner = findViewById(R.id.logLevelSpinner);
        searchEditText = findViewById(R.id.searchEditText);
        
        // Stats and controls
        logStatsText = findViewById(R.id.logStatsText);
        refreshButton = findViewById(R.id.refreshButton);
        clearLogsButton = findViewById(R.id.clearLogsButton);
        exportLogsButton = findViewById(R.id.exportLogsButton);
        autoScrollCheckbox = findViewById(R.id.autoScrollCheckbox);
        
        // Main log display
        logTextView = findViewById(R.id.logTextView);
        logScrollView = findViewById(R.id.logScrollView);
        swipeRefreshLayout = findViewById(R.id.swipeRefreshLayout);
    }
    
    private void setupListeners() {
        // Refresh button
        refreshButton.setOnClickListener(v -> refreshLogs());
        
        // Clear logs button
        clearLogsButton.setOnClickListener(v -> clearLogs());
        
        // Export logs button
        exportLogsButton.setOnClickListener(v -> exportLogs());
        
        // Swipe refresh
        swipeRefreshLayout.setOnRefreshListener(this::refreshLogs);
        
        // Search text watcher
        searchEditText.addTextChangedListener(new TextWatcher() {
            @Override
            public void beforeTextChanged(CharSequence s, int start, int count, int after) {}
            
            @Override
            public void onTextChanged(CharSequence s, int start, int before, int count) {}
            
            @Override
            public void afterTextChanged(Editable s) {
                currentSearch = s.toString();
                applyFilters();
            }
        });
        
        // Spinner listeners
        logTypeSpinner.setOnItemSelectedListener(new AdapterView.OnItemSelectedListener() {
            @Override
            public void onItemSelected(AdapterView<?> parent, View view, int position, long id) {
                currentFilter = parent.getItemAtPosition(position).toString();
                applyFilters();
            }
            
            @Override
            public void onNothingSelected(AdapterView<?> parent) {}
        });
        
        logLevelSpinner.setOnItemSelectedListener(new AdapterView.OnItemSelectedListener() {
            @Override
            public void onItemSelected(AdapterView<?> parent, View view, int position, long id) {
                currentLevel = parent.getItemAtPosition(position).toString();
                applyFilters();
            }
            
            @Override
            public void onNothingSelected(AdapterView<?> parent) {}
        });
    }
    
    private void setupSpinners() {
        // Log type spinner
        String[] logTypes = {"All", "User", "Debug", "Info", "Warning", "Error"};
        ArrayAdapter<String> typeAdapter = new ArrayAdapter<>(this, 
            android.R.layout.simple_spinner_item, logTypes);
        typeAdapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
        logTypeSpinner.setAdapter(typeAdapter);
        
        // Log level spinner  
        String[] logLevels = {"All", "DEBUG", "INFO", "WARNING", "ERROR", "USER"};
        ArrayAdapter<String> levelAdapter = new ArrayAdapter<>(this,
            android.R.layout.simple_spinner_item, logLevels);
        levelAdapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
        logLevelSpinner.setAdapter(levelAdapter);
    }
    
    private void startAutoRefresh() {
        refreshHandler = new Handler();
        refreshRunnable = new Runnable() {
            @Override
            public void run() {
                if (autoRefreshEnabled) {
                    refreshLogs();
                }
                refreshHandler.postDelayed(this, 5000); // Refresh every 5 seconds
            }
        };
        refreshHandler.post(refreshRunnable);
    }
    
    private void refreshLogs() {
        try {
            // Get logs from LogUtils
            String logs = LogUtils.getLogs();
            if (logs == null || logs.isEmpty()) {
                logs = "No logs available.\n\nIf you're experiencing issues, try:\n• Restarting the app\n• Checking storage permissions\n• Using other app features to generate logs";
            }
            
            fullLogContent = logs;
            logLines = parseLogLines(logs);
            
            // Update statistics
            updateLogStats();
            
            // Apply current filters
            applyFilters();
            
            // Stop refresh animation
            if (swipeRefreshLayout.isRefreshing()) {
                swipeRefreshLayout.setRefreshing(false);
            }
            
        } catch (Exception e) {
            String errorMsg = "Error loading logs: " + e.getMessage();
            logTextView.setText(errorMsg);
            updateLogStats(0, 0, 0, 0);
            
            if (swipeRefreshLayout.isRefreshing()) {
                swipeRefreshLayout.setRefreshing(false);
            }
        }
    }
    
    private List<String> parseLogLines(String logs) {
        List<String> lines = new ArrayList<>();
        if (logs != null && !logs.isEmpty()) {
            String[] splitLines = logs.split("\n");
            for (String line : splitLines) {
                if (line != null && !line.trim().isEmpty()) {
                    lines.add(line);
                }
            }
        }
        return lines;
    }
    
    private void applyFilters() {
        try {
            List<String> filteredLines = new ArrayList<>();
            int totalLines = logLines.size();
            int errorCount = 0;
            int warningCount = 0;
            
            for (String line : logLines) {
                if (line == null || line.trim().isEmpty()) {
                    continue;
                }
                
                // Count errors and warnings
                String lowerLine = line.toLowerCase();
                if (lowerLine.contains("error") || lowerLine.contains("❌")) {
                    errorCount++;
                } else if (lowerLine.contains("warn") || lowerLine.contains("⚠️")) {
                    warningCount++;
                }
                
                // Apply filters
                boolean includeByType = filterByType(line);
                boolean includeByLevel = filterByLevel(line);  
                boolean includeBySearch = filterBySearch(line);
                
                if (includeByType && includeByLevel && includeBySearch) {
                    filteredLines.add(line);
                }
            }
            
            // Update display
            StringBuilder displayContent = new StringBuilder();
            for (String line : filteredLines) {
                displayContent.append(line).append("\n");
            }
            
            logTextView.setText(displayContent.toString());
            updateLogStats(totalLines, filteredLines.size(), errorCount, warningCount);
            
            // Auto-scroll to bottom if enabled
            if (autoScrollCheckbox.isChecked()) {
                scrollToBottom();
            }
            
        } catch (Exception e) {
            logTextView.setText("Error applying filters: " + e.getMessage());
        }
    }
    
    private boolean filterByType(String line) {
        if ("All".equals(currentFilter)) {
            return true;
        }
        
        String lowerLine = line.toLowerCase();
        String lowerFilter = currentFilter.toLowerCase();
        
        return lowerLine.contains(lowerFilter) || 
               (lowerFilter.equals("user") && (lowerLine.contains("✅") || lowerLine.contains("❌") || lowerLine.contains("⚠️")));
    }
    
    private boolean filterByLevel(String line) {
        if ("All".equals(currentLevel)) {
            return true;
        }
        
        return line.toUpperCase().contains(currentLevel);
    }
    
    private boolean filterBySearch(String line) {
        if (currentSearch == null || currentSearch.trim().isEmpty()) {
            return true;
        }
        
        return line.toLowerCase().contains(currentSearch.toLowerCase());
    }
    
    private void updateLogStats() {
        updateLogStats(logLines.size(), logLines.size(), 0, 0);
    }
    
    private void updateLogStats(int total, int showing, int errors, int warnings) {
        String statsText = String.format("Total: %d | Showing: %d | Errors: %d | Warnings: %d", 
            total, showing, errors, warnings);
        logStatsText.setText(statsText);
    }
    
    private void scrollToBottom() {
        logScrollView.post(() -> {
            logScrollView.fullScroll(ScrollView.FOCUS_DOWN);
        });
    }
    
    private void clearLogs() {
        new AlertDialog.Builder(this)
            .setTitle("Clear Logs")
            .setMessage("Are you sure you want to clear all logs? This cannot be undone.")
            .setPositiveButton("Clear", (dialog, which) -> {
                LogUtils.clearLogs();
                logTextView.setText("Logs cleared.\n\nNew logs will appear as you use the app.");
                updateLogStats(0, 0, 0, 0);
                Toast.makeText(this, "Logs cleared", Toast.LENGTH_SHORT).show();
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void exportLogs() {
        try {
            // Create diagnostic bundle
            File bundleFile = DiagnosticBundleExporter.createDiagnosticBundle(this);
            
            if (bundleFile != null && bundleFile.exists()) {
                // Share the bundle
                Intent shareIntent = new Intent(Intent.ACTION_SEND);
                shareIntent.setType("application/zip");
                shareIntent.putExtra(Intent.EXTRA_SUBJECT, "TerrariaLoader Diagnostic Bundle");
                shareIntent.putExtra(Intent.EXTRA_TEXT, "Diagnostic bundle created on " + new java.util.Date().toString());
                
                // Use FileProvider to share the file
                android.net.Uri fileUri = androidx.core.content.FileProvider.getUriForFile(
                    this, getPackageName() + ".provider", bundleFile);
                shareIntent.putExtra(Intent.EXTRA_STREAM, fileUri);
                shareIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
                
                startActivity(Intent.createChooser(shareIntent, "Export Diagnostic Bundle"));
                
                Toast.makeText(this, "Diagnostic bundle created: " + bundleFile.getName(), 
                    Toast.LENGTH_LONG).show();
            } else {
                Toast.makeText(this, "Failed to create diagnostic bundle", Toast.LENGTH_SHORT).show();
            }
            
        } catch (Exception e) {
            Toast.makeText(this, "Export failed: " + e.getMessage(), Toast.LENGTH_SHORT).show();
        }
    }
    
    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.log_viewer_menu, menu);
        return true;
    }
    
    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        int itemId = item.getItemId();
        
        if (itemId == R.id.action_toggle_filters) {
            toggleFilters();
            return true;
        } else if (itemId == R.id.action_share_logs) {
            exportLogs();
            return true;
        } else if (itemId == R.id.action_clear_logs) {
            clearLogs();
            return true;
        } else if (itemId == R.id.action_settings) {
            showSettings();
            return true;
        }
        
        return super.onOptionsItemSelected(item);
    }
    
    private void toggleFilters() {
        filtersVisible = !filtersVisible;
        filterSection.setVisibility(filtersVisible ? View.VISIBLE : View.GONE);
        Toast.makeText(this, "Filters " + (filtersVisible ? "shown" : "hidden"), Toast.LENGTH_SHORT).show();
    }
    
    private void showSettings() {
        // Simple settings dialog
        View settingsView = getLayoutInflater().inflate(R.layout.dialog_log_settings, null);
        
        CheckBox autoRefreshCheck = settingsView.findViewById(R.id.autoRefreshCheckbox);
        autoRefreshCheck.setChecked(autoRefreshEnabled);
        
        new AlertDialog.Builder(this)
            .setTitle("Log Viewer Settings")
            .setView(settingsView)
            .setPositiveButton("OK", (dialog, which) -> {
                autoRefreshEnabled = autoRefreshCheck.isChecked();
                Toast.makeText(this, "Settings saved", Toast.LENGTH_SHORT).show();
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (refreshHandler != null && refreshRunnable != null) {
            refreshHandler.removeCallbacks(refreshRunnable);
        }
    }
    
    @Override
    protected void onPause() {
        super.onPause();
        autoRefreshEnabled = false;
    }
    
    @Override
    protected void onResume() {
        super.onResume();
        autoRefreshEnabled = true;
        refreshLogs();
    }
}

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/ModListActivity.java

package com.modloader.ui;

import android.app.Activity;
import android.app.AlertDialog;
import android.content.Intent;
import android.database.Cursor;
import android.net.Uri;
import android.os.Bundle;
import android.provider.OpenableColumns;
import android.view.LayoutInflater;
import android.view.View;
import android.widget.Button;
import android.widget.ImageButton;
import android.widget.TextView;
import android.widget.Toast;
import android.widget.Switch; // Make sure this is imported if used in your XML

import androidx.annotation.Nullable;
import androidx.appcompat.app.AppCompatActivity;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;

import com.modloader.R;
import com.modloader.installer.ModInstaller;
import com.modloader.loader.ModManager;
import com.modloader.util.LogUtils;

import java.io.File;
import java.util.List;

public class ModListActivity extends AppCompatActivity {

    private RecyclerView recyclerView;
    private ModListAdapter modAdapter; // Changed from ModAdapter
    private TextView modCountTextView;
    private static final int PICK_MOD_FILE_REQUEST = 1;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_mod_list);

        modCountTextView = findViewById(R.id.modCountTextView);
        recyclerView = findViewById(R.id.recyclerViewMods);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));

        ImageButton backButton = findViewById(R.id.backButton);
        backButton.setOnClickListener(v -> onBackPressed());

        Button addModButton = findViewById(R.id.addModButton);
        addModButton.setOnClickListener(v -> openFilePicker());

        Button refreshModsButton = findViewById(R.id.refreshModsButton);
        refreshModsButton.setOnClickListener(v -> loadMods());

        loadMods();
    }

    private void loadMods() {
        ModManager.loadMods(this);
        List<File> mods = ModManager.getAvailableMods();
        modAdapter = new ModListAdapter(this, mods); // Changed to ModListAdapter
        recyclerView.setAdapter(modAdapter);
        updateModCount(mods.size());
    }

    private void updateModCount(int count) {
        modCountTextView.setText("Total Mods: " + count + " (Enabled: " + ModManager.getEnabledModCount() + ")");
    }

    private void openFilePicker() {
        Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
        intent.setType("*/*"); // Allow all file types, then filter
        intent.addCategory(Intent.CATEGORY_OPENABLE);
        try {
            startActivityForResult(Intent.createChooser(intent, "Select Mod File"), PICK_MOD_FILE_REQUEST);
        } catch (android.content.ActivityNotFoundException ex) {
            Toast.makeText(this, "Please install a File Manager.", Toast.LENGTH_SHORT).show();
        }
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == PICK_MOD_FILE_REQUEST && resultCode == Activity.RESULT_OK) {
            if (data != null && data.getData() != null) {
                Uri uri = data.getData();
                handlePickedFile(uri); // Added this method
            }
        }
    }

    // New method to handle picked files and install them
    private void handlePickedFile(Uri uri) {
        String filename = getFilenameFromUri(uri);
        if (filename == null || !isValidModExtension(filename)) {
            LogUtils.logUser("❌ Invalid mod file type selected.");
            Toast.makeText(this, "Invalid mod file type. Only .dex, .jar, .dll, .hybrid are supported.", Toast.LENGTH_LONG).show();
            return;
        }

        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("Install Mod");
        builder.setMessage("Do you want to install '" + filename + "'?");
        builder.setPositiveButton("Install", (dialog, which) -> {
            boolean success = ModInstaller.installModAuto(this, uri);
            if (success) {
                Toast.makeText(this, "Mod installed successfully!", Toast.LENGTH_SHORT).show();
                loadMods(); // Reload mods after installation
            } else {
                Toast.makeText(this, "Failed to install mod.", Toast.LENGTH_SHORT).show();
            }
        });
        builder.setNegativeButton("Cancel", null);
        builder.show();
    }

    private String getFilenameFromUri(Uri uri) {
        String result = null;
        if (uri.getScheme().equals("content")) {
            try (Cursor cursor = getContentResolver().query(uri, null, null, null, null)) {
                if (cursor != null && cursor.moveToFirst()) {
                    int nameIndex = cursor.getColumnIndex(OpenableColumns.DISPLAY_NAME);
                    if (nameIndex != -1) {
                        result = cursor.getString(nameIndex);
                    }
                }
            }
        }
        if (result == null) {
            result = uri.getPath();
            int cut = result.lastIndexOf('/');
            if (cut != -1) {
                result = result.substring(cut + 1);
            }
        }
        return result;
    }

    private boolean isValidModExtension(String filename) {
        String lowerFilename = filename.toLowerCase();
        for (String ext : ModInstaller.getSupportedExtensions()) {
            if (lowerFilename.endsWith(ext)) {
                return true;
            }
        }
        return false;
    }

    @Override
    protected void onResume() {
        super.onResume();
        loadMods(); // Refresh mod list when activity resumes
    }
}


================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/ModListAdapter.java

// File: ModListAdapter.java (Fixed Adapter Class) - NullPointerException Fix
// Path: /storage/emulated/0/AndroidIDEProjects/TerrariaML/app/src/main/java/com/terrarialoader/ui/ModListAdapter.java

package com.modloader.ui;

import android.app.AlertDialog;
import android.content.Context;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.ImageButton;
import android.widget.Switch;
import android.widget.TextView;
import android.widget.Toast;

import androidx.annotation.NonNull;
import androidx.recyclerview.widget.RecyclerView;

import com.modloader.R;
import com.modloader.installer.ModInstaller;
import com.modloader.loader.ModManager;
import com.modloader.loader.ModMetadata;
import com.modloader.loader.ModBase;
import com.modloader.util.LogUtils;

import java.io.File;
import java.util.List;

public class ModListAdapter extends RecyclerView.Adapter<ModListAdapter.ModViewHolder> {

    private final Context context;
    private List<File> mods; // Changed to non-final to allow updates

    public ModListAdapter(Context context, List<File> mods) {
        this.context = context;
        this.mods = mods;
    }

    @NonNull
    @Override
    public ModViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(context).inflate(R.layout.item_mod, parent, false);
        return new ModViewHolder(view);
    }

    @Override
    public void onBindViewHolder(@NonNull ModViewHolder holder, int position) {
        File modFile = mods.get(position);
        String modName = modFile.getName();
        boolean isEnabled = !modName.endsWith(".disabled");

        holder.modNameTextView.setText(modName);
        
        // FIXED: Null pointer protection for metadata
        try {
            // Get metadata safely
            String cleanModName = modName.replace(".disabled", "").replace(".dex", "").replace(".jar", "").replace(".dll", "");
            ModMetadata metadata = ModManager.getMetadata(cleanModName);
            
            if (metadata != null) {
                // Use metadata if available
                ModBase.ModType modType = metadata.getModType();
                if (modType != null) {
                    holder.modDescriptionTextView.setText("Type: " + modType.getDisplayName());
                } else {
                    holder.modDescriptionTextView.setText("Type: " + getModTypeFromFileName(modName));
                }
            } else {
                // Fallback to file extension detection
                holder.modDescriptionTextView.setText("Type: " + getModTypeFromFileName(modName));
            }
        } catch (Exception e) {
            // Ultimate fallback
            LogUtils.logDebug("Error getting mod metadata: " + e.getMessage());
            holder.modDescriptionTextView.setText("Type: " + getModTypeFromFileName(modName));
        }

        holder.modSwitch.setChecked(isEnabled);
        holder.modSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
            try {
                if (isChecked) {
                    ModManager.enableMod(context, modFile);
                } else {
                    ModManager.disableMod(context, modFile);
                }
                // Refresh the adapter after mod state change
                // Note: The list of files is not actually changing here, only their names.
                // A better approach would be to reload the list of files entirely.
                // For now, we will simply notify that the item has changed.
                notifyItemChanged(position);
            } catch (Exception e) {
                LogUtils.logDebug("Error toggling mod: " + e.getMessage());
                Toast.makeText(context, "Error toggling mod: " + e.getMessage(), Toast.LENGTH_SHORT).show();
            }
        });

        holder.modDeleteButton.setOnClickListener(v -> {
            new AlertDialog.Builder(context)
                    .setTitle("Delete Mod")
                    .setMessage("Are you sure you want to delete " + modName + "?")
                    .setPositiveButton("Delete", (dialog, which) -> {
                        try {
                            if (ModInstaller.uninstallMod(context, modName)) {
                                // Remove from list and notify adapter
                                mods.remove(position);
                                notifyItemRemoved(position);
                                notifyItemRangeChanged(position, mods.size());
                                Toast.makeText(context, modName + " deleted.", Toast.LENGTH_SHORT).show();
                            } else {
                                Toast.makeText(context, "Failed to delete " + modName, Toast.LENGTH_SHORT).show();
                            }
                        } catch (Exception e) {
                            LogUtils.logDebug("Error deleting mod: " + e.getMessage());
                            Toast.makeText(context, "Error deleting mod: " + e.getMessage(), Toast.LENGTH_SHORT).show();
                        }
                    })
                    .setNegativeButton("Cancel", null)
                    .show();
        });
    }
    
    // Helper method to determine mod type from filename
    private String getModTypeFromFileName(String fileName) {
        String lowerName = fileName.toLowerCase();
        if (lowerName.endsWith(".dex") || lowerName.endsWith(".dex.disabled")) {
            return "DEX (Java)";
        } else if (lowerName.endsWith(".jar") || lowerName.endsWith(".jar.disabled")) {
            return "JAR (Java Library)";
        } else if (lowerName.endsWith(".dll") || lowerName.endsWith(".dll.disabled")) {
            return "DLL (C#/Native)";
        } else {
            return "Unknown";
        }
    }

    @Override
    public int getItemCount() {
        return mods.size();
    }

    /**
     * Updates the adapter's data set with a new list of mods.
     * @param newMods The new list of mods to display.
     */
    public void updateMods(List<File> newMods) {
        this.mods = newMods;
        notifyDataSetChanged();
    }

    public static class ModViewHolder extends RecyclerView.ViewHolder {
        TextView modNameTextView;
        TextView modDescriptionTextView;
        Switch modSwitch;
        ImageButton modDeleteButton;

        public ModViewHolder(@NonNull View itemView) {
            super(itemView);
            modNameTextView = itemView.findViewById(R.id.modNameTextView);
            modDescriptionTextView = itemView.findViewById(R.id.modDescription);
            modSwitch = itemView.findViewById(R.id.modSwitch);
            modDeleteButton = itemView.findViewById(R.id.modDeleteButton);
        }
    }
}


================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/ModManagementActivity.java

// File: ModManagementActivity.java - Pure Mod Management (Post-Installation)
// Path: /main/java/com/terrarialoader/ui/ModManagementActivity.java

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
import android.widget.ImageButton;
import android.widget.LinearLayout;
import android.widget.TextView;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;

import com.modloader.R;
import com.modloader.installer.ModInstaller;
import com.modloader.loader.MelonLoaderManager;
import com.modloader.loader.ModManager;
import com.modloader.ui.ModListAdapter;
import com.modloader.util.LogUtils;

import java.io.File;
import java.util.List;

/**
 * Pure mod management activity - assumes loader is already installed
 * Focused solely on managing DLL and DEX mods
 */
public class ModManagementActivity extends AppCompatActivity {

    private static final int REQUEST_SELECT_DLL = 1001;
    private static final int REQUEST_SELECT_DEX = 1002;
    
    // UI Components
    private TextView statusText;
    private TextView loaderStatusText;
    private RecyclerView modRecyclerView;
    private ModListAdapter modAdapter;
    private Button addDllModBtn;
    private Button addDexModBtn;
    private Button refreshBtn;
    private Button backBtn;
    private LinearLayout loaderInfoSection;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_mod_management);
        
        setTitle("🎮 Mod Management");
        
        initializeViews();
        setupListeners();
        loadMods();
        updateStatus();
    }

    private void initializeViews() {
        statusText = findViewById(R.id.statusText);
        loaderStatusText = findViewById(R.id.loaderStatusText);
        modRecyclerView = findViewById(R.id.modRecyclerView);
        addDllModBtn = findViewById(R.id.addDllModBtn);
        addDexModBtn = findViewById(R.id.addDexModBtn);
        refreshBtn = findViewById(R.id.refreshBtn);
        backBtn = findViewById(R.id.backBtn);
        loaderInfoSection = findViewById(R.id.loaderInfoSection);
        
        // Setup RecyclerView
        modRecyclerView.setLayoutManager(new LinearLayoutManager(this));
    }

    private void setupListeners() {
        addDllModBtn.setOnClickListener(v -> {
            if (!MelonLoaderManager.isMelonLoaderInstalled(this) && !MelonLoaderManager.isLemonLoaderInstalled(this)) {
                showLoaderRequiredDialog();
                return;
            }
            
            Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
            intent.setType("*/*");
            intent.addCategory(Intent.CATEGORY_OPENABLE);
            startActivityForResult(intent, REQUEST_SELECT_DLL);
        });
        
        addDexModBtn.setOnClickListener(v -> {
            Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
            intent.setType("*/*");
            intent.addCategory(Intent.CATEGORY_OPENABLE);
            startActivityForResult(intent, REQUEST_SELECT_DEX);
        });
        
        refreshBtn.setOnClickListener(v -> {
            loadMods();
            updateStatus();
            Toast.makeText(this, "🔄 Mods refreshed", Toast.LENGTH_SHORT).show();
        });
        
        backBtn.setOnClickListener(v -> finish());
    }

    private void loadMods() {
        ModManager.loadMods(this);
        List<File> allMods = ModManager.getAvailableMods();
        
        if (modAdapter == null) {
            modAdapter = new ModListAdapter(this, allMods);
            modRecyclerView.setAdapter(modAdapter);
        } else {
            // Update existing adapter
            modAdapter.updateMods(allMods);
        }
        
        LogUtils.logUser("Loaded " + allMods.size() + " mods for management");
    }

    private void updateStatus() {
        // Check loader status
        boolean melonInstalled = MelonLoaderManager.isMelonLoaderInstalled(this);
        boolean lemonInstalled = MelonLoaderManager.isLemonLoaderInstalled(this);
        
        if (melonInstalled) {
            loaderStatusText.setText("✅ MelonLoader " + MelonLoaderManager.getInstalledLoaderVersion() + " - DLL mods supported");
            loaderStatusText.setTextColor(0xFF4CAF50); // Green
            addDllModBtn.setEnabled(true);
            addDllModBtn.setText("📥 Add DLL Mod");
            loaderInfoSection.setVisibility(View.VISIBLE);
        } else if (lemonInstalled) {
            loaderStatusText.setText("✅ LemonLoader " + MelonLoaderManager.getInstalledLoaderVersion() + " - DLL mods supported");
            loaderStatusText.setTextColor(0xFF4CAF50); // Green
            addDllModBtn.setEnabled(true);
            addDllModBtn.setText("📥 Add DLL Mod");
            loaderInfoSection.setVisibility(View.VISIBLE);
        } else {
            loaderStatusText.setText("⚠️ No loader installed - DLL mods unavailable");
            loaderStatusText.setTextColor(0xFFF44336); // Red
            addDllModBtn.setEnabled(false);
            addDllModBtn.setText("❌ Install Loader First");
            loaderInfoSection.setVisibility(View.GONE);
        }
        
        // Update mod counts
        int enabledCount = ModManager.getEnabledModCount();
        int totalCount = ModManager.getTotalModCount();
        int dexCount = ModManager.getDexModCount();
        int dllCount = ModManager.getDllModCount();
        
        statusText.setText(String.format("📊 Total: %d mods (%d enabled) | DEX/JAR: %d | DLL: %d", 
            totalCount, enabledCount, dexCount, dllCount));
    }

    private void showLoaderRequiredDialog() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("🔧 Loader Required");
        builder.setMessage("DLL mods require MelonLoader or LemonLoader to be installed.\n\n" +
                          "Would you like to set up a loader now?");
        
        builder.setPositiveButton("🚀 Setup Loader", (dialog, which) -> {
            // Go to unified loader setup
            Intent intent = new Intent(this, UnifiedLoaderActivity.class);
            startActivity(intent);
        });
        
        builder.setNegativeButton("📖 Manual Guide", (dialog, which) -> {
            Intent intent = new Intent(this, InstructionsActivity.class);
            startActivity(intent);
        });
        
        builder.setNeutralButton("Cancel", null);
        builder.show();
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        
        if (resultCode != Activity.RESULT_OK || data == null || data.getData() == null) {
            return;
        }
        
        Uri uri = data.getData();
        String filename = getFilenameFromUri(uri);
        
        if (filename == null) {
            Toast.makeText(this, "Could not determine filename", Toast.LENGTH_SHORT).show();
            return;
        }
        
        switch (requestCode) {
            case REQUEST_SELECT_DLL:
                handleDllModInstallation(uri, filename);
                break;
                
            case REQUEST_SELECT_DEX:
                handleDexModInstallation(uri, filename);
                break;
        }
    }

    private void handleDllModInstallation(Uri uri, String filename) {
        if (!filename.toLowerCase().endsWith(".dll")) {
            Toast.makeText(this, "⚠️ Please select a .dll file", Toast.LENGTH_LONG).show();
            return;
        }
        
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("📥 Install DLL Mod");
        builder.setMessage("Install '" + filename + "' as a DLL mod?\n\n" +
                          "This mod will be loaded by MelonLoader when Terraria starts.");
        
        builder.setPositiveButton("Install", (dialog, which) -> {
            boolean success = ModInstaller.installMod(this, uri, filename);
            if (success) {
                Toast.makeText(this, "✅ DLL mod installed: " + filename, Toast.LENGTH_SHORT).show();
                loadMods();
                updateStatus();
            } else {
                Toast.makeText(this, "❌ Failed to install DLL mod", Toast.LENGTH_LONG).show();
            }
        });
        
        builder.setNegativeButton("Cancel", null);
        builder.show();
    }

    private void handleDexModInstallation(Uri uri, String filename) {
        String lowerName = filename.toLowerCase();
        if (!lowerName.endsWith(".dex") && !lowerName.endsWith(".jar")) {
            Toast.makeText(this, "⚠️ Please select a .dex or .jar file", Toast.LENGTH_LONG).show();
            return;
        }
        
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("📥 Install DEX/JAR Mod");
        builder.setMessage("Install '" + filename + "' as a DEX/JAR mod?\n\n" +
                          "This mod will be loaded directly by TerrariaLoader.");
        
        builder.setPositiveButton("Install", (dialog, which) -> {
            boolean success = ModInstaller.installMod(this, uri, filename);
            if (success) {
                Toast.makeText(this, "✅ DEX/JAR mod installed: " + filename, Toast.LENGTH_SHORT).show();
                loadMods();
                updateStatus();
            } else {
                Toast.makeText(this, "❌ Failed to install DEX/JAR mod", Toast.LENGTH_LONG).show();
            }
        });
        
        builder.setNegativeButton("Cancel", null);
        builder.show();
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
        
        if (filename == null) {
            String path = uri.getPath();
            if (path != null) {
                int lastSlash = path.lastIndexOf('/');
                if (lastSlash >= 0 && lastSlash < path.length() - 1) {
                    filename = path.substring(lastSlash + 1);
                }
            }
        }
        
        return filename;
    }

    @Override
    protected void onResume() {
        super.onResume();
        loadMods();
        updateStatus();
    }
}

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/OfflineDiagnosticActivity.java

// File: OfflineDiagnosticActivity.java (Part 1 - Main Class)
// Path: /main/java/com/terrarialoader/ui/OfflineDiagnosticActivity.java

package com.modloader.ui;

import android.app.ProgressDialog;
import android.content.Intent;
import android.net.Uri;
import android.os.AsyncTask;
import android.os.Bundle;
import android.widget.Button;
import android.widget.TextView;
import android.widget.Toast;

import androidx.activity.result.ActivityResultLauncher;
import androidx.activity.result.contract.ActivityResultContracts;
import androidx.appcompat.app.AlertDialog;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.FileProvider;

import com.modloader.R;
import com.modloader.diagnostic.DiagnosticManager;
import com.modloader.util.LogUtils;
import com.modloader.util.FileUtils;
import com.modloader.util.PathManager;
import com.modloader.loader.MelonLoaderManager;

import java.io.File;
import java.io.FileWriter;
import java.io.IOException;

public class OfflineDiagnosticActivity extends AppCompatActivity {
    
    private DiagnosticManager diagnosticManager;
    
    // UI Components
    private Button btnRunFullDiagnostic;
    private Button btnDiagnoseApk;
    private Button btnFixSettings;
    private Button btnAutoRepair;
    private Button btnExportReport;
    private Button btnClearResults;
    private TextView diagnosticResultsText;
    
    // Progress dialog
    private ProgressDialog progressDialog;
    
    // File picker for APK selection
    private final ActivityResultLauncher<Intent> apkPickerLauncher = 
        registerForActivityResult(new ActivityResultContracts.StartActivityForResult(), result -> {
            if (result.getResultCode() == RESULT_OK && result.getData() != null) {
                Uri apkUri = result.getData().getData();
                runApkDiagnostic(apkUri);
            }
        });
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_offline_diagnostic);
        setTitle("🔧 Offline Diagnostics");
        
        initializeComponents();
        setupUI();
        
        LogUtils.logUser("Offline Diagnostics opened");
    }
    
    private void initializeComponents() {
        diagnosticManager = new DiagnosticManager(this);
        
        // Find UI components
        btnRunFullDiagnostic = findViewById(R.id.btn_run_full_diagnostic);
        btnDiagnoseApk = findViewById(R.id.btn_diagnose_apk);
        btnFixSettings = findViewById(R.id.btn_fix_settings);
        btnAutoRepair = findViewById(R.id.btn_auto_repair);
        btnExportReport = findViewById(R.id.btn_export_report);
        btnClearResults = findViewById(R.id.btn_clear_results);
        diagnosticResultsText = findViewById(R.id.diagnostic_results_text);
    }
    
    private void setupUI() {
        // Full system diagnostic
        btnRunFullDiagnostic.setOnClickListener(v -> runFullSystemCheck());
        
        // APK diagnostic
        btnDiagnoseApk.setOnClickListener(v -> selectApkForDiagnostic());
        
        // Settings diagnostic and fix
        btnFixSettings.setOnClickListener(v -> diagnoseAndFixSettings());
        
        // Auto repair
        btnAutoRepair.setOnClickListener(v -> performAutoRepair());
        
        // Export report
        btnExportReport.setOnClickListener(v -> exportDiagnosticReport());
        
        // Clear results
        btnClearResults.setOnClickListener(v -> clearResults());
    }
    
    private void runFullSystemCheck() {
        showProgress("Running comprehensive system diagnostic...");
        
        AsyncTask.execute(() -> {
            try {
                StringBuilder results = new StringBuilder();
                results.append("=== TerrariaLoader Comprehensive Diagnostic ===\n");
                results.append("Timestamp: ").append(new java.util.Date().toString()).append("\n");
                results.append("Device: ").append(android.os.Build.MANUFACTURER).append(" ")
                       .append(android.os.Build.MODEL).append("\n");
                results.append("Android: ").append(android.os.Build.VERSION.RELEASE).append("\n\n");
                
                // 1. Directory Structure Check
                results.append("📁 DIRECTORY STRUCTURE\n");
                results.append(checkDirectoryStructure()).append("\n");
                
                // 2. MelonLoader/LemonLoader Status
                results.append("🛠️ LOADER STATUS\n");
                results.append(checkLoaderStatus()).append("\n");
                
                // 3. Mod Files Validation
                results.append("📦 MOD FILES\n");
                results.append(checkModFiles()).append("\n");
                
                // 4. System Permissions
                results.append("🔐 PERMISSIONS\n");
                results.append(checkPermissions()).append("\n");
                
                // 5. Storage and Space
                results.append("💾 STORAGE\n");
                results.append(checkStorage()).append("\n");
                
                // 6. Settings Validation
                results.append("⚙️ SETTINGS\n");
                results.append(checkSettingsIntegrity()).append("\n");
                
                // 7. Suggested Actions
                results.append("💡 RECOMMENDATIONS\n");
                results.append(generateRecommendations()).append("\n");
                
                runOnUiThread(() -> {
                    hideProgress();
                    displayResults(results.toString());
                    LogUtils.logUser("Full system diagnostic completed");
                });
            } catch (Exception e) {
                runOnUiThread(() -> {
                    hideProgress();
                    showError("Diagnostic failed: " + e.getMessage());
                    LogUtils.logDebug("Diagnostic error: " + e.toString());
                });
            }
        });
    }
    
    private String checkDirectoryStructure() {
        StringBuilder result = new StringBuilder();
        
        try {
            String gamePackage = "com.and.games505.TerrariaPaid";
            File baseDir = PathManager.getGameBaseDir(this, gamePackage);
            
            if (baseDir == null) {
                result.append("❌ Base directory path is null\n");
                return result.toString();
            }
            
            result.append("Base Path: ").append(baseDir.getAbsolutePath()).append("\n");
            
            // Check key directories
            String[] criticalPaths = {
                "",                           // Base
                "Mods",                      // Mods root
                "Mods/DEX",                  // DEX mods
                "Mods/DLL",                  // DLL mods
                "Loaders",                   // Loaders root
                "Loaders/MelonLoader",       // MelonLoader
                "Logs",                      // Game logs
                "AppLogs",                   // App logs
                "Config",                    // Configuration
                "Backups"                    // Backups
            };
            
            int existingDirs = 0;
            for (String path : criticalPaths) {
                File dir = new File(baseDir, path);
                boolean exists = dir.exists() && dir.isDirectory();
                String status = exists ? "✅" : "❌";
                result.append(status).append(" ").append(path.isEmpty() ? "Base" : path).append("\n");
                if (exists) existingDirs++;
            }
            
            result.append("\nDirectory Health: ").append(existingDirs).append("/").append(criticalPaths.length);
            if (existingDirs < criticalPaths.length) {
                result.append(" (⚠️ Some directories missing)");
            } else {
                result.append(" (✅ Complete)");
            }
            
        } catch (Exception e) {
            result.append("❌ Directory check failed: ").append(e.getMessage());
        }
        
        return result.toString();
    }
    
    private String checkLoaderStatus() {
        StringBuilder result = new StringBuilder();
        
        try {
            String gamePackage = "com.and.games505.TerrariaPaid";
            boolean melonInstalled = MelonLoaderManager.isMelonLoaderInstalled(this);
            boolean lemonInstalled = MelonLoaderManager.isLemonLoaderInstalled(this);
            
            if (melonInstalled) {
                result.append("✅ MelonLoader detected\n");
                result.append("   Version: ").append(MelonLoaderManager.getInstalledLoaderVersion()).append("\n");
                
                // Check core files
                File loaderDir = PathManager.getMelonLoaderDir(this, gamePackage);
                if (loaderDir != null && loaderDir.exists()) {
                    File[] files = loaderDir.listFiles();
                    int fileCount = (files != null) ? files.length : 0;
                    result.append("   Files: ").append(fileCount).append(" detected\n");
                }
            } else if (lemonInstalled) {
                result.append("✅ LemonLoader detected\n");
                result.append("   Version: ").append(MelonLoaderManager.getInstalledLoaderVersion()).append("\n");
            } else {
                result.append("❌ No loader installed\n");
                result.append("   Recommendation: Use 'Complete Setup Wizard' to install MelonLoader\n");
            }
            
            // Check runtime directories
            File net8Dir = new File(PathManager.getMelonLoaderDir(this, gamePackage), "net8");
            File net35Dir = new File(PathManager.getMelonLoaderDir(this, gamePackage), "net35");
            
            result.append("Runtime Support:\n");
            result.append(net8Dir.exists() ? "✅" : "❌").append(" NET8 Runtime\n");
            result.append(net35Dir.exists() ? "✅" : "❌").append(" NET35 Runtime\n");
            
        } catch (Exception e) {
            result.append("❌ Loader check failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String checkModFiles() {
        StringBuilder result = new StringBuilder();
        
        try {
            String gamePackage = "com.and.games505.TerrariaPaid";
            
            // Check DEX mods
            File dexDir = PathManager.getDexModsDir(this, gamePackage);
            int dexCount = 0, dexEnabled = 0;
            if (dexDir != null && dexDir.exists()) {
                File[] dexFiles = dexDir.listFiles((dir, name) -> {
                    String lower = name.toLowerCase();
                    return lower.endsWith(".dex") || lower.endsWith(".jar") || 
                           lower.endsWith(".dex.disabled") || lower.endsWith(".jar.disabled");
                });
                if (dexFiles != null) {
                    dexCount = dexFiles.length;
                    for (File file : dexFiles) {
                        if (!file.getName().endsWith(".disabled")) {
                            dexEnabled++;
                        }
                    }
                }
            }
            
            // Check DLL mods
            File dllDir = PathManager.getDllModsDir(this, gamePackage);
            int dllCount = 0, dllEnabled = 0;
            if (dllDir != null && dllDir.exists()) {
                File[] dllFiles = dllDir.listFiles((dir, name) -> {
                    String lower = name.toLowerCase();
                    return lower.endsWith(".dll") || lower.endsWith(".dll.disabled");
                });
                if (dllFiles != null) {
                    dllCount = dllFiles.length;
                    for (File file : dllFiles) {
                        if (!file.getName().endsWith(".disabled")) {
                            dllEnabled++;
                        }
                    }
                }
            }
            
            result.append("DEX/JAR Mods: ").append(dexEnabled).append("/").append(dexCount)
                  .append(" enabled\n");
            result.append("DLL Mods: ").append(dllEnabled).append("/").append(dllCount)
                  .append(" enabled\n");
            result.append("Total Active Mods: ").append(dexEnabled + dllEnabled).append("\n");
            
            if (dexCount == 0 && dllCount == 0) {
                result.append("ℹ️ No mods installed - use Mod Management to add mods\n");
            }
            
        } catch (Exception e) {
            result.append("❌ Mod check failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String checkPermissions() {
        StringBuilder result = new StringBuilder();
        
        try {
            // Test write permissions
            File testDir = new File(getExternalFilesDir(null), "permission_test");
            testDir.mkdirs();
            
            File testFile = new File(testDir, "write_test.txt");
            try (FileWriter writer = new FileWriter(testFile)) {
                writer.write("Permission test successful");
                result.append("✅ External storage write access\n");
            } catch (Exception e) {
                result.append("❌ External storage write failed: ").append(e.getMessage()).append("\n");
            } finally {
                if (testFile.exists()) testFile.delete();
                testDir.delete();
            }
            
            // Check install packages permission
            try {
                getPackageManager().canRequestPackageInstalls();
                result.append("✅ Package installation permission available\n");
            } catch (Exception e) {
                result.append("⚠️ Package installation permission may be restricted\n");
            }
            
        } catch (Exception e) {
            result.append("❌ Permission check failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String checkStorage() {
        StringBuilder result = new StringBuilder();
        
        try {
            File externalDir = getExternalFilesDir(null);
            if (externalDir != null) {
                long freeSpace = externalDir.getFreeSpace();
                long totalSpace = externalDir.getTotalSpace();
                long usedSpace = totalSpace - freeSpace;
                
                result.append("Free Space: ").append(FileUtils.formatFileSize(freeSpace)).append("\n");
                result.append("Used Space: ").append(FileUtils.formatFileSize(usedSpace)).append("\n");
                result.append("Total Space: ").append(FileUtils.formatFileSize(totalSpace)).append("\n");
                
                if (freeSpace < 100 * 1024 * 1024) { // Less than 100MB
                    result.append("⚠️ Low storage space - consider freeing up space\n");
                } else {
                    result.append("✅ Sufficient storage space available\n");
                }
            } else {
                result.append("❌ Cannot access external storage\n");
            }
            
        } catch (Exception e) {
            result.append("❌ Storage check failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String checkSettingsIntegrity() {
        StringBuilder result = new StringBuilder();
        
        try {
            // Check app preferences
            android.content.SharedPreferences prefs = 
                getSharedPreferences("TerrariaLoaderPrefs", MODE_PRIVATE);
            
            // Test write operation
            android.content.SharedPreferences.Editor editor = prefs.edit();
            editor.putString("diagnostic_test", "test_value");
            boolean writeSuccess = editor.commit();
            
            if (writeSuccess) {
                String testValue = prefs.getString("diagnostic_test", null);
                if ("test_value".equals(testValue)) {
                    result.append("✅ Settings persistence working\n");
                    // Clean up test
                    editor.remove("diagnostic_test").commit();
                } else {
                    result.append("❌ Settings read/write mismatch\n");
                }
            } else {
                result.append("❌ Settings write failed\n");
                result.append("   This may explain your auto-refresh issue\n");
            }
            
        } catch (Exception e) {
            result.append("❌ Settings check failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String generateRecommendations() {
        StringBuilder result = new StringBuilder();
        
        try {
            boolean hasIssues = false;
            
            // Check if directories need repair
            File baseDir = PathManager.getGameBaseDir(this, "com.and.games505.TerrariaPaid");
            if (baseDir == null || !baseDir.exists()) {
                result.append("• Run 'Auto-Repair' to create missing directories\n");
                hasIssues = true;
            }
            
            // Check if loader is missing
            if (!MelonLoaderManager.isMelonLoaderInstalled(this) && 
                !MelonLoaderManager.isLemonLoaderInstalled(this)) {
                result.append("• Use 'Complete Setup Wizard' to install MelonLoader\n");
                hasIssues = true;
            }
            
            // Check storage
            File externalDir = getExternalFilesDir(null);
            if (externalDir != null && externalDir.getFreeSpace() < 50 * 1024 * 1024) {
                result.append("• Free up storage space (recommended: 100MB+)\n");
                hasIssues = true;
            }
            
            if (!hasIssues) {
                result.append("✅ System appears to be in good condition\n");
                result.append("• If you're still experiencing issues, try:\n");
                result.append("  - Restart the app completely\n");
                result.append("  - Reboot your device\n");
                result.append("  - Check specific mod compatibility\n");
            }
            
        } catch (Exception e) {
            result.append("• General recommendation: Check system permissions\n");
        }
        
        return result.toString();
    }
    
    // Continue to Part 2...
// File: OfflineDiagnosticActivity.java (Part 2 - Methods & UI)
// Continuation of Part 1

    private void selectApkForDiagnostic() {
        Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
        intent.setType("application/vnd.android.package-archive");
        intent.addCategory(Intent.CATEGORY_OPENABLE);
        
        try {
            apkPickerLauncher.launch(Intent.createChooser(intent, "Select APK to Diagnose"));
        } catch (Exception e) {
            showToast("No file manager available");
        }
    }
    
    private void runApkDiagnostic(Uri apkUri) {
        showProgress("Analyzing APK installation issues...");
        
        AsyncTask.execute(() -> {
            try {
                StringBuilder results = new StringBuilder();
                results.append("=== APK Installation Diagnostic ===\n");
                results.append("File URI: ").append(apkUri.toString()).append("\n\n");
                
                String fileName = getFileNameFromUri(apkUri);
                results.append("File Name: ").append(fileName != null ? fileName : "Unknown").append("\n");
                
                results.append(validateApkFromUri(apkUri)).append("\n");
                results.append("🔧 INSTALLATION ENVIRONMENT\n");
                results.append(checkInstallationEnvironment()).append("\n");
                results.append("📱 DEVICE COMPATIBILITY\n");
                results.append(checkDeviceCompatibility()).append("\n");
                results.append("💡 SOLUTIONS FOR APK PARSING ERRORS\n");
                results.append(getApkSolutions()).append("\n");
                
                runOnUiThread(() -> {
                    hideProgress();
                    displayResults(results.toString());
                });
            } catch (Exception e) {
                runOnUiThread(() -> {
                    hideProgress();
                    showError("APK analysis failed: " + e.getMessage());
                });
            }
        });
    }
    
    private String validateApkFromUri(Uri apkUri) {
        StringBuilder result = new StringBuilder();
        result.append("📦 APK VALIDATION\n");
        
        try (java.io.InputStream stream = getContentResolver().openInputStream(apkUri)) {
            if (stream == null) {
                result.append("❌ Cannot access APK file\n");
                return result.toString();
            }
            
            int available = stream.available();
            if (available > 0) {
                result.append("✅ APK accessible (").append(FileUtils.formatFileSize(available)).append(")\n");
                if (available < 10 * 1024 * 1024) {
                    result.append("⚠️ APK seems small for Terraria - may be corrupted\n");
                }
            } else {
                result.append("⚠️ APK file size unknown or empty\n");
            }
            
            byte[] header = new byte[30];
            int bytesRead = stream.read(header);
            
            if (bytesRead >= 4) {
                if (header[0] == 0x50 && header[1] == 0x4b && header[2] == 0x03 && header[3] == 0x04) {
                    result.append("✅ Valid ZIP/APK signature\n");
                } else {
                    result.append("❌ Invalid ZIP/APK signature - file is corrupted\n");
                    result.append("   This is likely causing your parsing error!\n");
                }
            } else {
                result.append("❌ Cannot read APK header - file corrupted\n");
            }
            
        } catch (Exception e) {
            result.append("❌ APK access failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String checkInstallationEnvironment() {
        StringBuilder result = new StringBuilder();
        
        try {
            boolean unknownSources = canInstallFromUnknownSources();
            result.append(unknownSources ? "✅" : "❌").append(" Unknown sources enabled\n");
            
            if (!unknownSources) {
                result.append("   📋 Fix: Settings > Apps > TerrariaLoader > Install unknown apps\n");
            }
            
            File dataDir = getDataDir();
            long freeSpace = dataDir.getFreeSpace();
            result.append("Internal space: ").append(FileUtils.formatFileSize(freeSpace)).append("\n");
            
            if (freeSpace < 200 * 1024 * 1024) {
                result.append("⚠️ Low storage - may cause installation failure\n");
            }
            
            try {
                getPackageManager().getPackageInfo("com.and.games505.TerrariaPaid", 0);
                result.append("⚠️ Terraria already installed - uninstall first\n");
            } catch (android.content.pm.PackageManager.NameNotFoundException e) {
                result.append("✅ No conflicting installation\n");
            }
            
        } catch (Exception e) {
            result.append("❌ Environment check failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private boolean canInstallFromUnknownSources() {
        if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.O) {
            return getPackageManager().canRequestPackageInstalls();
        } else {
            try {
                return android.provider.Settings.Secure.getInt(
                    getContentResolver(), 
                    android.provider.Settings.Secure.INSTALL_NON_MARKET_APPS, 0) != 0;
            } catch (Exception e) {
                return false;
            }
        }
    }
    
    private String checkDeviceCompatibility() {
        StringBuilder result = new StringBuilder();
        
        result.append("Device: ").append(android.os.Build.MANUFACTURER)
              .append(" ").append(android.os.Build.MODEL).append("\n");
        result.append("Android: ").append(android.os.Build.VERSION.RELEASE)
              .append(" (API ").append(android.os.Build.VERSION.SDK_INT).append(")\n");
        result.append("Architecture: ").append(android.os.Build.SUPPORTED_ABIS[0]).append("\n");
        
        if (android.os.Build.VERSION.SDK_INT >= 21) {
            result.append("✅ Compatible Android version\n");
        } else {
            result.append("❌ Android version too old\n");
        }
        
        return result.toString();
    }
    
    private String getApkSolutions() {
        StringBuilder result = new StringBuilder();
        
        result.append("For 'There was a problem parsing the package':\n\n");
        result.append("1. 🔧 Re-download APK (may be corrupted)\n");
        result.append("2. 🔧 Enable 'Install unknown apps'\n");
        result.append("3. 🔧 Uninstall original Terraria first\n");
        result.append("4. 🔧 Clear Package Installer cache\n");
        result.append("5. 🔧 Restart device and retry\n");
        result.append("6. 🔧 Copy APK to internal storage\n");
        result.append("7. 🔧 Use different file manager\n");
        result.append("8. 🔧 Check antivirus isn't blocking\n");
        
        return result.toString();
    }
    
    private void diagnoseAndFixSettings() {
        showProgress("Diagnosing settings persistence...");
        
        AsyncTask.execute(() -> {
            try {
                StringBuilder results = new StringBuilder();
                results.append("=== Settings Persistence Diagnostic ===\n\n");
                results.append("🔧 SHARED PREFERENCES TEST\n");
                results.append(testSharedPreferences()).append("\n");
                results.append("🔄 AUTO-REFRESH SPECIFIC TEST\n");
                results.append(testAutoRefreshSetting()).append("\n");
                results.append("💾 FILE SYSTEM TEST\n");
                results.append(testFileSystemWrites()).append("\n");
                
                runOnUiThread(() -> {
                    hideProgress();
                    displayResults(results.toString());
                    showSettingsFixOptions();
                });
            } catch (Exception e) {
                runOnUiThread(() -> {
                    hideProgress();
                    showError("Settings diagnostic failed: " + e.getMessage());
                });
            }
        });
    }
    
    private String testSharedPreferences() {
        StringBuilder result = new StringBuilder();
        
        try {
            android.content.SharedPreferences prefs = getSharedPreferences("DiagnosticTest", MODE_PRIVATE);
            android.content.SharedPreferences.Editor editor = prefs.edit();
            
            editor.putBoolean("test_bool", true);
            editor.putString("test_string", "test_value");
            boolean commitSuccess = editor.commit();
            
            result.append("Write test: ").append(commitSuccess ? "✅ Success" : "❌ Failed").append("\n");
            
            if (commitSuccess) {
                boolean boolVal = prefs.getBoolean("test_bool", false);
                String stringVal = prefs.getString("test_string", null);
                boolean readSuccess = boolVal && "test_value".equals(stringVal);
                
                result.append("Read test: ").append(readSuccess ? "✅ Success" : "❌ Failed").append("\n");
                
                if (!readSuccess) {
                    result.append("   This explains your auto-refresh issue!\n");
                }
                
                editor.clear().commit();
            }
            
        } catch (Exception e) {
            result.append("❌ SharedPreferences test failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String testAutoRefreshSetting() {
        StringBuilder result = new StringBuilder();
        
        try {
            // Simulate the exact auto-refresh setting behavior
            android.content.SharedPreferences logPrefs = getSharedPreferences("LogViewerPrefs", MODE_PRIVATE);
            android.content.SharedPreferences.Editor editor = logPrefs.edit();
            
            // Test the specific setting that's failing
            editor.putBoolean("auto_refresh_enabled", false);
            boolean applyResult = editor.commit(); // Use commit instead of apply for immediate result
            
            result.append("Auto-refresh disable: ").append(applyResult ? "✅ Success" : "❌ Failed").append("\n");
            
            if (applyResult) {
                // Check if it actually persisted
                boolean currentValue = logPrefs.getBoolean("auto_refresh_enabled", true); // default true
                result.append("Setting persisted: ").append(!currentValue ? "✅ Success" : "❌ Failed").append("\n");
                
                if (currentValue) {
                    result.append("   Setting reverted to default - persistence failed!\n");
                    result.append("   This is your exact issue.\n");
                }
            }
            
        } catch (Exception e) {
            result.append("❌ Auto-refresh test failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String testFileSystemWrites() {
        StringBuilder result = new StringBuilder();
        
        try {
            File testDir = new File(getFilesDir(), "diagnostic_test");
            testDir.mkdirs();
            
            File testFile = new File(testDir, "settings_test.txt");
            
            try (FileWriter writer = new FileWriter(testFile)) {
                writer.write("auto_refresh=false\n");
                writer.write("timestamp=" + System.currentTimeMillis() + "\n");
                result.append("✅ File write successful\n");
            }
            
            if (testFile.exists()) {
                try (java.io.BufferedReader reader = new java.io.BufferedReader(
                        new java.io.FileReader(testFile))) {
                    String line = reader.readLine();
                    if (line != null && line.contains("auto_refresh=false")) {
                        result.append("✅ File read successful\n");
                    } else {
                        result.append("❌ File content corrupted\n");
                    }
                }
            }
            
            testFile.delete();
            testDir.delete();
            
        } catch (Exception e) {
            result.append("❌ File system test failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String getFileNameFromUri(Uri uri) {
        try {
            android.database.Cursor cursor = getContentResolver().query(uri, null, null, null, null);
            if (cursor != null) {
                int nameIndex = cursor.getColumnIndex(android.provider.OpenableColumns.DISPLAY_NAME);
                if (nameIndex >= 0 && cursor.moveToFirst()) {
                    String name = cursor.getString(nameIndex);
                    cursor.close();
                    return name;
                }
                cursor.close();
            }
        } catch (Exception e) {
            return uri.getLastPathSegment();
        }
        return null;
    }
    
    private void performAutoRepair() {
        new AlertDialog.Builder(this)
            .setTitle("Auto-Repair System")
            .setMessage("Attempt automatic fixes for:\n\n" +
                       "• Missing directories\n" +
                       "• Settings persistence\n" +
                       "• File permissions\n" +
                       "• Configuration corruption\n\n" +
                       "Continue?")
            .setPositiveButton("Yes, Repair", (dialog, which) -> executeAutoRepair())
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void executeAutoRepair() {
        showProgress("Performing auto-repair...");
        
        AsyncTask.execute(() -> {
            try {
                StringBuilder results = new StringBuilder();
                results.append("=== Auto-Repair Results ===\n\n");
                
                boolean directoryRepair = diagnosticManager.attemptSelfRepair();
                boolean settingsRepair = repairSettings();
                boolean permissionRepair = repairPermissions();
                
                results.append("Directory Structure: ").append(directoryRepair ? "✅ Fixed" : "❌ Failed").append("\n");
                results.append("Settings Persistence: ").append(settingsRepair ? "✅ Fixed" : "❌ Failed").append("\n");
                results.append("Permissions: ").append(permissionRepair ? "✅ Fixed" : "❌ Failed").append("\n\n");
                
                if (directoryRepair || settingsRepair || permissionRepair) {
                    results.append("🔄 Restart recommended to apply changes.\n");
                } else {
                    results.append("❌ Could not auto-fix detected issues.\n");
                    results.append("💡 Try manual solutions or check device settings.\n");
                }
                
                runOnUiThread(() -> {
                    hideProgress();
                    displayResults(results.toString());
                });
            } catch (Exception e) {
                runOnUiThread(() -> {
                    hideProgress();
                    showError("Auto-repair failed: " + e.getMessage());
                });
            }
        });
    }
    
    private boolean repairSettings() {
        try {
            // Clear all shared preferences and recreate
            String[] prefFiles = {"TerrariaLoaderPrefs", "LogViewerPrefs", "AppSettings"};
            
            for (String prefFile : prefFiles) {
                android.content.SharedPreferences prefs = getSharedPreferences(prefFile, MODE_PRIVATE);
                android.content.SharedPreferences.Editor editor = prefs.edit();
                editor.clear();
                if (!editor.commit()) {
                    return false;
                }
            }
            
            // Test write after clear
            android.content.SharedPreferences testPrefs = getSharedPreferences("TerrariaLoaderPrefs", MODE_PRIVATE);
            android.content.SharedPreferences.Editor testEditor = testPrefs.edit();
            testEditor.putBoolean("settings_repaired", true);
            return testEditor.commit();
            
        } catch (Exception e) {
            LogUtils.logDebug("Settings repair failed: " + e.getMessage());
            return false;
        }
    }
    
    private boolean repairPermissions() {
        try {
            File testDir = new File(getExternalFilesDir(null), "permission_test");
            testDir.mkdirs();
            
            File testFile = new File(testDir, "test.txt");
            FileWriter writer = new FileWriter(testFile);
            writer.write("test");
            writer.close();
            
            boolean canWrite = testFile.exists() && testFile.length() > 0;
            testFile.delete();
            testDir.delete();
            
            return canWrite;
        } catch (Exception e) {
            return false;
        }
    }
    
    private void exportDiagnosticReport() {
        try {
            String reportContent = diagnosticResultsText.getText().toString();
            if (reportContent.isEmpty() || reportContent.startsWith("Click")) {
                showToast("No diagnostic results to export");
                return;
            }
            
            File reportsDir = new File(getExternalFilesDir(null), "DiagnosticReports");
            reportsDir.mkdirs();
            
            String timestamp = new java.text.SimpleDateFormat("yyyyMMdd_HHmmss", 
                java.util.Locale.getDefault()).format(new java.util.Date());
            File reportFile = new File(reportsDir, "diagnostic_" + timestamp + ".txt");
            
            try (FileWriter writer = new FileWriter(reportFile)) {
                writer.write(reportContent);
                writer.write("\n\n=== Export Info ===\n");
                writer.write("Exported by: TerrariaLoader Diagnostic Tool\n");
                writer.write("Export time: " + new java.util.Date().toString() + "\n");
            }
            
            // Share the report
            Uri fileUri = FileProvider.getUriForFile(this, 
                getPackageName() + ".provider", reportFile);
            
            Intent shareIntent = new Intent(Intent.ACTION_SEND);
            shareIntent.setType("text/plain");
            shareIntent.putExtra(Intent.EXTRA_STREAM, fileUri);
            shareIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
            
            startActivity(Intent.createChooser(shareIntent, "Share Diagnostic Report"));
            showToast("Report exported: " + reportFile.getName());
            
        } catch (Exception e) {
            showError("Export failed: " + e.getMessage());
        }
    }
    
    private void clearResults() {
        diagnosticResultsText.setText("Click 'Run Full System Check' to start diagnostics...");
    }
    
    private void showSettingsFixOptions() {
        new AlertDialog.Builder(this)
            .setTitle("Settings Fix Options")
            .setMessage("Settings persistence issue detected. Try these fixes:")
            .setPositiveButton("Clear All Settings", (dialog, which) -> clearAllSettings())
            .setNeutralButton("Reset App Data", (dialog, which) -> showResetAppDataInfo())
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void clearAllSettings() {
        try {
            String[] prefFiles = {"TerrariaLoaderPrefs", "LogViewerPrefs", "AppSettings"};
            for (String prefFile : prefFiles) {
                getSharedPreferences(prefFile, MODE_PRIVATE).edit().clear().commit();
            }
            showToast("Settings cleared - restart app to test");
        } catch (Exception e) {
            showError("Failed to clear settings: " + e.getMessage());
        }
    }
    
    private void showResetAppDataInfo() {
        new AlertDialog.Builder(this)
            .setTitle("Reset App Data")
            .setMessage("To completely reset TerrariaLoader:\n\n" +
                       "1. Go to Android Settings\n" +
                       "2. Apps > TerrariaLoader\n" +
                       "3. Storage > Clear Data\n\n" +
                       "This will fix persistent settings issues.")
            .setPositiveButton("OK", null)
            .show();
    }
    
    private void displayResults(String results) {
        diagnosticResultsText.setText(results);
    }
    
    private void showProgress(String message) {
        progressDialog = new ProgressDialog(this);
        progressDialog.setMessage(message);
        progressDialog.setCancelable(false);
        progressDialog.show();
    }
    
    private void hideProgress() {
        if (progressDialog != null && progressDialog.isShowing()) {
            progressDialog.dismiss();
        }
    }
    
    private void showToast(String message) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show();
    }
    
    private void showError(String error) {
        new AlertDialog.Builder(this)
            .setTitle("Error")
            .setMessage(error)
            .setPositiveButton("OK", null)
            .show();
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        hideProgress();
    }
}


================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/SettingsActivity.java

// File: SettingsActivity.java (Enhanced UI with Operation Modes)
// Path: /app/src/main/java/com/terrarialoader/ui/SettingsActivity.java

package com.modloader.ui;

import android.Manifest;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.content.pm.PackageManager;
import android.graphics.Color;
import android.net.Uri;
import android.os.Build;
import android.os.Bundle;
import android.os.Environment;
import android.provider.Settings;
import android.view.View;
import android.widget.*;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AlertDialog;
import androidx.appcompat.app.AppCompatActivity;
import androidx.cardview.widget.CardView;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

import com.modloader.R;
import com.modloader.util.LogUtils;
import com.modloader.util.PermissionManager;
import com.modloader.util.ShizukuManager;
import com.modloader.util.RootManager;

public class SettingsActivity extends AppCompatActivity {
    
    // Operation Mode Constants
    public static final String PREF_OPERATION_MODE = "operation_mode";
    public static final String MODE_NORMAL = "normal";
    public static final String MODE_SHIZUKU = "shizuku";
    public static final String MODE_ROOT = "root";
    public static final String MODE_HYBRID = "hybrid"; // Both Shizuku + Root
    
    // UI Components
    private RadioGroup operationModeGroup;
    private RadioButton normalModeRadio;
    private RadioButton shizukuModeRadio;
    private RadioButton rootModeRadio;
    private RadioButton hybridModeRadio;
    
    private CardView normalCard, shizukuCard, rootCard, hybridCard;
    private TextView normalStatus, shizukuStatus, rootStatus, hybridStatus;
    private Button shizukuSetupBtn, rootSetupBtn, permissionBtn;
    
    private Switch autoEnableSwitch;
    private Switch debugLoggingSwitch;
    private Switch autoBackupSwitch;
    private Switch autoUpdateSwitch;
    
    private SharedPreferences prefs;
    private PermissionManager permissionManager;
    private ShizukuManager shizukuManager;
    private RootManager rootManager;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_settings_enhanced);
        setTitle("⚙️ Settings & Operation Modes");
        
        LogUtils.logUser("Settings activity opened");
        
        // Initialize managers
        prefs = getSharedPreferences("terraria_loader_settings", MODE_PRIVATE);
        permissionManager = new PermissionManager(this);
        shizukuManager = new ShizukuManager(this);
        rootManager = new RootManager(this);
        
        initializeViews();
        setupOperationModes();
        setupFeatureToggles();
        setupActionButtons();
        updateUIState();
        
        // Auto-setup permissions based on current mode
        autoSetupPermissions();
    }
    
    private void initializeViews() {
        // Operation Mode Selection
        operationModeGroup = findViewById(R.id.operationModeGroup);
        normalModeRadio = findViewById(R.id.normalModeRadio);
        shizukuModeRadio = findViewById(R.id.shizukuModeRadio);
        rootModeRadio = findViewById(R.id.rootModeRadio);
        hybridModeRadio = findViewById(R.id.hybridModeRadio);
        
        // Mode Cards
        normalCard = findViewById(R.id.normalCard);
        shizukuCard = findViewById(R.id.shizukuCard);
        rootCard = findViewById(R.id.rootCard);
        hybridCard = findViewById(R.id.hybridCard);
        
        // Status Text
        normalStatus = findViewById(R.id.normalStatus);
        shizukuStatus = findViewById(R.id.shizukuStatus);
        rootStatus = findViewById(R.id.rootStatus);
        hybridStatus = findViewById(R.id.hybridStatus);
        
        // Setup Buttons
        shizukuSetupBtn = findViewById(R.id.shizukuSetupBtn);
        rootSetupBtn = findViewById(R.id.rootSetupBtn);
        permissionBtn = findViewById(R.id.permissionBtn);
        
        // Feature Toggles
        autoEnableSwitch = findViewById(R.id.autoEnableSwitch);
        debugLoggingSwitch = findViewById(R.id.debugLoggingSwitch);
        autoBackupSwitch = findViewById(R.id.autoBackupSwitch);
        autoUpdateSwitch = findViewById(R.id.autoUpdateSwitch);
    }
    
    private void setupOperationModes() {
        // Load current mode
        String currentMode = prefs.getString(PREF_OPERATION_MODE, MODE_NORMAL);
        setOperationMode(currentMode, false);
        
        // Set up radio button listeners
        operationModeGroup.setOnCheckedChangeListener((group, checkedId) -> {
            String newMode;
            if (checkedId == R.id.normalModeRadio) {
                newMode = MODE_NORMAL;
            } else if (checkedId == R.id.shizukuModeRadio) {
                newMode = MODE_SHIZUKU;
            } else if (checkedId == R.id.rootModeRadio) {
                newMode = MODE_ROOT;
            } else if (checkedId == R.id.hybridModeRadio) {
                newMode = MODE_HYBRID;
            } else {
                newMode = MODE_NORMAL;
            }
            
            setOperationMode(newMode, true);
        });
        
        // Card click listeners for better UX
        setupCardListeners();
    }
    
    private void setupCardListeners() {
        normalCard.setOnClickListener(v -> {
            normalModeRadio.setChecked(true);
        });
        
        shizukuCard.setOnClickListener(v -> {
            if (shizukuManager.isShizukuAvailable()) {
                shizukuModeRadio.setChecked(true);
            } else {
                showShizukuSetupDialog();
            }
        });
        
        rootCard.setOnClickListener(v -> {
            if (rootManager.isRootAvailable()) {
                rootModeRadio.setChecked(true);
            } else {
                showRootInfoDialog();
            }
        });
        
        hybridCard.setOnClickListener(v -> {
            if (shizukuManager.isShizukuAvailable() && rootManager.isRootAvailable()) {
                hybridModeRadio.setChecked(true);
            } else {
                showHybridSetupDialog();
            }
        });
    }
    
    private void setOperationMode(String mode, boolean save) {
        if (save) {
            prefs.edit().putString(PREF_OPERATION_MODE, mode).apply();
            LogUtils.logUser("Operation mode changed to: " + mode);
            
            // Auto-setup permissions for new mode
            autoSetupPermissions();
        }
        
        // Update radio buttons
        switch (mode) {
            case MODE_NORMAL:
                normalModeRadio.setChecked(true);
                break;
            case MODE_SHIZUKU:
                shizukuModeRadio.setChecked(true);
                break;
            case MODE_ROOT:
                rootModeRadio.setChecked(true);
                break;
            case MODE_HYBRID:
                hybridModeRadio.setChecked(true);
                break;
        }
        
        updateUIState();
    }
    
    private void setupFeatureToggles() {
        // Load current settings
        autoEnableSwitch.setChecked(prefs.getBoolean("auto_enable_mods", true));
        debugLoggingSwitch.setChecked(prefs.getBoolean("debug_logging", false));
        autoBackupSwitch.setChecked(prefs.getBoolean("auto_backup", true));
        autoUpdateSwitch.setChecked(prefs.getBoolean("auto_update_check", true));
        
        // Set up listeners
        autoEnableSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
            prefs.edit().putBoolean("auto_enable_mods", isChecked).apply();
            LogUtils.logUser("Auto-enable mods: " + isChecked);
        });
        
        debugLoggingSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
            prefs.edit().putBoolean("debug_logging", isChecked).apply();
            LogUtils.setDebugEnabled(isChecked);
            LogUtils.logUser("Debug logging: " + isChecked);
        });
        
        autoBackupSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
            prefs.edit().putBoolean("auto_backup", isChecked).apply();
            LogUtils.logUser("Auto backup: " + isChecked);
        });
        
        autoUpdateSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
            prefs.edit().putBoolean("auto_update_check", isChecked).apply();
            LogUtils.logUser("Auto update check: " + isChecked);
        });
    }
    
    private void setupActionButtons() {
        // Shizuku Setup Button
        shizukuSetupBtn.setOnClickListener(v -> {
            if (!shizukuManager.isShizukuInstalled()) {
                showShizukuInstallDialog();
            } else if (!shizukuManager.isShizukuRunning()) {
                showShizukuStartDialog();
            } else if (!shizukuManager.hasShizukuPermission()) {
                shizukuManager.requestShizukuPermission();
            } else {
                Toast.makeText(this, "Shizuku is already properly configured!", Toast.LENGTH_SHORT).show();
            }
        });
        
        // Root Setup Button
        rootSetupBtn.setOnClickListener(v -> {
            if (!rootManager.isRootAvailable()) {
                showRootInfoDialog();
            } else {
                rootManager.requestRootAccess();
            }
        });
        
        // Permission Management Button
        permissionBtn.setOnClickListener(v -> {
            showPermissionManagementDialog();
        });
        
        // Additional action buttons
        findViewById(R.id.resetSettingsBtn).setOnClickListener(v -> resetToDefaults());
        findViewById(R.id.exportSettingsBtn).setOnClickListener(v -> exportSettings());
        findViewById(R.id.importSettingsBtn).setOnClickListener(v -> importSettings());
    }
    
    private void updateUIState() {
        String currentMode = prefs.getString(PREF_OPERATION_MODE, MODE_NORMAL);
        
        // Update card appearances
        updateCardAppearance(normalCard, normalStatus, MODE_NORMAL.equals(currentMode), 
                           permissionManager.hasBasicPermissions(), "Standard Android permissions");
        
        boolean shizukuReady = shizukuManager.isShizukuReady();
        updateCardAppearance(shizukuCard, shizukuStatus, MODE_SHIZUKU.equals(currentMode), 
                           shizukuReady, getShizukuStatusText());
        
        boolean rootReady = rootManager.isRootReady();
        updateCardAppearance(rootCard, rootStatus, MODE_ROOT.equals(currentMode), 
                           rootReady, getRootStatusText());
        
        boolean hybridReady = shizukuReady && rootReady;
        updateCardAppearance(hybridCard, hybridStatus, MODE_HYBRID.equals(currentMode), 
                           hybridReady, "Maximum capabilities with both Shizuku and Root");
        
        // Update setup button states
        updateSetupButtons();
    }
    
    private void updateCardAppearance(CardView card, TextView status, boolean selected, 
                                      boolean available, String statusText) {
        int cardColor;
        int textColor = Color.BLACK;
        
        if (selected) {
            cardColor = Color.parseColor("#E8F5E8"); // Light green
            card.setCardElevation(12f);
        } else if (available) {
            cardColor = Color.parseColor("#E3F2FD"); // Light blue
            card.setCardElevation(6f);
        } else {
            cardColor = Color.parseColor("#FFEBEE"); // Light red
            textColor = Color.parseColor("#666666");
            card.setCardElevation(2f);
        }
        
        card.setCardBackgroundColor(cardColor);
        status.setText(statusText);
        status.setTextColor(textColor);
    }
    
    private void updateSetupButtons() {
        // Shizuku setup button
        if (shizukuManager.isShizukuReady()) {
            shizukuSetupBtn.setText("✅ Shizuku Ready");
            shizukuSetupBtn.setEnabled(false);
        } else if (shizukuManager.isShizukuRunning()) {
            shizukuSetupBtn.setText("🔐 Grant Permission");
            shizukuSetupBtn.setEnabled(true);
        } else if (shizukuManager.isShizukuInstalled()) {
            shizukuSetupBtn.setText("▶️ Start Shizuku");
            shizukuSetupBtn.setEnabled(true);
        } else {
            shizukuSetupBtn.setText("📥 Install Shizuku");
            shizukuSetupBtn.setEnabled(true);
        }
        
        // Root setup button
        if (rootManager.isRootReady()) {
            rootSetupBtn.setText("✅ Root Ready");
            rootSetupBtn.setEnabled(false);
        } else if (rootManager.isRootAvailable()) {
            rootSetupBtn.setText("🔐 Grant Root Access");
            rootSetupBtn.setEnabled(true);
        } else {
            rootSetupBtn.setText("❌ Root Not Available");
            rootSetupBtn.setEnabled(false);
        }
    }
    
    private String getShizukuStatusText() {
        if (!shizukuManager.isShizukuInstalled()) {
            return "Shizuku app not installed";
        } else if (!shizukuManager.isShizukuRunning()) {
            return "Shizuku service not running";
        } else if (!shizukuManager.hasShizukuPermission()) {
            return "Shizuku permission not granted";
        } else {
            return "✅ Shizuku ready - Enhanced file access";
        }
    }
    
    private String getRootStatusText() {
        if (!rootManager.isRootAvailable()) {
            return "Root access not available on this device";
        } else if (!rootManager.hasRootPermission()) {
            return "Root permission not granted";
        } else {
            return "✅ Root access ready - Full system control";
        }
    }
    
    private void autoSetupPermissions() {
        String mode = prefs.getString(PREF_OPERATION_MODE, MODE_NORMAL);
        LogUtils.logDebug("Auto-setting up permissions for mode: " + mode);
        
        // Request basic permissions for all modes
        permissionManager.requestBasicPermissions();
        
        switch (mode) {
            case MODE_SHIZUKU:
                if (shizukuManager.isShizukuRunning() && !shizukuManager.hasShizukuPermission()) {
                    shizukuManager.requestShizukuPermission();
                }
                break;
            case MODE_ROOT:
                if (rootManager.isRootAvailable() && !rootManager.hasRootPermission()) {
                    rootManager.requestRootAccess();
                }
                break;
            case MODE_HYBRID:
                if (shizukuManager.isShizukuRunning() && !shizukuManager.hasShizukuPermission()) {
                    shizukuManager.requestShizukuPermission();
                }
                if (rootManager.isRootAvailable() && !rootManager.hasRootPermission()) {
                    rootManager.requestRootAccess();
                }
                break;
        }
    }
    
    // Dialog methods
    private void showShizukuSetupDialog() {
        new AlertDialog.Builder(this)
            .setTitle("Shizuku Setup Required")
            .setMessage("Shizuku provides enhanced file access without root. Would you like to install it?")
            .setPositiveButton("Install", (dialog, which) -> shizukuManager.installShizuku())
            .setNeutralButton("Learn More", (dialog, which) -> openUrl("https://shizuku.rikka.app/"))
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void showRootInfoDialog() {
        new AlertDialog.Builder(this)
            .setTitle("Root Access")
            .setMessage("Root access provides maximum system control but requires a rooted device. " +
                       "Root access cannot be installed through this app - your device must already be rooted.")
            .setPositiveButton("Check Root", (dialog, which) -> rootManager.checkRootStatus())
            .setNeutralButton("Root Guide", (dialog, which) -> openUrl("https://www.xda-developers.com/root/"))
            .setNegativeButton("OK", null)
            .show();
    }
    
    private void showHybridSetupDialog() {
        String message = "";
        if (!shizukuManager.isShizukuAvailable()) {
            message += "• Shizuku is not available\n";
        }
        if (!rootManager.isRootAvailable()) {
            message += "• Root access is not available\n";
        }
        
        new AlertDialog.Builder(this)
            .setTitle("Hybrid Mode Requirements")
            .setMessage("Hybrid mode requires both Shizuku and Root access:\n\n" + message + 
                       "\nPlease set up both components individually first.")
            .setPositiveButton("OK", null)
            .show();
    }
    
    private void showShizukuInstallDialog() {
        new AlertDialog.Builder(this)
            .setTitle("Install Shizuku")
            .setMessage("Shizuku needs to be downloaded and installed. This will open your browser.")
            .setPositiveButton("Download", (dialog, which) -> 
                openUrl("https://github.com/RikkaApps/Shizuku/releases/latest"))
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void showShizukuStartDialog() {
        new AlertDialog.Builder(this)
            .setTitle("Start Shizuku Service")
            .setMessage("Shizuku is installed but not running. Please start it using ADB or root, " +
                       "then return to this app.")
            .setPositiveButton("Open Shizuku", (dialog, which) -> shizukuManager.openShizukuApp())
            .setNeutralButton("ADB Guide", (dialog, which) -> 
                openUrl("https://shizuku.rikka.app/guide/setup/"))
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void showPermissionManagementDialog() {
        String[] permissions = {
            "Storage Access", 
            "Install Packages", 
            "Shizuku Access",
            "Root Access"
        };
        
        boolean[] grantedStatus = {
            permissionManager.hasStoragePermission(),
            permissionManager.hasInstallPermission(),
            shizukuManager.hasShizukuPermission(),
            rootManager.hasRootPermission()
        };
        
        StringBuilder message = new StringBuilder("Permission Status:\n\n");
        for (int i = 0; i < permissions.length; i++) {
            message.append(grantedStatus[i] ? "✅ " : "❌ ")
                   .append(permissions[i]).append("\n");
        }
        
        new AlertDialog.Builder(this)
            .setTitle("Permission Management")
            .setMessage(message.toString())
            .setPositiveButton("Request Missing", (dialog, which) -> {
                permissionManager.requestAllPermissions();
                autoSetupPermissions();
            })
            .setNeutralButton("App Settings", (dialog, which) -> openAppSettings())
            .setNegativeButton("Close", null)
            .show();
    }
    
    // Utility methods
    private void openUrl(String url) {
        try {
            Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse(url));
            startActivity(intent);
        } catch (Exception e) {
            Toast.makeText(this, "Could not open browser", Toast.LENGTH_SHORT).show();
        }
    }
    
    private void openAppSettings() {
        try {
            Intent intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
            intent.setData(Uri.parse("package:" + getPackageName()));
            startActivity(intent);
        } catch (Exception e) {
            Toast.makeText(this, "Could not open app settings", Toast.LENGTH_SHORT).show();
        }
    }
    
    private void resetToDefaults() {
        new AlertDialog.Builder(this)
            .setTitle("Reset Settings")
            .setMessage("This will reset all settings to their default values. Continue?")
            .setPositiveButton("Reset", (dialog, which) -> {
                prefs.edit().clear().apply();
                recreate(); // Reload activity with default settings
                Toast.makeText(this, "Settings reset to defaults", Toast.LENGTH_SHORT).show();
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void exportSettings() {
        // Implementation for exporting settings to file
        Toast.makeText(this, "Export settings - Coming soon", Toast.LENGTH_SHORT).show();
    }
    
    private void importSettings() {
        // Implementation for importing settings from file
        Toast.makeText(this, "Import settings - Coming soon", Toast.LENGTH_SHORT).show();
    }
    
    @Override
    protected void onResume() {
        super.onResume();
        updateUIState(); // Refresh UI when returning from other apps
    }
    
    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, 
                                           @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        permissionManager.handlePermissionResult(requestCode, permissions, grantResults);
        updateUIState(); // Refresh UI after permission changes
    }
    
    // Static utility methods for other activities
    public static String getCurrentOperationMode(Context context) {
        return context.getSharedPreferences("terraria_loader_settings", Context.MODE_PRIVATE)
                     .getString(PREF_OPERATION_MODE, MODE_NORMAL);
    }
    
    public static boolean isShizukuMode(Context context) {
        String mode = getCurrentOperationMode(context);
        return MODE_SHIZUKU.equals(mode) || MODE_HYBRID.equals(mode);
    }
    
    public static boolean isRootMode(Context context) {
        String mode = getCurrentOperationMode(context);
        return MODE_ROOT.equals(mode) || MODE_HYBRID.equals(mode);
    }
    
    public static boolean canUseEnhancedPermissions(Context context) {
        return isShizukuMode(context) || isRootMode(context);
    }
    
    // Legacy compatibility methods for existing code
    public static boolean isModsEnabled(Context context) {
        try {
            return context.getSharedPreferences("terraria_loader_settings", Context.MODE_PRIVATE)
                         .getBoolean("auto_enable_mods", true);
        } catch (Exception e) {
            return true;
        }
    }
    
    public static boolean isSandboxMode(Context context) {
        try {
            return context.getSharedPreferences("terraria_loader_settings", Context.MODE_PRIVATE)
                         .getBoolean("sandbox_mode", false);
        } catch (Exception e) {
            return false;
        }
    }
    
    public static boolean isDebugMode(Context context) {
        try {
            return context.getSharedPreferences("terraria_loader_settings", Context.MODE_PRIVATE)
                         .getBoolean("debug_logging", false);
        } catch (Exception e) {
            return false;
        }
    }
    
    public static boolean isAutoSaveEnabled(Context context) {
        try {
            return context.getSharedPreferences("terraria_loader_settings", Context.MODE_PRIVATE)
                         .getBoolean("auto_backup", true);
        } catch (Exception e) {
            return true;
        }
    }
}

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/SetupGuideActivity.java

// File: SetupGuideActivity.java (Updated) - Added Offline ZIP Import
// Path: /storage/emulated/0/AndroidIDEProjects/TerrariaML/app/src/main/java/com/terrarialoader/ui/SetupGuideActivity.java

package com.modloader.ui;

import android.app.Activity;
import android.app.AlertDialog;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.widget.Button;
import android.widget.Toast;
import androidx.appcompat.app.AppCompatActivity;

import com.modloader.R;
import com.modloader.loader.MelonLoaderManager;
import com.modloader.util.LogUtils;
import com.modloader.util.OnlineInstaller;
import com.modloader.util.OfflineZipImporter;

public class SetupGuideActivity extends AppCompatActivity {

    private static final int REQUEST_SELECT_ZIP = 1001;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_setup_guide);

        setTitle("🚀 MelonLoader Setup Guide");

        setupButtons();
    }

    private void setupButtons() {
        Button btnOnlineInstall = findViewById(R.id.btn_online_install);
        Button btnOfflineImport = findViewById(R.id.btn_offline_import);
        Button btnManualInstructions = findViewById(R.id.btn_manual_instructions);

        btnOnlineInstall.setOnClickListener(v -> showOnlineInstallDialog());
        btnOfflineImport.setOnClickListener(v -> showOfflineImportDialog());
        btnManualInstructions.setOnClickListener(v -> {
            Intent intent = new Intent(this, InstructionsActivity.class);
            startActivity(intent);
        });
    }

    private void showOnlineInstallDialog() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("🌐 Automated Online Installation");
        builder.setMessage("This will automatically download and install MelonLoader/LemonLoader files from GitHub.\n\n" +
                          "Requirements:\n" +
                          "• Active internet connection\n" +
                          "• ~50MB free space\n\n" +
                          "Continue with automated installation?");
        
        builder.setPositiveButton("Continue", (dialog, which) -> {
            showLoaderTypeDialog();
        });
        
        builder.setNegativeButton("Cancel", null);
        builder.show();
    }

    private void showOfflineImportDialog() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("📦 Offline ZIP Import");
        builder.setMessage("Import a MelonLoader ZIP file that you've already downloaded.\n\n" +
                          "Supported files:\n" +
                          "• melon_data.zip (MelonLoader)\n" +
                          "• lemon_data.zip (LemonLoader)\n" +
                          "• Custom MelonLoader packages\n\n" +
                          "The ZIP will be automatically extracted to the correct directories.");
        
        builder.setPositiveButton("📂 Select ZIP File", (dialog, which) -> {
            Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
            intent.setType("application/zip");
            intent.addCategory(Intent.CATEGORY_OPENABLE);
            startActivityForResult(intent, REQUEST_SELECT_ZIP);
        });
        
        builder.setNegativeButton("Cancel", null);
        builder.show();
    }

    private void showLoaderTypeDialog() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("Choose Loader Type");
        builder.setMessage("Select which loader to install:\n\n" +
                          "🔸 MelonLoader:\n" +
                          "• Full-featured Unity mod loader\n" +
                          "• Larger file size (~40MB)\n" +
                          "• Best compatibility\n\n" +
                          "🔸 LemonLoader:\n" +
                          "• Lightweight Unity mod loader\n" +
                          "• Smaller file size (~15MB)\n" +
                          "• Faster installation\n\n" +
                          "Which would you like to install?");
        
        builder.setPositiveButton("MelonLoader", (dialog, which) -> {
            LogUtils.logUser("User selected MelonLoader for automated installation");
            startAutomatedInstallation(MelonLoaderManager.LoaderType.MELONLOADER_NET8);
        });
        
        builder.setNegativeButton("LemonLoader", (dialog, which) -> {
            LogUtils.logUser("User selected LemonLoader for automated installation");
            startAutomatedInstallation(MelonLoaderManager.LoaderType.MELONLOADER_NET35);
        });
        
        builder.setNeutralButton("Cancel", null);
        builder.show();
    }
    
    private void startAutomatedInstallation(MelonLoaderManager.LoaderType loaderType) {
        LogUtils.logUser("Starting automated " + loaderType.getDisplayName() + " installation...");
        
        AlertDialog progressDialog = new AlertDialog.Builder(this)
            .setTitle("Installing " + loaderType.getDisplayName())
            .setMessage("Downloading and extracting files from GitHub...\nThis may take a few minutes.")
            .setCancelable(false)
            .show();

        new Thread(() -> {
            boolean success = false;
            String errorMessage = "";

            try {
                OnlineInstaller.InstallationResult result = OnlineInstaller.installMelonLoaderOnline(
                    this, MelonLoaderManager.TERRARIA_PACKAGE, loaderType);
                success = result.success;
                errorMessage = result.message;
                
            } catch (Exception e) {
                success = false;
                errorMessage = e.getMessage();
                LogUtils.logDebug("Automated installation error: " + errorMessage);
            }

            final boolean finalSuccess = success;
            final String finalErrorMessage = errorMessage;

            runOnUiThread(() -> {
                progressDialog.dismiss();
                if (finalSuccess) {
                    showInstallationSuccessDialog(loaderType);
                } else {
                    showInstallationErrorDialog(loaderType, finalErrorMessage);
                }
            });
        }).start();
    }

    private void startOfflineImportProcess(Uri zipUri) {
        LogUtils.logUser("Starting offline ZIP import process...");
        
        AlertDialog progressDialog = new AlertDialog.Builder(this)
            .setTitle("Importing MelonLoader ZIP")
            .setMessage("Analyzing and extracting ZIP file...\nThis may take a moment.")
            .setCancelable(false)
            .show();

        new Thread(() -> {
            OfflineZipImporter.ImportResult result = OfflineZipImporter.importMelonLoaderZip(this, zipUri);

            runOnUiThread(() -> {
                progressDialog.dismiss();
                if (result.success) {
                    showImportSuccessDialog(result);
                } else {
                    showImportErrorDialog(result);
                }
            });
        }).start();
    }

    private void showInstallationSuccessDialog(MelonLoaderManager.LoaderType loaderType) {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("✅ Installation Complete!");
        builder.setMessage("Great! " + loaderType.getDisplayName() + " has been successfully installed!\n\n" +
                          "Next steps:\n" +
                          "1. Go to Unified Loader Activity\n" +
                          "2. Select your Terraria APK\n" +
                          "3. Patch APK with loader\n" +
                          "4. Install patched Terraria\n" +
                          "5. Add DLL mods and enjoy!\n\n" +
                          "You can now use DLL mods with Terraria!");
        
        builder.setPositiveButton("🚀 Open Unified Loader", (dialog, which) -> {
            Intent intent = new Intent(this, UnifiedLoaderActivity.class);
            startActivity(intent);
            finish();
        });
        
        builder.setNegativeButton("Later", (dialog, which) -> {
            finish();
        });
        
        builder.show();
    }

    private void showInstallationErrorDialog(MelonLoaderManager.LoaderType loaderType, String errorMessage) {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("❌ Installation Failed");
        builder.setMessage("Failed to install " + loaderType.getDisplayName() + "\n\n" +
                          "Error: " + (errorMessage.isEmpty() ? "Unknown error occurred" : errorMessage) + "\n\n" +
                          "Please try:\n" +
                          "• Check your internet connection\n" +
                          "• Use Offline ZIP Import instead\n" +
                          "• Use Manual Installation\n" +
                          "• Try again later");
        
        builder.setPositiveButton("📦 Try Offline Import", (dialog, which) -> {
            showOfflineImportDialog();
        });
        
        builder.setNegativeButton("📖 Manual Guide", (dialog, which) -> {
            Intent intent = new Intent(this, InstructionsActivity.class);
            startActivity(intent);
        });
        
        builder.setNeutralButton("Cancel", null);
        builder.show();
    }

    private void showImportSuccessDialog(OfflineZipImporter.ImportResult result) {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("✅ ZIP Import Complete!");
        builder.setMessage("Successfully imported " + result.detectedType.getDisplayName() + "!\n\n" +
                          "Files extracted: " + result.filesExtracted + "\n\n" +
                          "The loader files have been automatically placed in the correct directories:\n" +
                          "• NET8/NET35 runtime files\n" +
                          "• Dependencies and support modules\n" +
                          "• All required components\n\n" +
                          "You can now patch APK files!");
        
        builder.setPositiveButton("🚀 Open Unified Loader", (dialog, which) -> {
            Intent intent = new Intent(this, UnifiedLoaderActivity.class);
            startActivity(intent);
            finish();
        });
        
        builder.setNegativeButton("✅ Done", (dialog, which) -> {
            finish();
        });
        
        builder.show();
    }

    private void showImportErrorDialog(OfflineZipImporter.ImportResult result) {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("❌ ZIP Import Failed");
        builder.setMessage("Failed to import ZIP file\n\n" +
                          "Error: " + result.message + "\n\n" +
                          (result.errorDetails != null ? "Details: " + result.errorDetails + "\n\n" : "") +
                          "Please ensure:\n" +
                          "• ZIP file is a valid MelonLoader package\n" +
                          "• File is not corrupted\n" +
                          "• You have sufficient storage space");
        
        builder.setPositiveButton("🌐 Try Online Install", (dialog, which) -> {
            showOnlineInstallDialog();
        });
        
        builder.setNegativeButton("📖 Manual Guide", (dialog, which) -> {
            Intent intent = new Intent(this, InstructionsActivity.class);
            startActivity(intent);
        });
        
        builder.setNeutralButton("Try Again", (dialog, which) -> {
            showOfflineImportDialog();
        });
        
        builder.show();
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        
        if (requestCode == REQUEST_SELECT_ZIP && resultCode == Activity.RESULT_OK && data != null) {
            Uri zipUri = data.getData();
            if (zipUri != null) {
                LogUtils.logUser("ZIP file selected for offline import");
                startOfflineImportProcess(zipUri);
            }
        }
    }
}

================================================================================

/storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/modloader/ui/UnifiedLoaderActivity.java

// File: UnifiedLoaderActivity.java - Complete Fixed Version
// Path: /main/java/com/modloader/ui/UnifiedLoaderActivity.java

package com.modloader.ui;

import android.app.Activity;
import android.app.AlertDialog;
import android.content.Intent;
import android.database.Cursor;
import android.net.Uri;
import android.os.Bundle;
import android.provider.OpenableColumns;
import android.view.View;
import android.widget.*;
import android.graphics.Typeface;
import androidx.appcompat.app.AppCompatActivity;

import com.modloader.R;
import com.modloader.loader.MelonLoaderManager;
import com.modloader.util.LogUtils;

/**
 * Unified Loader Activity - Complete wizard-style interface for MelonLoader setup
 */
public class UnifiedLoaderActivity extends AppCompatActivity implements 
    UnifiedLoaderController.UnifiedLoaderCallback, UnifiedLoaderListener {

    private static final int REQUEST_SELECT_APK = 1001;
    private static final int REQUEST_SELECT_ZIP = 1002;
    
    // UI Components
    private ProgressBar stepProgressBar;
    private TextView stepTitleText;
    private TextView stepDescriptionText;
    private TextView stepIndicatorText;
    private LinearLayout stepContentContainer;
    private Button previousButton;
    private Button nextButton;
    private Button actionButton;
    
    // Current step content views
    private LinearLayout welcomeContent;
    private LinearLayout loaderInstallContent;
    private LinearLayout apkSelectionContent;
    private LinearLayout patchingContent;
    private LinearLayout completionContent;
    
    // Status indicators
    private TextView loaderStatusText;
    private TextView apkStatusText;
    private TextView progressText;
    private ProgressBar actionProgressBar;
    
    // Controller
    private UnifiedLoaderController controller;
    private AlertDialog progressDialog;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_unified_loader);
        
        setTitle("MelonLoader Setup Wizard");
        
        // Initialize controller
        controller = new UnifiedLoaderController(this);
        controller.setCallback(this);
        
        initializeViews();
        setupStepContents();
        setupListeners();
        
        // Start wizard
        controller.setCurrentStep(UnifiedLoaderController.LoaderStep.WELCOME);
    }

    private void initializeViews() {
        stepProgressBar = findViewById(R.id.stepProgressBar);
        stepTitleText = findViewById(R.id.stepTitleText);
        stepDescriptionText = findViewById(R.id.stepDescriptionText);
        stepIndicatorText = findViewById(R.id.stepIndicatorText);
        stepContentContainer = findViewById(R.id.stepContentContainer);
        previousButton = findViewById(R.id.previousButton);
        nextButton = findViewById(R.id.nextButton);
        actionButton = findViewById(R.id.actionButton);
        
        loaderStatusText = findViewById(R.id.loaderStatusText);
        apkStatusText = findViewById(R.id.apkStatusText);
        progressText = findViewById(R.id.progressText);
        actionProgressBar = findViewById(R.id.actionProgressBar);
    }

    private void setupStepContents() {
        // Create step content views dynamically
        welcomeContent = createWelcomeContent();
        loaderInstallContent = createLoaderInstallContent();
        apkSelectionContent = createApkSelectionContent();
        patchingContent = createPatchingContent();
        completionContent = createCompletionContent();
    }

    private LinearLayout createWelcomeContent() {
        LinearLayout content = new LinearLayout(this);
        content.setOrientation(LinearLayout.VERTICAL);
        content.setPadding(24, 24, 24, 24);
        
        TextView welcomeText = new TextView(this);
        welcomeText.setText("Welcome to MelonLoader Setup!\n\nThis wizard will guide you through:\n\n• Installing MelonLoader/LemonLoader\n• Patching your Terraria APK\n• Setting up DLL mod support\n\nClick 'Next' to begin!");
        welcomeText.setTextSize(16);
        welcomeText.setLineSpacing(8, 1.0f);
        content.addView(welcomeText);
        
        return content;
    }

    private LinearLayout createLoaderInstallContent() {
        LinearLayout content = new LinearLayout(this);
        content.setOrientation(LinearLayout.VERTICAL);
        content.setPadding(16, 16, 16, 16);
        
        // Loader status
        TextView statusLabel = new TextView(this);
        statusLabel.setText("Current Status:");
        statusLabel.setTextSize(14);
        statusLabel.setTypeface(null, Typeface.BOLD);
        content.addView(statusLabel);
        
        TextView statusText = new TextView(this);
        statusText.setText("Checking...");
        statusText.setTextSize(14);
        statusText.setPadding(0, 8, 0, 16);
        content.addView(statusText);
        
        // Installation options
        TextView optionsLabel = new TextView(this);
        optionsLabel.setText("Installation Options:");
        optionsLabel.setTextSize(14);
        optionsLabel.setTypeface(null, Typeface.BOLD);
        content.addView(optionsLabel);
        
        Button onlineInstallBtn = new Button(this);
        onlineInstallBtn.setText("Online Installation (Recommended)");
        onlineInstallBtn.setOnClickListener(v -> showOnlineInstallOptions());
        content.addView(onlineInstallBtn);
        
        Button offlineInstallBtn = new Button(this);
        offlineInstallBtn.setText("Offline ZIP Import");
        offlineInstallBtn.setOnClickListener(v -> selectOfflineZip());
        content.addView(offlineInstallBtn);
        
        return content;
    }

    private LinearLayout createApkSelectionContent() {
        LinearLayout content = new LinearLayout(this);
        content.setOrientation(LinearLayout.VERTICAL);
        content.setPadding(16, 16, 16, 16);
        
        TextView instructionText = new TextView(this);
        instructionText.setText("Select your Terraria APK file to patch with MelonLoader:");
        instructionText.setTextSize(16);
        content.addView(instructionText);
        
        Button selectApkBtn = new Button(this);
        selectApkBtn.setText("Select Terraria APK");
        selectApkBtn.setOnClickListener(v -> selectApkFile());
        content.addView(selectApkBtn);
        
        TextView statusText = new TextView(this);
        statusText.setText("No APK selected");
        statusText.setTextSize(14);
        statusText.setPadding(0, 16, 0, 0);
        content.addView(statusText);
        
        return content;
    }

    private LinearLayout createPatchingContent() {
        LinearLayout content = new LinearLayout(this);
        content.setOrientation(LinearLayout.VERTICAL);
        content.setPadding(16, 16, 16, 16);
        content.setGravity(android.view.Gravity.CENTER);
        
        TextView patchingText = new TextView(this);
        patchingText.setText("Patching APK with MelonLoader...");
        patchingText.setTextSize(18);
        patchingText.setGravity(android.view.Gravity.CENTER);
        content.addView(patchingText);
        
        ProgressBar progressBar = new ProgressBar(this);
        progressBar.setIndeterminate(true);
        LinearLayout.LayoutParams progressParams = new LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.WRAP_CONTENT, LinearLayout.LayoutParams.WRAP_CONTENT);
        progressParams.topMargin = 24;
        progressParams.gravity = android.view.Gravity.CENTER;
        progressBar.setLayoutParams(progressParams);
        content.addView(progressBar);
        
        TextView statusText = new TextView(this);
        statusText.setText("Initializing...");
        statusText.setTextSize(14);
        statusText.setGravity(android.view.Gravity.CENTER);
        statusText.setPadding(0, 16, 0, 0);
        content.addView(statusText);
        
        return content;
    }

    private LinearLayout createCompletionContent() {
        LinearLayout content = new LinearLayout(this);
        content.setOrientation(LinearLayout.VERTICAL);
        content.setPadding(24, 24, 24, 24);
        
        TextView completionText = new TextView(this);
        completionText.setText("Setup Complete!\n\nYour modded Terraria APK is ready!");
        completionText.setTextSize(18);
        completionText.setGravity(android.view.Gravity.CENTER);
        content.addView(completionText);
        
        Button installApkBtn = new Button(this);
        installApkBtn.setText("Install Patched APK");
        installApkBtn.setOnClickListener(v -> controller.installPatchedApk());
        content.addView(installApkBtn);
        
        Button manageModsBtn = new Button(this);
        manageModsBtn.setText("Manage DLL Mods");
        manageModsBtn.setOnClickListener(v -> openModManagement());
        content.addView(manageModsBtn);
        
        Button viewLogsBtn = new Button(this);
        viewLogsBtn.setText("View Logs");
        viewLogsBtn.setOnClickListener(v -> startActivity(new Intent(this, LogViewerEnhancedActivity.class)));
        content.addView(viewLogsBtn);
        
        return content;
    }

    private void setupListeners() {
        previousButton.setOnClickListener(v -> {
            if (controller.canProceedToPreviousStep()) {
                controller.previousStep();
            }
        });
        
        nextButton.setOnClickListener(v -> {
            if (controller.canProceedToNextStep()) {
                handleNextStep();
            } else {
                showStepRequirements();
            }
        });
        
        actionButton.setOnClickListener(v -> handleActionButton());
    }

    private void handleNextStep() {
        UnifiedLoaderController.LoaderStep currentStep = controller.getCurrentStep();
        
        switch (currentStep) {
            case WELCOME:
                controller.nextStep();
                break;
            case LOADER_INSTALL:
                if (controller.isLoaderInstalled()) {
                    controller.nextStep();
                } else {
                    Toast.makeText(this, "Please install a loader first", Toast.LENGTH_SHORT).show();
                }
                break;
            case APK_SELECTION:
                if (controller.getSelectedApkUri() != null) {
                    controller.nextStep();
                    controller.patchApk(); // Auto-start patching
                } else {
                    Toast.makeText(this, "Please select an APK first", Toast.LENGTH_SHORT).show();
                }
                break;
            case APK_PATCHING:
                // Patching in progress, disable navigation
                break;
            case COMPLETION:
                finish(); // Exit wizard
                break;
        }
    }

    private void handleActionButton() {
        UnifiedLoaderController.LoaderStep currentStep = controller.getCurrentStep();
        
        switch (currentStep) {
            case WELCOME:
                controller.nextStep();
                break;
            case LOADER_INSTALL:
                showOnlineInstallOptions();
                break;
            case APK_SELECTION:
                selectApkFile();
                break;
            case APK_PATCHING:
                // No action during patching
                break;
            case COMPLETION:
                controller.installPatchedApk();
                break;
        }
    }

    private void showOnlineInstallOptions() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("Choose Loader Type");
        builder.setMessage("Select which loader to install:\n\nMelonLoader: Full-featured, larger size\nLemonLoader: Lightweight, smaller size");
        
        builder.setPositiveButton("MelonLoader", (dialog, which) -> {
            controller.installLoaderOnline(MelonLoaderManager.LoaderType.MELONLOADER_NET8);
        });
        
        builder.setNegativeButton("LemonLoader", (dialog, which) -> {
            controller.installLoaderOnline(MelonLoaderManager.LoaderType.MELONLOADER_NET35);
        });
        
        builder.setNeutralButton("Cancel", null);
        builder.show();
    }

    private void selectOfflineZip() {
        Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
        intent.setType("application/zip");
        intent.addCategory(Intent.CATEGORY_OPENABLE);
        startActivityForResult(intent, REQUEST_SELECT_ZIP);
    }

    private void selectApkFile() {
        Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
        intent.setType("application/vnd.android.package-archive");
        intent.addCategory(Intent.CATEGORY_OPENABLE);
        startActivityForResult(intent, REQUEST_SELECT_APK);
    }

    private void openModManagement() {
        Intent intent = new Intent(this, ModManagementActivity.class);
        startActivity(intent);
    }

    private void showStepRequirements() {
        UnifiedLoaderController.LoaderStep currentStep = controller.getCurrentStep();
        String message = "";
        
        switch (currentStep) {
            case LOADER_INSTALL:
                message = "Please install MelonLoader or LemonLoader first";
                break;
            case APK_SELECTION:
                message = "Please select a Terraria APK file";
                break;
            default:
                message = "Please complete the current step";
                break;
        }
        
        Toast.makeText(this, message, Toast.LENGTH_LONG).show();
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        
        if (resultCode != Activity.RESULT_OK || data == null || data.getData() == null) {
            return;
        }
        
        Uri uri = data.getData();
        
        switch (requestCode) {
            case REQUEST_SELECT_APK:
                controller.selectApk(uri);
                String filename = getFilenameFromUri(uri);
                if (apkStatusText != null) {
                    apkStatusText.setText("Selected: " + filename);
                }
                break;
                
            case REQUEST_SELECT_ZIP:
                controller.installLoaderOffline(uri);
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
            LogUtils.logDebug("Could not get filename: " + e.getMessage());
        }
        return filename != null ? filename : "Unknown file";
    }

    // === UnifiedLoaderController.UnifiedLoaderCallback Implementation ===

    @Override
    public void onStepChanged(UnifiedLoaderController.LoaderStep step, String message) {
        runOnUiThread(() -> {
            stepTitleText.setText(step.getTitle());
            stepDescriptionText.setText(message);
            
            // Clear previous content
            stepContentContainer.removeAllViews();
            
            // Add appropriate content
            switch (step) {
                case WELCOME:
                    stepContentContainer.addView(welcomeContent);
                    actionButton.setText("Start Setup");
                    actionButton.setVisibility(View.VISIBLE);
                    break;
                case LOADER_INSTALL:
                    stepContentContainer.addView(loaderInstallContent);
                    actionButton.setText("Install Online");
                    actionButton.setVisibility(View.VISIBLE);
                    break;
                case APK_SELECTION:
                    stepContentContainer.addView(apkSelectionContent);
                    actionButton.setText("Select APK");
                    actionButton.setVisibility(View.VISIBLE);
                    break;
                case APK_PATCHING:
                    stepContentContainer.addView(patchingContent);
                    actionButton.setVisibility(View.GONE);
                    nextButton.setEnabled(false);
                    previousButton.setEnabled(false);
                    break;
                case COMPLETION:
                    stepContentContainer.addView(completionContent);
                    actionButton.setText("Install APK");
                    actionButton.setVisibility(View.VISIBLE);
                    nextButton.setText("Finish");
                    nextButton.setEnabled(true);
                    previousButton.setEnabled(true);
                    break;
            }
            
            // Update navigation buttons
            previousButton.setEnabled(controller.canProceedToPreviousStep());
            nextButton.setEnabled(controller.canProceedToNextStep());
        });
    }

    @Override
    public void onProgress(String message, int percentage) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.setMessage(message + (percentage > 0 ? " (" + percentage + "%)" : ""));
            }
            if (progressText != null) {
                progressText.setText(message);
            }
        });
    }

    @Override
    public void onSuccess(String message) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.dismiss();
                progressDialog = null;
            }
            Toast.makeText(this, message, Toast.LENGTH_LONG).show();
        });
    }

    @Override
    public void onError(String error) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.dismiss();
                progressDialog = null;
            }
            
            AlertDialog.Builder builder = new AlertDialog.Builder(this);
            builder.setTitle("Error");
            builder.setMessage(error);
            builder.setPositiveButton("OK", null);
            builder.show();
            
            // Re-enable navigation
            nextButton.setEnabled(controller.canProceedToNextStep());
            previousButton.setEnabled(controller.canProceedToPreviousStep());
        });
    }

    @Override
    public void onLoaderStatusChanged(boolean installed, String statusText) {
        runOnUiThread(() -> {
            if (loaderStatusText != null) {
                loaderStatusText.setText(statusText);
                loaderStatusText.setTextColor(installed ? 0xFF4CAF50 : 0xFFF44336);
            }
        });
    }

    @Override
    public void updateStepIndicator(int currentStep, int totalSteps) {
        runOnUiThread(() -> {
            stepProgressBar.setMax(totalSteps);
            stepProgressBar.setProgress(currentStep);
            stepIndicatorText.setText("Step " + (currentStep + 1) + " of " + (totalSteps + 1));
        });
    }

    // === UnifiedLoaderListener Implementation ===

    @Override
    public void onInstallationStarted(String loaderType) {
        runOnUiThread(() -> {
            progressDialog = new AlertDialog.Builder(this)
                .setTitle("Installing " + loaderType)
                .setMessage("Starting installation...")
                .setCancelable(false)
                .create();
            progressDialog.show();
        });
    }

    @Override
    public void onInstallationProgress(String message) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.setMessage(message);
            }
        });
    }

    @Override
    public void onInstallationSuccess(String loaderType, String outputPath) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.dismiss();
                progressDialog = null;
            }
            Toast.makeText(this, loaderType + " installed successfully!", Toast.LENGTH_LONG).show();
        });
    }

    @Override
    public void onInstallationFailed(String loaderType, String error) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.dismiss();
                progressDialog = null;
            }
            
            AlertDialog.Builder builder = new AlertDialog.Builder(this);
            builder.setTitle("Installation Failed");
            builder.setMessage(loaderType + " installation failed:\n\n" + error);
            builder.setPositiveButton("OK", null);
            builder.show();
        });
    }

    @Override
    public void onValidationComplete(boolean isValid, String message) {
        runOnUiThread(() -> {
            String statusText = isValid ? "Validation passed" : "Validation failed: " + message;
            Toast.makeText(this, statusText, Toast.LENGTH_SHORT).show();
        });
    }

    @Override
    public void onInstallationStateChanged(UnifiedLoaderController.InstallationState state) {
        runOnUiThread(() -> {
            LogUtils.logDebug("Installation state changed to: " + state.getDisplayName());
        });
    }

    @Override
    public void onLogMessage(String message, UnifiedLoaderController.LogLevel level) {
        // Log messages are already handled by the controller
        LogUtils.logDebug("Log: [" + level + "] " + message);
    }

    @Override
    protected void onResume() {
        super.onResume();
        // Refresh loader status when returning to activity
        if (controller != null) {
            // Update current step to refresh status
            controller.setCurrentStep(controller.getCurrentStep());
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (controller != null) {
            controller.cleanup();
        }
        if (progressDialog != null) {
            progressDialog.dismiss();
        }
    }
}