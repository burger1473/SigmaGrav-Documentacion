# Análisis Técnico y Especificación del Protocolo de Comunicación RS-485 para Surtidores Wayne Modelo 3G3399P

## Contexto Técnico y Arquitectura Electrónica del Surtidor Wayne 3G3399P

El surtidor Wayne modelo 3G3399P pertenece a la familia de dispensadores de combustible Global Century 3G desarrollados comercial e industrialmente por Dresser Wayne y distribuidos en el mercado latinoamericano a través de Dresser Utility Solutions Brasil y Dover Fueling Solutions^^. La denominación 3G3399P identifica a un equipo con configuración hidráulica de dos productos y dos mangueras de suministro simultáneo, dotado de dos pantallas electrónicas digitales independientes y soporte para impresora de recibos integrada^^. En su diseño volumétrico nominal, opera con caudales diferenciados por bomba, ofreciendo una capacidad de despacho de 4 L/min a 40 L/min en la primera línea de producto y de 7 L/min a 70 L/min en la segunda línea^^.

La arquitectura de control centralizada de este modelo está construida en torno al computador de cabezal electrónico Wayne iGEM (Integrated Global Electronic Module) o GEM^^. Este módulo electrónico procesa las señales de los generadores de impulsos electromecánicos iMETER, gestiona la apertura y cierre proporcional de las electroválvulas de corte, controla las pantallas de indicación monetaria y de volumen, y administra los puertos de comunicación serie diseñados para la interconexión con controladores de pista (FCC) y sistemas de punto de venta (POS)^^.

| **Parámetro del Equipo**       | **Especificación Técnica**                                           |
| ------------------------------------- | ---------------------------------------------------------------------------- |
| Fabricante / Entidad Distribuidora    | Dresser Wayne / Dresser Utility Solutions Brasil / Dover Fueling Solutions^^ |
| Modelo Comercial y Serie              | 3G3399P (Serie Global Century 3G)^^                                          |
| Arquitectura del Computador Principal | Wayne iGEM / GEM Computer^^                                                  |
| Configuración Hidráulica            | 2 Productos, 2 Mangueras, 2 Displays^^                                       |
| Rangos de Caudal Nominales            | Bomba 1: 4 - 40 L/min; Bomba 2: 7 - 70 L/min^^                               |
| Capa Física de Comunicación Serie   | RS-485 Semidúplex / Bucle de Corriente US (Current Loop)^^                  |
| Protocolos Principales Soportados     | Wayne DART Protocol (Standard/Full), Protocolo RS-485 BCD/Modbus^^           |

Para llevar a cabo el gobierno, monitoreo y extracción de datos en tiempo real de este surtidor mediante el puerto RS-485, se requiere dominar tanto la configuración física de la capa de enlace como los parámetros internos de programación del iGEM y el formato de las tramas de comandos que componen su protocolo de comunicación^^.

## Capa Física e Interfaz de Comunicación RS-485

La comunicación serie entre el controlador de la estación de servicio y el surtidor Wayne 3G3399P se realiza mediante un bus asíncrono RS-485 diferencial semidúplex^^. La interfaz física utiliza dos conductores activos balanceados denotados como línea A (Data+) y línea B (Data-), acompañados por una referencia de masa común (GND) destinada a estabilizar los voltajes de modo común en el bus^^.

El estándar de transmisión opera con niveles de tensión diferencial EIA-485, donde un estado lógico '1' se define por un voltaje **$\Delta V_{AB} > +200\text{ mV}$** y un estado '0' por **$\Delta V_{AB} < -200\text{ mV}$**. La velocidad de transmisión por defecto en las interfaces de automatización RS-485 de Wayne está fijada en 9600 baudios, aunque el firmware admite velocidades de 1200 bps a 38400 bps^^. La estructura del carácter comprende 1 bit de inicio (start bit), 8 bits de datos, sin paridad (o paridad impar según la variante DART habilitada) y 1 bit de parada (stop bit)^^.

