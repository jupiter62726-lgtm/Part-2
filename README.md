
--- FILE: /ModLoader/app/src/main/java/com/modloader/loader/debugger/ApkProcessTracker.java ---
// File: ApkProcessTracker.java - Real-time APK operation monitoring and debugging
// Path: app/src/main/java/com/modloader/loader/debug/ApkProcessTracker.java

package com.modloader.loader.debug;

import android.content.Context;
import com.modloader.util.LogUtils;
import com.modloader.util.ApkValidator;
import com.modloader.util.ApkPatcher;
import java.io.*;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

/**
 * ApkProcessTracker - Real-time monitoring and debugging of APK operations
 * Single responsibility: Track, monitor, and debug APK patching processes in real-time
 */
public class ApkProcessTracker {
    private static final String TAG = "APKTracker";
    private final Context context;
    private final Map<String, ActiveProcess> activeProcesses = new ConcurrentHashMap<>();
    private final AtomicLong processIdCounter = new AtomicLong(0);
    private final List<ProcessListener> listeners = new ArrayList<>();
    
    public ApkProcessTracker(Context context) {
        this.context = context;
        LogUtils.logDebug("ApkProcessTracker initialized");
    }
    
    /**
     * Start tracking an APK process with detailed monitoring
     */
    public String startProcess(String operation, File inputApk, File outputApk) {
        String processId = "APK_" + System.currentTimeMillis() + "_" + processIdCounter.incrementAndGet();
        
        ActiveProcess process = new ActiveProcess(processId, operation, inputApk, outputApk);
        activeProcesses.put(processId, process);
        
        LogUtils.logApkProcessStart(operation, inputApk != null ? inputApk.getName() : "Unknown");
        LogUtils.logDebug("Started tracking process: " + processId);
        
        // Notify listeners
        notifyListeners(listener -> listener.onProcessStarted(process));
        
        return processId;
    }
    
    /**
     * Log a detailed step with performance metrics
     */
    public void logStep(String processId, String stepName, String details) {
        ActiveProcess process = activeProcesses.get(processId);
        if (process == null) {
            LogUtils.logDebug("Process not found for step logging: " + processId);
            return;
        }
        
        ProcessStep step = new ProcessStep(stepName, details);
        process.addStep(step);
        
        LogUtils.logApkProcessStep(stepName, details);
        LogUtils.logDebug(String.format("[%s] Step: %s (%dms since start)", 
                                      processId, stepName, step.getElapsedTime()));
        
        // Notify listeners
        notifyListeners(listener -> listener.onStepCompleted(process, step));
    }
    
    /**
     * Log step with file size information
     */
    public void logStepWithSize(String processId, String stepName, String details, long fileSize) {
        String enhancedDetails = details + " (Size: " + formatFileSize(fileSize) + ")";
        logStep(processId, stepName, enhancedDetails);
        
        // Track size changes
        ActiveProcess process = activeProcesses.get(processId);
        if (process != null) {
            process.trackSizeChange(stepName, fileSize);
        }
    }
    
    /**
     * Log step with timing information
     */
    public void logStepWithTiming(String processId, String stepName, String details, long durationMs) {
        String enhancedDetails = details + " (Duration: " + durationMs + "ms)";
        logStep(processId, stepName, enhancedDetails);
        
        // Track performance
        ActiveProcess process = activeProcesses.get(processId);
        if (process != null) {
            process.trackPerformance(stepName, durationMs);
        }
    }
    
    /**
     * Log validation results for the process
     */
    public void logValidationResults(String processId, ApkValidator.ValidationResult validation) {
        ActiveProcess process = activeProcesses.get(processId);
        if (process == null) return;
        
        process.validationResult = validation;
        
        String status = validation.isValid ? "PASSED" : "FAILED";
        String details = String.format("Validation %s - Size: %s, Entries: %d, Issues: %d", 
                                     status, formatFileSize(validation.fileSize), 
                                     validation.totalEntries, validation.issues.size());
        
        logStep(processId, "APK Validation", details);
        
        if (!validation.isValid) {
            for (String issue : validation.issues) {
                LogUtils.logError("Validation issue: " + issue);
            }
        }
    }
    
    /**
     * Log patching results for the process
     */
    public void logPatchingResults(String processId, ApkPatcher.PatchResult patchResult) {
        ActiveProcess process = activeProcesses.get(processId);
        if (process == null) return;
        
        process.patchResult = patchResult;
        
        String status = patchResult.success ? "SUCCESS" : "FAILED";
        String details = String.format("Patching %s - Added: %d files, Modified: %d files, Size change: %s", 
                                     status, patchResult.addedFiles, patchResult.modifiedFiles,
                                     formatFileSize(patchResult.patchedSize - patchResult.originalSize));
        
        logStep(processId, "APK Patching", details);
        
        if (!patchResult.success && patchResult.errorMessage != null) {
            LogUtils.logError("Patching error: " + patchResult.errorMessage);
        }
    }
    
    /**
     * Complete a process with final results
     */
    public void completeProcess(String processId, boolean success, String result) {
        ActiveProcess process = activeProcesses.get(processId);
        if (process == null) {
            LogUtils.logDebug("Process not found for completion: " + processId);
            return;
        }
        
        process.complete(success, result);
        
        LogUtils.logApkProcessComplete(success, result);
        LogUtils.logDebug(String.format("Process completed: %s (%s, %dms total)", 
                                       processId, success ? "SUCCESS" : "FAILED", 
                                       process.getTotalDuration()));
        
        // Notify listeners
        notifyListeners(listener -> listener.onProcessCompleted(process));
        
        // Remove from active processes
        activeProcesses.remove(processId);
    }
    
    /**
     * Get detailed process report
     */
    public String getProcessReport(String processId) {
        ActiveProcess process = activeProcesses.get(processId);
        if (process == null) {
            return "Process not found: " + processId;
        }
        
        return process.generateDetailedReport();
    }
    
    /**
     * Get summary of all active processes
     */
    public String getActiveProcessesSummary() {
        if (activeProcesses.isEmpty()) {
            return "No active APK processes";
        }
        
        StringBuilder summary = new StringBuilder();
        summary.append("=== Active APK Processes ===\n");
        summary.append("Total: ").append(activeProcesses.size()).append("\n\n");
        
        for (ActiveProcess process : activeProcesses.values()) {
            summary.append(process.getSummary()).append("\n");
        }
        
        return summary.toString();
    }
    
    /**
     * Get performance statistics
     */
    public ProcessStatistics getStatistics() {
        ProcessStatistics stats = new ProcessStatistics();
        
        for (ActiveProcess process : activeProcesses.values()) {
            stats.totalProcesses++;
            stats.totalSteps += process.steps.size();
            
            if (process.isCompleted) {
                if (process.success) {
                    stats.successfulProcesses++;
                } else {
                    stats.failedProcesses++;
                }
                
                stats.totalDuration += process.getTotalDuration();
                
                // Track step performance
                for (ProcessStep step : process.steps) {
                    stats.stepPerformance.merge(step.name, step.getElapsedTime(), Long::sum);
                }
            }
        }
        
        return stats;
    }
    
    /**
     * Kill a running process (if possible)
     */
    public boolean killProcess(String processId) {
        ActiveProcess process = activeProcesses.get(processId);
        if (process == null) {
            return false;
        }
        
        process.killed = true;
        logStep(processId, "Process Killed", "Process terminated by user");
        completeProcess(processId, false, "Process killed by user");
        
        LogUtils.logUser("Killed APK process: " + processId);
        return true;
    }
    
    /**
     * Clean up completed processes older than specified time
     */
    public int cleanupOldProcesses(long maxAgeMs) {
        int cleaned = 0;
        long cutoff = System.currentTimeMillis() - maxAgeMs;
        
        Iterator<Map.Entry<String, ActiveProcess>> iterator = activeProcesses.entrySet().iterator();
        while (iterator.hasNext()) {
            Map.Entry<String, ActiveProcess> entry = iterator.next();
            ActiveProcess process = entry.getValue();
            
            if (process.isCompleted && process.startTime < cutoff) {
                iterator.remove();
                cleaned++;
            }
        }
        
        LogUtils.logDebug("Cleaned up " + cleaned + " old APK processes");
        return cleaned;
    }
    
    /**
     * Add a process listener for real-time updates
     */
    public void addProcessListener(ProcessListener listener) {
        synchronized (listeners) {
            listeners.add(listener);
        }
    }
    
    /**
     * Remove a process listener
     */
    public void removeProcessListener(ProcessListener listener) {
        synchronized (listeners) {
            listeners.remove(listener);
        }
    }
    
    // === PRIVATE HELPER METHODS ===
    
    private void notifyListeners(ListenerAction action) {
        synchronized (listeners) {
            for (ProcessListener listener : listeners) {
                try {
                    action.execute(listener);
                } catch (Exception e) {
                    LogUtils.logError("Process listener error", e);
                }
            }
        }
    }
    
    private String formatFileSize(long bytes) {
        if (bytes < 1024) return bytes + " B";
        if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
        return String.format("%.1f MB", bytes / (1024.0 * 1024.0));
    }
    
    // === INTERFACES AND CLASSES ===
    
    @FunctionalInterface
    private interface ListenerAction {
        void execute(ProcessListener listener);
    }
    
    public interface ProcessListener {
        void onProcessStarted(ActiveProcess process);
        void onStepCompleted(ActiveProcess process, ProcessStep step);
        void onProcessCompleted(ActiveProcess process);
    }
    
    public static class ActiveProcess {
        public final String processId;
        public final String operation;
        public final File inputApk;
        public final File outputApk;
        public final long startTime;
        public final List<ProcessStep> steps = new ArrayList<>();
        public final Map<String, Long> sizeChanges = new HashMap<>();
        public final Map<String, Long> performanceMetrics = new HashMap<>();
        
        public boolean isCompleted = false;
        public boolean success = false;
        public boolean killed = false;
        public String result;
        public long completionTime;
        public ApkValidator.ValidationResult validationResult;
        public ApkPatcher.PatchResult patchResult;
        
        public ActiveProcess(String processId, String operation, File inputApk, File outputApk) {
            this.processId = processId;
            this.operation = operation;
            this.inputApk = inputApk;
            this.outputApk = outputApk;
            this.startTime = System.currentTimeMillis();
        }
        
        public void addStep(ProcessStep step) {
            synchronized (steps) {
                steps.add(step);
            }
        }
        
        public void complete(boolean success, String result) {
            this.isCompleted = true;
            this.success = success;
            this.result = result;
            this.completionTime = System.currentTimeMillis();
        }
        
        public void trackSizeChange(String stepName, long size) {
            sizeChanges.put(stepName, size);
        }
        
        public void trackPerformance(String stepName, long durationMs) {
            performanceMetrics.put(stepName, durationMs);
        }
        
        public long getTotalDuration() {
            long endTime = isCompleted ? completionTime : System.currentTimeMillis();
            return endTime - startTime;
        }
        
        public String getSummary() {
            return String.format("[%s] %s: %s (%d steps, %dms)", 
                               processId, operation, 
                               isCompleted ? (success ? "SUCCESS" : "FAILED") : "RUNNING",
                               steps.size(), getTotalDuration());
        }
        
        public String generateDetailedReport() {
            StringBuilder report = new StringBuilder();
            
            report.append("=== APK Process Report ===\n");
            report.append("Process ID: ").append(processId).append("\n");
            report.append("Operation: ").append(operation).append("\n");
            report.append("Input APK: ").append(inputApk != null ? inputApk.getName() : "None").append("\n");
            report.append("Output APK: ").append(outputApk != null ? outputApk.getName() : "None").append("\n");
            report.append("Started: ").append(new Date(startTime)).append("\n");
            
            if (isCompleted) {
                report.append("Completed: ").append(new Date(completionTime)).append("\n");
                report.append("Duration: ").append(getTotalDuration()).append("ms\n");
                report.append("Status: ").append(success ? "SUCCESS" : "FAILED").append("\n");
                if (result != null) {
                    report.append("Result: ").append(result).append("\n");
                }
            } else {
                report.append("Status: RUNNING (").append(getTotalDuration()).append("ms elapsed)\n");
            }
            
            if (killed) {
                report.append("NOTE: Process was killed\n");
            }
            
            report.append("\n=== Process Steps ===\n");
            synchronized (steps) {
                if (steps.isEmpty()) {
                    report.append("No steps recorded\n");
                } else {
                    for (int i = 0; i < steps.size(); i++) {
                        ProcessStep step = steps.get(i);
                        report.append(String.format("Step %d: %s\n", i + 1, step.getDetailedString()));
                        if (step.details != null) {
                            report.append("  Details: ").append(step.details).append("\n");
                        }
                        report.append("  Elapsed: ").append(step.getElapsedTime()).append("ms\n");
                        
                        // Add performance metrics if available
                        Long performance = performanceMetrics.get(step.name);
                        if (performance != null) {
                            report.append("  Duration: ").append(performance).append("ms\n");
                        }
                        
                        // Add size information if available
                        Long size = sizeChanges.get(step.name);
                        if (size != null) {
                            report.append("  File Size: ").append(formatSize(size)).append("\n");
                        }
                        
                        report.append("\n");
                    }
                }
            }
            
            // Add validation results
            if (validationResult != null) {
                report.append("=== Validation Results ===\n");
                report.append("Valid: ").append(validationResult.isValid).append("\n");
                report.append("File Size: ").append(formatSize(validationResult.fileSize)).append("\n");
                report.append("ZIP Entries: ").append(validationResult.totalEntries).append("\n");
                if (validationResult.packageName != null) {
                    report.append("Package: ").append(validationResult.packageName).append("\n");
                }
                if (!validationResult.issues.isEmpty()) {
                    report.append("Issues:\n");
                    for (String issue : validationResult.issues) {
                        report.append("  - ").append(issue).append("\n");
                    }
                }
                report.append("\n");
            }
            
            // Add patching results
            if (patchResult != null) {
                report.append("=== Patching Results ===\n");
                report.append("Success: ").append(patchResult.success).append("\n");
                report.append("Original Size: ").append(formatSize(patchResult.originalSize)).append("\n");
                report.append("Patched Size: ").append(formatSize(patchResult.patchedSize)).append("\n");
                report.append("Size Change: ").append(formatSize(patchResult.patchedSize - patchResult.originalSize)).append("\n");
                report.append("Files Added: ").append(patchResult.addedFiles).append("\n");
                report.append("Files Modified: ").append(patchResult.modifiedFiles).append("\n");
                
                if (!patchResult.injectedFiles.isEmpty()) {
                    report.append("Injected Files:\n");
                    for (String file : patchResult.injectedFiles) {
                        report.append("  + ").append(file).append("\n");
                    }
                }
                
                if (patchResult.errorMessage != null) {
                    report.append("Error: ").append(patchResult.errorMessage).append("\n");
                }
                report.append("\n");
            }
            
            return report.toString();
        }
        
        private String formatSize(long bytes) {
            if (bytes < 1024) return bytes + " B";
            if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
            return String.format("%.1f MB", bytes / (1024.0 * 1024.0));
        }
    }
    
    public static class ProcessStep {
        public final String name;
        public final String details;
        public final long timestamp;
        public final long startTime;
        
        public ProcessStep(String name, String details) {
            this.name = name;
            this.details = details;
            this.timestamp = System.currentTimeMillis();
            this.startTime = System.currentTimeMillis();
        }
        
        public long getElapsedTime() {
            return timestamp - startTime;
        }
        
        public String getDetailedString() {
            return String.format("[%s] %s", 
                               new java.text.SimpleDateFormat("HH:mm:ss.SSS", Locale.getDefault()).format(new Date(timestamp)),
                               name);
        }
    }
    
    public static class ProcessStatistics {
        public int totalProcesses = 0;
        public int successfulProcesses = 0;
        public int failedProcesses = 0;
        public int totalSteps = 0;
        public long totalDuration = 0;
        public Map<String, Long> stepPerformance = new HashMap<>();
        
        public double getSuccessRate() {
            int completed = successfulProcesses + failedProcesses;
            return completed > 0 ? (double) successfulProcesses / completed * 100 : 0;
        }
        
        public long getAverageDuration() {
            return totalProcesses > 0 ? totalDuration / totalProcesses : 0;
        }
        
        @Override
        public String toString() {
            return String.format("Stats[Processes: %d, Success: %.1f%%, Avg Duration: %dms]",
                               totalProcesses, getSuccessRate(), getAverageDuration());
        }
        
        public String getDetailedString() {
            StringBuilder sb = new StringBuilder();
            sb.append("=== APK Process Statistics ===\n");
            sb.append("Total Processes: ").append(totalProcesses).append("\n");
            sb.append("Successful: ").append(successfulProcesses).append("\n");
            sb.append("Failed: ").append(failedProcesses).append("\n");
            sb.append("Success Rate: ").append(String.format("%.1f%%", getSuccessRate())).append("\n");
            sb.append("Total Steps: ").append(totalSteps).append("\n");
            sb.append("Average Duration: ").append(getAverageDuration()).append("ms\n");
            
            if (!stepPerformance.isEmpty()) {
                sb.append("\nStep Performance:\n");
                for (Map.Entry<String, Long> entry : stepPerformance.entrySet()) {
                    sb.append("  ").append(entry.getKey()).append(": ").append(entry.getValue()).append("ms\n");
                }
            }
            
            return sb.toString();
        }
    }
}


--- FILE: /ModLoader/app/src/main/java/com/modloader/loader/debugger/MelonLoaderDebugger.java ---
// File: MelonLoaderDebugger.java - Advanced MelonLoader integration debugging
// Path: app/src/main/java/com/modloader/loader/debug/MelonLoaderDebugger.java

package com.modloader.loader.debug;

import android.content.Context;
import com.modloader.util.LogUtils;
import com.modloader.util.PathManager;
import com.modloader.loader.MelonLoaderManager;
import java.io.*;
import java.text.SimpleDateFormat;
import java.util.*;

/**
 * MelonLoaderDebugger - Advanced debugging for MelonLoader integration issues
 * Single responsibility: Diagnose and debug MelonLoader installation and patching problems
 */
public class MelonLoaderDebugger {
    private static final String TAG = "MLDebugger";
    private final Context context;
    private final String gamePackage;
    private final SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS", Locale.getDefault());
    
    public MelonLoaderDebugger(Context context, String gamePackage) {
        this.context = context;
        this.gamePackage = gamePackage != null ? gamePackage : MelonLoaderManager.TERRARIA_PACKAGE;
    }
    
    /**
     * Comprehensive MelonLoader diagnosis
     */
    public DiagnosisReport runFullDiagnosis() {
        LogUtils.logUser("🔍 Starting comprehensive MelonLoader diagnosis...");
        
        DiagnosisReport report = new DiagnosisReport();
        report.gamePackage = gamePackage;
        report.timestamp = new Date();
        
        try {
            // Step 1: Directory structure analysis
            LogUtils.logApkProcessStep("Diagnosis", "Checking directory structure");
            report.directoryCheck = checkDirectoryStructure();
            
            // Step 2: MelonLoader files validation
            LogUtils.logApkProcessStep("Diagnosis", "Validating MelonLoader files");
            report.filesCheck = validateMelonLoaderFiles();
            
            // Step 3: Dependencies analysis
            LogUtils.logApkProcessStep("Diagnosis", "Analyzing dependencies");
            report.dependenciesCheck = checkDependencies();
            
            // Step 4: Architecture compatibility
            LogUtils.logApkProcessStep("Diagnosis", "Checking architecture compatibility");
            report.architectureCheck = checkArchitectureCompatibility();
            
            // Step 5: Permissions analysis
            LogUtils.logApkProcessStep("Diagnosis", "Checking permissions");
            report.permissionsCheck = checkPermissions();
            
            // Step 6: Previous installation analysis
            LogUtils.logApkProcessStep("Diagnosis", "Analyzing previous installations");
            report.previousInstallCheck = checkPreviousInstallations();
            
            // Overall health assessment
            report.overallHealth = calculateOverallHealth(report);
            
            LogUtils.logUser("✅ MelonLoader diagnosis completed");
            LogUtils.logUser("🎯 Overall health: " + report.overallHealth.getDisplayName());
            
        } catch (Exception e) {
            LogUtils.logError("MelonLoader diagnosis failed", e);
            report.criticalError = e.getMessage();
        }
        
        return report;
    }
    
