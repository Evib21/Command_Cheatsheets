## Section 1: Apache Spark - Workflow Breakdown

### Architecture Diagram:

![[Pasted image 20250625165454.png]]

```python
graph TD
	A[Your Machine: Scala Script]
	--> B[Driver Program - JVM]
	--> C[SparkContext]
	--> DAG[DAG: Logical Plan<br>filter → map → collect]
	--> D[Cluster Manager]
	  
	D --> E1[Worker Node A]
	D --> E2[Worker Node B]
	
	E1 --> F1[Executor 1<br>2 Threads]
	F1 --> T1[Thread 1 <br> Task for Partition 1]
	F1 --> T2[Thread 2 <br> Task for Partition 2]
	  
	E2 --> F2[Executor 2<br>2 Threads]
	F2 --> T3[Thread 1 <br> Task for Partition 3]
	F2 --> T4[Thread 2 <br> Task for Partition 4]  
	
	E2 --> F3[Executor 3<br>1 Thread]
	F3 --> T5[Thread 1 <br> Task for Partition 5]
	  
	T1 --> G[Apply: <br> filter → map]
	T2 --> G
	T3 --> G
	T4 --> G
	T5 --> G  
	
	G --> H[Driver collects results <br> returns to program]
	H --> A
```


### High-Level Overview

#### A.  No_Shuffle -> 1 Stage

- Job Work-Flow:
	- My Machine or Cluster Head Node
		- Run Script 
		- Job is triggered when  .action() method 
	- Driver Program
		- Creates .SparkContext()
			- Connects to Cluster Manager
			- Creates RDDs:
		        - RDD = Resilient Distributed Dataset
		        - Immutable, distributed collection of data
		        - Built lazily through transformations like `.map()`, `.filter()`
		        - Each RDD has **lineage** (its transformation history)
		        - These RDDs become the **nodes in the DAG**
			- Builds DAG
				- Flow of operation blueprint 
				- Stages *based* on Shuffles
					- Tasks per Stage
						- 1 Task <-> 1 Partition
	- Cluster Manager
		- YARN, Standalone, Kubernetes etc..
		- Cluster Agnostic
		- Resource Allocator
		- Driver --> Computational needs --> Cluster Manager --> Launch Worker Nodes
			- Worker nodes
				- Individual Computers *in* the Cloud
				- Physical or Virtual Machines
				- CPU, Memory and other resources pre-determined by the Cluster Manager
				- Within a Worker_Node
					- Executor
						- JVM process
						- Assigned 1 *or* more tasks
						- Threads (1 or *many*)
							- 1 Thread *per* Task
							- 1 Task *per* Partition
							- Task:
								- Task = smallest unit of work
								- Each Task = one partition’s full chain of operations	
								- non-shuffling `.methods()` applied *as* defined per DAG work-flow-logic
						- Executor Returns Results *to* Driver
	- Driver gathers all partition results (RDD → Array)
	- Data Returned to your Program, and the local script now has a full array of the Results.

#### No_Shuffle Diagram 

![[Pasted image 20250625215633.png]]

- graph TD
  A[My Machine or Head Node] --> B[Run Script]
  B --> C[Action Triggered]
  C --> D[Driver Program]
  D --> E[SparkContext]
  E --> F[Connects to Cluster Manager]
  E --> G[Builds DAG]
  G --> H[No Shuffle Detected]
  H --> I[1 Stage Created]
  I --> J[5 Tasks (1 per Partition)]
  F --> K[Cluster Manager]
  K --> L[Launches Worker Nodes]
  
  L --> M[Worker Node A]
  L --> N[Worker Node B]
  
  M --> O[Executor 1]
  N --> P[Executor 2]
  N --> Q[Executor 3]
  
  O --> R[Thread 1 → Task 1]
  O --> S[Thread 2 → Task 2]
  P --> T[Thread 1 → Task 3]
  P --> U[Thread 2 → Task 4]
  Q --> V[Thread 1 → Task 5]
  
  R --> W[Applies Transformations]
  S --> W
  T --> W
  U --> W
  V --> W
  
  W --> X[Executor Returns Results to Driver]
  X --> Y[Driver Collects All Results]
  Y --> Z[Data Returned to Script]


####
---

#### B.  With_Shuffle -> Multiple Stages

- Job Work-Flow:
	- My Machine or Cluster Head Node
		- Run Script 
		- Job is triggered when `.action()` method is executed
	- Driver Program
		- Creates `.SparkContext()`
			- Connects to Cluster Manager
			- Creates RDDs:
		        - RDD = Resilient Distributed Dataset
		        - Immutable, distributed collection of data
		        - Built lazily through transformations like `.map()`, `.filter()`
		        - Each RDD has **lineage** (its transformation history)
		        - These RDDs become the **nodes in the DAG**
			- Initializes:
				- DAG Scheduler
				- Task Scheduler
			- Builds DAG
				- Flow of operations blueprint (transformation chain)
				- Lazy: built during `.transformation()` calls
				- Triggers execution only upon `.action()` call
				- DAG = **Logical Plan**
					- Directed (has direction: data flow)
					- Acyclic (no loops)
					- Graph (nodes = ops, edges = dependencies)
				- Handles:
					- Shuffle detection:
						- Wide transformation detected (e.g., groupByKey, reduceByKey, join)
						- Triggers need to **repartition and exchange data** across partitions/executors
					- Stage creation:
						- DAG is **split into Stages** based on shuffle boundaries
						- **Narrow dependencies** (e.g., map, filter) → same Stage
						- **Wide dependencies** (e.g., groupByKey) → boundary for new Stage
						- One Stage = group of Tasks that can run without needing data from other partitions
					- Task creation:
						- 1 Task ⇄ 1 Partition
						- Task = executes all transformations (filter → map, etc.) for its partition
	- Cluster Manager
		- YARN, Standalone, Kubernetes etc..
		- Cluster Agnostic
		- Resource Allocator
		- Driver → Computational needs → Cluster Manager → Launch Worker Nodes
			- Worker Nodes
				- Individual Computers *in* the Cloud
				- Physical or Virtual Machines
				- CPU, Memory and other resources pre-determined by the Cluster Manager
				- Within a Worker_Node
					- Executor
						- JVM process
						- Assigned 1 *or* more tasks
						- Threads (1 or *many*)
							- 1 Thread *per* Task
							- 1 Task *per* Partition
							- Task:
								- Task = smallest unit of work
								- Each Task = one partition’s full chain of operations	
								- Applies transformations *as* defined by the DAG for that Stage
								- If shuffle is involved:
									- Stage 1 (map side): 
										- Tasks process partitions
										- **Shuffle Write** → local disk → partitioned by hash
									- Stage 2 (reduce side):
										- Tasks **read** shuffled data from all other partitions (**Shuffle Read**)
										- Executes logic like `groupBy`, `reduceByKey`, `join`, etc.
						- Executor Returns Results *to* Driver
	- Driver gathers all partition results (RDD → Array)
	- Data Returned to your Program, and the local script now has a full array of the Results.