En aquellos casos donde la placa principal del iGEM cuenta originalmente con salidas en bucle de corriente (Current Loop), la conexión al bus RS-485 de la estación de servicio se realiza a través de placas conversoras dedicadas de interfaz, como la tarjeta Invenco PIB RS485 o módulos de conversión tipo CommVerter / PTS^^. El controlador maestro debe coordinar la señal de control de dirección (RTS) para conmutar la línea RS-485 entre los estados de transmisión y recepción, evitando colisiones de bus^^.

## Parametrización y Configuración iGEM para Control Serie

El computador iGEM requiere ser configurado internamente para transferir la autoridad de control al puerto serie y responder a las peticiones transmitidas por el bus RS-485^^. Esta parametrización se efectúa accediendo a las funciones del equipo mediante el control remoto por infrarrojos Wayne (IRC, P/N WM002290) tras introducir las contraseñas de acceso (PASS 1 y PASS 2)^^.

El procedimiento de configuración abarca el cambio de modo de operación local a modo serie, la asignación de direcciones lógicas de punto de suministro (FPID) para cada lado del surtidor y la selección del protocolo de red en la función F20^^.

La función F01 define el modo de llenado (Filling Mode)^^. La sub-función `.00` debe programarse en el valor `1` (Serial Mode) para habilitar el control remoto mediante el canal de comunicaciones serie^^. Los valores `2` o `4` corresponden a modos autónomos (Stand-Alone), los cuales ignoran las órdenes enviadas desde el bus RS-485^^.

La asignación de la dirección del punto de suministro se efectúa en las funciones F05 y F06^^. La sub-función F05.00 define la dirección lógica del lado A, permitiendo seleccionar números enteros entre `1` y `98`^^. De manera análoga, la sub-función F06.00 asigna la dirección correspondiente al lado B^^. El valor `0` desactiva el lado correspondiente^^.

En versiones de firmware iGEM 83.05 o superiores, se debe retirar el puente físico de modo autónomo (standalone jumper) de la placa electrónica^^. Si este puente permanece instalado estando el surtidor configurado en modo serie, la pantalla mostrará la leyenda de bloqueo "Closed" y declinará cualquier petición de servicio^^.

| **Sub-función F20** | **Descripción del Parámetro**      | **Opciones de Configuración**                                                                                        | **Ajuste Recomendado RS-485**        |
| -------------------------- | ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| **F20.00**           | Protocolo Serie Seleccionado               | 0 = Off Link``1 = Standard DART``2 = FULL DART``4 = US Current Loop``12 = Nuovo Pignone RS485^^ | `1`(Standard DART) o `2`(Full DART)^^  |
| **F20.01**           | Velocidad de Transmisión (Baud Rate)      | 1 = 1200 bps``2 = 2400 bps``3 = 4800 bps``4 = 9600 bps``5 = 19200 bps``6 = 38400 bps^^   | `4`(9600 bps - estándar de industria)^^ |
| **F20.02**           | Suma de Comprobación (CRC)                | 1 = Habilitado``2 = Deshabilitado^^                                                                                  | `1`(CRC Habilitado)^^                    |
| **F20.03**           | Envío de "Llenado Completo" en venta cero | 1 = Sí``2 = No^^                                                                                                    | `1`(Sí)^^                               |
| **F20.04**           | Control Serie de Lámpara de Cabezal       | 1 = Sí``2 = No^^                                                                                                    | Según requerimiento del sitio             |

## Estructura del Protocolo Wayne DART y Trama de Datos RS-485

La comunicación en surtidores Wayne se rige mediante el protocolo propietario **Wayne DART** (en sus modalidades Standard DART o Full DART)^^. El protocolo opera bajo una topología maestro-esclavo mediante sondeo periódico (Master-Slave Polling), donde el sistema central interroga continuamente a las direcciones asignadas y el surtidor responde únicamente al recibir una trama dirigida a su identificador^^.

El ciclo de vida de la comunicación se basa en una máquina de estados finitos que transita entre reposo (Idle), petición de autorización (Nozzle Lift), autorización concedida (Authorized/Preset), despacho activo (Fueling) y fin de transacción (Filling Complete)^^.

