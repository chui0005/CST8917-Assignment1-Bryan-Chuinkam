# CST8917 — Critical Analysis: *Serverless Computing: One Step Forward, Two Steps Back*
**Name:** Bryan Chuinkam  
**Student Number:** 040811108  
**Course:** CST8917  
**Assignment Title:** Critical Analysis + Azure Durable Functions Deep Dive  
**Date:** February 7, 2026  

---

## Part 1: Paper Summary (400–600 words)

The research paper makes the arguement that first-generation “serverless” platforms, specifically Functions-as-a-Service (FaaS), deliver a real usability breakthrough (automatic autoscaling + pay-per-use) while simultaneously regressing on two fundamentals of modern computing: 
- *data-centric performance* and 
- *distributed systems capabilities*. 

Their central thesis is that today’s FaaS is “one step forward, two steps back” because it makes it easier to run code elastically, but makes it hard (or prohibitively inefficient) to build *innovative data systems* and *coordinated distributed applications* in the cloud. In other words, the platform’s operational simplicity comes packaged with architectural constraints that push developers toward vendor-managed services and away from building new systems.

The paper identifies several concrete limitations in first-gen FaaS:  
1) **execution time constraints**: function instances are short-lived and forcibly terminated (e.g., 15 minutes in AWS Lambda), and there is no guarantee that successive invocations land on the same worker, so developers must assume state cannot be preserved in-memory across calls. 
2)  **communication and network limitations**: functions are not directly network-addressable, so they cannot act like stable endpoints. This forces coordination and “conversation continuity” (stickiness) to be implemented via external storage, which adds latency and cost. 
3) The authors also emphasize **I/O bottlenecks**: function instances access storage/services over the network and can see bandwidth far below local disks, and bandwidth can degrade as more functions are packed onto shared hosts. 

These issues combine into what the authors call the **“data shipping” anti-pattern**: instead of pushing code to where the data is, FaaS typically pulls data across the network into isolated, ephemeral compute. They argue this is structurally bad for latency, bandwidth, and cost. They further argue that the lack of addressability and reliance on slow storage intermediaries **stymies distributed computing**, because classic distributed protocols (membership, leader election, commit/consensus) require fine-grained, low-latency communication that storage-based mediation cannot provide efficiently. 

4) Another limitation is **restricted hardware access**: FaaS offerings typically expose only CPU/RAM slices and do not offer specialized hardware (e.g., GPUs), which is increasingly important for modern ML and data workloads. 

Finally, these constraints make **stateful workloads** awkward and slow in practice, often resulting in “workflow” designs stitched through queues and object stores with high end-to-end latency.

For future cloud programming, the authors propose directions that preserve autoscaling but remove today’s architectural handcuffs. At minimum, they call for: 
1) **fluid code/data placement** (shipping code to data when needed),
2) **heterogeneous hardware support**,  
3) **long-running, addressable virtual agents** (durable identities/endpoints that still allow elastic remapping). 
4) “**disorderly**” (asynchronous, coordination-aware) programming models that match the realities of distributed systems, rather than forcing everything into short-lived stateless invocations. 
---

## Part 2: Azure Durable Functions Deep Dive (5 topics, 100–150 words each)

### 1) Orchestration model (orchestrator + activity + client)
Azure Durable Functions adds a workflow layer on top of basic Azure Functions. A **client function** starts an orchestration instance (and can query or signal it), an **orchestrator function** defines the control-flow of the workflow, and **activity functions** perform the actual work (I/O, CPU tasks, API calls). This differs from “plain” FaaS where composition is typically done externally (queues, storage triggers etc). The orchestrator centralizes the workflow logic while the runtime handles scheduling, retries, and persistence behind the scenes. 

Relative to the paper’s criticism, Durable Functions partially mitigates “functions as disconnected, ephemeral shards” by providing a structured composition model that is durable and resumable. Even though the underlying compute is still serverless and elastic.

### 2) State management (event sourcing, checkpointing, replay)
Durable Functions makes workflows “stateful” by persisting orchestration progress to a **durable store** (Azure Storage by default). The orchestrator uses an **event-sourcing** model: actions taken by the orchestration are recorded as history events, and the orchestrator can **replay** deterministically from history to reconstruct local state after restarts, scale-outs, or failures. 

The docs describe this as saving state at **checkpoints** into durable storage, where the stored “orchestration history” is the source of truth. This directly addresses the paper’s “stateless function” critique at the programming-model level: you can write long-running logic that appears stateful, while the runtime handles persistence and recovery.

### 3) Execution timeouts (what’s bypassed, what still applies)
Durable Functions are designed for *long-running workflows*, but they don’t magically remove all limits. The platform avoids the “single invocation must run for hours” problem by letting the orchestrator **checkpoint and suspend** between awaits/timers and later resume, so the workflow can span minutes to days without relying on one continuously running process. That said, Microsoft’s guidance notes that **activity, orchestrator, and entity functions are still subject to Azure Functions timeouts**, and Durable treats timeouts similarly to failures (requiring retries/handling). Also, for HTTP-triggered endpoints, there is a hard response-time ceiling (e.g., Azure Load Balancer idle timeout), so Durable’s async patterns are recommended for longer work. 

In paper terms: Durable Functions reduces the pain of short lifetimes for *workflows*, but long-running *compute steps* inside activities can still be constrained. 