    /**
     * Quick health check for MelonLoader integration
     */
    public HealthStatus quickHealthCheck() {
        try {
            // Check essential components quickly
            File baseDir = PathManager.getGameBaseDir(context, gamePackage);
            File melonDir = PathManager.getMelonLoaderDir(context, gamePackage);
            File dllModsDir = PathManager.getDllModsDir(context, gamePackage);
            
            if (baseDir == null || !baseDir.exists()) {
                return HealthStatus.CRITICAL;
            }
            
            if (melonDir == null || !melonDir.exists()) {
                return HealthStatus.NOT_INSTALLED;
            }
            
            if (dllModsDir == null || !dllModsDir.exists()) {
                return HealthStatus.DEGRADED;
            }
            
            // Check for core files
            File net8Dir = new File(melonDir, "net8");
            File net35Dir = new File(melonDir, "net35");
            
            if (net8Dir.exists() || net35Dir.exists()) {
                return HealthStatus.HEALTHY;
            }
            
            return HealthStatus.DEGRADED;
            
        } catch (Exception e) {
            LogUtils.logError("Quick health check failed", e);
            return HealthStatus.CRITICAL;
        }
    }
    
    /**
     * Generate detailed diagnosis report text
     */
    public String generateDiagnosisReport(DiagnosisReport report) {
        StringBuilder sb = new StringBuilder();
        
        sb.append("=== MelonLoader Diagnosis Report ===\n");
        sb.append("Game Package: ").append(report.gamePackage).append("\n");
        sb.append("Diagnosis Time: ").append(dateFormat.format(report.timestamp)).append("\n");
        sb.append("Overall Health: ").append(report.overallHealth.getDisplayName()).append("\n\n");
        
        if (report.criticalError != null) {
            sb.append("CRITICAL ERROR: ").append(report.criticalError).append("\n\n");
        }
        
        // Directory structure
        sb.append("=== Directory Structure ===\n");
        appendCheckResult(sb, report.directoryCheck);
        
        // Files validation
        sb.append("=== MelonLoader Files ===\n");
        appendCheckResult(sb, report.filesCheck);
        
        // Dependencies
        sb.append("=== Dependencies ===\n");
        appendCheckResult(sb, report.dependenciesCheck);
        
        // Architecture
        sb.append("=== Architecture Compatibility ===\n");
        appendCheckResult(sb, report.architectureCheck);
        
        // Permissions
        sb.append("=== Permissions ===\n");
        appendCheckResult(sb, report.permissionsCheck);
        
        // Previous installations
        sb.append("=== Previous Installations ===\n");
        appendCheckResult(sb, report.previousInstallCheck);
        
        // Recommendations
        sb.append("=== Recommendations ===\n");
        appendRecommendations(sb, report);
        
        return sb.toString();
    }
    
    /**
     * Auto-repair common MelonLoader issues
     */
    public RepairResult attemptAutoRepair() {
        LogUtils.logUser("🔧 Attempting MelonLoader auto-repair...");
        
        RepairResult result = new RepairResult();
        
        try {
            // Repair directory structure
            LogUtils.logApkProcessStep("Auto-repair", "Fixing directory structure");
            if (repairDirectoryStructure()) {
                result.repairedIssues.add("Directory structure recreated");
                result.successCount++;
            }
            
            // Repair permissions
            LogUtils.logApkProcessStep("Auto-repair", "Fixing permissions");
            if (repairPermissions()) {
                result.repairedIssues.add("Permissions fixed");
                result.successCount++;
            }
            
            // Clean up corrupted files
            LogUtils.logApkProcessStep("Auto-repair", "Cleaning corrupted files");
            if (cleanupCorruptedFiles()) {
                result.repairedIssues.add("Corrupted files cleaned");
                result.successCount++;
            }
            
            // Verify repair success
            HealthStatus healthAfterRepair = quickHealthCheck();
            result.success = healthAfterRepair != HealthStatus.CRITICAL;
            result.finalHealth = healthAfterRepair;
            
            if (result.success) {
                LogUtils.logUser("✅ Auto-repair completed successfully");
                LogUtils.logUser("🎯 Final health: " + healthAfterRepair.getDisplayName());
            } else {
                LogUtils.logUser("⚠️ Auto-repair completed with issues");
            }
            
        } catch (Exception e) {
            LogUtils.logError("Auto-repair failed", e);
            result.error = e.getMessage();
        }
        
        return result;
    }
    
    /**
     * Get installation troubleshooting steps
     */
    public List<TroubleshootingStep> getTroubleshootingSteps() {
        List<TroubleshootingStep> steps = new ArrayList<>();
        HealthStatus health = quickHealthCheck();
        
        switch (health) {
            case NOT_INSTALLED:
                steps.add(new TroubleshootingStep("Install MelonLoader", 
                    "Use the 'Automated Installation' option to download and install MelonLoader files", 
                    TroubleshootingStep.Priority.HIGH));
                break;
                
            case CRITICAL:
                steps.add(new TroubleshootingStep("Check Storage Space", 
                    "Ensure you have at least 100MB free space", 
                    TroubleshootingStep.Priority.HIGH));
                steps.add(new TroubleshootingStep("Check Permissions", 
                    "Grant storage permissions to the app", 
                    TroubleshootingStep.Priority.HIGH));
                break;
                
            case DEGRADED:
                steps.add(new TroubleshootingStep("Reinstall MelonLoader", 
                    "Try reinstalling MelonLoader to fix missing components", 
                    TroubleshootingStep.Priority.MEDIUM));
                steps.add(new TroubleshootingStep("Clear App Data", 
                    "Clear app data and restart the installation process", 
                    TroubleshootingStep.Priority.LOW));
                break;
                
            case HEALTHY:
                steps.add(new TroubleshootingStep("Check APK Compatibility", 
                    "Ensure your Terraria APK is compatible with MelonLoader", 
                    TroubleshootingStep.Priority.LOW));
                break;
        }
        
        // Common troubleshooting steps
        steps.add(new TroubleshootingStep("Restart App", 
            "Close and restart UniversalLoader", 
            TroubleshootingStep.Priority.LOW));
        steps.add(new TroubleshootingStep("Check Logs", 
            "Review error logs for specific issues", 
            TroubleshootingStep.Priority.LOW));
        
        return steps;
    }
    
    // === PRIVATE DIAGNOSTIC METHODS ===
    
    private CheckResult checkDirectoryStructure() {
        CheckResult result = new CheckResult("Directory Structure");
        
        try {
            File baseDir = PathManager.getGameBaseDir(context, gamePackage);
            File melonDir = PathManager.getMelonLoaderDir(context, gamePackage);
            File dllModsDir = PathManager.getDllModsDir(context, gamePackage);
            File dexModsDir = PathManager.getDexModsDir(context, gamePackage);
            
            // Check base directory
            if (baseDir == null) {
                result.issues.add("Base directory path is null");
                result.status = CheckResult.Status.FAILED;
            } else if (!baseDir.exists()) {
                result.issues.add("Base directory does not exist: " + baseDir.getAbsolutePath());
                result.status = CheckResult.Status.FAILED;
            } else {
                result.details.add("Base directory: " + baseDir.getAbsolutePath());
                result.passedChecks++;
            }
            
            // Check MelonLoader directory
            if (melonDir == null) {
                result.issues.add("MelonLoader directory path is null");
            } else if (!melonDir.exists()) {
                result.issues.add("MelonLoader directory missing");
            } else {
                result.details.add("MelonLoader directory: " + melonDir.getAbsolutePath());
                result.passedChecks++;
            }
            
            // Check mods directories
            if (dllModsDir != null && dllModsDir.exists()) {
                result.details.add("DLL mods directory: " + dllModsDir.getAbsolutePath());
                result.passedChecks++;
            } else {
                result.issues.add("DLL mods directory missing");
            }
            
            if (dexModsDir != null && dexModsDir.exists()) {
                result.details.add("DEX mods directory: " + dexModsDir.getAbsolutePath());
                result.passedChecks++;
            } else {
                result.issues.add("DEX mods directory missing");
            }
            
            result.totalChecks = 4;
            
            if (result.passedChecks == result.totalChecks) {
                result.status = CheckResult.Status.PASSED;
            } else if (result.passedChecks > 0) {
                result.status = CheckResult.Status.WARNING;
            } else {
                result.status = CheckResult.Status.FAILED;
            }
            
        } catch (Exception e) {
            result.issues.add("Directory check error: " + e.getMessage());
            result.status = CheckResult.Status.FAILED;
        }
        
        return result;
    }
    
    private CheckResult validateMelonLoaderFiles() {
        CheckResult result = new CheckResult("MelonLoader Files");
        
        try {
            File melonDir = PathManager.getMelonLoaderDir(context, gamePackage);
            if (melonDir == null || !melonDir.exists()) {
                result.issues.add("MelonLoader directory not found");
                result.status = CheckResult.Status.FAILED;
                return result;
            }
            
            // Check runtime directories
            File net8Dir = new File(melonDir, "net8");
            File net35Dir = new File(melonDir, "net35");
            
            boolean hasNet8 = checkRuntimeFiles(net8Dir, "NET8", result);
            boolean hasNet35 = checkRuntimeFiles(net35Dir, "NET35", result);
            
            if (hasNet8 || hasNet35) {
                result.details.add("Runtime support: " + 
                    (hasNet8 ? "NET8 " : "") + (hasNet35 ? "NET35" : ""));
                result.passedChecks++;
            } else {
                result.issues.add("No valid runtime found (NET8 or NET35)");
            }
            
            // Check dependencies
            File depsDir = new File(melonDir, "Dependencies");
            if (depsDir.exists()) {
                result.details.add("Dependencies directory found");
                result.passedChecks++;
                
                File supportDir = new File(depsDir, "SupportModules");
                if (supportDir.exists()) {
                    result.details.add("Support modules found");
                    result.passedChecks++;
                }
            } else {
                result.issues.add("Dependencies directory missing");
            }
            
            result.totalChecks = 3;
            
            if (result.passedChecks == result.totalChecks) {
                result.status = CheckResult.Status.PASSED;
            } else if (result.passedChecks > 0) {
                result.status = CheckResult.Status.WARNING;
            } else {
                result.status = CheckResult.Status.FAILED;
            }
            
        } catch (Exception e) {
            result.issues.add("File validation error: " + e.getMessage());
            result.status = CheckResult.Status.FAILED;
        }
        
        return result;
    }
    
    private boolean checkRuntimeFiles(File runtimeDir, String runtimeName, CheckResult result) {
        if (!runtimeDir.exists()) {
            return false;
        }
        
        // Check for essential runtime files
        String[] essentialFiles = {
            "MelonLoader.dll",
            "0Harmony.dll"
        };
        
        int foundFiles = 0;
        for (String filename : essentialFiles) {
            File file = new File(runtimeDir, filename);
            if (file.exists() && file.length() > 0) {
                foundFiles++;
            }
        }
        
        if (foundFiles >= essentialFiles.length / 2) {
            result.details.add(runtimeName + " runtime files: " + foundFiles + "/" + essentialFiles.length);
            return true;
        }
        
        return false;
    }
    
    private CheckResult checkDependencies() {
        CheckResult result = new CheckResult("Dependencies");
        
        try {
            File melonDir = PathManager.getMelonLoaderDir(context, gamePackage);
            if (melonDir == null || !melonDir.exists()) {
                result.issues.add("MelonLoader directory not found");
                result.status = CheckResult.Status.FAILED;
                return result;
            }
            
            File depsDir = new File(melonDir, "Dependencies");
            if (!depsDir.exists()) {
                result.issues.add("Dependencies directory not found");
                result.status = CheckResult.Status.FAILED;
                return result;
            }
            
            // Check key dependency directories
            String[] depDirs = {
                "SupportModules",
                "Il2CppAssemblyGenerator",
                "CompatibilityLayers"
            };
            
            for (String dirName : depDirs) {
                File dir = new File(depsDir, dirName);
                if (dir.exists()) {
                    result.details.add(dirName + ": Found");
                    result.passedChecks++;
                } else {
                    result.issues.add(dirName + ": Missing");
                }
            }
            
            result.totalChecks = depDirs.length;
            
            if (result.passedChecks == result.totalChecks) {
                result.status = CheckResult.Status.PASSED;
            } else if (result.passedChecks > 0) {
                result.status = CheckResult.Status.WARNING;
            } else {
                result.status = CheckResult.Status.FAILED;
            }
            
        } catch (Exception e) {
            result.issues.add("Dependencies check error: " + e.getMessage());
            result.status = CheckResult.Status.FAILED;
        }
        
        return result;
    }
    
    private CheckResult checkArchitectureCompatibility() {
        CheckResult result = new CheckResult("Architecture Compatibility");
        
        try {
            String deviceArch = android.os.Build.SUPPORTED_ABIS[0];
            result.details.add("Device architecture: " + deviceArch);
            
            File melonDir = PathManager.getMelonLoaderDir(context, gamePackage);
            if (melonDir != null && melonDir.exists()) {
                File runtimesDir = new File(melonDir, "Dependencies/Il2CppAssemblyGenerator/runtimes");
                if (runtimesDir.exists()) {
                    // Check for architecture-specific libraries
                    boolean hasArm64 = new File(runtimesDir, "linux-arm64").exists();
                    boolean hasArm = new File(runtimesDir, "linux-arm").exists();
                    
                    result.details.add("ARM64 support: " + hasArm64);
                    result.details.add("ARM support: " + hasArm);
                    
                    if ((deviceArch.contains("arm64") && hasArm64) || 
                        (deviceArch.contains("arm") && hasArm)) {
                        result.status = CheckResult.Status.PASSED;
                        result.details.add("Architecture compatibility: Good");
                    } else {
                        result.status = CheckResult.Status.WARNING;
                        result.issues.add("Native libraries may not match device architecture");
                    }
                } else {
                    result.status = CheckResult.Status.WARNING;
                    result.issues.add("Native libraries directory not found");
                }
            } else {
                result.status = CheckResult.Status.FAILED;
                result.issues.add("MelonLoader not installed");
            }
            
        } catch (Exception e) {
            result.issues.add("Architecture check error: " + e.getMessage());
            result.status = CheckResult.Status.FAILED;
        }
        
        return result;
    }
    
    private CheckResult checkPermissions() {
        CheckResult result = new CheckResult("Permissions");
        
        try {
            File baseDir = PathManager.getGameBaseDir(context, gamePackage);
            
            // Test write permission
            File testFile = new File(baseDir != null ? baseDir : context.getExternalFilesDir(null), 
                                   "permission_test.tmp");
            
            try {
                if (testFile.createNewFile()) {
                    result.details.add("Write permission: Available");
                    result.passedChecks++;
                    testFile.delete();
                } else {
                    result.issues.add("Cannot create test file");
                }
            } catch (Exception e) {
                result.issues.add("Write permission test failed: " + e.getMessage());
            }
            
            // Check directory permissions
            if (baseDir != null) {
                result.details.add("Base directory readable: " + baseDir.canRead());
                result.details.add("Base directory writable: " + baseDir.canWrite());
                
                if (baseDir.canRead() && baseDir.canWrite()) {
                    result.passedChecks++;
                }
            }
            
            result.totalChecks = 2;
            result.status = result.passedChecks == result.totalChecks ? 
                           CheckResult.Status.PASSED : CheckResult.Status.WARNING;
            
        } catch (Exception e) {
            result.issues.add("Permissions check error: " + e.getMessage());
            result.status = CheckResult.Status.FAILED;
        }
        
        return result;
    }
    
    private CheckResult checkPreviousInstallations() {
        CheckResult result = new CheckResult("Previous Installations");
        
        try {
            // Check for backup APKs
            File backupDir = PathManager.getBackupsDir(context, gamePackage);
            if (backupDir != null && backupDir.exists()) {
                File[] backups = backupDir.listFiles((dir, name) -> name.endsWith(".apk"));
                if (backups != null && backups.length > 0) {
                    result.details.add("APK backups found: " + backups.length);
                    result.passedChecks++;
                } else {
                    result.details.add("No APK backups found");
                }
            }
            
            // Check for legacy installations
            File legacyDir = PathManager.getLegacyModsDir(context);
            if (legacyDir.exists()) {
                result.details.add("Legacy installation detected");
                result.issues.add("Legacy installation may cause conflicts");
            }
            
            result.totalChecks = 1;
            result.status = CheckResult.Status.PASSED; // This is informational
            
        } catch (Exception e) {
            result.issues.add("Previous installation check error: " + e.getMessage());
            result.status = CheckResult.Status.WARNING;
        }
        
        return result;
    }
    
    // === REPAIR METHODS ===
    
    private boolean repairDirectoryStructure() {
        try {
            return PathManager.initializeGameDirectories(context, gamePackage);
        } catch (Exception e) {
            LogUtils.logError("Directory structure repair failed", e);
            return false;
        }
    }
    
    private boolean repairPermissions() {
        try {
            File baseDir = PathManager.getGameBaseDir(context, gamePackage);
            if (baseDir != null && baseDir.exists()) {
                // Try to fix directory permissions (limited on Android)
                baseDir.setReadable(true, false);
                baseDir.setWritable(true, false);
                return true;
            }
        } catch (Exception e) {
            LogUtils.logError("Permissions repair failed", e);
        }
        return false;
    }
    
    private boolean cleanupCorruptedFiles() {
        try {
            // Remove zero-byte files that might be corrupted
            File melonDir = PathManager.getMelonLoaderDir(context, gamePackage);
            if (melonDir != null && melonDir.exists()) {
                return cleanupCorruptedFilesInDirectory(melonDir);
            }
        } catch (Exception e) {
            LogUtils.logError("Corrupted files cleanup failed", e);
        }
        return false;
    }
    
    private boolean cleanupCorruptedFilesInDirectory(File directory) {
        boolean cleaned = false;
        File[] files = directory.listFiles();
        
        if (files != null) {
            for (File file : files) {
                if (file.isDirectory()) {
                    cleaned |= cleanupCorruptedFilesInDirectory(file);
                } else if (file.length() == 0) {
                    // Remove zero-byte files
                    if (file.delete()) {
                        LogUtils.logDebug("Removed corrupted file: " + file.getName());
                        cleaned = true;
                    }
                }
            }
        }
        
        return cleaned;
    }
    
    // === HELPER METHODS ===
    
    private HealthStatus calculateOverallHealth(DiagnosisReport report) {
        int passedChecks = 0;
        int totalChecks = 0;
        int failedChecks = 0;
        
        CheckResult[] checks = {
            report.directoryCheck, report.filesCheck, report.dependenciesCheck,
            report.architectureCheck, report.permissionsCheck, report.previousInstallCheck
        };
        
        for (CheckResult check : checks) {
            if (check != null) {
                totalChecks++;
                if (check.status == CheckResult.Status.PASSED) {
                    passedChecks++;
                } else if (check.status == CheckResult.Status.FAILED) {
                    failedChecks++;
                }
            }
        }
        
        if (report.criticalError != null) {
            return HealthStatus.CRITICAL;
        }
        
        if (failedChecks > totalChecks / 2) {
            return HealthStatus.CRITICAL;
        } else if (passedChecks == totalChecks) {
            return HealthStatus.HEALTHY;
        } else if (passedChecks > totalChecks / 2) {
            return HealthStatus.DEGRADED;
        } else {
            return HealthStatus.NOT_INSTALLED;
        }
    }
    
