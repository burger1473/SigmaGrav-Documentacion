# Protocolo de Comunicación CORITEC 2 / CORITEC 4 (ASPRO)
> **Especificación Técnica y Referencia Completa para Integración e Inteligencia Artificial / Modelos de Lenguaje**

Documentación estructurada y detallada del protocolo de comunicación serie RS-485 para controladores de surtidores/dispensers de GNC/Combustibles ASPRO modelos **CORITEC 2** y **CORITEC 4** (Basado en Especificación v4.1).

---

## 1. Configuración de Capa Física y Enlace Serie (RS-485)

- **Interfaz:** RS-485 Half-Duplex (multinodo maestro-esclavo)
- **Baudrate:** 9600 bps
- **Bits de Datos:** 8 bits
- **Paridad:** Ninguna (N)
- **Bits de Parada:** 1 bit
- **Delimitador de Campos:** `|` (pipe, ASCII `0x7C`)
- **Prefijo de Trama:** `:` (dos puntos, ASCII `0x3A`)
- **Prefijo de Checksum:** `%` (porcentaje, ASCII `0x25`)

---

## 2. Estructura General de Tramas

### 2.1 Trama de Consulta / Comando (PC / Concentrador → Surtidor)
```text
:nodo|funcion|argumentos|%<CRC_H><CRC_L><terminador>
```
- **`:nodo`**: Dirección del surtidor/manguera (ej. `1`, `2`).
- **`funcion`**: Carácter (1 byte) identificador de la función (`R`, `P`, `D`, `N`, `C`).
- **`argumentos`**: (Opcional) Datos del comando o parámetro.
- **`%`**: Delimitador del campo CRC16.
- **`<CRC_H><CRC_L>`**: 2 bytes binarios del CRC16 calculados sobre la cadena (desde `:` hasta el último argumento/pipe inclusive).
- **`<terminador>`**: Carácter de fin de comando:
  - `?` (ASCII `0x3F`): Para consultas de lectura (Get).
  - `!` (ASCII `0x21`): Para comandos de escritura/configuración (Set).

### 2.2 Trama de Respuesta (Surtidor → PC / Concentrador)
```text
:nodo|funcion|status|argumentos|%<CRC_H><CRC_L>*
```
- **`status`**: Código de estado numérico/alfanumérico actual del surtidor (ver Sección 4).
- **`<terminador>`**: Carácter `*` (ASCII `0x2A`) indica respuesta desde el surtidor.

---

## 3. Comandos y Funciones Básicas

| Carácter | Función | Descripción |
|---|---|---|
| **`R`** | Real Time | Estado en tiempo real, totalizador, venta acumulada, volumen y precio. |
| **`P`** | Precio Unitario | Consulta y seteo del precio unitario por litro/m³. |
| **`D`** | Densidad | Consulta y seteo de la densidad del producto. |
| **`N`** | Versión / Serie | Versión de firmware/hardware y número de serie (fecha y revisión). |
| **`C`** | Aforador / Aceptación | Confirmación/Reconocimiento de lectura de aforador solicitada desde el surtidor. |

---

### 3.1 Función `R`: Estado en Tiempo Real (Real Time)
Consulta el estado actual de la manguera y los valores en proceso o de la última venta.

- **Consulta PC:** `:1|R|%<CRC_H><CRC_L>?`
- **Respuesta Surtidor:** `:nodo|R|status|aforador|venta|volumen|precio_u|futuro0|futuro1|%<CRC_H><CRC_L>*`

#### Ejemplo:
- **Envío:** `:1|R|%!¯?`
- **Respuesta:** `:1|R|3|882785.12|184.06|183.88|1.001|0|1|%ùH*`

#### Campos de la Respuesta `R`:
1. `nodo`: `1`
2. `funcion`: `R`
3. `status`: `3` (Despachando)
4. `aforador`: Totalizador acumulado (ej. `882785.12`)
5. `venta`: Importe acumulado de la venta en curso/última (ej. `184.06`)
6. `volumen`: Volumen acumulado de la venta en curso/última (ej. `183.88`)
7. `precio_u`: Precio unitario configurado (ej. `1.001`)
8. `futuro0`: Reservado / Uso futuro (ej. `0`)
9. `futuro1`: Reservado / Uso futuro (ej. `1`)

---

### 3.2 Función `P`: Precio Unitario

#### A. Consulta de Precio Unitario (Get)
- **Envío PC:** `:1|P|%<CRC_H><CRC_L>?`  *(Ejemplo: `:1|P|%Ç▼?` donde `▼` = `0x1F`)*
- **Respuesta Surtidor:** `:1|P|3|1.001|%¦f*`
  - Campo 3 (`status`): `3`
  - Campo 4 (`precio`): `1.001`

