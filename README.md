# **Guidebooks and Resources in this Repository**

# **GOOGLE GEMMA 4 Technical Summary** 

Version 2.0 (Revised with Model Card & p-RoPE Analysis)

On 2 April 2026, Google DeepMind released Gemma 4, a family of four open-weight multimodal models based on the Gemini 3 research. 

Gemma 4 is the first in the series to use the Apache License 2.0, replacing the earlier Gemma license, which limited use and allowed unilateral modifications by Google. The family includes four models, ranging from sub-3B edge models to a 31B dense model, supporting text, images, and audio, with context windows up to 256K tokens. All are available on Hugging Face, Kaggle, and Ollama, with day-one support for inference frameworks like llama.cpp, vLLM, MLX, and Ollama.


# **TeamPCP Supply Chain Attack Campaign 20260328**

I compiled a technical report on the TeamPCP supply chain attack campaign, a significant multi-ecosystem credential-chaining operation. This incident demonstrates how Team PCP exploited security scanners and the CI/CD pipeline for weeks.

The report analyses several key elements, including:
 • The claw hackerbot for initial access.
 •⁠ ⁠The CanisterWorm, the first self-propagating npm worm using blockchain for C2.
 •⁠ ⁠A Kubernetes wiper targeting Iran.
 •⁠ ⁠The LiteLLM auto-execution trick with .pth files.
 •⁠ ⁠The Telnyx WAV steganography method for malware delivery.

Included are IOCs with hashes, C2 infrastructure details, and behavioural hunt indicators sourced from organisations such as Wiz, Datadog, Microsoft, and more than 20 others.

An impact assessment compares this campaign to SolarWinds and Log4j, highlighting that while it may be less ubiquitous and strategic, it poses a greater structural threat by weaponising build processes.

Actionable remediation strategies are provided for immediate to medium-term responses.

The main takeaway is that the risk initially lies not in mass exploitation but in the theft of secrets that can compromise CI/CD processes and, in turn, downstream software. If your organisation uses related tools or manages sensitive credentials, this report will be valuable.

#TeamPCP #SupplyChainSecurity #CICD #DevOps #CyberSecurity


# **Your Secrets Are Already in the Cloud. Your AI Coding Agent Sent Them There.**

How Cloud AI Coding Agents Silently Expose Your Secrets — Without Warning

Deep in an agentic coding session with Claude Code, the agent was scaffolding files — normal workflow. Then this:
❌ API Error: 400 — "Output blocked by content filtering policy"

I ignored it three times. Continue. Retry. Continue. 
My final reaction was very different.

The filter wasn't broken. It was working. It detected a credential pattern and blocked it — but by the time it fired, the request had already left my machine, carrying everything the agent was authorised to read from my project directory. Including my .env.local file. Including the secrets inside it.
This isn't a Claude Code bug. It isn't a developer mistake. It's how every cloud AI coding agent works. 
I documented the full chain and wrote up the lessons learned — link in the comments.

#AISecurity #DeveloperSecurity #AIAgents

===========================

# **30 Best Practices That Make Claude Cowork 100x More Powerful**

Adapted and expanded from Nav Toor's original "17 Best Practices That Make Claude Cowork 100x More Powerful."

Build on Nav's foundational work; an additional 13 practices identified through Anthropic's official documentation, community workflows, and independent research.

* **Part 1:** Context Architecture (Practices 1–5)
* **Part 2:** Task Design (Practices 6–10)
* **Part 3:** Automation & Scheduling (Practices 11–13)
* **Part 4:** Plugins & Skills (Practices 14–16)
* **Part 5:** Safety & Efficiency (Practice 17)
* **Part 6:** Continuity & Quality Assurance (Practices 18–22)
* **Part 7:** Workflow Intelligence (Practices 23–30)

===========================

# **Secure Coding: The Dangers of Unhandled Errors**

Lessons learned from The Cloudflare Nov 18, 2025 outage

