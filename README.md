Each person gets a file template they can start coding in immediately, with all function signatures, class structures, and TODO markers already placed.

Everything is organized so integration by Person 5 will be smooth.

---

#  **PROJECT STRUCTURE**

```
pom.xml
src/
├── main/
│   └── java/
│       ├── core/
│       │   ├── Process.java 
│       │   ├── SchedulerBase.java
│       │   ├── ExecutionSlice.java
│       │   └── SchedulingResult.java
│       └── schedulers/
│           ├── SJFPreemptiveScheduler.java
│           ├── RoundRobinScheduler.java
│           ├── PriorityPreemptiveScheduler.java
│           └── AGScheduler.java
└── test/
    ├── java/
    │   ├── schedulers/
    │   │   ├── SJFPreemptiveSchedulerTest.java
    │   |   ├── RoundRobinSchedulerTest.java
    │   |   ├── PriorityPreemptiveSchedulerTest.java
    │   |   └── AGSchedulerTest.java
    |   └── utils/
    |        └── TestUtils.java
    └── resources/
        ├── otherschedulers/
        │   ├── test_1.json
        │   ├── test_2.json
        │   ├── test_3.json
        │   ├── test_4.json
        │   ├── test_5.json
        │   └── test_6.json
        └── agscheduler/
            ├── AG_test1.json
            ├── AG_test2.json
            ├── AG_test3.json
            ├── AG_test4.json
            ├── AG_test5.json
            └── AG_test6.json
```

---
# **pom.xml**

```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.scheduling</groupId>
    <artifactId>cpu-schedulers</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>CPU Schedulers</name>
    <description>Implementation of various CPU scheduling algorithms</description>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <junit.version>5.9.3</junit.version>
    </properties>

    <dependencies>
        <!-- JUnit 5 for testing -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-api</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-engine</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>2.10.1</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Compiler plugin -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <source>11</source>
                    <target>11</target>
                </configuration>
            </plugin>

            <!-- Surefire plugin for running tests -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
            </plugin>
        </plugins>
    </build>
</project>
```

---
# **1. Person 1 — Core Files + SJF Template**
---
## **core/Process.java**

```java
package core;

public class Process {
    private final String name;
    private final int arrivalTime;
    private final int burstTime;
    private int remainingTime;
    private int priority;  // Lower number = higher priority (for Priority/AG schedulers)
    private int quantum;   // Per-process quantum (for AG scheduler)

    // Metrics (computed during scheduling)
    private int completionTime = -1;
    private int waitingTime = 0;
    private int turnaroundTime = 0;

    public Process(String name, int arrivalTime, int burstTime, int priority, int quantum) {
        this.name = name;
        this.arrivalTime = arrivalTime;
        this.burstTime = burstTime;
        this.remainingTime = burstTime;
        this.priority = priority;
        this.quantum = quantum;
    }

    // Getters
    public String getName() { return name; }
    public int getArrivalTime() { return arrivalTime; }
    public int getBurstTime() { return burstTime; }
    public int getRemainingTime() { return remainingTime; }
    public int getPriority() { return priority; }
    public int getQuantum() { return quantum; }
    public int getCompletionTime() { return completionTime; }
    public int getWaitingTime() { return waitingTime; }
    public int getTurnaroundTime() { return turnaroundTime; }

    // Setters (for mutable fields during scheduling)
    public void decreaseRemaining(int amount) { remainingTime -= amount; }
    public void setPriority(int priority) { this.priority = priority; }
    public void setQuantum(int quantum) { this.quantum = quantum; }
    public void setCompletionTime(int completionTime) { this.completionTime = completionTime; }
    public void setWaitingTime(int waitingTime) { this.waitingTime = waitingTime; }
    public void setTurnaroundTime(int turnaroundTime) { this.turnaroundTime = turnaroundTime; }

    // Copy method (to avoid modifying original list in schedulers)
    public Process copy() {
        Process copy = new Process(name, arrivalTime, burstTime, priority, quantum);
        copy.remainingTime = this.remainingTime;
        copy.completionTime = this.completionTime;
        copy.waitingTime = this.waitingTime;
        copy.turnaroundTime = this.turnaroundTime;
        return copy;
    }
}

```

---

## **core/SchedulerBase.java**

```java
package core;

import java.util.ArrayList;
import java.util.List;

public abstract class SchedulerBase {

    protected List<ExecutionSlice> slices = new ArrayList<>();

    // All schedulers must implement this; rrQuantum is for RR, ignored by others
    public abstract void run(List<Process> processes, int contextSwitchTime, int rrQuantum);

    // Helper to add a Gantt slice (process or "CS" or "IDLE")
    protected void addSlice(String name, int start, int end) {
        if (start < end) {
            slices.add(new ExecutionSlice(name, start, end));
        }
    }

    // Compute metrics after scheduling (uses completionTime set during run)
    protected void computeMetrics(List<Process> processes) {
        for (Process p : processes) {
            if (p.getCompletionTime() != -1) {
                p.setTurnaroundTime(p.getCompletionTime() - p.getArrivalTime());
                p.setWaitingTime(p.getTurnaroundTime() - p.getBurstTime());
            }
        }
    }

    public List<ExecutionSlice> getSlices() {
        return slices;
    }
}

```

---

## **core/ExecutionSlice.java**

```java

package core;

public class ExecutionSlice {
    public final String processName;  // Process name, "CS" for context switch, or "IDLE"
    public final int start;
    public final int end;

    public ExecutionSlice(String processName, int start, int end) {
        this.processName = processName;
        this.start = start;
        this.end = end;
    }

    @Override
    public String toString() {
        return String.format("%s[%d-%d]", processName, start, end);
    }
}
```

---

## **core/SchedulingResult.java** (Person 3)

```java
package core;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

/**
 * Container for scheduling results and metrics
 */
public class SchedulingResult {
    private final List<ExecutionSlice> executionSlices;
    private final List<Process> processes;
    private final double averageWaitingTime;
    private final double averageTurnaroundTime;
    private final Map<String, List<Integer>> quantumHistory;  // For AG scheduler

    public SchedulingResult(List<ExecutionSlice> executionSlices,
                            List<Process> processes,
                            Map<String, List<Integer>> quantumHistory) {
        this.executionSlices = executionSlices;
        this.processes = processes;
        this.quantumHistory = quantumHistory;

        // Calculate averages
        this.averageWaitingTime = processes.stream()
                .mapToInt(Process::getWaitingTime)
                .average()
                .orElse(0.0);

        this.averageTurnaroundTime = processes.stream()
                .mapToInt(Process::getTurnaroundTime)
                .average()
                .orElse(0.0);
    }

    public List<ExecutionSlice> getExecutionSlices() {
        return executionSlices;
    }

    public List<Process> getProcesses() {
        return processes;
    }

    public double getAverageWaitingTime() {
        return averageWaitingTime;
    }

    public double getAverageTurnaroundTime() {
        return averageTurnaroundTime;
    }

    public Map<String, List<Integer>> getQuantumHistory() {
        return quantumHistory;
    }

    /**
     * Get execution order without CS and IDLE, consolidating consecutive slices
     * FIXED: Now consolidates consecutive executions of the same process
     * Example: [P1, P1, P1, P2, P2] becomes [P1, P2]
     */
    public List<String> getExecutionOrder() {
        List<String> consolidated = new ArrayList<>();
        String lastProcess = null;

        for (ExecutionSlice slice : executionSlices) {
            // Skip CS and IDLE
            if (slice.processName.equals("CS") || slice.processName.equals("IDLE")) {
                continue;
            }

            // Only add if different from last process (consolidate consecutive)
            if (!slice.processName.equals(lastProcess)) {
                consolidated.add(slice.processName);
                lastProcess = slice.processName;
            }
        }

        return consolidated;
    }

    /**
     * Print formatted results
     */
    public void printResults(String schedulerName) {
        System.out.println("\n" + "=".repeat(60));
        System.out.println(schedulerName + " Scheduling Results");
        System.out.println("=".repeat(60));

        System.out.println("\nExecution Order: " + getExecutionOrder());

        System.out.println("\nProcess Metrics:");
        System.out.println(String.format("%-10s %-12s %-15s %-15s",
                "Process", "Waiting Time", "Turnaround Time", "Completion Time"));
        System.out.println("-".repeat(60));

        for (Process p : processes) {
            System.out.println(String.format("%-10s %-12d %-15d %-15d",
                    p.getName(),
                    p.getWaitingTime(),
                    p.getTurnaroundTime(),
                    p.getCompletionTime()));
        }

        System.out.println("-".repeat(60));
        System.out.println(String.format("Average Waiting Time: %.2f", averageWaitingTime));
        System.out.println(String.format("Average Turnaround Time: %.2f", averageTurnaroundTime));

        // Print quantum history if available (for AG scheduler)
        if (quantumHistory != null && !quantumHistory.isEmpty()) {
            System.out.println("\nQuantum History:");
            for (Map.Entry<String, List<Integer>> entry : quantumHistory.entrySet()) {
                System.out.print(entry.getKey() + " Quantum: ");
                List<Integer> hist = entry.getValue();
                for (int i = 0; i < hist.size(); i++) {
                    System.out.print(hist.get(i));
                    if (i < hist.size() - 1) System.out.print(" -> ");
                }
                System.out.println();
            }
        }
    }

    /**
     * Print Gantt chart
     */
    public void printGanttChart() {
        System.out.println("\nGantt Chart:");
        System.out.print("|");
        for (ExecutionSlice slice : executionSlices) {
            System.out.print(String.format(" %s |", slice.processName));
        }
        System.out.println();

        System.out.print(" ");
        for (ExecutionSlice slice : executionSlices) {
            System.out.print(String.format("%d", slice.start));
            int spaces = slice.processName.length() + 1;
            for (int i = 0; i < spaces; i++) System.out.print(" ");
        }
        if (!executionSlices.isEmpty()) {
            System.out.println(executionSlices.get(executionSlices.size() - 1).end);
        }
    }
}

```