    private void appendCheckResult(StringBuilder sb, CheckResult check) {
        if (check == null) {
            sb.append("Check not performed\n\n");
            return;
        }
        
        sb.append("Status: ").append(check.status.getDisplayName()).append("\n");
        sb.append("Passed: ").append(check.passedChecks).append("/").append(check.totalChecks).append("\n");
        
        if (!check.details.isEmpty()) {
            sb.append("Details:\n");
            for (String detail : check.details) {
                sb.append("  ✓ ").append(detail).append("\n");
            }
        }
        
        if (!check.issues.isEmpty()) {
            sb.append("Issues:\n");
            for (String issue : check.issues) {
                sb.append("  ❌ ").append(issue).append("\n");
            }
        }
        
        sb.append("\n");
    }
    
    private void appendRecommendations(StringBuilder sb, DiagnosisReport report) {
        List<String> recommendations = new ArrayList<>();
        
        if (report.overallHealth == HealthStatus.NOT_INSTALLED) {
            recommendations.add("Install MelonLoader using the automated installation option");
        }
        
        if (report.directoryCheck != null && report.directoryCheck.status == CheckResult.Status.FAILED) {
            recommendations.add("Create missing directories using the 'Initialize Structure' option");
        }
        
        if (report.filesCheck != null && report.filesCheck.status != CheckResult.Status.PASSED) {
            recommendations.add("Reinstall MelonLoader files");
        }
        
        if (report.permissionsCheck != null && report.permissionsCheck.status != CheckResult.Status.PASSED) {
            recommendations.add("Grant storage permissions to the app in Android settings");
        }
        
        if (recommendations.isEmpty()) {
            recommendations.add("No specific recommendations - system appears healthy");
        }
        
        for (String rec : recommendations) {
            sb.append("• ").append(rec).append("\n");
        }
    }
    
    // === ENUMS AND CLASSES ===
    
    public enum HealthStatus {
        HEALTHY("Healthy", "✅"),
        DEGRADED("Degraded", "⚠️"),
        NOT_INSTALLED("Not Installed", "❌"),
        CRITICAL("Critical", "🔥");
        
        private final String displayName;
        private final String icon;
        
        HealthStatus(String displayName, String icon) {
            this.displayName = displayName;
            this.icon = icon;
        }
        
        public String getDisplayName() {
            return icon + " " + displayName;
        }
    }
    
    public static class DiagnosisReport {
        public String gamePackage;
        public Date timestamp;
        public HealthStatus overallHealth;
        public String criticalError;
        
        public CheckResult directoryCheck;
        public CheckResult filesCheck;
        public CheckResult dependenciesCheck;
        public CheckResult architectureCheck;
        public CheckResult permissionsCheck;
        public CheckResult previousInstallCheck;
    }
    
    public static class CheckResult {
        public enum Status {
            PASSED("Passed", "✅"),
            WARNING("Warning", "⚠️"),
            FAILED("Failed", "❌");
            
            private final String displayName;
            private final String icon;
            
            Status(String displayName, String icon) {
                this.displayName = displayName;
                this.icon = icon;
            }
            
            public String getDisplayName() {
                return icon + " " + displayName;
            }
        }
        
        public String checkName;
        public Status status = Status.FAILED;
        public List<String> details = new ArrayList<>();
        public List<String> issues = new ArrayList<>();
        public int passedChecks = 0;
        public int totalChecks = 0;
        
        public CheckResult(String checkName) {
            this.checkName = checkName;
        }
    }
    
    public static class RepairResult {
        public boolean success = false;
        public int successCount = 0;
        public List<String> repairedIssues = new ArrayList<>();
        public String error = null;
        public HealthStatus finalHealth = HealthStatus.CRITICAL;
    }
    
    public static class TroubleshootingStep {
        public enum Priority {
            HIGH, MEDIUM, LOW
        }
        
        public String title;
        public String description;
        public Priority priority;
        
        public TroubleshootingStep(String title, String description, Priority priority) {
            this.title = title;
            this.description = description;
            this.priority = priority;
        }
    }
}


--- FILE: /ModLoader/app/src/main/java/com/modloader/logging/AdvancedLogManager.java ---
// =========================
// AdvancedLogManager.java
// =========================
// File: AdvancedLogManager.java - Complete Implementation
// Path: /app/src/main/java/com/modloader/logging/AdvancedLogManager.java

package com.modloader.logging;

import android.content.Context;
import android.util.Log;
import java.io.*;
import java.text.SimpleDateFormat;
import java.util.*;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

/**
 * Advanced Log Manager for comprehensive file operations, log rotation, and memory management
 */
public class AdvancedLogManager {
    private static final String TAG = "AdvancedLogManager";
    
    // Configuration constants
    private static final long MAX_LOG_FILE_SIZE = 10 * 1024 * 1024; // 10MB
    private static final int MAX_LOG_FILES = 10;
    private static final int MAX_MEMORY_LOGS = 5000;
    private static final long LOG_ROTATION_INTERVAL = 24 * 60 * 60 * 1000; // 24 hours
    
    // Context and file management
    private final Context context;
    private final File logDirectory;
    private final File currentLogFile;
    private FileWriter logFileWriter;
    
    // Memory log storage
    private final Queue<LogEntry> memoryLogs = new ConcurrentLinkedQueue<>();
    private final Map<String, UserAction> userActions = new ConcurrentHashMap<>();
    private final Map<String, AtomicLong> logCategoryStats = new ConcurrentHashMap<>();
    
    // Performance tracking
    private final AtomicLong totalLogWrites = new AtomicLong(0);
    private final AtomicLong totalBytesWritten = new AtomicLong(0);
    private final AtomicLong lastRotationTime = new AtomicLong(System.currentTimeMillis());
    
    // Date formatters
    private static final SimpleDateFormat filenameDateFormat = 
        new SimpleDateFormat("yyyyMMdd_HHmmss", Locale.getDefault());
    private static final SimpleDateFormat logTimestampFormat = 
        new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS", Locale.getDefault());
    
    // Log filtering and processing
    private final Set<String> filteredPatterns = new HashSet<>();
    private volatile boolean compressionEnabled = true;
    
    public AdvancedLogManager(Context context) {
        this.context = context;
        
        // Initialize log directory structure
        File baseDir = context.getExternalFilesDir(null);
        File gameDir = new File(baseDir, "ModLoader/com.and.games505.TerrariaPaid");
        this.logDirectory = new File(gameDir, "AppLogs");
        
        // Ensure directory exists
        if (!logDirectory.exists() && !logDirectory.mkdirs()) {
            Log.e(TAG, "Failed to create log directory: " + logDirectory.getAbsolutePath());
        }
        
        // Initialize current log file
        this.currentLogFile = new File(logDirectory, "AppLog.txt");
        
        // Initialize log category stats
        logCategoryStats.put("DEBUG", new AtomicLong(0));
        logCategoryStats.put("INFO", new AtomicLong(0));
        logCategoryStats.put("WARN", new AtomicLong(0));
        logCategoryStats.put("ERROR", new AtomicLong(0));
        logCategoryStats.put("USER", new AtomicLong(0));
        
        // Initialize filtered patterns for noise reduction
        initializeFilterPatterns();
        
        // Initialize log file writer
        initializeLogWriter();
        
        Log.d(TAG, "AdvancedLogManager initialized - Log directory: " + logDirectory.getAbsolutePath());
    }
    
    /**
     * Initialize patterns to filter out noise from logs
     */
    private void initializeFilterPatterns() {
        // Common noise patterns that should be filtered
        filteredPatterns.add(".*Created: /storage/emulated/0.*");
        filteredPatterns.add(".*exists: true.*");
        filteredPatterns.add(".*checking.*found.*");
        filteredPatterns.add(".*Step \\d+ of \\d+.*");
        filteredPatterns.add(".*\\d+/\\d+ directories created.*");
    }
    
    /**
     * Initialize log file writer
     */
    private void initializeLogWriter() {
        try {
            logFileWriter = new FileWriter(currentLogFile, true); // Append mode
            writeLogHeader();
        } catch (IOException e) {
            Log.e(TAG, "Failed to initialize log writer", e);
        }
    }
    
    /**
     * Write header information to log file
     */
    private void writeLogHeader() {
        try {
            String header = String.format("=== ModLoader Log Session Started ===\n" +
                "Timestamp: %s\n" +
                "Device: %s %s\n" +
                "Android: %s (API %d)\n" +
                "App Version: %s\n" +
                "Log Directory: %s\n" +
                "=====================================\n\n",
                logTimestampFormat.format(new Date()),
                android.os.Build.MANUFACTURER,
                android.os.Build.MODEL,
                android.os.Build.VERSION.RELEASE,
                android.os.Build.VERSION.SDK_INT,
                getAppVersion(),
                logDirectory.getAbsolutePath()
            );
            
            if (logFileWriter != null) {
                logFileWriter.write(header);
                logFileWriter.flush();
            }
        } catch (IOException e) {
            Log.e(TAG, "Failed to write log header", e);
        }
    }
    
    /**
     * Get app version information
     */
    private String getAppVersion() {
        try {
            return context.getPackageManager()
                .getPackageInfo(context.getPackageName(), 0).versionName;
        } catch (Exception e) {
            return "Unknown";
        }
    }
    
    /**
     * Track user action with detailed context
     */
    public void trackUserAction(String action) {
        if (action == null || action.trim().isEmpty()) return;
        
        UserAction userAction = new UserAction(action, System.currentTimeMillis());
        userActions.put(UUID.randomUUID().toString(), userAction);
        
        // Keep only recent actions (last 1000)
        if (userActions.size() > 1000) {
            Iterator<String> iterator = userActions.keySet().iterator();
            for (int i = 0; i < 100 && iterator.hasNext(); i++) {
                iterator.next();
                iterator.remove();
            }
        }
        
        // Write to file
        writeToFile("USER_ACTION", "INFO", action);
        
        Log.d(TAG, "Tracked user action: " + action);
    }
    
    /**
     * Write log entry to both memory and file
     */
    public void writeLog(String category, String level, String message) {
        writeLog(category, level, message, null);
    }
    
    /**
     * Write log entry with optional throwable
     */
    public void writeLog(String category, String level, String message, Throwable throwable) {
        if (shouldFilterMessage(message)) {
            return; // Skip filtered messages
        }
        
        // Create log entry
        LogEntry entry = new LogEntry(category, level, message, throwable, System.currentTimeMillis());
        
        // Add to memory storage
        memoryLogs.offer(entry);
        if (memoryLogs.size() > MAX_MEMORY_LOGS) {
            memoryLogs.poll(); // Remove oldest
        }
        
        // Update statistics
        totalLogWrites.incrementAndGet();
        logCategoryStats.get(level).incrementAndGet();
        
        // Write to file
        writeToFile(category, level, message, throwable);
        
        // Check if rotation is needed
        checkLogRotation();
    }
    
    /**
     * Check if message should be filtered out
     */
    private boolean shouldFilterMessage(String message) {
        if (message == null) return true;
        
        for (String pattern : filteredPatterns) {
            if (message.matches(pattern)) {
                return true;
            }
        }
        
        return false;
    }
    
    /**
     * Write entry to physical log file
     */
    private void writeToFile(String category, String level, String message) {
        writeToFile(category, level, message, null);
    }
    
    /**
     * Write entry to physical log file with throwable
     */
    private void writeToFile(String category, String level, String message, Throwable throwable) {
        try {
            if (logFileWriter == null) {
                initializeLogWriter();
            }
            
            String logLine = formatLogEntry(category, level, message, throwable);
            logFileWriter.write(logLine);
            logFileWriter.flush();
            
            totalBytesWritten.addAndGet(logLine.length());
            
        } catch (IOException e) {
            Log.e(TAG, "Failed to write to log file", e);
            // Try to reinitialize writer
            try {
                if (logFileWriter != null) {
                    logFileWriter.close();
                }
                initializeLogWriter();
            } catch (IOException ex) {
                Log.e(TAG, "Failed to reinitialize log writer", ex);
            }
        }
    }
    
    /**
     * Format log entry for file output
     */
    private String formatLogEntry(String category, String level, String message, Throwable throwable) {
        StringBuilder entry = new StringBuilder();
        
        entry.append("[").append(logTimestampFormat.format(new Date())).append("] ");
        entry.append("[").append(level).append("] ");
        entry.append("[").append(category).append("] ");
        entry.append(message).append("\n");
        
        if (throwable != null) {
            StringWriter sw = new StringWriter();
            PrintWriter pw = new PrintWriter(sw);
            throwable.printStackTrace(pw);
            entry.append("Exception: ").append(sw.toString()).append("\n");
        }
        
        return entry.toString();
    }
    
    /**
     * Perform comprehensive log rotation
     */
    public void performLogRotation() {
        try {
            long currentTime = System.currentTimeMillis();
            boolean forceRotation = (currentTime - lastRotationTime.get()) > LOG_ROTATION_INTERVAL;
            boolean sizeRotation = currentLogFile.length() > MAX_LOG_FILE_SIZE;
            
            if (!forceRotation && !sizeRotation) {
                return; // No rotation needed
            }
            
            Log.d(TAG, "Starting log rotation - Force: " + forceRotation + ", Size: " + sizeRotation);
            
            // Close current writer
            if (logFileWriter != null) {
                logFileWriter.close();
                logFileWriter = null;
            }
            
            // Rotate existing files
            rotateLogFiles();
            
            // Create new current log file
            initializeLogWriter();
            
            // Clean up old files
            cleanupOldLogFiles();
            
            lastRotationTime.set(currentTime);
            
            Log.d(TAG, "Log rotation completed successfully");
            
        } catch (Exception e) {
            Log.e(TAG, "Log rotation failed", e);
            try {
                // Ensure we have a working log writer
                initializeLogWriter();
            } catch (Exception ex) {
                Log.e(TAG, "Failed to reinitialize after rotation failure", ex);
            }
        }
    }
    
    /**
     * Rotate log files (AppLog.txt -> AppLog1.txt -> AppLog2.txt, etc.)
     */
    private void rotateLogFiles() throws IOException {
        // Move files in reverse order
        for (int i = MAX_LOG_FILES - 1; i >= 1; i--) {
            File currentFile = new File(logDirectory, "AppLog" + i + ".txt");
            File nextFile = new File(logDirectory, "AppLog" + (i + 1) + ".txt");
            
            if (currentFile.exists()) {
                if (nextFile.exists()) {
                    nextFile.delete();
                }
                currentFile.renameTo(nextFile);
            }
        }
        
        // Move current log to AppLog1.txt
        if (currentLogFile.exists()) {
            File rotatedFile = new File(logDirectory, "AppLog1.txt");
            if (rotatedFile.exists()) {
                rotatedFile.delete();
            }
            currentLogFile.renameTo(rotatedFile);
        }
    }
    
    /**
     * Clean up old log files beyond the maximum limit
     */
    private void cleanupOldLogFiles() {
        try {
            File[] logFiles = logDirectory.listFiles((dir, name) -> 
                name.startsWith("AppLog") && name.endsWith(".txt"));
            
            if (logFiles == null) return;
            
            // Sort by modification time (oldest first)
            Arrays.sort(logFiles, Comparator.comparingLong(File::lastModified));
            
            // Delete excess files
            for (int i = 0; i < logFiles.length - MAX_LOG_FILES; i++) {
                if (logFiles[i].delete()) {
                    Log.d(TAG, "Deleted old log file: " + logFiles[i].getName());
                }
            }
            
        } catch (Exception e) {
            Log.e(TAG, "Failed to clean up old log files", e);
        }
    }
    
    /**
     * Check if log rotation is needed
     */
    private void checkLogRotation() {
        if (currentLogFile.length() > MAX_LOG_FILE_SIZE) {
            performLogRotation();
        }
    }
    
    /**
     * Get all logs formatted as a single string
     */
    public String getAllLogsFormatted() {
        StringBuilder allLogs = new StringBuilder();
        
        // Add header
        allLogs.append("=== In-Memory Logs (Recent ").append(memoryLogs.size()).append(" entries) ===\n\n");
        
        // Add memory logs
        for (LogEntry entry : memoryLogs) {
            allLogs.append(entry.getFormattedEntry()).append("\n");
        }
        
        // Add file logs if memory is empty
        if (memoryLogs.isEmpty()) {
            allLogs.append("=== Reading from log file ===\n\n");
            try {
                if (currentLogFile.exists()) {
                    BufferedReader reader = new BufferedReader(new FileReader(currentLogFile));
                    String line;
                    while ((line = reader.readLine()) != null) {
                        allLogs.append(line).append("\n");
                    }
                    reader.close();
                }
            } catch (IOException e) {
                allLogs.append("Error reading log file: ").append(e.getMessage()).append("\n");
            }
        }
        
        return allLogs.toString();
    }
    
    /**
     * Clear all logs from memory and files
     */
    public void clearAllLogs() {
        try {
            // Clear memory logs
            memoryLogs.clear();
            
            // Close current writer
            if (logFileWriter != null) {
                logFileWriter.close();
                logFileWriter = null;
            }
            
            // Delete all log files
            File[] logFiles = logDirectory.listFiles((dir, name) -> 
                name.startsWith("AppLog") && name.endsWith(".txt"));
            
            if (logFiles != null) {
                for (File file : logFiles) {
                    if (file.delete()) {
                        Log.d(TAG, "Deleted log file: " + file.getName());
                    }
                }
            }
            
            // Reset statistics
            totalLogWrites.set(0);
            totalBytesWritten.set(0);
            for (AtomicLong counter : logCategoryStats.values()) {
                counter.set(0);
            }
            
            // Reinitialize log writer
            initializeLogWriter();
            
            Log.d(TAG, "All logs cleared successfully");
            
        } catch (Exception e) {
            Log.e(TAG, "Failed to clear logs", e);
        }
    }
    
    /**
     * Get list of available log files
     */
    public List<File> getAvailableLogFiles() {
        List<File> files = new ArrayList<>();
        try {
            File[] logFiles = logDirectory.listFiles((dir, name) -> 
                name.startsWith("AppLog") && name.endsWith(".txt"));
            
            if (logFiles != null) {
                // Sort by modification time (newest first)
                Arrays.sort(logFiles, (a, b) -> Long.compare(b.lastModified(), a.lastModified()));
                files.addAll(Arrays.asList(logFiles));
            }
        } catch (Exception e) {
            Log.e(TAG, "Failed to get available log files", e);
        }
        return files;
    }
    
    /**
     * Read content from a specific log file
     */
    public String readLogFile(File logFile) {
        if (logFile == null || !logFile.exists()) {
            return "Log file does not exist";
        }
        
        try {
            StringBuilder content = new StringBuilder();
            BufferedReader reader = new BufferedReader(new FileReader(logFile));
            String line;
            
            while ((line = reader.readLine()) != null) {
                content.append(line).append("\n");
            }
            
            reader.close();
            return content.toString();
            
        } catch (IOException e) {
            Log.e(TAG, "Failed to read log file: " + logFile.getName(), e);
            return "Error reading file: " + e.getMessage();
        }
    }
    