### 4) Communication between functions (how messages flow)
Durable orchestrations coordinate work primarily through the **Durable Task Framework** and a **task hub** stored in durable storage. The orchestrator doesn’t directly “call over the network” to a specific activity instance; instead, scheduling an activity results in messages/events being persisted and picked up by workers, while progress is tracked in the task hub. This improves developer experience compared to manual queue/storage chaining because the messaging, correlation, and resumption are built into the framework. But relative to the paper’s complaint about “communication through slow storage intermediaries,” Durable’s reliability mechanisms still fundamentally depend on durable storage as the coordination backbone (history tables, queued events, persisted state). So it *standardizes* storage-mediated coordination, but does not replace it with low-latency, addressable, point-to-point function communication. 

### 5) Parallel execution (fan-out/fan-in)
Durable Functions supports a canonical **fan-out/fan-in** pattern: an orchestrator starts many activity functions concurrently (fan-out), waits for them all to complete, then aggregates results (fan-in). Microsoft documents this pattern as a standard scenario for parallelizing independent tasks and then combining outputs inside the orchestration. This maps directly to the paper’s “distributed computing” concerns, *but only for a subset of distributed problems*. It works well when tasks are mostly independent and coordination is limited to “start many jobs, wait, combine.” For richer distributed systems (membership, leader election, consensus, fine-grained messaging), Durable still routes coordination through the task hub/history mechanisms rather than offering addressable agents and low-latency inter-function messaging. So it meaningfully improves serverless parallel workflows, but it does not turn serverless into a general distributed-systems substrate. 

---

## Part 3: Critical Evaluation (400–600 words)

### 1) Limitations that remain unresolved

**(A) Data shipping and locality remain fundamental.**  
The paper’s biggest architectural critique is that FaaS tends to “ship data to code” rather than “code to data,” causing avoidable latency/bandwidth/cost due to memory-hierarchy realities. Durable Functions improves *workflow durability* but does not change where compute runs relative to storage. Orchestrators and activities still execute on elastic workers that access data/services over the network, and the durability mechanism itself persists state and history into storage. In other words, Durable Functions adds a robust coordination layer, but its default “physics” still relies on remote reads/writes and storage-backed progress tracking. This means that for data-intensive workloads (large scans, training, heavy ETL), the paper’s locality critique largely persists. Durable can coordinate steps, but not fundamentally colocate compute and data in the way the authors argue is required for efficient cloud-scale innovation.

**(B) Lack of addressable, long-running agents (and low-latency inter-agent networking).**  
A second core critique is that serverless functions are not directly addressable and must communicate through slow intermediaries, making classic distributed protocols impractical. Durable Functions does not provide “addressable agents with network-like performance”. It provides *instances* that can be managed (query, signal, terminate) via client APIs, but coordination and messaging are still mediated through the task hub and persisted events/history. That’s excellent for reliability and orchestration, but it is not the same as the paper’s proposed model of durable, addressable endpoints that support fine-grained communication comparable to normal networks. So Durable Functions partially addresses “statefulness” at the workflow level, but only partially addresses the deeper “distributed computing substrate” criticism.

### 2) Verdict
Azure Durable Functions represents *real progress* toward one slice of what the authors want. It makes stateful coordination feasible in serverless by checkpointing, and reliable resumption. In that sense, it pushes beyond naïve “glue code + queues + storage” composition patterns the paper criticizes as slow and awkward. 

However, it is better described as a **workaround layer** than a full architectural solution to “one step forward, two steps back.” The paper’s forward-looking proposals emphasize fluid code/data placement, heterogeneous hardware, and long-running addressable virtual agents that can recoup locality/affinity costs across requests. Durable Functions doesn’t deliver those. Its durability is powered by storage-backed history (helpful, but still storage-mediated), and it does not create a new low-latency addressable fabric for distributed systems. 

In conclusion my position is that durable Functions represents meaningful progress for coordinating serverless workflows, yet it does not fully address the deeper architectural concerns raised in the paper, particularly around data locality and efficient distributed coordination. While it enhances developer productivity and reliability within current serverless constraints, it largely adapts to those constraints rather than fundamentally reshaping them in the way required for broader cloud and data-systems innovation.

---

## AI Disclosure Statement:
- Used ChatGPT to: 
  - help explain certain sections of the paper. 
  - help find other resources on Serverless functions 
  - check my understanding of the paper and the logic of my answers. 

## References (with hyperlinks)

### Paper
- Hellerstein, J. M., Faleiro, J., Gonzalez, J. E., Schleier-Smith, J., Sreekanti, V., Tumanov, A., & Wu, C. (2019). *Serverless Computing: One Step Forward, Two Steps Back (CIDR 2019).* (Provided PDF). 

### Official Microsoft Documentation (Azure Durable Functions)
- [Durable Functions overview](https://learn.microsoft.com/en-ca/azure/azure-functions/durable/durable-functions-overview)
- [Durable orchestrations](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-orchestrations?tabs=csharp-inproc)  
- [Orchestrator code constraints (determinism + replay)](https://learn.microsoft.com/en-ca/azure/azure-functions/durable/durable-functions-code-constraints)  
- [Task hubs in Durable Functions](https://learn.microsoft.com/en-ca/azure/azure-functions/durable/durable-functions-task-hubs)  
- [Azure Storage provider for Durable Functions (history table)](https://learn.microsoft.com/en-ca/azure/azure-functions/durable/durable-functions-azure-storage-provider) 
- [Performance and scale in Durable Functions (timeouts)](https://learn.microsoft.com/en-ca/azure/azure-functions/durable/durable-functions-perf-and-scale) 
- [Azure Functions scale and hosting (HTTP response limit)](https://learn.microsoft.com/en-ca/azure/azure-functions/functions-scale)
- [Fan-out/fan-in scenario sample](https://learn.microsoft.com/en-ca/azure/azure-functions/durable/durable-functions-cloud-backup) 
