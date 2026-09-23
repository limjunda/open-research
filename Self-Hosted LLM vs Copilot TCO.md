# **Comparative Architectural and Financial Analysis: Self-Hosting Open-Source Large Language Models on Google Cloud Platform Versus Microsoft 365 Copilot Enterprise Licensing**

## **Executive Summary**

Enterprise decisions regarding the deployment of generative artificial intelligence center on the trade-off between managed Software-as-a-Service (SaaS) productivity suites and self-hosted open-source foundation models on public cloud Infrastructure-as-a-Service (IaaS). The primary commercial benchmark for workplace collaboration is Microsoft 365 Copilot, which includes deep native integration into Microsoft Teams and carries an enterprise list price of $30.00 per user per month with an annual commitment1. This cost exists as an add-on to qualifying base enterprise suites, resulting in an effective all-in subscription footprint of $69.00 to $90.00 per seat per month when accounting for Microsoft 365 E3 or E5 requirements1.

Organizations evaluating whether to bypass commercial licensing in favor of private deployments on Google Cloud Platform (GCP) face stark architectural divides. At one end of the spectrum is GLM-5.2 (744B), an open-weight Mixture-of-Experts (MoE) foundation model with 744 billion total parameters, 40 billion active parameters per forward pass, Multi-head Latent Attention (MLA), and DeepSeek Sparse Attention (DSA)4. At the opposite end is a quantized 8-billion-parameter (8B 4-bit) dense model, which requires minimal compute overhead.

The operational viability of private cloud hosting rests on two primary variables: the computational engine and the total organizational scale. While small quantized models run cost-effectively on lightweight virtual machines or entry-level graphics processing units (GPUs), running frontier-class models like GLM-5.2 requires either memory-streaming CPU architectures like colibri7, hyper-memory CPU instances, or multi-GPU clusters spanning NVIDIA A100 or H100 hardware8.

Deploying open-source models on GCP fundamentally alters the corporate cost structure: it trades fixed per-seat operational expenditures (OpEx) for compute infrastructure, extensive custom software engineering, continuous retrieval-augmented generation (RAG) maintenance, and identity-driven access control compliance1.

## **Compute Topology and Hardware Benchmarking on GCP**

Evaluating self-hosted open-source models requires mapping model weights and operational key-value (KV) caches to specific Google Compute Engine (GCE) machine shapes across Central Processing Unit (CPU) architectures, hybrid offload topologies, and pure GPU accelerator configurations.

### **Hardware Topologies and Memory Requirements**

An 8B model quantized to 4-bit precision occupies approximately 4.5 to 5.5 GB of memory for model weights, with dynamic memory overhead scaling alongside context length. This small footprint allows deployment on commodity general-purpose CPU instances, such as the C3 Intel Sapphire Rapids family, or entry-level accelerator instances equipped with a single NVIDIA L4 GPU providing 24 GB of VRAM12. The compute density of the L4 allows high concurrency and dynamic request batching, transforming small dense models into scalable microservices8.

In contrast, GLM-5.2 (744B MoE) contains 744 billion parameters across 256 routed experts and 1 shared expert, with 75 of its 78 transformer blocks arranged as sparse feed-forward networks4. At 4-bit quantization, the model weights alone demand roughly 380 GB of storage and memory7. Deploying this parameter volume on GCP can be achieved through four distinct topologies:

In a pure CPU disk-streaming configuration using the colibri engine, inference relies on a hand-managed three-tier memory hierarchy written in pure C7. The dense components of the architecture—comprising roughly 17 billion parameters or 9.9 GB in INT4—remain permanently resident in system RAM, while the 21,504 routed experts totaling approximately 370 GB are streamed on demand from high-throughput local Non-Volatile Memory Express (NVMe) solid-state drives7. Because only about 11 GB of expert weights change between successive generated tokens, the workload can execute on modest virtual machines equipped with 25 GB of RAM and local NVMe scratch disks, such as the c3-standard-8-lssd or c3-standard-22-lssd machine types7.

A RAM-resident CPU configuration eliminates disk input/output bottlenecks by utilizing memory-optimized shapes, such as the m3-ultramem-32 instance, which provides 976 GB of DDR5 system memory15. This configuration maintains the entire quantized model checkpoint and associated KV caches in system RAM, avoiding storage latency at the cost of substantial hourly compute rates15.

A hybrid CPU-GPU offloading architecture combines moderate accelerator instances, such as the g2-standard-24 featuring two NVIDIA L4 GPUs with 48 GB of aggregate VRAM, with host system memory8. In this layout, the dense attention layers and active latent cache reside within GPU memory, while the bulk of the sparse expert weights are swapped across PCIe channels into host RAM on demand5.

A dedicated enterprise multi-GPU cluster fits all model parameters directly into high-bandwidth memory (HBM). To support GLM-5.2 in FP8 or INT4 precision, the cluster must provide between 512 GB and 640 GB of aggregate VRAM. On GCP, this architecture is fulfilled either by the a2-ultragpu-8g node, which supplies eight NVIDIA A100 80GB SXM4 GPUs with 640 GB of aggregate VRAM and 1,360 GB of host memory, or the a3-highgpu-8g node, which incorporates eight NVIDIA H100 80GB SXM5 GPUs with 640 GB of HBM3 memory and 1,872 GB of host memory8.

### **Throughput and Performance Benchmarks**

Inference throughput, quantified in tokens per second (TPS), governs conversational usability and platform scaling limits. Real-time workplace interfaces require 20 to 50 TPS per stream to match average human reading comprehension and maintain interactive responsiveness.

&nbsp;

