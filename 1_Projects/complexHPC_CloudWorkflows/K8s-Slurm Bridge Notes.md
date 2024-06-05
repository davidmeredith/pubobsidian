https://slurm.schedmd.com/SC23/Slurm-and-or-vs-Kubernetes.pdf
https://slurm.schedmd.com/kubernetes.html
https://www.youtube.com/watch?v=7MTb8iyvG5Q&t=258 (must watch)


DM: I'm tending toward the KCP side (bridging using a multi-cluster approach) rather than SUNK which is a more integrated approach. I can definitely see why sunk is great for the cloud hyperscalers i.e. to use K8s as the sole/main management software, but for Hartree 
## SUNK 
Navarre Pratt video: Machine learning engineer, CoreWeave. https://www.youtube.com/watch?v=48ONt0UKiew
- Balancing slurm + K8s resources WITHIN THE SAME K8s CLUSTER
- Everything is k8s in CoreWeave, even VMs.
- Approaches that SUNK will replace:
https://slurm.schedmd.com/SLUG23/NERSC-SLUG23.pdf
- Will be open sourced here in ~2024:  https://github.com/coreweave/sunk

Approach allows both Burst & Batch workloads on the same cluster (rather than the bridging/multi-cluster approach). Ability to schedule large batch HPC on same resource as well as long-running K8s workloads.