Yesterday's massive Cloudflare outage highlights a critical principle in secure coding: error handling must function as a security boundary. The failure was caused not by a cyberattack, but by a backend configuration change that resulted in an oversized feature file exceeding a 200-feature resource limit in the FL2 Rust proxy engine. 

The software written in RUST failed because it used unsafe code. .unwrap() method to process the check, which, upon encountering the Err (error) value, immediately triggered a panic! And crashed the entire worker thread. This flaw converted a predictable resource validation failure into an uncontrolled system crash (Denial-of-Service), demonstrating that developers must replace fatal methods like .unwrap() with secure alternatives like the ? Operator to gracefully propagate errors and maintain system stability.

===========================

# **AWS DynamoDB Comprehensive Review Notes**

This document provides a detailed technical reference and architectural best practices guide for Amazon DynamoDB, AWS's fully managed, high-performance NoSQL key-value and document database service.

## **Key Takeaways**

**Foundational Concepts:** Comprehensive explanation of primary key types (Partition and Composite) and the schema-less attribute data type descriptors (S, N, L, M, etc.) required for API interactions.

**CLI Operations:** Step-by-step AWS CLI commands are provided for administrative tasks (Table creation, description, deletion) and core data manipulation (PutItem, UpdateItem, GetItem, Query, and Scan).

**Read Consistency & Capacity:** Distinguishes between Eventually Consistent (default, cheaper) and Strongly Consistent reads, and outlines the methodology for calculating Read and Write Capacity Units (RCU/WCU) in Provisioned Mode.

**Advanced Architectural Patterns:** Deep dives into critical NoSQL best practices, including:

* Conditional Writes and Optimistic Locking for ensuring data integrity and concurrency control.
* Single-Table Design (STD), which leverages composite keys and prefixes to co-locate related entities for maximum query efficiency.
* DynamoDB Streams, TTL, DAX, and Global Tables for event-driven architecture, automated cleanup, caching, and multi-region high availability.

The notes serve as an essential resource for developers and architects seeking to optimize performance, manage cost, and ensure data integrity in DynamoDB environments.

===========================

# **Edge Appliances: The New Tier-0 Battlefield**

### **The Problem:**
Recent cyber breaches underline the importance of edge appliances in cybersecurity. These devices form the first line of defense against external threats. If compromised, they can allow attackers to infiltrate networks, steal data, and deploy ransomware.

### **The Solution:**
Treat internet-facing edges as Tier-0 infrastructure: inherently hostile, isolated from internal management, patched immediately, and equipped with independent telemetry.

Learn more about the three phrases:
1. **Prevent**: Fortify the Edge Before It’s Tested
2. **Detect**: Assume the Box Can Lie
3. **Response**: The 72-Hour Containment Cycle

Modern defense hinges on treating the perimeter as an adversarial zone, shifting the engagement terms in your favor.

===========================

# **CVE-2025-24257 iOS Integer Underflow Vulnerability: Educational Lab**

## Overview
This lab demonstrates CVE-2025-24257, an integer underflow vulnerability in Apple's IOGPUFamily kernel driver. We'll reproduce the core issue in safe userland C code, then learn to detect it through fuzzing.

**Learning Objectives:**
- Understand integer underflow vulnerabilities
- Learn safe input validation patterns
- Practice coverage-guided fuzzing with AFL++
- Apply secure coding principles

This lab takes technical reference from an article titled "Depicting an iOS Vulnerability" by Tomi Tokics, @tomitokics, from Dataflow Forensics & Ben Sparkes, @iBSparkes, from Dataflow Security. Credits to them. 
https://blog.dfsec.com/ios/2025/10/14/Depicting-an-iOS-Vulnerability/

===========================

# **Git & GitHub Automation: From Zero to CI/CD in 2 Hours**

### **The Problem:**
Companies lose $1.2 trillion annually to poor software quality, with 23% of production bugs preventable through automated testing. A startup lost $10,000 in one night because untested code reached production—a 15-line automation workflow would have prevented it entirely.