| Model Precision & Target Engine | GCP Machine Type | Hardware Configuration | Memory Allocation | Generation Throughput (TPS) | Latency Profile & Viability |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **8B 4-bit** (llama.cpp / vLLM) | c3-standard-4 | 4 vCPU (Sapphire Rapids) | 16 GB DDR5 | 25 – 30 TPS | Viable for single-user text summarization13 |
| **8B 4-bit** (llama.cpp / vLLM) | c3-standard-8 | 8 vCPU (Sapphire Rapids) | 32 GB DDR5 | 40 – 45 TPS | Fluid interactive chat for 1–2 concurrent users13 |
| **8B 4-bit** (vLLM Engine) | g2-standard-4 | 1x NVIDIA L4 (24 GB) | 16 GB Host / 24 GB VRAM | 75 TPS (Single) / 500 TPS (Batch) | High-concurrency enterprise microservice12 |
| **GLM-5.2 744B** (colibri C-Engine) | c3-standard-8-lssd | 8 vCPU \+ Local NVMe SSD | 32 GB RAM \+ 375 GB NVMe | 0.3 – 1.0 TPS (Avg 0.8 TPS) | Unusable for interactive chat; batch-only7 |
| **GLM-5.2 744B** (colibri C-Engine) | c3-standard-22-lssd | 22 vCPU \+ Local NVMe SSD | 88 GB RAM \+ NVMe Cache | 0.8 – 1.5 TPS (Avg 1.0 TPS) | Severe I/O wait; limited to asynchronous agents7 |
| **GLM-5.2 744B** (CPU In-Memory) | m3-ultramem-32 | 32 vCPU (Intel Xeon) | 976 GB RAM (Zero-Disk) | 4.0 – 6.0 TPS (Avg 5.0 TPS) | High memory-bandwidth stall; poor interactivity15 |
| **GLM-5.2 744B** (Hybrid Offload) | g2-standard-24 | 2x NVIDIA L4 (48 GB VRAM) | 96 GB RAM \+ Host Swapping | 1.5 – 2.5 TPS (Avg 2.0 TPS) | PCIe bus bottleneck limits practical throughput17 |
| **GLM-5.2 744B** (TensorRT-LLM / vLLM) | a2-ultragpu-8g | 8x NVIDIA A100 (80 GB SXM4) | 640 GB VRAM / 1.36 TB RAM | 45 – 55 TPS (Single) / 300 TPS (Batch) | Enterprise production grade; multi-tenant8 |
| **GLM-5.2 744B** (TensorRT-LLM / vLLM) | a3-highgpu-8g | 8x NVIDIA H100 (80 GB SXM5) | 640 GB HBM3 / 1.87 TB RAM | 70 – 90 TPS (Single) / 450 TPS (Batch) | Frontier-grade performance; real-time agentic execution8 |

The technical benchmark indicates that running GLM-5.2 across CPU architectures via NVMe disk streaming introduces severe operational constraints. While the colibri engine successfully avoids out-of-memory errors on small hardware footprints, its sustained generation throughput of approximately 0.8 TPS means an 800-word executive summary (roughly 1,000 tokens) requires over 20 minutes to complete7. Consequently, CPU-based disk-streaming architectures cannot serve as conversational workplace assistants; their utility is strictly confined to asynchronous batch jobs or offline agentic reasoning where real-time latency is not an operational requirement7.

## **Direct Economic Modeling: Cost Per One Million Tokens**

To quantify cloud hosting economics, compute costs must be translated into direct expenditures per 1 million generated tokens (![][image1]). The theoretical cost under fully saturated hardware utilization is modeled as:

![][image2]

![][image3]

Where ![][image4] represents the continuous machine hours required to produce one million tokens.

In corporate environments, compute infrastructure rarely achieves continuous 100% saturation. Enterprise workloads exhibit diurnal utilization curves, where peak business-hour activity gives way to overnight and weekend idleness. The true effective cost per million tokens depends directly on the capacity utilization rate:

![][image5]

The table below contrasts theoretical saturated performance against realistic enterprise duty cycles (10% to 50% capacity utilization) using standard US-region GCP on-demand and committed use discount (CUD) pricing8.

&nbsp;

| Architecture & Configuration | Effective Hourly Compute Cost | Sustained TPS | Hours per 1M Tokens (H1M​) | Saturated Cost per 1M Tokens | Effective Cost per 1M Tokens (50% Utilization) | Effective Cost per 1M Tokens (20% Utilization) | Effective Cost per 1M Tokens (10% Utilization) |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| **8B 4-bit** | CPU Only (c3-standard-4) | $0.201613 | 30 | 9.26 | $1.87 | $3.74 | $9.35 | $18.70 |
| **8B 4-bit** | CPU Only (c3-standard-8) | $0.418013 | 45 | 6.17 | $2.58 | $5.16 | $12.90 | $25.80 |
| **8B 4-bit** | GPU Single (g2-standard-4) | $0.706812 | 75 | 3.70 | $2.62 | $5.24 | $13.10 | $26.20 |
| **8B 4-bit** | GPU Batched (g2-standard-4) | $0.706812 | 500 | 0.56 | $0.39 | $0.78 | $1.95 | $3.90 |
| **GLM-5.2 744B** | colibri NVMe (c3-standard-8-lssd) | $0.480013 | 0.8 | 347.22 | $166.67 | $333.34 | $833.35 | $1,666.70 |
| **GLM-5.2 744B** | RAM-Resident (m3-ultramem-32) | $6.091215 | 5.0 | 55.56 | $338.40 | $676.80 | $1,692.00 | $3,384.00 |
| **GLM-5.2 744B** | Hybrid Offload (g2-standard-24) | $2.000817 | 2.0 | 138.89 | $277.78 | $555.56 | $1,388.90 | $2,777.80 |
| **GLM-5.2 744B** | 8x A100 Single (a2-ultragpu-8g) | $40.550018 | 50 | 5.56 | $225.28 | $450.56 | $1,126.40 | $2,252.80 |
| **GLM-5.2 744B** | 8x A100 Batched (a2-ultragpu-8g) | $40.550018 | 300 | 0.93 | $37.55 | $75.10 | $187.75 | $375.50 |
| **GLM-5.2 744B** | 8x H100 Batched (On-Demand) | $87.83008 | 450 | 0.62 | $54.22 | $108.44 | $271.10 | $542.20 |
| **GLM-5.2 744B** | 8x H100 Batched (3-Year CUD) | $38.860024 | 450 | 0.62 | $23.99 | $47.98 | $119.95 | $239.90 |

This quantitative breakdown exposes the economic paradox of low-throughput execution engines. While running colibri on an inexpensive CPU instance appears cost-effective on an hourly basis ($0.48/hr), its generating speed of 0.8 TPS inflates processing duration to 347 continuous machine hours per million tokens7. As a result, its saturated cost reaches $166.67 per million tokens20.