#### B. Seteo de Precio Unitario (Set)
- **Envío PC:** `:1|P|0.333|%<CRC_H><CRC_L>!`  *(Ejemplo: `:1|P|0.333|%-+!`)*
- **Respuesta Surtidor:** `:1|P|0|0.333|%?Y*`
  - Campo 3 (`status`): `0`
  - Campo 4 (`precio_aplicado`): `0.333`

---

### 3.3 Función `D`: Densidad del Sistema

#### A. Consulta de Densidad (Get)
- **Envío PC:** `:1|D|%<CRC_H><CRC_L>?`  *(Ejemplo: `:1|D|%+←?` donde `←` = `0x1B`)*
- **Respuesta Surtidor:** `:1|D|0|0.740|%~↔*` *(donde `↔` = `0x1D`)*

#### B. Seteo de Densidad (Set)
- **Envío PC:** `:1|D|0.743|%<CRC_H><CRC_L>!` *(Ejemplo: `:1|D|0.743|%¦m!`)*
- **Respuesta Surtidor:** `:1|D|0|0.743|%Ä↔*`

---

### 3.4 Función `N`: Versión y Número de Serie
Devuelve la información de fabricación y versión de firmware de la controladora.

- **Envío PC:** `:1|N|%<CRC_H><CRC_L>?`  *(Ejemplo: `:1|N|%a↓?` donde `↓` = `0x19`)*
- **Respuesta Surtidor:** `:1|N|0|08097006|%♪d*` *(donde `♪` = `0x0D`)*

#### Descomposición del Número de Serie (`08097006`):
- `08`: Mes de fabricación (2 dígitos, MM)
- `09`: Año de fabricación (2 dígitos, YY)
- `7006`: Versión de Software/Firmware (4 dígitos, ej. 7.00.6)

---

### 3.5 Función `C`: Reconocimiento / Aceptación de Aforador
Cuando el playero/operador solicita la lectura de aforador directamente desde el teclado/surtidor, el surtidor notifica este estado a la PC mediante respuestas a las consultas `R`. La PC **debe** responder enviando una trama con función `C` para confirmar y desbloquear la operación.

- **Envío PC:** `:1|C|1|%<CRC_H><CRC_L>?`
  *(El argumento `1` se requiere para completar el formato de trama).*
- **Respuesta Surtidor:** `:1|C|0|882812.98|%ff*`
  - Campo 4: Valor del aforador registrado (`882812.98`).
- **Efecto visual:** En la pantalla del surtidor apareciendo la letra `F` en el primer dígito de la línea superior confirmando la operación.

---

## 4. Tabla de Estados (`status`) del Surtidor

El valor de `status` presente en el tercer campo de las respuestas indica la fase o estado operativo actual de la manguera:

| Valor | Estado Operativo | Descripción |
|---|---|---|
| **`0`** | Reposo | Manguera colgada, surtidor listo para iniciar ciclo. |
| **`1`**, **`2`** | Manguera descolgada | Manguera fuera de portamanguera, sin iniciar carga. |
| **`3`** | Despachando | Carga/Despacho de combustible activo. |
| **`4`**, **`5`** | Fin de Carga | Venta finalizada (manguera aún descolgada). |
| **`6`** | Time Out | Tiempo de espera agotado con manguera descolgada. |
| **`7`** | Haciendo CERO | En proceso de puesta a cero de los displays antes del despacho. |
| **`8`** | Menú | Operador interactuando con el menú local del surtidor. |
| **`9`** | Venta Fuera de Control | Alarma/Anomalía (posible falla de solenoide o intento de manipulación/robo). |
| **`A`** | Batería | Estado de advertencia o funcionamiento en batería de respaldo. |
| **`G`** | CERO Finalizado | Proceso de puesta a cero completado con éxito. |

---

## 5. Respuestas de Error del Surtidor

Si el surtidor recibe una trama inválida, con error de CRC o fuera de secuencia, responde con una trama especial de error:

```text
:nodo|funcion|status|E|codigo_error|%<CRC_H><CRC_L>*
```
*Ejemplo de respuesta a comando con CRC incorrecto:* `:1|P|0|E|3|% t*`

### Códigos de Error:

| Código | Descripción del Error |
|---|---|
| **`0`** | OK (Sin error) |
| **`1`** | Función no soportada |
| **`2`** | Función inválida |
| **`3`** | Error de CRC |
| **`4`** | Error de formato del argumento |
| **`5`** | Error de longitud del argumento |
| **`6`** | Error en el valor del argumento |
| **`7`** | Función no soportada en el estado actual |
| **`8`** | Función redundante |
| **`9`** | Terminador inválido para esta función |
| **`A`** | Error durante proceso de CERO |

