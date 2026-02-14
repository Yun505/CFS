# Completely Fair Scheduler (CFS) Implementation

A C implementation of the Linux Completely Fair Scheduler, demonstrating core operating system scheduling concepts and data structures.
https://youtu.be/KFnGH8oHJ5A
## Table of Contents

- [Overview](#overview)
- [What is CFS](#what-is-cfs)
- [The Problem CFS Solves](#the-problem-cfs-solves)
- [Core Concepts](#core-concepts)
  - [The Ideal Multitasking CPU](#the-ideal-multitasking-cpu)
  - [Virtual Runtime](#virtual-runtime)
  - [Weights and Priorities](#weights-and-priorities)
  - [Red-Black Tree Structure](#red-black-tree-structure)
- [How CFS Works](#how-cfs-works)
  - [Scheduling Algorithm](#scheduling-algorithm)
  - [Time Slice Calculation](#time-slice-calculation)
  - [Preemption](#preemption)
  - [Sleep and Wakeup Behavior](#sleep-and-wakeup-behavior)
- [Implementation Details](#implementation-details)
- [Building and Running](#building-and-running)
- [Design Decisions](#design-decisions)
- [Performance Characteristics](#performance-characteristics)
- [Comparison to Other Schedulers](#comparison-to-other-schedulers)

## Overview

This project implements the Completely Fair Scheduler (CFS), the default process scheduler in the Linux kernel since version 2.6.23. CFS replaced the O(1) scheduler and fundamentally changed how Linux allocates CPU time to processes.

This implementation was developed collaboratively via VS Code Live Share by Hayun, Mark, and Om.

**Key Achievement:** CFS models an ideal multitasking processor mathematically, ensuring long-term fairness without relying on fixed time slices.

## What is CFS

The Completely Fair Scheduler is designed around a single principle:

> Model an ideal multitasking CPU where each task receives an exactly proportional share of processing time based on its priority.

In an ideal world with infinite parallelism:
- If N tasks are runnable, each receives exactly 1/N of the CPU
- Higher priority tasks receive proportionally more time
- No task ever starves

Since real CPUs can only run one task at a time, CFS simulates this ideal by:
1. Tracking how much CPU time each task has received
2. Always selecting the task that is most "behind" in receiving its fair share
3. Normalizing time by task priority to ensure proportional fairness

## The Problem CFS Solves

Traditional schedulers face several challenges:

**Round Robin Problems:**
- Fixed time slices waste CPU on frequent context switches
- Doesn't account for task priority properly
- Poor handling of I/O-bound vs CPU-bound tasks

**Priority Queues Problems:**
- Can lead to starvation of low-priority tasks
- Complex heuristics needed to prevent unfairness
- Difficult to predict behavior under varying loads

**O(1) Scheduler Problems:**
- Required heuristics to distinguish interactive vs batch tasks
- Complex active/expired array management
- Could exhibit unfair behavior in edge cases

CFS solves these by making fairness the fundamental property, not an emergent behavior.

## Core Concepts

### The Ideal Multitasking CPU

Imagine a hypothetical CPU that can split its processing power infinitely:

```
Real CPU (sequential):     Ideal CPU (parallel):
┌───────┐                  ┌─┬─┬─┬─┐
│ Task A│                  │A│B│C│D│  All tasks run
├───────┤                  │ │ │ │ │  simultaneously
│ Task B│                  │A│B│C│D│  with 25% each
├───────┤                  │ │ │ │ │
│ Task C│                  │A│B│C│D│
├───────┤                  └─┴─┴─┴─┘
│ Task D│
└───────┘
```

On the ideal CPU, after time T:
- Each task has received exactly T/4 processing time
- No task is ahead or behind

CFS tracks deviation from this ideal and always schedules the task furthest behind.

### Virtual Runtime

Instead of tracking raw CPU time, CFS uses **virtual runtime (vruntime)**:

```c
// From scheduler.c line 84:
task->v_runtime += (scheduler->min_granularity * NICE_0) / nice_to_weight(task->nice);
```

Where:
- `NICE_0` = 1024 (weight of a normal priority task)
- `nice_to_weight(task->nice)` = weight from nice_array lookup
- `min_granularity` = actual time the task just ran

This can be rewritten as:
```
vruntime += actual_runtime × (NICE_0 / task_weight)
```

**Why virtual runtime?**

It normalizes time by priority:

```
High priority task (weight = 2048):
  vruntime += 10ms × (1024/2048) = 5ms virtual time
  
Low priority task (weight = 512):
  vruntime += 10ms × (1024/512) = 20ms virtual time
```

After running for the same physical time:
- High priority task's vruntime increases slowly
- Low priority task's vruntime increases quickly

This means:
- High priority tasks get selected more often
- Fairness is maintained proportionally to weight

### Weights and Priorities

Linux uses nice values from -20 (highest priority) to +19 (lowest priority), with 0 as default.

**Weight Mapping (from scheduler.c):**

```c
const int nice_array[40] = { 
    /* -20 */ 88761, 71755, 56483, 46273, 36291,
    /* -15 */ 29154, 23254, 18705, 14949, 11916,
    /* -10 */ 9548, 7620, 6100, 4904, 3906,
    /*  -5 */ 3121, 2501, 1991, 1586, 1277,
    /*   0 */ 1024, 820, 655, 526, 423,
    /*  +5 */ 335, 272, 215, 172, 137,
    /* +10 */ 110, 87, 70, 56, 45,
    /* +15 */ 36, 29, 23, 18, 15
};
```

**Key Values:**

| Nice Value | Weight | Relative CPU Share |
|------------|--------|-------------------|
| -20 | 88761 | ~86x |
| -10 | 9548 | ~9x |
| 0 | 1024 | 1x (baseline) |
| +10 | 110 | ~0.1x |
| +19 | 15 | ~0.015x |

The relationship is exponential:
- Each nice level difference ≈ 10% CPU difference
- Nice -1 gets ~10% more CPU than nice 0
- Nice +1 gets ~10% less CPU than nice 0

**Fairness Formula:**

If Task A has weight W_A and Task B has weight W_B:

```
CPU_share_A / CPU_share_B = W_A / W_B
```

Example with 2 tasks:
- Task A: nice 0, weight 1024
- Task B: nice 5, weight 335

Task A gets: 1024/(1024+335) ≈ 75% of CPU
Task B gets: 335/(1024+335) ≈ 25% of CPU

### Red-Black Tree Structure

CFS stores runnable tasks in a **red-black tree** ordered by vruntime.

**Why Red-Black Tree?**

| Operation | Complexity | Reason |
|-----------|-----------|---------|
| Insert task | O(log n) | Must maintain sorted order |
| Remove task | O(log n) | Efficient deletion |
| Find minimum vruntime | O(1)* | Cache leftmost node |
| Rebalance | O(log n) | Self-balancing property |

*The leftmost node (minimum vruntime) is cached, making access constant time.

**Tree Structure:**

```
                    vruntime=50
                   ┌─────┴─────┐
              vruntime=30    vruntime=70
              ┌────┴────┐        │
         vruntime=20  vruntime=40  vruntime=80
                │
         vruntime=15 ← leftmost (next to run)
```

The leftmost node always has the smallest vruntime and represents the task most deserving of CPU time.

### Two-Tree Architecture

This implementation uses an innovative two-tree design:

**1. Running Tree (`running_tree`):**
- Contains tasks that are currently runnable
- Sorted by `v_runtime` (lowest vruntime = leftmost)
- Used by `run_task()` to select next task to execute

**2. Schedule Tree (`schedule_tree`):**
- Contains tasks waiting for their arrival time
- Sorted by `arrival` time (earliest arrival = leftmost)
- Tasks move to running_tree when `arrival <= current_runtime`

**Why Two Trees?**

This allows the simulator to handle workloads where tasks arrive at different times:

```c
// From run_all_tasks():
while(next_schedule && next_schedule->metrics.arrival <= scheduler->runtime){
    rb_delete(scheduler->schedule_tree, next_schedule);
    add_task(scheduler, next_schedule);  // Move to running_tree
    next_schedule = /* get next */;
}
```

This design enables realistic simulations of task arrivals without pre-loading all tasks.

## How CFS Works

### Scheduling Algorithm

The core scheduling loop:

```
1. Select task with minimum vruntime (leftmost in tree)
2. Remove it from the tree
3. Run the task for its time slice
4. Update task's vruntime:
   vruntime += runtime × (NICE_0_LOAD / weight)
5. Reinsert task into tree (if still runnable)
6. Repeat
```

**Concrete Example:**

Initial state (4 tasks, all weight 1024):
```
Task A: vruntime = 100ms
Task B: vruntime = 120ms
Task C: vruntime = 110ms
Task D: vruntime = 105ms
```

Step 1: Select Task A (minimum = 100ms)
Step 2: Run Task A for 10ms
Step 3: Update: vruntime_A = 100 + 10 × (1024/1024) = 110ms
Step 4: Reinsert Task A

New state:
```
Task D: vruntime = 105ms  ← next to run
Task A: vruntime = 110ms
Task C: vruntime = 110ms
Task B: vruntime = 120ms
```

Over time, all vruntimes converge, ensuring fairness.

### Time Slice Calculation

Unlike Round Robin, CFS doesn't use fixed time slices. Instead:

**Target Scheduling Latency:**
- Default: 6ms (sysctl_sched_latency)
- The period in which all runnable tasks should run at least once

**Per-Task Time Slice:**

```
timeslice = (sched_latency / nr_running) × (task_weight / total_weight)
```

Example with 4 tasks (latency = 20ms):
```
Task A (weight=1024): 20ms / 4 × (1024/4096) = 1.25ms
Task B (weight=1024): 1.25ms
Task C (weight=1024): 1.25ms  
Task D (weight=1024): 1.25ms
Total = 5ms each → Not quite right, see below
```

Actually, with equal weights:
```
Each gets: 20ms / 4 = 5ms
```

With different weights (A=2048, B=1024, C=1024, D=1024):
```
Total weight = 5120
Task A: 20ms × (2048/5120) = 8ms
Task B: 20ms × (1024/5120) = 4ms
Task C: 20ms × (1024/5120) = 4ms
Task D: 20ms × (1024/5120) = 4ms
```

**Minimum Granularity:**
- Default: 0.75ms (sysctl_sched_min_granularity)
- Prevents time slices from becoming too small when many tasks exist
- If calculated slice < min_granularity, use min_granularity instead

This ensures:
- Responsiveness when few tasks are running (larger slices)
- Prevents excessive context switching when many tasks exist

### Preemption

CFS is **fully preemptive**. A running task can be preempted if:

**Condition 1: New task with significantly lower vruntime arrives**
```
Running: Task A, vruntime = 100ms
Wakes:   Task B, vruntime = 50ms
→ Preempt Task A immediately
```

**Condition 2: Current task exceeds its ideal share**
```
Task has run for its calculated timeslice
→ Preempt and schedule next task
```

**Condition 3: Higher priority task becomes runnable**
```
Running: Task A (nice +10)
Wakes:   Task B (nice -10)
→ B's weight is much higher
→ B's vruntime grows slower
→ Likely has lower vruntime
→ Preempt A
```

Preemption ensures:
- Interactive tasks respond quickly
- No task monopolizes CPU
- Fair distribution is maintained

### Sleep and Wakeup Behavior

When a task sleeps (e.g., waiting for I/O):

**On Sleep:**
```
1. Task removed from run queue (red-black tree)
2. vruntime stops accumulating
3. Task enters wait queue
```

**On Wakeup:**
```
1. Task's vruntime hasn't changed during sleep
2. CPU-bound tasks have accumulated higher vruntime
3. Sleeping task now has relatively low vruntime
4. Reinserted into tree at favorable position
5. Likely scheduled soon
```

**Example:**

```
Time 0ms:
  Task A (CPU-bound): vruntime = 100ms
  Task B (I/O-bound): vruntime = 100ms

Task B sleeps for I/O...

Time 50ms:
  Task A: vruntime = 150ms (ran for 50ms)
  Task B: vruntime = 100ms (unchanged, sleeping)

Task B wakes up:
  Task B inserted with vruntime = 100ms
  Task A at vruntime = 150ms
  → Task B runs next (lower vruntime)
```

This behavior:
- Improves responsiveness for interactive tasks
- Rewards tasks that yield the CPU
- Maintains fairness over long term

**Special Case: Long Sleepers**

If a task sleeps for a very long time:
```
vruntime_min = minimum vruntime of all runnable tasks
vruntime_sleep = max(vruntime_old, vruntime_min - threshold)
```

This prevents:
- Tasks from getting excessive priority after long sleep
- Starvation of continuously running tasks

## Implementation Details

### Project Structure

```
CFS/
├── include/
│   ├── scheduler.h       # Scheduler interface and structures
│   └── task.h           # Task structures and comparison functions
├── libs/
│   └── rbtree/          # Red-black tree library
│       ├── rb.c
│       └── rb.h
├── src/
│   ├── scheduler.c      # Core CFS implementation
│   ├── task.c          # Task creation and comparison
│   └── main.c          # Simulator entry point and workload parser
├── workloads/          # Test workload files
│   ├── test_1.txt
│   ├── test_2.txt
│   └── workload_*.txt
└── CMakeLists.txt      # Build configuration
```

### Key Data Structures

**Task Structure (from task.h):**
```c
typedef struct task {
    unsigned long pid;                    // Process ID
    unsigned long v_runtime;              // Virtual runtime
    char nice;                            // Nice value (-20 to +19)
    char (*run)(unsigned long, struct task*); // Task execution function
    void *param;                          // Task-specific parameters
    task_metrics_t metrics;               // Performance metrics
} task_t;
```

**Task Metrics:**
```c
typedef struct task_metrics {
    unsigned long arrival;      // Arrival time
    unsigned long bursts;       // Number of execution bursts
    unsigned long first_run;    // Time of first execution
    unsigned long duration;     // Total execution time
    unsigned long completion;   // Completion time
} task_metrics_t;
```

**Scheduler Structure (from scheduler.h):**
```c
typedef struct scheduler {
    unsigned long time_quantum;         // Scheduling latency period
    unsigned long min_granularity;      // Minimum time slice
    unsigned long quantum;              // Current dynamic quantum
    unsigned long runtime;              // Global runtime counter
    unsigned long last_run_task;        // Last executed task PID
    
    struct rb_tree *running_tree;       // Red-black tree of runnable tasks
    struct rb_tree *schedule_tree;      // Red-black tree of future arrivals
    
    unsigned int completed_tasks_count; // Number of completed tasks
    task_t **completed_tasks;           // Array of completed tasks
} scheduler_t;
```

**Two-Tree Design:**

The implementation uses two red-black trees:

1. **running_tree**: Tasks sorted by `v_runtime` (currently runnable)
2. **schedule_tree**: Tasks sorted by `arrival` time (future arrivals)

This allows efficient handling of tasks that arrive at different times.

### Core Functions

**Initialize Scheduler:**
```c
void initialize(scheduler_t *scheduler)
{
    scheduler->running_tree = rb_create(compare_running_tasks, NULL);
    scheduler->schedule_tree = rb_create(compare_scheduled_tasks, NULL);
    scheduler->runtime = 0;
    scheduler->completed_tasks_count = 0;
    scheduler->completed_tasks = NULL;
    scheduler->last_run_task = -1;
}
```

**Add Task to Running Queue:**
```c
void add_task(scheduler_t *scheduler, task_t *task)
{
    rb_insert(scheduler->running_tree, task);
    // Recalculate dynamic quantum
    scheduler->quantum = scheduler->time_quantum / scheduler->running_tree->count;
    if(scheduler->quantum < scheduler->min_granularity){
        scheduler->quantum = scheduler->min_granularity;
    }
}
```

**Schedule Task for Future Arrival:**
```c
void schedule_task(scheduler_t *scheduler, task_t *task)
{
    rb_insert(scheduler->schedule_tree, task);
}
```

**Get Task with Minimum vruntime:**
```c
rb_node *get_min(rb_node *node){
    while(node->link[0] != NULL){
        node = node->link[0];  // Walk left to find minimum
    }
    return node;
}
```

**Run Next Task:**
```c
char run_task(scheduler_t *scheduler)
{
    // Get task with smallest v_runtime
    task_t *task = rb_delete(scheduler->running_tree, 
                             get_min(&scheduler->running_tree->root)->data);
    
    // Calculate max vruntime for this execution
    unsigned long max_vruntime = task->v_runtime + scheduler->quantum;
    
    // Execute task
    do {
        task->v_runtime += (scheduler->min_granularity * NICE_0) / 
                          nice_to_weight(task->nice);
        task->metrics.duration += scheduler->min_granularity;
        scheduler->runtime += scheduler->min_granularity;
    } while(task->run(...) == 0 && task->v_runtime < max_vruntime);
    
    // Reinsert or complete task
    // ...
}
```

**Task Comparison Functions (from task.c):**
```c
int compare_running_tasks(task_t *t1, task_t *t2)
{
    int diff = t1->v_runtime - t2->v_runtime;
    if(diff)
        return diff;
    else
        return t1->pid - t2->pid;  // Tiebreaker
}

int compare_scheduled_tasks(task_t *t1, task_t *t2)
{
    int diff = t1->metrics.arrival - t2->metrics.arrival;
    if (diff)
        return diff;
    else
        return t1->pid - t2->pid;  // Tiebreaker
}
```

## Building and Running

### Prerequisites
- GCC or Clang
- CMake 3.10+
- Make

### Build Instructions

```bash
# Clone repository
git clone https://github.com/Yun505/CFS.git
cd CFS

# Create build directory
mkdir build && cd build

# Generate build files
cmake ..

# Compile
make

# Run simulator
./cfs_simulator
```

### Running Workloads

```bash
# Run specific workload
./cfs_simulator workloads/test_1.txt

# Run different workloads
./cfs_simulator workloads/workload_01.txt
```

### Example Workload File

The first two lines specify scheduler parameters, followed by task definitions:

```
0.1              # time_quantum in seconds (converted to nanoseconds)
0.004            # min_granularity in seconds (converted to nanoseconds)

# Format: arrival_time nice_value duration
0.0 0 0.1        # Task arrives at 0s, nice 0, runs for 0.1s
0.01 -5 0.2      # Task arrives at 0.01s, nice -5, runs for 0.2s
0.02 10 0.15     # Task arrives at 0.02s, nice 10, runs for 0.15s
```

**Parameters:**
- `arrival_time`: When task becomes runnable (seconds)
- `nice_value`: Priority (-20 to +19, 0 is normal)
- `duration`: Total CPU time needed (seconds)

## Design Decisions

### Why Red-Black Trees Over Other Structures

**Alternatives Considered:**

| Structure | Insert | Delete | Find Min | Why Not Used |
|-----------|--------|--------|----------|--------------|
| Array | O(1) | O(n) | O(n) | Find min too slow |
| Sorted Array | O(n) | O(n) | O(1) | Insert/delete too slow |
| Heap | O(log n) | O(log n) | O(1) | Can't efficiently remove arbitrary tasks |
| Red-Black Tree | O(log n) | O(log n) | O(1)* | **Optimal balance** |

*With cached leftmost pointer

**Red-Black Tree Advantages:**
- Self-balancing ensures O(log n) worst-case
- Efficient arbitrary task removal (for sleeping tasks)
- Cache-friendly with leftmost pointer
- Well-understood properties

### Why Virtual Runtime Instead of Raw Time

**Problem with Raw Time:**
```
Task A (nice -10): runs for 100ms → gets 100ms credit
Task B (nice +10): runs for 100ms → gets 100ms credit
→ Equal treatment despite different priorities!
```

**Solution with Virtual Runtime:**
```
Task A (weight 9548): vruntime += 100ms × (1024/9548) ≈ 10.7ms
Task B (weight 110):  vruntime += 100ms × (1024/110) ≈ 930ms
→ Task B's vruntime grows much faster
→ Task A gets scheduled more often
→ Proportional to weight!
```

### Why Dynamic Time Slices

**Fixed Time Slices (Round Robin):**
- Wastes CPU on context switches when few tasks
- Too long slices hurt responsiveness when many tasks
- Doesn't adapt to workload

**Dynamic Calculation:**
```
Few tasks → Larger slices → Less overhead
Many tasks → Smaller slices → Better fairness
```

Adapts automatically to system load.

## Performance Characteristics

### Time Complexity

| Operation | Complexity | Notes |
|-----------|-----------|--------|
| Schedule next task | O(1) | Cached leftmost |
| Enqueue task | O(log n) | Tree insertion |
| Dequeue task | O(log n) | Tree deletion |
| Update vruntime | O(1) | Simple arithmetic |

Where n = number of runnable tasks.

### Space Complexity

```
O(n) where n = total number of tasks
```

Each task requires:
- Task control block: ~64 bytes
- Red-black tree node: ~24 bytes
- Total per task: ~88 bytes

### Scalability

**Best Case:**
- Few tasks (< 10): Excellent performance
- Tree operations fast
- Low overhead

**Typical Case:**
- Moderate tasks (10-1000): Good performance
- O(log n) operations still very fast
- log₂(1000) ≈ 10 operations

**Worst Case:**
- Many tasks (10,000+): Acceptable performance
- log₂(10000) ≈ 13 operations
- Context switching overhead becomes dominant factor

**Multi-core:**
- Each CPU has separate run queue
- Load balancing redistributes tasks
- Scales well to many cores

## Comparison to Other Schedulers

### vs Round Robin

| Aspect | Round Robin | CFS |
|--------|-------------|-----|
| Time Slice | Fixed | Dynamic |
| Priority Handling | Poor | Excellent |
| Fairness | Short-term only | Long-term guaranteed |
| Starvation | Possible with priorities | Impossible |
| Complexity | O(1) operations | O(log n) operations |
| Overhead | Low | Moderate |

### vs O(1) Scheduler

| Aspect | O(1) | CFS |
|--------|------|-----|
| Complexity | All O(1) | O(log n) for enqueue/dequeue |
| Fairness | Heuristic-based | Mathematically guaranteed |
| Interactive Detection | Complex heuristics | Emergent from vruntime |
| Code Complexity | High (~2000 LOC) | Lower (~1000 LOC) |
| Predictability | Low | High |

**Why CFS Replaced O(1):**
- Simpler implementation
- More predictable behavior
- Better fairness guarantees
- Fewer edge cases
- Easier to reason about

### vs Priority Scheduling

| Aspect | Priority Queue | CFS |
|--------|---------------|-----|
| Starvation | Possible | Impossible |
| Priority Handling | Direct | Via weight/vruntime |
| Fairness | Not guaranteed | Guaranteed |
| Implementation | Simple | Moderate |

**CFS Advantage:**
- Combines priority with fairness
- High priority tasks get more CPU but don't starve others
- Weight mechanism provides smooth degradation

## Key Insights

### What Makes CFS "Fair"

**Mathematical Definition:**

Over a sufficiently long time period T, each task i receives:

```
CPU_time_i = T × (weight_i / Σ weight_j)
                         j∈runnable
```

This is provably fair because:
1. Vruntime tracks deviation from ideal
2. Always scheduling minimum vruntime corrects deviations
3. Weights ensure proportional allocation

### Why Vruntime Never Decreases

```c
// This invariant is maintained:
vruntime_new >= vruntime_old
```

Reasons:
1. Ensures progress (tasks move forward in time)
2. Prevents gaming the scheduler
3. Simplifies reasoning about fairness

### The Role of min_vruntime

The run queue maintains `min_vruntime`:

```c
cfs_rq->min_vruntime = max(cfs_rq->min_vruntime, 
                            leftmost->vruntime)
```

This monotonically increasing value:
- Anchors the timeline
- Prevents overflow issues
- Helps with task migration between CPUs
- Handles sleeping tasks correctly

### Interactive vs Batch Tasks

CFS doesn't explicitly distinguish task types, but behavior emerges:

**Interactive Tasks:**
- Sleep frequently (waiting for user input)
- Wake with low vruntime (haven't accumulated time)
- Get scheduled quickly
- Appear responsive

**Batch Tasks:**
- Rarely sleep
- Continuously accumulate vruntime
- Get scheduled when interactive tasks sleep
- Utilize leftover CPU

This emergence is more robust than heuristic-based detection.

## Conclusion

This implementation demonstrates the core principles of the Completely Fair Scheduler:

1. **Mathematical Fairness**: Uses virtual runtime to ensure proportional CPU allocation
2. **Efficient Data Structures**: Red-black trees provide O(log n) operations
3. **Adaptive Behavior**: Dynamic time slices adjust to system load
4. **Simple Design**: Fairness emerges from simple rules, not complex heuristics

The CFS represents a significant advancement in scheduler design, trading some overhead (O(log n) vs O(1)) for provable fairness and simpler implementation.

## License

This is an educational project implementing concepts from the Linux kernel CFS. 