---

# **2. Schedulers (Persons 1,2,3,4)**

---

## **schedulers/SJFPreemptiveScheduler.java** (Person 1)

```java
package schedulers;

import core.Process;
import core.SchedulerBase;
import core.ExecutionSlice;

import java.util.*;

/**
 * FIXED: Preemptive Shortest Job First (SJF) Scheduler
 * Key fixes:
 * - Consolidates consecutive execution slices for same process
 * - Properly updates metrics in working processes
 */
public class SJFPreemptiveScheduler extends SchedulerBase {

    // Comparator for ready queue: shortest remaining time, tie-break by arrival time
    private static class ProcessComparator implements Comparator<Process> {
        @Override
        public int compare(Process p1, Process p2) {
            if (p1.getRemainingTime() != p2.getRemainingTime()) {
                return Integer.compare(p1.getRemainingTime(), p2.getRemainingTime());
            }
            return Integer.compare(p1.getArrivalTime(), p2.getArrivalTime());
        }
    }

    @Override
    public void run(List<Process> processes, int contextSwitchTime, int rrQuantum) {
        // Work directly with the processes list to ensure metrics persist
        // But make copies to avoid modifying the original burst times
        Map<String, Process> processMap = new HashMap<>();
        List<Process> workingProcesses = new ArrayList<>();

        for (Process p : processes) {
            Process copy = new Process(p.getName(), p.getArrivalTime(), p.getBurstTime(),
                    p.getPriority(), p.getQuantum());
            workingProcesses.add(copy);
            processMap.put(copy.getName(), p); // Map to original for updates
        }

        // Sort by arrival for efficient addition
        List<Process> arrivalOrder = new ArrayList<>(workingProcesses);
        arrivalOrder.sort(Comparator.comparingInt(Process::getArrivalTime));

        PriorityQueue<Process> readyQueue = new PriorityQueue<>(new ProcessComparator());
        Set<Process> pending = new HashSet<>(workingProcesses);

        int currentTime = 0;
        int arrivalIndex = 0;
        Process currentProcess = null;
        boolean isFirstSwitch = true;

        // Track current slice for consolidation
        String currentSliceProcess = null;
        int currentSliceStart = -1;

        while (!pending.isEmpty()) {
            // Add all processes that have arrived by currentTime
            while (arrivalIndex < arrivalOrder.size() &&
                    arrivalOrder.get(arrivalIndex).getArrivalTime() <= currentTime) {
                readyQueue.add(arrivalOrder.get(arrivalIndex));
                arrivalIndex++;
            }

            // Handle idle time
            if (readyQueue.isEmpty() && arrivalIndex < arrivalOrder.size()) {
                // Finalize current slice if any
                if (currentSliceProcess != null) {
                    addSlice(currentSliceProcess, currentSliceStart, currentTime);
                    currentSliceProcess = null;
                }

                int startIdle = currentTime;
                currentTime = arrivalOrder.get(arrivalIndex).getArrivalTime();
                addSlice("IDLE", startIdle, currentTime);
                continue;
            } else if (readyQueue.isEmpty()) {
                break;
            }

            // Check for preemption
            Process nextBest = readyQueue.peek();
            if (currentProcess != null && nextBest.getRemainingTime() < currentProcess.getRemainingTime()) {
                readyQueue.add(currentProcess);
                currentProcess = null;
            }

            // Select next process if needed
            if (currentProcess == null) {
                // Finalize previous slice
                if (currentSliceProcess != null) {
                    addSlice(currentSliceProcess, currentSliceStart, currentTime);
                    currentSliceProcess = null;
                }

                currentProcess = readyQueue.poll();

                // Add context switch
                if (!isFirstSwitch && contextSwitchTime > 0) {
                    int startCS = currentTime;
                    currentTime += contextSwitchTime;
                    addSlice("CS", startCS, currentTime);

                    // Add arrivals during context switch
                    while (arrivalIndex < arrivalOrder.size() &&
                            arrivalOrder.get(arrivalIndex).getArrivalTime() <= currentTime) {
                        readyQueue.add(arrivalOrder.get(arrivalIndex));
                        arrivalIndex++;
                    }
                }
                isFirstSwitch = false;

                // Start new slice
                currentSliceProcess = currentProcess.getName();
                currentSliceStart = currentTime;
            }

            // Determine execution amount
            int nextEventTime = (arrivalIndex < arrivalOrder.size()) ?
                    arrivalOrder.get(arrivalIndex).getArrivalTime() : Integer.MAX_VALUE;
            int executeAmount = Math.min(currentProcess.getRemainingTime(), nextEventTime - currentTime);

            if (executeAmount <= 0) {
                continue;
            }

            // Execute
            currentTime += executeAmount;
            currentProcess.decreaseRemaining(executeAmount);

            // Check if process finishes
            if (currentProcess.getRemainingTime() == 0) {
                currentProcess.setCompletionTime(currentTime);

                // Update original process
                Process original = processMap.get(currentProcess.getName());
                original.setCompletionTime(currentTime);

                pending.remove(currentProcess);

                // Finalize slice
                addSlice(currentSliceProcess, currentSliceStart, currentTime);
                currentSliceProcess = null;
                currentProcess = null;
            }
        }

        // Finalize any remaining slice
        if (currentSliceProcess != null) {
            addSlice(currentSliceProcess, currentSliceStart, currentTime);
        }

        // Compute metrics on original processes
        computeMetrics(processes);
    }
}

```

---

## **schedulers/RoundRobinScheduler.java** (Person 2)


```java
package schedulers;

import core.*;
import java.util.*;

/**
 * FIXED: Round Robin Scheduler with Context Switching
 * Key fixes:
 * - Consolidates consecutive execution into single slices
 * - Properly updates metrics in original processes
 */
public class RoundRobinScheduler extends SchedulerBase {

    @Override
    public void run(List<core.Process> processes, int contextSwitchTime, int rrQuantum) {
        // Work with process mapping to update originals
        Map<String, core.Process> processMap = new HashMap<>();
        List<core.Process> workingProcesses = new ArrayList<>();

        for (core.Process p : processes) {
            core.Process copy = p.copy();
            workingProcesses.add(copy);
            processMap.put(copy.getName(), p);
        }

        // Sort by arrival time
        List<core.Process> arrivalOrder = new ArrayList<>(workingProcesses);
        arrivalOrder.sort(Comparator.comparingInt(core.Process::getArrivalTime));

        Queue<core.Process> readyQueue = new LinkedList<>();
        Set<core.Process> addedToQueue = new HashSet<>();

        int currentTime = 0;
        int arrivalIndex = 0;
        boolean isFirstExecution = true;

        while (true) {
            // Check if all complete
            boolean allDone = workingProcesses.stream().allMatch(p -> p.getRemainingTime() == 0);
            if (allDone) break;

            // Add newly arrived processes
            while (arrivalIndex < arrivalOrder.size() &&
                    arrivalOrder.get(arrivalIndex).getArrivalTime() <= currentTime) {
                core.Process p = arrivalOrder.get(arrivalIndex);
                if (p.getRemainingTime() > 0 && !addedToQueue.contains(p)) {
                    readyQueue.add(p);
                    addedToQueue.add(p);
                }
                arrivalIndex++;
            }

            // Handle idle
            if (readyQueue.isEmpty()) {
                if (arrivalIndex < arrivalOrder.size()) {
                    int nextArrival = arrivalOrder.get(arrivalIndex).getArrivalTime();
                    addSlice("IDLE", currentTime, nextArrival);
                    currentTime = nextArrival;
                }
                continue;
            }

            // Get next process
            core.Process currentProcess = readyQueue.poll();
            addedToQueue.remove(currentProcess);

            // Add context switch
            if (!isFirstExecution && contextSwitchTime > 0) {
                int csStart = currentTime;
                currentTime += contextSwitchTime;
                addSlice("CS", csStart, currentTime);

                // Add arrivals during CS
                while (arrivalIndex < arrivalOrder.size() &&
                        arrivalOrder.get(arrivalIndex).getArrivalTime() <= currentTime) {
                    core.Process p = arrivalOrder.get(arrivalIndex);
                    if (p.getRemainingTime() > 0 && !addedToQueue.contains(p)) {
                        readyQueue.add(p);
                        addedToQueue.add(p);
                    }
                    arrivalIndex++;
                }
            }
            isFirstExecution = false;

            // Execute for quantum or remaining time
            int executeTime = Math.min(rrQuantum, currentProcess.getRemainingTime());
            int startTime = currentTime;

            // Execute in one consolidated slice
            for (int i = 0; i < executeTime; i++) {
                currentTime++;
                currentProcess.decreaseRemaining(1);

                // Check for arrivals during execution
                while (arrivalIndex < arrivalOrder.size() &&
                        arrivalOrder.get(arrivalIndex).getArrivalTime() <= currentTime) {
                    core.Process p = arrivalOrder.get(arrivalIndex);
                    if (p.getRemainingTime() > 0 && !addedToQueue.contains(p)) {
                        readyQueue.add(p);
                        addedToQueue.add(p);
                    }
                    arrivalIndex++;
                }
            }

            // Add consolidated slice
            addSlice(currentProcess.getName(), startTime, currentTime);

            // Check completion
            if (currentProcess.getRemainingTime() == 0) {
                currentProcess.setCompletionTime(currentTime);

                // Update original
                core.Process original = processMap.get(currentProcess.getName());
                original.setCompletionTime(currentTime);
            } else {
                // Re-queue
                readyQueue.add(currentProcess);
                addedToQueue.add(currentProcess);
            }
        }

        // Compute metrics on originals
        computeMetrics(processes);
    }
}

```

---

## **schedulers/PriorityPreemptiveScheduler.java** (Person 3)


