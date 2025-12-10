Each person gets a file template they can start coding in immediately, with all function signatures, class structures, and TODO markers already placed.

Everything is organized so integration by Person 5 will be smooth.

---

#  **PROJECT STRUCTURE**

```
src/
 ├── core/
 │    ├── Process.java
 │    ├── SchedulerBase.java
 │    ├── ExecutionSlice.java
 │    ├── ResultFormatter.java
 │    └── GanttChartPrinter.java
 ├── schedulers/
 │    ├── SJFPreemptiveScheduler.java
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
    public String name;
    public int arrivalTime;
    public int burstTime;
    public int remainingTime;
    public int priority;
    public int quantum;

    // Metrics
    public int waitingTime = 0;
    public int turnaroundTime = 0;
    public int completionTime = 0;
    
    public Process(String name, int arrival, int burst, int priority, int quantum) {
        this.name = name;
        this.arrivalTime = arrival;
        this.burstTime = burst;
        this.remainingTime = burst;
        this.priority = priority;
        this.quantum = quantum;
    }
}
```

---

## **core/SchedulerBase.java**

```java
package core;

import java.util.List;

public abstract class SchedulerBase {

    // Will store execution segments → for Gantt chart
    protected List<ExecutionSlice> slices;

    public abstract void run(List<Process> processes, int contextSwitch);

    // Helper for computing metrics AFTER scheduling finishes
    protected void computeMetrics(List<Process> processes) {
        for (Process p : processes) {
            p.turnaroundTime = p.completionTime - p.arrivalTime;
            p.waitingTime = p.turnaroundTime - p.burstTime;
        }
    }

    // Getter for integration
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
    public String processName;
    public int start;
    public int end;

    public ExecutionSlice(String name, int start, int end) {
        this.processName = name;
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

import core.*;
import java.util.*;

public class SJFPreemptiveScheduler extends SchedulerBase {

    @Override
    public void run(List<Process> processes, int contextSwitch) {
        slices = new ArrayList<>();

        // TODO: Implement preemptive SJF using a priority queue
        // Priority: shortest remaining time first

        // Steps:
        // 1. Use currentTime to track progress
        // 2. Add processes as they arrive
        // 3. Preempt if new process has smaller remaining time
        // 4. Record each execution slice
        // 5. Add context switch time after each preemption
        // 6. Set completion times

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
