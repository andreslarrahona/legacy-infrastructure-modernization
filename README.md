<details>
  <summary style="cursor:pointer"><strong>Versión en español:</strong>
  </summary>

# Reestructuración operacional: Migración de Infraestructura Crítica a Contenedores Ordenados
*Proyecto realizado por Andrés Larrahona*

## Objetivo del proyecto

Este proyecto detalla la estrategia utilizada para migrar servicios de infraestructura, automatización de datos (ETL) y *frontends* desde un entorno **Windows Server** centralizado y con despliegues descentralizados hacia una arquitectura **Docker Compose** dedicada.

El objetivo principal fue **reducir el riesgo operativo, aislar los servicios y centralizar la orquestación**. Al mover los procesos de tareas programadas de Windows y cronjobs de Linux a **Apache Airflow**, logramos una plataforma de DataOps estable y con trazabilidad, liberando el servidor de Active Directory/Archivos de cargas ajenas a su función principal.


---

## El Desafío: El Contexto Legacy

La arquitectura sobre la cual estaba configurada toda la infraestructura, aunque funcional, presentaba vulnerabilidades significativas para un entorno de producción o pre-producción:

* **Descentralización y riesgos:** Teniendo servicios, automatizaciones y *frontends* críticos en una misma máquina virtual de forma descentralizada, corriendo terminales en simultáneo (XAMPP, MkDocs, VueJS en *runtime* de desarrollo `npm run dev`), entre los riesgos que se pueden mencionar, resultaba extremadamente **difícil tener un seguimiento adecuado de los servicios**. Esta arquitectura generaba conflictos de recursos y dependencias, afectando no solo la estabilidad del servidor de Active Directory, sino también su **rendimiento y seguridad**.
<br/>
* **Procesos ETL sin trazabilidad:** Las automatizaciones clave (sincronización de bases de datos y envío de correos) **se ejecutaban mediante *scripts* (`.bat` y *cronjobs*) sin un *framework* de orquestación**. Esta falta de gestión impedía el *logging* centralizado, la definición de SLAs y la capacidad de reintento automático, esenciales para la operación de datos.


### Infraestructura original
![Legacy structure](assets/legacystructure.png)




---

## La Solución planteada: Docker Compose y Orquestación

Se implementó una arquitectura basada en **Docker Compose** en un nuevo *host* dedicado, aplicando principios de **aislamiento estricto** y **gestión centralizada de flujos de trabajo**.

### 1. El Nuevo Stack Contenerizado

Se definió la composición de cinco servicios clave que operan de forma aislada y gestionable:

| Servicio Contenerizado | Función | Valor Operacional (Tech Lead / Operations) |
| :--- | :--- | :--- |
| **Apache Airflow** | Plataforma de orquestación para todos los flujos de datos. | Centralización de ETL, monitoreo y reintentos (sustituye archivos *.bat* en Tareas de Windows y cronjobs de Linux). |
| **Apache** | Servidor web estático. | Sustitución de servidor web alojado con XAMPP a servidor apache dockerizado permite estandarizar y centralizar los servicios en un único panel de control y configuración. |
| **Nginx** | Servidor web estático. | La migración de un *runtime* de desarrollo (ejecución `npm run dev`) a un *build* de producción garantiza un despliegue estable con un consumo de recursos mínimo y optimizado para la entrega de contenido estático.|
| **Metabase** | Se mantiene misma funcionalidad que en el servidor original. | Servicio aislado y dedicado para el consumo de datos modelados. |
| **MkDocs** | Se mantiene misma funcionalidad que en el servidor original. | Consistencia del ambiente y facilidad de mantenimiento. |

### Propuesta de arquitectura
![Stack proposal](assets/proposal.png)


<details>
<summary style="cursor:pointer"><strong>docker-compose.yml (simplificado):</strong>
</summary>