```java

package schedulers;

import core.Process;
import core.SchedulerBase;

import java.util.*;

/**
 * FINAL CORRECT: Preemptive Priority Scheduling with Aging
 * Aging happens at EVERY time unit for waiting processes
 * Every AGING_INTERVAL time units of waiting reduces priority by 1
 */
public class PriorityPreemptiveScheduler extends SchedulerBase {

    private static final int AGING_INTERVAL = 5;

    @Override
    public void run(List<Process> processes, int contextSwitchTime, int rrQuantum) {
        // Map to originals for metric updates
        Map<String, Process> processMap = new HashMap<>();
        List<Process> workProcesses = new ArrayList<>();

        for (Process p : processes) {
            Process copy = p.copy();
            workProcesses.add(copy);
            processMap.put(copy.getName(), p);
        }

        List<Process> sortedByArrival = new ArrayList<>(workProcesses);
        sortedByArrival.sort(Comparator.comparingInt(Process::getArrivalTime));

        PriorityQueue<Process> readyQueue = new PriorityQueue<>(
                Comparator.comparingInt(Process::getPriority)
                        .thenComparingInt(Process::getArrivalTime)
        );

        Set<Process> addedToQueue = new HashSet<>();
        Map<Process, Integer> waitingTimeMap = new HashMap<>();
        Map<Process, Integer> originalPriorityMap = new HashMap<>();

        int currentTime = 0;
        Process currentProcess = null;
        boolean isFirstExecution = true;
        int arrivalIndex = 0;

        // Track current slice
        String currentSliceProcess = null;
        int currentSliceStart = -1;

        while (workProcesses.stream().anyMatch(p -> p.getRemainingTime() > 0)) {
            // Add newly arrived
            while (arrivalIndex < sortedByArrival.size() &&
                    sortedByArrival.get(arrivalIndex).getArrivalTime() <= currentTime) {
                Process p = sortedByArrival.get(arrivalIndex);
                if (p.getRemainingTime() > 0 && !addedToQueue.contains(p)) {
                    readyQueue.add(p);
                    waitingTimeMap.put(p, 0);
                    originalPriorityMap.put(p, p.getPriority());
                    addedToQueue.add(p);
                }
                arrivalIndex++;
            }

            // Check for aging-based preemption BEFORE checking normal priority preemption
            // This ensures aged processes get a chance to run
            if (!readyQueue.isEmpty()) {
                updatePrioritiesByAging(readyQueue, waitingTimeMap, originalPriorityMap);
            }

            // Check preemption
            if (currentProcess != null && !readyQueue.isEmpty()) {
                Process highestPriority = readyQueue.peek();
                if (highestPriority.getPriority() < currentProcess.getPriority()) {
                    // Finalize current slice
                    if (currentSliceProcess != null) {
                        addSlice(currentSliceProcess, currentSliceStart, currentTime);
                        currentSliceProcess = null;
                    }

                    readyQueue.add(currentProcess);
                    addedToQueue.add(currentProcess);
                    waitingTimeMap.put(currentProcess, 0);
                    currentProcess = null;
                }
            }

            // Select process
            if (currentProcess == null && !readyQueue.isEmpty()) {
                // Finalize previous slice
                if (currentSliceProcess != null) {
                    addSlice(currentSliceProcess, currentSliceStart, currentTime);
                    currentSliceProcess = null;
                }

                // Context switch
                if (!isFirstExecution && contextSwitchTime > 0) {
                    int csStart = currentTime;
                    currentTime += contextSwitchTime;
                    addSlice("CS", csStart, currentTime);

                    // Arrivals during CS
                    while (arrivalIndex < sortedByArrival.size() &&
                            sortedByArrival.get(arrivalIndex).getArrivalTime() <= currentTime) {
                        Process p = sortedByArrival.get(arrivalIndex);
                        if (p.getRemainingTime() > 0 && !addedToQueue.contains(p)) {
                            readyQueue.add(p);
                            waitingTimeMap.put(p, 0);
                            originalPriorityMap.put(p, p.getPriority());
                            addedToQueue.add(p);
                        }
                        arrivalIndex++;
                    }

                    // Update waiting times during CS
                    for (Process p : readyQueue) {
                        waitingTimeMap.put(p, waitingTimeMap.get(p) + contextSwitchTime);
                    }

                    // Check aging after context switch
                    updatePrioritiesByAging(readyQueue, waitingTimeMap, originalPriorityMap);
                }

                currentProcess = readyQueue.poll();
                addedToQueue.remove(currentProcess);
                isFirstExecution = false;

                // Start new slice
                currentSliceProcess = currentProcess.getName();
                currentSliceStart = currentTime;
            }

            // Handle idle
            if (currentProcess == null) {
                if (arrivalIndex < sortedByArrival.size()) {
                    int nextArrival = sortedByArrival.get(arrivalIndex).getArrivalTime();
                    if (nextArrival > currentTime) {
                        addSlice("IDLE", currentTime, nextArrival);
                        currentTime = nextArrival;
                    }
                }
                continue;
            }

            // Execute for 1 time unit
            currentTime++;
            currentProcess.decreaseRemaining(1);

            // Update waiting times for processes in queue
            for (Process p : readyQueue) {
                waitingTimeMap.put(p, waitingTimeMap.getOrDefault(p, 0) + 1);
            }

            // Check completion
            if (currentProcess.getRemainingTime() == 0) {
                currentProcess.setCompletionTime(currentTime);

                // Update original
                Process original = processMap.get(currentProcess.getName());
                original.setCompletionTime(currentTime);

                waitingTimeMap.remove(currentProcess);
                originalPriorityMap.remove(currentProcess);

                // Finalize slice
                addSlice(currentSliceProcess, currentSliceStart, currentTime);
                currentSliceProcess = null;
                currentProcess = null;
            }
        }

        // Finalize any remaining slice
        if (currentSliceProcess != null) {
            addSlice(currentSliceProcess, currentSliceStart, currentTime);
        }

        // Compute metrics
        computeMetrics(processes);
    }

    /**
     * Update process priorities based on continuous aging
     * Each AGING_INTERVAL time units of waiting reduces effective priority by 1
     */
    private void updatePrioritiesByAging(PriorityQueue<Process> readyQueue,
                                         Map<Process, Integer> waitingMap,
                                         Map<Process, Integer> originalPriorityMap) {
        // Remove all from queue
        List<Process> tempList = new ArrayList<>();
        while (!readyQueue.isEmpty()) {
            tempList.add(readyQueue.poll());
        }

        // Update priorities based on waiting time
        for (Process p : tempList) {
            int originalPriority = originalPriorityMap.get(p);
            int waitTime = waitingMap.getOrDefault(p, 0);

            // Calculate effective priority: reduce by 1 for each AGING_INTERVAL of waiting
            int agingBoost = waitTime / AGING_INTERVAL;
            int effectivePriority = Math.max(0, originalPriority - agingBoost);

            p.setPriority(effectivePriority);
        }

        // Re-add all (queue will re-sort)
        readyQueue.addAll(tempList);
    }
}

```

---

## **schedulers/AGScheduler.java** (Person 4)