#### With_Shuffle Diagram 

![[Pasted image 20250625220316.png]]


- graph TD
  A[My Machine or Head Node] --> B[Run Script]
  B --> C[Action Triggered]
  C --> D[Driver Program]
  D --> E[SparkContext]
  E --> F[Connects to Cluster Manager]
  E --> G[Initializes DAG Scheduler]
  E --> H[Initializes Task Scheduler]
  E --> I[Builds DAG]
  I --> J[Shuffle Detected]
  J --> K[Split DAG into 2 Stages]
  
  K --> L[Stage 1: Map Side]
  L --> M[5 Tasks → Apply filter, map]
  M --> N[Shuffle Write to Disk]
  
  K --> O[Stage 2: Reduce Side]
  O --> P[5 Tasks → Shuffle Read]
  P --> Q[Apply groupBy, reduceByKey, etc]
  
  F --> R[Cluster Manager]
  R --> S[Launch Worker Nodes]
  
  S --> T[Worker Node A]
  S --> U[Worker Node B]
  
  T --> V[Executor 1]
  U --> W[Executor 2]
  U --> X[Executor 3]
  
  V --> Y[Thread 1 → Task]
  V --> Z[Thread 2 → Task]
  W --> AA[Thread 1 → Task]
  W --> AB[Thread 2 → Task]
  X --> AC[Thread 1 → Task]
  
  Q --> AD[Executors Return Results]
  AD --> AE[Driver Collects All Results]
  AE --> AF[Data Returned to Script]



####
---

### Concepts Deep Dive

```sql
.No_Shuffle:
	- MY MACHINE 
		- (Local development machine or cluster head node)
		- Scala Script:
			val data = sc.textFile("data.txt")    // Load file into RDD (lazy)
			val result = data
			    .filter(_.contains("error"))      // Transformation 1 (lazy)
			    .map(_.toUpperCase)               // Transformation 2 (lazy)
			    .collect()                        // Action (triggers execution)
		- This script is submitted to Spark using spark-submit or similar tool.
		- The moment .collect() is called, Spark begins execution.
	- DRIVER PROGRAM 
		- (runs on your machine or in the cluster)
		- Your code runs inside the DRIVER program (a JVM process).
		- The Driver creates a .SparkContext() object -> the entry point to Spark.
		- .SparkContext():
		    - Connects to the **Cluster Manager**
		    - Builds a **DAG** of your transformations
		    - Splits the DAG into **1 stage** (no shuffle detected)
		    - Breaks the stage into **5 tasks** (1 per partition)
	- CLUSTER MANAGER 
		- (e.g., YARN, Standalone, Kubernetes)
		- Cluster Manager is a .resource_allocator program.
		- It receives requests from the Driver:
		    - “I need 5 task slots, X CPU cores, Y GB of RAM.”
		- It launches .Worker_Nodes or assigns slots on existing ones.
	- WORKER NODES
		- (physical or virtual machines)
		- Cluster Manager assigns 2 Worker Nodes (A and B).
		- These are real machines with CPU, memory, etc.
		  
		  Worker Node A:
		    - Executor 1: JVM process with 2 threads
		      - Thread 1 → Task for Partition 1
		      - Thread 2 → Task for Partition 2
		
		  Worker Node B:
		    - Executor 2: JVM process with 2 threads
		      - Thread 1 → Task for Partition 3
		      - Thread 2 → Task for Partition 4
		    - Executor 3: JVM process with 1 thread
		      - Thread 1 → Task for Partition 5
	- EXECUTORS
		- (run inside Worker Nodes)
		- Executors are .JVMs started by the .Cluster_Manager.
		- Each Executor is assigned a portion of the work .one_or_more_tasks
		- Inside each Executor:
		    - Threads are created .one_per_task
		    - Each thread:
		        - Loads its 25-row partition
		        - Applies `.filter()` to remove non-error rows
		        - Applies `.map()` to convert to uppercase
		        - Returns result to the driver
	- TASKS
		- (smallest unit of work)
		- Each Task = one partition’s full chain of operations:
		    - Read 25 rows
		    - Apply filter → map (all within the same task/thread)
	- BACK TO DRIVER
	- Each executor sends its results back to the Driver.
	- Driver gathers all partition results (RDD → Array)
	- `collect()` finishes and returns data to your program
	- Your local script now has a full array of the results.
```

