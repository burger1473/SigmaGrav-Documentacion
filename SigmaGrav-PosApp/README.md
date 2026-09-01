# Documentación Técnica: SigmaGrav-PosApp

Esta documentación describe la arquitectura, comportamiento, modos de operación y flujos de resiliencia del sistema **SigmaGrav-PosApp** (Terminal POS en punto de venta/pista), su integración con el backend central **SigmaGrav-BackEnd** (Laravel) y con la infraestructura local/concentradores **SigmaGrav-Concentrador** (Python).

---

## 1. Visión General y Arquitectura de Red

El ecosistema **SigmaGrav** opera mediante una arquitectura híbrida y distribuida diseñada para garantizar alta disponibilidad operativa durante el despacho de combustible y la fidelización de clientes en estaciones de servicio.

```mermaid
graph TD
    subgraph Cloud / Estación Central
        BE[SigmaGrav-BackEnd<br/>Laravel API]
        DB[(Base de Datos Central)]
        BE --- DB
    end

    subgraph Estación Local / Pista
        POS[SigmaGrav-PosApp<br/>Terminal Android / React]
        CONC1[Concentrador Local 1<br/>Python Server / Surtidores]
        CONC2[Concentrador Local 2<br/>Python Server / Surtidores]
        
        POS -- 1. HTTP Nominal (Online) --> BE
        POS -- 2. Polling Estado (Status) --> BE
        POS -- 3. Rescate Directo (Offline Fallback) --> CONC1
        POS -- 3. Rescate Directo (Offline Fallback) --> CONC2
    end
```

### Componentes Clave:
* **SigmaGrav-PosApp**: Aplicación web embebida en Android (Capacitor/React) para terminales de pago móviles (ej. Pax A910). Ejecuta la lógica de identificación de clientes, selección de manguera, cobro y fidelización.
* **SigmaGrav-BackEnd (Estación Central)**: Servidor de negocio central (Laravel Octane en puerto 8000/API). Administra usuarios, tarjetas, reglas de fidelización, sincronización de despacho y registro de transacciones globales.
* **SigmaGrav-Concentrador (Estación Local)**: Servidor local de pista (Python API en `/api/v1/`) directamente conectado a la placa de surtidores/aforadores. Mantiene las cargas en memoria local para consulta inmediata y contingencia.

---

## 2. Modos de Operación (`PosAppMode`)

`SigmaGrav-PosApp` ajusta su interfaz y flujo transaccional dinámicamente según la configuración asignada al perfil del POS en el backend (`pos_modo`).

| ID Modo | Nombre | Descripción del Comportamiento |
| :--- | :--- | :--- |
| **1** | `FIDELIZACION` | **Modo Fidelización Pura**: Identifica cliente (vía tarjeta QR/RFID o DNI) y registra/canjea puntos. No gestiona el pago de la carga ni selecciona mangueras. |
| **2** | `FIDELIZACION_COBRO_MANUAL` | **Fidelización + Cobro Manual**: El operador escanea al cliente, consulta las cargas disponibles por manguera y efectúa el cobro en el POS (Efectivo/Tarjeta/QR). |
| **3** | `FIDELIZACION_COBRO_MANUAL_FACTURACION` | **Cobro Manual con Consulta de Factura**: Similar al Modo 2, pero requiere paso previo de consulta/verificación de comprobantes o facturación existente. |
| **4** | `FIDELIZACION_COBRO_PDV` | **Fidelización + Cobro PDV**: Cobro integrado sincronizado desde un Punto de Venta (PDV) centralizado. |
| **5** | `FIDELIZACION_COBRO_PDV_FACTURACION` | **Cobro PDV con Facturación**: Integración PDV + Facturación activa. |

```mermaid
flowchart TD
    A[Inicio / Login POS] --> B{Validar pos_modo}
    B -->|Modo 1| C[Fidelización Pura]
    B -->|Modo 2| D[Fidelización + Cobro Manual]
    B -->|Modo 3| E[Fidelización + Cobro Manual + Facturación]
    B -->|Modo 4| F[Fidelización + Cobro PDV]
    B -->|Modo 5| G[Fidelización + Cobro PDV + Facturación]

    C --> C1[Escanear Cliente -> Seleccionar Puntos -> Finalizar]
    D --> D1[Escanear Cliente -> Seleccionar Manguera/Carga -> Cobrar]
    E --> E1[Escanear Cliente -> Consultar Factura -> Seleccionar Carga -> Cobrar]
    F --> F1[Vincular a PDV -> Cobrar]
    G --> G1[Vincular a PDV -> Consultar Factura -> Cobrar]
```

---

## 3. Resiliencia: Modo Nominal vs Modo de Falla

`SigmaGrav-PosApp` incluye un mecanismo de tolerancia a fallos de red sin interrupción de ventas (*Zero-Downtime Fallback*).

```mermaid
sequenceDiagram
    autonumber
    participant POS as PosApp
    participant CB as ConnectionBlocker
    participant BE as Backend Central
    participant CONC as Concentrador Local

    rect rgb(235, 255, 235)
        note over POS, BE: MODO NOMINAL (ONLINE)
        POS->>BE: /posapp/login (Obtiene token JWT + Datos e inmutabiliza en local)
        POS->>BE: /posapp/obtenerCargas (Consulta cargas centrales)
        BE-->>POS: Cargas, Posnets y Precios
    end

    rect rgb(255, 235, 235)
        note over POS, CONC: TRANSICIÓN A MODO DE FALLA (OFFLINE)
        CB->>BE: Heartbeat /public/system-status (Falla de Red / Timeout)
        BE--xCB: Sin respuesta / ECONNABORTED
        CB-->>POS: Evento 'server-status-change' (online: false)
        POS->>POS: loginOffline() (Activa Usuario OPERADOR FALLO)
        POS->>CONC: getHosesSecondary() -> GET /api/v1/obtener-cargas-por-mangueras/
        CONC-->>POS: Cargas Locales de Pista
        POS->>POS: Aplica Precios & Timeout Viejo/Futuro (Storage Local)
        POS->>POS: Imprime Comprobante "MODO TEMPORAL"
    end
```