```java
package schedulers;

import core.*;
import core.Process;

import java.util.*;

public class AGScheduler extends SchedulerBase {

    private Map<String, List<Integer>> quantumHistory = new LinkedHashMap<>();

    public void run(List<Process> processes, int contextSwitchTime, int rrQuantum) {
        slices = new ArrayList<>();

        List<Process> working = processes; // Work directly on input processes

        for (Process p : working) {
            quantumHistory.put(p.getName(), new ArrayList<>());
            quantumHistory.get(p.getName()).add(p.getQuantum());
        }

        working.sort(Comparator.comparingInt(Process::getArrivalTime));

        LinkedList<Process> readyQueue = new LinkedList<>();
        int time = 0;
        int nextProcessIndex = 0;
        boolean first = true;

        while (true) {
            boolean allDone = true;
            for (Process p : working) {
                if (p.getRemainingTime() > 0) {
                    allDone = false;
                    break;
                }
            }
            if (allDone) break;

            while (nextProcessIndex < working.size() &&
                    working.get(nextProcessIndex).getArrivalTime() <= time) {
                Process p = working.get(nextProcessIndex);
                if (p.getRemainingTime() > 0) {
                    readyQueue.offer(p);
                }
                nextProcessIndex++;
            }

            if (readyQueue.isEmpty()) {
                if (nextProcessIndex < working.size()) {
                    int nextArrival = working.get(nextProcessIndex).getArrivalTime();
                    addSlice("IDLE", time, nextArrival);
                    time = nextArrival;
                }
                continue;
            }

            Process current = readyQueue.poll();

            if (!first && contextSwitchTime > 0) {
                addSlice("CS", time, time + contextSwitchTime);
                time += contextSwitchTime;

                while (nextProcessIndex < working.size() &&
                        working.get(nextProcessIndex).getArrivalTime() <= time) {
                    Process p = working.get(nextProcessIndex);
                    if (p.getRemainingTime() > 0) {
                        readyQueue.offer(p);
                    }
                    nextProcessIndex++;
                }
            }
            first = false;

            int quantum = current.getQuantum();
            int executedThisTurn = 0;

            int fcfsPhase = (int) Math.ceil(quantum * 0.25);
            int priorityPhase = (int) Math.ceil(quantum * 0.25);
            int sjfPhase = quantum - fcfsPhase - priorityPhase;

            boolean preempted = false;
            int preemptReason = 0;

            // FCFS Phase
            for (int t = 0; t < fcfsPhase && current.getRemainingTime() > 0; t++) {
                addSlice(current.getName(), time, time + 1);
                current.decreaseRemaining(1);
                time++;
                executedThisTurn++;

                while (nextProcessIndex < working.size() &&
                        working.get(nextProcessIndex).getArrivalTime() <= time) {
                    Process p = working.get(nextProcessIndex);
                    if (p.getRemainingTime() > 0) {
                        readyQueue.offer(p);
                    }
                    nextProcessIndex++;
                }
            }

            if (current.getRemainingTime() == 0) {
                current.setCompletionTime(time);
                finishProcess(current);
                continue;
            }

            // Priority Phase
            for (int t = 0; t < priorityPhase && current.getRemainingTime() > 0 && !preempted; t++) {
                // Find best preemptor: lowest priority number
                Process preemptor = null;
                for (Process p : readyQueue) {
                    if (p.getPriority() < current.getPriority()) {
                        if (preemptor == null || p.getPriority() < preemptor.getPriority()) {
                            preemptor = p;
                        }
                    }
                }
                if (preemptor != null) {
                    preempted = true;
                    preemptReason = 1;
                    readyQueue.remove(preemptor);
                    handlePreemption(current, quantum, executedThisTurn, preemptReason);
                    readyQueue.offer(current);
                    readyQueue.addFirst(preemptor);
                    break;
                }

                addSlice(current.getName(), time, time + 1);
                current.decreaseRemaining(1);
                time++;
                executedThisTurn++;

                while (nextProcessIndex < working.size() &&
                        working.get(nextProcessIndex).getArrivalTime() <= time) {
                    Process p = working.get(nextProcessIndex);
                    if (p.getRemainingTime() > 0) {
                        readyQueue.offer(p);
                    }
                    nextProcessIndex++;
                }
            }

            if (preempted) continue;

            if (current.getRemainingTime() == 0) {
                current.setCompletionTime(time);
                finishProcess(current);
                continue;
            }

            // SJF Phase
            for (int t = 0; t < sjfPhase && current.getRemainingTime() > 0 && !preempted; t++) {
                // Find best preemptor: smallest remaining time, tie break by priority
                Process preemptor = null;
                for (Process p : readyQueue) {
                    if (p.getRemainingTime() < current.getRemainingTime()) {
                        if (preemptor == null ||
                                p.getRemainingTime() < preemptor.getRemainingTime() ||
                                (p.getRemainingTime() == preemptor.getRemainingTime() && p.getPriority() < preemptor.getPriority())) {
                            preemptor = p;
                        }
                    }
                }
                if (preemptor != null) {
                    preempted = true;
                    preemptReason = 2;
                    readyQueue.remove(preemptor);
                    handlePreemption(current, quantum, executedThisTurn, preemptReason);
                    readyQueue.offer(current);
                    readyQueue.addFirst(preemptor);
                    break;
                }

                addSlice(current.getName(), time, time + 1);
                current.decreaseRemaining(1);
                time++;
                executedThisTurn++;

                while (nextProcessIndex < working.size() &&
                        working.get(nextProcessIndex).getArrivalTime() <= time) {
                    Process p = working.get(nextProcessIndex);
                    if (p.getRemainingTime() > 0) {
                        readyQueue.offer(p);
                    }
                    nextProcessIndex++;
                }
            }

            if (preempted) continue;

            if (current.getRemainingTime() == 0) {
                current.setCompletionTime(time);
                finishProcess(current);
            } else {
                int newQuantum = current.getQuantum() + 2;
                current.setQuantum(newQuantum);
                quantumHistory.get(current.getName()).add(newQuantum);
                readyQueue.offer(current);
            }
        }

        computeMetrics(working);
    }

    private void finishProcess(Process p) {
        p.setQuantum(0);
        quantumHistory.get(p.getName()).add(0);
    }

    private void handlePreemption(Process p, int originalQuantum, int executed, int reason) {
        int remainingInQuantum = originalQuantum - executed;
        int newQuantum = p.getQuantum() + (reason == 1 ? (int) Math.ceil(remainingInQuantum / 2.0) : remainingInQuantum);
        p.setQuantum(newQuantum);
        quantumHistory.get(p.getName()).add(newQuantum);
    }

    public Map<String, List<Integer>> getQuantumHistory() {
        return quantumHistory;
    }

    public void printQuantumHistory() {
        System.out.println("\nQuantum History:");
        for (Map.Entry<String, List<Integer>> entry : quantumHistory.entrySet()) {
            System.out.print(entry.getKey() + " Quantum: ");
            List<Integer> hist = entry.getValue();
            for (int i = 0; i < hist.size(); i++) {
                System.out.print(hist.get(i));
                if (i < hist.size() - 1) System.out.print(" -> ");
            }
            System.out.println();
        }
    }
}
```

---


# **3. Integration Files (Person 5)**

---

## **TestUtils.java**


```java
package utils;

import com.google.gson.*;
import core.Process;
import core.SchedulingResult;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.*;

/**
 * Utility class for loading and parsing test case JSON files
 * FIXED: Now properly handles AG scheduler JSON format
 */
public class TestUtils {

    /**
     * Load a test case from a JSON file
     */
    public static TestCase loadTestCase(String filepath) throws IOException {
        String content = new String(Files.readAllBytes(Paths.get(filepath)));
        JsonObject json = JsonParser.parseString(content).getAsJsonObject();

        TestCase testCase = new TestCase();

        // Parse input
        JsonObject input = json.getAsJsonObject("input");

        if (input.has("contextSwitch")) {
            testCase.contextSwitch = input.get("contextSwitch").getAsInt();
        }

        if (input.has("rrQuantum")) {
            testCase.rrQuantum = input.get("rrQuantum").getAsInt();
        }

        if (input.has("agingInterval")) {
            testCase.agingInterval = input.get("agingInterval").getAsInt();
        }

        // Parse processes
        JsonArray processesArray = input.getAsJsonArray("processes");
        testCase.processes = new ArrayList<>();

        for (JsonElement elem : processesArray) {
            JsonObject procObj = elem.getAsJsonObject();
            String name = procObj.get("name").getAsString();
            int arrival = procObj.get("arrival").getAsInt();
            int burst = procObj.get("burst").getAsInt();
            int priority = procObj.has("priority") ? procObj.get("priority").getAsInt() : 0;
            int quantum = procObj.has("quantum") ? procObj.get("quantum").getAsInt() : 0;

            testCase.processes.add(new Process(name, arrival, burst, priority, quantum));
        }

        // Parse expected output
        if (json.has("expectedOutput")) {
            JsonElement expectedOutputElem = json.get("expectedOutput");

            // FIXED: Check if expectedOutput is a JsonObject or direct structure
            if (expectedOutputElem.isJsonObject()) {
                JsonObject expectedOutput = expectedOutputElem.getAsJsonObject();

                // Check if this is an AG test (direct structure) or regular test (with scheduler keys)
                if (expectedOutput.has("executionOrder") && expectedOutput.has("processResults")) {
                    // This is AG format - direct expected output
                    testCase.expectedOutput = new HashMap<>();
                    testCase.expectedOutput.put("AG", parseSingleExpectedResult(expectedOutput));
                } else {
                    // This is regular format - multiple schedulers
                    testCase.expectedOutput = parseExpectedOutput(expectedOutput);
                }
            }
        }

        return testCase;
    }

    /**
     * Parse expected output for all schedulers (SJF, RR, Priority format)
     */
    private static Map<String, ExpectedResult> parseExpectedOutput(JsonObject expectedOutput) {
        Map<String, ExpectedResult> results = new HashMap<>();

        for (String schedulerName : expectedOutput.keySet()) {
            JsonElement schedulerElem = expectedOutput.get(schedulerName);

            // Skip if not a JsonObject
            if (!schedulerElem.isJsonObject()) {
                continue;
            }

            JsonObject schedulerOutput = schedulerElem.getAsJsonObject();
            results.put(schedulerName, parseSingleExpectedResult(schedulerOutput));
        }

        return results;
    }

    /**
     * Parse a single scheduler's expected result
     * FIXED: Extracted to handle both AG and regular formats
     */
    private static ExpectedResult parseSingleExpectedResult(JsonObject schedulerOutput) {
        ExpectedResult result = new ExpectedResult();

        // Parse execution order
        if (schedulerOutput.has("executionOrder")) {
            JsonElement execOrderElem = schedulerOutput.get("executionOrder");

            // FIXED: Handle both JsonArray and JsonObject
            if (execOrderElem.isJsonArray()) {
                JsonArray execOrder = execOrderElem.getAsJsonArray();
                result.executionOrder = new ArrayList<>();
                for (JsonElement elem : execOrder) {
                    result.executionOrder.add(elem.getAsString());
                }
            }
        }

        // Parse process results
        if (schedulerOutput.has("processResults")) {
            JsonElement procResultsElem = schedulerOutput.get("processResults");

            // FIXED: Handle both JsonArray and JsonObject
            if (procResultsElem.isJsonArray()) {
                JsonArray procResults = procResultsElem.getAsJsonArray();
                result.processResults = new ArrayList<>();
                for (JsonElement elem : procResults) {
                    JsonObject procObj = elem.getAsJsonObject();
                    ProcessResult pr = new ProcessResult();
                    pr.name = procObj.get("name").getAsString();
                    pr.waitingTime = procObj.get("waitingTime").getAsInt();
                    pr.turnaroundTime = procObj.get("turnaroundTime").getAsInt();

                    if (procObj.has("quantumHistory")) {
                        pr.quantumHistory = new ArrayList<>();
                        JsonArray qh = procObj.getAsJsonArray("quantumHistory");
                        for (JsonElement qElem : qh) {
                            pr.quantumHistory.add(qElem.getAsInt());
                        }
                    }

                    result.processResults.add(pr);
                }
            }
        }

        // Parse averages
        if (schedulerOutput.has("averageWaitingTime")) {
            result.averageWaitingTime = schedulerOutput.get("averageWaitingTime").getAsDouble();
        }

        if (schedulerOutput.has("averageTurnaroundTime")) {
            result.averageTurnaroundTime = schedulerOutput.get("averageTurnaroundTime").getAsDouble();
        }

        return result;
    }

    /**
     * Compare actual scheduling result with expected result
     */
    public static boolean compareResults(SchedulingResult actual, ExpectedResult expected, String schedulerName) {
        List<String> errors = new ArrayList<>();

        // Compare execution order (ignore CS and IDLE)
        List<String> actualOrder = actual.getExecutionOrder();
        if (!actualOrder.equals(expected.executionOrder)) {
            errors.add("Execution order mismatch: expected " + expected.executionOrder +
                    " but got " + actualOrder);
        }

        // Compare process metrics
        for (ProcessResult expectedProc : expected.processResults) {
            Process actualProc = findProcessByName(actual.getProcesses(), expectedProc.name);
            if (actualProc == null) {
                errors.add("Process " + expectedProc.name + " not found in results");
                continue;
            }

            if (actualProc.getWaitingTime() != expectedProc.waitingTime) {
                errors.add(String.format("Process %s: waiting time mismatch (expected %d, got %d)",
                        expectedProc.name, expectedProc.waitingTime, actualProc.getWaitingTime()));
            }

            if (actualProc.getTurnaroundTime() != expectedProc.turnaroundTime) {
                errors.add(String.format("Process %s: turnaround time mismatch (expected %d, got %d)",
                        expectedProc.name, expectedProc.turnaroundTime, actualProc.getTurnaroundTime()));
            }
        }

        // Compare averages (with tolerance for floating point)
        double avgWaitDiff = Math.abs(actual.getAverageWaitingTime() - expected.averageWaitingTime);
        if (avgWaitDiff > 0.01) {
            errors.add(String.format("Average waiting time mismatch (expected %.2f, got %.2f)",
                    expected.averageWaitingTime, actual.getAverageWaitingTime()));
        }

        double avgTurnDiff = Math.abs(actual.getAverageTurnaroundTime() - expected.averageTurnaroundTime);
        if (avgTurnDiff > 0.01) {
            errors.add(String.format("Average turnaround time mismatch (expected %.2f, got %.2f)",
                    expected.averageTurnaroundTime, actual.getAverageTurnaroundTime()));
        }

        if (!errors.isEmpty()) {
            System.err.println("\n" + schedulerName + " FAILED:");
            for (String error : errors) {
                System.err.println("  - " + error);
            }
            return false;
        }

        return true;
    }

    /**
     * Compare quantum history for AG scheduler
     */
    public static boolean compareQuantumHistory(Map<String, List<Integer>> actual,
                                                List<ProcessResult> expected) {
        List<String> errors = new ArrayList<>();

        for (ProcessResult expectedProc : expected) {
            if (expectedProc.quantumHistory == null) continue;

            List<Integer> actualHistory = actual.get(expectedProc.name);
            if (actualHistory == null) {
                errors.add("No quantum history found for process " + expectedProc.name);
                continue;
            }

            if (!actualHistory.equals(expectedProc.quantumHistory)) {
                errors.add(String.format("Process %s quantum history mismatch: expected %s but got %s",
                        expectedProc.name, expectedProc.quantumHistory, actualHistory));
            }
        }

        if (!errors.isEmpty()) {
            System.err.println("\nQuantum History FAILED:");
            for (String error : errors) {
                System.err.println("  - " + error);
            }
            return false;
        }

        return true;
    }

    private static Process findProcessByName(List<Process> processes, String name) {
        for (Process p : processes) {
            if (p.getName().equals(name)) {
                return p;
            }
        }
        return null;
    }

    /**
     * Container for test case data
     */
    public static class TestCase {
        public int contextSwitch = 0;
        public int rrQuantum = 0;
        public int agingInterval = 0;
        public List<Process> processes;
        public Map<String, ExpectedResult> expectedOutput;
    }

    /**
     * Container for expected results
     */
    public static class ExpectedResult {
        public List<String> executionOrder;
        public List<ProcessResult> processResults;
        public double averageWaitingTime;
        public double averageTurnaroundTime;
    }

    /**
     * Container for expected process result
     */
    public static class ProcessResult {
        public String name;
        public int waitingTime;
        public int turnaroundTime;
        public List<Integer> quantumHistory;
    }
}
```
I wanted to clarify something. In your code you should write unit tests to verify the correctness of your logic
So you will need to:
For each test case you will parse the json file inputs and outputs
For each test case you should run the schedule with the specified inputs
For each test case you have to use assert to verify that your code is producing the same as expected output
All of this should be done in the unit testing code there shouldn't be any manual comparisons