    /**
     * Get log statistics
     */
    public Map<String, Object> getLogStatistics() {
        Map<String, Object> stats = new HashMap<>();
        
        stats.put("totalLogWrites", totalLogWrites.get());
        stats.put("totalBytesWritten", totalBytesWritten.get());
        stats.put("memoryLogsCount", memoryLogs.size());
        stats.put("userActionsCount", userActions.size());
        stats.put("logDirectoryPath", logDirectory.getAbsolutePath());
        stats.put("currentLogFileSize", currentLogFile.length());
        stats.put("lastRotationTime", new Date(lastRotationTime.get()));
        
        // Category statistics
        Map<String, Long> categoryStats = new HashMap<>();
        for (Map.Entry<String, AtomicLong> entry : logCategoryStats.entrySet()) {
            categoryStats.put(entry.getKey(), entry.getValue().get());
        }
        stats.put("categoryStats", categoryStats);
        
        // File statistics
        List<File> logFiles = getAvailableLogFiles();
        stats.put("totalLogFiles", logFiles.size());
        
        long totalLogSize = 0;
        for (File file : logFiles) {
            totalLogSize += file.length();
        }
        stats.put("totalLogSize", totalLogSize);
        
        return stats;
    }
    
    /**
     * Export logs to external file
     */
    public File exportLogs(String exportFileName) {
        try {
            File exportsDir = new File(context.getExternalFilesDir(null), "exports");
            if (!exportsDir.exists()) {
                exportsDir.mkdirs();
            }
            
            File exportFile = new File(exportsDir, exportFileName);
            FileWriter writer = new FileWriter(exportFile);
            
            // Write header
            writer.write("=== ModLoader Log Export ===\n");
            writer.write("Export Date: " + logTimestampFormat.format(new Date()) + "\n");
            writer.write("Total Entries: " + memoryLogs.size() + "\n\n");
            
            // Write all logs
            writer.write(getAllLogsFormatted());
            
            // Write statistics
            writer.write("\n\n=== Log Statistics ===\n");
            Map<String, Object> stats = getLogStatistics();
            for (Map.Entry<String, Object> entry : stats.entrySet()) {
                writer.write(entry.getKey() + ": " + entry.getValue() + "\n");
            }
            
            writer.close();
            
            Log.d(TAG, "Logs exported to: " + exportFile.getAbsolutePath());
            return exportFile;
            
        } catch (IOException e) {
            Log.e(TAG, "Failed to export logs", e);
            return null;
        }
    }
    
    /**
     * Clean shutdown with final log write
     */
    public void shutdown() {
        try {
            Log.d(TAG, "AdvancedLogManager shutting down");
            
            // Write shutdown message
            writeToFile("SYSTEM", "INFO", "AdvancedLogManager shutdown - Session ended");
            
            // Write final statistics
            Map<String, Object> stats = getLogStatistics();
            writeToFile("SYSTEM", "INFO", "Final statistics: " + stats.toString());
            
            // Close log writer
            if (logFileWriter != null) {
                logFileWriter.close();
                logFileWriter = null;
            }
            
            Log.d(TAG, "AdvancedLogManager shutdown complete");
            
        } catch (Exception e) {
            Log.e(TAG, "Error during shutdown", e);
        }
    }
    
    // =========================
    // INNER CLASSES
    // =========================
    
    /**
     * Represents a log entry with full context
     */
    private static class LogEntry {
        final String category;
        final String level;
        final String message;
        final Throwable throwable;
        final long timestamp;
        final String formattedTime;
        
        LogEntry(String category, String level, String message, Throwable throwable, long timestamp) {
            this.category = category;
            this.level = level;
            this.message = message;
            this.throwable = throwable;
            this.timestamp = timestamp;
            this.formattedTime = logTimestampFormat.format(new Date(timestamp));
        }
        
        String getFormattedEntry() {
            StringBuilder entry = new StringBuilder();
            entry.append("[").append(formattedTime).append("] ");
            entry.append("[").append(level).append("] ");
            entry.append("[").append(category).append("] ");
            entry.append(message);
            
            if (throwable != null) {
                entry.append("\n  Exception: ").append(throwable.getMessage());
            }
            
            return entry.toString();
        }
    }
    
    /**
     * Represents a user action with timestamp
     */
    private static class UserAction {
        final String action;
        final long timestamp;
        final String formattedTime;
        
        UserAction(String action, long timestamp) {
            this.action = action;
            this.timestamp = timestamp;
            this.formattedTime = logTimestampFormat.format(new Date(timestamp));
        }
    }
}


--- FILE: /ModLoader/app/src/main/java/com/modloader/logging/ApkProcessLogger.java ---
// File: ApkProcessLogger.java - Specialized APK process tracking
// Path: app/src/main/java/com/terrarialoader/logging/ApkProcessLogger.java

package com.modloader.logging;

import android.content.Context;
import java.io.*;
import java.text.SimpleDateFormat;
import java.util.*;

/**
 * ApkProcessLogger - Tracks APK patching, validation, and installation processes
 * Single responsibility: Log detailed APK operation steps and results
 */
public class ApkProcessLogger {
    private static final String APK_LOG_FILE = "APKProcess.log";
    private static final int MAX_PROCESSES = 100;
    
    private final Context context;
    private final File logFile;
    private final SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS", Locale.getDefault());
    private final List<ApkProcess> activeProcesses = new ArrayList<>();
    private final List<ApkProcess> completedProcesses = new ArrayList<>();
    
    public ApkProcessLogger(Context context) {
        this.context = context;
        File logDir = new File(context.getExternalFilesDir(null), "TerrariaLoader/com.and.games505.TerrariaPaid/AppLogs");
        logDir.mkdirs();
        this.logFile = new File(logDir, APK_LOG_FILE);
        
        // Load existing processes if log file exists
        loadExistingProcesses();
    }
    
    /**
     * Start tracking a new APK process
     */
    public synchronized String startProcess(String operation, String apkName) {
        String processId = generateProcessId();
        ApkProcess process = new ApkProcess(processId, operation, apkName);
        activeProcesses.add(process);
        
        logToFile("=== PROCESS STARTED ===");
        logToFile("Process ID: " + processId);
        logToFile("Operation: " + operation);
        logToFile("APK: " + apkName);
        logToFile("Started: " + dateFormat.format(new Date()));
        logToFile("");
        
        return processId;
    }
    
    /**
     * Log a step in the current process
     */
    public synchronized void logStep(String step, String details) {
        ApkProcess currentProcess = getCurrentProcess();
        if (currentProcess == null) {
            // Create anonymous process if none exists
            currentProcess = new ApkProcess(generateProcessId(), "Unknown", "Unknown");
            activeProcesses.add(currentProcess);
        }
        
        ProcessStep processStep = new ProcessStep(step, details);
        currentProcess.addStep(processStep);
        
        logToFile(String.format("[%s] STEP: %s", 
                               dateFormat.format(processStep.timestamp), step));
        if (details != null && !details.isEmpty()) {
            logToFile("  Details: " + details);
        }
    }
    
    /**
     * Complete the current process
     */
    public synchronized void completeProcess(boolean success, String result) {
        ApkProcess currentProcess = getCurrentProcess();
        if (currentProcess == null) return;
        
        currentProcess.complete(success, result);
        activeProcesses.remove(currentProcess);
        completedProcesses.add(currentProcess);
        
        // Keep only recent completed processes
        while (completedProcesses.size() > MAX_PROCESSES) {
            completedProcesses.remove(0);
        }
        
        logToFile("=== PROCESS COMPLETED ===");
        logToFile("Process ID: " + currentProcess.processId);
        logToFile("Status: " + (success ? "SUCCESS" : "FAILED"));
        logToFile("Duration: " + currentProcess.getDurationString());
        if (result != null) {
            logToFile("Result: " + result);
        }
        logToFile("Total Steps: " + currentProcess.steps.size());
        logToFile("");
    }
    
    /**
     * Log APK validation results
     */
    public synchronized void logValidation(String apkName, boolean valid, String details) {
        logToFile("=== APK VALIDATION ===");
        logToFile("APK: " + apkName);
        logToFile("Valid: " + (valid ? "YES" : "NO"));
        logToFile("Timestamp: " + dateFormat.format(new Date()));
        if (details != null) {
            logToFile("Details:");
            String[] lines = details.split("\n");
            for (String line : lines) {
                logToFile("  " + line);
            }
        }
        logToFile("");
    }
    
    /**
     * Get detailed report of current/recent processes
     */
    public synchronized String getProcessReport() {
        StringBuilder report = new StringBuilder();
        
        report.append("=== APK PROCESS REPORT ===\n");
        report.append("Generated: ").append(dateFormat.format(new Date())).append("\n\n");
        
        // Active processes
        if (!activeProcesses.isEmpty()) {
            report.append("ACTIVE PROCESSES (" + activeProcesses.size() + "):\n");
            for (ApkProcess process : activeProcesses) {
                report.append(process.getDetailedReport()).append("\n");
            }
        }
        
        // Recent completed processes (last 10)
        if (!completedProcesses.isEmpty()) {
            report.append("RECENT COMPLETED PROCESSES:\n");
            int start = Math.max(0, completedProcesses.size() - 10);
            for (int i = start; i < completedProcesses.size(); i++) {
                ApkProcess process = completedProcesses.get(i);
                report.append(process.getSummaryReport()).append("\n");
            }
        }
        
        return report.toString();
    }
    
    /**
     * Export APK process logs to a file
     */
    public boolean exportProcessLogs(File outputFile) {
        try (PrintWriter writer = new PrintWriter(new FileWriter(outputFile))) {
            writer.println(getProcessReport());
            
            // Include raw log content
            writer.println("\n=== RAW LOG CONTENT ===\n");
            if (logFile.exists()) {
                try (BufferedReader reader = new BufferedReader(new FileReader(logFile))) {
                    String line;
                    while ((line = reader.readLine()) != null) {
                        writer.println(line);
                    }
                }
            }
            
            return true;
        } catch (Exception e) {
            android.util.Log.e("ApkProcessLogger", "Failed to export process logs", e);
            return false;
        }
    }
    
    /**
     * Clear all process logs
     */
    public synchronized void clearLogs() {
        activeProcesses.clear();
        completedProcesses.clear();
        if (logFile.exists()) {
            logFile.delete();
        }
    }
    
    /**
     * Flush any pending log writes
     */
    public void flush() {
        // File operations are synchronous, no need to flush
    }
    
    // === PRIVATE HELPER METHODS ===
    
    private ApkProcess getCurrentProcess() {
        return activeProcesses.isEmpty() ? null : activeProcesses.get(activeProcesses.size() - 1);
    }
    
    private String generateProcessId() {
        return "APK_" + System.currentTimeMillis() + "_" + (int)(Math.random() * 1000);
    }
    
    private synchronized void logToFile(String message) {
        try (PrintWriter writer = new PrintWriter(new FileWriter(logFile, true))) {
            writer.println(message);
        } catch (Exception e) {
            android.util.Log.e("ApkProcessLogger", "Failed to write to log file", e);
        }
    }
    
    private void loadExistingProcesses() {
        // Implementation could load previous processes from file
        // For now, start fresh each time
    }
    
    // === HELPER CLASSES ===
    
    public static class ApkProcess {
        public final String processId;
        public final String operation;
        public final String apkName;
        public final Date startTime;
        public Date endTime;
        public boolean success;
        public String result;
        public final List<ProcessStep> steps = new ArrayList<>();
        
        public ApkProcess(String processId, String operation, String apkName) {
            this.processId = processId;
            this.operation = operation;
            this.apkName = apkName;
            this.startTime = new Date();
        }
        
        public void addStep(ProcessStep step) {
            steps.add(step);
        }
        
        public void complete(boolean success, String result) {
            this.endTime = new Date();
            this.success = success;
            this.result = result;
        }
        
        public long getDuration() {
            Date end = endTime != null ? endTime : new Date();
            return end.getTime() - startTime.getTime();
        }
        
        public String getDurationString() {
            long duration = getDuration();
            long seconds = duration / 1000;
            long minutes = seconds / 60;
            seconds = seconds % 60;
            
            if (minutes > 0) {
                return minutes + "m " + seconds + "s";
            } else {
                return seconds + "s";
            }
        }
        
        public String getSummaryReport() {
            SimpleDateFormat format = new SimpleDateFormat("HH:mm:ss", Locale.getDefault());
            return String.format("[%s] %s: %s - %s (%s, %d steps)",
                               format.format(startTime),
                               processId,
                               operation,
                               apkName,
                               endTime != null ? (success ? "SUCCESS" : "FAILED") : "RUNNING",
                               steps.size());
        }
        
        public String getDetailedReport() {
            StringBuilder report = new StringBuilder();
            SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault());
            
            report.append("Process: ").append(processId).append("\n");
            report.append("Operation: ").append(operation).append("\n");
            report.append("APK: ").append(apkName).append("\n");
            report.append("Started: ").append(format.format(startTime)).append("\n");
            
            if (endTime != null) {
                report.append("Completed: ").append(format.format(endTime)).append("\n");
                report.append("Duration: ").append(getDurationString()).append("\n");
                report.append("Success: ").append(success).append("\n");
                if (result != null) {
                    report.append("Result: ").append(result).append("\n");
                }
            } else {
                report.append("Status: RUNNING\n");
            }
            
            if (!steps.isEmpty()) {
                report.append("Steps:\n");
                SimpleDateFormat stepFormat = new SimpleDateFormat("HH:mm:ss.SSS", Locale.getDefault());
                for (ProcessStep step : steps) {
                    report.append("  [").append(stepFormat.format(step.timestamp)).append("] ");
                    report.append(step.step);
                    if (step.details != null && !step.details.isEmpty()) {
                        report.append(" - ").append(step.details);
                    }
                    report.append("\n");
                }
            }
            
            return report.toString();
        }
    }
    
    public static class ProcessStep {
        public final String step;
        public final String details;
        public final Date timestamp;
        
        public ProcessStep(String step, String details) {
            this.step = step;
            this.details = details;
            this.timestamp = new Date();
        }
    }
}


--- FILE: /ModLoader/app/src/main/java/com/modloader/logging/BasicLogger.java ---
package com.modloader.logging;

public class BasicLogger {
}



--- FILE: /ModLoader/app/src/main/java/com/modloader/logging/ErrorLogger.java ---
// File: ErrorLogger.java - Specialized error and exception tracking
// Path: app/src/main/java/com/terrarialoader/logging/ErrorLogger.java

package com.modloader.logging;

import android.content.Context;
import java.io.*;
import java.text.SimpleDateFormat;
import java.util.*;

/**
 * ErrorLogger - Specialized component for tracking errors and exceptions
 * Single responsibility: Log detailed error information with stack traces
 */
public class ErrorLogger {
    private static final String ERROR_LOG_FILE = "Errors.log";
    private static final String CRASH_LOG_FILE = "Crashes.log";
    private static final int MAX_ERROR_ENTRIES = 500;
    
    private final Context context;
    private final File errorLogFile;
    private final File crashLogFile;
    private final SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS", Locale.getDefault());
    private final List<ErrorEntry> recentErrors = new ArrayList<>();
    private final Object errorLock = new Object();
    
    public ErrorLogger(Context context) {
        this.context = context;
        File logDir = new File(context.getExternalFilesDir(null), 
                             "TerrariaLoader/com.and.games505.TerrariaPaid/AppLogs");
        logDir.mkdirs();
        
        this.errorLogFile = new File(logDir, ERROR_LOG_FILE);
        this.crashLogFile = new File(logDir, CRASH_LOG_FILE);
        
        // Set up uncaught exception handler
        setupUncaughtExceptionHandler();
        
        logError("ErrorLogger initialized at " + logDir.getAbsolutePath(), null);
    }
    
    /**
     * Log an error with optional exception
     */
    public void logError(String message, Exception exception) {
        ErrorEntry entry = new ErrorEntry(message, exception);
        
        synchronized (errorLock) {
            // Add to recent errors list
            recentErrors.add(entry);
            if (recentErrors.size() > MAX_ERROR_ENTRIES) {
                recentErrors.remove(0);
            }
            
            // Write to error log file
            writeErrorToFile(entry, errorLogFile);
        }
    }
    
    /**
     * Log a critical error (system crash, major failure)
     */
    public void logCriticalError(String message, Throwable throwable) {
        ErrorEntry entry = new ErrorEntry(message, throwable, true);
        
        synchronized (errorLock) {
            recentErrors.add(entry);
            if (recentErrors.size() > MAX_ERROR_ENTRIES) {
                recentErrors.remove(0);
            }
            
            // Write to both error log and crash log
            writeErrorToFile(entry, errorLogFile);
            writeErrorToFile(entry, crashLogFile);
        }
        
        // Also log to Android system
        android.util.Log.e("TerrariaLoader", "CRITICAL: " + message, throwable);
    }
    
    /**
     * Log APK-specific errors with context
     */
    public void logApkError(String operation, String apkName, String error, Exception exception) {
        String contextualMessage = String.format("APK Operation Failed - %s on %s: %s", 
                                                operation, apkName, error);
        logError(contextualMessage, exception);
    }
    
    /**
     * Log validation errors with details
     */
    public void logValidationError(String item, String issue, String details) {
        String message = String.format("Validation Failed - %s: %s", item, issue);
        if (details != null && !details.isEmpty()) {
            message += " (" + details + ")";
        }
        logError(message, null);
    }
    
    /**
     * Get recent errors (for UI display)
     */
    public List<ErrorEntry> getRecentErrors(int maxCount) {
        synchronized (errorLock) {
            int start = Math.max(0, recentErrors.size() - maxCount);
            return new ArrayList<>(recentErrors.subList(start, recentErrors.size()));
        }
    }
    
    /**
     * Get errors by type/category
     */
    public List<ErrorEntry> getErrorsByType(String keyword) {
        List<ErrorEntry> filtered = new ArrayList<>();
        
        synchronized (errorLock) {
            for (ErrorEntry error : recentErrors) {
                if (error.message != null && error.message.toLowerCase().contains(keyword.toLowerCase())) {
                    filtered.add(error);
                }
            }
        }
        
        return filtered;
    }
    
    /**
     * Get comprehensive error report for debugging
     */
    public String getErrorReport() {
        StringBuilder report = new StringBuilder();
        
        synchronized (errorLock) {
            report.append("=== TerrariaLoader Error Report ===\n");
            report.append("Generated: ").append(dateFormat.format(new Date())).append("\n");
            report.append("Total Errors in Memory: ").append(recentErrors.size()).append("\n");
            
            ErrorStats stats = getErrorStats();
            report.append("Critical Errors: ").append(stats.criticalErrors).append("\n");
            report.append("APK-related Errors: ").append(getErrorsByType("APK").size()).append("\n");
            report.append("Validation Errors: ").append(getErrorsByType("Validation").size()).append("\n");
            report.append("\n");
            
            if (recentErrors.isEmpty()) {
                report.append("No errors recorded in memory.\n");
            } else {
                // Show last 10 errors with full details
                int start = Math.max(0, recentErrors.size() - 10);
                report.append("Recent Errors (last 10):\n");
                report.append("========================\n");
                
                for (int i = start; i < recentErrors.size(); i++) {
                    ErrorEntry error = recentErrors.get(i);
                    report.append(String.format("Error #%d:\n", i + 1));
                    report.append(error.getDetailedString()).append("\n");
                    report.append("---\n");
                }
            }
            
            // Add file statistics
            appendFileStats(report);
        }
        
        return report.toString();
    }
    
    /**
     * Get a quick error summary for status displays
     */
    public String getErrorSummary() {
        ErrorStats stats = getErrorStats();
        
        if (stats.totalMemoryErrors == 0) {
            return "No errors recorded";
        }
        
        StringBuilder summary = new StringBuilder();
        summary.append("Errors: ").append(stats.totalMemoryErrors);
        
        if (stats.criticalErrors > 0) {
            summary.append(" (").append(stats.criticalErrors).append(" critical)");
        }
        
        // Show most recent error time
        synchronized (errorLock) {
            if (!recentErrors.isEmpty()) {
                ErrorEntry lastError = recentErrors.get(recentErrors.size() - 1);
                SimpleDateFormat timeFormat = new SimpleDateFormat("HH:mm:ss", Locale.getDefault());
                summary.append(" - Last: ").append(timeFormat.format(lastError.timestamp));
            }
        }
        
        return summary.toString();
    }
    