Para controladores de automatización y placas conversoras serie, el protocolo Wayne implementa una trama estructurada mediante bytes en BCD comprimido con verificación de integridad CRC8^^.

| **Campo de la Trama**      | **Byte de Inicio** | **Dirección**        | **Código Función** | **Dirección Origen Datos** | **Longitud Datos (N)** | **Bloque de Datos**                   | **Suma de Comprobación** |
| -------------------------------- | ------------------------ | --------------------------- | -------------------------- | --------------------------------- | ---------------------------- | ------------------------------------------- | ------------------------------- |
| **Trama Maestro (Master)** | `0xA5`[cite: 6]        | `0x01`–`0x99`[cite: 6] | `0x03`/`0x10`[cite: 6] | `0xXX`[cite: 6]                 | `0xNN`[cite: 6]            | Data**$0 \dots \text{Data } N$**[cite: 6] | CRC8 (1 byte)^^                 |
| **Trama Esclavo (Slave)**  | `0xA5`[cite: 6]        | `0x01`–`0x99`[cite: 6] | `0x03`/`0x10`[cite: 6] | N/A (implícito)                  | `0xNN`[cite: 6]            | Data**$0 \dots \text{Data } N$**[cite: 6] | CRC8 (1 byte)^^                 |

Los códigos de función primarios empleados en este protocolo son:

* `0x03`: Función de Lectura de Registros y Consulta de Estado (Read Data / Poll Status)^^.
* `0x0C`: Función de Lectura de Eventos de Registro (Read Event)^^.
* `0x10`: Función de Escritura de Parámetros y Comandos (Write Data / Authorize / Set Price)^^.
* `0x83` / `0x90`: Trama de Excepción o Error devuelta por el surtidor ante un fallo de lectura o escritura respectivamente^^.

## Especificación de Comandos, Consultas y Control Operativo

### Consulta de Estado (Status Poll)

El sistema central envía una trama de sondeo con el código de función de lectura `0x03` dirigida a la dirección de origen de datos `0x00` para determinar la condición operativa actual del punto de suministro^^.

La trama transmitida por el maestro responde a la siguiente secuencia hexadecimal:

`0xA5` + `[Dirección]` + `0x03` + `0x00` + `0x01` + `[CRC8]`

[cite: 6]

El byte de respuesta retornado por el surtidor iGEM indica el estado en el que se encuentra la máquina de estados interna del equipo^^:

* `0xA5` + `[Dirección]` + `0x03` + `0x01` + `0x00` + `[CRC8]`: **Punto de Suministro Libre / En Reposo (Idle)**^^.
* `0xA5` + `[Dirección]` + `0x03` + `0x01` + `0x01` + `[CRC8]`: **Petición de Inicio con Predeterminación en Litros**^^.
* `0xA5` + `[Dirección]` + `0x03` + `0x01` + `0x02` + `[CRC8]`: **Petición de Inicio con Predeterminación en Importe**^^.
* `0xA5` + `[Dirección]` + `0x03` + `0x01` + `0x03` + `[CRC8]`: **Dispositivo Detenido / Requiere Parada (Stop Required)**^^.
* `0xA5` + `[Dirección]` + `0x03` + `0x01` + `0x04` + `[CRC8]`: **Surtidor en Despacho / Ocupado (Busy / Fueling)**^^.
* `0xA5` + `[Dirección]` + `0x03` + `0x01` + `0x05` + `[CRC8]`: **Manguera Colgada / Fin de Venta Pendiente de Cierre**^^.
* `0xA5` + `[Dirección]` + `0x03` + `0x01` + `0x06` + `[CRC8]`: **Estado Post-Reinicio de Cabezal Electronico**^^.

### Configuración y Consulta de Precio Unitario

El precio unitario del producto debe programarse o actualizarse mediante la función de escritura `0x10` sobre la dirección origen de datos `0x55`^^. La secuencia utiliza 3 bytes en formato BCD comprimido para representar el valor numérico^^.

