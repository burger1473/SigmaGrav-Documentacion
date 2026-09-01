# Documentación de Protección Binaria, Licenciamiento y Optimización de Distribuibles

Bienvenido a la documentación oficial del sistema de protección, licenciamiento y empaquetado binario de **SigmaGrav**.

Esta documentación ha sido estructurada tanto para lectura humana (desarrolladores, arquitectos de software) como para procesamiento por modelos de Inteligencia Artificial (asistentes de código, agentes autónomos).

---

## Índice de Documentos

1. [**01. Arquitectura y Flujo Actual**](file:///home/burger/workspace/repos/SigmaGrav/SigmaGrav-Documentacion/EncriptacionBinaria/01_ARQUITECTURA_Y_FLUJO_ACTUAL.md)
   * Descripción completa de la arquitectura actual (`FrankPhpEmbed`, `encriptadorBinario`, `loader.cpp`, `EjecutadorDockerSinBinario`).
   * Diagrama de flujo Mermaid de la compilación y ejecución fileless en RAM (`memfd_create` + `fexecve`).
   * Estructura de datos de licencias (`LicenseData`) y parámetros de cifrado AES-256-CBC.

2. [**02. Diagnóstico de Problemas y Bottlenecks**](file:///home/burger/workspace/repos/SigmaGrav/SigmaGrav-Documentacion/EncriptacionBinaria/02_DIAGNOSTICO_DE_PROBLEMAS_Y_BOTTLENECKS.md)
   * Análisis de la causa raíz de los tiempos de build elevados (15+ minutos).
   * Análisis del peso de las actualizaciones (80 MB+ por actualización de cliente).
   * Evaluación del impacto de rendimiento de `eval(gzinflate(...))` y vulnerabilidades frente a dumps de memoria RAM.

3. [**03. Alternativas Gratuitas y Libres**](file:///home/burger/workspace/repos/SigmaGrav/SigmaGrav-Documentacion/EncriptacionBinaria/03_ALTERNATIVAS_GRATUITAS_Y_LIBRES.md)
   * **Arquitectura Desacoplada (Recomendada):** Separación de FrankenPHP Engine del paquete de aplicación cifrada incremental.
   * **Herramientas Libres de PHP:** Extensiones C (`PHP-Beast`, `PHP-Bolt`) y ofuscadores AST (`YAKPRO-PO`).
   * **Protección Frontend:** Mangle avanzado con `javascript-obfuscator` y módulos WebAssembly (WASM).
   * Tabla comparativa de soluciones por costo, velocidad de build y seguridad.

4. [**04. Estrategia Recomendada y Plan de Implementación**](file:///home/burger/workspace/repos/SigmaGrav/SigmaGrav-Documentacion/EncriptacionBinaria/04_ESTRATEGIA_RECOMENDADA_Y_PLAN_IMPLEMENTACION.md)
   * Hoja de ruta por fases para implementar el desacoplamiento.
   * Plan para reducir builds de 15 minutos a 5 segundos y descargas de 80MB a 1.5MB.
   * Checklist de verificación para desarrolladores y agentes IA.