    /**
     * Export all error logs to a file
     */
    public boolean exportErrorLogs(File outputFile) {
        synchronized (errorLock) {
            try (PrintWriter writer = new PrintWriter(new FileWriter(outputFile))) {
                writer.println("=== Complete TerrariaLoader Error Log Export ===");
                writer.println("Export Date: " + dateFormat.format(new Date()));
                writer.println("Export Location: " + outputFile.getAbsolutePath());
                writer.println();
                
                // Export error statistics
                ErrorStats stats = getErrorStats();
                writer.println("=== Error Statistics ===");
                writer.println(stats.getDetailedString());
                writer.println();
                
                // Export memory errors
                writer.println("=== Recent Errors (In Memory) ===");
                if (recentErrors.isEmpty()) {
                    writer.println("No errors in memory");
                } else {
                    for (int i = 0; i < recentErrors.size(); i++) {
                        ErrorEntry error = recentErrors.get(i);
                        writer.println("Memory Error #" + (i + 1) + ":");
                        writer.println(error.getDetailedString());
                        writer.println();
                    }
                }
                
                // Export error log file
                exportLogFile(errorLogFile, "Error Log File", writer);
                
                // Export crash log file  
                exportLogFile(crashLogFile, "Crash Log File", writer);
                
                writer.println("=== End of Error Export ===");
                return true;
                
            } catch (Exception e) {
                android.util.Log.e("ErrorLogger", "Failed to export error logs", e);
                return false;
            }
        }
    }
    
    /**
     * Clear all error logs
     */
    public void clearLogs() {
        synchronized (errorLock) {
            int clearedMemoryErrors = recentErrors.size();
            recentErrors.clear();
            
            boolean errorFileDeleted = false;
            boolean crashFileDeleted = false;
            
            if (errorLogFile.exists()) {
                errorFileDeleted = errorLogFile.delete();
            }
            
            if (crashLogFile.exists()) {
                crashFileDeleted = crashLogFile.delete();
            }
            
            // Log the clearing action
            logError(String.format("Error logs cleared - Memory: %d, ErrorFile: %s, CrashFile: %s", 
                                 clearedMemoryErrors, errorFileDeleted, crashFileDeleted), null);
        }
    }
    
    /**
     * Get detailed error statistics
     */
    public ErrorStats getErrorStats() {
        ErrorStats stats = new ErrorStats();
        
        synchronized (errorLock) {
            stats.totalMemoryErrors = recentErrors.size();
            
            for (ErrorEntry error : recentErrors) {
                if (error.isCritical) {
                    stats.criticalErrors++;
                }
                if (error.exception != null) {
                    stats.exceptionsWithStackTrace++;
                }
                
                // Categorize errors
                if (error.message != null) {
                    String msg = error.message.toLowerCase();
                    if (msg.contains("apk")) {
                        stats.apkErrors++;
                    }
                    if (msg.contains("validation")) {
                        stats.validationErrors++;
                    }
                    if (msg.contains("file") || msg.contains("io")) {
                        stats.fileErrors++;
                    }
                }
            }
            
            // Count file-based errors
            stats.totalFileErrors = countErrorsInFile(errorLogFile);
            stats.totalCrashes = countErrorsInFile(crashLogFile);
        }
        
        return stats;
    }
    
    /**
     * Check if there are any recent critical errors
     */
    public boolean hasRecentCriticalErrors(long timeWindowMs) {
        long cutoff = System.currentTimeMillis() - timeWindowMs;
        
        synchronized (errorLock) {
            for (ErrorEntry error : recentErrors) {
                if (error.isCritical && error.timestamp.getTime() > cutoff) {
                    return true;
                }
            }
        }
        
        return false;
    }
    
    /**
     * Flush any pending writes
     */
    public void flush() {
        // Error logging is synchronous, no need to flush
        // But we can verify file integrity
        synchronized (errorLock) {
            if (!errorLogFile.getParentFile().exists()) {
                errorLogFile.getParentFile().mkdirs();
            }
            if (!crashLogFile.getParentFile().exists()) {
                crashLogFile.getParentFile().mkdirs();
            }
        }
    }
    
    // === PRIVATE HELPER METHODS ===
    
    private void setupUncaughtExceptionHandler() {
        Thread.UncaughtExceptionHandler defaultHandler = Thread.getDefaultUncaughtExceptionHandler();
        
        Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> {
            // Log the crash with full context
            String crashMessage = String.format("Uncaught exception in thread '%s': %s", 
                                               thread.getName(), throwable.getMessage());
            logCriticalError(crashMessage, throwable);
            
            // Call the original handler to maintain normal crash behavior
            if (defaultHandler != null) {
                defaultHandler.uncaughtException(thread, throwable);
            }
        });
    }
    
    private void writeErrorToFile(ErrorEntry entry, File logFile) {
        try (PrintWriter writer = new PrintWriter(new FileWriter(logFile, true))) {
            writer.println("=== ERROR ENTRY ===");
            writer.println(entry.getDetailedString());
            writer.println("===================");
            writer.println();
        } catch (Exception e) {
            android.util.Log.e("ErrorLogger", "Failed to write error to file: " + e.getMessage());
            // Don't throw here to avoid infinite recursion
        }
    }
    
    private void exportLogFile(File logFile, String sectionTitle, PrintWriter writer) {
        writer.println("=== " + sectionTitle + " ===");
        
        if (!logFile.exists()) {
            writer.println("File does not exist: " + logFile.getName());
            writer.println();
            return;
        }
        
        writer.println("File: " + logFile.getName());
        writer.println("Size: " + formatFileSize(logFile.length()));
        writer.println("Last Modified: " + new Date(logFile.lastModified()));
        writer.println("Entries: " + countErrorsInFile(logFile));
        writer.println();
        
        try (BufferedReader reader = new BufferedReader(new FileReader(logFile))) {
            String line;
            while ((line = reader.readLine()) != null) {
                writer.println(line);
            }
        } catch (Exception e) {
            writer.println("Error reading file: " + e.getMessage());
        }
        
        writer.println();
    }
    
    private void appendFileStats(StringBuilder report) {
        report.append("\nFile Statistics:\n");
        report.append("================\n");
        
        if (errorLogFile.exists()) {
            report.append("Error Log: ").append(formatFileSize(errorLogFile.length()))
                  .append(" (").append(countErrorsInFile(errorLogFile)).append(" entries)\n");
            report.append("  Location: ").append(errorLogFile.getAbsolutePath()).append("\n");
        } else {
            report.append("Error Log: Not found\n");
        }
        
        if (crashLogFile.exists()) {
            report.append("Crash Log: ").append(formatFileSize(crashLogFile.length()))
                  .append(" (").append(countErrorsInFile(crashLogFile)).append(" entries)\n");
            report.append("  Location: ").append(crashLogFile.getAbsolutePath()).append("\n");
        } else {
            report.append("Crash Log: Not found\n");
        }
    }
    
    private int countErrorsInFile(File file) {
        if (!file.exists()) return 0;
        
        int errorEntries = 0;
        try (BufferedReader reader = new BufferedReader(new FileReader(file))) {
            String line;
            while ((line = reader.readLine()) != null) {
                if (line.contains("=== ERROR ENTRY ===")) {
                    errorEntries++;
                }
            }
        } catch (Exception e) {
            return -1; // Error counting
        }
        return errorEntries;
    }
    
    private String formatFileSize(long bytes) {
        if (bytes < 1024) return bytes + " B";
        if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
        return String.format("%.1f MB", bytes / (1024.0 * 1024.0));
    }
    
    // === HELPER CLASSES ===
    
    public static class ErrorEntry {
        public final String message;
        public final Throwable exception;
        public final Date timestamp;
        public final boolean isCritical;
        public final String threadName;
        public final String className;
        
        public ErrorEntry(String message, Throwable exception) {
            this(message, exception, false);
        }
        
        public ErrorEntry(String message, Throwable exception, boolean isCritical) {
            this.message = message;
            this.exception = exception;
            this.timestamp = new Date();
            this.isCritical = isCritical;
            this.threadName = Thread.currentThread().getName();
            
            // Get calling class name
            StackTraceElement[] stack = Thread.currentThread().getStackTrace();
            this.className = stack.length > 4 ? stack[4].getClassName() : "Unknown";
        }
        
        public String getDetailedString() {
            StringBuilder sb = new StringBuilder();
            SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS", Locale.getDefault());
            
            sb.append("Timestamp: ").append(format.format(timestamp)).append("\n");
            sb.append("Thread: ").append(threadName).append("\n");
            sb.append("Class: ").append(className).append("\n");
            sb.append("Critical: ").append(isCritical ? "YES" : "NO").append("\n");
            sb.append("Message: ").append(message != null ? message : "No message").append("\n");
            
            if (exception != null) {
                sb.append("Exception Type: ").append(exception.getClass().getName()).append("\n");
                sb.append("Exception Message: ").append(exception.getMessage()).append("\n");
                
                // Include cause if available
                Throwable cause = exception.getCause();
                if (cause != null) {
                    sb.append("Caused By: ").append(cause.getClass().getName())
                      .append(": ").append(cause.getMessage()).append("\n");
                }
                
                sb.append("Stack Trace:\n");
                StringWriter sw = new StringWriter();
                PrintWriter pw = new PrintWriter(sw);
                exception.printStackTrace(pw);
                sb.append(sw.toString());
            } else {
                sb.append("No exception details\n");
            }
            
            return sb.toString();
        }
        
        public String getSummaryString() {
            SimpleDateFormat format = new SimpleDateFormat("HH:mm:ss", Locale.getDefault());
            String exceptionName = exception != null ? exception.getClass().getSimpleName() : "NoException";
            String criticalMark = isCritical ? "[CRITICAL] " : "";
            
            return String.format("[%s] %s%s: %s", 
                               format.format(timestamp), 
                               criticalMark,
                               exceptionName, 
                               message != null ? truncateString(message, 100) : "No message");
        }
        
        private String truncateString(String str, int maxLength) {
            if (str.length() <= maxLength) {
                return str;
            }
            return str.substring(0, maxLength - 3) + "...";
        }
    }
    
    public static class ErrorStats {
        public int totalMemoryErrors = 0;
        public int criticalErrors = 0;
        public int exceptionsWithStackTrace = 0;
        public int totalFileErrors = 0;
        public int totalCrashes = 0;
        public int apkErrors = 0;
        public int validationErrors = 0;
        public int fileErrors = 0;
        
        @Override
        public String toString() {
            return String.format("ErrorStats[Memory: %d, Critical: %d, Exceptions: %d, File: %d, Crashes: %d]",
                               totalMemoryErrors, criticalErrors, exceptionsWithStackTrace, 
                               totalFileErrors, totalCrashes);
        }
        
        public String getDetailedString() {
            StringBuilder sb = new StringBuilder();
            sb.append("Memory Errors: ").append(totalMemoryErrors).append("\n");
            sb.append("Critical Errors: ").append(criticalErrors).append("\n");
            sb.append("Exceptions with Stack Trace: ").append(exceptionsWithStackTrace).append("\n");
            sb.append("APK-related Errors: ").append(apkErrors).append("\n");
            sb.append("Validation Errors: ").append(validationErrors).append("\n");
            sb.append("File I/O Errors: ").append(fileErrors).append("\n");
            sb.append("File-based Errors: ").append(totalFileErrors).append("\n");
            sb.append("Total Crashes: ").append(totalCrashes).append("\n");
            
            return sb.toString();
        }
        
        public boolean hasErrors() {
            return totalMemoryErrors > 0 || totalFileErrors > 0 || totalCrashes > 0;
        }
        
        public boolean hasCriticalErrors() {
            return criticalErrors > 0 || totalCrashes > 0;
        }
    }
}


--- FILE: /ModLoader/app/src/main/java/com/modloader/logging/FileLogger.java ---
// File: FileLogger.java (Complete Fixed Version) - Full logging system with all properties
// Path: /storage/emulated/0/AndroidIDEProjects/ModLoader/app/src/main/java/com/modloader/logging/FileLogger.java

package com.modloader.logging;

import android.content.Context;
import android.util.Log;
import java.io.*;
import java.text.SimpleDateFormat;
import java.util.*;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class FileLogger {
    private static final String TAG = "FileLogger";
    private static final int MAX_LOG_FILES = 5;
    private static final long MAX_LOG_SIZE = 10 * 1024 * 1024; // 10MB per log file
    private static final SimpleDateFormat DATE_FORMAT = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS", Locale.US);
    
    private Context context;
    private File logDirectory;
    private File currentLogFile;
    private FileWriter logWriter;
    private ExecutorService executorService;
    private ConcurrentLinkedQueue<String> logQueue;
    private boolean isInitialized = false;
    
    // Singleton instance
    private static volatile FileLogger instance;
    
    private FileLogger(Context context) {
        this.context = context.getApplicationContext();
        this.executorService = Executors.newSingleThreadExecutor();
        this.logQueue = new ConcurrentLinkedQueue<>();
        initialize();
    }
    
    public static FileLogger getInstance(Context context) {
        if (instance == null) {
            synchronized (FileLogger.class) {
                if (instance == null) {
                    instance = new FileLogger(context);
                }
            }
        }
        return instance;
    }
    
    private void initialize() {
        try {
            // Create log directory in app-specific external files directory
            logDirectory = new File(context.getExternalFilesDir(null), "TerrariaLoader/com.and.games505.TerrariaPaid/AppLogs");
            if (!logDirectory.exists()) {
                logDirectory.mkdirs();
            }
            
            // Create current log file
            currentLogFile = new File(logDirectory, "AppLog.txt");
            
            // Check if rotation is needed
            if (currentLogFile.exists() && currentLogFile.length() > MAX_LOG_SIZE) {
                rotateLogFiles();
            }
            
            // Initialize file writer
            logWriter = new FileWriter(currentLogFile, true);
            isInitialized = true;
            
            // Write initialization message
            writeLogEntry("INFO", "FileLogger", "FileLogger initialized - " + currentLogFile.getAbsolutePath());
            
        } catch (IOException e) {
            Log.e(TAG, "Failed to initialize FileLogger", e);
            isInitialized = false;
        }
    }
    
    public void logDebug(String tag, String message) {
        logMessage("DEBUG", tag, message);
    }
    
    public void logInfo(String tag, String message) {
        logMessage("INFO", tag, message);
    }
    
    public void logWarning(String tag, String message) {
        logMessage("WARN", tag, message);
    }
    
    public void logError(String tag, String message) {
        logMessage("ERROR", tag, message);
    }
    
    public void logError(String tag, String message, Throwable throwable) {
        StringBuilder sb = new StringBuilder(message);
        if (throwable != null) {
            sb.append("\n").append(Log.getStackTraceString(throwable));
        }
        logMessage("ERROR", tag, sb.toString());
    }
    
    public void logUser(String message) {
        logMessage("USER", "TerrariaLoader", message);
    }
    
    private void logMessage(String level, String tag, String message) {
        if (!isInitialized) {
            // Fallback to Android log
            Log.println(Log.INFO, tag, message);
            return;
        }
        
        String timestamp = DATE_FORMAT.format(new Date());
        String logEntry = String.format("[%s] %s/%s: %s", timestamp, level, tag, message);
        
        // Add to queue for async processing
        logQueue.offer(logEntry);
        
        // Process queue asynchronously
        executorService.submit(this::processLogQueue);
        
        // Also log to Android logcat based on level
        switch (level) {
            case "DEBUG":
                Log.d(tag, message);
                break;
            case "INFO":
            case "USER":
                Log.i(tag, message);
                break;
            case "WARN":
                Log.w(tag, message);
                break;
            case "ERROR":
                Log.e(tag, message);
                break;
        }
    }
    
    private void processLogQueue() {
        String logEntry;
        while ((logEntry = logQueue.poll()) != null) {
            writeLogEntry(logEntry);
        }
    }
    
    private void writeLogEntry(String level, String tag, String message) {
        String timestamp = DATE_FORMAT.format(new Date());
        String logEntry = String.format("[%s] %s/%s: %s", timestamp, level, tag, message);
        writeLogEntry(logEntry);
    }
    
    private synchronized void writeLogEntry(String logEntry) {
        if (!isInitialized || logWriter == null) {
            return;
        }
        
        try {
            logWriter.write(logEntry + "\n");
            logWriter.flush();
            
            // Check if rotation is needed
            if (currentLogFile.length() > MAX_LOG_SIZE) {
                rotateLogFiles();
            }
            
        } catch (IOException e) {
            Log.e(TAG, "Failed to write log entry", e);
            // Try to reinitialize
            closeCurrentWriter();
            initialize();
        }
    }
    
    private void rotateLogFiles() {
        try {
            closeCurrentWriter();
            
            // Move existing log files
            for (int i = MAX_LOG_FILES - 1; i >= 1; i--) {
                File oldFile = new File(logDirectory, "AppLog" + i + ".txt");
                File newFile = new File(logDirectory, "AppLog" + (i + 1) + ".txt");
                
                if (i == MAX_LOG_FILES - 1) {
                    // Delete the oldest file
                    if (oldFile.exists()) {
                        oldFile.delete();
                    }
                } else {
                    // Move file to next number
                    if (oldFile.exists()) {
                        oldFile.renameTo(newFile);
                    }
                }
            }
            
            // Move current log to AppLog1.txt
            if (currentLogFile.exists()) {
                File archivedFile = new File(logDirectory, "AppLog1.txt");
                currentLogFile.renameTo(archivedFile);
            }
            
            // Create new current log file
            currentLogFile = new File(logDirectory, "AppLog.txt");
            logWriter = new FileWriter(currentLogFile, true);
            
            writeLogEntry("INFO", "FileLogger", "Log rotation completed");
            
        } catch (IOException e) {
            Log.e(TAG, "Failed to rotate log files", e);
        }
    }
    
    private void closeCurrentWriter() {
        if (logWriter != null) {
            try {
                logWriter.close();
            } catch (IOException e) {
                Log.e(TAG, "Failed to close log writer", e);
            }
            logWriter = null;
        }
    }
    
    public String readCurrentLog() {
        StringBuilder sb = new StringBuilder();
        if (currentLogFile != null && currentLogFile.exists()) {
            try (BufferedReader reader = new BufferedReader(new FileReader(currentLogFile))) {
                String line;
                while ((line = reader.readLine()) != null) {
                    sb.append(line).append("\n");
                }
            } catch (IOException e) {
                Log.e(TAG, "Failed to read current log", e);
                sb.append("Error reading log file: ").append(e.getMessage());
            }
        } else {
            sb.append("No log file found");
        }
        return sb.toString();
    }
    
    public String readAllLogs() {
        StringBuilder sb = new StringBuilder();
        
        // Read current log
        sb.append("=== CURRENT LOG ===\n");
        sb.append(readCurrentLog());
        
        // Read archived logs
        for (int i = 1; i <= MAX_LOG_FILES; i++) {
            File archivedLog = new File(logDirectory, "AppLog" + i + ".txt");
            if (archivedLog.exists()) {
                sb.append("\n=== ARCHIVED LOG ").append(i).append(" ===\n");
                try (BufferedReader reader = new BufferedReader(new FileReader(archivedLog))) {
                    String line;
                    while ((line = reader.readLine()) != null) {
                        sb.append(line).append("\n");
                    }
                } catch (IOException e) {
                    Log.e(TAG, "Failed to read archived log " + i, e);
                    sb.append("Error reading archived log ").append(i).append(": ").append(e.getMessage()).append("\n");
                }
            }
        }
        
        return sb.toString();
    }
    
    public boolean exportLogs(File exportFile) {
        try (FileWriter writer = new FileWriter(exportFile)) {
            writer.write("=== TerrariaLoader Export ===\n");
            writer.write("Export Date: " + new Date().toString() + "\n");
            writer.write("Log Directory: " + logDirectory.getAbsolutePath() + "\n\n");
            writer.write(readAllLogs());
            return true;
        } catch (IOException e) {
            Log.e(TAG, "Failed to export logs", e);
            return false;
        }
    }
    
    public void clearLogs() {
        try {
            closeCurrentWriter();
            
            // Delete all log files
            if (currentLogFile != null && currentLogFile.exists()) {
                currentLogFile.delete();
            }
            
            for (int i = 1; i <= MAX_LOG_FILES; i++) {
                File archivedLog = new File(logDirectory, "AppLog" + i + ".txt");
                if (archivedLog.exists()) {
                    archivedLog.delete();
                }
            }
            
            // Reinitialize
            initialize();
            
        } catch (Exception e) {
            Log.e(TAG, "Failed to clear logs", e);
        }
    }
    
    // FIXED: Complete LogStats class with all required properties
    public static class LogStats {
        public int totalFiles = 0;
        public long totalSize = 0;        // FIXED: Added missing field
        public int totalLines = 0;        // FIXED: Added missing field
        public String oldestEntry = "";   // FIXED: Added missing field
        public String newestEntry = "";   // FIXED: Added missing field
        public String currentLogFile = "";
        public List<String> archivedFiles = new ArrayList<>();
        
        public LogStats() {
            // Initialize any default values if needed
        }
        
        @Override
        public String toString() {
            return String.format("LogStats{totalFiles=%d, totalSize=%d bytes, totalLines=%d, oldest='%s', newest='%s'}",
                    totalFiles, totalSize, totalLines, oldestEntry, newestEntry);
        }
        
        public String getDetailedReport() {
            StringBuilder sb = new StringBuilder();
            sb.append("=== Log Statistics ===\n");
            sb.append("Total Files: ").append(totalFiles).append("\n");
            sb.append("Total Size: ").append(formatFileSize(totalSize)).append("\n");
            sb.append("Total Lines: ").append(totalLines).append("\n");
            sb.append("Oldest Entry: ").append(oldestEntry).append("\n");
            sb.append("Newest Entry: ").append(newestEntry).append("\n");
            sb.append("Current Log: ").append(currentLogFile).append("\n");
            if (!archivedFiles.isEmpty()) {
                sb.append("Archived Files:\n");
                for (String archived : archivedFiles) {
                    sb.append("  - ").append(archived).append("\n");
                }
            }
            return sb.toString();
        }
        
        private String formatFileSize(long bytes) {
            if (bytes < 1024) return bytes + " B";
            if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
            if (bytes < 1024 * 1024 * 1024) return String.format("%.1f MB", bytes / (1024.0 * 1024.0));
            return String.format("%.1f GB", bytes / (1024.0 * 1024.0 * 1024.0));
        }
    }
    
    // FIXED: Complete getLogStats method with all property calculations
    public LogStats getLogStats() {
        LogStats stats = new LogStats();
        
        if (logDirectory != null && logDirectory.exists()) {
            File[] logFiles = logDirectory.listFiles((dir, name) -> name.endsWith(".txt"));
            if (logFiles != null) {
                stats.totalFiles = logFiles.length;
                
                // Calculate total size and collect file info
                for (File logFile : logFiles) {
                    stats.totalSize += logFile.length();  // FIXED: Now totalSize exists
                    stats.archivedFiles.add(logFile.getName());
                }
                
                // Set current log file path
                if (currentLogFile != null) {
                    stats.currentLogFile = currentLogFile.getName();
                }
                
                // Calculate total lines and find oldest/newest entries
                String firstLine = null;
                String lastLine = null;
                
                // Read through all log files to count lines and find date range
                for (File logFile : logFiles) {
                    try (BufferedReader reader = new BufferedReader(new FileReader(logFile))) {
                        String line;
                        boolean isFirstLineOfAllFiles = (firstLine == null);
                        
                        while ((line = reader.readLine()) != null) {
                            if (!line.trim().isEmpty()) {
                                stats.totalLines++;  // FIXED: Now totalLines exists
                                
                                // Capture first line ever read
                                if (isFirstLineOfAllFiles && firstLine == null) {
                                    firstLine = line;
                                    isFirstLineOfAllFiles = false;
                                }
                                
                                // Always update last line
                                lastLine = line;
                            }
                        }
                    } catch (IOException e) {
                        Log.e(TAG, "Error reading log file for stats: " + logFile.getName(), e);
                    }
                }
                
                // FIXED: Extract timestamps and set oldest/newest entries
                stats.oldestEntry = firstLine != null ? extractTimeFromLogLine(firstLine) : "";  // FIXED: Now oldestEntry exists
                stats.newestEntry = lastLine != null ? extractTimeFromLogLine(lastLine) : "";    // FIXED: Now newestEntry exists
            }
        }
        
        return stats;
    }
    
    // Helper method to extract timestamp from log line
    private String extractTimeFromLogLine(String logLine) {
        if (logLine == null || logLine.isEmpty()) {
            return "";
        }
        
        try {
            // Look for timestamp pattern [YYYY-MM-DD HH:MM:SS.mmm]
            int start = logLine.indexOf('[');
            int end = logLine.indexOf(']');
            if (start >= 0 && end > start) {
                return logLine.substring(start + 1, end);
            }
        } catch (Exception e) {
            Log.e(TAG, "Error extracting timestamp from log line", e);
        }
        
        // Fallback: return first 23 characters if they look like a timestamp
        if (logLine.length() >= 23) {
            return logLine.substring(0, 23);
        }
        
        return logLine.length() > 50 ? logLine.substring(0, 50) + "..." : logLine;
    }
    
    public File getLogDirectory() {
        return logDirectory;
    }
    
    public File getCurrentLogFile() {
        return currentLogFile;
    }
    
    public boolean isInitialized() {
        return isInitialized;
    }
    
    public void shutdown() {
        if (executorService != null && !executorService.isShutdown()) {
            executorService.shutdown();
        }
        closeCurrentWriter();
    }
    
    // Cleanup method for proper resource management
    @Override
    protected void finalize() throws Throwable {
        try {
            shutdown();
        } finally {
            super.finalize();
        }
    }
}