Para fijar un precio de 236.85 por litro en la dirección `0x01`, el comando del maestro se estructura como sigue:

`0xA5` + `0x01` + `0x10` + `0x55` + `0x03` + `0x02` + `0x36` + `0x85` + `[CRC8]`

[cite: 6]

Si el procesamiento es correcto, el surtidor responde ratificando la dirección y longitud aceptada:

`0xA5` + `0x01` + `0x10` + `0x55` + `0x03` + `[CRC8]`

[cite: 6]

En caso de inconsistencia, el surtidor emite una trama de excepción:

`0xA5` + `0x01` + `0x90` + `[Código Error 0x01-0x07]` + `[CRC8]`

[cite: 6]

Para verificar el precio unitario activo en el cabezal, el maestro ejecuta una orden de lectura:

`0xA5` + `0x01` + `0x03` + `0x55` + `0x03` + `[CRC8]`

[cite: 6]

### Comandos de Autorización y Envió de Predeterminación

La autorización habilita el encendido del motor de la bomba y la apertura de las válvulas selenoides^^. Existen tres formatos para autorizar un despacho:

#### Autorización a Tanque Lleno (Sin Predeterminación)

Esta orden habilita el despacho libre hasta el límite de seguridad del sistema (9999.99 litros)^^.
`0xA5` + `[Dirección]` + `0x03` + `0x04` + `0x99` + `0x99` + `0x99` + `0x00` + `[CRC8]`^^

#### Autorización con Predeterminación de Volumen (Litros)

Para autorizar un volumen exacto (ejemplo: 123.45 litros), se codifica la cifra dentro de la trama de predeterminación en BCD^^.
`0xA5` + `[Dirección]` + `0x03` + `0x04` + `0x01` + `0x23` + `0x45` + `0x00` + `[CRC8]`^^

#### Autorización con Predeterminación de Importe (Venta Monetaria)

Para establecer un tope monetario de venta (ejemplo: 123.45 unidades monetarias), se emplea la dirección de origen `0x14` con un bloque de 4 bytes de datos^^.
`0xA5` + `[Dirección]` + `0x03` + `0x14` + `0x04` + `0x00` + `0x01` + `0x23` + `0x45` + `[CRC8]`^^

### Comando de Interrupción / Parada de Emergencia

Para forzar la detención inmediata del despacho desde el controlador central, se transmite una orden de escritura de estado de corte a la dirección de origen `0x5A`^^.

`0xA5` + `[Dirección]` + `0x10` + `0x5A` + `0x01` + `0x00` + `[CRC8]`

[cite: 6]

El surtidor confirma la recepción de la orden de detención respondiendo:

`0xA5` + `[Dirección]` + `0x10` + `0x5A` + `0x01` + `[CRC8]`

[cite: 6]

### Lectura de Datos de Transacción y Totalizadores

Durante el proceso de suministro o al finalizar la venta, el controlador solicita los valores acumulados enviando un comando de lectura a la dirección de origen `0x70` con una longitud solicitada de 12 bytes (`0x0C`)^^.

`0xA5` + `[Dirección]` + `0x03` + `0x70` + `0x0C` + `[CRC8]`

[cite: 6]

El surtidor responde con un paquete de 12 bytes conteniendo el volumen y el importe de la venta en BCD comprimido^^:
`0xA5` + `[Dirección]` + `0x03` + `0x0C` + `[Volumen 5 bytes]` + `[Importe 7 bytes]` + `[CRC8]`^^

En una transacción con un volumen de 58.37 litros y un importe de 123,456.78 unidades monetarias, el bloque de datos emitido por el surtidor toma la siguiente forma hexadecimal^^:
`0xA5` + `0x01` + `0x03` + `0x0C` + `0x00` + `0x00` + `0x00` + `0x58` + `0x37` + `0x00` + `0x00` + `0x00` + `0x12` + `0x34` + `0x56` + `0x78` + `[CRC8]`^^

