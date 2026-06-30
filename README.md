# Modernización del Proyecto Java

Este proyecto se encuentra en un proceso de migración de **Java 8 a Java 25**. Este repositorio utiliza **Cline** como agente de IA para automatizar la refactorización, actualización de dependencias y validación de código.

## 🚀 Cómo empezar con Cline en VS Code

Para trabajar en la modernización de este proyecto, utilizamos la extensión **Cline**. Sigue estos pasos para configurarlo:

1. **Instalación:**
   - Abre Visual Studio Code.
   - Ve a la sección de **Extensiones** (`Ctrl+Shift+X`).
   - Busca e instala **Cline**.

2. **Configuración:**
   - Haz clic en el icono de Cline en la barra lateral.
   - Configura tu **API Key** (recomendamos usar *Claude 3.5 Sonnet* para mejores resultados en refactorización de Java).
   - En caso de no tener se puede crear una cuenta gratuita con acceso limitado, la cual usaremos para esta demostracion

3. **Uso en el Proyecto:**
   - Abre el proyecto en VS Code.
   - Utilizamos este promt para inicializar el agente:
   Actúa como un arquitecto de software experto en migraciones de sistemas Java. Necesito que realices un análisis completo de este repositorio para planificar una migración desde Java 8 hacia Java 25.
Por favor, sigue estos pasos:
Auditoría de Dependencias: Escanea los archivos de construcción (pom.xml o build.gradle) e identifica dependencias que puedan ser incompatibles con versiones recientes de Java o Spring Boot 3.x+. Haz una lista de librerías críticas que deban ser actualizadas o reemplazadas.
Detección de Riesgos: Identifica el uso de APIs de Java eliminadas, acceso a APIs internas (sun.misc.*), o dependencias obsoletas de Jakarta EE/Java EE que causarán errores de compilación.
Plan de Acción: Crea un documento llamado MIGRATION_PLAN.md en la raíz del proyecto. Este plan debe estar dividido en fases iterativas (Java 8 -> 11 -> 17 -> 21 -> 25) y proponer tareas concretas para cada etapa.
Baseline: Identifica cómo ejecutar actualmente la suite de pruebas unitarias y de integración para asegurar que tengamos una línea base antes de empezar cualquier cambio.
Empieza realizando el escaneo inicial y listando las dependencias que requieren atención inmediata.
   - En el chat de Cline, puedes empezar usando los prompts de migración definidos en nuestro `MIGRATION_PLAN.md`.

## 📋 Plan de Migración
Puedes consultar nuestra hoja de ruta detallada en el archivo [MIGRATION_PLAN.md](./MIGRATION_PLAN.md). Este archivo contiene las fases, riesgos identificados y las tareas pendientes para completar la actualización a Java 25.


