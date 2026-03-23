---
title: "OCI"
description: "Oracle Cloud Infrastructure — Oracle's cloud platform, offering significant licensing advantages for Oracle databases through the BYOL program."
translationKey: "glossary_oci"
aka: "Oracle Cloud Infrastructure"
articles:
  - "/posts/oracle/oracle-cloud-migration"
---

**OCI** (Oracle Cloud Infrastructure) is Oracle's cloud platform, launched in its second generation in 2018. Unlike other cloud providers, OCI is natively designed for Oracle Database workloads and offers significant licensing and performance advantages.

## Why OCI for Oracle Database

The main advantage is licensing. On OCI, Oracle recognizes its own OCPUs (Oracle CPUs) with a 1:1 ratio for license counting purposes. On other cloud providers like AWS or Azure, the vCPU-to-license ratio is less favorable and the audit risk is real.

The **BYOL** (Bring Your Own License) program allows reusing existing on-premises licenses on OCI at no additional cost — a decisive factor for organizations that have already invested in Enterprise Edition licenses.

## Key services for DBAs

- **Bare Metal DB Systems**: dedicated physical servers with pre-installed Oracle Database
- **VM DB Systems**: virtual instances with flexible configuration (Flex shapes)
- **Exadata Cloud Service**: fully managed Exadata in the cloud
- **Autonomous Database**: fully managed database with automatic tuning

## Networking and connectivity

OCI offers **FastConnect** for dedicated high-bandwidth connections between on-premises data centers and cloud regions, plus site-to-site VPN for scenarios with lower bandwidth requirements. Link latency and bandwidth are critical factors in cross-site Data Guard migrations.