--- FILE: /ModLoader/app/src/main/java/com/modloader/logging/LogAnalytics.java ---
package com.modloader.logging;

import android.content.Context;
import org.apache.logging.log4j.Level;
import com.modloader.util.LogUtils;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

/**
 * LogAnalytics class to process and generate insights from log entries.
 * This class collects log data and provides a high-level summary.
 */
public class LogAnalytics {

    private final Map<String, List<LogEntry>> logsByCategory = new ConcurrentHashMap<>();
    private final AtomicLong errorCount = new AtomicLong(0);
    private final AtomicLong totalLogsCount = new AtomicLong(0);
    private final ConcurrentHashMap<String, Long> categoryCounts = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, Long> operationDurations = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, AtomicLong> operationCounts = new ConcurrentHashMap<>();
    
    public LogAnalytics(Context context) {
        // Initialization can happen here if needed, but the class is stateless for now.
    }

    /**
     * Records a log entry for analysis.
     *
     * @param logUtilsEntry The log entry from LogUtils.
     */
    public void recordLogEntry(LogUtils.LogEntry logUtilsEntry) {
        if (logUtilsEntry == null) {
            return;
        }

        // Increment total log count
        totalLogsCount.incrementAndGet();

        // Increment category count
        categoryCounts.merge(logUtilsEntry.getCategory(), 1L, Long::sum);

        // Check for error level logs
        if (logUtilsEntry.getLevel() == Level.ERROR) {
            errorCount.incrementAndGet();
        }

        // Add to the list of logs for the category
        LogEntry entry = new LogEntry(
            logUtilsEntry.getCategory(),
            logUtilsEntry.getLevel(),
            logUtilsEntry.getMessage(),
            logUtilsEntry.getCorrelationId(),
            logUtilsEntry.getTimestamp(),
            logUtilsEntry.getThrowable()
        );
        logsByCategory.computeIfAbsent(entry.getCategory(), k -> new ArrayList<>()).add(entry);
    }
    
    /**
     * Generates a comprehensive report with key log metrics.
     *
     * @return A map containing analytics data.
     */
    public Map<String, Object> generateReport() {
        Map<String, Object> report = new HashMap<>();

        long total = totalLogsCount.get();
        long errors = errorCount.get();
        double errorRate = (total > 0) ? ((double) errors / total) * 100 : 0.0;
        
        report.put("totalLogs", total);
        report.put("totalErrors", errors);
        report.put("errorRate", errorRate);
        report.put("categoryCounts", categoryCounts);

        String mostActiveCategory = "N/A";
        long maxCount = 0;
        for (Map.Entry<String, Long> entry : categoryCounts.entrySet()) {
            if (entry.getValue() > maxCount) {
                maxCount = entry.getValue();
                mostActiveCategory = entry.getKey();
            }
        }
        report.put("mostActiveCategory", mostActiveCategory);

        // Calculate average performance metrics
        long totalDuration = 0;
        long totalOps = 0;
        for (Map.Entry<String, Long> entry : operationDurations.entrySet()) {
            String opName = entry.getKey();
            long duration = entry.getValue();
            long count = operationCounts.getOrDefault(opName, new AtomicLong(0)).get();
            if (count > 0) {
                totalDuration += duration;
                totalOps += count;
            }
        }
        double avgOpTime = totalOps > 0 ? (double) totalDuration / totalOps : 0.0;
        report.put("avgOperationTime", avgOpTime);
        report.put("operationDurations", operationDurations);

        return report;
    }

    /**
     * Tracks a user action for UX insights.
     *
     * @param action The user action to track.
     */
    public void trackUserAction(String action) {
        // Future implementation to log user-specific events to a separate stream
        // for more detailed UX analytics.
    }

    /**
     * Internal class to represent a stripped-down log entry for analytics purposes.
     */
    private static class LogEntry {
        private final String category;
        private final Level level; // Fixed type to org.apache.logging.log4j.Level
        private final String message;
        private final String correlationId;
        private final long timestamp;
        private final Throwable throwable;

        LogEntry(String category, Level level, String message, String correlationId, long timestamp, Throwable throwable) {
            this.category = category;
            this.level = level;
            this.message = message;
            this.correlationId = correlationId;
            this.timestamp = timestamp;
            this.throwable = throwable;
        }

        // Getters
        public String getCategory() { return category; }
        public Level getLevel() { return level; }
        public String getMessage() { return message; }
        public String getCorrelationId() { return correlationId; }
        public long getTimestamp() { return timestamp; }
        public Throwable getThrowable() { return throwable; }
    }
}




--- FILE: /ModLoader/app/src/main/java/com/modloader/logging/LogCorrelation.java ---
// =========================
// LogCorrelation.java
// =========================
// File: LogCorrelation.java - Complete Implementation
// Path: /app/src/main/java/com/modloader/logging/LogCorrelation.java

package com.modloader.logging;

import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Log Correlation ID Manager for tracking related operations
 */
public class LogCorrelation {
    private final Map<String, OperationContext> activeOperations = new ConcurrentHashMap<>();
    private final ThreadLocal<String> currentCorrelationId = new ThreadLocal<>();
    
    /**
     * Generate a new correlation ID with prefix
     */
    public String generateCorrelationId(String prefix) {
        String correlationId = prefix + "_" + UUID.randomUUID().toString().substring(0, 8);
        currentCorrelationId.set(correlationId);
        return correlationId;
    }
    
    /**
     * Get current correlation ID or generate new one
     */
    public String getCurrentOrGenerate(String prefix) {
        String current = currentCorrelationId.get();
        if (current == null) {
            current = generateCorrelationId(prefix);
        }
        return current;
    }
    
    /**
     * Start a correlated operation
     */
    public String startOperation(String operationName) {
        String correlationId = generateCorrelationId(operationName.toUpperCase());
        activeOperations.put(correlationId, new OperationContext(operationName, System.currentTimeMillis()));
        return correlationId;
    }
    
    /**
     * End a correlated operation
     */
    public void endOperation(String correlationId, boolean success) {
        OperationContext context = activeOperations.remove(correlationId);
        if (context != null) {
            context.endTime = System.currentTimeMillis();
            context.success = success;
        }
        
        // Clear from thread local if it matches
        if (correlationId.equals(currentCorrelationId.get())) {
            currentCorrelationId.remove();
        }
    }
    
    /**
     * Set current correlation ID for thread
     */
    public void setCurrentCorrelationId(String correlationId) {
        currentCorrelationId.set(correlationId);
    }
    
    /**
     * Get current correlation ID for thread
     */
    public String getCurrentCorrelationId() {
        return currentCorrelationId.get();
    }
    
    /**
     * Clear correlation ID for current thread
     */
    public void clearCurrentCorrelationId() {
        currentCorrelationId.remove();
    }
    
    /**
     * Get active operations count
     */
    public int getActiveOperationsCount() {
        return activeOperations.size();
    }
    
    /**
     * Operation context for tracking
     */
    private static class OperationContext {
        final String operationName;
        final long startTime;
        long endTime;
        boolean success;
        
        OperationContext(String operationName, long startTime) {
            this.operationName = operationName;
            this.startTime = startTime;
        }
        
        long getDuration() {
            return endTime > 0 ? endTime - startTime : System.currentTimeMillis() - startTime;
        }
    }
}


--- FILE: /ModLoader/app/src/main/java/com/modloader/logging/LogExporter.java ---
package com.terrarialoader.logging;

public class LogExporter {
}



--- FILE: /ModLoader/app/src/main/java/com/modloader/logging/PerformanceMetrics.java ---
package com.modloader.logging;

import android.content.Context;
import android.os.Debug;
import android.os.Process;
import android.app.ActivityManager;
import android.content.pm.PackageManager;
import android.content.pm.ApplicationInfo;
import android.os.Build;
import android.os.Environment;
import android.os.StatFs;
import android.util.Log;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileReader;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.text.SimpleDateFormat;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import java.util.regex.Pattern;

/**
 * Enhanced Performance Metrics with comprehensive system monitoring
 * Tracks memory, CPU, storage, network, and custom operation metrics.
 * This version is compatible with the Android platform.
 */
public class PerformanceMetrics {
    private static final String TAG = "PerformanceMetrics";

    // Context and configuration
    private final Context context;
    private final boolean enableDetailedMetrics;
    private final long startTime;

    // Metrics storage
    private final Map<String, OperationMetrics> operationMetrics = new ConcurrentHashMap<>();
    private final Map<String, List<Long>> recentOperationTimes = new ConcurrentHashMap<>();
    private final Map<String, AtomicLong> counters = new ConcurrentHashMap<>();

    // System metrics
    private final SystemMetrics systemMetrics;
    private final MemoryTracker memoryTracker;
    private final CpuTracker cpuTracker;
    private final StorageTracker storageTracker;

    // Thresholds for alerts
    private static final long SLOW_OPERATION_THRESHOLD_MS = 5000; // 5 seconds
    private static final double HIGH_MEMORY_THRESHOLD = 0.85; // 85% of available memory
    private static final double HIGH_CPU_THRESHOLD = 0.80; // 80% CPU usage

    // Date formatter for timestamps
    private static final SimpleDateFormat timestampFormat =
            new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS", Locale.getDefault());

    public PerformanceMetrics() {
        this(null, true);
    }

    public PerformanceMetrics(Context context) {
        this(context, true);
    }

    public PerformanceMetrics(Context context, boolean enableDetailedMetrics) {
        this.context = context;
        this.enableDetailedMetrics = enableDetailedMetrics;
        this.startTime = System.currentTimeMillis();

        // Initialize system monitoring components
        this.systemMetrics = new SystemMetrics();
        this.memoryTracker = new MemoryTracker();
        this.cpuTracker = new CpuTracker();
        this.storageTracker = new StorageTracker();

        // Initialize counters
        counters.put("operations_total", new AtomicLong(0));
        counters.put("operations_slow", new AtomicLong(0));
        counters.put("operations_failed", new AtomicLong(0));
        counters.put("memory_warnings", new AtomicLong(0));
        counters.put("cpu_warnings", new AtomicLong(0));

        Log.d(TAG, "PerformanceMetrics initialized with detailed monitoring: " + enableDetailedMetrics);
    }

    // =========================
    // OPERATION TRACKING METHODS
    // =========================

    /**
     * Records operation performance metrics.
     * @param operation The name of the operation.
     * @param durationMs The duration of the operation in milliseconds.
     */
    public void recordOperation(String operation, long durationMs) {
        recordOperation(operation, durationMs, null);
    }

    /**
     * Records operation with additional custom metrics.
     * @param operation The name of the operation.
     * @param durationMs The duration of the operation in milliseconds.
     * @param customMetrics A map of custom key-value metrics.
     */
    public void recordOperation(String operation, long durationMs, Map<String, Object> customMetrics) {
        if (operation == null) return;

        counters.get("operations_total").incrementAndGet();

        // Update operation metrics
        operationMetrics.computeIfAbsent(operation, k -> new OperationMetrics(k))
                .recordExecution(durationMs, customMetrics);

        // Track recent operation times for trend analysis
        recentOperationTimes.computeIfAbsent(operation, k -> new ArrayList<>())
                .add(durationMs);

        // Clean old entries (keep only last 100)
        List<Long> recentTimes = recentOperationTimes.get(operation);
        if (recentTimes.size() > 100) {
            recentTimes.subList(0, recentTimes.size() - 100).clear();
        }

        // Check for slow operations
        if (durationMs > SLOW_OPERATION_THRESHOLD_MS) {
            counters.get("operations_slow").incrementAndGet();
            Log.w(TAG, String.format("Slow operation detected: %s took %dms", operation, durationMs));
        }

        // Collect system metrics if enabled
        if (enableDetailedMetrics) {
            collectSystemMetrics();
        }

        Log.d(TAG, String.format("Recorded operation: %s (%dms)", operation, durationMs));
    }

    /**
     * Records a failed operation.
     * @param operation The name of the operation.
     * @param error A descriptive error message.
     */
    public void recordFailedOperation(String operation, String error) {
        recordFailedOperation(operation, error, null);
    }

    /**
     * Records a failed operation with additional context.
     * @param operation The name of the operation.
     * @param error A descriptive error message.
     * @param throwable The Throwable that caused the failure.
     */
    public void recordFailedOperation(String operation, String error, Throwable throwable) {
        if (operation == null) return;

        counters.get("operations_failed").incrementAndGet();

        operationMetrics.computeIfAbsent(operation, k -> new OperationMetrics(k))
                .recordFailure(error, throwable);

        Log.w(TAG, String.format("Failed operation recorded: %s - %s", operation, error), throwable);
    }

    /**
     * Starts timing an operation and returns the start time for manual timing.
     * @param operation The name of the operation.
     * @return The start time in milliseconds.
     */
    public long startOperation(String operation) {
        long startTime = System.currentTimeMillis();
        Log.d(TAG, "Started timing operation: " + operation);
        return startTime;
    }

    /**
     * Ends timing an operation using the start time from startOperation.
     * @param operation The name of the operation.
     * @param startTime The start time from startOperation.
     */
    public void endOperation(String operation, long startTime) {
        long duration = System.currentTimeMillis() - startTime;
        recordOperation(operation, duration);
    }

    // =========================
    // SYSTEM METRICS COLLECTION
    // =========================

    /**
     * Collects comprehensive system metrics.
     */
    private void collectSystemMetrics() {
        try {
            // Memory metrics
            memoryTracker.updateMetrics();

            // CPU metrics
            cpuTracker.updateMetrics();

            // Storage metrics
            if (context != null) {
                storageTracker.updateMetrics(context);
            }

            // Check for alerts
            checkSystemAlerts();

        } catch (Exception e) {
            Log.e(TAG, "Error collecting system metrics", e);
        }
    }

