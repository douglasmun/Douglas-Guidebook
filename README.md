# **Guidebooks and Resources in this Repository**

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