---

## **AGSchedulerTest.java**


```java

package schedulers;

import core.Process;
import core.SchedulingResult;
import org.junit.jupiter.api.Test;
import utils.TestUtils;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Unit tests for AG (Adaptive Guarantee) Scheduler
 */
public class AGSchedulerTest {

    @Test
    public void testAGCase1() throws IOException {
        runTestFromFile("src/test/resources/agscheduler/AG_test1.json");
    }

    @Test
    public void testAGCase2() throws IOException {
        runTestFromFile("src/test/resources/agscheduler/AG_test2.json");
    }

    @Test
    public void testAGCase3() throws IOException {
        runTestFromFile("src/test/resources/agscheduler/AG_test3.json");
    }

    @Test
    public void testAGCase4() throws IOException {
        runTestFromFile("src/test/resources/agscheduler/AG_test4.json");
    }

    @Test
    public void testAGCase5() throws IOException {
        runTestFromFile("src/test/resources/agscheduler/AG_test5.json");
    }

    @Test
    public void testAGCase6() throws IOException {
        runTestFromFile("src/test/resources/agscheduler/AG_test6.json");
    }

    /**
     * Helper method to run a test from a JSON file
     */
    private void runTestFromFile(String filepath) throws IOException {
        // Load test case
        TestUtils.TestCase testCase = TestUtils.loadTestCase(filepath);

        // Make copies of processes for the test
        List<Process> processes = new ArrayList<>();
        for (Process p : testCase.processes) {
            processes.add(p.copy());
        }

        // Run scheduler
        AGScheduler scheduler = new AGScheduler();
        scheduler.run(processes, 0, 0);  // AG doesn't use contextSwitch or rrQuantum from test

        // Get quantum history
        Map<String, List<Integer>> quantumHistory = scheduler.getQuantumHistory();

        // Create result object
        SchedulingResult result = new SchedulingResult(
                scheduler.getSlices(),
                processes,
                quantumHistory
        );

        // Get expected results - AG tests don't have scheduler key, direct output
        TestUtils.ExpectedResult expected;
        if (testCase.expectedOutput.containsKey("expectedOutput")) {
            expected = testCase.expectedOutput.get("expectedOutput");
        } else {
            // For AG tests, the structure is different - output is directly in expectedOutput
            expected = testCase.expectedOutput.get("AG");
            if (expected == null) {
                // Parse from direct structure
                expected = parseAGExpectedOutput(testCase);
            }
        }

        assertNotNull(expected, "No expected output found for AG Scheduler");

        // Compare general results
        boolean passed = TestUtils.compareResults(result, expected, "AG Scheduler");

        // Compare quantum history
        boolean quantumPassed = TestUtils.compareQuantumHistory(quantumHistory, expected.processResults);

        assertTrue(passed && quantumPassed, "Test failed - see error messages above");

        System.out.println("✓ AG Scheduler test passed: " + filepath);
    }

    /**
     * Parse expected output specifically for AG test format
     */
    private TestUtils.ExpectedResult parseAGExpectedOutput(TestUtils.TestCase testCase) {
        // This is a fallback - the TestUtils should handle it, but just in case
        // For AG tests, we need to look at the raw expected output
        return testCase.expectedOutput.values().iterator().next();
    }
}

```
---

## **PriorityPreemptiveSchedulerTest.java**


```java

package schedulers;

import core.Process;
import core.SchedulingResult;
import org.junit.jupiter.api.Test;
import utils.TestUtils;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Unit tests for Priority Preemptive Scheduler
 */
public class PriorityPreemptiveSchedulerTest {

    @Test
    public void testCase1_BasicMixedArrivals() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_1.json", "Priority");
    }

    @Test
    public void testCase2_AllArrivalsAtZero() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_2.json", "Priority");
    }

    @Test
    public void testCase3_VariedBurstTimes() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_3.json", "Priority");
    }

    @Test
    public void testCase4_LargeBurstsWithGaps() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_4.json", "Priority");
    }

    @Test
    public void testCase5_ShortBurstsHighFrequency() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_5.json", "Priority");
    }

    @Test
    public void testCase6_MixedScenario() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_6.json", "Priority");
    }

    /**
     * Helper method to run a test from a JSON file
     */
    private void runTestFromFile(String filepath, String schedulerKey) throws IOException {
        // Load test case
        TestUtils.TestCase testCase = TestUtils.loadTestCase(filepath);

        // Make copies of processes for the test
        List<Process> processes = new ArrayList<>();
        for (Process p : testCase.processes) {
            processes.add(p.copy());
        }

        // Run scheduler
        PriorityPreemptiveScheduler scheduler = new PriorityPreemptiveScheduler();
        scheduler.run(processes, testCase.contextSwitch, testCase.rrQuantum);

        // Create result object
        SchedulingResult result = new SchedulingResult(
                scheduler.getSlices(),
                processes,
                null  // Priority doesn't have quantum history
        );

        // Get expected results
        TestUtils.ExpectedResult expected = testCase.expectedOutput.get(schedulerKey);
        assertNotNull(expected, "No expected output found for " + schedulerKey);

        // Compare and assert
        boolean passed = TestUtils.compareResults(result, expected, "Priority Preemptive Scheduler");
        assertTrue(passed, "Test failed - see error messages above");

        System.out.println("✓ " + schedulerKey + " test passed: " + filepath);
    }
}

```
---

## **RoundRobinSchedulerTest.java**