| **Dirección Origen (Hex)** | **Función Asignada**       | **Longitud Payload** | **Descripción de la Operación Serie**                        |
| --------------------------------- | --------------------------------- | -------------------------- | -------------------------------------------------------------------- |
| `0x00`/`0x01`                 | `0x03`(Read)^^                  | 1 byte^^                   | Consulta de Estado Operativo del Surtidor (Idle, Busy, Stop, etc.)^^ |
| `0x04`                          | `0x03`(Read/Write)^^            | 4 bytes^^                  | Registro de Predeterminación por Volumen (Litros en BCD)^^          |
| `0x14`                          | `0x03`(Read/Write)^^            | 4 bytes^^                  | Registro de Predeterminación por Importe Monetario (BCD)^^          |
| `0x55`                          | `0x10`(Write) /`0x03`(Read)^^ | 3 bytes^^                  | Registro y Modificación del Precio Unitario por Litro^^             |
| `0x5A`                          | `0x10`(Write)^^                 | 1 byte^^                   | Comando de Control de Estado (Detención / Reanudación)^^           |
| `0x70`                          | `0x03`(Read)^^                  | 12 bytes (`0x0C`)^^      | Registro de Venta en Curso / Finalizada (Volumen e Importe)^^        |

## Algoritmo de Verificación CRC8 y Temporización de Línea

La comprobación de la integridad del mensaje se realiza mediante la adición de un byte de suma de verificación CRC8 al final de cada trama^^. Este byte se calcula sobre los elementos constitutivos de la trama a excepción del byte delimitador de cabecera `0xA5`^^.

La fórmula de cómputo del CRC8 para la trama emitida por el nodo maestro se define como:

$$
\text{CRC8} = f_{\text{CRC8}}(\text{Dirección} + \text{Código Función} + \text{Dirección Origen} + \text{Longitud Datos} + [\text{Datos}])
$$

language[cite: 6]

Para las tramas devueltas por el surtidor esclavo, el cálculo abarca la siguiente secuencia de bytes:

$$
\text{CRC8} = f_{\text{CRC8}}(\text{Dirección} + \text{Código Función} + \text{Longitud Datos} + [\text{Datos}])
$$

language[cite: 6]

En la gestión temporal del canal RS-485, el surtidor Wayne iGEM responde a las peticiones del maestro en un tiempo de latencia **$T_s \le 100\text{ ms}$**^^. El software del controlador de la estación de servicio debe configurar un tiempo de espera de respuesta (timeout) de al menos 150 ms antes de declarar una falla de comunicación^^. Adicionalmente, el maestro debe liberar la línea de transmisión (RTS) en un tiempo menor a 1 ms tras enviar el byte final de CRC8 para ceder el control del bus al surtidor^^.

## Conclusiones y Guía de Implementación

La integración del surtidor Wayne 3G3399P mediante comunicación serie RS-485 requiere coordinar adecuadamente la interfaz de red, los parámetros del computador de a bordo iGEM y la lógica del software de control^^.

El proceso de puesta en marcha debe iniciar verificando si la interfaz física del computador iGEM expone líneas RS-485 nativas o si requiere el acople de un módulo conversor de bucle de corriente a RS-485^^. Posteriormente, mediante el control remoto IRC, se debe acceder a la programación del cabezal para fijar la sub-función `F01.00 = 1` (Serial Mode), asignar las direcciones de los lados A y B en `F05.00` y `F06.00`, y seleccionar la función `F20.00 = 1` (Standard DART) con velocidad `F20.01 = 4` (9600 bps) y comprobación CRC habilitada en `F20.02 = 1`^^. Es indispensable comprobar la remoción del puente de modo autónomo en la tarjeta electrónica si se ejecutan versiones de firmware recientes^^.

Finalmente, el controlador de pista debe implementar una rutina de sondeo en bucle basada en el envio de comandos de estado (`0x03`), la actualización periódica de precios unitarios (`0x10` en registro `0x55`), la autorización de suministros con o sin predeterminación (`0x03` en registros `0x04` u `0x14`) y la lectura acumulativa de los registros de venta (`0x03` en registro `0x70`), asegurando la correcta validación del byte CRC8 en cada intercambio de información^^.
