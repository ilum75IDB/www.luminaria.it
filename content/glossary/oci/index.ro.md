---
title: "OCI"
description: "Oracle Cloud Infrastructure — platforma cloud Oracle, cu avantaje semnificative de licențiere pentru bazele de date Oracle prin programul BYOL."
translationKey: "glossary_oci"
aka: "Oracle Cloud Infrastructure"
articles:
  - "/posts/oracle/oracle-cloud-migration"
---

**OCI** (Oracle Cloud Infrastructure) este platforma cloud a Oracle, lansata in a doua generatie in 2018. Spre deosebire de alti furnizori cloud, OCI este proiectata nativ pentru workload-uri Oracle Database si ofera avantaje semnificative in ceea ce priveste licentierea si performanta.

## De ce OCI pentru Oracle Database

Avantajul principal tine de licentiere. Pe OCI, Oracle recunoaste propriile OCPU (Oracle CPU) cu un raport 1:1 pentru calculul licentelor. Pe alti furnizori cloud precum AWS sau Azure, raportul vCPU-licente este mai putin favorabil si riscul de audit este real.

Programul **BYOL** (Bring Your Own License) permite reutilizarea licentelor on-premises existente pe OCI fara costuri suplimentare — un factor decisiv pentru organizatiile care au investit deja in licente Enterprise Edition.

## Servicii principale pentru DBA

- **Bare Metal DB Systems**: servere fizice dedicate cu Oracle Database preinstalat
- **VM DB Systems**: instante virtuale cu configuratie flexibila (Flex shapes)
- **Exadata Cloud Service**: Exadata complet gestionat in cloud
- **Autonomous Database**: baza de date complet gestionata cu tuning automat

## Networking si conectivitate

OCI ofera **FastConnect** pentru conexiuni dedicate de banda larga intre centrele de date on-premises si regiunile cloud, plus VPN site-to-site pentru scenarii cu cerinte de banda mai mici. Latenta si bandwidth-ul conexiunii sunt factori critici in migrarile cu Data Guard cross-site.