```java

package schedulers;

import core.Process;
import core.SchedulingResult;
import org.junit.jupiter.api.Test;
import utils.TestUtils;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Unit tests for Round Robin Scheduler
 */
public class RoundRobinSchedulerTest {

    @Test
    public void testCase1_BasicMixedArrivals() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_1.json", "RR");
    }

    @Test
    public void testCase2_AllArrivalsAtZero() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_2.json", "RR");
    }

    @Test
    public void testCase3_VariedBurstTimes() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_3.json", "RR");
    }

    @Test
    public void testCase4_LargeBurstsWithGaps() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_4.json", "RR");
    }

    @Test
    public void testCase5_ShortBurstsHighFrequency() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_5.json", "RR");
    }

    @Test
    public void testCase6_MixedScenario() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_6.json", "RR");
    }

    /**
     * Helper method to run a test from a JSON file
     */
    private void runTestFromFile(String filepath, String schedulerKey) throws IOException {
        // Load test case
        TestUtils.TestCase testCase = TestUtils.loadTestCase(filepath);

        // Make copies of processes for the test
        List<Process> processes = new ArrayList<>();
        for (Process p : testCase.processes) {
            processes.add(p.copy());
        }

        // Run scheduler
        RoundRobinScheduler scheduler = new RoundRobinScheduler();
        scheduler.run(processes, testCase.contextSwitch, testCase.rrQuantum);

        // Create result object
        SchedulingResult result = new SchedulingResult(
                scheduler.getSlices(),
                processes,
                null  // RR doesn't have quantum history
        );

        // Get expected results
        TestUtils.ExpectedResult expected = testCase.expectedOutput.get(schedulerKey);
        assertNotNull(expected, "No expected output found for " + schedulerKey);

        // Compare and assert
        boolean passed = TestUtils.compareResults(result, expected, "Round Robin Scheduler");
        assertTrue(passed, "Test failed - see error messages above");

        System.out.println("✓ " + schedulerKey + " test passed: " + filepath);
    }
}

```
---

## **SJFPreemptiveSchedulerTest.java**


```java

package schedulers;

import core.Process;
import core.SchedulingResult;
import org.junit.jupiter.api.Test;
import utils.TestUtils;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Unit tests for SJF Preemptive Scheduler
 */
public class SJFPreemptiveSchedulerTest {

    @Test
    public void testCase1_BasicMixedArrivals() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_1.json", "SJF");
    }

    @Test
    public void testCase2_AllArrivalsAtZero() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_2.json", "SJF");
    }

    @Test
    public void testCase3_VariedBurstTimes() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_3.json", "SJF");
    }

    @Test
    public void testCase4_LargeBurstsWithGaps() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_4.json", "SJF");
    }

    @Test
    public void testCase5_ShortBurstsHighFrequency() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_5.json", "SJF");
    }

    @Test
    public void testCase6_MixedScenario() throws IOException {
        runTestFromFile("src/test/resources/otherschedulers/test_6.json", "SJF");
    }

    /**
     * Helper method to run a test from a JSON file
     */
    private void runTestFromFile(String filepath, String schedulerKey) throws IOException {
        // Load test case
        TestUtils.TestCase testCase = TestUtils.loadTestCase(filepath);

        // Make copies of processes for the test
        List<Process> processes = new ArrayList<>();
        for (Process p : testCase.processes) {
            processes.add(p.copy());
        }

        // Run scheduler
        SJFPreemptiveScheduler scheduler = new SJFPreemptiveScheduler();
        scheduler.run(processes, testCase.contextSwitch, testCase.rrQuantum);

        // Create result object
        SchedulingResult result = new SchedulingResult(
                scheduler.getSlices(),
                processes,
                null  // SJF doesn't have quantum history
        );

        // Get expected results
        TestUtils.ExpectedResult expected = testCase.expectedOutput.get(schedulerKey);
        assertNotNull(expected, "No expected output found for " + schedulerKey);

        // Compare and assert
        boolean passed = TestUtils.compareResults(result, expected, "SJF Preemptive Scheduler");
        assertTrue(passed, "Test failed - see error messages above");

        System.out.println("✓ " + schedulerKey + " test passed: " + filepath);
    }
}

```

---

# **5. Json files (test/resources)**

---

## **test_1.json**

```java
{
  "name": "Test Case 1: Basic mixed arrivals",
  "input": {
    "contextSwitch": 1,
    "rrQuantum": 2,
    "agingInterval": 5,
    "processes": [
      {"name": "P1", "arrival": 0, "burst": 8, "priority": 3},
      {"name": "P2", "arrival": 1, "burst": 4, "priority": 1},
      {"name": "P3", "arrival": 2, "burst": 2, "priority": 4},
      {"name": "P4", "arrival": 3, "burst": 1, "priority": 2},
      {"name": "P5", "arrival": 4, "burst": 3, "priority": 5}
    ]
  },
  "expectedOutput": {
    "SJF": {
      "executionOrder": ["P1", "P2", "P4", "P3", "P2", "P5", "P1"],
      "processResults": [
        {"name": "P1", "waitingTime": 16, "turnaroundTime": 24},
        {"name": "P2", "waitingTime": 7, "turnaroundTime": 11},
        {"name": "P3", "waitingTime": 4, "turnaroundTime": 6},
        {"name": "P4", "waitingTime": 1, "turnaroundTime": 2},
        {"name": "P5", "waitingTime": 9, "turnaroundTime": 12}
      ],
      "averageWaitingTime": 7.4,
      "averageTurnaroundTime": 11.0
    },
    "RR": {
      "executionOrder": ["P1", "P2", "P3", "P1", "P4", "P5", "P2", "P1", "P5", "P1"],
      "processResults": [
        {"name": "P1", "waitingTime": 19, "turnaroundTime": 27},
        {"name": "P2", "waitingTime": 14, "turnaroundTime": 18},
        {"name": "P3", "waitingTime": 4, "turnaroundTime": 6},
        {"name": "P4", "waitingTime": 9, "turnaroundTime": 10},
        {"name": "P5", "waitingTime": 17, "turnaroundTime": 20}
      ],
      "averageWaitingTime": 12.6,
      "averageTurnaroundTime": 16.2
    },
    "Priority": {
      "executionOrder": ["P1", "P2", "P1", "P4", "P1", "P3", "P1", "P5", "P1"],
      "processResults": [
        {"name": "P1", "waitingTime": 18, "turnaroundTime": 26},
        {"name": "P2", "waitingTime": 1, "turnaroundTime": 5},
        {"name": "P3", "waitingTime": 12, "turnaroundTime": 14},
        {"name": "P4", "waitingTime": 6, "turnaroundTime": 7},
        {"name": "P5", "waitingTime": 16, "turnaroundTime": 19}
      ],
      "averageWaitingTime": 10.6,
      "averageTurnaroundTime": 14.2
    }
  }
}

```
---

## **test_2.json**

```java
{
  "name": "Test Case 2: All processes arrive at time 0",
  "input": {
    "contextSwitch": 1,
    "rrQuantum": 3,
    "agingInterval": 5,
    "processes": [
      {"name": "P1", "arrival": 0, "burst": 6, "priority": 3},
      {"name": "P2", "arrival": 0, "burst": 3, "priority": 1},
      {"name": "P3", "arrival": 0, "burst": 8, "priority": 2},
      {"name": "P4", "arrival": 0, "burst": 4, "priority": 4},
      {"name": "P5", "arrival": 0, "burst": 2, "priority": 5}
    ]
  },
  "expectedOutput": {
    "SJF": {
      "executionOrder": ["P5", "P2", "P4", "P1", "P3"],
      "processResults": [
        {"name": "P1", "waitingTime": 12, "turnaroundTime": 18},
        {"name": "P2", "waitingTime": 3, "turnaroundTime": 6},
        {"name": "P3", "waitingTime": 19, "turnaroundTime": 27},
        {"name": "P4", "waitingTime": 7, "turnaroundTime": 11},
        {"name": "P5", "waitingTime": 0, "turnaroundTime": 2}
      ],
      "averageWaitingTime": 8.2,
      "averageTurnaroundTime": 12.8
    },
    "RR": {
      "executionOrder": ["P1", "P2", "P3", "P4", "P5", "P1", "P3", "P4", "P3"],
      "processResults": [
        {"name": "P1", "waitingTime": 16, "turnaroundTime": 22},
        {"name": "P2", "waitingTime": 4, "turnaroundTime": 7},
        {"name": "P3", "waitingTime": 23, "turnaroundTime": 31},
        {"name": "P4", "waitingTime": 24, "turnaroundTime": 28},
        {"name": "P5", "waitingTime": 16, "turnaroundTime": 18}
      ],
      "averageWaitingTime": 16.6,
      "averageTurnaroundTime": 21.2
    },
    "Priority": {
      "executionOrder": ["P2", "P3", "P1", "P4", "P3", "P5", "P4"],
      "processResults": [
        {"name": "P1", "waitingTime": 11, "turnaroundTime": 17},
        {"name": "P2", "waitingTime": 0, "turnaroundTime": 3},
        {"name": "P3", "waitingTime": 15, "turnaroundTime": 23},
        {"name": "P4", "waitingTime": 25, "turnaroundTime": 29},
        {"name": "P5", "waitingTime": 24, "turnaroundTime": 26}
      ],
      "averageWaitingTime": 15.0,
      "averageTurnaroundTime": 19.6
    }
  }
}

```
---

## **test_3.json**