---

## 6. Algoritmo de Cálculo de Checksum (CRC16 Modbus Modificado)

El algoritmo empleado es un **CRC16 con polinomio `0xA001` (LSB first)** y valor inicial `0xFFFF`, con una **regla de sustitución obligatoria**: si cualquier byte resultante del CRC16 es `0x00`, debe ser sustituido por `0xFF` para evitar bytes nulos en el envío serie ASCII.

### 6.1 Reglas Críticas del CRC:
1. **Rango de Cálculo:** Incluye desde el carácter `:` inicial hasta el último carácter antes de `%`.
2. **Sustitución de Byte Nulo (`0x00 → 0xFF`):**
   - Si `CRC_L == 0x00` $\rightarrow$ `CRC_L = 0xFF`
   - Si `CRC_H == 0x00` $\rightarrow$ `CRC_H = 0xFF`
3. **Orden de Transmisión:** Primero se envía `CRC_H` y luego `CRC_L`.

---

### 6.2 Implementación de Referencia en C

```c
/*
  Variables globales o de salida: CRC_L y CRC_H
  ptr: Puntero al inicio de la cadena (comenzando en ':')
  lonMen: Cantidad de caracteres de la cadena a calcular (excluyendo '%' y terminadores)
*/

unsigned char CRC_L, CRC_H;

void CRC16(unsigned char *ptr, int lonMen) {
    unsigned int crc = 0xFFFF;
    unsigned char Carry;
    int i;

    while (lonMen > 0) {
        crc = crc ^ (*ptr);
        for (i = 0; i < 8; i++) {
            Carry = (unsigned char)(crc & 0x0001);
            crc >>= 1;
            if (Carry) {
                crc = crc ^ 0xA001;
            }
        }
        ptr++;
        lonMen--;
    }

    CRC_L = (unsigned char)((crc >> 8) & 0xFF);
    CRC_H = (unsigned char)(crc & 0xFF);

    if (CRC_L == 0x00) CRC_L = 0xFF;
    if (CRC_H == 0x00) CRC_H = 0xFF;
}
```

---

### 6.3 Implementación de Referencia en Python 3

```python
def calculate_coritec_crc16(data: bytes) -> bytes:
    """
    Calcula el CRC16 del protocolo CORITEC para un array de bytes (desde ':' hasta previo a '%').
    Aplica la sustitución de 0x00 por 0xFF y retorna los 2 bytes CRC (CRC_H, CRC_L).
    """
    crc = 0xFFFF
    for byte in data:
        crc ^= byte
        for _ in range(8):
            carry = crc & 0x0001
            crc >>= 1
            if carry:
                crc ^= 0xA001
                
    crc_l = (crc >> 8) & 0xFF
    crc_h = crc & 0xFF
    
    # Regla de sustitución de bytes nulos del protocolo CORITEC
    if crc_l == 0x00:
        crc_l = 0xFF
    if crc_h == 0x00:
        crc_h = 0xFF
        
    return bytes([crc_h, crc_l])

def build_coritec_frame(node: int, function: str, args: str = "", is_set: bool = False) -> bytes:
    """
    Construye una trama completa del protocolo CORITEC.
    """
    body = f":{node}|{function}"
    if args:
        body += f"|{args}"
    
    body_bytes = body.encode('latin1')
    crc_bytes = calculate_coritec_crc16(body_bytes)
    terminator = b'!' if is_set else b'?'
    
    return body_bytes + b'%' + crc_bytes + terminator
```

---

## 7. Resumen de Flujos de Diálogo Comunes

### Flujo 1: Monitoreo continuo (Polling)
```text
PC  --->  :1|R|%<CRC_H><CRC_L>?  --->  Surtidor
PC  <---  :1|R|0|882785.12|0.00|0.00|1.001|0|1|%<CRC_H><CRC_L>*  <--- Surtidor (Reposo)
```

### Flujo 2: Despacho en curso
```text
PC  --->  :1|R|%<CRC_H><CRC_L>?  --->  Surtidor
PC  <---  :1|R|3|882785.12|54.20|54.14|1.001|0|1|%<CRC_H><CRC_L>*  <--- Surtidor (Despachando)
```

### Flujo 3: Cambio de Precio Unitario
```text
PC  --->  :1|P|1.250|%<CRC_H><CRC_L>!  --->  Surtidor
PC  <---  :1|P|0|1.250|%<CRC_H><CRC_L>*    <--- Surtidor (Precio aceptado)
```

---
*Documentación generada para procesamiento por modelos de lenguaje (LLM) e integración de software concentrador SigmaGrav.*