``` text
version: '3.8'

x-airflow-common: &airflow-common
    image: ${AIRFLOW_IMAGE_NAME:-apache/airflow:2.7.1}
    env_file: [ .env ]
    volumes:
        - ${SERVICES_PATH}/airflow/dags:/opt/airflow/dags
        - ${SERVICES_PATH}/airflow/logs:/opt/airflow/logs
    user: "50000:0"
    depends_on: [ postgres ]
services:
    postgres:
        image: postgres:16
        environment:
            POSTGRES_USER: ${POSTGRES_USER}
            POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
            POSTGRES_DB: ${POSTGRES_DB}
        volumes:
            - postgres-db-volume:/var/lib/postgresql/data
        restart: always
    mkdocs:
        build: ${SERVICES_PATH}/mkdocs
        container_name: centec_mkdocs
        ports:
            - "${PORT_MKDOCS}:8000"
        volumes:
            - ${VOLUMES_PATH}/mkdocs-data:/docs
        restart: always
    kanban:
        build: ${SERVICES_PATH}/kanban
        container_name: centec_kanban
        ports:
            - ${PORT_KANBAN}:80
        restart: always
    metabase:
        image: metabase/metabase
        container_name: centec_metabase
        ports:
            - ${PORT_METABASE}:3000
        volumes:
            - ${VOLUMES_PATH}/metabase-data:/metabase-data
        environment:
            - MB_DB_FILE=/metabase-data/metabase.db
        restart: always
    apache:
        build: ./services/apache
        container_name: centec_apache
        ports:
            - "${PORT_APACHE}:80" 
        volumes:
            - ${VOLUMES_PATH}/apache_data:/var/www/html
        restart: always
    airflow_web:
        <<: *airflow-common
        container_name: centec_airflow_web
        ports: ["${PORT_AIRFLOW}:8080"]
        command: webserver
    airflow_scheduler:
        <<: *airflow-common
        container_name: centec_airflow_scheduler
        command: scheduler
    ...

```
</details>
<br>


### 2. Énfasis en DataOps: Orquestación Airflow

La tarea fundamental de esta migración fue consolidar los procesos de automatización que estaban dispersos y sin control, pasándolos a Apache Airflow. Antes, estos procesos dependían de la programación a nivel del sistema operativo, lo cual **limitaba la visibilidad y la gestión de fallos**.

Se buscó centralizar tres procesos críticos en Airflow, transformándolos en DAGs (Directed Acyclic Graphs) que se puedan monitorear, auditar y gestionar:

- **Sincronización de Base de Datos**: Este DAG absorbió la Tarea Programada de Windows que se encargaba de asegurar la réplica de la base de datos MySQL local hacia una base de datos gestionada en la nube. El valor agregado es la posibilidad de gestionar los reintentos en caso de fallas.

- **Envío de Notificaciones Automáticas**: El proceso de notificación a clientes, que se ejecutaba como otra Tarea Programada de Windows, fue migrado para ser gestionado y disparado por Airflow.

- **Carga de Certificados (pdf)  a Data Lake**: Una tarea ejecutada como Cronjob de Ubuntu, que se encargaba de subir archivos PDF de certificados a un Data Lake para su posterior consumo, fue reescrito e integrado como un DAG en la nueva plataforma.

<br/>

![Airflow DAGS screenshot](assets/airflow.png)

### Servicio adicional
Como se puede observar en la captura de pantalla de la UI de Airflow, se agregó un dag adicional que sirve como **monitoreo del estado de cada uno de los servicios** propuestos en la presente arquitectura (*monitor_lab*). De esta forma, el estado de todos los servicios es reportado a la base de datos MySQL local para monitoreo constante, plasmado en Metabase.


## 3. Conclusiones

Este proyecto permitió pasar a un esquema basado en el orden y la visibilidad. Los resultados se resumen en tres ejes principales:

- **Aislamiento y estabilidad:** Al aislar los servicios de desarrollo y ETL en un host dedicado con Docker, se lograron tres cosas inmediatas: se liberó el servidor principal de Active Directory y archivos de conflictos de recursos, se eliminó el riesgo de que una aplicación experimental o un runtime fallido comprometiera los servicios esenciales del laboratorio y se eliminó, también, la supervisión manual de todos estos procesos.

- **Transparencia y control de las operaciones:** La migración de tres automatizaciones críticas (sincronización DB, notificaciones y carga de certificados) de scripts locales a Airflow eliminó la incertidumbre. Ahora los procesos tienen trazabilidad completa y la capacidad de gestionar fallos y reintentos de manera consistente. Si un pipeline se detiene, sabemos dónde, cuándo y por qué, mejorando significativamente el tiempo de respuesta.

- **Claridad Operativa y Accesibilidad:** Establecer una arquitectura basada en contenedores (Docker Compose) y servidores web estables (Nginx y Apache) significa que el entorno ahora es claro y autodescriptivo. La configuración está centralizada en un docker-compose.yml y los flujos en la interfaz gráfica de Airflow. Esto reduce drásticamente la dependencia de un conocimiento histórico y facilita la integración de futuros colaboradores al equipo, permitiéndoles entender y mantener la plataforma sin ser expertos en el stack legacy original.

</details>

 <br>


<details open>
<summary style="cursor:pointer"><strong>English version:</strong>
</summary>

# Operational Restructuring: Migrating Critical Infrastructure to Organized Containers
*by Andrés Larrahona*

## Project Objective

This project details the strategy used to migrate infrastructure services, data automation (ETL), and frontends from a centralized, ad-hoc **Windows Server** environment to a dedicated **Docker Compose** architecture. 

The main goal was to **reduce operational risk, isolate services, and centralize orchestration**. By moving legacy Windows Scheduled Tasks and Linux cronjobs to **Apache Airflow**, we established a stable DataOps platform with full traceability, successfully offloading non-core tasks from the Active Directory/File Server.

---

## The Challenge: Legacy Context

The original architecture, while functional, presented significant vulnerabilities for a production or pre-production environment:

* **Decentralization and Risks:** Running critical services, automations, and frontends on a single virtual machine (including XAMPP, MkDocs, and VueJS in *runtime* development mode `npm run dev`) made **service monitoring extremely difficult**. This setup led to resource and dependency conflicts, impacting not only the stability of the Active Directory server but also its **performance and security**.
<br/>
* **Lack of ETL Traceability:** Key automations (database synchronization and automated emailing) **ran via `.bat` scripts and cronjobs without an orchestration framework**. This lack of management prevented centralized logging, SLA definition, and automated retries—all essential for reliable data operations.

### Original Infrastructure
![Legacy structure](assets/legacystructure.png)

---

## Proposed Solution: Docker Compose and Orchestration

A new architecture was implemented on a dedicated host using **Docker Compose**, applying principles of **strict isolation** and **centralized workflow management**.

### 1. The New Containerized Stack

Five key services were defined to operate in an isolated and manageable way:

| Containerized Service | Function | Operational Value (Tech Lead / Operations) |
| :--- | :--- | :--- |
| **Apache Airflow** | Data workflow orchestration platform. | Centralizes ETL, monitoring, and retries (replaces Windows .bat files and Linux cronjobs). |
| **Apache** | Static web server. | Moving from XAMPP to a dockerized Apache server standardizes services under a single configuration panel. |
| **Nginx** | Static web server. | Migrating from a development runtime (`npm run dev`) to a production build ensures a stable deployment with minimal resource consumption. |
| **Metabase** | Retains original functionality. | Dedicated, isolated service for consuming modeled data. |
| **MkDocs** | Retains original functionality. | Environment consistency and ease of maintenance. |

### Proposed Architecture
![Stack proposal](assets/proposal.png)