```java
{
  "name": "Test Case 3: Varied burst times with starvation risk",
  "input": {
    "contextSwitch": 1,
    "rrQuantum": 4,
    "agingInterval": 4,
    "processes": [
      {"name": "P1", "arrival": 0, "burst": 10, "priority": 5},
      {"name": "P2", "arrival": 2, "burst": 5, "priority": 1},
      {"name": "P3", "arrival": 5, "burst": 3, "priority": 2},
      {"name": "P4", "arrival": 8, "burst": 7, "priority": 1},
      {"name": "P5", "arrival": 10, "burst": 2, "priority": 3}
    ]
  },
  "expectedOutput": {
    "SJF": {
      "executionOrder": ["P1", "P2", "P3", "P5", "P4", "P1"],
      "processResults": [
        {"name": "P1", "waitingTime": 22, "turnaroundTime": 32},
        {"name": "P2", "waitingTime": 1, "turnaroundTime": 6},
        {"name": "P3", "waitingTime": 4, "turnaroundTime": 7},
        {"name": "P4", "waitingTime": 8, "turnaroundTime": 15},
        {"name": "P5", "waitingTime": 3, "turnaroundTime": 5}
      ],
      "averageWaitingTime": 7.6,
      "averageTurnaroundTime": 13.0
    },
    "RR": {
      "executionOrder": ["P1", "P2", "P1", "P3", "P4", "P2", "P5", "P1", "P4"],
      "processResults": [
        {"name": "P1", "waitingTime": 21, "turnaroundTime": 31},
        {"name": "P2", "waitingTime": 18, "turnaroundTime": 23},
        {"name": "P3", "waitingTime": 10, "turnaroundTime": 13},
        {"name": "P4", "waitingTime": 20, "turnaroundTime": 27},
        {"name": "P5", "waitingTime": 16, "turnaroundTime": 18}
      ],
      "averageWaitingTime": 17.0,
      "averageTurnaroundTime": 22.4
    },
    "Priority": {
      "executionOrder": ["P1", "P2", "P4", "P3", "P4", "P1", "P5", "P1"],
      "processResults": [
        {"name": "P1", "waitingTime": 24, "turnaroundTime": 34},
        {"name": "P2", "waitingTime": 1, "turnaroundTime": 6},
        {"name": "P3", "waitingTime": 9, "turnaroundTime": 12},
        {"name": "P4", "waitingTime": 6, "turnaroundTime": 13},
        {"name": "P5", "waitingTime": 13, "turnaroundTime": 15}
      ],
      "averageWaitingTime": 10.6,
      "averageTurnaroundTime": 16.0
    }
  }
}

```
---

## **test_4.json**

```java
{
  "name": "Test Case 4: Large bursts with gaps in arrivals",
  "input": {
    "contextSwitch": 2,
    "rrQuantum": 5,
    "agingInterval": 6,
    "processes": [
      {"name": "P1", "arrival": 0, "burst": 12, "priority": 2},
      {"name": "P2", "arrival": 4, "burst": 9, "priority": 3},
      {"name": "P3", "arrival": 8, "burst": 15, "priority": 1},
      {"name": "P4", "arrival": 12, "burst": 6, "priority": 4},
      {"name": "P5", "arrival": 16, "burst": 11, "priority": 2},
      {"name": "P6", "arrival": 20, "burst": 5, "priority": 5}
    ]
  },
  "expectedOutput": {
    "SJF": {
      "executionOrder": ["P1", "P4", "P6", "P2", "P5", "P3"],
      "processResults": [
        {"name": "P1", "waitingTime": 0, "turnaroundTime": 12},
        {"name": "P2", "waitingTime": 25, "turnaroundTime": 34},
        {"name": "P3", "waitingTime": 45, "turnaroundTime": 60},
        {"name": "P4", "waitingTime": 2, "turnaroundTime": 8},
        {"name": "P5", "waitingTime": 24, "turnaroundTime": 35},
        {"name": "P6", "waitingTime": 2, "turnaroundTime": 7}
      ],
      "averageWaitingTime": 16.33,
      "averageTurnaroundTime": 26.0
    },
    "RR": {
      "executionOrder": ["P1", "P2", "P1", "P3", "P4", "P2", "P5", "P1", "P6", "P3", "P4", "P5", "P3", "P5"],
      "processResults": [
        {"name": "P1", "waitingTime": 38, "turnaroundTime": 50},
        {"name": "P2", "waitingTime": 26, "turnaroundTime": 35},
        {"name": "P3", "waitingTime": 58, "turnaroundTime": 73},
        {"name": "P4", "waitingTime": 49, "turnaroundTime": 55},
        {"name": "P5", "waitingTime": 57, "turnaroundTime": 68},
        {"name": "P6", "waitingTime": 32, "turnaroundTime": 37}
      ],
      "averageWaitingTime": 43.33,
      "averageTurnaroundTime": 53.0
    },
    "Priority": {
      "executionOrder": ["P1", "P3", "P1", "P2", "P3", "P5", "P4", "P2", "P6", "P5", "P2"],
      "processResults": [
        {"name": "P1", "waitingTime": 14, "turnaroundTime": 26},
        {"name": "P2", "waitingTime": 65, "turnaroundTime": 74},
        {"name": "P3", "waitingTime": 16, "turnaroundTime": 31},
        {"name": "P4", "waitingTime": 38, "turnaroundTime": 44},
        {"name": "P5", "waitingTime": 48, "turnaroundTime": 59},
        {"name": "P6", "waitingTime": 44, "turnaroundTime": 49}
      ],
      "averageWaitingTime": 37.5,
      "averageTurnaroundTime": 47.17
    }
  }
}

```
---

## **test_5.json**

```java
{
  "name": "Test Case 5: Short bursts with high frequency",
  "input": {
    "contextSwitch": 1,
    "rrQuantum": 2,
    "agingInterval": 3,
    "processes": [
      {"name": "P1", "arrival": 0, "burst": 3, "priority": 3},
      {"name": "P2", "arrival": 1, "burst": 2, "priority": 1},
      {"name": "P3", "arrival": 2, "burst": 4, "priority": 2},
      {"name": "P4", "arrival": 3, "burst": 1, "priority": 4},
      {"name": "P5", "arrival": 4, "burst": 3, "priority": 5}
    ]
  },
  "expectedOutput": {
    "SJF": {
      "executionOrder": ["P1", "P4", "P2", "P5", "P3"],
      "processResults": [
        {"name": "P1", "waitingTime": 0, "turnaroundTime": 3},
        {"name": "P2", "waitingTime": 5, "turnaroundTime": 7},
        {"name": "P3", "waitingTime": 11, "turnaroundTime": 15},
        {"name": "P4", "waitingTime": 1, "turnaroundTime": 2},
        {"name": "P5", "waitingTime": 5, "turnaroundTime": 8}
      ],
      "averageWaitingTime": 4.4,
      "averageTurnaroundTime": 7.0
    },
    "RR": {
      "executionOrder": ["P1", "P2", "P3", "P1", "P4", "P5", "P3", "P5"],
      "processResults": [
        {"name": "P1", "waitingTime": 7, "turnaroundTime": 10},
        {"name": "P2", "waitingTime": 2, "turnaroundTime": 4},
        {"name": "P3", "waitingTime": 12, "turnaroundTime": 16},
        {"name": "P4", "waitingTime": 8, "turnaroundTime": 9},
        {"name": "P5", "waitingTime": 13, "turnaroundTime": 16}
      ],
      "averageWaitingTime": 8.4,
      "averageTurnaroundTime": 11.0
    },
    "Priority": {
      "executionOrder": ["P1", "P2", "P1", "P3", "P1", "P4", "P5", "P1"],
      "processResults": [
        {"name": "P1", "waitingTime": 17, "turnaroundTime": 20},
        {"name": "P2", "waitingTime": 1, "turnaroundTime": 3},
        {"name": "P3", "waitingTime": 5, "turnaroundTime": 9},
        {"name": "P4", "waitingTime": 10, "turnaroundTime": 11},
        {"name": "P5", "waitingTime": 11, "turnaroundTime": 14}
      ],
      "averageWaitingTime": 8.8,
      "averageTurnaroundTime": 11.4
    }
  }
}

```
---

## **test_6.json**

```java
{
  "name": "Test Case 6: Mixed scenario - comprehensive test",
  "input": {
    "contextSwitch": 1,
    "rrQuantum": 4,
    "agingInterval": 5,
    "processes": [
      {"name": "P1", "arrival": 0, "burst": 14, "priority": 4},
      {"name": "P2", "arrival": 3, "burst": 7, "priority": 2},
      {"name": "P3", "arrival": 6, "burst": 10, "priority": 5},
      {"name": "P4", "arrival": 9, "burst": 5, "priority": 1},
      {"name": "P5", "arrival": 12, "burst": 8, "priority": 3},
      {"name": "P6", "arrival": 15, "burst": 4, "priority": 6}
    ]
  },
  "expectedOutput": {
    "SJF": {
      "executionOrder": ["P1", "P2", "P4", "P6", "P5", "P3", "P1"],
      "processResults": [
        {"name": "P1", "waitingTime": 40, "turnaroundTime": 54},
        {"name": "P2", "waitingTime": 1, "turnaroundTime": 8},
        {"name": "P3", "waitingTime": 26, "turnaroundTime": 36},
        {"name": "P4", "waitingTime": 3, "turnaroundTime": 8},
        {"name": "P5", "waitingTime": 11, "turnaroundTime": 19},
        {"name": "P6", "waitingTime": 3, "turnaroundTime": 7}
      ],
      "averageWaitingTime": 14.0,
      "averageTurnaroundTime": 22.0
    },
    "RR": {
      "executionOrder": ["P1", "P2", "P1", "P3", "P4", "P2", "P5", "P1", "P6", "P3", "P4", "P5", "P1", "P3"],
      "processResults": [
        {"name": "P1", "waitingTime": 44, "turnaroundTime": 58},
        {"name": "P2", "waitingTime": 18, "turnaroundTime": 25},
        {"name": "P3", "waitingTime": 45, "turnaroundTime": 55},
        {"name": "P4", "waitingTime": 36, "turnaroundTime": 41},
        {"name": "P5", "waitingTime": 35, "turnaroundTime": 43},
        {"name": "P6", "waitingTime": 24, "turnaroundTime": 28}
      ],
      "averageWaitingTime": 33.67,
      "averageTurnaroundTime": 41.67
    },
    "Priority": {
      "executionOrder": ["P1", "P2", "P4", "P2", "P1", "P5", "P3", "P1", "P6", "P1"],
      "processResults": [
        {"name": "P1", "waitingTime": 43, "turnaroundTime": 57},
        {"name": "P2", "waitingTime": 8, "turnaroundTime": 15},
        {"name": "P3", "waitingTime": 31, "turnaroundTime": 41},
        {"name": "P4", "waitingTime": 1, "turnaroundTime": 6},
        {"name": "P5", "waitingTime": 16, "turnaroundTime": 24},
        {"name": "P6", "waitingTime": 36, "turnaroundTime": 40}
      ],
      "averageWaitingTime": 22.5,
      "averageTurnaroundTime": 30.5
    }
  }
}

```
---