At enterprise scale, high-throughput accelerators invert this relationship: an 8x H100 cluster under heavy dynamic batching (450 TPS) costs $87.83/hr on-demand, but processes 1M tokens in just 37 minutes, reducing the saturated compute cost to $54.22 per million tokens, or $23.99 under a 3-year committed use discount8.

## **Organizational Scaling Analysis: 1 Person, 10-User Team, and 1,000-Seat Enterprise**

To assess commercial viability against Microsoft 365 Copilot, infrastructure costs must be evaluated across organizational tiers. Standard knowledge workers consume an average of 1.5 million blended tokens per month across interactive chats, document syntheses, and collaboration queries25. Highly active developers or agentic workflow users average approximately 5 million tokens per month25. For this analysis, baseline knowledge worker consumption is modeled at 2.0 million tokens per user per month.

The comparison benchmarks Microsoft 365 Copilot's flat fee of $30.00 per user per month against self-hosted infrastructure running 24/7 (730 hours per month) to support on-demand enterprise access1. Direct monthly token demand scales linearly as the seat count multiplied by 2,000,000 tokens, while monthly Copilot licensing equals the seat count multiplied by $30.001. Self-hosted cloud compute represents the instance hourly rate multiplied by 730 monthly operating hours13.

&nbsp;

| Deployment Tier & Scale | Total Monthly Tokens | Microsoft Copilot Expense ($30/seat) | 8B 4-bit on CPU (c3-standard-4) | 8B 4-bit on GPU (g2-standard-4) | GLM-5.2 744B on CPU (colibri lssd) | GLM-5.2 744B on 8x A100 (a2-ultragpu) | GLM-5.2 744B on 8x H100 (3-Yr CUD) |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| **Single User** (1 Seat) | 2 Million | **$30.00** | $147.17 | $515.96 | $350.40 | $29,601.50 | $28,367.80 |
| **Small Team** (10 Seats) | 20 Million | **$300.00** | $147.17 | $515.96 | $350.40 (Overloaded) | $29,601.50 | $28,367.80 |
| **Mid-Market** (100 Seats) | 200 Million | **$3,000.00** | $294.34 (2 Nodes) | $515.96 (Batched) | Hardware Failure (Throughput Deficit) | $29,601.50 | $28,367.80 |
| **Enterprise** (1,000 Seats) | 2 Billion | **$30,000.00** | $1,471.70 (10 Nodes) | $2,063.84 (4 Nodes) | Hardware Failure (Throughput Deficit) | $29,601.50 (1 Node saturated) | $28,367.80 (1 Node saturated) |
| **Large Enterprise** (5,000 Seats) | 10 Billion | **$150,000.00** | $7,358.50 (50 Nodes) | $10,319.20 (20 Nodes) | Hardware Failure (Throughput Deficit) | $148,007.50 (5 Nodes) | $141,839.00 (5 Nodes) |

The financial trajectory reveals clear scale-dependent inflection points. For a single user or a small team of 10 seats, self-hosting a frontier 744B parameter model on dedicated GPU infrastructure on GCP is economically indefensible. Provisioning an 8x A100 or 8x H100 instance requires a baseline commitment of $28,000 to $30,000 per month18. Generating only 20 million tokens across a 10-person team yields an effective direct infrastructure cost exceeding $1,400 per million tokens due to extreme idle capacity. Conversely, Microsoft 365 Copilot absorbs all compute elasticity into its $30.00 per-seat fee, costing a 10-person team just $300.00 monthly1.

For an 8B 4-bit model, the cost equation inverts quickly. A single g2-standard-4 GPU instance costing $515.96 per month easily processes up to 200 million tokens monthly via continuous dynamic batching12. At 100 users, the self-hosted direct compute expense of $515.96 is significantly lower than Copilot's $3,000.00 monthly license fee1.

At the scale of 1,000 seats generating 2 billion tokens per month, an 8x H100 node under a 3-year commitment ($28,367.80/month) reaches near cost parity with Copilot's $30,000.00 monthly licensing1. However, this parity reflects raw compute costs alone and excludes the auxiliary software engineering, security, and integration overhead required to build and maintain a custom enterprise stack.

## **Task Completion, Reasoning Fidelity, and Enterprise Utility**

Selecting an inference architecture requires balancing direct costs against functional utility within workplace environments. Standard enterprise collaboration requires specific core capabilities: unstructured data extraction, document generation, cross-repository software debugging, and real-time meeting synthesis.

&nbsp;

| Evaluation Dimension | Small Dense Model (8B 4-bit Quantized) | Frontier Sparse MoE (GLM-5.2 744B) | Microsoft 365 Copilot (Teams Integrated) |
| :---- | :---- | :---- | :---- |
| **Underlying Model Scale** | \~8 Billion dense parameters | 744 Billion MoE (40B active parameters)4 | GPT-4o / Claude 3.5 Sonnet hybrid30 |
| **Context Window Size** | 8,192 – 128,000 tokens | 1,000,000 tokens (1M native context)4 | Dynamic RAG context window (\~128K effective)34 |
| **Coding & Engineering (SWE-Bench / AAII)** | Limited: Basic single-function generation; syntax errors on complex logic | State-of-the-Art: 92.7% code correctness, 62.1% SWE-bench Pro22 | Moderate: Contextually aware via GitHub Copilot, but standard Copilot is basic |
| **Long-Horizon Reasoning (GPQA Diamond)** | Poor (\<35% accuracy); frequent chain-of-thought degradation | Superior: 91.2% accuracy; Dual Thinking modes (High / Max)5 | Strong: Frontier commercial model reasoning31 |
| **MS Teams Live Meeting Diarization** | Zero native capability; requires external multi-speaker ASR pipeline | Zero native capability; requires speech-to-text pipeline and ingest bots | Fully Native: Real-time speaker attribution, sentiment, thread recaps34 |
| **Enterprise Authorization & Graph Grounding** | None: Prone to leaking access across flat vector embeddings10 | None: Requires custom security-trimmed RAG middleware10 | Fully Native: Enforces Entra ID ACLs and security trimming out of the box10 |
| **Interactive Latency Profile** | High speed (\<50 ms Time-to-First-Token on GPU)20 | High speed on 8x H100; Unacceptable on CPU (\>15s TTFT)14 | Near Real-Time: Managed streaming via Microsoft backend |

