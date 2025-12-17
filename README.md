Each person gets a file template they can start coding in immediately, with all function signatures, class structures, and TODO markers already placed.

Everything is organized so integration by Person 5 will be smooth.

---

#  **PROJECT STRUCTURE**

```
src/
├── main/
│   └── java/
│       ├── core/
│       │   ├── Process.java 
│       │   ├── SchedulerBase.java
│       │   ├── ExecutionSlice.java
│       │   └── ResultFormatter.java
│       ├── schedulers/
│       │   ├── SJFPreemptiveScheduler.java
│       │   ├── RoundRobinScheduler.java
│       │   ├── PriorityPreemptiveScheduler.java
│       │   └── AGScheduler.java
│       ├── io/
│       │   └── InputParser.java
│       └── Main.java
└── test/
    ├── java/
    │   └── schedulers/
    │       ├── SJFPreemptiveSchedulerTest.java
    │       ├── RoundRobinSchedulerTest.java
    │       ├── PriorityPreemptiveSchedulerTest.java
    │       └── AGSchedulerTest.java
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
        return new Process(name, arrivalTime, burstTime, priority, quantum);
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
}
```

---

## **core/ResultFormatter.java** (Person 3)

```java
package core;

import java.util.List;

/**
 * Formats and displays scheduling results in a clear, organized manner
 * Supports multiple output formats and detailed statistics
 */
public class ResultFormatter {
    
    /**
     * Main method to print comprehensive scheduling results
     * 
     * @param schedulerName Name of the scheduler algorithm used
     * @param processes List of processes with computed metrics
     * @param executionOrder List of execution slices (Gantt chart data)
     * @param contextSwitchTime Context switch duration used in scheduling
     */
    public static void printResults(String schedulerName,
                                   List<Process> processes,
                                   List<ExecutionSlice> executionOrder,
                                   int contextSwitchTime) {
        
        printHeader(schedulerName);
        printGanttChart(executionOrder);
        printProcessMetrics(processes);
        printStatistics(processes, executionOrder, contextSwitchTime);
    }
    
    /**
     * Prints a formatted header with scheduler name
     */
    private static void printHeader(String schedulerName) {
        System.out.println("\n" + "═".repeat(60));
        System.out.println("    CPU SCHEDULING SIMULATOR - " + schedulerName.toUpperCase());
        System.out.println("═".repeat(60));
    }
    
    /**
     * Prints Gantt chart showing execution order
     * Visual representation of process execution timeline
     */
    private static void printGanttChart(List<ExecutionSlice> executionOrder) {
        System.out.println("\nGANTT CHART (Execution Timeline):");
        System.out.println("─".repeat(60));
        
        if (executionOrder.isEmpty()) {
            System.out.println("No execution recorded.");
            return;
        }
        
        // Print process names in timeline format
        System.out.print("│ ");
        for (ExecutionSlice slice : executionOrder) {
            String name = slice.processName;
            int duration = slice.end - slice.start;
            
            // Special formatting for context switches and idle time
            if (name.equals("CS")) {
                System.out.print(" [CS] ");
            } else if (name.equals("IDLE")) {
                for (int i = 0; i < duration; i++) {
                    System.out.print("░░");
                }
            } else {
                for (int i = 0; i < duration; i++) {
                    System.out.print(name);
                }
                System.out.print(" ");
            }
        }
        System.out.println("│");
        
        // Print time markers below Gantt chart
        System.out.print("0");
        for (ExecutionSlice slice : executionOrder) {
            int time = slice.end;
            int spaces = (slice.end - slice.start) * 2 - 1;
            if (slice.processName.equals("CS")) {
                spaces = 4;
            }
            System.out.printf("%" + (spaces + 1) + "d", time);
        }
        System.out.println("\n" + "─".repeat(60));
    }
    
    /**
     * Prints detailed metrics table for each process
     * Shows arrival time, burst time, completion time, waiting time, and turnaround time
     */
    private static void printProcessMetrics(List<Process> processes) {
        System.out.println("\nPROCESS METRICS TABLE:");
        System.out.println("┌────────┬─────────┬───────┬────────────┬──────────────┬─────────────┐");
        System.out.println("│ Process│ Arrival │ Burst │ Completion │ Waiting Time │ Turnaround  │");
        System.out.println("├────────┼─────────┼───────┼────────────┼──────────────┼─────────────┤");
        
        int totalWaitingTime = 0;
        int totalTurnaroundTime = 0;
        
        for (Process p : processes) {
            System.out.printf("│ %-6s │ %-7d │ %-5d │ %-10d │ %-12d │ %-11d │\n",
                p.getName(),
                p.getArrivalTime(),
                p.getBurstTime(),
                p.getCompletionTime(),
                p.getWaitingTime(),
                p.getTurnaroundTime());
            
            totalWaitingTime += p.getWaitingTime();
            totalTurnaroundTime += p.getTurnaroundTime();
        }
        
        // Calculate averages
        double avgWaitingTime = (double) totalWaitingTime / processes.size();
        double avgTurnaroundTime = (double) totalTurnaroundTime / processes.size();
        
        System.out.println("├────────┼─────────┼───────┼────────────┼──────────────┼─────────────┤");
        System.out.printf("│ AVERAGE│         │       │            │ %-12.2f │ %-11.2f │\n",
            avgWaitingTime, avgTurnaroundTime);
        System.out.println("└────────┴─────────┴───────┴────────────┴──────────────┴─────────────┘");
    }
    
    /**
     * Prints summary statistics including context switch analysis
     */
    private static void printStatistics(List<Process> processes,
                                       List<ExecutionSlice> executionOrder,
                                       int contextSwitchTime) {
        System.out.println("\nPERFORMANCE STATISTICS:");
        System.out.println("┌──────────────────────────────────────────┬──────────────┐");
        
        // Calculate statistics
        long contextSwitchCount = countContextSwitches(executionOrder);
        int totalContextSwitchTime = (int) contextSwitchCount * contextSwitchTime;
        double cpuUtilization = calculateCpuUtilization(executionOrder);
        double throughput = calculateThroughput(processes, executionOrder);
        
        // Print each statistic
        System.out.printf("│ %-40s │ %-12d │\n", 
            "Total Processes", processes.size());
        
        System.out.printf("│ %-40s │ %-12.2f │\n",
            "Average Waiting Time", 
            processes.stream().mapToInt(Process::getWaitingTime).average().orElse(0));
        
        System.out.printf("│ %-40s │ %-12.2f │\n",
            "Average Turnaround Time",
            processes.stream().mapToInt(Process::getTurnaroundTime).average().orElse(0));
        
        System.out.printf("│ %-40s │ %-12d │\n",
            "Context Switches", contextSwitchCount);
        
        System.out.printf("│ %-40s │ %-12d │\n",
            "Total Context Switch Time", totalContextSwitchTime);
        
        System.out.printf("│ %-40s │ %-11.1f%% │\n",
            "CPU Utilization", cpuUtilization);
        
        System.out.printf("│ %-40s │ %-12.2f │\n",
            "Throughput (processes/100 units)", throughput);
        
        System.out.println("└──────────────────────────────────────────┴──────────────┘");
    }
    
    /**
     * Counts number of context switches in execution order
     */
    private static long countContextSwitches(List<ExecutionSlice> executionOrder) {
        return executionOrder.stream()
            .filter(slice -> slice.processName.equals("CS"))
            .count();
    }
    
    /**
     * Calculates CPU utilization percentage
     * (Useful CPU time / Total time) * 100
     */
    private static double calculateCpuUtilization(List<ExecutionSlice> executionOrder) {
        if (executionOrder.isEmpty()) return 0.0;
        
        int totalTime = executionOrder.get(executionOrder.size() - 1).end;
        if (totalTime == 0) return 0.0;
        
        int usefulTime = executionOrder.stream()
            .filter(slice -> !slice.processName.equals("IDLE") && !slice.processName.equals("CS"))
            .mapToInt(slice -> slice.end - slice.start)
            .sum();
        
        return ((double) usefulTime / totalTime) * 100;
    }
    
    /**
     * Calculates throughput: processes completed per 100 time units
     */
    private static double calculateThroughput(List<Process> processes, 
                                             List<ExecutionSlice> executionOrder) {
        if (executionOrder.isEmpty()) return 0.0;
        
        int totalTime = executionOrder.get(executionOrder.size() - 1).end;
        if (totalTime == 0) return 0.0;
        
        return ((double) processes.size() * 100) / totalTime;
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
        // Make a working copy to avoid modifying originals
        List<Process> workingProcesses = new ArrayList<>();
        for (Process p : processes) {
            workingProcesses.add(p.copy());
        }

        // Sort by arrival for efficient addition
        List<Process> arrivalOrder = new ArrayList<>(workingProcesses);
        arrivalOrder.sort(Comparator.comparingInt(Process::getArrivalTime));

        PriorityQueue<Process> readyQueue = new PriorityQueue<>(new ProcessComparator());
        Set<Process> pending = new HashSet<>(workingProcesses);  // Track unfinished processes

        int currentTime = 0;
        int arrivalIndex = 0;
        Process currentProcess = null;
        boolean isFirstSwitch = true;

        while (!pending.isEmpty()) {
            // Add all processes that have arrived by currentTime
            while (arrivalIndex < arrivalOrder.size() && arrivalOrder.get(arrivalIndex).getArrivalTime() <= currentTime) {
                readyQueue.add(arrivalOrder.get(arrivalIndex));
                arrivalIndex++;
            }

            // If ready queue is empty but more processes will arrive, idle until next arrival
            if (readyQueue.isEmpty() && arrivalIndex < arrivalOrder.size()) {
                int startIdle = currentTime;
                currentTime = arrivalOrder.get(arrivalIndex).getArrivalTime();
                addSlice("IDLE", startIdle, currentTime);
                continue;
            } else if (readyQueue.isEmpty()) {
                break;  // All done
            }

            // Check for preemption: if a better (shorter remaining) process is ready
            Process nextBest = readyQueue.peek();
            if (currentProcess != null && nextBest.getRemainingTime() < currentProcess.getRemainingTime()) {
                // Preempt: put current back in queue
                readyQueue.add(currentProcess);
                currentProcess = null;
            }

            // If no current process, select the next best and handle context switch
            if (currentProcess == null) {
                currentProcess = readyQueue.poll();

                // Add context switch time if not the first execution and switching
                if (!isFirstSwitch) {
                    int startCS = currentTime;
                    currentTime += contextSwitchTime;
                    addSlice("CS", startCS, currentTime);

                    // Add any processes that arrived during context switch
                    while (arrivalIndex < arrivalOrder.size() && arrivalOrder.get(arrivalIndex).getArrivalTime() <= currentTime) {
                        readyQueue.add(arrivalOrder.get(arrivalIndex));
                        arrivalIndex++;
                    }
                }
                isFirstSwitch = false;
            }

            // Determine how long to execute: until next arrival or process finishes
            int nextEventTime = (arrivalIndex < arrivalOrder.size()) ? arrivalOrder.get(arrivalIndex).getArrivalTime() : Integer.MAX_VALUE;
            int executeAmount = Math.min(currentProcess.getRemainingTime(), nextEventTime - currentTime);

            if (executeAmount <= 0) {
                continue;  // No execution possible, loop to handle arrivals
            }

            // Execute and record slice
            int start = currentTime;
            currentTime += executeAmount;
            currentProcess.decreaseRemaining(executeAmount);
            addSlice(currentProcess.getName(), start, currentTime);

            // If process finishes
            if (currentProcess.getRemainingTime() == 0) {
                currentProcess.setCompletionTime(currentTime);
                pending.remove(currentProcess);
                currentProcess = null;
            }
        }

        // Compute final metrics (waiting and turnaround)
        computeMetrics(workingProcesses);
    }
}
```