    /**
     * Checks system metrics against thresholds and generates alerts.
     */
    private void checkSystemAlerts() {
        // Memory alert
        double memoryUsage = memoryTracker.getMemoryUsagePercentage();
        if (memoryUsage > HIGH_MEMORY_THRESHOLD) {
            counters.get("memory_warnings").incrementAndGet();
            Log.w(TAG, String.format("High memory usage detected: %.1f%%", memoryUsage * 100));
        }

        // CPU alert
        double cpuUsage = cpuTracker.getCpuUsagePercentage();
        if (cpuUsage > HIGH_CPU_THRESHOLD) {
            counters.get("cpu_warnings").incrementAndGet();
            Log.w(TAG, String.format("High CPU usage detected: %.1f%%", cpuUsage * 100));
        }
    }

    /**
     * Gets current memory usage in bytes.
     * @return The current memory usage in bytes.
     */
    public long getCurrentMemoryUsage() {
        return memoryTracker.getCurrentMemoryUsageBytes();
    }

    /**
     * Gets current memory usage as a percentage (0.0 to 1.0).
     * @return The memory usage percentage.
     */
    public double getCurrentMemoryUsagePercentage() {
        return memoryTracker.getMemoryUsagePercentage();
    }

    /**
     * Gets current CPU usage percentage (0.0 to 1.0).
     * @return The CPU usage percentage.
     */
    public double getCurrentCpuUsage() {
        cpuTracker.updateMetrics();
        return cpuTracker.getCpuUsagePercentage();
    }

    /**
     * Gets available storage space in bytes.
     * @return The available storage space in bytes.
     */
    public long getAvailableStorage() {
        if (context != null) {
            storageTracker.updateMetrics(context);
            return storageTracker.getAvailableStorageBytes();
        }
        return -1;
    }

    // =========================
    // METRICS REPORTING
    // =========================

    /**
     * Generates a comprehensive performance report.
     * @return A formatted string of the performance report.
     */
    public String generateReport() {
        StringBuilder report = new StringBuilder();

        report.append("=== Performance Metrics Report ===\n");
        report.append("Generated: ").append(timestampFormat.format(new Date())).append("\n");
        report.append("Uptime: ").append(formatDuration(System.currentTimeMillis() - startTime)).append("\n\n");

        // Overall statistics
        report.append("=== Overall Statistics ===\n");
        report.append("Total Operations: ").append(counters.get("operations_total").get()).append("\n");
        report.append("Failed Operations: ").append(counters.get("operations_failed").get()).append("\n");
        report.append("Slow Operations: ").append(counters.get("operations_slow").get()).append("\n");
        report.append("Memory Warnings: ").append(counters.get("memory_warnings").get()).append("\n");
        report.append("CPU Warnings: ").append(counters.get("cpu_warnings").get()).append("\n\n");

        // System metrics
        report.append("=== Current System Metrics ===\n");
        collectSystemMetrics(); // Refresh current metrics
        report.append(systemMetrics.getSystemInfoReport()).append("\n");
        report.append(memoryTracker.getMemoryReport()).append("\n");
        report.append(cpuTracker.getCpuReport()).append("\n");
        if (context != null) {
            report.append(storageTracker.getStorageReport()).append("\n");
        }

        // Operation-specific metrics
        report.append("=== Operation Performance ===\n");
        for (Map.Entry<String, OperationMetrics> entry : operationMetrics.entrySet()) {
            report.append(entry.getValue().getReport()).append("\n");
        }

        return report.toString();
    }

    /**
     * Gets a lightweight performance summary as a Map.
     * @return A map containing a summary of performance data.
     */
    public Map<String, Object> getPerformanceSummary() {
        Map<String, Object> summary = new HashMap<>();

        // Basic counters
        summary.put("totalOperations", counters.get("operations_total").get());
        summary.put("failedOperations", counters.get("operations_failed").get());
        summary.put("slowOperations", counters.get("operations_slow").get());
        summary.put("uptimeMs", System.currentTimeMillis() - startTime);

        // Current system state
        collectSystemMetrics();
        summary.put("memoryUsagePercent", memoryTracker.getMemoryUsagePercentage());
        summary.put("memoryUsageBytes", memoryTracker.getCurrentMemoryUsageBytes());
        summary.put("cpuUsagePercent", cpuTracker.getCpuUsagePercentage());

        if (context != null) {
            summary.put("availableStorageBytes", storageTracker.getAvailableStorageBytes());
        }

        // Average operation times
        Map<String, Double> avgTimes = new HashMap<>();
        for (Map.Entry<String, OperationMetrics> entry : operationMetrics.entrySet()) {
            avgTimes.put(entry.getKey(), entry.getValue().getAverageTime());
        }
        summary.put("averageOperationTimes", avgTimes);

        return summary;
    }

    /**
     * Exports metrics to JSON format.
     * @return A JSON string of the performance data.
     */
    public String exportToJson() {
        Map<String, Object> data = getPerformanceSummary();
        return formatAsJson(data);
    }

    // =========================
    // UTILITY METHODS
    // =========================

    private String formatDuration(long millis) {
        long seconds = millis / 1000;
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

    private String formatAsJson(Map<String, Object> data) {
        StringBuilder json = new StringBuilder();
        json.append("{");

        boolean first = true;
        for (Map.Entry<String, Object> entry : data.entrySet()) {
            if (!first) json.append(",");
            json.append("\"").append(entry.getKey()).append("\":");

            Object value = entry.getValue();
            if (value instanceof String) {
                json.append("\"").append(value).append("\"");
            } else if (value instanceof Map) {
                json.append(formatAsJson((Map<String, Object>) value));
            } else {
                json.append(value);
            }
            first = false;
        }

        json.append("}");
        return json.toString();
    }

    // =========================
    // INNER CLASSES
    // =========================

    /**
     * Tracks metrics for individual operations.
     */
    private static class OperationMetrics {
        private final String operationName;
        private final AtomicLong executionCount = new AtomicLong(0);
        private final AtomicLong totalTime = new AtomicLong(0);
        private final AtomicLong failureCount = new AtomicLong(0);
        private long minTime = Long.MAX_VALUE;
        private long maxTime = Long.MIN_VALUE;
        private final List<String> recentErrors = new ArrayList<>();
        private final long createdTime;

        public OperationMetrics(String operationName) {
            this.operationName = operationName;
            this.createdTime = System.currentTimeMillis();
        }

        public void recordExecution(long durationMs, Map<String, Object> customMetrics) {
            executionCount.incrementAndGet();
            totalTime.addAndGet(durationMs);

            synchronized (this) {
                if (durationMs < minTime) minTime = durationMs;
                if (durationMs > maxTime) maxTime = durationMs;
            }
        }

        public void recordFailure(String error, Throwable throwable) {
            failureCount.incrementAndGet();

            synchronized (recentErrors) {
                recentErrors.add(error);
                if (recentErrors.size() > 10) {
                    recentErrors.remove(0);
                }
            }
        }

        public double getAverageTime() {
            long count = executionCount.get();
            return count > 0 ? (double) totalTime.get() / count : 0.0;
        }

        public String getReport() {
            long count = executionCount.get();
            long failures = failureCount.get();
            double successRate = count > 0 ? ((double) (count - failures) / count) * 100 : 0;

            StringBuilder report = new StringBuilder();
            report.append("Operation: ").append(operationName).append("\n");
            report.append("  Executions: ").append(count).append("\n");
            report.append("  Failures: ").append(failures).append("\n");
            report.append("  Success Rate: ").append(String.format("%.1f%%", successRate)).append("\n");

            if (count > 0) {
                report.append("  Average Time: ").append(String.format("%.1fms", getAverageTime())).append("\n");
                report.append("  Min Time: ").append(minTime).append("ms\n");
                report.append("  Max Time: ").append(maxTime).append("ms\n");
            }

            if (!recentErrors.isEmpty()) {
                report.append("  Recent Errors: ").append(recentErrors.toString()).append("\n");
            }

            return report.toString();
        }
    }

    /**
     * System information tracker.
     */
    private static class SystemMetrics {
        public String getSystemInfoReport() {
            StringBuilder info = new StringBuilder();
            info.append("Android Version: ").append(Build.VERSION.RELEASE).append(" (API ").append(Build.VERSION.SDK_INT).append(")\n");
            info.append("Device: ").append(Build.MANUFACTURER).append(" ").append(Build.MODEL).append("\n");
            info.append("ABI: ").append(Build.SUPPORTED_ABIS[0]).append("\n");

            // Java/Runtime info
            Runtime runtime = Runtime.getRuntime();
            info.append("Available Processors: ").append(runtime.availableProcessors()).append("\n");

            return info.toString();
        }
    }

    /**
     * Memory usage tracker.
     */
    private class MemoryTracker {
        private long currentMemoryBytes = 0;
        private double memoryUsagePercentage = 0.0;
        private long maxMemoryBytes = 0;
        private long freeMemoryBytes = 0;

        public void updateMetrics() {
            try {
                // Use Runtime class, which is available in Android
                Runtime runtime = Runtime.getRuntime();
                maxMemoryBytes = runtime.maxMemory();
                freeMemoryBytes = runtime.freeMemory();
                long totalMemory = runtime.totalMemory();
                currentMemoryBytes = totalMemory - freeMemoryBytes;

                memoryUsagePercentage = (double) currentMemoryBytes / maxMemoryBytes;

                // Also get Android-specific memory info
                if (context != null) {
                    ActivityManager activityManager = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
                    if (activityManager != null) {
                        ActivityManager.MemoryInfo memoryInfo = new ActivityManager.MemoryInfo();
                        activityManager.getMemoryInfo(memoryInfo);
                    }
                }

            } catch (Exception e) {
                Log.e(TAG, "Error updating memory metrics", e);
            }
        }

        public long getCurrentMemoryUsageBytes() {
            return currentMemoryBytes;
        }

        public double getMemoryUsagePercentage() {
            return memoryUsagePercentage;
        }

        public String getMemoryReport() {
            return String.format("Memory Usage: %.1f%% (%s / %s)",
                    memoryUsagePercentage * 100,
                    formatBytes(currentMemoryBytes),
                    formatBytes(maxMemoryBytes));
        }

        private String formatBytes(long bytes) {
            if (bytes < 1024) return bytes + " B";
            if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
            if (bytes < 1024 * 1024 * 1024) return String.format("%.1f MB", bytes / (1024.0 * 1024));
            return String.format("%.1f GB", bytes / (1024.0 * 1024 * 1024));
        }
    }

    /**
     * CPU usage tracker.
     */
    private static class CpuTracker {
        private double cpuUsagePercentage = 0.0;
        private long lastCpuTime = 0;
        private long lastAppTime = 0;

        public void updateMetrics() {
            try {
                // Reading /proc/stat and /proc/<pid>/stat for CPU usage is the standard
                // and most reliable way on Android. This is a basic implementation.
                long[] cpuTimes = getCpuTimes();
                long[] appCpuTimes = getAppCpuTimes();

                if (lastCpuTime == 0 || lastAppTime == 0) {
                    lastCpuTime = cpuTimes[0];
                    lastAppTime = appCpuTimes[0];
                    return;
                }

                long cpuDelta = cpuTimes[0] - lastCpuTime;
                long appDelta = appCpuTimes[0] - lastAppTime;

                if (cpuDelta > 0) {
                    cpuUsagePercentage = (double) appDelta / cpuDelta;
                } else {
                    cpuUsagePercentage = 0.0;
                }

                lastCpuTime = cpuTimes[0];
                lastAppTime = appCpuTimes[0];

            } catch (Exception e) {
                Log.e(TAG, "Error updating CPU metrics", e);
            }
        }

        private long[] getCpuTimes() throws IOException {
            RandomAccessFile reader = new RandomAccessFile("/proc/stat", "r");
            String load = reader.readLine();
            reader.close();

            String[] toks = load.split(" ");

            long idle = Long.parseLong(toks[4]);
            long total = 0;
            for (String s : toks) {
                try {
                    total += Long.parseLong(s);
                } catch (NumberFormatException e) {
                    // Ignore non-numeric values
                }
            }
            return new long[]{total, idle};
        }

        private long[] getAppCpuTimes() throws IOException {
            int pid = Process.myPid();
            RandomAccessFile reader = new RandomAccessFile("/proc/" + pid + "/stat", "r");
            String load = reader.readLine();
            reader.close();

            String[] toks = load.split(" ");
            long utime = Long.parseLong(toks[13]);
            long stime = Long.parseLong(toks[14]);
            return new long[]{utime + stime};
        }

        public double getCpuUsagePercentage() {
            return cpuUsagePercentage;
        }

        public String getCpuReport() {
            return String.format("CPU Usage: %.1f%%", cpuUsagePercentage * 100);
        }
    }

    /**
     * Storage space tracker.
     */
    private static class StorageTracker {
        private long availableStorageBytes = 0;
        private long totalStorageBytes = 0;

        public void updateMetrics(Context context) {
            try {
                File externalFilesDir = context.getExternalFilesDir(null);
                if (externalFilesDir != null) {
                    StatFs stat = new StatFs(externalFilesDir.getPath());

                    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR2) {
                        availableStorageBytes = stat.getAvailableBytes();
                        totalStorageBytes = stat.getTotalBytes();
                    } else {
                        // Fallback for older Android versions
                        long blockSize = stat.getBlockSize();
                        availableStorageBytes = stat.getAvailableBlocks() * blockSize;
                        totalStorageBytes = stat.getBlockCount() * blockSize;
                    }
                }
            } catch (Exception e) {
                Log.e(TAG, "Error updating storage metrics", e);
            }
        }

        public long getAvailableStorageBytes() {
            return availableStorageBytes;
        }

        public String getStorageReport() {
            double usagePercentage = totalStorageBytes > 0 ?
                    ((double) (totalStorageBytes - availableStorageBytes) / totalStorageBytes) * 100 : 0;

            return String.format("Storage: %.1f%% used (%s available / %s total)",
                    usagePercentage,
                    formatBytes(availableStorageBytes),
                    formatBytes(totalStorageBytes));
        }

        private String formatBytes(long bytes) {
            if (bytes < 1024) return bytes + " B";
            if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
            if (bytes < 1024 * 1024 * 1024) return String.format("%.1f MB", bytes / (1024.0 * 1024));
            return String.format("%.1f GB", bytes / (1024.0 * 1024 * 1024));
        }
    }

    /**
     * Resets all metrics.
     */
    public void reset() {
        operationMetrics.clear();
        recentOperationTimes.clear();

        for (AtomicLong counter : counters.values()) {
            counter.set(0);
        }

        Log.d(TAG, "Performance metrics reset");
    }

    /**
     * Clean shutdown.
     */
    public void shutdown() {
        Log.d(TAG, "PerformanceMetrics shutting down");
        Log.i(TAG, "Final Performance Summary:\n" + generateReport());
    }
}




--- FILE: /ModLoader/app/src/main/java/com/modloader/ui/BaseActivity.java ---
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


--- FILE: /ModLoader/app/src/main/java/com/modloader/ui/DllModActivity.java ---
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


--- FILE: /ModLoader/app/src/main/java/com/modloader/ui/DllModController.java ---
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


--- FILE: /ModLoader/app/src/main/java/com/modloader/ui/InstructionsActivity.java ---
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


--- FILE: /ModLoader/app/src/main/java/com/modloader/ui/LogCategoryAdapter.java ---
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


--- FILE: /ModLoader/app/src/main/java/com/modloader/ui/LogEntry.java ---
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


--- FILE: /ModLoader/app/src/main/java/com/modloader/ui/LogViewerActivity.java ---
// File: LogViewerActivity.java (FIXED) - Complete Enhanced Log Viewer with Persistent Settings
// Path: /app/src/main/java/com/modloader/ui/LogViewerActivity.java

package com.modloader.ui;

import android.app.AlertDialog;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.graphics.Color;
import android.graphics.Typeface;
import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.text.Editable;
import android.text.TextWatcher;
import android.view.Menu;
import android.view.MenuItem;
import android.view.View;
import android.widget.*;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.FileProvider;
import androidx.swiperefreshlayout.widget.SwipeRefreshLayout;

import com.modloader.R;
import com.modloader.util.LogUtils;

import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.*;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.regex.Pattern;

public class LogViewerActivity extends AppCompatActivity {
    private static final String TAG = "LogViewerActivity";
    
    // FIXED: SharedPreferences for persistent settings
    private static final String PREFS_NAME = "log_viewer_settings";
    private static final String PREF_AUTO_REFRESH = "auto_refresh_enabled";
    private static final String PREF_SYNTAX_HIGHLIGHTING = "syntax_highlighting_enabled";
    private static final String PREF_TEXT_SIZE = "text_size";
    private static final String PREF_AUTO_SCROLL = "auto_scroll_enabled";
    private static final String PREF_LOG_TYPE_FILTER = "log_type_filter";
    private static final String PREF_LOG_LEVEL_FILTER = "log_level_filter";
    
    // UI Components
    private SwipeRefreshLayout swipeRefreshLayout;
    private ScrollView logScrollView;
    private TextView logTextView;
    private TextView logStatsText;
    private Button refreshButton;
    
    // Filter Components
    private LinearLayout filterSection;
    private Spinner logTypeSpinner;
    private Spinner logLevelSpinner;
    private EditText searchEditText;
    private CheckBox autoScrollCheckbox;
    private Button clearLogsButton;
    private Button exportLogsButton;
    
    // Settings Components (for settings modal)
    private SharedPreferences settings;
    private boolean autoRefreshEnabled = true;
    private boolean syntaxHighlightingEnabled = true;
    private int textSize = 12;
    private boolean autoScrollEnabled = true;
    
    // Data Management
    private List<LogEntry> allLogEntries = new ArrayList<>();
    private List<LogEntry> filteredLogEntries = new ArrayList<>();
    private String currentSearchQuery = "";
    private String currentTypeFilter = "ALL";
    private String currentLevelFilter = "ALL";
    
    // Auto-refresh
    private Handler autoRefreshHandler;
    private Runnable autoRefreshRunnable;
    private static final int AUTO_REFRESH_INTERVAL = 5000; // 5 seconds
    
    // Threading
    private ExecutorService executorService = Executors.newSingleThreadExecutor();
    
    // Log Entry Model
    private static class LogEntry {
        public String timestamp;
        public String level;
        public String tag;
        public String message;
        public String type;
        public String fullText;
        
        public LogEntry(String fullText) {
            this.fullText = fullText;
            parseLogEntry(fullText);
        }
        