Deploying an 8B 4-bit model to displace Microsoft 365 Copilot introduces severe functional gaps. While an 8B parameter model reliably extracts text, triages emails, and generates short conversational answers, it struggles with complex enterprise reasoning. When evaluating conflicting financial tables across separate departmental presentations, an 8B model's limited active attention capacity frequently produces hallucinations or omits nuanced edge cases38.

GLM-5.2 (744B MoE) delivers frontier reasoning capability. Its 92.7% code correctness rating, 91.2% GPQA Diamond score, and stable 1-million-token context window match or exceed commercial frontier models across long-horizon code refactoring, contract analysis, and agentic workflows22. However, running GLM-5.2 over the colibri CPU disk-streaming engine creates a latency profile that is functionally unusable for interactive work7.

Because colibri generates between 0.3 and 1.2 TPS, an employee requesting a synthesis of a recorded Microsoft Teams discussion would wait 15 to 30 minutes for a response7. The model's frontier intelligence is effectively neutralized by the physical I/O limits of reading NVMe storage per token7. Consequently, self-hosting GLM-5.2 for collaborative enterprise tasks requires dedicated, high-cost multi-GPU clusters (a2-ultragpu or a3-highgpu) to deliver acceptable interaction speeds8.

## **Total Cost of Ownership: Enterprise Implementation and Operations**

A common pitfall in enterprise IT planning is equating raw cloud infrastructure costs with the Total Cost of Ownership (TCO). While compute infrastructure represents nearly 100% of the cost of SaaS licensing, it typically accounts for less than 30% of the true cost of a private cloud deployment.

### **Upfront Engineering and Integration Scope**

Microsoft Teams Copilot does not function merely as an isolated text model; it operates as an integrated enterprise collaboration layer. Replicating this functionality on GCP requires substantial custom software engineering across several critical layers.

First, extracting live meeting audio from Teams requires building a custom application using the Microsoft Graph Communications Calling API. This service must capture WebRTC and SRTP real-time audio streams across separate voice channels, process them through high-accuracy automatic speech recognition engines such as Whisper Large-v3, perform speaker diarization to associate spoken transcripts with corporate Entra ID identities, and route the text into the inference pipeline39.

Second, the model must index documents stored across SharePoint sites, OneDrive directories, and Exchange mailboxes. Engineering this pipeline requires building custom extractors, setting up vector databases such as pgvector or Milvus on GCP, and managing continuous document synchronization pipelines. Developing a robust, production-grade custom Teams AI bot with streaming audio integration, vector indexing, and backchannel user interface components requires four to six specialized software engineers over a six-to-nine-month period. At an average fully loaded enterprise compensation rate of $180,000 per engineer annually, the upfront software engineering investment ranges between $360,000 and $540,000 before serving the first production query30.

### **Governance, Security, and Permission Trimming**

The most complex operational challenge in self-hosted RAG implementations is authorization and permission trimming. Microsoft 365 Copilot inherits security boundaries directly: when a user issues a query, the underlying Microsoft Graph search only accesses documents for which that specific user holds explicit Entra ID Read permissions10.

Recreating this access model on self-hosted GCP infrastructure presents significant security and compliance hurdles. Implementing permission-aware vector search requires injecting Access Control Lists (ACLs) directly into vector embeddings or metadata filtering pipelines10. In enterprise environments where document permissions frequently shift across security groups, keeping a private vector database synchronized with corporate access controls requires complex event-driven reindexing systems10.

Furthermore, if an enterprise RAG service queries an open-source model using a centralized service account rather than user-delegated tokens, security trimming defaults to application-level logic36. Any misconfiguration or software defect can expose confidential HR records, executive communications, or pre-release earnings data to unauthorized employees, introducing severe data governance risks recognized under OWASP LLM09: Vector and Embedding Weaknesses10.

### **Ongoing Maintenance, MLOps, and Operational Overhead**

Maintaining a self-hosted AI compute platform requires continuous operational support. Operating a high-availability Google Kubernetes Engine (GKE) cluster equipped with multi-GPU nodes requires 1.5 to 2.0 full-time equivalent (FTE) MLOps and Site Reliability Engineers, translating to $270,000 to $360,000 annually. Responsibilities include managing node health, handling GPU driver updates, resolving NCCL interconnect failures, scaling Triton or vLLM inference containers, and maintaining fallback configurations41.

Cluster availability and redundancy introduce further costs. A single 8x H100 node on GCP represents a single point of failure. Delivering an enterprise-grade 99.9% service level agreement (SLA) requires provisioning at least two active nodes across multiple zones, effectively doubling base infrastructure costs to over $56,000 monthly under committed use discounts8.

### **Legal Governance and Intellectual Property Indemnification**

Using commercial models introduces legal protections that open-source models do not provide. Commercial enterprise subscriptions include the Microsoft Customer Copyright Commitment (CCC)31. Under this framework, Microsoft legally defends enterprise customers and pays damages resulting from third-party copyright infringement claims generated by Copilot outputs, provided the customer used the system's built-in guardrails31.

When an enterprise hosts an open-weight model like GLM-5.2 on its own cloud infrastructure, it assumes full legal liability. Open-weight models are trained on massive public web datasets that may contain copyrighted material6. The enterprise bears complete responsibility for output filtering, intellectual property claims, and compliance audits under emerging regulatory standards such as the European Union AI Act7.

### **Comprehensive Three-Year TCO Comparison (1,000 Enterprise Users)**

The following financial model compares the three-year Total Cost of Ownership for an enterprise with 1,000 knowledge workers consuming an average of 2 million tokens per user per month, representing a total volume of 24 billion tokens over 36 months. Copilot 3-year TCO represents 1,000 users at $30 per month over 36 months1. The GCP self-hosted 3-year TCO sums upfront research and development, 36 months of compute infrastructure, 3 years of allocated MLOps personnel, and external security auditing.

&nbsp;

