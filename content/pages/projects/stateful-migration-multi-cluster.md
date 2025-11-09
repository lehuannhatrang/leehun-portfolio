---
type: ProjectLayout
title: Workload Live Migration in Multi Cluster Environment
date: '2025-05-01'
client: DCN Lab
description: >-
  This project enables true stateful live migration in a multi-cluster
  environment. It's designed to move complex applications from one cluster to
  another by checkpointing the entire workload state, including CPU and memory
  data, and replicating its persistent volumes (PV).
featuredImage:
  type: ImageBlock
  url: /images/migration-no-velero.drawio.png
  altText: Project thumbnail image
  caption: ''
  elementId: ''
media:
  type: ImageBlock
  url: /images/migration-no-velero.drawio.png
  altText: Project image
  caption: System Architecture
  elementId: ''
addTitleSuffix: true
colors: colors-a
backgroundImage:
  type: BackgroundImage
  url: /images/bg2.jpg
  backgroundSize: cover
  backgroundPosition: center
  backgroundRepeat: no-repeat
  opacity: 100
bottomSections:
  - type: LabelsSection
    title: Keywords
    subtitle: ''
    items:
      - type: Label
        label: Kubernetes
        url: ''
      - type: Label
        label: Multi-Cluster
        url: ''
      - type: Label
        label: Karmada
        url: ''
      - type: Label
        label: CRIU
        url: ''
      - type: Label
        label: Migration
        url: ''
      - type: Label
        label: Workload Resilience
        url: ''
      - type: Label
        label: Golang
        url: ''
      - type: Label
        label: Kubernetes Operator
        url: ''
    colors: colors-f
    elementId: ''
    styles:
      self:
        height: auto
        width: wide
        padding:
          - pt-36
          - pb-36
          - pl-4
          - pr-4
        textAlign: left
---
This project delivers a complete solution for live-migrating stateful workloads (such as databases or AI models) between different, geographically distributed Kubernetes clusters with near-zero downtime. The primary challenge is preserving the *entire* application state, which includes not only its Persistent Volume (PV) data but also its live in-memory state (CPU and RAM).

**Orchestration Layer: Karmada**

The system is built on Karmada, which acts as the multi-cluster control plane. Karmada orchestrates where workloads are placed, making it the ideal layer to initiate and manage the cross-cluster migration policy. However, Karmada itself does not have the built-in capability to checkpoint and restore a running application's memory.

Core Migration Technology: CRIU & The "Checkpoint Agent"

To solve the in-memory state problem, I leveraged CRIU (Checkpoint/Restore In Userspace). CRIU is a powerful Linux utility that can freeze a running application, save its complete memory and CPU state to disk, and then restore it later, even on a different machine.

The primary technical contribution of this project was developing a custom "Checkpoint Agent" (as a DaemonSet) that runs on every node in the cluster. This agent acts as the "brain" of the migration process and bridges the gap between Kubernetes and CRIU:

1.  Integration with Kubernetes: The agent interfaces directly with the `kubelet`'s built-in checkpointing feature, which was introduced to support this exact use case.

2.  Integration with Container Runtime: When a migration is triggered, the agent calls the `kubelet` API. The `kubelet` then commands the container runtime, CRI-O, to perform the checkpoint. CRI-O has native support for this, using CRIU as its underlying mechanism to freeze the container's processes and dump their state.

**The Migration Flow Explained:**

When an administrator triggers a migration for a stateful application:

1.  **Initiation**: Karmada updates the workload's policy, signaling the intent to move it from Cluster A to Cluster B.

2.  **Checkpointing**: My "Checkpoint Agent" on the source node detects this and invokes the kubelet/CRI-O checkpointing API. CRIU freezes the application and creates a memory state dump.

3.  **Data** Replication: Simultaneously, the system snapshots the application's Persistent Volume (PV) and begins replicating it to the storage system in Cluster B.

4.  **Transfer**: The memory state dump (which is relatively small and fast to move) is transferred to the target node in Cluster B.

5.  **Restore**: Once the PV data and memory dump are in place, the "Checkpoint Agent" on the target node uses CRI-O/CRIU to restore the application from the checkpoint. The application resumes execution precisely where it left off, attached to its replicated PV.

This process enables true, stateful live migration, allowing for cluster maintenance, load balancing, and failover with minimal service interruption.