```sql
.With_Shuffle
	- MY MACHINE
		- (Local development machine or cluster head node)
		- Scala Script:
			val data = sc.textFile("data.txt")       /Load file into RDD (lazy)
			val pairs = data
				.filter(_.contains("error"))         /T 1 (lazy, narrow)
			    .map(line => (line.split(" ")(0), 1))/T 2 (lazy, narrow)
			val grouped = pairs
				.groupByKey()                        /T 3 (❗lazy, wide)
				.mapValues(_.sum)                    /T 4 (lazy, narrow)
				.collect()                           /Action (triggers execution)
		- This script is submitted to Spark using spark-submit or similar tool.
		- The moment .collect() is called, Spark begins execution.
	- DRIVER PROGRAM
		- (runs on your machine or on a designated driver node in cluster)
		- Your code runs inside the DRIVER program (a JVM process).
		- The Driver creates a .SparkContext() object → the entry point to Spark.
		- .SparkContext():
			- Connects to the **Cluster Manager**
			- Builds a **DAG** of your transformations
			- Analyzes dependencies:
				- filter → map → (narrow dependencies)
				- groupByKey → (❗wide dependency, shuffle required)
				- mapValues → collect → (narrow dependencies)
			- Splits the DAG into **2 stages**:
				- Stage 1: filter + map
				- Stage 2: groupByKey + mapValues
			- Breaks each stage into **5 tasks** (one per partition)
			- Schedules Stage 1 tasks on available executors
	- CLUSTER MANAGER
		- (e.g., YARN, Standalone, Kubernetes)
		- Cluster Manager is a .resource_allocator program.
		- It receives requests from the Driver:
			- “I need 5 task slots, X CPU cores, Y GB of RAM.”
		- It launches .Worker_Nodes or assigns slots on existing ones.
	- WORKER NODES
		- (physical or virtual machines with CPU, memory, etc.)
		  
		  Worker Node A:
		    - Executor 1: JVM process with 2 threads
		      - Thread 1 → Task for Partition 1
		      - Thread 2 → Task for Partition 2
		
		  Worker Node B:
		    - Executor 2: JVM process with 2 threads
		      - Thread 1 → Task for Partition 3
		      - Thread 2 → Task for Partition 4
		    - Executor 3: JVM process with 1 thread
		      - Thread 1 → Task for Partition 5
	- STAGE 1 EXECUTION (MAP SIDE)
		- Each executor loads its assigned partition (25 rows each).
		- Each Task:
			- Applies `.filter()` to remove non-error rows
			- Applies `.map()` to extract key and emit (key, 1)
			- Writes result to local disk (temporary shuffle files)
				- Output is hashed by key into buckets
				- Each bucket corresponds to a reducer in Stage 2
	- SHUFFLE PHASE (DATA MOVEMENT)
		- Spark performs a **shuffle read**:
			- Each Stage 2 task pulls its needed buckets (for specific key ranges)
			- Data is fetched across the network from multiple executors
			- All values for the same key are gathered together (grouped)
	- STAGE 2 EXECUTION (REDUCE SIDE)
		- Each reduce-side Task:
			- Reads its corresponding buckets from all 5 shuffle files
			- Performs `groupByKey` (rebuilds key → Seq[value])
			- Applies `.mapValues(_.sum)`
			- Sends final output (e.g., (key, total)) back to Driver
	- BACK TO DRIVER
		- Driver gathers all partition results into memory
		- `collect()` returns an Array of key-value pairs
		- Your local Scala program receives the full result
```

#### 1. Write Spark Code (User Side)

You write code in **Scala, Python, Java, or R** using:
- **RDD API**
- **DataFrame API**
- **SQL API**

#### 2. Submit Spark Application

Submit **your job** using `spark-submit` script:
- `spark-submit --master yarn --deploy-mode cluster your_script.jar`

**Two Deploy Modes:**
- `client`: Your driver runs on your local machine
- `cluster`: The driver runs inside the cluster (preferred for production).

This is the **entry point** to all runtime coordination.

#### 3. SparkContext Initialization

At the start of your app, this line runs:
- `val sc = new SparkContext(conf)`

This:
- Loads your config.
- Connects to the **Cluster Manager**.
- Initializes internal components like the **DAG Scheduler** and **Task Scheduler**.

**SparkContext is the runtime gateway** — it creates everything else and manages the job lifecycle.
- **Runtime Gateway:**
	-  *“The primary interface between your code and the system that runs your code.”*
- 
##### *A. What is SparkContext Exactly?*
**`SparkContext`** is a **Scala class** (from the `org.apache.spark` package - *Spark **Core** module*) that serves as the **entry point to Spark** for low-level RDD-based APIs.

##### *B. What Does `SparkContext` Do?*
Think of `SparkContext` as the **"Spark runtime gateway"** for your application. It:
1. **Initializes** Spark's execution environment.
2. **Connects** to the cluster manager (e.g., local, YARN, Mesos, Kubernetes).
3. **Allocates resources** (executors, memory, cores).
4. **Creates RDDs**, handles job DAGs, and dispatches tasks.
5. **Manages communication** between:
    - Driver (your program)
    - Cluster manager (YARN, standalone, etc.)
    - Executors (worker processes)

##### *C. What Happens Under the Hood?*
- You Create a new Instance of `SparkContext()` Class:
	- `val sc = new SparkContext("local[*]", "RatingsCounter")`
- It Initializes a new Instance of  `Configuration()` Class And:
	- Calls its Methods:
		- `setMaster()` - Where the job will be done (Local or Cluster)
		- `setAppName()` - How Spark Executor will identify the specific job! (Usually takes name of main function for user-readability and clarity)
	- `val conf = new SparkConf().setMaster("local[*]").setAppName("RatingsCounter")`
	- Both `SparkContext()` and  `Configuration()` are part of Spark-Core-Module
		- *(so, already created for us)*
- Cluster Manager Connection:
	Based on `setMaster()`:
	- `"local[*]"`: Run locally using all cores.
	- `"yarn"`: Connect to Hadoop's YARN Resource Manager.
	- `"spark://..."`: Connect to a standalone Spark cluster.
	**SparkContext** contacts the **Cluster Manager** (or acts as one in local mode).
- Executor Allocation:
	- The cluster manager assigns **executors** (JVM processes on worker nodes).
	- Executors run your code and store data in memory/disk.
