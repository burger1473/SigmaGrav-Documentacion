# Documentación Técnica: SigmaGrav Concentrador Software

## 1. Descripción General y Arquitectura

**SigmaGrav Concentrador Software** es un servicio backend desarrollado en **Python / Django** (Django REST Framework). Su función principal es actuar como middleware y panel de gestión entre la plataforma central y la placa concentradora hardware basada en **ESP32**.

### Roles Principales
* **Middleware TCP Binario:** Ofrece una interfaz cliente socket para comunicarse directamente con el servidor binario TCP del ESP32 concentrador.
* **Gestor de Estaciones:** Almacena y administra el estado en tiempo real de los surtidores, mangueras y transacciones de despacho.
* **API REST & Dashboard:** Expone endpoints RESTful para la integración con otros subsistemas y proporciona un panel web visual para el monitoreo de cargas.

---

## 2. Modos de Operación

El software soporta **dos modos de operación** configurables mediante la variable de entorno `MODO_OPERACION` (`RESPALDO` o `INTEGRADO`).

### 1. Modo RESPALDO (`MODO_OPERACION=RESPALDO`)
* **Base de Datos:** Levanta un contenedor PostgreSQL local propio (`sigmagrav_concentrador_db`) en puerto `5442`.
* **Tareas activas:** Ejecuta las 3 tareas en segundo plano:
  1. Monitoreo RS485.
  2. Sincronizador Backend.
  3. Mantenimiento Diario (1 AM).
* **Campo Endpoint:** En este modo se presenta como **ENDPOINT BACKEND SISTEMA** (URL del servidor central al que sincronizar).
* **Despliegue Docker:** `./docker-run.sh` (utiliza `--profile respaldo`).

### 2. Modo INTEGRADO (`MODO_OPERACION=INTEGRADO`)
* **Base de Datos:** **No levanta PostgreSQL local**, reduciendo recursos y conectándose directamente a la base de datos externa configurada.
* **Tareas activas:** **Solo ejecuta la tarea de Monitoreo RS485**. Omite automáticamente las tareas de Sincronizador y Mantenimiento Diario.
* **Campo Endpoint:** Se presenta en el panel como **ENDPOINT CONTRADOR RESPALDO** (URL del concentrador de respaldo).
* **Despliegue Docker:** `MODO_OPERACION=INTEGRADO ./docker-run.sh` (utiliza `--profile ""`).

---

## 3. Configuración Persistente en JSON y Autenticación MD5