| Expense Category | Microsoft 365 Copilot (1,000 Seats) | Self-Hosted 8B 4-bit on GCP (High-Availability G2 GPU) | Self-Hosted GLM-5.2 744B on GCP (High-Availability 8x H100 GPU) |
| :---- | :---- | :---- | :---- |
| **Upfront Engineering & Bot Development** | $0.00 (Native Turnkey Activation) | $250,000 (Teams bot, basic RAG)30 | $450,000 (Teams bot, complex RAG, dynamic routing)30 |
| **Direct Cloud Compute (36 Months)** | $0.00 (Included in seat license) | $74,298 (2x g2-standard-4 3-Yr CUD)12 | $2,042,481 (2x a3-highgpu-8g 3-Yr CUD for HA)24 |
| **Storage, Vector DB, and Ingestion I/O** | $0.00 (Included in tenant Graph) | $36,000 (Cloud SQL pgvector \+ Storage) | $90,000 (Dedicated Milvus Cluster \+ Cloud Storage) |
| **MLOps & DevOps Personnel (3 Years)** | $0.00 (Fully managed by Microsoft) | $450,000 (0.75 FTE allocation) | $900,000 (1.5 FTE dedicated allocations) |
| **Security Auditing, Compliance & Pen-Testing** | $0.00 (Covered under EDP / ISO / SOC2)31 | $90,000 (Third-party vector/RAG audits) | $150,000 (Full-stack AI safety & pen-testing) |
| **Software Subscriptions (Copilot Seats / Licenses)** | $1,080,000 ($30/user/mo over 36 mos)1 | $0.00 (Open-source Apache 2.0 / MIT)7 | $0.00 (Open-weight checkpoint)33 |
| **Total 3-Year TCO** | **$1,080,000** | **$900,298** | **$3,632,481** |
| **Effective Cost per User per Month** | **$30.00** | **$25.01** | **$100.90** |

This multi-year financial model demonstrates that hosting a frontier 744B parameter model on GCP does not reduce costs relative to Microsoft Copilot licensing; rather, it increases total expenditure by more than 300% ($100.90 per user per month versus $30.00). While deploying an 8B model reduces three-year costs by approximately 16% ($25.01 per user per month), it does so by sacrificing advanced reasoning capabilities, deep cross-document analysis, and reliable meeting summarization.

## **Strategic Decision Framework**

Deciding between self-hosted models on GCP and commercial Microsoft 365 Copilot licensing requires evaluating specific operational trade-offs across security, performance, engineering capacity, and financial return.

The primary architectural filter rests on whether native Microsoft Teams meeting summarization and dynamic SharePoint grounding are mandatory requirements. If deep integration into Microsoft 365 collaboration tools is essential, the enterprise must evaluate whether it maintains the internal software engineering capacity to construct WebRTC audio bots and manage real-time Entra ID access control synchronization.

In the absence of dedicated engineering teams, commercial Microsoft 365 Copilot remains the only viable path to production. Even if engineering capacity exists, electing to self-host requires strict regulatory or air-gap data sovereignty mandates that legally prohibit using commercial SaaS APIs; otherwise, the multi-million-dollar premium of building and maintaining a private infrastructure stack cannot be justified.

Conversely, when enterprise requirements center on task-specific, non-interactive, or automated agentic workflows, self-hosting becomes advantageous. For lightweight classification, entity extraction, or structured data transformations, self-hosting an 8B 4-bit model on GCP using entry-level L4 GPUs provides exceptional unit economics at high request volumes. For proprietary software engineering, deep codebase reasoning, or long-document analysis, GLM-5.2 running on dedicated multi-GPU clusters offers competitive capabilities, provided the enterprise operates at sufficient concurrency to saturate the hardware cluster.

&nbsp;

| Decision Factor | Self-Hosted Open-Source on GCP (8B / GLM-5.2) | Commercial Microsoft 365 Copilot SaaS |
| :---- | :---- | :---- |
| **Data Sovereignty & Boundary Control** | **Superior:** Complete enterprise custody. Weights and prompts remain within a dedicated Virtual Private Cloud (VPC) with zero external exposure. | **Moderate to Strong:** Protected by Enterprise Data Protection (EDP), but dependent on Microsoft tenant data governance boundaries31. |
| **Time to Value and Deployment Speed** | **Slow:** 6 to 9 months required to architect pipelines, integrate APIs, validate ACL trimming, and stabilize nodes30. | **Immediate:** Enabled across the corporate tenant via an administrative toggle in the Microsoft 365 Admin Center30. |
| **Customizability and Extensibility** | **High:** Full architectural control to customize inference engines, fine-tune weights on internal corpora, and modify system prompts22. | **Constrained:** Configuration limited to Copilot Studio plugins, Graph connectors, and administrative toggles28. |
| **Meeting & Collaboration User Experience** | **Fragmented:** Custom bots introduce noticeable UI friction, require separate invites, and lack inline Office capabilities34. | **Seamless:** Deeply integrated into Teams meetings, Word, Excel, PowerPoint, and Outlook28. |
| **Financial Risk Profile** | **High Fixed Capital Risk:** Fixed multi-year cloud commitments create financial exposure if user adoption lags24. | **Linear Predictable Risk:** Predictable annual operating expense directly tied to active user count1. |

## **Nuanced Conclusions and Strategic Recommendations**

The comparative analysis reveals that replacing Microsoft 365 Copilot with a self-hosted open-source model is rarely justified on direct cost alone. The financial and operational conclusions depend heavily on the model architecture and organizational scale:

First, pure-C disk-streaming inference engines like colibri represent impressive technical achievements for running massive models on accessible consumer hardware, but their generation throughput of 0.8 TPS makes them operationally unviable for interactive enterprise workloads7. Deploying GLM-5.2 over CPU disk streaming costs $166.67 per million tokens under continuous execution while introducing unacceptable latency for standard knowledge work7.

Second, hosting an 8B 4-bit model on GCP using a g2-standard-4 GPU instance is highly cost-effective, running at $0.39 to $2.62 per million tokens12. However, this class of model cannot replace Microsoft Teams Copilot. Its limited parameter capacity lacks the reasoning depth, attention retention, and long-context processing needed to synthesize multi-speaker meetings or parse complex enterprise documentation without hallucination22.