- SUNK is deployed as a HelmChart: Slurm accounting, Slurmd, Slurm Controller, prologue+epilogue scripts. 
- No loss of Slurm features. 
- High Availability (HA) of slurm control plane for free (K8s will restart sunk control plane on issues, don't need to run failover slurm, bespoke liveness&readiness probes), 
- Configure any auth scheme. 
- Compute resource management with requests & limits. 
- Requires long-lived SSH login nodes - appears like regular SLURM to user, no differences 
- Adds Prioritisation, which is not a central aspect of pure K8s cloud orchestration where all workload is expected to run concurrently by default

You create a shared FS via persistent volume claims. Is easy to mount into both login node and worker node, but also other K8s nodes when needed for flexible integration - Nice.  

Workflows can be designed and built using GitHub workflows for automation. 
![](attachments/Pasted%20image%2020240605142011.png)
![](attachments/Pasted%20image%2020240605135413.png)
### SUNK K8s Scheduler (Updated SUNK scheduler supporting both K8s + Slurm)
Allows scheduling from both sides. Effectively schedules native k8s workloads via slurm sched + native SLURM batch workloads: Is itself based on SLURM advanced scheduler. Supports:  Priority, pre-emption/Eviction, Drain/Active, Partitions all through the scheduler. <mark style="background: #FFB86CA6;">This scheduler replaces the regular K8s scheduler as it supports all K8s out of box + SLURM extras.  </mark>

### SUNK cluster architecture
https://slurm.schedmd.com/SLUG23/CoreWeave-SLUG23.pdf
1. Cluster wide control plane / Cluster scoped operator (single install per cluster): NodeSet Controller, Node controller, Slurm cluster controller. 
2. Compute Nodes in the middle layer 
3. Per-tenant namespace (can be duplicated per tenant). All run as pods. Slurm login nodes ssh into login pods with SSH. 


![](attachments/Pasted%20image%2020240408090510.png)

![](attachments/Pasted%20image%2020240408151127.png)
**Section A:** All of the typical Slurm components are deployed within a pod, each with its own configurable resource requests. Not pictured here, but this <mark style="background: #BBFABBA6;">also includes login nodes that users connect to in order to interact with the Slurm cluster (SSH)</mark>. Once connected to these login nodes, Kubernetes is abstracted away, and you get the experience of a normal Slurm cluster. 

‍**Section B:** Slurm configuration is needed throughout many of these components. By deploying them as <mark style="background: #BBFABBA6;">k8s ConfigMaps and Secrets</mark>, Slurm configuration files can be managed in a single place and mounted everywhere they’re needed. This includes Slurm configuration, <mark style="background: #BBFABBA6;">topology, prolog/epilog scripts, and sensitive information like DB passwords</mark>.

‍**Section C:** The most important aspects of an HPC cluster are the compute nodes, which are shown in the middle as bare-metal Kubernetes nodes. The <mark style="background: #BBFABBA6;">slurmd’s are run within compute pods shown above in a 1:1 mapping, akin to daemonset</mark>. The deployment of compute pods are managed by a CRD called a Nodeset (**Section F**). The <mark style="background: #BBFABBA6;">Nodeset maintains a series of status fields representing the state in Slurm. This provides mechanisms for protected updates and scaling of Slurm nodes based on the states in both kubernetes and Slurm</mark> . The Nodeset pods run both slurmd and munged, mounting in shared configmaps for things like the Slurm config, the prolog and epilog, and shared filesystem volumes as PVCs.

**Section D:** <mark style="background: #FFB86CA6;">Many of the features we’ve talked about require the state of compute from the Slurm and Kubernetes sides to be in sync. The Slurm syncer acts as a middleman between the two sides by sending and pulling information through Slurm’s REST API. On a big cluster, this can cause lots of traffic and so cacheing is used.
</mark>

‍**Section E:** Once the syncer gets state information from Slurm into Kubernetes, there are many different places the information needs to be to stay consistent. The cluster-wide operators monitor different resources and make changes when appropriate, whether the change originates from Kubernetes or Slurm.
### Nodeset Pods (Section F)
- Run Nodeset pod on the nodes you want to make up your hybrid cluster - they don't have to be all nodes, but the Nodeset pod supports both the regular K8s workloads + slurm 
- Cross between Daemonset and Statefulset because it doesn't request normal GPU resource as you would in normal GPU workloads, but rather a special <mark style="background: #BBFABBA6;">SUNK accelerator resource</mark>. This nodeset is deployed on all pods and supports both SLURM scheduler + normal K8s workloads too. 

![](attachments/Pasted%20image%2020240408090314.png)

- Above: Shows 2 SUNK accelerator resource statuses (slurm-a40, slurm-h100). 
- First 5 statuses are from regular K8s, Running and Draining are from SLUM. 
- Protected/rolling updates mean we don't affect any customer jobs.
### Syncer
![](attachments/Pasted%20image%2020240408091138.png)

### Scheduler 
AKA host both regular K8s workloads (inference on left) and SLURM batch reservations (on right)
![](attachments/Pasted%20image%2020240408091912.png)
Allows dynamic switching between jobs based on their priority - e.g. if a high priority training job comes in, existing jobs can be evicted. So, <mark style="background: #BBFABBA6;">there is only one process from either SLURM OR from K8s using those nodes at any one time</mark>. 


CoreWeave Hold the MLPerf record. 

![](attachments/Pasted%20image%2020240408092626.png)

![](attachments/Pasted%20image%2020240408130105.png)
### Another SUNK vid
https://www.youtube.com/watch?v=3E1knT313tI

How the Record breaking cloud native AI Supercomputer was built - Peter Salanki, CoreWeave

"Traditional HPC involves spending 2yrs building an HPC, then you test it, and it then stays static. This doesn't really work in today's age of rapid AI evolution where we need elasticity and something agile where we don't have to take down the cluster for a week like you do in a traditional academic environment."

“Building an ecosystem around Kubernetes makes it very easy for us to plug in new things. And [get metrics](https://thenewstack.io/prometheus-at-10-whats-been-its-impact-on-observability/) out without having to build a bunch of glue between proprietary systems and Kubernetes itself,” Salanki said. https://thenewstack.io/hpc-kubernetes-ai-training-on-3500-gpus/

“Everything is stateless,” Salanki said. “It’s fully ephemeral, which means we can plug in your notes and get them up and running on a Kubernetes cluster immediately.”

In this setup, the [Kubernetes API](https://thenewstack.io/kubernetes-api-gateway-1-0-goes-live-as-maintainers-plan-for-the-future/) server is central. “Every action flows through Kubernetes. There is no path that does not go through Kubernetes,” he said. An admin that wants to reboot a node sets a condition on the node, which will trigger a reboot by the node controller.  The whole flow is captured by event logging.

“By centralizing the entire management flow on Kubernetes, we can get a lot of stuff for free, including a programming model that many developers already know," he said.

Q. When is the combined SUNK approach useful? 
A. In the scenario when you have model training for a huge high priority job but you also have less important research jobs that need to run alongside production inference workloads (really need K8s for the latter),  

Q. What advantages does it give the HPC community?j
A. More availability and portability rapid/piecemeal container deployment (package deps in container), huge K8s infrastructure becomes available for free, dynamic isolation & better handling of noisy neighbours is better than anything available in traditional HPC. Don't need a separate HPC - one hardware. 




HPC on OpenShift (to look at)

# Cloud native SLUM - KCP (Nvidia)
Eduardo Arango (Nvidia)
https://www.youtube.com/watch?v=7MTb8iyvG5Q&t=258
Presented 03/04/24

Currently, below tools are being tested and improved, but they aren't there yet.
### New Tools Emerging: 
- Kueue, JobSet (these are pre-scheduling tools; they help to optimise on a pre-scheduling decision, once Kueue posts the job, it hands the job to the 'K8s scheduler plugin')
- Sunk
### K8s Scheduling Patterns: 
- Scheduling Plugin for K8s, Flux Framework operator

**Issue:** 
With the above tools and approaches (integrating with the K8s scheduler plugin) you have to sync state across two systems; Kueue/JobSet + K8s Scheduler Plugin. 

MPI Operator: great tool, important for HPC/AI on k8s, being proposed as a standalone tool in CNCF.

Priyanka Sharma - KubeCon keynote, SLURM on K8s. 


![](attachments/Pasted%20image%2020240410082258.png)
Looks a lot like K8s
Slurmd is similar to K8s API server 
Slurmdbd similar to K8s etc (state)

PMIx presents a unified API that hides many of the complexities of communication with these back-end run-time environments. Open MPI uses the PMIx API to discover, communicate, and coordinate with any supported back-end run-time system without needing to know the intimiate details of that system.

![](attachments/Pasted%20image%2020240410083114.png)



## KCP 
https://github.com/kcp-dev/kcp
A uniform API to communicate with any platform on K8s - say you have a platform that you want to expose on K8s, you can use KCP to provide API access to that 

![](attachments/Pasted%20image%2020240410083825.png)

![](attachments/Pasted%20image%2020240410084109.png)

![](attachments/Pasted%20image%2020240410084446.png)
Like a stripped down version of K8s (without pods, containers,nodes) but with a control plane with API server + K8s good stuff. 

![](attachments/Pasted%20image%2020240410085100.png)

Multi-cluster approach: don't need to Sync across the two clusters (unlike SUNK). In the above slide, the SLURM/KCP resource is not a K8s cluster

Infra Provisioner / Bare metal provisioning (small box, bottom right): multiple tools from HPC community: Warewolf, former Bright Computing (now owned by NVIDIA), many others, help you to cloud provision across the vendors. https://warewulf.org/ 

-  Audience Q. In the MultiCluster model (Cloud Native SLURM, above), how do you reason about the resource management across 2/multiple clusters?
   - A. Eduardo: Use a central control plane (small 'management' cluster with few nodes to act as a central control plane to reason across the clusters - the top row with the embedded red box 'The Job Controller' - they are working towards adding a cluster inventory in forthcoming K8s alpha release).  
   - MarkB is right in that that Job Controller / 'small management cluster' would still need to have scheduling across the two clusters for more complex scenarios. 


![](attachments/Pasted%20image%2020240410090212.png)

KCP to be open sourced soon, below slide was presented on Apr 3rd:
![](attachments/Pasted%20image%2020240410090412.png)

Q. Why twist K8s to HPC? 
A. We don't, add SLURM as a tool into CNCF and use multi-cluster approach like KCP to bridge (unlike SUNK)

### Q&A
- Dennis Marttinen - Finish Centre for SuperComputing (CSC) Louis supercomputer. Also looking at K8s integration. 
- CSC Challenge: Most of their code on Louis is GPU heavy, their CPUs are mostly idle. For the CPU side, their users want to run bursty workloads/Kafka/Spark in parallel, but is much easier to do Github types of deployments on K8s, hence need a bridge to exploit the unused HPC CPU. 
- CSC Interlink (a topic of future talk for Dennis) - a bridge between HPC and K8s needed as an interim before ultimately/perhaps running k8s on the HPC compute nodes (but CSC hpc folks are v.stuck in their ways & firewalled, hence a bridge layer is needed to demonstrate the benefits). Related: https://www.youtube.com/watch?v=M3uLQiekqo8

-  Audience Q. In the MultiCluster model, how do you reason about the resource management across 2/multiple clusters?
   - A. Eduardo: Use a central control plane (small 'management' cluster with few nodes to act as a central control plane to reason across the clusters - the top row 'The Controller' - working towards adding a cluster inventory in forthcoming K8s alpha release)


Q. Alex Scammon: What about SUNK? (balancing slurm + K8s resources WITHIN THE SAME K8s CLUSTER). 
A. Eduardo: It works across ~100 nodes, but at scale synchronisation issues occur. Eduardo doesn't like this approach, suggests using the multi-cluster approach. But the again, CoreWeave is huge and they use it in a seemingly highly successful way so maybe that's just sour grapes?  


