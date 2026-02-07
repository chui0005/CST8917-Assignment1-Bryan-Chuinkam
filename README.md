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

Finally, these constraints make **stateful workloads** awkward and slow in practice—often resulting in “workflow” designs stitched through queues and object stores with high end-to-end latency.

For future cloud programming, the authors propose directions that preserve autoscaling but remove today’s architectural handcuffs. At minimum, they call for: 
1) **fluid code/data placement** (shipping code to data when needed),
2) **heterogeneous hardware support**,  
3) **long-running, addressable virtual agents** (durable identities/endpoints that still allow elastic remapping). 
4) “**disorderly**” (asynchronous, coordination-aware) programming models that match the realities of distributed systems, rather than forcing everything into short-lived stateless invocations. 
---