- **Task Scheduling & DAGs:**
	- ***As you build*** - RDDs (via `map`, `filter`, etc.), 
	- ***SparkContext builds*** - a **DAG (Directed Acyclic Graph)** of transformations.
	- When you trigger an **action** (like `count()`), it:
		- Submits a **job** to the DAG Scheduler.
		- The **DAG Scheduler** splits it into **stages**.
		- The **Task Scheduler** turns those into **tasks**, which are sent to the **executors**.
- Tracking + Metadata:
	SparkContext:
	- Keeps track of all RDDs and their lineage.
	- Coordinates caching/persistence.
	- Talks to the Spark UI for monitoring.

##### *D. Is SparkContext Connected to Other Components?*
	
- Yes — think of it like a central router:
	
| **Component**       | **Role**                                    | **SparkContext’s Relation**             |
| ------------------- | ------------------------------------------- | --------------------------------------- |
| **Driver Program**  | Your main Scala/Java app                    | Created by your code                    |
| **Cluster Manager** | Allocates resources (cores/memory on nodes) | SparkContext talks to it to get workers |
| **Executors**       | Run the code and store data                 | SparkContext sends tasks to them        |
| **DAG Scheduler**   | Builds job stages from your RDD actions     | SparkContext invokes it                 |
| **Task Scheduler**  | Converts stages to tasks, schedules them    | SparkContext coordinates with it        |
| **Spark UI**        | Web UI for monitoring jobs/stages/executors | Powered by SparkContext’s metadata      |

##### *E. Visualization (High-Level Architecture):*
              [Your Scala Code] 
                      |
               [SparkContext] 
                      |
    ----------------------------------
    |               |               |
[DAG Scheduler] [Task Scheduler] [Task Scheduler]     [Cluster Manager]      
                      |
              [Executors on Nodes]

##### *F. Other Relevant Insights:*
1. **One SparkContext Per JVM**
	- You can **only create one SparkContext per JVM**.
	- This is because it manages global state, ports, listeners, etc.
2. **RDD API vs. SparkSession**
	- Newer Spark versions use `SparkSession` (wrapper over `SparkContext` + SQLContext).
		- `val spark = SparkSession.builder().getOrCreate()`
		- `val sc = spark.sparkContext`
3. **Lazy Evaluation**
	- Spark doesn’t execute anything until you call an **action** (like `count()`, `collect()`).
	- All `.map`, `.filter`, etc., are **just building a DAG** inside the SparkContext.

##### *G. Summary Table:*

| Feature           | Description                                                 |
| ----------------- | ----------------------------------------------------------- |
| What is it?       | A class and the main entry point to Spark                   |
| Role              | Manages jobs, resources, DAGs, RDDs, execution              |
| Connects to       | Cluster manager, DAG scheduler, executors, Spark UI         |
| How it works      | Builds DAGs, schedules tasks, manages distributed execution |
| When it's created | Instantiated in your code to start a Spark app              |

#### 4. Cluster Resource Allocation

- Cluster Manager Options:
	- **YARN** (Yet Another Resource Negotiator): Common in Hadoop ecosystems.
		- What is YARN?
			- A general-purpose cluster resource manager.
			- Sits on top of Hadoop.
			- Controls how CPU & memory resources are allocated to applications like Spark.
	- **Kubernetes**: Cloud-native cluster manager.
	- **Standalone**: Spark’s built-in cluster manager.
	- **Mesos** (legacy).

-  What the Cluster Manager Does:
	- Accepts your app request (via SparkContext).
	- Finds available machines (called **nodes** or **worker nodes**).
	- Launches:
	    - 1 **Driver** (for your SparkContext)
	    - N **Executors** (that run tasks)

- **Cluster Manager Deep Dive**:

	- **What it is the Cluster Manager:**
		- It's a system that sits on top of your cluster infrastructure (a collection of machines). *Think of it as the central traffic controller for your distributed computing environment.*
		- It's responsible for allocating CPU, memory, and other resources to different applications that want to run on the cluster.
		- It acts as an intermediary between your Spark application (specifically the SparkContext and Driver) and the underlying cluster hardware.
		
	- **Key Functions of a Cluster Manager:**
		1. **Resource Allocation:** 
			- This is its primary role. 
			- When your Spark application starts, its SparkContext requests resources (like CPU cores and memory) from the cluster manager.
			- The cluster manager then finds available worker nodes and allocates these resources to create "executors" for your Spark application.
		2. **Task Scheduling (at a high level):** 
			- While the Spark Driver handles the fine-grained scheduling of individual tasks within your application, 
			- the cluster manager plays a role in deciding _where_ those executors (which run the tasks) should be launched on the cluster.
		3. **Scalability:** 
			- It enables Spark to scale across many nodes, from a few to thousands, 
			- by efficiently managing resource distribution.
		4. **Fault Tolerance (to some extent):** 
			- Cluster managers often have mechanisms to handle node failures. 
			- If a worker node goes down, the cluster manager can detect this and potentially 
				- **reallocate resources** or 
				- **restart failed components**.
		5. **Isolation:** 
			- Different Spark applications running on the same cluster 
			- are often isolated by the cluster manager, 
			- ensuring they don't interfere with each other's 
				- resources or
				- execution.
		6. **Multi-Framework Support:** 
			- Many cluster managers (like YARN or Mesos) are 
				- designed to ***manage resources for multiple distributed computing frameworks***, not just Spark. 
			- This allows you to ***run Spark alongside other applications*** (e.g., Hadoop MapReduce, Flink) on the ***same shared cluster***.
		
	- **Types of Cluster Managers in Spark:**
		> 
		> *Spark is designed to be "**agnostic**" to the underlying cluster manager, meaning it can work with several popular ones.*
		
		***The most common types include***:
		- **Standalone:** 
			- This is Spark's own simple cluster manager, included with the Spark distribution.
			- It's easy to set up for dedicated Spark clusters and is often used for development and smaller deployments.
		- **Apache YARN (Yet Another Resource Negotiator):** 
			- This is the resource manager ***used in the Hadoop ecosystem***.
			- If you're running Spark in a Hadoop environment, YARN is often the default or preferred choice as it ***integrates well with HDFS and other Hadoop components***.
		- **Apache Mesos:** 
			- A general-purpose distributed kernel that can manage various workloads, including Spark.
			- It's known for its fine-grained resource sharing.
		- **Kubernetes:** 
			- An open-source system for
				- automating deployment, 
				- scaling, and 
				- management of containerized applications.
			- Spark can run on Kubernetes, leveraging its container orchestration capabilities.
		- **Local Mode:** 
			- While not a true cluster manager in the distributed sense, 
			- "local mode" allows Spark to run on a single machine, 
			- simulating a cluster by using multiple threads or processes. 
			- This is primarily for development, testing, and learning.
	
	*In summary, the cluster manager is a critical component in a Spark ecosystem, providing the foundational resource management that allows Spark applications to run efficiently and scalably across a cluster of machines.*