---

## **schedulers/RoundRobinScheduler.java** (Person 2)

not implemented yet

```java
package schedulers;

import core.*;
import java.util.*;

public class RoundRobinScheduler extends SchedulerBase {

    @Override
    public void run(List<Process> processes, int contextSwitch) {
        slices = new ArrayList<>();

        // TODO: implement RR with quantum from user input
        // Use a queue
        // Preempt every quantum
        // Add context switch time
        // Track execution slices

        computeMetrics(processes);
    }
}
```

---

## **schedulers/PriorityPreemptiveScheduler.java** (Person 3)

recommended implementation

```java
package schedulers;

import core.Process;
import core.SchedulerBase;

import java.util.*;

public class PriorityPreemptiveScheduler extends SchedulerBase {

    @Override
    public void run(List<Process> processes, int contextSwitchTime, int rrQuantum) {
        // rrQuantum not used here, but agingInterval is in test cases — use a default or ignore
        // Test cases use agingInterval, but assignment doesn't specify — use 5 as safe default
        final int AGING_INTERVAL = 5;

        List<Process> working = new ArrayList<>();
        for (Process p : processes) {
            working.add(p.copy());
        }

        // Sort for arrival handling
        List<Process> arrivals = new ArrayList<>(working);
        arrivals.sort(Comparator.comparingInt(Process::getArrivalTime));

        // Ready queue: lower priority number = higher priority
        PriorityQueue<Process> ready = new PriorityQueue<>(
            Comparator.comparingInt(Process::getPriority)
                      .thenComparingInt(Process::getArrivalTime)
        );

        int time = 0;
        int arrivalIdx = 0;
        Process current = null;
        boolean firstRun = true;

        while (true) {
            boolean allDone = working.stream().allMatch(p -> p.getRemainingTime() == 0);
            if (allDone) break;

            // Add arrived processes
            while (arrivalIdx < arrivals.size() && arrivals.get(arrivalIdx).getArrivalTime() <= time) {
                ready.offer(arrivals.get(arrivalIdx));
                arrivalIdx++;
            }

            // Idle if nothing ready
            if (ready.isEmpty() && arrivalIdx < arrivals.size()) {
                int idleStart = time;
                time = arrivals.get(arrivalIdx).getArrivalTime();
                addSlice("IDLE", idleStart, time);
                continue;
            }

            // Preemption check
            if (current != null && !ready.isEmpty() && ready.peek().getPriority() < current.getPriority()) {
                ready.offer(current);
                current = null;
            }

            // Select new process if none running
            if (current == null) {
                if (ready.isEmpty()) {
                    time++;
                    continue;
                }
                current = ready.poll();

                if (!firstRun && contextSwitchTime > 0) {
                    int csStart = time;
                    time += contextSwitchTime;
                    addSlice("CS", csStart, time);

                    // Add arrivals during CS
                    while (arrivalIdx < arrivals.size() && arrivals.get(arrivalIdx).getArrivalTime() <= time) {
                        ready.offer(arrivals.get(arrivalIdx));
                        arrivalIdx++;
                    }
                }
                firstRun = false;
            }

            // Aging: every AGING_INTERVAL time units, boost waiting processes
            if (time > 0 && time % AGING_INTERVAL == 0) {
                List<Process> temp = new ArrayList<>();
                while (!ready.isEmpty()) {
                    Process p = ready.poll();
                    p.setPriority(Math.max(0, p.getPriority() - 1)); // increase priority
                    temp.add(p);
                }
                ready.addAll(temp);
            }

            // Execute one unit
            int execStart = time;
            time++;
            current.decreaseRemaining(1);
            addSlice(current.getName(), execStart, time);

            // Check completion
            if (current.getRemainingTime() == 0) {
                current.setCompletionTime(time);
                current = null;
            }
        }

        computeMetrics(working);
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

    //keepin track of the quantum history of each process
    private Map<String, List<Integer>> quantumHistory = new LinkedHashMap<>();
    
    
    public void run(List<Process> processes, int contextSwitchTime, int rrQuantum) {
        // Initialize the slices list for recording execution intervals
        slices = new ArrayList<>();

        
        // Create copies of all processes to work with, avoiding modification of the original inputs
        List<Process> working = new ArrayList<>();
        for (Process p : processes) {
            Process copy = p.copy();
            working.add(copy);
            quantumHistory.put(copy.getName(), new ArrayList<>());
            quantumHistory.get(copy.getName()).add(copy.getQuantum());
        }

        //Sort processes by arrival time for correct initial scheduling
        working.sort(Comparator.comparingInt(Process::getArrivalTime));

        //Ready Queue
        Queue<Process> readyQueue = new LinkedList<>();
        int time = 0;  // Tracks current cpu time
        int index = 0; // Index to traverse the sorted process list
        boolean first = true; // Flag to handle first proces..no context switch initiallyy

        while (true) {
            //Check if all processes are done.. if yes, terminate the scheduling loop
            boolean allDone = true;
            for (Process p : working) {
                if (p.getRemainingTime() > 0) {
                    allDone = false;
                    break;
                }
            }
            if (allDone) break;

            //Add newly arrived processes to the ready queue
            while (index < working.size() && working.get(index).getArrivalTime() <= time) {
                if (working.get(index).getRemainingTime() > 0) {
                    readyQueue.add(working.get(index));
                }
                index++;
            }

            //Handle idle CPU when no processes are ready
            if (readyQueue.isEmpty()) {
                if (index < working.size()) {
                    addSlice("IDLE", time, working.get(index).getArrivalTime());
                    time = working.get(index).getArrivalTime();
                }
                continue;
            }

            //get the next process using FCFS from the ready queue
            Process current = readyQueue.poll();

            //Applying context switch time if not the first process
            if (!first && contextSwitchTime > 0) {
                addSlice("CS", time, time + contextSwitchTime);
                time += contextSwitchTime;

                //Add any newly arrived processes during the context switch
                while (index < working.size() && working.get(index).getArrivalTime() <= time) {
                    if (working.get(index).getRemainingTime() > 0) {
                        readyQueue.add(working.get(index));
                    }
                    index++;
                }
            }
            first = false;

            int quantum = current.getQuantum(); // Current process quantum
            int executed = 0; //Trackinng how much of the quantum has been executed

            //Compute phase lengths for FCFS, Priority, and SJF
            int fcfsLen = (int)Math.ceil(quantum * 0.25);
            int prioLen = (int)Math.ceil(quantum * 0.25);
            if (fcfsLen + prioLen > quantum) prioLen = quantum - fcfsLen;
            int sjfLen = quantum - (fcfsLen + prioLen);

            boolean preempted = false; //Flag to track if the process was preempted

            //FCFS Phase 
            for (int i = 0; i < fcfsLen && current.getRemainingTime() > 0; i++) {
                addSlice(current.getName(), time, time + 1); // Record execution slice
                current.decreaseRemaining(1); // Reduce remaining burst time
                time++;
                executed++;

                //Adding newly arrived processes during FCFS execution
                while (index < working.size() && working.get(index).getArrivalTime() <= time) {
                    if (working.get(index).getRemainingTime() > 0) {
                        readyQueue.add(working.get(index));
                    }
                    index++;
                }
            }

            //if process finished during FCFS so update its quantum and completion time
            if (current.getRemainingTime() == 0) {
                current.setCompletionTime(time);
                current.setQuantum(0);
                quantumHistory.get(current.getName()).add(0);
                continue;
            }

            //Priority Phase 
            for (int i = 0; i < prioLen && current.getRemainingTime() > 0 && !preempted; i++) {
                addSlice(current.getName(), time, time + 1);
                current.decreaseRemaining(1);
                time++;
                executed++;

                //add newly arrived processes during Priority execution
                while (index < working.size() && working.get(index).getArrivalTime() <= time) {
                    if (working.get(index).getRemainingTime() > 0) {
                        readyQueue.add(working.get(index));
                    }
                    index++;
                }

                //Preempt if a higher priority process is ready
                for (Process p : readyQueue) {
                    if (p.getPriority() < current.getPriority()) {
                        preempted = true;
                        int remainingQuantum = quantum - executed;
                        int newQuantum = current.getQuantum() + (int)Math.ceil(remainingQuantum / 2.0);
                        current.setQuantum(newQuantum);
                        quantumHistory.get(current.getName()).add(newQuantum);
                        readyQueue.add(current);
                        break;
                    }
                }
            }

            if (preempted) continue;

            if (current.getRemainingTime() == 0) {
                current.setCompletionTime(time);
                current.setQuantum(0);
                quantumHistory.get(current.getName()).add(0);
                continue;
            }

            //SJF Phase 
            for (int i = 0; i < sjfLen && current.getRemainingTime() > 0 && !preempted; i++) {
                addSlice(current.getName(), time, time + 1);
                current.decreaseRemaining(1);
                time++;
                executed++;

                // Adding newly arrived processses during SJF execution
                while (index < working.size() && working.get(index).getArrivalTime() <= time) {
                    if (working.get(index).getRemainingTime() > 0) {
                        readyQueue.add(working.get(index));
                    }
                    index++;
                }

                //Preempt if a shorter job is ready in the readyQueue
                for (Process p : readyQueue) {
                    if (p.getRemainingTime() < current.getRemainingTime()) {
                        preempted = true;
                        int remainingQuantum = quantum - executed;
                        int newQuantum = current.getQuantum() + remainingQuantum;
                        current.setQuantum(newQuantum);
                        quantumHistory.get(current.getName()).add(newQuantum);
                        readyQueue.add(current);
                        break;
                    }
                }
            }

            if (preempted) continue;

            //After quantum finished.. 4th case 
            if (current.getRemainingTime() == 0) {
                current.setCompletionTime(time);
                current.setQuantum(0);
                quantumHistory.get(current.getName()).add(0);
            } else {
                int newQuantum = current.getQuantum() + 2; // Increase quantum if process used all its quantum
                current.setQuantum(newQuantum);
                quantumHistory.get(current.getName()).add(newQuantum);
                readyQueue.add(current);
            }
        }

        //Compute waiting time ,turnaround time, and other metrics for all processes
        computeMetrics(working);
    }

    //Getter for quantum history
    public Map<String, List<Integer>> getQuantumHistory() {
        return quantumHistory;
    }

    //method to print quantum history of all processes in a good op format
    public void printQuantumHistory() {
        System.out.println("\nQuantum History:");
        for (String name : quantumHistory.keySet()) {
            List<Integer> hist = quantumHistory.get(name);
            System.out.print(name + " Quantum: ");
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

# **3. IO Module (Person 2)**

---

## **io/InputParser.java**

not implemented yet

```java
package io;