### **The Solution:**
A comprehensive but intensive 2-hour course with 70% hands-on practice hope to transforms beginners into competent DevOps practitioners. Students build a real, working CI/CD pipeline that automatically tests code, prevents bugs, and enables confident deployment. 

* **Module 1:** Git Essentials for Automation (25 min)
* **Module 2:** GitHub Actions Fundamentals (40 min)
* **Module 3:** Build Your First CI Pipeline (30 min)
* **Module 4:** Troubleshooting & Debugging (10 min)
* **Module 5:** Next Steps & Resources (5 min)

### **Target Audience:**
* The course serves individual developers, corporate training programs, and educational institutions globally. With unlimited scalability (digital, self-paced), it democratizes access to career-transforming DevOps knowledge.

===========================

# **C Secure Coding & Fuzzing Lecture Lab: Samsung QuramSoft CVE-2025-21043 (DNG Parser Bug)**

**CVE-2025-21043** was a critical vulnerability discovered in Samsung's closed-source library `libimagecodec.quram.so`. It affected the **DNG (Digital Negative)** image parser, which handles "opcode lists" inside raw images. The vulnerability allowed **remote code execution**: just receiving a malicious DNG image could compromise a device.

This lab demonstrates how small input-validation mistakes in C lead to memory corruption and remote code execution in real products. Using a compact, educational "look-alike" of Samsung's QuramSoft DNG opcode parser, students practice building, fuzzing, and triaging a heap out-of-bounds write. A fixed version models proper defenses. This lab integrates forensic-level analysis of the actual vulnerability, production detection strategies, and real-world attack campaign context.

This lab takes technical reference from reseachers listed below. Credits to them:
- **@__suto (qriousec)**: Vulnerability analysis and decompilation
- **Matt Suiche**: ELEGANTBOUNCER detection framework and forensic research

===========================

# **The Kubernetes Guidebook: Mastering Cloud-Native Orchestration From Fundamentals to Production** 

## **A Practical Journey from Core Concepts to Production Excellence**

This guidebook is designed not just to explain what Kubernetes is, but why it works the way it does, how to effectively apply it in real-world scenarios, and how to troubleshoot when things inevitably go wrong. We focus on building a strong mental model, emphasizing practical hands-on experience, and adhering to best practices for a resilient, production-ready environment.

### **Target Audience:**
* **Developers:** Who need to deploy, scale, and troubleshoot their applications on Kubernetes.  
* **DevOps Engineers:** Responsible for building and maintaining Kubernetes infrastructure and CI/CD pipelines.  
* **System Administrators:** Transitioning to cloud-native environments and managing Kubernetes clusters.  
* **Solution Architects:** Designing robust and scalable containerized solutions.

===========================

# **Technical Paper: Cloud Observability Stack with Agentic AI**

## **Today's cloud applications are complex, and traditional monitoring methods can't keep up. This leads to fragmented data, alert overload, and lost productivity for teams. we need more than just a dashboard, we need true observability to understand system behavior and fix problems fast.** 

This guidebook is designed not just to explain what a unified observability platform is, one that's a scalable and cost-effective alternative to expensive proprietary tools. It integrates metrics, logs, and traces into a single control plane using leading open-source technologies like Grafana, Loki, Tempo, and Mimir.

But what will truly sets us apart is Agentic AI. It's not a basic chatbot; it's an intelligent co-pilot that can reason through telemetry data, create investigative plans, and provide synthesized root cause analysis with clear remediation recommendations.

For Site Reliability Engineering (SRE), DevOps, and platform teams, this means:

i. Natural Language Queries: Simply ask, "Why is the checkout service slow?" and get a detailed answer, not just raw data.
ii. Proactive Operations: Agentic AI detects anomalies and anticipates outages before they happen.
iii. Faster Recovery: Our platform dramatically reduces Mean Time to Recovery (MTTR), freeing your teams from endless operational burdens.

Observability has become a strategic advantage, transforming IT team from reactive to proactive and paving the way for autonomous, self-healing systems.

### **Target Audience:** 
* **Site Reliability Engineering (SRE), DevOps, and platform teams.**

