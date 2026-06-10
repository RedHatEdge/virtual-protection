# Red Hat Virtual Protection reference architecture

The vPAC Alliance / Intel "Virtual Protection Relay" whitepaper describes the VPR
concept in **vendor-neutral** terms — an IEC 61850 digital substation whose
bay-level IEDs are collapsed into protection VMs on a real-time virtualized host.
This document keeps that reference model and plugs in the **Red Hat
implementation** for each layer: RHEL (image-mode), KVM/libvirt, the real-time
stack, Pacemaker/Ceph for high availability, and `linuxptp` for time.

Source model: *Virtual Protection Relay — A Paradigm Shift in Power System
Protection* (vPAC Alliance / Intel / Kalkitech), Figures 2–8. All diagrams below
are original Mermaid representations and render natively on GitHub.

---

## 1. Substation context (IEC 61850) — where the host sits

The reference model (whitepaper Fig 2) splits a digital substation into three
levels joined by a **process bus** (raw current/voltage as Sampled Values, plus
GOOSE) and a **station bus** (MMS/GOOSE to SCADA and operators). The Red Hat VPR
host replaces a rack of discrete bay-level IEDs with protection VMs.

```mermaid
flowchart LR
    subgraph FIELD["Process level (field)"]
        direction TB
        CTPT["CT / PT<br/>current + voltage sensors"]
        MU["Merging Unit (MU)"]
        BCU["Breaker Control Unit (BCU)"]
        CTPT --> MU
    end

    subgraph BAY["Bay level — virtualized"]
        direction TB
        HOST["Red Hat VPR host<br/>protection VMs replace discrete IEDs"]
    end

    subgraph STATION["Station level"]
        direction TB
        SCADA["SCADA"]
        HMI["HMI / operator"]
        ENG["Engineering + DR analysis"]
    end

    MU -- "Sampled Values (61850-9-2)" --> PB(["Process bus<br/>HSR/PRP, PTP-synced"])
    BCU -- "GOOSE" --> PB
    PB --> HOST
    HOST --> SB(["Station bus<br/>MMS / GOOSE / HTTPS"])
    SB --> SCADA
    SB --> HMI
    SB --> ENG
    HOST -. "trip (GOOSE)" .-> BCU

    classDef rh fill:#ee0000,stroke:#a30000,color:#fff;
    class HOST rh;
```

> Red in every diagram = the part Red Hat provides. Field devices (MU/BCU,
> CT/PT) and the SCADA/operator front end are unchanged customer/vendor kit.

---

## 2. The Red Hat VPR host — end-to-end signal path

This is the whitepaper's master reference application (Fig 8) with the Red Hat
platform plugged in. Sampled Values arrive from the process bus, are processed by
the vendor protection stack inside a **Virtual IED** VM, and a trip is published
back as GOOSE — all on a real-time-tuned RHEL/KVM host.

```mermaid
flowchart TB
    subgraph INGRESS["From the process bus"]
        SV["SV streams (61850-9-2)"]
        GIN["GOOSE in"]
    end

    subgraph HOST["Red Hat VPR host (RHEL + KVM)"]
        direction TB

        subgraph NICV["NIC virtualization"]
            SRIOV["SR-IOV VFs / macvtap (VEPA)<br/>process bus → VM, low latency"]
        end

        subgraph VIED["Virtual IED — vendor protection VM"]
            direction TB
            SVSUB["SV subscriber"]
            COND["Signal conditioning / phasor estimation"]
            PF["Protection functions<br/>21 · 50 · 51 · 50N · 51N · 87T · 50BF"]
            MMS["MMS server + GOOSE pub/sub"]
            SVSUB --> COND --> PF --> MMS
        end

        SRIOV --> SVSUB
        PTP["linuxptp (ptp4l / phc2sys)<br/>dedicated PTP NIC"] -. "PHC time" .-> VIED
    end

    subgraph EGRESS["To the station bus"]
        GOUT["GOOSE trip → breaker"]
        REPORT["MMS reports → SCADA / HMI"]
    end

    SV --> SRIOV
    GIN --> SRIOV
    MMS --> GOUT
    MMS --> REPORT

    classDef rh fill:#ee0000,stroke:#a30000,color:#fff;
    classDef vendor fill:#7b3fbf,stroke:#4a2575,color:#fff;
    class SRIOV,PTP,HOST rh;
    class VIED,SVSUB,COND,PF,MMS vendor;
```

---

## 3. The Red Hat platform stack (real-time enablement)

The whitepaper's real-time enablement (Fig 5, Intel TCC) and server/NIC layers
(Fig 6) map directly onto the Red Hat real-time stack. Bottom to top:

```mermaid
flowchart TB
    HW["Hardware — IEC 61850-3 / IEEE 1613 server<br/>NIC with HSR/PRP + PTP + SR-IOV"]
    OS["RHEL 9 (image-mode / bootc)<br/>kernel-rt · isolcpus / nohz_full / rcu_nocbs"]
    TUNE["Real-time tuning<br/>tuned realtime-virtual-host · 1 GiB hugepages<br/>resctrl / Cache Allocation (LLC isolation) · perf governor"]
    TIME["Time sync — linuxptp on a dedicated NIC<br/>NTP sources removed when PTP is authoritative"]
    VIRT["Virtualization — KVM + libvirt (modular sockets)<br/>CPU pinning · locked hugepage memory · no memballoon / watchdog"]
    HA["High availability (3-node) — Pacemaker + corosync<br/>STONITH fencing · Ceph (CephFS / RBD) · sanlock leases"]
    APP["Virtual IED VMs — vendor protection stacks<br/>ABB SSC600 (virtio) · NovaTech Orion (SATA + e1000)"]

    HW --> OS --> TUNE --> TIME --> VIRT --> HA --> APP

    classDef rh fill:#ee0000,stroke:#a30000,color:#fff;
    classDef vendor fill:#7b3fbf,stroke:#4a2575,color:#fff;
    class OS,TUNE,TIME,VIRT,HA rh;
    class APP vendor;
    style HW fill:#444,stroke:#000,color:#fff;
```

---

## 4. High availability — single node vs 3-node cluster

The whitepaper notes a VPR gains redundancy from the "host server or VM
management console" handling failover (Fig 7). Red Hat implements that with
**Pacemaker/corosync + STONITH** for orchestration and **Ceph** for the shared
disk a VM needs to restart on another node. Single-node skips all of it.

```mermaid
flowchart TB
    subgraph SINGLE["Single node (image-mode)"]
        direction TB
        S1["RHEL image-mode host<br/>local file-backed relay disk<br/>no Pacemaker / Ceph / STONITH"]
    end

    subgraph CLUSTER["3-node cluster"]
        direction TB
        PCS["Pacemaker + corosync (dedicated heartbeat net)<br/>VirtualDomain resources · STONITH fencing"]
        CEPH["Ceph — CephFS / RBD shared storage<br/>relay disks + sanlock leases"]
        N1["Node A"]
        N2["Node B"]
        N3["Node C"]
        PCS --- N1 & N2 & N3
        CEPH --- N1 & N2 & N3
        N1 -. "VM fails over" .-> N2
    end

    classDef rh fill:#ee0000,stroke:#a30000,color:#fff;
    class S1,PCS,CEPH rh;
```

---

## 5. Reference model → Red Hat implementation

| Whitepaper concept | Fig | Red Hat implementation |
|---|---|---|
| Virtualized host / hypervisor | 3–7 | **RHEL + KVM + libvirt** (modular sockets); image-mode (bootc) for single-node |
| Software container option | 2.3 | **Podman** (per-function microservices where used) |
| Real-time board support package | 5 | **kernel-rt**, `isolcpus`/`nohz_full`/`rcu_nocbs`, 1 GiB hugepages |
| Real-time tuning / TCC tools | 5 | **tuned `realtime-virtual-host`**, performance governor |
| Cache Allocation (LLC isolation) | 2.4.1 | **resctrl** mount + CAT classes pinned to the relay's cores |
| Deterministic ~5 ms system timescale | 2.4.2 | pinned vCPUs + `SCHED_FIFO` (SSC600 prio 50, Orion 40), locked memory |
| Precision Time Protocol | 2.4.3 | **linuxptp** (`ptp4l`/`phc2sys`) on a **dedicated** NIC; NTP removed |
| Next-gen NIC (HSR/PRP, PTP, SR-IOV) | 2.4.4 / 6 | **SR-IOV VFs / macvtap (VEPA)** to the process bus; PTP-capable NIC |
| Process bus (SV + GOOSE) | 2 / 8 | dedicated **process-bus** interface, reserved, macvtap-attached |
| Station bus (MMS / GOOSE) | 2 / 8 | **station-bus** Linux bridge; mgmt bridge for the host |
| VPR redundancy / HA cluster | 7 | **Pacemaker + corosync + STONITH** on a dedicated heartbeat network |
| Shared storage for VM failover | 7 | **Ceph** (CephFS / RBD) + **sanlock** leases (3-node only) |
| Virtual IED (SV sub, conditioning, PF, MMS) | 3/4/8 | the **vendor relay VM** (ABB SSC600 / NovaTech Orion), hosted unchanged |
| Front end (SCADA, HMI, config, DR) | 8 | customer/vendor side, unchanged — reached over the station bus |

---

*Field devices (MU/BCU, CT/PT) and the SCADA/operator front end are out of Red
Hat's scope and shown only for context. Everything marked red is delivered by the
Red Hat platform.*