import core.Process;

import java.util.ArrayList;
import java.util.List;
import java.util.Scanner;

public class InputParser {

    /**
     * Reads input exactly as specified in the assignment:
     * 1. Number of processes
     * 2. Round Robin Time Quantum
     * 3. Context switching time
     * 4. For each process: Name, Arrival Time, Burst Time, Priority
     * 
     * Note: Quantum is NOT read from user input — it's only used internally for AG Scheduling
     */
    public static InputData readInput() {
        Scanner sc = new Scanner(System.in);

        System.out.print("Enter number of processes: ");
        int n = sc.nextInt();

        System.out.print("Enter Round Robin Time Quantum: ");
        int rrQuantum = sc.nextInt();

        System.out.print("Enter context switching time: ");
        int contextSwitch = sc.nextInt();

        List<Process> processes = new ArrayList<>();

        for (int i = 0; i < n; i++) {
            System.out.println("\nProcess " + (i + 1));
            System.out.print("Name: ");
            String name = sc.next();

            System.out.print("Arrival Time: ");
            int arrival = sc.nextInt();

            System.out.print("Burst Time: ");
            int burst = sc.nextInt();

            System.out.print("Priority: ");
            int priority = sc.nextInt();

            // Quantum is NOT read from user — default to 0, AG will manage it
            processes.add(new Process(name, arrival, burst, priority, 0));
        }

        return new InputData(processes, rrQuantum, contextSwitch);
    }