### 3.1. Modo Nominal (Online)
1. **Autenticación**: El usuario ingresa su CUIL y contraseña. Se envía POST a `/posapp/login` del servidor central.
2. **Inmutabilización de Datos Locales**: Al autenticar con éxito, el POS guarda de forma inmutable y encriptada (`setSecureItem`) la configuración operativa actual:
   * Módulos asignados (`pos_modulos`)
   * Concentradores y sus mangueras (`pos_concentradores`)
   * Reglas de fidelización y precios (`pos_fidelizacion_configs`, `pos_combustibles`)
   * Datos de la empresa/estación para impresión (`pos_datos_rotulo`: Razón Social, CUIT, Domicilio).
3. **Flujo Cargas**: Consulta regular vía POST `/posapp/obtenerCargas` al backend central.

### 3.2. Modo de Falla (Offline / Fallback)
1. **Detección Automática**: El componente `ConnectionBlocker` ejecuta una verificación periódica (cada 3 segundos) hacia `/system-status.json` o `/public/system-status`. Si falla o da tiempo de espera agotado, emite la alerta `'server-status-change'` (`window.isServerOnline = false`).
2. **Ingreso Offline**: El POS habilita el botón de contingencia **"Entrar a modo de fallo"**, ejecutando `loginOffline()`. Se asigna una sesión local temporal (`OPERADOR FALLO` con token `offline_fallback_token`).
3. **Consulta Distribución de Mangueras (`getHosesSecondary`)**:
   * PosApp identifica los concentradores no dependientes guardados localmente.
   * Consulta directamente vía HTTP local a los endpoints de los concentradores en pista:
     `POST {endpoint_concentrador}/api/v1/obtener-cargas-por-mangueras/`
   * Si no responden, recurre al `api_endpoint_secondary` configurado en el POS.
4. **Validación de Filtros y Precios Locales**:
   * Asocia las mangueras a los combustibles mapeados localmente (`pos_combustibles`).
   * Aplica reglas de timeout de cargas registradas (`TIMEOUT VIEJO` / `TIMEOUT FUTURO`) usando la fecha local.
5. **Impresión de Comprobante contingencia**:
   * Genera tickets ESC/POS locales etiquetados explícitamente como **"MODO TEMPORAL"**.
   * Se deshabilita la venta anticipada o canje diferido hasta el restablecimiento de la conexión.

---

## 4. Flujo de Pantallas de la Aplicación (`State Flow`)

El estado de navegación de `LayoutHome` evoluciona secuencialmente a través de la propiedad `step`:

```mermaid
stateDiagram-v2
    [*] --> start: Inicio (Botón Escanear / Manual / DNI)
    start --> scan: Iniciar Cámara / Escáner QR
    start --> manual: Entrada Teclado Numérico
    
    scan --> hose_selection: Cliente Validado
    manual --> hose_selection: Cliente Validado
    
    hose_selection --> consultar_factura: En Modo 3 ó 5 (Facturación)
    consultar_factura --> payment_selection: Factura Verificada
    
    hose_selection --> payment_selection: En Modo 2 ó 4 (Sin Facturación)
    
    payment_selection --> result: Cobro Procesado con Éxito
    result --> start: Imprimir Ticket y Finalizar
```

### Descripción de Pasos:
1. **`start`**: Pantalla inicial con las opciones de escaneo QR/Tarjeta o ingreso por DNI.
2. **`scan` / `manual`**: Captura del identificador de la tarjeta o número de documento del cliente.
3. **`hose_selection`**: Visualización interactiva de surtidores/mangueras con despachos pendientes (litros, importe, combustible).
4. **`consultar_factura`**: (Exclusivo Modos 3 y 5) Verificación del estado de factura emitido.
5. **`payment_selection`**: Selección del medio de pago (Efectivo, Tarjeta de Débito/Crédito, Mercado Pago, Puntos).
6. **`result`**: Confirmación del procesamiento y comando de impresión de comprobante vía impresora térmica ESC/POS integradas.

---

## 5. Resumen de Métodos API Relevantes

| Componente | Método | Endpoint / Destino | Función |
| :--- | :--- | :--- | :--- |
| **Config.jsx** | `getLogin` | `POST /posapp/login` (Central) | Autenticación de operador y carga de configuración. |
| **Config.jsx** | `obtenerCargas` | `POST /posapp/obtenerCargas` (Central) | Obtiene cargas pendientes centralizadas en Modo Nominal. |
| **Config.jsx** | `obtenerCargasPorMangueras` | `POST /api/v1/obtener-cargas-por-mangueras/` (Local) | Obtiene cargas directamente del concentrador local en Modo Fallo. |
| **ApiService.jsx** | `getHosesSecondary` | Algoritmo Multinegocio | Distribuye la consulta entre concentradores locales no dependientes. |
| **ConnectionBlocker.jsx** | `checkStatus` | `GET /public/system-status` | Polling constante de conectividad para detectar Modo Fallo/Nominal. |
