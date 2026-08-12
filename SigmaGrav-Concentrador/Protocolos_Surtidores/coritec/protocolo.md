# Análisis Técnico y Especificación del Protocolo de Comunicación RS-485 para Surtidores y Controladoras CORITEC 2 y CORITEC 4

## Contexto Técnico y Arquitectura Electrónica de las Controladoras CORITEC 2 y CORITEC 4

Las controladoras electrónicas **CORITEC 2** y **CORITEC 4** de ASPRO representan arquitecturas centralizadas de cabezal electrónico diseñadas para el control, cálculo volumétrico y despacho de GNC (Gas Natural Comprimido) y combustibles líquidos en dispensadores de la marca ASPRO. La denominación CORITEC 2 hace referencia a unidades capaces de controlar hasta 2 mangueras / posiciones de carga, mientras que la CORITEC 4 administra hasta 4 mangueras / productos de forma independiente y simultánea.

La arquitectura interna de la controladora CORITEC procesa las señales de alta precisión de los medidores de masa (tipo Coriolis o sensores de pulso), gestiona las válvulas solenoides de corte y etapas de presión, administra las pantallas de indicación monetaria, de volumen y precio unitario, y expone interfaces serie para la interconexión con sistemas concentradores de pista (Forecourt Controllers) e inteligencia de automatización de estaciones de servicio.

| **Parámetro del Equipo** | **Especificación Técnica** |
| --- | --- |
| **Fabricante / Entidad Distribuidora** | ASPRO / Sistemas de Compresión S.A. |
| **Modelos de Controladora** | CORITEC 2 y CORITEC 4 (Firmware v4.1 / SI-2000) |
| **Aplicación Principal** | Dispensadores de GNC y Combustibles Líquidos ASPRO |
| **Configuración Hidráulica / Neumática** | 2 Mangueras (CORITEC 2) / 4 Mangueras (CORITEC 4) |
| **Capa Física de Comunicación** | RS-485 Semidúplex (2 hilos) |
| **Protocolo Lógico** | Protocolo Propietario ASCII/Binario CORITEC |
| **Control de Integridad** | CRC16 (Polinomio `0xA001`, Init `0xFFFF` con sustitución `0x00` $\rightarrow$ `0xFF`) |

---

## Capa Física e Interfaz de Comunicación RS-485

La interconexión serie entre el computador concentrador maestro de la estación de servicio y la controladora CORITEC se realiza mediante un bus asíncrono RS-485 diferencial semidúplex (Half-Duplex). La interfaz física utiliza dos conductores activos balanceados denotados como línea A (Data+) y línea B (Data-), acompañados por una referencia de masa común (GND) destinada a estabilizar los voltajes de modo común en el bus.

El estándar de transmisión opera con niveles de tensión diferencial EIA/TIA-485, donde un estado lógico '1' se define por un voltaje diferencial $\Delta V_{AB} > +200\text{ mV}$ y un estado '0' por $\Delta V_{AB} < -200\text{ mV}$.

| **Parámetro Serial** | **Especificación Estándar CORITEC** |
| --- | --- |
| **Velocidad de Transmisión (Baud Rate)** | 9600 bps |
| **Bits de Datos** | 8 bits |
| **Paridad** | Ninguna (No Parity - N) |
| **Bits de Parada (Stop Bits)** | 1 bit |
| **Esquema de Red** | Maestro-Esclavo Multipunto RS-485 |
| **Topología Físico-Lógica** | Bus 2 hilos semidúplex con delimitador de pipe `|` |

---

## Configuración de Capa de Enlace y Enmarcado Serial

El protocolo CORITEC opera bajo una topología maestro-esclavo mediante sondeo periódico (*Master-Slave Polling*), donde el sistema concentrador maestro interroga continuamente a los nodos direccionados y cada manguera/surtidor responde únicamente al recibir una trama dirigida a su identificador de nodo.

### Esquema de Direccionamiento Multipunto

Cada manguera o posición física de carga administrada por la controladora CORITEC posee un identificador de nodo (`nodo`) codificado como número entero en ASCII (ejemplo: `1`, `2`, `3`, `4`). El concentrador maestro envía tramas dirigidas al nodo de la manguera y evalúa la respuesta retornada por dicho nodo.

---

## Estructura del Protocolo y Enmarcado de Tramas

El protocolo de comunicación CORITEC estructura sus mensajes mediante campos delimitados por el carácter pipe `|` (Hex `0x7C`). Las tramas utilizan prefijos y terminadores específicos para diferenciar las peticiones enviadas por el maestro de las respuestas generadas por la controladora.

### 1. Estructura de Trama de Consulta (PC / Concentrador Maestro $\rightarrow$ Surtidor)

