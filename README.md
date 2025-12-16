# Reestructuración Operacional: Migración de Infraestructura Crítica a Contenedores Ordenados
*Proyecto realizado por Andrés Larrahona*

## Objetivo del proyecto

Este proyecto detalla la estrategia utilizada para migrar servicios de infraestructura, automatización de datos (ETL) y *frontends* desde un entorno **Windows Server** centralizado y con despliegues descentralizados hacia una arquitectura **Docker Compose** dedicada.

El objetivo principal fue **mitigar el riesgo operativo, aislar los servicios y centralizar la orquestación**. Al mover los procesos de tareas programadas de Windows a **Apache Airflow**, logramos una plataforma de DataOps estable y con trazabilidad, liberando el servidor de Active Directory/Archivos de cargas ajenas a su función principal.


---

## El Desafío: El Contexto Legacy

La configuración anterior, aunque funcional, presentaba vulnerabilidades significativas para un entorno de producción o pre-producción:

* **Descentralización y Riesgo Compartido:** Teniendo servicios, automatizaciones y *frontends* críticos en una misma máquina virtual de forma descentralizada, corriendo terminales en simultáneo (XAMPP, MkDocs, VueJS en `npm run dev`), resultaba extremadamente **difícil tener un seguimiento adecuado de los servicios**. Esta consolidación generaba conflictos de recursos y dependencias, afectando no solo la estabilidad del servidor de Active Directory, sino también su **rendimiento y seguridad**.
* **Procesos ETL sin Trazabilidad:** Las automatizaciones clave (sincronización de bases de datos y envío de correos) **se ejecutaban mediante *scripts* (`.bat` y *cronjobs*) sin un *framework* de orquestación**. Esta falta de gestión impedía el *logging* centralizado, la definición de SLAs y la capacidad de reintento automático, esenciales para la operación de datos.
* **Despliegue Inmaduro:** Servir aplicaciones de *front-end* y documentación **desde *runtimes* de desarrollo** (terminales abiertas) o servidores *ad-hoc* (XAMPP) comprometía la eficiencia, el rendimiento y la gestión de versiones.


---

## La Solución planteada: Docker Compose y Orquestación

Se implementó una arquitectura basada en **Docker Compose** en un nuevo *host* dedicado, aplicando principios de **aislamiento estricto** y **gestión centralizada de flujos de trabajo**.

### 1. El Nuevo Stack Contenerizado

Se definió la composición de cuatro servicios clave que operan de forma aislada y gestionable:

| Servicio Contenerizado | Función | Valor Operacional (Tech Lead / Operations) |
| :--- | :--- | :--- |
| **Apache Airflow** | Plataforma de orquestación para todos los flujos de datos. | Centralización de ETL, monitoreo y reintentos (sustituye archivos *.bat* en Tareas de Windows y cronjobs de Linux). |
| **Apache** | Servidor web estático. | Sustitución de servidor web alojado con XAMPP a servidor apache dockerizado permite estandarizar y centralizar los servicios en un único panel de control y configuración. |
| **Nginx** | Servidor web estático. | La migración de un *runtime* de desarrollo (ejecución `npm run dev`) a un *build* de producción garantiza un despliegue estable con un consumo de recursos mínimo y optimizado para la entrega de contenido estático.|
| **Metabase** | Se mantiene misma funcionalidad que en el servidor original. | Servicio aislado y dedicado para el consumo de datos modelados. |
| **MkDocs** | Se mantiene misma funcionalidad que en el servidor original. | Consistencia del ambiente y facilidad de mantenimiento. |




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

## 3. Conclusiones

Este proyecto permitió pasar a un esquema basado en el orden y la visibilidad. Los resultados se resumen en tres ejes principales:

- **Aislamiento y estabilidad:** Al aislar los servicios de desarrollo y ETL en un host dedicado con Docker, se lograron tres cosas inmediatas: se liberó el servidor principal de Active Directory y archivos de conflictos de recursos, se eliminó el riesgo de que una aplicación experimental o un runtime fallido comprometiera los servicios esenciales del laboratorio y se eliminó, también, la supervisión manual de todos estos procesos.

- **Transparencia y control de las operaciones:** La migración de tres automatizaciones críticas (sincronización DB, notificaciones y carga de certificados) de scripts locales a Airflow eliminó la incertidumbre. Ahora los procesos tienen trazabilidad completa y la capacidad de gestionar fallos y reintentos de manera consistente. Si un pipeline se detiene, sabemos dónde, cuándo y por qué, mejorando significativamente el tiempo de respuesta.

- **Claridad Operativa y Accesibilidad:** Establecer una arquitectura basada en contenedores (Docker Compose) y servidores web estables (Nginx y Apache) significa que el entorno ahora es claro y autodescriptivo. La configuración está centralizada en un docker-compose.yml y los flujos en la interfaz gráfica de Airflow. Esto reduce drásticamente la dependencia de un conocimiento histórico y facilita la integración de futuros colaboradores al equipo, permitiéndoles entender y mantener la plataforma sin ser expertos en el stack legacy original.