#### 5. DAG Construction (Logical Plan)

- **Directed Acyclic Graph**
		
	- Spark builds a **graph of operations** based on your transformations 
		- (map, filter, join, etc.).
	- Each node = transformation
	- Each edge = data dependency
		
	- DAGs are lazy:
		- Spark builds the **blueprint** of how to execute your logic, 
		- **but doesn’t run it yet**.

- DAG Deeper Dive
		
	- What is a Directed Acyclic Graph (DAG)?
		- **Directed**:
			- Each **edge** (or arrow) between nodes has a **direction**, showing the **flow of data** or execution:  
			- e.g., node A → node B → node C.
		- **Acyclic**:
			- There are **no cycles** — you can't start at one node and loop back to it. 
			- This ensures that operations **have a clear start and end**.
		- **Graph**:
			- A **network** of **nodes (operations)** and **edges (dependencies)** representing how data flows from one operation to another.
		
	- What is a DAG _in Spark_?
		- In Apache Spark, the **DAG represents your entire job's logic** — all the transformations you've defined in your code — as a **graph of operations**.
			- **Nodes** = Spark transformations (`map`, `filter`, `groupBy`, etc.)
			- **Edges** = Dependencies between them (which operation depends on which output)
		- It's essentially a **blueprint of computation**.
		
	- Who Builds the DAG?
		- **Spark’s internal engine builds the DAG**, but here’s the step-by-step:
			1. **You write code**
				- In **Scala, Python, Java, or R** using the Spark API:
					```python
					rdd = sc.textFile("data.txt")
					         .map(lambda x: x.split(","))
					         .filter(lambda x: x[0] == "Boston")
					```
			2. **Spark’s API (RDD/DataFrame) captures your transformations**
				- These operations are **lazy** — nothing is executed immediately.
			3. **Spark constructs the DAG internally**
				- Using internal components like:
					- `RDD` lineage (for RDDs)
					- Catalyst Optimizer (for DataFrames/Datasets)
				- These components **record the sequence of transformations**, which are used to build the DAG.
		
	- Where does the DAG exist? (Where does it "live"?)
		- The DAG **lives in memory** inside the **Driver program** — the main controller in a Spark application.
			- It’s stored in objects like:
				- `RDD` lineage chains (for low-level RDD API)
				- `LogicalPlan` / `OptimizedLogicalPlan` (for DataFrames/Datasets via Catalyst)
			- These objects describe **what needs to be done**, not _how_ it will be executed yet.
		
	- How is the DAG built? (Through code? Library?) - Yes, the DAG is built:
		- **Through your code** (Scala, Python, Java, or R)
		- Using Spark’s **core libraries**:
		    - RDD API → uses **RDD lineage tracking**
		    - DataFrame API → uses the **Catalyst Optimizer** to construct and optimize a logical plan
        
		- **Under the hood**:
			- Spark is **written in Scala**, and all high-level APIs in other languages (Python, Java) call into the **Scala-based core engine**
			- Modules involved:
			    - `org.apache.spark.rdd.RDD` for RDD lineage
			    - `org.apache.spark.sql.catalyst.plans.logical` for logical plans (DAGs for SQL/DataFrames)
			    - Catalyst optimizer is written in Scala and constructs the DAGs for query execution
		
	- **Conceptually**, what _is_ a DAG at its core? 
		- At its most **fundamental level**, a DAG in Spark is:
		
		- A **mathematical graph structure** *that* describes 
			- A **series of computations** 
				- Where each step **depends on the previous**, *with* 
					- **no loops or cycles**, and 
					- a **clear execution flow**.
				
			```scala
			textFile("data.txt")
			    ↓
			   map(split)
			    ↓
			  filter(city == "Boston")
			    ↓
				count()
			```
				
			- Each arrow is a **dependency** between transformations. 
			- When `count()` is triggered (an **action**), Spark uses the DAG to know *how* to:
				- Read the data
				- Apply `map`
				- Apply `filter`
				- Count results
		
	- ***Summary*** KeyPoints:
			
		- **What is it?**
			- A graph of transformations with directed edges and no cycles
			
		- **Who builds it?**
			- Spark engine (based on your code)
			
		- **How is it built?**
			- Through Spark API, using Scala-based libraries
			
		- **Where does it live?**
			- In memory, inside the Spark Driver
			
		- **Why use it?**
			- To plan, optimize, and execute distributed computation efficiently

#### 6. Action Triggers Execution

- The moment an action is called, Spark begins real execution.
	
- Examples of actions:
	- `count()`
	- `collect()`
	- `saveAsTextFile()`
	
- These *tell* Spark:
	> "Okay, I’m done defining my job. Go execute the DAG now."