## **AG_test1.json**

```java
{
  "input": {
    "processes": [
      {"name": "P1", "arrival": 0, "burst": 17, "priority": 4, "quantum": 7},
      {"name": "P2", "arrival": 2, "burst": 6, "priority": 7, "quantum": 9},
      {"name": "P3", "arrival": 5, "burst": 11, "priority": 3, "quantum": 4},
      {"name": "P4", "arrival": 15, "burst": 4, "priority": 6, "quantum": 6}
    ]
  },
  "expectedOutput": {
    "executionOrder": ["P1","P2","P3","P2","P1","P3","P4","P3","P1","P4"],
    "processResults": [
      {"name": "P1", "waitingTime": 19, "turnaroundTime": 36, "quantumHistory": [7,10,14,0]},
      {"name": "P2", "waitingTime": 4, "turnaroundTime": 10, "quantumHistory": [9,12,0]},
      {"name": "P3", "waitingTime": 10, "turnaroundTime": 21, "quantumHistory": [4,6,8,0]},
      {"name": "P4", "waitingTime": 19, "turnaroundTime": 23, "quantumHistory": [6,8,0]}
    ],
    "averageWaitingTime": 13.0,
    "averageTurnaroundTime": 22.5
  }
}

```
---

## **AG_test2.json**

```java
{
  "input": {
    "processes": [
      {"name": "P1", "arrival": 0, "burst": 10, "priority": 3, "quantum": 4},
      {"name": "P2", "arrival": 0, "burst": 8, "priority": 1, "quantum": 5},
      {"name": "P3", "arrival": 0, "burst": 12, "priority": 2, "quantum": 6},
      {"name": "P4", "arrival": 0, "burst": 6, "priority": 4, "quantum": 3},
      {"name": "P5", "arrival": 0, "burst": 9, "priority": 5, "quantum": 4}
    ]
  },
  "expectedOutput": {
    "executionOrder": ["P1","P2","P3","P2","P4","P3","P4","P3","P5","P1","P4","P1","P5","P4","P5"],
    "processResults": [
      {"name": "P1", "waitingTime": 25, "turnaroundTime": 35, "quantumHistory": [4,6,8,0]},
      {"name": "P2", "waitingTime": 3, "turnaroundTime": 11, "quantumHistory": [5,7,0]},
      {"name": "P3", "waitingTime": 11, "turnaroundTime": 23, "quantumHistory": [6,8,12,0]},
      {"name": "P4", "waitingTime": 33, "turnaroundTime": 39, "quantumHistory": [3,4,6,8,0]},
      {"name": "P5", "waitingTime": 36, "turnaroundTime": 45, "quantumHistory": [4,6,8,0]}
    ],
    "averageWaitingTime": 21.6,
    "averageTurnaroundTime": 30.6
  }
}

```
---

## **AG_test3.json**

```java
{
  "input": {
    "processes": [
      {"name": "P1", "arrival": 0, "burst": 20, "priority": 5, "quantum": 8},
      {"name": "P2", "arrival": 3, "burst": 4, "priority": 3, "quantum": 6},
      {"name": "P3", "arrival": 6, "burst": 3, "priority": 4, "quantum": 5},
      {"name": "P4", "arrival": 10, "burst": 2, "priority": 2, "quantum": 4},
      {"name": "P5", "arrival": 15, "burst": 5, "priority": 6, "quantum": 7},
      {"name": "P6", "arrival": 20, "burst": 6, "priority": 1, "quantum": 3}
    ]
  },
  "expectedOutput": {
    "executionOrder": ["P1","P2","P1","P4","P3","P1","P6","P5","P6","P1","P5"],
    "processResults": [
      {"name": "P1", "waitingTime": 17, "turnaroundTime": 37, "quantumHistory": [8,12,17,23,0]},
      {"name": "P2", "waitingTime": 1, "turnaroundTime": 5, "quantumHistory": [6,0]},
      {"name": "P3", "waitingTime": 7, "turnaroundTime": 10, "quantumHistory": [5,0]},
      {"name": "P4", "waitingTime": 1, "turnaroundTime": 3, "quantumHistory": [4,0]},
      {"name": "P5", "waitingTime": 20, "turnaroundTime": 25, "quantumHistory": [7,10,0]},
      {"name": "P6", "waitingTime": 3, "turnaroundTime": 9, "quantumHistory": [3,5,0]}
    ],
    "averageWaitingTime": 8.17,
    "averageTurnaroundTime": 14.83
  }
}


```
---

## **AG_test4.json**

```java
{
  "input": {
    "processes": [
      {"name": "P1", "arrival": 0, "burst": 3, "priority": 2, "quantum": 10},
      {"name": "P2", "arrival": 2, "burst": 4, "priority": 3, "quantum": 12},
      {"name": "P3", "arrival": 5, "burst": 2, "priority": 1, "quantum": 8},
      {"name": "P4", "arrival": 8, "burst": 5, "priority": 4, "quantum": 15},
      {"name": "P5", "arrival": 12, "burst": 3, "priority": 5, "quantum": 9}
    ]
  },
  "expectedOutput": {
    "executionOrder": ["P1","P2","P3","P2","P4","P5"],
    "processResults": [
      {"name": "P1", "waitingTime": 0, "turnaroundTime": 3, "quantumHistory": [10,0]},
      {"name": "P2", "waitingTime": 3, "turnaroundTime": 7, "quantumHistory": [12,17,0]},
      {"name": "P3", "waitingTime": 1, "turnaroundTime": 3, "quantumHistory": [8,0]},
      {"name": "P4", "waitingTime": 1, "turnaroundTime": 6, "quantumHistory": [15,0]},
      {"name": "P5", "waitingTime": 2, "turnaroundTime": 5, "quantumHistory": [9,0]}
    ],
    "averageWaitingTime": 1.4,
    "averageTurnaroundTime": 4.8
  }
}


```
---

## **AG_test5.json**

```java

{
  "input": {
    "processes": [
      {"name": "P1", "arrival": 0, "burst": 25, "priority": 3, "quantum": 5},
      {"name": "P2", "arrival": 1, "burst": 18, "priority": 2, "quantum": 4},
      {"name": "P3", "arrival": 3, "burst": 22, "priority": 4, "quantum": 6},
      {"name": "P4", "arrival": 5, "burst": 15, "priority": 1, "quantum": 3},
      {"name": "P5", "arrival": 8, "burst": 20, "priority": 5, "quantum": 7},
      {"name": "P6", "arrival": 12, "burst": 12, "priority": 6, "quantum": 4}
    ]
  },
  "expectedOutput": {
    "executionOrder": ["P1","P2","P1","P4","P3","P4","P2","P4","P5","P2","P1","P2","P6","P1","P3","P1","P5","P3","P6","P3","P5","P6","P5","P6"],
    "processResults": [
      {"name": "P1","waitingTime":40,"turnaroundTime":65,"quantumHistory":[5,7,10,14,16,0]},
      {"name": "P2","waitingTime":25,"turnaroundTime":43,"quantumHistory":[4,6,8,10,0]},
      {"name": "P3","waitingTime":63,"turnaroundTime":85,"quantumHistory":[6,8,11,16,0]},
      {"name": "P4","waitingTime":7,"turnaroundTime":22,"quantumHistory":[3,5,7,0]},
      {"name": "P5","waitingTime":77,"turnaroundTime":97,"quantumHistory":[7,10,14,16,0]},
      {"name": "P6","waitingTime":88,"turnaroundTime":100,"quantumHistory":[4,6,8,11,0]}
    ],
    "averageWaitingTime":50.0,
    "averageTurnaroundTime":68.67
  }
}

```
---

## **AG_test6.json**

```java

{
  "input": {
    "processes": [
      {"name": "P1","arrival":0,"burst":14,"priority":4,"quantum":6},
      {"name": "P2","arrival":4,"burst":9,"priority":2,"quantum":8},
      {"name": "P3","arrival":7,"burst":16,"priority":5,"quantum":5},
      {"name": "P4","arrival":10,"burst":7,"priority":1,"quantum":10},
      {"name": "P5","arrival":15,"burst":11,"priority":3,"quantum":4},
      {"name": "P6","arrival":20,"burst":5,"priority":6,"quantum":7},
      {"name": "P7","arrival":25,"burst":8,"priority":7,"quantum":9}
    ]
  },
  "expectedOutput": {
    "executionOrder":["P1","P2","P1","P4","P3","P2","P1","P5","P6","P5","P6","P3","P5","P7","P1","P3","P7","P3","P7"],
    "processResults":[
      {"name":"P1","waitingTime":39,"turnaroundTime":53,"quantumHistory":[6,8,11,15,0]},
      {"name":"P2","waitingTime":11,"turnaroundTime":20,"quantumHistory":[8,10,0]},
      {"name":"P3","waitingTime":45,"turnaroundTime":61,"quantumHistory":[5,7,10,14,0]},
      {"name":"P4","waitingTime":4,"turnaroundTime":11,"quantumHistory":[10,0]},
      {"name":"P5","waitingTime":19,"turnaroundTime":30,"quantumHistory":[4,6,8,0]},
      {"name":"P6","waitingTime":13,"turnaroundTime":18,"quantumHistory":[7,10,0]},
      {"name":"P7","waitingTime":37,"turnaroundTime":45,"quantumHistory":[9,12,17,0]}
    ],
    "averageWaitingTime":24.0,
    "averageTurnaroundTime":34.0
  }
}

```