Third, hosting GLM-5.2 on dedicated enterprise GPU hardware using an a3-highgpu-8g instance running eight NVIDIA H100s delivers high-throughput, frontier-grade intelligence8. However, the substantial base infrastructure expense of $28,367 per month under a 3-year commitment means that for organizations with fewer than 1,000 seats, direct compute costs alone match or exceed Copilot's $30.00 per-seat licensing1. When accounting for the development and maintenance of real-time Teams media bots, vector databases, and permission-trimming systems, the 3-year Total Cost of Ownership reaches $100.90 per user per month, more than three times the cost of Copilot SaaS licensing.

Enterprises should adopt a bifurcated hybrid AI strategy. Standard knowledge workers should be equipped with Microsoft 365 Copilot at the flat $30.00 per-user rate1. This avoids extensive custom engineering, natively secures sensitive corporate data through inherited Entra ID permissions, eliminates infrastructure maintenance, and provides legal indemnification through Microsoft's Customer Copyright Commitment31.

Simultaneously, enterprises should deploy focused GCP compute resources strictly for specialized, non-Teams workloads that require custom fine-tuning, automated batch processing, or sovereign data isolation. High-volume document classification and text triage should run on lightweight 8B models using single L4 GPU instances (g2-standard-4)12. Dedicated multi-GPU clusters (a3-highgpu-8g) should be reserved exclusively for proprietary, high-value engineering domains where advanced reasoning over sovereign datasets justifies the associated infrastructure and operational investments8.

#### **Works cited**