- Transformation Vs Action
	- Transformation
		- **Lazy**: They just define a _step in the computation_, but **don’t trigger execution**.
			- They return a **new RDD** or DataFrame that _remembers_ the transformation applied.
			- Spark just records it in the DAG (the blueprint).
		- Lazy (deferred) - Describes _what_ to do with the data 
			- `map()`
			- `filter()`
			- `flatMap()` 
			- `distinct()` 
			- `union()` 
			- `join()` 
	- Action 
		- Triggers Real Execution
			- Ask Spark to Materialize results by:
				- Bringing data to driver *or*
				- Writing in memory
		- Triggers job - Says “Okay Spark, run everything now”
			- `count()`
			- `collect()`
			- `first()`
			- `take(n)`
			- `reduce()`
			- `saveAsTextFile()`
		
	- Ex:
		- `filter()` → just _defines_ a transformation. 
			- It **returns another RDD**, but doesn’t need to compute anything yet.
		- `count()` → needs to **return a result** (a number), 
			- so Spark must **read the data and process everything** to compute that result.

#### 7. DAG Scheduler Breaks Job into Stages 

- High-Level
	- Spark’s **DAG Scheduler** splits your DAG into **stages**.
	- A **stage** is a set of tasks that can run together without shuffling data.
	- Stages are separated by **shuffle boundaries** (e.g., groupBy, join).
	
- Concepts 
	- Shuffle:
		- Happens when data needs to be **repartitioned** or **moved across executors**.
    
- Summary
	- DAG Scheduler analyzes the DAG and creates **Stage 1, Stage 2, Stage 3**, etc.


#### 8. Task Scheduler Assigns Tasks to Executors

Each stage is divided into **tasks** — one per data partition.

##### 🔹 Components:

- **Task Scheduler**: Sends tasks to executors.
    
- **Cluster Manager**: Already allocated the machines.
    
- **Executors**: JVM processes running on the workers.
    

🧠 Spark tries to **schedule tasks close to the data** (data locality).

---

#### 9. Executors Run the Tasks

Each executor:

- Reads its input partition (from HDFS, S3, etc.)
    
- Runs the transformation logic
    
- Stores intermediate results in **memory** (if cached) or **disk**
    
- May shuffle data to other executors
    

Executors also:

- Send **heartbeat signals** to the driver
    
- Log results
    
- Participate in **failure recovery**
    

---

#### 10. SparkContext Collects and Finalizes Output

Once all tasks are done:

- Results are either:
    
    - Returned to the **driver** (e.g., `collect()`)
        
    - Written to storage (e.g., `saveAsTextFile`)
        
- SparkContext ensures that:
    
    - All stages were completed.
        
    - Resources are released.
        
    - Metrics are logged (visible in Spark UI).


#### Summary Table

|Component|What it is|Where it lives|Role|
|---|---|---|---|
|**Scala Script**|Your program|Your machine|Defines logic|
|**Driver**|Spark controller process|Your machine or cluster|Orchestrates everything|
|**SparkContext**|Gateway to Spark cluster|Inside Driver|Builds DAG, requests resources|
|**Cluster Manager**|Resource allocator|Separate process|Launches executors on workers|
|**Worker Nodes**|Machines in the cluster|Physical or cloud|Run your code|
|**Executors**|JVMs inside workers|Launched by CM|Run tasks using threads|
|**Threads**|Parallel units of execution|Inside executors|Run 1 task each|
|**Task**|Work on 1 partition|On a thread|Applies transformations|
|**Partition**|Chunk of RDD (e.g., 25 rows)|Logical|Assigned to 1 task|
|**Stage**|Group of tasks (no shuffle)|Logical|Created by driver|
|**Job**|The entire process triggered by an action|Full pipeline|Collects final output|


#### How Spark Changes With Shuffle

|Aspect|No Shuffle|With Shuffle|
|---|---|---|
|**Stages**|1 stage|Multiple stages|
|**Intermediate Data**|In memory|Written to disk (shuffle files)|
|**Data Movement**|None (partition-local)|Across executors (network I/O)|
|**Tasks per stage**|1 per partition|1 per partition per stage|
|**Failure Recovery**|Rerun task only|Rerun shuffle read/write tasks|
|**Performance**|Fast|Slower (I/O & network heavy)|


---



## Section 2: Scala Crash Course