<details>
<summary style="cursor:pointer"><strong>docker-compose.yml (simplified):</strong></summary>

    ```yaml
        version: '3.8'

        x-airflow-common: &airflow-common
            image: ${AIRFLOW_IMAGE_NAME:-apache/airflow:2.7.1}
            env_file: [ .env ]
            volumes:
                - ${SERVICES_PATH}/airflow/dags:/opt/airflow/dags
                - ${SERVICES_PATH}/airflow/logs:/opt/airflow/logs
            user: "50000:0"
            depends_on: [ postgres ]

        services:
            postgres:
                image: postgres:16
                environment:
                    POSTGRES_USER: ${POSTGRES_USER}
                    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
                    POSTGRES_DB: ${POSTGRES_DB}
                volumes:
                    - postgres-db-volume:/var/lib/postgresql/data
                restart: always

            mkdocs:
                build: ${SERVICES_PATH}/mkdocs
                container_name: centec_mkdocs
                ports:
                    - "${PORT_MKDOCS}:8000"
                volumes:
                    - ${VOLUMES_PATH}/mkdocs-data:/docs
                restart: always

            kanban:
                build: ${SERVICES_PATH}/kanban
                container_name: centec_kanban
                ports:
                    - ${PORT_KANBAN}:80
                restart: always

            metabase:
                image: metabase/metabase
                container_name: centec_metabase
                ports:
                    - ${PORT_METABASE}:3000
                volumes:
                    - ${VOLUMES_PATH}/metabase-data:/metabase-data
                environment:
                    - MB_DB_FILE=/metabase-data/metabase.db
                restart: always

            apache:
                build: ./services/apache
                container_name: centec_apache
                ports:
                    - "${PORT_APACHE}:80" 
                volumes:
                    - ${VOLUMES_PATH}/apache_data:/var/www/html
                restart: always

            airflow_web:
                <<: *airflow-common
                container_name: centec_airflow_web
                ports: ["${PORT_AIRFLOW}:8080"]
                command: webserver

            airflow_scheduler:
                <<: *airflow-common
                container_name: centec_airflow_scheduler
                command: scheduler
    ```
</details>


### 2. DataOps Focus: Airflow Orchestration
The core of this migration was consolidating the scattered automation processes into Apache Airflow. Previously, these processes relied on OS-level scheduling, which limited visibility and fault management.

Three critical processes were centralized into Airflow as DAGs (Directed Acyclic Graphs) for better monitoring and auditing:

- **Database Synchronization:** This DAG took over a Windows Scheduled Task responsible for replicating the local MySQL database to a cloud-managed database. The added value here is the built-in management of retries.

- **Automated Notifications:** The client notification process, formerly another Windows task, was migrated to be triggered and managed by Airflow.

- **Certificate Uploads (PDF) to Data Lake:** A task previously running as an Ubuntu cronjob—handling PDF certificate uploads for portal consumption—was rewritten and integrated as a DAG.

<br/>

![Airflow DAGS screenshot](assets/airflow.png)

### Additional Service
As seen in the Airflow UI, an extra DAG was added to **monitor the health of each service** in this architecture (monitor_lab). Service status is reported to the local MySQL database for constant monitoring and visualized through Metabase.

### 3. Conclusions
This project successfully transitioned the infrastructure toward a model based on order and visibility. The results center on three main pillars:

- **Isolation and Stability:** Moving development and ETL services to a dedicated Docker host provided immediate relief: the Active Directory server was freed from resource conflicts, the risk of a failing experimental runtime compromising essential services was eliminated, and the need for manual oversight was removed.

- **Operational Transparency:** Migrating three critical automations from local scripts to Airflow eliminated uncertainty. Processes now have full traceability and consistent retry logic. If a pipeline stops, we know exactly where, when, and why, significantly improving response times.

- **Operational Clarity and Accessibility:** Establishing an architecture based on containers and stable web servers (Nginx and Apache) means the environment is now self-describing. Configuration is centralized in a docker-compose.yml file and workflows are visible in Airflow. This reduces reliance on "tribal knowledge" and makes it much easier for new team members to understand and maintain the platform without needing to be experts in the original legacy stack.


</details>