> 1. Microsoft Copilot Enterprise Pricing 2026 \- Coworker AI, [https://coworker.ai/blog/microsoft-copilot-enterprise-pricing](https://coworker.ai/blog/microsoft-copilot-enterprise-pricing)  
> 2. Microsoft Copilot Pricing Explained: Plans, Cost & Licensing (2026, [https://copilot-experts.com/microsoft-copilot-pricing-guide/](https://copilot-experts.com/microsoft-copilot-pricing-guide/)  
> 3. Microsoft 365 Copilot Plans and Pricing—AI for Enterprise, [https://www.microsoft.com/en-us/microsoft-365-copilot/pricing/enterprise](https://www.microsoft.com/en-us/microsoft-365-copilot/pricing/enterprise)  
> 4. GLM-5.2 744B: Sparse Attention Meets Efficient MoE \- Luca Berton, [https://lucaberton.com/blog/glm-5-2-744b-moe-architecture-2026/](https://lucaberton.com/blog/glm-5-2-744b-moe-architecture-2026/)  
> 5. GLM-5.2 \- How to Run Locally | Unsloth Documentation, [https://unsloth.ai/docs/models/glm-5.2](https://unsloth.ai/docs/models/glm-5.2)  
> 6. GLM-5: From Vibe Coding to Agentic Engineering \- Z.ai, [https://z.ai/blog/glm-5](https://z.ai/blog/glm-5)  
> 7. colibri: GLM 5.2 (744B) on 25 GB of RAM, streaming the experts, [https://www.noze.it/en/insights/colibri-glm-5-2-local/](https://www.noze.it/en/insights/colibri-glm-5-2-local/)  
> 8. GPU Instance Pricing \- AWS, Google Cloud & Azure GPUs for AI/ML, [https://www.doit.com/compute/gpu](https://www.doit.com/compute/gpu)  
> 9. GPU Cloud Pricing 2026: What AI Compute Really Costs, [https://cast.ai/blog/gpu-cloud-pricing/](https://cast.ai/blog/gpu-cloud-pricing/)  
> 10. private RAG authorization-aware retrieval security \- M365 FM Podcast, [https://www.m365.fm/private-rag-isnt-enough-the-missing-layer-between-data-sovereignty-and-data-security/](https://www.m365.fm/private-rag-isnt-enough-the-missing-layer-between-data-sovereignty-and-data-security/)  
> 11. Home | Microsoft AI Decision Framework, [https://microsoft.github.io/Microsoft-AI-Decision-Framework/](https://microsoft.github.io/Microsoft-AI-Decision-Framework/)  
> 12. Google Cloud GPU Pricing, [https://cloudgpuprices.com/vendors/google-cloud](https://cloudgpuprices.com/vendors/google-cloud)  
> 13. c3-standard-4 pricing: $147.17 monthly | GCP Compute Engine, [https://www.economize.cloud/resources/gcp/pricing/compute-engine/c3-standard-4/](https://www.economize.cloud/resources/gcp/pricing/compute-engine/c3-standard-4/)  
> 14. GLM 5.2 model — 744 billion parameters / 384 GB — running on a, [https://www.reddit.com/r/LocalLLM/comments/1vmt0kr/glm\_52\_model\_744\_billion\_parameters\_384\_gb/](https://www.reddit.com/r/LocalLLM/comments/1vmt0kr/glm_52_model_744_billion_parameters_384_gb/)  
> 15. Memory-optimized VM Pricing | Google Cloud, [https://cloud.google.com/products/compute/pricing/memory-optimized](https://cloud.google.com/products/compute/pricing/memory-optimized)  
> 16. Pricing & Price-Performance \- CloudBench, [https://www.cloudbench.dev/pricing](https://www.cloudbench.dev/pricing)  
> 17. Google Compute Engine Machine Type g2-standard-24, [https://gcloud-compute.com/g2-standard-24.html](https://gcloud-compute.com/g2-standard-24.html)  
> 18. a2-ultragpu-8g pricing: $29601.78 monthly | GCP Compute Engine, [https://www.economize.cloud/resources/gcp/pricing/compute-engine/a2-ultragpu-8g/](https://www.economize.cloud/resources/gcp/pricing/compute-engine/a2-ultragpu-8g/)  
> 19. Accelerator-optimized VM Pricing \- Google Cloud, [https://cloud.google.com/products/compute/pricing/accelerator-optimized](https://cloud.google.com/products/compute/pricing/accelerator-optimized)  
> 20. [unknown\_url](http://docs.google.com/unknown_url)  
> 21. Google Compute Engine Persistent Disk Types, [https://gcloud-compute.com/disks.html](https://gcloud-compute.com/disks.html)  
> 22. In-Depth Analysis of GLM-5.2's 744B Parameter Open-Source Model, [https://intl.cloud.baidu.com/en/article/8757153](https://intl.cloud.baidu.com/en/article/8757153)  
> 23. a3-highgpu-8g pricing and specs \- Holori, [https://calculator.holori.com/gcp/vm/a3-highgpu-8g](https://calculator.holori.com/gcp/vm/a3-highgpu-8g)  
> 24. H100 GPU Cloud Pricing Comparison: AWS vs GCP vs Azure, [https://gpusmith.com/articles/en/h100-gpu-cloud-pricing-comparison](https://gpusmith.com/articles/en/h100-gpu-cloud-pricing-comparison)  
> 25. Token Usage Guide 2026: How Many Tokens AI Really Uses, [https://iternal.ai/token-usage-guide](https://iternal.ai/token-usage-guide)  
> 26. The Token Cost Predictability Nightmare of Open-Ended ... \- Medium, [https://medium.com/@khayyam.h/the-token-cost-predictability-nightmare-of-open-ended-autonomous-agents-b4cb909ff313](https://medium.com/@khayyam.h/the-token-cost-predictability-nightmare-of-open-ended-autonomous-agents-b4cb909ff313)  
> 27. Tokens Are Becoming New Standard to Measure Tech Firm's ... \- Yicai, [https://www.yicaiglobal.com/news/tokens-are-becoming-new-standard-to-measure-tech-firms-competitiveness](https://www.yicaiglobal.com/news/tokens-are-becoming-new-standard-to-measure-tech-firms-competitiveness)  
> 28. Guide to Microsoft Copilot Pricing & Licensing \- AlphaBOLD, [https://www.alphabold.com/guide-to-microsoft-copilot-pricing-licensing/](https://www.alphabold.com/guide-to-microsoft-copilot-pricing-licensing/)  
> 29. Microsoft Copilot Pricing 2026: Plans, License Cost, and Total Cost, [https://www.gosearch.ai/blog/microsoft-copilot-pricing/](https://www.gosearch.ai/blog/microsoft-copilot-pricing/)  
> 30. Copilot Is an Imposition – My Experience After Real‑World Use, [https://www.reddit.com/r/microsoft\_365\_copilot/comments/1sd1vjz/copilot\_is\_an\_imposition\_my\_experience\_after/](https://www.reddit.com/r/microsoft_365_copilot/comments/1sd1vjz/copilot_is_an_imposition_my_experience_after/)  
> 31. Anthropic in Microsoft 365: Panic or Pragmatic Governance?, [https://www.copilot-coaching.de/en/blog/anthropic-in-m365-panik-oder-pragmatismus/](https://www.copilot-coaching.de/en/blog/anthropic-in-m365-panik-oder-pragmatismus/)  
> 32. zai-org/GLM-5.2 \- Hugging Face, [https://huggingface.co/zai-org/GLM-5.2](https://huggingface.co/zai-org/GLM-5.2)  
> 33. GLM-5.2: Features, Setup, Benchmarks, and Model Switching Guide, [https://www.datacamp.com/blog/glm-5-2](https://www.datacamp.com/blog/glm-5-2)  
> 34. Comprehensive Guide to Effective Microsoft Copilot Prompts, [https://www.visualsp.com/blog/comprehensive-guide-to-effective-microsoft-copilot-prompts/](https://www.visualsp.com/blog/comprehensive-guide-to-effective-microsoft-copilot-prompts/)  
> 35. Microsoft Copilot Complete Guide 2026: Features, Pricing, and How, [https://aitoolsdevpro.com/ai-tools/microsoft-copilot-guide/](https://aitoolsdevpro.com/ai-tools/microsoft-copilot-guide/)  
> 36. AI Access Exposure Review: Copilot Readiness \- IntuitionLabs, [https://intuitionlabs.ai/services/ai-security/ai-access-exposure-review](https://intuitionlabs.ai/services/ai-security/ai-access-exposure-review)  
> 37. CHANGELOG.md \- Azure/gpt-rag-orchestrator \- GitHub, [https://github.com/Azure/gpt-rag-orchestrator/blob/main/CHANGELOG.md](https://github.com/Azure/gpt-rag-orchestrator/blob/main/CHANGELOG.md)  
> 38. Token Economics: Understanding the True Cost of LLM Operations, [https://www.plexor.dev/blog/token-economics-llm-costs](https://www.plexor.dev/blog/token-economics-llm-costs)  
> 39. Microsoft Teams transcription via API \- Gladia, [https://www.gladia.io/blog/microsoft-teams-transcription-via-api](https://www.gladia.io/blog/microsoft-teams-transcription-via-api)  
> 40. Building an AI Microsoft Teams Bot with LangGraph and Azure OpenAI, [https://blogg.sikt.no/building-an-ai-microsoft-teams-bot-with-langgraph-and-azure-openai-bb8bf2270c1d](https://blogg.sikt.no/building-an-ai-microsoft-teams-bot-with-langgraph-and-azure-openai-bb8bf2270c1d)  
> 41. Why Desktop AI Agents Need AI Gateways, [https://api7.ai/blog/why-desktop-ai-agents-need-ai-gateways](https://api7.ai/blog/why-desktop-ai-agents-need-ai-gateways)  
> 42. Managed Database Services: Oracle, SQL Server & PostgreSQL, [https://datrick.com/data-operations.html](https://datrick.com/data-operations.html)  
> 43. Microsoft Security Copilot Data and Compliance Frequently Asked, [https://learn.microsoft.com/en-us/copilot/security/faq-data-compliance](https://learn.microsoft.com/en-us/copilot/security/faq-data-compliance)  
> 44. Microsoft Copilot is now generally available, [https://www.microsoft.com/en-us/microsoft-copilot/blog/2023/12/01/microsoft-copilot-is-now-generally-available/](https://www.microsoft.com/en-us/microsoft-copilot/blog/2023/12/01/microsoft-copilot-is-now-generally-available/)

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACkAAAAaCAYAAAAqjnX1AAACCklEQVR4Xu2WvytGYRTHj1Dkd8mPKJKBDPJzYkMZDCwGJjbZDMom+QcYlFFhYJMMlCuDwcpiQkkUgzLJj+/XeR7v81xeveS9Ge63PnXvOfd6v/c5zzkPkVix/rcyQRmoNOT46U/KBuWSeD7DT6dHNMUfuwIvYA/keU/4ugWvYEL0PX5kZDoDa+AUlIZyVrVgQ9RktZ+KRgugBzyCtlCO6gUr4FrUZOQqACOgWdTAgJ9+1xZoEt0SJHINgRJRszQ576elBRSJ7l/mt/10NGKpqSxRE6tOLhesm+s60fx0Ih2NGkWbxoomLpx7ltk2CVf4HFR9ZH+mZ/nlfmapj517/pE7c82VHTbXLDXLTNxZWgzGQ7Fk4nY6DAdTEV9i01g9iBrlgN914v0mzgngahTsg/xQ/CvxmSAcTEUsNbvaiqWmmVkw5cRZasbDpb4ER6Be/NXkyTQj/qRwTdpDhHwrHmlsEvdo2xQ10+3EqGTzMTC4K3kC5pz7G9Hx5ZqsFZ29SY9VJgZFu5YrMSaJ442dTjP2Za5cn4mRTtGtYBUYXJOco+624AHBSliTHaLHb2QKDDTQamJPoMtcUzTJClmTDaIm3Y9NqwIDDSyaGFecU8OKpifFLzcrsiQ6QdKudtH/jHZAjYkVihpeBgegwsTvRT+AzcrtYLdQJCvKTv7qh9i5qYymWLFi/YXeABmUZlZrX8FeAAAAAElFTkSuQmCC>

[image2]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAmwAAAA9CAYAAAAQ2DVeAAAEe0lEQVR4Xu3dPYhdRRgG4CNGUPxJYUDEQERQEO3EiKCNCNpoE0RLwcLCMoJgtRY2FiKCCmKTQm3SqRDB4oIggp0gkTTGRkFQQUgh4s+83Bl2djz765rN7j4PfJyZuXfnnu1ezjkzZ5oAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADoPXS/1Z6u/xgxmnSr1b2xdKnajtk6WO1vZn9dja+Sz+KnVV99ko32t/u5U5ch45n8j5AwAcaLdOWwts+c6Ntf1QqYu1/Xs9xh2l7qvtfs6zpV7q+qPMcby2N5vjyDD+6rRxGAQA2Pe2E9ia9jc3DOPpf1Lbl7rxBK2fu36vzZFj6280xxPT2t9M/9GuDwBw4Ox2YFvU9hi2+n5vLrAtantujrnAlgIAOLAENgCAK9x2AlsLVXnGLIsA2niTuZ6fGT8zrd7mnJPv5m9jszluGcYT5I51fQCAA2cusKX/zDB2vtS9tZ2Q9EZtn6vHeLLUdbWdOdpigIul7qntrOr8sLabzPF4bW9ljoTF9p1FPQIAHDofjQPFzaVeK3X1MP5UqTdLXTOMPzCtbgXS+3gcKK6ftjfHXaVOj4MAAJtJ4MhVodQ3daz1Uwkl+0FC03iFbTd9MA4AAFxOf0zLvcF64y3HK91348Au+nSybxoAsMcSzq7t+tkI9v2uDwDAHmurJ5tcbbt/GAMAYI/klU0b3Q7NA/RfTWtf09S/2il+Gvq7oX+O7rAVAMAaZ6flLdAmiwzG0JA9zPrAlj3HspVF5BVL2W8MAID/yRjOcrUtixB6Y2CLdhv1sWnjwJZn47Jv2no1brcBAECV/clemJaB7c46dnJa3u78dlq7CGEusK2UerC2NwpsAABcBnOBLbv2/1LbexXY5q7ezXmr1NvT8nbvi8NnAAD8B4uuncDYXph++7QaEscXqz9d+23ftP6zjO1GuPxhWjt/e+XU0VLP1vYX9Rh5tVXTvwEh49nw90SpC904AMC+8XXX7gNbLOpxDGx52Xv6WRgxvnQ9diOwZc52xTHtdl796tkW0o6Uerkbb2Eyb2X4vBsfnxsEANh3xsDWjIEtL3tPP8/oRYJT+llMMb7zs9eudDUrQ39OgtevtZ3tUfI72eIkx0fqeM6vP+98lpW1309rw2P6AAD72maBLbcaU2kf7z7PKtSMteo/G70zLUPaGN7m5OpYnuG7qfZzfn1wbEFuLrClL7ABAAfOZoFtK05Nq0FqPQlt58fBDeS3H57+HdgW0/KWrMAGABwaOwlsWTXaNvht1vtubPUKW+bIs3JxqdRvpW6r481iWv5+Qlt73i3ynWOl3qvfaTIPAMC+ttPA1q/QjC+HfjOGtJWh38vvtefh0l6p7f6drD927XNdu51Pglx/tW+9/wEA4EDLVbAEq7un5fNtc4Fvp7IKtd+io3ml1OlhLIsT5sbjuWn5vlYAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACY8Q/14DB9N1auegAAAABJRU5ErkJggg==>

[image3]: <data:image/png;base64,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>

[image4]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACcAAAAaCAYAAAA0R0VGAAABjklEQVR4Xu2VvytFYRzGHzEQg0EkWZSNkcmoTIyyGwirjO5ik7LaFaVY7MriH1AWgwETFhTy43n6vqd777f33ut29d6rzqeeuu/7nNN57vs93+8BcnKaSyfV6zdbhR5qkHqgvqnxsM7Q75ngnYe17knKByxAjDmYt+SNFHTAHn7vDdgpnVGXVF+5lQY9VOFOvUFGYKH3qTbnJUHl+qKmvUG2YMEnvJEClfQIFu6Y2nPSqSlcU0o6BWsGnVAMBavUKO3UfMl6mXqGXX9Rsp+xBvM+qUPnRVFJdUOspELeq9vronapW+rFeeIa9oc9J9QbykdVVdSFCtDtDVgzyFMzxNhAPJxOU8015PY1L29QR7hG5lulcLPUQlDGADWMOsI1Ot+qhdPYeaLGYM8pwELVDKcbJ6lVWLgrahQWSC95P7UOez/kqUz6BnuqhROqSgEWcBO/DPdX1Aqn7tSI2kbxG94y4XRiKu1iWCcJp85+RHEG3lE71ApshmnvPVyrptI7p3u0J0+z8CD4OTk5/5IfBaljhkzAM7oAAAAASUVORK5CYII=>

[image5]: <data:image/png;base64,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>