### Links to learning Scala:
1. [Scala Docs](https://docs.scala-lang.org/tour/tour-of-scala.html)
2. [Scala Official Page](https://www.scala-lang.org/)
3. [Git Hub - Scala Notes](https://github.com/jultty/scala-notes)
4. [Databricks - Scala Crash Course](chrome-extension://efaidnbmnnnibpcajpcglclefindmkaj/https://lintool.github.io/SparkTutorial/slides/day1_Scala_crash_course.pdf)
5. [learnscala.org](https://www.learnscala.org/)

---

### 1. Scala Basic Data Structures:

#### val --> Immutable
val hello: String = "Hello World"
val hello2 = hello + " + String added to immutable var"

// cannot do something like this:
// hello = "Something Else"


#### var --> Mutable
var hello3 = "Hello World"
hello3 = "Something Else"
println(hello3)


#### Data Types:
val truth: Boolean = true
var letterA: Char = 'a'

val numberOne: Int = 1
val pi: Double = 3.14159265
val piSinglePrecision: Float = 3.14159265f
// float = SinglePrecisiion ==> 7 decimal digits
// Double = DoublePrecisiion ==> 15–17 digits
// Without f, Scala assumes all decimal numbers are Double by default.

---

val bigNumber: Long = 123456789
val smallNumber: Byte = 127

**Big and Small Number:**
In Scala, Long and Byte are both integer types, but they differ in size and range:

- Long is a 64-bit signed integer, allowing it to store very large numbers—from -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807. So bigNumber: Long = 123456789 easily fits within that range.
- Byte is an 8-bit signed integer, with a much smaller range—from -128 to 127. So smallNumber: Byte = 127 is valid, but even 128 would overflow and cause an error.

***In short***: use Byte for tiny values (e.g. raw data, low-level hardware), and Long when you need to handle very large integers.

---

**Concat with + for ALL Data Types:**  
- println("Concat Diff Data Types with + operator " + numberOne + truth + letterA + pi + bigNumber)

---

**// f ans s  in println():**  
- println(f"Pi is about $piSinglePrecision%.3f")
- println(f"Zero padding on the left: $numberOne%05d")
- println(s"I can use the s prefix to use variables like $numberOne $truth $letterA")
- println(s"The s prefix isn't limited to variables; I can include any expression. Like ${1+2}")

---

**// RegEx:**
- val theUltimateAnswer: String = "To life, the universe, and everything is 42."
- val pattern = """.* ([\d]+).*""".r    //  RegEx
- val pattern(answerString) = theUltimateAnswer   //  take RegEx, apply to theUltimateAnswer, and insert into answerString
- val answer = answerString.toInt   //  Convert answerString to INT and assign to answer
- println(answer)   // print final result as INT

---

**// Boolean:**
- val isGreater = 1 > 2
- val isLesser = 1 < 2
- val impossible = isGreater & isLesser
- val logicalAnd = isGreater && isLesser    //  Logical Operator
- val anotherWay = isGreater || isLesser  
  

- val picard: String = "Picard"
- val bestCaptain: String = "Picard"
- val isBest = picard == bestCaptain    // == compares inside the string each value, so not the object ref



---

### 2. Flow Control (if/else):


#### // Flow Control (if/else):


// 1st example - Writing all in one line:
if (1 > 3) println("Clearly False") else println("The condition is false")

// 2nd example - More readable format:
if (1 > 3) {
println("Not supposed to execute")
} else {
println("Its false, therefore the else statement executed")
}

#### // CASE Statement with Match Keyword:
var number = 2
var number = 3
number match {
case 1 => println("One")
case 2 => println("Two")
case 3 => println("Three")
case _ => println("Something else")
}

#### // for loop
for (x <- 1 to 4) {
val squared = x * x
println(squared)
}
// Clarifying val squared inside the loop:
//You're not reassigning val squared multiple times — you're creating a brand new squared variable in each loop iteration, scoped only inside the {} block of that iteration.
//  🔍 What does that mean?
//val means the variable cannot be reassigned within the same scope.
//  But each time the loop runs, it creates a new scope.

#### // while loops:
var x = 10
while (x >= 0) {
println(x)
x -= 1
}
// Here we declare var instead of val because:
// var scope outside the while loop and:
// inside the while loop var gets reassigned
// this would not work with val

#### // do .. while loop:
x = 1
do { println(x); x+=1 } while (x <= 10)

#### // expressions:
{val x = 10; x + 20}
// the expression in {} is a function in and of itself
// we can see it returned the last value in the expression
// this happens implicity
// qe can also return explicity if we add return

// Last piece of code in the expression can be treated as its own entity
// ex:
println({val x = 10; x + 20})

#### // Exercise - Fib:
var fib: Vector[Int] = Vector(0, 1)
var i = 2
while (i < 10) {
val next_num = fib(i-1) + fib(i-2)
fib = fib :+  next_num
i = i + 1
}
println(fib)
//return fib --> error because you are not inside a function



---

### 3. Functions:


####  Functions:

**Scala is a functional programming language**

//  Format:
def squareIt(x: Int): Int = {
x * x
}
squareIt(3)

def cubeIt(x: Int): Int = { x * x * x }
cubeIt(3)
//  def <func_name> (parameter_name: parameter_type): return_type = {
//  function_body
//  }

#### //  Nested Functions:
def nestedFunction (x: Int, f: Int => Int): Int = {
f(x)
}
//  define <func_name>
//  Takes in:
//  param_name: return_type
//  func_name: param_type_of_this_func => return_type_of_this_func
//  nestedFunction (...): return_type = {function_body}
//  f(x) = the passed in function, which takes the passed in param

// Example - Nested Function:
val result = nestedFunction(2, cubeIt)
println(s"The result of cubeit(2) is $result")

#### //  Lambda Functions: Declare a function in-Line WITHOUT a Name
//  Also called:
//  Functions Literals
//  Anonymous Functions
nestedFunction(3, x => x * x * x )
// Instead of passing cubeit we define it in this line
//  The passed in function -> Created on the spot
println("The result of cubeit(3) is " + nestedFunction(3, x => x * x * x ) )

#### //  Passing a Mulltiline Expression in Lambda:
nestedFunction( 2, x => {val y = x * 2; y * y} )
println("The result of this Lambda is " + nestedFunction( 2, x => {val y = x * 2; y * y} ))

#### //  Exercise:
//  use .toUpperCase and Lambda
//  Test your String.toUpperCase function

### 4. Data Structures:


#### [1. Tuples](https://docs.scala-lang.org/tour/tuples.html)

##### //  Tuples - Are ONE-BASED Index:
val my_string_tuple = ("a", "b", "c", "d", "e")
println(my_string_tuple)

//  Indexed:
println(my_string_tuple._3)  // ONE-BASED Indexed ==> "c"

##### //  key-value pair:
val my_key_value_pair = ("key1" -> "value1", "key2" -> "value2")
println(my_key_value_pair._2)

//  println(my_key_value_pair."key1") ==> cannot do this, only by index

##### //  A tuple can have multiple data types inside:
val my_mixed_tuple = (1, "a", 2.0, true)
println(my_mixed_tuple)

#### [2. Immutable Lists](https://docs.scala-lang.org/overviews/collections-2.13/concrete-immutable-collection-classes.html#lists)

#####  Characteristics: 
- Singly
- Same Type
- 0-INDEXED


#####  Simple ex with for loop
val my_list = List(1, 2, 3, 4, 5)
for (i <- my_list) {
println(i)
}


#####  Function Literal to a list - We can use MAP
val my_list_2 = my_list.map(x => x * 2)
println(my_list_2)


##### reduce() --> combine together all items in a collection using a func
val my_list_3 = my_list.reduce((x, y) => x + y)
println(my_list_3)

#####  Key Differences map() vs reduce():
//  1. map --> maps each element of a given collection
//  2. reduce --> "Collapses" elements by combining them until you have ONE FINAL OUTPUT


#####  filter()
val my_list_4 = my_list.filter(x => x % 2 == 0)
println(my_list_4)
val my_list_5 = my_list.filter(x => x != 5)
println(my_list_5)
val my_list_6 = my_list.filter(_ != 3)
println(my_list_6)

##### UNDERSCORE
- The Underscore symbol:
  - Is just a shorthand for:
  - a Lambda simple func
  - so is equivalent to x:Int => x * 2 for ex.


#####  Concatenate Lists:
val concatenated_list = my_list ++ my_list_2


##### Reverse:
my_list.reverse


#####  Sorting:
my_list.sorted


#####  Duplicates:
val distinct_list = concatenated_list.distinct


#####  max
distinct_list.max


#####  Total
distinct_list.sum


#####  Check if it CONTAINS a value:
distinct_list.contains(1)
distinct_list.contains(100)

####  [3. Maps (Dictionaries or key-value pairs)](https://docs.scala-lang.org/overviews/collections-2.13/maps.html#inner-main)


#####  MAPS (Dictionaries or KEY-VALUE Pairs):
- val map_object = Map(1 -> "a", 2 -> "b", 3 -> "c", 4 -> "d", 5 -> "e")
  - println(map_object)
  - println(map_object(3))  
  - println(map_object.contains(3))
  - println(map_object.contains(10))

- println(map_object.contains("c")) 
  - --> Can Only Search by key not value.
  - println(map_object.get(3))
  - println(map_object.get(10))

- val checking_map_object = util.Try(map_object(10)) getOrElse("Not Found")
  - println(checking_map_object)

#####  scala.util.try
- Why util.Try and not just try:
  - util.Try is a **Type/Class:** 
    - util.Try refers to a specific type or class 
      - provided in Scala's standard library. 
      - This class has methods like map, flatMap, and getOrElse that make it easy to work with potentially failing computations.

  - Difference in Semantics: 
    - util.Try wraps the computation result, 
    - whereas try...catch handles exceptions by diverting the program flow.

  - Functional Style: 
    - util.Try is ***preferred*** in functional programming 
    - as it fits better with the concept of working with values and transforming them.

#####  Exercise:
val main_list = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20)
println(main_list.filter(_ % 3 == 0))

var filtered_list = List[Int]()
for (i <- main_list) {
if (i % 3 == 0) {
filtered_list = filtered_list :+ i
}
}
println(filtered_list)

### 5. [Class or Object Oriented](https://docs.scala-lang.org/scala3/book/domain-modeling-oop.html)

Go Through the link above for detailed insights into:
- Classes or object oriented use in SCALA

## Section 3: Spark Basics and the RDD Interface

### 1. The Resilient Distributed Dataset

####  RDD - Definition and Structure
- Dataset structure with many rows;
- Rows get distributed across different machines
- Spark's job to make sure its resilient

#### Spark Context
- Is responsible for making sure RDD is Resilient and Distributed
- Created by the driver program
- Creates the RDD's
- Creates a "sc" object if using Spark Shell

#### Creating RDD
- Multiple Sources:
	- Local file - sc.textFile("file:///c:/users/text_file_path_here.txt")
	- S3n://
	- hdfs:// - *This is Hadoop File System*
  - Can also create from:
    - Hive:
      - `hiveCtx = HiveContext(sc)`
      - `rows = hiveCtx.sql("SELECT name, age FROM users")`
    - JDBC
    - Cassandra
    - HBase
    - Elasticsearch
    - JSON, CSV, sequence files, object files, various compressed formats

#### Transforming RDD's
- **map**: Applies a function to each element of the RDD, returning a new RDD with the results.
- **flatMap**: Similar to `map`, but each input item can be mapped to zero or more output items (flattens the results).
- **filter**: Returns a new RDD containing only elements that satisfy a given condition.
- **distinct**: Removes duplicate elements from the RDD.
- **sample**: Returns a random sample of elements from the RDD.
- **union, intersection, subtract, cartesian**: Set operations for combining or comparing RDDs; 
	- `union` merges, 
	- `intersection` finds common elements, 
	- `subtract` removes elements, and 
	- `cartesian` returns all possible pairs.
---
- MAP Example:  
	```sql
	val rdd = sc.parallelize(List(1, 2, 3, 4))  
	val squares = rdd.map(x => x*x) 
	```
- This example is important because:
	- It shows that `rdd.map(x => x*x)` takes in a func not just a parameter
	- *This is where*::::: **SCALA --> Functional Programming**::::: *Comes in*
	- A Lambda Func was used here (func literal, or in other words: *"def as we go"*)

#### RDD Actions
- **collect**: Returns all elements of the RDD as an array to the driver.
- **count**: Returns the number of elements in the RDD.
- **countByValue**: Returns a map of each unique value and its count in the RDD.
- **take**: Returns the first N elements of the RDD as an array.
- **top**: Returns the top N elements from the RDD (by default, in descending order).
- **reduce**: Aggregates the elements of the RDD using a specified function.
- There are additional actions like: 
  - `first`: Returns the first element of the RDD.
  - `takeSample`: Returns a random sample of elements from the RDD.
  - `foreach`: Applies a function to each element of the RDD (useful for side effects).
  -  *etc., for retrieving or processing RDD data*: Other actions exist for accessing or manipulating RDD contents.

#### Lazy Evaluation
- **Definition:** Computation is deferred until an action is triggered.
- **Purpose:** Optimizes execution by building a logical plan before running tasks.
- **Effect:** No data processing occurs until an action (like `collect` or `count`) is called.
- **Benefit:** Reduces unnecessary computations and improves performance.

### 2. Apache Spark Workflow: Full Breakdown