$$
\text{Trama Consulta} = \text{`:`} + [\text{nodo}] + \text{`|`} + [\text{funcion}] + \text{`|`} + [\text{argumentos}] + \text{`|%`} + [\text{CRC\_H}] + [\text{CRC\_L}] + [\text{terminador}]
$$

- **`:`** (Hex `0x3A`): Carácter ASCII de inicio de trama.
- **`nodo`**: Dirección del surtidor o manguera en ASCII (ej. `1`).
- **`funcion`**: Carácter ASCII de 1 byte que especifica el comando (`R`, `P`, `D`, `N`, `C`).
- **`argumentos`**: (Opcional) Datos del argumento del comando separados por pipes `|`.
- **`%`** (Hex `0x25`): Delimitador de inicio del campo de suma de comprobación CRC16.
- **`CRC_H CRC_L`**: 2 bytes binarios del checksum CRC16 calculados sobre el cuerpo de la trama.
- **`terminador`**: Carácter de fin de trama de consulta:
  - `?` (Hex `0x3F`): Utilizado para peticiones de consulta / lectura (*Get*).
  - `!` (Hex `0x21`): Utilizado para comandos de seteo / escritura (*Set*).

### 2. Estructura de Trama de Respuesta (Surtidor $\rightarrow$ PC / Concentrador Maestro)

$$
\text{Trama Respuesta} = \text{`:`} + [\text{nodo}] + \text{`|`} + [\text{funcion}] + \text{`|`} + [\text{status}] + \text{`|`} + [\text{argumentos}] + \text{`|%`} + [\text{CRC\_H}] + [\text{CRC\_L}] + \text{`*`}
$$

- **`status`**: Código de estado operativo actual del surtidor (ver tabla de estados).
- **`*`** (Hex `0x2A`): Carácter ASCII de fin de trama que identifica una respuesta devuelta por el surtidor.

---

## Catálogo Completo y Especificación de Comandos

El protocolo CORITEC implementa 5 funciones principales identificadas por un único carácter ASCII:

| Carácter | Función | Descripción y Comportamiento Operativo |
| --- | --- | --- |
| **`R`** | Real Time | Consulta del estado en tiempo real, totalizador acumulado, venta monetaria, volumen y precio unitario. |
| **`P`** | Precio Unitario | Consulta y modificación del precio unitario por litro o metro cúbico. |
| **`D`** | Densidad | Consulta y modificación de la densidad del producto. |
| **`N`** | Versión / Serie | Retorna el número de serie de la manguera (fecha de fabricación) y versión de firmware. |
| **`C`** | Aforador / Aceptación | Aceptación y confirmación de lectura del aforador solicitada desde el teclado del surtidor. |

---

### 1. Función `R`: Estado en Tiempo Real (Real Time)

La función `R` es el comando primario de sondeo (*polling*). Permite monitorear continuamente el estado del surtidor, obtener las lecturas instantáneas del despacho en curso y registrar los valores finales de las ventas concluidas.

#### Trama de Consulta (PC $\rightarrow$ Surtidor):
```text
:1|R|%<CRC_H><CRC_L>?
```
*(Ejemplo en ASCII/Hex: `:1|R|%!¯?`)*

#### Trama de Respuesta (Surtidor $\rightarrow$ PC):
```text
:1|R|status|aforador|venta|volumen|precio_u|futuro0|futuro1|%<CRC_H><CRC_L>*
```
*(Ejemplo real: `:1|R|3|882785.12|184.06|183.88|1.001|0|1|%ùH*`)*

#### Desglose de Campos de Respuesta `R`:
1. **`nodo`**: `1` (Dirección del nodo).
2. **`funcion`**: `R`.
3. **`status`**: `3` (Estado operativo: *Despachando*).
4. **`aforador`**: Lectura del totalizador acumulado volumétrico (ej. `882785.12`).
5. **`venta`**: Importe monetario acumulado del despacho en curso o de la última venta (ej. `184.06`).
6. **`volumen`**: Volumen acumulado del despacho en curso o de la última venta (ej. `183.88`).
7. **`precio_u`**: Precio unitario configurado y aplicado en el despacho (ej. `1.001`).
8. **`futuro0`**: Campo reservado para uso futuro (ej. `0`).
9. **`futuro1`**: Campo reservado para uso futuro (ej. `1`).

---

### 2. Función `P`: Consulta y Seteo de Precio Unitario

Permite consultar o actualizar dinámicamente el precio unitario del producto programado en la controladora CORITEC.

#### A. Consulta de Precio Unitario (Get PPU)
- **Consulta:** `:1|P|%<CRC_H><CRC_L>?` *(Ejemplo: `:1|P|%Ç▼?` donde `▼` = `0x1F`)*
- **Respuesta:** `:1|P|3|1.001|%¦f*`
  - `status`: `3`
  - `precio_u`: `1.001`