    // Wrapper class to return all input data together
    public static class InputData {
        public final List<Process> processes;
        public final int rrQuantum;
        public final int contextSwitchTime;

        public InputData(List<Process> processes, int rrQuantum, int contextSwitchTime) {
            this.processes = processes;
            this.rrQuantum = rrQuantum;
            this.contextSwitchTime = contextSwitchTime;
        }
    }
}
```

---

# **4. Integration File (Person 5)**

---

## **Main.java**

not implemented yet

```java
import core.*;
import io.InputParser;
import io.InputParser.InputData;
import schedulers.*;

import java.util.List;

public class Main {
    public static void main(String[] args) {
        // Read all input at once
        InputData input = InputParser.readInput();
        List<Process> originalProcesses = input.processes;
        int contextSwitchTime = input.contextSwitchTime;
        int rrQuantum = input.rrQuantum;

        System.out.println("\nChoose scheduler:");
        System.out.println("1. SJF (Preemptive)");
        System.out.println("2. Round Robin");
        System.out.println("3. Priority (Preemptive w/ aging)");
        System.out.println("4. AG Scheduling");

        java.util.Scanner sc = new java.util.Scanner(System.in);
        int choice = sc.nextInt();

        SchedulerBase scheduler = null;
        String schedulerName = "";

        switch (choice) {
            case 1:
                scheduler = new SJFPreemptiveScheduler();
                schedulerName = "Preemptive Shortest Job First (SJF)";
                break;
            case 2:
                scheduler = new RoundRobinScheduler();
                schedulerName = "Round Robin";
                break;
            case 3:
                scheduler = new PriorityPreemptiveScheduler();
                schedulerName = "Preemptive Priority (with Aging)";
                break;
            case 4:
                scheduler = new AGScheduler();
                schedulerName = "AG Scheduling";
                break;
            default:
                System.out.println("Invalid choice!");
                return;
        }

        // Run scheduler on a copy of processes
        scheduler.run(originalProcesses, contextSwitchTime, rrQuantum);

        // Print results using the beautiful formatter
        ResultFormatter.printResults(
            schedulerName,
            originalProcesses,
            scheduler.getSlices(),
            contextSwitchTime
        );

        // Special output for AG: Quantum History
        if (scheduler instanceof AGScheduler agScheduler) {
            agScheduler.printQuantumHistory();
        }
    }
}
}
```
