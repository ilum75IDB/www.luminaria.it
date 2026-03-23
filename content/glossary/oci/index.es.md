---
title: "OCI"
description: "Oracle Cloud Infrastructure — la plataforma cloud de Oracle, con ventajas significativas de licensing para bases de datos Oracle gracias al programa BYOL."
translationKey: "glossary_oci"
aka: "Oracle Cloud Infrastructure"
articles:
  - "/posts/oracle/oracle-cloud-migration"
---

**OCI** (Oracle Cloud Infrastructure) es la plataforma cloud de Oracle, lanzada en su segunda generacion en 2018. A diferencia de otros proveedores cloud, OCI esta disenada nativamente para cargas de trabajo Oracle Database y ofrece ventajas significativas en licensing y rendimiento.

## Por que OCI para Oracle Database

La ventaja principal es el licensing. En OCI, Oracle reconoce sus propias OCPU (Oracle CPU) con una relacion 1:1 para el conteo de licencias. En otros proveedores cloud como AWS o Azure, la relacion vCPU-licencias es menos favorable y el riesgo de auditoria es real.

El programa **BYOL** (Bring Your Own License) permite reutilizar las licencias on-premises existentes en OCI sin costes adicionales — un factor decisivo para las empresas que ya han invertido en licencias Enterprise Edition.

## Servicios principales para DBAs

- **Bare Metal DB Systems**: servidores fisicos dedicados con Oracle Database preinstalado
- **VM DB Systems**: instancias virtuales con configuracion flexible (Flex shapes)
- **Exadata Cloud Service**: Exadata completo gestionado en cloud
- **Autonomous Database**: base de datos completamente gestionada con tuning automatico

## Networking y conectividad

OCI ofrece **FastConnect** para conexiones dedicadas de alta banda entre centros de datos on-premises y la region cloud, ademas de VPN site-to-site para escenarios con requisitos de banda menores. La latencia y el ancho de banda del enlace son factores criticos en las migraciones con Data Guard cross-site.