        private void parseLogEntry(String logText) {
            // Parse log entry format: [2025-08-31 13:42:28] LEVEL: Message
            try {
                if (logText.startsWith("[")) {
                    int timestampEnd = logText.indexOf("]");
                    if (timestampEnd > 0) {
                        timestamp = logText.substring(1, timestampEnd);
                        String remaining = logText.substring(timestampEnd + 1).trim();
                        
                        if (remaining.contains(":")) {
                            String[] parts = remaining.split(":", 2);
                            level = parts[0].trim();
                            message = parts.length > 1 ? parts[1].trim() : "";
                        } else {
                            level = "INFO";
                            message = remaining;
                        }
                    } else {
                        // Fallback parsing
                        timestamp = "Unknown";
                        level = "INFO";
                        message = logText;
                    }
                } else {
                    // Simple format without timestamp
                    timestamp = "Unknown";
                    level = "INFO";
                    message = logText;
                }
                
                // Determine type based on content
                if (message.contains("USER:") || level.equals("USER")) {
                    type = "USER";
                } else if (message.contains("ERROR") || level.equals("ERROR")) {
                    type = "ERROR";
                } else if (message.contains("WARNING") || level.equals("WARNING")) {
                    type = "WARNING";
                } else if (message.contains("DEBUG") || level.equals("DEBUG")) {
                    type = "DEBUG";
                } else {
                    type = "INFO";
                }
                
                // Extract tag if present
                if (message.startsWith("[") && message.contains("]")) {
                    int tagEnd = message.indexOf("]");
                    tag = message.substring(1, tagEnd);
                } else {
                    tag = "System";
                }
            } catch (Exception e) {
                // Fallback for malformed log entries
                timestamp = "Unknown";
                level = "INFO";
                tag = "System";
                message = logText;
                type = "INFO";
            }
        }
    }
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_log_viewer_enhanced);
        setTitle("📋 Advanced Log Viewer");
        
        // FIXED: Initialize SharedPreferences
        settings = getSharedPreferences(PREFS_NAME, MODE_PRIVATE);
        loadSettings();
        
        initializeViews();
        setupFilters();
        setupListeners();
        setupAutoRefresh();
        
        // Initial log load
        refreshLogs();
        
        LogUtils.logUser("Advanced Log Viewer opened");
    }
    
    // FIXED: Load settings from SharedPreferences
    private void loadSettings() {
        autoRefreshEnabled = settings.getBoolean(PREF_AUTO_REFRESH, true);
        syntaxHighlightingEnabled = settings.getBoolean(PREF_SYNTAX_HIGHLIGHTING, true);
        textSize = settings.getInt(PREF_TEXT_SIZE, 12);
        autoScrollEnabled = settings.getBoolean(PREF_AUTO_SCROLL, true);
        currentTypeFilter = settings.getString(PREF_LOG_TYPE_FILTER, "ALL");
        currentLevelFilter = settings.getString(PREF_LOG_LEVEL_FILTER, "ALL");
        
        LogUtils.logDebug("Settings loaded - AutoRefresh: " + autoRefreshEnabled + 
            ", SyntaxHighlighting: " + syntaxHighlightingEnabled + 
            ", TextSize: " + textSize);
    }
    
    // FIXED: Save settings to SharedPreferences
    private void saveSettings() {
        SharedPreferences.Editor editor = settings.edit();
        editor.putBoolean(PREF_AUTO_REFRESH, autoRefreshEnabled);
        editor.putBoolean(PREF_SYNTAX_HIGHLIGHTING, syntaxHighlightingEnabled);
        editor.putInt(PREF_TEXT_SIZE, textSize);
        editor.putBoolean(PREF_AUTO_SCROLL, autoScrollEnabled);
        editor.putString(PREF_LOG_TYPE_FILTER, currentTypeFilter);
        editor.putString(PREF_LOG_LEVEL_FILTER, currentLevelFilter);
        editor.apply(); // FIXED: Use apply() for asynchronous save
        
        LogUtils.logDebug("Settings saved successfully");
    }
    
    private void initializeViews() {
        // Main components
        swipeRefreshLayout = findViewById(R.id.swipeRefreshLayout);
        logScrollView = findViewById(R.id.logScrollView);
        logTextView = findViewById(R.id.logTextView);
        logStatsText = findViewById(R.id.logStatsText);
        refreshButton = findViewById(R.id.refreshButton);
        
        // Filter components
        filterSection = findViewById(R.id.filterSection);
        logTypeSpinner = findViewById(R.id.logTypeSpinner);
        logLevelSpinner = findViewById(R.id.logLevelSpinner);
        searchEditText = findViewById(R.id.searchEditText);
        autoScrollCheckbox = findViewById(R.id.autoScrollCheckbox);
        clearLogsButton = findViewById(R.id.clearLogsButton);
        exportLogsButton = findViewById(R.id.exportLogsButton);
        
        // FIXED: Apply loaded settings to UI components
        applySettingsToUI();
    }
    
    // FIXED: Apply loaded settings to UI components
    private void applySettingsToUI() {
        if (logTextView != null) {
            logTextView.setTextSize(textSize);
        }
        if (autoScrollCheckbox != null) {
            autoScrollCheckbox.setChecked(autoScrollEnabled);
        }
    }
    
    private void setupFilters() {
        // Setup type filter spinner
        String[] typeOptions = {"ALL", "USER", "ERROR", "WARNING", "DEBUG", "INFO"};
        ArrayAdapter<String> typeAdapter = new ArrayAdapter<>(this, 
            android.R.layout.simple_spinner_item, typeOptions);
        typeAdapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
        logTypeSpinner.setAdapter(typeAdapter);
        
        // FIXED: Set spinner to saved filter value
        int typePosition = Arrays.asList(typeOptions).indexOf(currentTypeFilter);
        if (typePosition >= 0) {
            logTypeSpinner.setSelection(typePosition);
        }
        
        // Setup level filter spinner
        String[] levelOptions = {"ALL", "ERROR", "WARNING", "INFO", "DEBUG", "USER"};
        ArrayAdapter<String> levelAdapter = new ArrayAdapter<>(this, 
            android.R.layout.simple_spinner_item, levelOptions);
        levelAdapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
        logLevelSpinner.setAdapter(levelAdapter);
        
        // FIXED: Set spinner to saved filter value
        int levelPosition = Arrays.asList(levelOptions).indexOf(currentLevelFilter);
        if (levelPosition >= 0) {
            logLevelSpinner.setSelection(levelPosition);
        }
    }
    
    private void setupListeners() {
        // Swipe to refresh
        swipeRefreshLayout.setOnRefreshListener(this::refreshLogs);
        
        // Manual refresh button
        refreshButton.setOnClickListener(v -> refreshLogs());
        
        // Filter listeners
        logTypeSpinner.setOnItemSelectedListener(new AdapterView.OnItemSelectedListener() {
            @Override
            public void onItemSelected(AdapterView<?> parent, View view, int position, long id) {
                currentTypeFilter = parent.getItemAtPosition(position).toString();
                saveSettings(); // FIXED: Save when filter changes
                applyFilters();
            }
            
            @Override
            public void onNothingSelected(AdapterView<?> parent) {}
        });
        
        logLevelSpinner.setOnItemSelectedListener(new AdapterView.OnItemSelectedListener() {
            @Override
            public void onItemSelected(AdapterView<?> parent, View view, int position, long id) {
                currentLevelFilter = parent.getItemAtPosition(position).toString();
                saveSettings(); // FIXED: Save when filter changes
                applyFilters();
            }
            
            @Override
            public void onNothingSelected(AdapterView<?> parent) {}
        });
        
        // Search functionality
        searchEditText.addTextChangedListener(new TextWatcher() {
            @Override
            public void beforeTextChanged(CharSequence s, int start, int count, int after) {}
            
            @Override
            public void onTextChanged(CharSequence s, int start, int before, int count) {
                currentSearchQuery = s.toString().toLowerCase().trim();
                applyFilters();
            }
            
            @Override
            public void afterTextChanged(Editable s) {}
        });
        
        // Auto-scroll checkbox
        autoScrollCheckbox.setOnCheckedChangeListener((buttonView, isChecked) -> {
            autoScrollEnabled = isChecked;
            saveSettings(); // FIXED: Save when setting changes
            if (isChecked) {
                scrollToBottom();
            }
        });
        
        // Action buttons
        clearLogsButton.setOnClickListener(v -> showClearLogsDialog());
        exportLogsButton.setOnClickListener(v -> exportLogs());
    }
    
    private void setupAutoRefresh() {
        autoRefreshHandler = new Handler(Looper.getMainLooper());
        autoRefreshRunnable = new Runnable() {
            @Override
            public void run() {
                if (autoRefreshEnabled) {
                    refreshLogs();
                    autoRefreshHandler.postDelayed(this, AUTO_REFRESH_INTERVAL);
                }
            }
        };
        
        // Start auto-refresh if enabled
        if (autoRefreshEnabled) {
            autoRefreshHandler.postDelayed(autoRefreshRunnable, AUTO_REFRESH_INTERVAL);
        }
    }
    
    private void refreshLogs() {
        executorService.execute(() -> {
            try {
                // Get logs from LogUtils
                String rawLogs = LogUtils.getLogs();
                
                runOnUiThread(() -> {
                    parseAndDisplayLogs(rawLogs);
                    swipeRefreshLayout.setRefreshing(false);
                });
            } catch (Exception e) {
                runOnUiThread(() -> {
                    logTextView.setText("❌ Error loading logs: " + e.getMessage());
                    swipeRefreshLayout.setRefreshing(false);
                });
            }
        });
    }
    
    private void parseAndDisplayLogs(String rawLogs) {
        // Parse logs into entries
        allLogEntries.clear();
        if (rawLogs != null && !rawLogs.trim().isEmpty()) {
            String[] logLines = rawLogs.split("\n");
            for (String line : logLines) {
                if (!line.trim().isEmpty()) {
                    allLogEntries.add(new LogEntry(line.trim()));
                }
            }
        }
        
        // Apply current filters
        applyFilters();
    }
    
    private void applyFilters() {
        filteredLogEntries.clear();
        
        for (LogEntry entry : allLogEntries) {
            boolean matchesType = currentTypeFilter.equals("ALL") || 
                entry.type.equals(currentTypeFilter);
            boolean matchesLevel = currentLevelFilter.equals("ALL") || 
                entry.level.equals(currentLevelFilter);
            boolean matchesSearch = currentSearchQuery.isEmpty() || 
                entry.fullText.toLowerCase().contains(currentSearchQuery);
            
            if (matchesType && matchesLevel && matchesSearch) {
                filteredLogEntries.add(entry);
            }
        }
        
        displayFilteredLogs();
        updateStatistics();
    }
    
    private void displayFilteredLogs() {
        if (filteredLogEntries.isEmpty()) {
            logTextView.setText("📋 No logs match the current filters.\n\n" +
                "Try:\n" +
                "• Changing filter settings\n" +
                "• Clearing search query\n" +
                "• Refreshing logs\n" +
                "• Using the app to generate logs");
            return;
        }
        
        StringBuilder displayText = new StringBuilder();
        
        for (LogEntry entry : filteredLogEntries) {
            if (syntaxHighlightingEnabled) {
                displayText.append(formatLogEntryWithSyntax(entry));
            } else {
                displayText.append(entry.fullText);
            }
            displayText.append("\n");
        }
        
        // FIXED: Apply text size from settings
        logTextView.setText(displayText.toString());
        logTextView.setTextSize(textSize);
        
        // Auto-scroll if enabled
        if (autoScrollEnabled) {
            scrollToBottom();
        }
    }
    
    private String formatLogEntryWithSyntax(LogEntry entry) {
        // This would ideally use SpannableString for actual coloring
        // For now, we'll add visual indicators
        String indicator = "";
        switch (entry.type) {
            case "ERROR":
                indicator = "❌ ";
                break;
            case "WARNING":
                indicator = "⚠️ ";
                break;
            case "DEBUG":
                indicator = "🔍 ";
                break;
            case "USER":
                indicator = "👤 ";
                break;
            default:
                indicator = "ℹ️ ";
                break;
        }
        
        return indicator + entry.fullText;
    }
    
    private void updateStatistics() {
        int totalLogs = allLogEntries.size();
        int showingLogs = filteredLogEntries.size();
        int errorCount = 0;
        int warningCount = 0;
        
        for (LogEntry entry : allLogEntries) {
            if ("ERROR".equals(entry.type)) errorCount++;
            else if ("WARNING".equals(entry.type)) warningCount++;
        }
        
        String statsText = String.format(Locale.getDefault(),
            "📊 Total: %d | Showing: %d | Errors: %d | Warnings: %d",
            totalLogs, showingLogs, errorCount, warningCount);
        
        logStatsText.setText(statsText);
    }
    
    private void scrollToBottom() {
        logScrollView.post(() -> {
            logScrollView.fullScroll(View.FOCUS_DOWN);
        });
    }
    
    // FIXED: Settings dialog with proper persistence
    private void showSettingsDialog() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        
        // Create custom view for settings
        LinearLayout settingsView = new LinearLayout(this);
        settingsView.setOrientation(LinearLayout.VERTICAL);
        settingsView.setPadding(48, 32, 48, 32);
        
        // Title
        TextView title = new TextView(this);
        title.setText("⚙️ Log Viewer Settings");
        title.setTextSize(18);
        title.setTypeface(null, Typeface.BOLD);
        title.setPadding(0, 0, 0, 24);
        settingsView.addView(title);
        
        // Auto-refresh setting
        CheckBox autoRefreshCheck = new CheckBox(this);
        autoRefreshCheck.setText("🔄 Auto-refresh logs (every 5 seconds)");
        autoRefreshCheck.setChecked(autoRefreshEnabled);
        settingsView.addView(autoRefreshCheck);
        
        // Syntax highlighting setting
        CheckBox syntaxHighlightCheck = new CheckBox(this);
        syntaxHighlightCheck.setText("🎨 Enable syntax highlighting");
        syntaxHighlightCheck.setChecked(syntaxHighlightingEnabled);
        settingsView.addView(syntaxHighlightCheck);
        
        // Text size setting
        TextView textSizeLabel = new TextView(this);
        textSizeLabel.setText("📝 Text Size: " + textSize);
        textSizeLabel.setPadding(0, 24, 0, 8);
        settingsView.addView(textSizeLabel);
        
        SeekBar textSizeSeek = new SeekBar(this);
        textSizeSeek.setMin(8);
        textSizeSeek.setMax(24);
        textSizeSeek.setProgress(textSize);
        textSizeSeek.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
            @Override
            public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
                textSizeLabel.setText("📝 Text Size: " + progress);
            }
            @Override
            public void onStartTrackingTouch(SeekBar seekBar) {}
            @Override
            public void onStopTrackingTouch(SeekBar seekBar) {}
        });
        settingsView.addView(textSizeSeek);
        
        // Note about persistence
        TextView persistNote = new TextView(this);
        persistNote.setText("💡 Changes are applied immediately and persist during this session.");
        persistNote.setTextSize(12);
        persistNote.setPadding(0, 16, 0, 0);
        persistNote.setTextColor(Color.GRAY);
        settingsView.addView(persistNote);
        
        builder.setView(settingsView);
        builder.setPositiveButton("✅ Apply", (dialog, which) -> {
            // FIXED: Apply and save settings
            autoRefreshEnabled = autoRefreshCheck.isChecked();
            syntaxHighlightingEnabled = syntaxHighlightCheck.isChecked();
            textSize = textSizeSeek.getProgress();
            
            // Save to SharedPreferences
            saveSettings();
            
            // Apply immediately
            applySettingsToUI();
            displayFilteredLogs(); // Refresh display with new settings
            
            // Restart auto-refresh if needed
            setupAutoRefresh();
            
            Toast.makeText(this, "✅ Settings applied and saved!", Toast.LENGTH_SHORT).show();
            LogUtils.logUser("Log viewer settings updated - AutoRefresh: " + autoRefreshEnabled + 
                ", Syntax: " + syntaxHighlightingEnabled + ", TextSize: " + textSize);
        });
        
        builder.setNegativeButton("Cancel", null);
        builder.show();
    }
    
    private void showClearLogsDialog() {
        new AlertDialog.Builder(this)
            .setTitle("🗑️ Clear Logs")
            .setMessage("Are you sure you want to clear all logs? This action cannot be undone.")
            .setPositiveButton("Clear", (dialog, which) -> {
                LogUtils.clearLogs();
                allLogEntries.clear();
                filteredLogEntries.clear();
                logTextView.setText("📋 Logs cleared.\n\nNew logs will appear here as you use the app.");
                updateStatistics();
                Toast.makeText(this, "🗑️ Logs cleared", Toast.LENGTH_SHORT).show();
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void exportLogs() {
        try {
            File exportDir = new File(getExternalFilesDir(null), "exports");
            if (!exportDir.exists()) {
                exportDir.mkdirs();
            }
            
            String timestamp = new SimpleDateFormat("yyyyMMdd_HHmmss", Locale.getDefault())
                .format(new Date());
            File exportFile = new File(exportDir, "terraria_logs_" + timestamp + ".txt");
            
            try (FileWriter writer = new FileWriter(exportFile)) {
                writer.write("=== ModLoader Log Export ===\n");
                writer.write("Exported: " + new Date().toString() + "\n");
                writer.write("Total Entries: " + allLogEntries.size() + "\n");
                writer.write("Filtered Entries: " + filteredLogEntries.size() + "\n");
                writer.write("Filters - Type: " + currentTypeFilter + ", Level: " + currentLevelFilter + "\n");
                writer.write("Search Query: " + (currentSearchQuery.isEmpty() ? "None" : currentSearchQuery) + "\n");
                writer.write("\n=== LOG ENTRIES ===\n\n");
                
                for (LogEntry entry : filteredLogEntries) {
                    writer.write(entry.fullText + "\n");
                }
            }
            
            // Share the exported file
            Intent shareIntent = new Intent(Intent.ACTION_SEND);
            shareIntent.setType("text/plain");
            shareIntent.putExtra(Intent.EXTRA_STREAM,
                FileProvider.getUriForFile(this, getPackageName() + ".provider", exportFile));
            shareIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
            
            startActivity(Intent.createChooser(shareIntent, "Share Log Export"));
            Toast.makeText(this, "✅ Logs exported: " + exportFile.getName(), Toast.LENGTH_LONG).show();
            
        } catch (Exception e) {
            Toast.makeText(this, "❌ Export failed: " + e.getMessage(), Toast.LENGTH_LONG).show();
            LogUtils.logDebug("Log export error: " + e.getMessage());
        }
    }
    
    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        menu.add(0, 1, 0, "⚙️ Settings")
            .setShowAsAction(MenuItem.SHOW_AS_ACTION_NEVER);
        menu.add(0, 2, 0, "🔄 Toggle Auto-refresh")
            .setShowAsAction(MenuItem.SHOW_AS_ACTION_NEVER);
        menu.add(0, 3, 0, "📤 Export Logs")
            .setShowAsAction(MenuItem.SHOW_AS_ACTION_NEVER);
        return true;
    }
    
    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch (item.getItemId()) {
            case 1:
                showSettingsDialog();
                return true;
            case 2:
                autoRefreshEnabled = !autoRefreshEnabled;
                saveSettings(); // FIXED: Save when toggled
                setupAutoRefresh();
                String status = autoRefreshEnabled ? "enabled" : "disabled";
                Toast.makeText(this, "🔄 Auto-refresh " + status, Toast.LENGTH_SHORT).show();
                return true;
            case 3:
                exportLogs();
                return true;
            default:
                return super.onOptionsItemSelected(item);
        }
    }
    
    @Override
    protected void onResume() {
        super.onResume();
        // Refresh logs when returning to activity
        refreshLogs();
        
        // Restart auto-refresh if it was enabled
        if (autoRefreshEnabled) {
            setupAutoRefresh();
        }
    }
    
    @Override
    protected void onPause() {
        super.onPause();
        // Stop auto-refresh when leaving activity
        if (autoRefreshHandler != null && autoRefreshRunnable != null) {
            autoRefreshHandler.removeCallbacks(autoRefreshRunnable);
        }
        
        // FIXED: Save settings when leaving activity
        saveSettings();
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        // Clean up resources
        if (executorService != null && !executorService.isShutdown()) {
            executorService.shutdown();
        }
        if (autoRefreshHandler != null && autoRefreshRunnable != null) {
            autoRefreshHandler.removeCallbacks(autoRefreshRunnable);
        }
        
        LogUtils.logUser("Advanced Log Viewer closed");
    }
}


--- FILE: /ModLoader/app/src/main/java/com/modloader/ui/LogViewerEnhancedActivity.java ---
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


--- FILE: /ModLoader/app/src/main/java/com/modloader/ui/ModListActivity.java ---
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



--- FILE: /ModLoader/app/src/main/java/com/modloader/ui/ModListAdapter.java ---
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



--- FILE: /ModLoader/app/src/main/java/com/modloader/ui/ModManagementActivity.java ---
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


--- FILE: /ModLoader/app/src/main/java/com/modloader/ui/OfflineDiagnosticActivity.java ---
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


