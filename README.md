Each person gets a file template they can start coding in immediately, with all function signatures, class structures, and TODO markers already placed.

Everything is organized so integration by Person 5 will be smooth.

---

#  **PROJECT STRUCTURE**

```
src/
 ├── core/
 │    ├── Process.java ✓
 │    ├── SchedulerBase.java ✓
 │    ├── ExecutionSlice.java ✓
 │    ├── ResultFormatter.java
 │    └── GanttChartPrinter.java
 ├── schedulers/
 │    ├── SJFPreemptiveScheduler.java ✓
 │    ├── RoundRobinScheduler.java
 │    ├── PriorityPreemptiveScheduler.java
 │    └── AGScheduler.java
 ├── io/
 │    └── InputParser.java
 └── Main.java
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

public class ResultFormatter {

    public static void printTable(List<Process> processes) {
        System.out.println("---------------------------------------------------");
        System.out.println("Process | Waiting Time | Turnaround Time | Completion");
        System.out.println("---------------------------------------------------");

        for (Process p : processes) {
            System.out.printf("%-7s | %-12d | %-15d | %-10d\n",
                    p.name, p.waitingTime, p.turnaroundTime, p.completionTime);
        }
        System.out.println("---------------------------------------------------");

        double totalWait = 0, totalTurn = 0;
        for (Process p : processes) {
            totalWait += p.waitingTime;
            totalTurn += p.turnaroundTime;
        }

        System.out.println("Average Waiting Time = " + totalWait / processes.size());
        System.out.println("Average Turnaround Time = " + totalTurn / processes.size());
    }
}
```

---

## **core/GanttChartPrinter.java** (Person 4)

```java
package core;

import java.util.List;

public class GanttChartPrinter {

    public static void print(List<ExecutionSlice> slices) {
        System.out.println("\nGantt Chart:");
        for (ExecutionSlice s : slices) {
            System.out.print("| " + s.processName + " ");
        }
        System.out.println("|");

        for (ExecutionSlice s : slices) {
            System.out.print(s.start + "      ");
        }
        System.out.println(slices.get(slices.size() - 1).end);
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

```java
package schedulers;

import core.*;
import java.util.*;

public class PriorityPreemptiveScheduler extends SchedulerBase {

    @Override
    public void run(List<Process> processes, int contextSwitch) {
        slices = new ArrayList<>();

        // TODO:
        // - Use priority queue based on priority
        // - Aging must be applied
        // - Preempt when a higher-priority process arrives
        // - Add context switching

        computeMetrics(processes);
    }
}
```

---

## **schedulers/AGScheduler.java** (Person 4)

```java
package schedulers;

import core.*;
import java.util.*;

public class AGScheduler extends SchedulerBase {

    @Override
    public void run(List<Process> processes, int contextSwitch) {
        slices = new ArrayList<>();

        // TODO: Implement AG scheduling:
        // Phase 1: FCFS for 25%
        // Phase 2: Non-preemptive priority for next 25%
        // Phase 3: Preemptive SJF for remaining

        // Handle 4 quantum update rules
        // Track quantum history for each process

        computeMetrics(processes);
    }
}
```

---

# **3. IO Module (Person 2)**

---

## **io/InputParser.java**

```java
package io;

import core.Process;
import java.util.*;

public class InputParser {

    public static List<Process> readProcesses() {
        Scanner sc = new Scanner(System.in);
        List<Process> list = new ArrayList<>();

        System.out.print("Enter number of processes: ");
        int n = sc.nextInt();

        for (int i = 0; i < n; i++) {
            System.out.println("Process " + (i + 1));
            System.out.print("Name: ");
            String name = sc.next();

            System.out.print("Arrival Time: ");
            int a = sc.nextInt();

            System.out.print("Burst Time: ");
            int b = sc.nextInt();

            System.out.print("Priority: ");
            int p = sc.nextInt();

            System.out.print("Quantum (for AG/optional): ");
            int q = sc.nextInt();

            list.add(new Process(name, a, b, p, q));
        }

        return list;
    }
}
```

---

# **4. Integration File (Person 5)**

---

## **Main.java**

```java
import schedulers.*;
import core.*;
import io.*;
import java.util.*;

public class Main {
    public static void main(String[] args) {

        List<Process> processes = InputParser.readProcesses();

        System.out.print("Enter context switching time: ");
        Scanner sc = new Scanner(System.in);
        int cs = sc.nextInt();

        System.out.println("Choose scheduler:");
        System.out.println("1. SJF (Preemptive)");
        System.out.println("2. Round Robin");
        System.out.println("3. Priority (Preemptive w/ aging)");
        System.out.println("4. AG Scheduling");

        int choice = sc.nextInt();

        SchedulerBase scheduler = null;

        switch (choice) {
            case 1:
                scheduler = new SJFPreemptiveScheduler();
                break;
            case 2:
                scheduler = new RoundRobinScheduler();
                break;
            case 3:
                scheduler = new PriorityPreemptiveScheduler();
                break;
            case 4:
                scheduler = new AGScheduler();
                break;
            default:
                System.out.println("Invalid choice!");
                return;
        }

        scheduler.run(processes, cs);
        ResultFormatter.printTable(processes);
        GanttChartPrinter.print(scheduler.getSlices());
    }
}
```