El sistema utiliza un gestor de configuración en disco ([`json_config_manager.py`](file:///home/burger/workspace/repos/SigmaGrav/SigmaGrav-Concentrador/SigmaGrav-Concentrador-Software/surtidores/services/json_config_manager.py)) almacenado en un volumen Docker persistente (`config_data` montado en `/app/config_storage/configuracion.json`).

* **Formato del JSON:** Almacena el usuario, la contraseña encriptada en **MD5** (`password_md5`) y el `endpoint` configurado.
* **Verificación al Iniciar:** Si el archivo JSON o el usuario no existen al intentar ingresar al Dashboard, el sistema avisa e invita al administrador a crear el usuario y contraseña iniciales.
* **Importación y Exportación:** El panel permite exportar la configuración actual en archivo `.json` y restaurarla/importarla en cualquier momento desde el Dashboard.

---

## 3. Estructura del Proyecto

```
SigmaGrav-Concentrador-Software/
├── config/                 # Configuración del proyecto Django (settings, urls, wsgi)
├── surtidores/             # Aplicación principal del dominio de surtidores
│   ├── models.py           # Modelos de base de datos (Surtidor, Manguera, Transaccion, Carga)
│   ├── views.py            # API ViewSets de Django REST Framework
│   ├── views_dashboard.py  # Vistas para el panel de control web
│   ├── views_public.py     # Endpoints de la API pública e integración
│   ├── serializers.py      # Serializadores de datos JSON (DRF)
│   └── services/           # Lógica de comunicación de red y negocio
│       ├── concentrador_client.py  # Cliente Socket TCP y protocolo binario ESP32
│       ├── monitor.py              # Servicio de sondeo y monitoreo continuo
│       └── sync.py                 # Sincronización de transacciones con el backend central
├── templates/              # Plantillas HTML del Dashboard web
├── Dockerfile              # Configuración de imagen Docker
├── docker-compose.yml      # Definición del contenedor y servicios
├── docker-run.sh           # Script de arranque inteligente según el MODO_OPERACION
├── .env.example            # Plantilla de variables de entorno
├── manage.py               # CLI de Django
└── requirements.txt        # Dependencias del proyecto Python
```

---

## 4. Protocolo de Comunicación TCP Binario (`ConcentradorSigma`)

El módulo `surtidores/services/concentrador_client.py` implementa la clase `ConcentradorSigma`, encargada del empaquetado, envío y validación de las tramas enviadas a cualquiera de los **8 canales RS485** del concentrador.

### Estructura de la Cabecera de Solicitud (TCP -> ESP32)

| Campo | Tipo / Tamaño | Descripción |
| :--- | :--- | :--- |
| `startMarker` | `2 bytes` (`0xAA55`) | Identificador de inicio de trama |
| `rs485Channel` | `1 byte` (`uint8`) | Canal RS485 de destino (`0` al `7`) |
| `baudrate` | `4 bytes` (`uint32`) | Velocidad en baudios (ej: 9600, 19200) |
| `timeoutMs` | `2 bytes` (`uint16`) | Tiempo de espera de respuesta RS485 (ms) |
| `dataLength` | `2 bytes` (`uint16`) | Longitud del payload del comando RS485 |
| `payload` | *N bytes* | Comando en bytes según protocolo del surtidor |
| `crc8` | `1 byte` | Suma de comprobación XOR del payload |

### Códigos de Estado Devueltos por el Firmware

* `0x00` (`STATUS_OK`): Transmisión exitosa y respuesta RS485 recibida.
* `0x01` (`STATUS_TIMEOUT`): Dispositivo en el canal RS485 no respondió a tiempo.
* `0x02` (`STATUS_INVALID_CHANNEL`): Canal fuera del rango válido (0 a 7).
* `0x03` (`STATUS_INVALID_HEADER`): Marcador binario o cabecera TCP corrupta.
* `0x04` (`STATUS_INVALID_CHECKSUM`): Error de CRC8 en la trama.

---

## 5. Tareas en Segundo Plano y Mantenimiento

El sistema ejecuta 3 tareas concurrentes en hilos daemon ([`surtidores/tasks.py`](file:///home/burger/workspace/repos/SigmaGrav/SigmaGrav-Concentrador/SigmaGrav-Concentrador-Software/surtidores/tasks.py)) cuyo estado se persiste en la tabla `ConfiguracionTarea`:

1. **Monitoreo (`monitor.py`):** Realiza sondeo periódico de surtidores activos (polling RS485).
2. **Sincronización (`sync.py`):** Sincroniza transacciones locales pendientes enviándolas por HTTP POST al servidor central.
3. **Mantenimiento Diario (`maintenance.py`):** 
   * **Horario:** Se ejecuta automáticamente todos los días a la **1:00 AM** (01:00 hs).
   * **Función:** Elimina automáticamente registros de cargas y transacciones ya sincronizadas que superen el límite de retención configurado (`dias_retencion`).
   * **Configuración Persistente:** Se puede activar/desactivar y configurar los días de retención (ej: 15, 30, 60 días) directamente desde el Dashboard o la API `/api/v1/dashboard/tareas/`.

---

## 6. Modelo de Datos (`surtidores/models.py`)

1. **`Surtidor`**:
   * **Atributos:** `numero`, `marca` (Wayne, Gilbarco, Bennett, etc.), `estado` (`LIBRE`, `DESCOLGADO`, `DESPACHANDO`, `BLOQUEADO`, `FUERA_DE_SERVICIO`), `ip_address`, `puerto`, `activo`.
2. **`Manguera`**:
   * **Atributos:** `surtidor` (FK), `numero_manguera`, `producto`, `precio_unitario`, `totalizador_volumen`, `totalizador_dinero`.
3. **`TransaccionSurtidor`**:
   * **Atributos:** `surtidor` (FK), `manguera` (FK), `volumen` (litros), `monto`, `precio`, `fecha_hora`, `procesado_backend`.
4. **`Carga`**:
   * **Atributos:** Registro de eventos de carga/despacho con información adicional de la isla, cliente, estados de facturación y procesamiento.

---

## 7. Ejemplo de Uso de la Librería

```python
from surtidores.services import ConcentradorSigma, ConcentradorSigmaTimeoutError

try:
    with ConcentradorSigma(host="192.168.4.1", port=5000) as client:
        # Enviar comando Modbus / RS485 al canal 0
        respuesta = client.send_rs485_command(
            rs485_channel=0,
            payload=b"\x01\x03\x00\x00\x00\x02\xC4\x0B",
            baudrate=9600,
            timeout_ms=1000
        )
        print("Respuesta del surtidor (hex):", respuesta.hex())

except ConcentradorSigmaTimeoutError:
    print("El surtidor no respondió dentro del tiempo límite.")
except Exception as e:
    print("Error de comunicación:", e)
```