#### B. Seteo de Precio Unitario (Set PPU)
- **Consulta:** `:1|P|0.333|%<CRC_H><CRC_L>!` *(Ejemplo: `:1|P|0.333|%-+!`)*
- **Respuesta:** `:1|P|0|0.333|%?Y*`
  - `status`: `0`
  - `precio_u`: `0.333` (Confirmación de precio aplicado).

---

### 3. Función `D`: Consulta y Seteo de Densidad

Administra el valor de la densidad del producto configurado en la memoria de la controladora CORITEC para corrección de masa/volumen.

#### A. Consulta de Densidad (Get Density)
- **Consulta:** `:1|D|%<CRC_H><CRC_L>?` *(Ejemplo: `:1|D|%+←?` donde `←` = `0x1B`)*
- **Respuesta:** `:1|D|0|0.740|%~↔*` *(donde `↔` = `0x1D`)*

#### B. Seteo de Densidad (Set Density)
- **Consulta:** `:1|D|0.743|%<CRC_H><CRC_L>!` *(Ejemplo: `:1|D|0.743|%¦m!`)*
- **Respuesta:** `:1|D|0|0.743|%Ä↔*`

---

### 4. Función `N`: Consulta de Versión y Número de Serie

Retorna la información de hardware, fecha de fabricación y revisión de firmware de la manguera seleccionada.

- **Consulta:** `:1|N|%<CRC_H><CRC_L>?` *(Ejemplo: `:1|N|%a↓?` donde `↓` = `0x19`)*
- **Respuesta:** `:1|N|0|08097006|%♪d*` *(donde `♪` = `0x0D`)*

#### Estructura del Número de Serie Devuelto (`08097006`):
- `08`: Mes de fabricación (2 dígitos, MM).
- `09`: Año de fabricación (2 dígitos, YY).
- `7006`: Versión de Software/Firmware (4 dígitos, equivalente a v7.00.6).

---

### 5. Función `C`: Aceptación y Reconocimiento de Lectura de Aforador

Cuando el playero/operador solicita la lectura del aforador directamente desde el teclado del surtidor, la controladora notifica esta condición a la PC mediante las respuestas a la función `R`. Para proceder y desbloquear la operativización del surtidor, la PC **debe** responder enviando una trama de aceptación con la función `C`.

- **Consulta PC:** `:1|C|1|%<CRC_H><CRC_L>?`
  *(El argumento `1` es un parámetro de relleno para completar la longitud del formato).*
- **Respuesta Surtidor:** `:1|C|0|882812.98|%ff*`
  - Retorna el valor del aforador en el campo 4 (`882812.98`).
- **Efecto en Pantalla:** Al procesar la función `C`, aparecerá una letra `F` en el primer dígito de la línea superior del display de la manguera correspondiente, confirmando al operador el éxito de la operación.

---

## Máquina de Estados Finita y Ciclo de Vida de la Transacción

El control de un surtidor CORITEC requiere que el software concentrador mantenga una máquina de estados finitos que interprete el código de `status` retornado en el tercer campo de cada respuesta:

| Código `status` | Estado Operativo | Descripción y Comportamiento en Pista |
| --- | --- | --- |
| **`0`** | Reposo (Idle) | Manguera colgada en su soporte. Surtidor listo para nuevo ciclo. |
| **`1`**, **`2`** | Descolgado | Manguera fuera de soporte sin iniciar flujo de producto. |
| **`3`** | Despachando | Entrega activa de producto (medidor de pulsos/masa activo). |
| **`4`**, **`5`** | Fin de Carga | Despacho finalizado con manguera aún descolgada. |
| **`6`** | Time Out | Tiempo de espera de despacho agotado con manguera descolgada. |
| **`7`** | Haciendo CERO | Puesta a cero de contadores e indicación en pantallas. |
| **`8`** | Menú | Operador navegando en el menú de configuración local. |
| **`9`** | Venta Fuera de Control | Alarma por posible fallo de solenoide o intento de robo/despacho no autorizado. |
| **`A`** | Batería | Operación en modo de respaldo por batería o fallo de alimentación principal. |
| **`G`** | CERO Finalizado | Proceso de puesta a cero completado con éxito. |

---

## Respuestas de Error de la Controladora

Ante condiciones anómalas, errores de suma de comprobación CRC16 o comandos fuera de secuencia, la controladora CORITEC devuelve una trama de error estructurada con el carácter `E` en el campo 4:

```text
:nodo|funcion|status|E|codigo_error|%<CRC_H><CRC_L>*
```
*(Ejemplo de respuesta a comando con CRC erróneo:* `:1|P|0|E|3|% t*`*)*

### Catálogo de Códigos de Error:

| Código | Tipo de Error | Descripción |
| --- | --- | --- |
| **`0`** | OK | Operación exitosa (sin error). |
| **`1`** | Función no soportada | El código de función enviado no existe en el firmware. |
| **`2`** | Función inválida | Sintaxis o estructura de función incorrecta. |
| **`3`** | Error de CRC | El byte de CRC16 no coincide con el cálculo interno. |
| **`4`** | Error de formato del argumento | El argumento no cumple la especificación de formato. |
| **`5`** | Error de longitud del argumento | Cantidad de caracteres del argumento fuera de rango. |
| **`6`** | Error en el valor del argumento | Valor numérico del argumento fuera de límites permitidos. |
| **`7`** | Función no soportada en este estado | Intento de ejecutar un comando en una fase no válida. |
| **`8`** | Función redundante | Se intentó ejecutar una orden ya activa o procesada. |
| **`9`** | Terminador inválido | Se utilizó un carácter terminador distinto de `?` o `!`. |
| **`A`** | Error de CERO | Fallo en la rutina de puesta a cero del display. |

---

## Algoritmo de Verificación CRC16 e Implementación de Código

El control de integridad del protocolo CORITEC utiliza un algoritmo **CRC16 con polinomio `0xA001` (LSB first)** e inicialización en `0xFFFF`.

### Regla Obligatoria de Sustitución de Byte Nulo (`0x00 → 0xFF`):
Debido a que ciertos sistemas serie y microcontroladores truncan las tramas al recibir un byte nulo (`0x00`), la especificación de CORITEC exige reemplazar cualquier byte final del CRC que resulte igual a `0x00` por el valor `0xFF`:

$$
\text{Si } \text{CRC\_L} == \text{0x00} \implies \text{CRC\_L} = \text{0xFF}
$$

$$
\text{Si } \text{CRC\_H} == \text{0x00} \implies \text{CRC\_H} = \text{0xFF}
$$

**Orden de Transmisión del CRC:** En la trama se transmite primero `CRC_H` y luego `CRC_L` inmediatamente después del carácter `%`.

---

### Implementación de Referencia en Lenguaje C

```c
/*
  CRC_L y CRC_H son variables globales o de retorno.
  ptr: Apunta al comienzo de la cadena (carácter ':').
  lonMen: Cantidad de caracteres de la cadena a calcular (excluyendo '%' y terminadores).
*/

unsigned char CRC_L, CRC_H;

void CRC16(unsigned char *ptr, int lonMen) {
    unsigned int crc = 0xFFFF;
    unsigned char Carry;
    char x;

    while (--lonMen >= 0) {
        crc = crc ^ (*ptr);
        x = 0x08;
        do {
            Carry = (unsigned char)(crc & 0x01);
            crc >>= 1;
            if (Carry) {
                crc = crc ^ 0xA001;
            }
        } while (--x);
        ptr++;
    }

    CRC_L = (unsigned char)((crc >> 8) & 0xFF);
    CRC_H = (unsigned char)(crc & 0xFF);

    /* Regla de sustitución de bytes nulos del protocolo CORITEC */
    if (CRC_L == 0x00) CRC_L = 0xFF;
    if (CRC_H == 0x00) CRC_H = 0xFF;
}
```

---

### Implementación de Referencia en Python 3

```python
def calculate_coritec_crc16(data: bytes) -> bytes:
    """
    Calcula el CRC16 Modbus modificado para el protocolo CORITEC.
    Recibe el cuerpo de la trama en bytes (desde ':' hasta el último carácter previo a '%').
    Retorna 2 bytes: (CRC_H, CRC_L) aplicando la sustitución de 0x00 por 0xFF.
    """
    crc = 0xFFFF
    for byte in data:
        crc ^= byte
        for _ in range(8):
            carry = crc & 0x01
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


def build_coritec_request(node: int, function: str, args: str = "", is_set: bool = False) -> bytes:
    """
    Construye una trama de consulta para enviar a una controladora CORITEC.
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

## Conclusiones y Guía de Implementación para Concentradores

1. **Configuración de Línea:** Ajustar el puerto serie a **9600 bps, 8 bits de datos, Paridad Ninguna (N), 1 bit de parada (8-N-1)** sobre topología RS-485 semidúplex.
2. **Ciclo de Sondeo:** Implementar una rutina de sondeo periódico enviando la función `R` a cada nodo de manguera registrado a intervalos de entre 100 ms y 250 ms.
3. **Manejo de Aceptaciones de Aforador:** Cuando el surtidor retorne requerimiento de aforador en pantalla, el concentrador debe responder emitiendo la trama de función `C` (`:1|C|1|%...`) para permitir la normalización del servicio.
4. **Verificación de Integrity Check:** Validar estrictamente el byte de CRC16 e implementar el algoritmo de sustitución de `0x00` por `0xFF` tanto en la generación de tramas hacia el surtidor como en la decodificación de las respuestas recibidas.
