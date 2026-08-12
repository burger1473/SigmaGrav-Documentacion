# Protocolo de Comunicación y Especificaciones Técnicas para Surtidores Gilbarco Advantage

## Introducción y Arquitectura de Comunicación Forecourt

Los surtidores y dispensadores de combustible Gilbarco de la serie Advantage representan una de las arquitecturas electrónicas de mayor permanencia e instalación en estaciones de servicio a nivel global y regional^^. En mercados latinoamericanos, empresas especializadas como ARPEC S.A. han importado y distribuido ampliamente estos equipos, integrándolos con diversos controladores de pista (Forecourt Controllers) y sistemas de punto de venta (POS)^^. Independientemente del canal comercial o del distribuidor local que comercialice el equipamiento, la electrónica de la serie Advantage responde a estándares globales unificados desarrollados por Gilbarco Veeder-Root^^.

Para comandar y consultar un surtidor Gilbarco Advantage de forma remota, es fundamental diferenciar entre el **estándar físico de transmisión** y el **protocolo lógico de aplicación**^^. Nativamente, el cabezal electrónico del Advantage no implementa un puerto directo EIA/TIA-485 (RS-485), sino una interfaz propietaria de lazo de corriente de dos hilos ( **2-Wire Current Loop** ) de 45 mA^^. El protocolo lógico que corre sobre esta capa física se denomina formalmente **Gilbarco Two-Wire Protocol**^^.

Cuando un sistema de control requiere comunicarse con el surtidor utilizando el estándar físico RS-485, la arquitectura del sistema debe incorporar un módulo de conversión de interfaz (convertidor de lazo de corriente a RS-485) o una tarjeta de distribución equipada con transceptores diferencial de línea^^. Una vez solventada la adaptación eléctrica, el intercambio de tramas de control, predeterminaciones, cambios de precio, consultas de estado y lecturas de aforadores o registros de ventas sigue de forma rigurosa la especificación lógica del protocolo Gilbarco Two-Wire^^.

## Capa Física e Interconexión de Hardware: Lazo de Corriente y Adaptación a RS-485

La capa física original de los surtidores Gilbarco Advantage fue concebida para brindar una alta inmunidad al ruido electromagnético en entornos industriales agresivos, donde conviven motores trifásicos, bombas sumergibles y conmutadores de potencia^^. El esquema de lazo de corriente de 45 mA utiliza variaciones de corriente continua para representar los niveles lógicos binarios, lo que permite tiradas de cableado extensas hacia la consola de control^^.

| **Parámetro Eléctrico / Físico** | **Especificación Nativa Gilbarco (2-Wire)**              | **Adaptación para Bus RS-485**                                                          |
| ----------------------------------------- | --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **Tipo de Señal**                  | Lazo de corriente activo de 45 mA (12 VDC)^^                    | Tensión diferencial EIA/TIA-485 (**$\pm 1.5\text{ V}$**a**$\pm 5\text{ V}$**) |
| **Topología de Red**               | Punto a punto canalizado desde Caja de Distribución^^          | Bus multipunto semi-dúplex (Half-Duplex 2 hilos)^^                                            |
| **Tipo de Cable Requerido**         | Par trenzado sin blindaje (UTP) 14 a 18 AWG^^                   | Par trenzado blindado (STP) con impedancia de**$120\ \Omega$**                               |
| **Distancia Máxima de Cableado**   | Hasta 792 metros (2600 pies)^^                                  | Hasta 1200 metros (dependiendo del baud rate)                                                  |
| **Aislamiento Eléctrico**          | Aislamiento óptico integrado en tarjeta CPU/D-Box^^            | Requiere aislamiento galvánico en transceptores^^                                             |
| **Capacidad por Canal**             | 1 surtidor o posición física por canal de lazo de corriente^^ | Hasta 16 posiciones de carga lógicas por bus RS-485^^                                         |

Para interconectar un controlador industrial, PC o PLC basado en RS-485 con la serie Advantage, se utilizan comúnmente tres alternativas de hardware:

1. **Cajas de Distribución Oficiales (Gilbarco D-Box PA0242 / PA0306 / Universal D-Box)** : Módulos pasivos y activos que acondicionan los bucles individuales de cada surtidor y entregan interfaces seriales de comunicación hacia la consola central^^.
2. **Convertidores de Interfaz de Terceros (ej. Technotrade GB-4, Levtech LSP-FCG)** : Placas electrónicas que realizan la traducción bidireccional entre los niveles de tensión diferencial del bus RS-485 (líneas A y B) y el lazo de corriente de 45 mA de 2 hilos^^. Estos dispositivos permiten ajustar la corriente del bucle (30 mA, 45 mA o 60 mA) mediante puentes de configuración (jumpers)^^.
3. **Módulos Adaptadores Integrados (ej. Invenco PIB RS485, Verifone Current Loop Kit)** : Tarjetas de expansión internas para controladores de pista que exponen interfaces RS-485 directas o convertidores hacia puertos serie virtuales USB^^.

## Configuración de Capa de Enlace y Enmarcado Serial

Al establecer la comunicación transparente a través del convertidor RS-485, la trama de bytes debe respetar la configuración del puerto serie de la tarjeta principal del surtidor Gilbarco Advantage^^.

| **Parámetro Serial**                     | **Estándar América / Advantage**                 | **Estándar Corporativo / Internacional**          |
| ----------------------------------------------- | -------------------------------------------------------- | -------------------------------------------------------- |
| **Velocidad de Transmisión (Baud Rate)** | 4800 bps^^                                               | 5787 bps (Baud rate propietario Gilbarco)^^              |
| **Bits de Datos**                         | 8 bits^^                                                 | 8 bits^^                                                 |
| **Paridad**                               | Par (Even Parity)^^                                      | Par (Even Parity)^^                                      |
| **Bits de Parada (Stop Bits)**            | 1 bit^^                                                  | 1 bit^^                                                  |
| **Modo de Transmisión**                  | Semi-dúplex (Half-Duplex)^^                             | Semi-dúplex (Half-Duplex)^^                             |
| **Estructura Maestro-Esclavo**            | El controlador es Maestro; los surtidores son Esclavos^^ | El controlador es Maestro; los surtidores son Esclavos^^ |

### Esquema de Direccionamiento Multipunto

El protocolo Gilbarco Two-Wire es un protocolo direccionable^^. Permite gestionar hasta 16 posiciones de carga lógicas (Fueling Positions) dentro de una misma línea de datos^^. En surtidores de doble cara (como los modelos Advantage MPD o de mangueras múltiples), cada cara posee una dirección única asignada en el equipo^^.

Las direcciones se codifican en caracteres ASCII según el siguiente esquema^^:

* Posiciones de carga 1 a 15: Caracteres ASCII `'1'`, `'2'`, `'3'`, `'4'`, `'5'`, `'6'`, `'7'`, `'8'`, `'9'`, `'A'`, `'B'`, `'C'`, `'D'`, `'E'`, `'F'`^^.
* Posición de carga 16: Carácter ASCII `'0'`^^.

El controlador maestro interroga cíclicamente a cada posición mediante comandos de sondeo (Status Poll)^^. Un surtidor únicamente responde cuando detecta un paquete dirigido a su dirección asignada^^.

## Estructura de Mensajes y Formatos de Trama

El protocolo Gilbarco Two-Wire clasifica los mensajes en dos categorías principales: **Comandos Simples (Línea Corta)** y **Tramas de Datos Extendidas (Línea Larga)**^^.

### Comandos Simples (Sin Control de Error LRC)

Los comandos lógicos de ciclo de vida rápido y alto tráfico —como las consultas periódicas de estado, las autorizaciones inmediatas o las paradas de emergencia— utilizan tramas de longitud fija compuestas por dos caracteres ASCII sin código de redundancia longitudinal (LRC)^^.

La estructura de un comando simple hacia el surtidor es la siguiente^^:

* **Byte 1** : Código de Comando ASCII (ejemplo: `'0'` para Poll, `'1'` para Authorize, `'3'` para Pump Stop)^^.
* **Byte 2** : Dirección ASCII de la posición de carga (ejemplo: `'1'` a `'F'`, `'0'`)^^.

Ejemplo: La cadena `'01'` representa un sondeo de estado (Poll) enviado a la posición de carga número 1^^.

### Tramas de Datos Extendidas (Con Control de Error LRC)

Cuando el controlador necesita transferir o solicitar bloques de datos estructurados —tales como la fijación de predeterminaciones, cambio de precios unitarios (PPU), consulta de aforadores totalizadores o descarga del registro de venta de la última carga— se utiliza una trama extendida delimitada por caracteres de control estándar^^.

La secuencia de bytes de una trama extendida enviada al surtidor comprende^^:

$$
\text{Trama Extendida} = [\text{STX}] + [\text{DL}] + [\text{Dirección}] + [\text{Comando}] + [\text{Payload de Datos}] + [\text{ETX}] + [\text{LRC}]
$$

language* **STX (Start of Text)** : Carácter de control ASCII de inicio de trama (Hex `0x02`)^^.
* **DL (Data Length)** : Byte que define la cantidad de bytes incluidos en el bloque de datos o el tipo de formato^^.
* **Dirección** : Carácter ASCII de la posición de carga destinataria (`'1'`–`'F'`, `'0'`)^^.
* **Código de Comando / DCW** : Byte de comando funcional o Data Control Word (ej. `'2'` para Preset, `'4'` para Venta, `'5'` para Aforadores, `'6'` para Cambio de Precio)^^.
* **Payload de Datos** : Campo de longitud variable que transmite o recibe valores numéricos (monto, litros, precios PPU, acumulación de aforadores) codificados en ASCII o BCD^^.
* **ETX (End of Text)** : Carácter de control ASCII de fin de texto (Hex `0x03`)^^.
* **LRC (Longitudinal Redundancy Check)** : Byte de verificación de errores por redundancia longitudinal^^.

### Algoritmo de Cálculo de la Suma de Comprobación LRC

El byte **LRC** garantiza la integridad de los datos frente a interferencias electromagnéticas en la línea^^. Se calcula realizando una suma OR Exclusiva (XOR) acumulativa sobre todos los bytes transmitidos a partir del byte inmediatamente posterior a STX y finalizando en el carácter ETX inclusive^^.

La expresión matemática del cálculo del byte LRC se define como:

$$
\text{LRC} = \bigoplus_{i=\text{Byte después de STX}}^{\text{ETX}} \text{Byte}_i
$$

languageEn el caso de que la CPU del surtidor reciba una trama extendida cuyo cálculo interno de LRC no coincida con el byte enviado al final del mensaje, el comando es rechazado y el equipo mantendrá su estado operativo previo^^.

## Catálogo Completo de Comandos del Protocolo Gilbarco Two-Wire

La interacción con los surtidores Gilbarco Advantage se realiza mediante un conjunto de comandos estandarizados por la especificación del protocolo^^.

| Código de Comando | Nombre del Comando | Tipo de Trama | Descripción y Comportamiento Operativo |
| :--- :--- | :--- | :--- | :--- |
| **`0`** | **Status Poll** | Simple | Interrogación periódica de estado. El surtidor responde con su estado actual (Inactivo, Llamada, Despachando, Finalizado)^^. |
| **`1`** | **Authorize** | Simple | Orden de autorización. Permite habilitar el bombeo cuando el surtidor se encuentra en estado de llamada (manguera elevada)^^. |
| **`2`** | **Preset Data** | Extendida (LRC) | Transmite la predeterminación de volumen o dinero, junto con el grado de producto y nivel de precio seleccionado^^. |
| **`3`** | **Pump Stop** | Simple | Parada de la posición de carga. Detiene el flujo de combustible en curso o cancela una autorización pendiente^^. |
| **`4`** | **Request Transaction Data** | Extendida (LRC) | Consulta y lectura del registro de la última carga efectuada (volumen, importe total, precio unitario PPU y manguera)^^. |
| **`5`** | **Request Pump Totals** | Extendida (LRC) | Consulta y lectura de los aforadores electrónicos acumulatvos (totales volumétricos y monetarios por manguera)^^. |
| **`6`** | **Change Price Per Unit (PPU)** | Extendida (LRC) | Comando de modificación remota de precios unitarios por producto y nivel de precio en la electrónica del surtidor^^. |
| **`F C`** | **All Stop (Broadcast)** | Broadcast | Comando global de parada de emergencia para todas las posiciones de carga de la red^^. |

## Detalle Técnico de Comandos Operativos de Control y Consulta

### 1. Comando de Predeterminación (Preset Data - `'2'`) y Función FC8

Permite definir un límite de corte para el despacho antes de autorizar la carga^^. La trama incluye bytes que identifican si el Preset es por importe monetario o por volumen, el nivel de precio a aplicar (Nivel 1 o Nivel 2) y el grado de producto (Grade)^^.

Por especificación estándar de Gilbarco, la tarjeta CPU del surtidor únicamente acepta **un mensaje de Preset por transacción**^^. Si se envía un segundo Preset sin haber iniciado el despacho, el surtidor lo ignora a menos que se emita previamente un comando de parada (`Pump Stop - '3'`) para cancelar la predeterminación anterior^^.

Mediante la programación de la computadora del surtidor (código de función de campo FC8), es posible modificar este comportamiento^^:

* **FC8 = 1 (Default)** : Permite solo un Preset. Cualesquiera Presets secundarios requieren un `Pump Stop` previo^^.
* **FC8 = 2 (Allow Any Preset)** : Sobrescribe la predeterminación previa con el nuevo valor enviado, siempre que el surtidor se encuentre en estado inactivo, autorizado o de llamada, y antes de que se detecte flujo de producto^^.

### 2. Comando de Consulta de Últimas Cargas / Transacciones (Request Transaction Data - `'4'`)

Para obtener la información detallada de la última carga efectuada (o la venta en curso una vez finalizada), el controlador maestro envía la solicitud mediante el comando extendido `'4'`^^.

#### Estructura de la Petición del Maestro:

[STX] [DL] [Dirección] ['4'] [ETX] [LRC]

* **DL** : Byte de longitud que especifica el formato de lectura solicitado^^.
* **Dirección** : Carácter ASCII de la posición de carga (ej. `'1'`)^^.
* **Comando** : Byte ASCII `'4'`^^.

#### Estructura de la Respuesta del Surtidor Advantage:

El surtidor responde con una trama extendida conteniendo el desglose completo de la transacción finalizada^^:
[STX] [DL] [Dirección] [Código Estado] [Datos Venta] [ETX] [LRC]
El payload de **[Datos Venta]** entrega los siguientes valores codificados en cadena de dígitos BCD/ASCII:

1. **Importe Total de la Venta (Total Sale Amount)** : Campo numérico de 6 u 8 dígitos (ejemplo: `001500` para $150.00 o según los decimales configurados)^^.
2. **Volumen Despachado (Volume Delivered)** : Campo numérico de 6 u 8 dígitos que representa los litros o galones entregados^^.
3. **Precio Unitario (PPU - Price Per Unit)** : Valor del precio por litro/galón aplicado durante la transacción^^.
4. **Identificador de Grado / Manguera (Grade / Hose ID)** : Código que especifica qué manguera física o producto realizó el despacho^^.

*Nota de integración:* Una vez recuperada esta información por el controlador de pista, el envío del siguiente sondeo de estado (`Status Poll - '0'`) confirma la recepción al surtidor, permitiéndole borrar el registro de pantalla y retornar al estado inactivo reposo^^.

### 3. Comando de Lectura de Aforadores Totales / Totalizadores (Request Pump Totals - `'5'`)

Los aforadores electrónicos acumulativos representan la memoria histórica inalterable del surtidor y registran el acumulado total del volumen despachado y el monto monetario facturado por cada manguera o producto^^. Para consultar estos valores de auditoría, el controlador utiliza el comando extendido `'5'`^^.

#### Estructura de la Petición del Maestro:

[STX] [DL] [Dirección] ['5'] [Identificador Manguera/Grado] [ETX] [LRC]

* **Comando** : Byte ASCII `'5'`^^.
* **Identificador Manguera/Grado** : Byte opcional o numérico ASCII que especifica qué manguera o producto se desea auditar (ejemplo: `'1'` para Grado 1, `'2'` para Grado 2, o `'0'` para consulta secuencial de todas las mangueras)^^.

#### Estructura de la Respuesta del Surtidor:

La CPU del Advantage retorna la lectura de los acumuladores en el payload de la trama extendida^^:
[STX] [DL] [Dirección] ['5'] [Aforador Monetario] [Aforador Volumétrico] [ETX] [LRC]

* **Aforador Monetario Total (Total Money Counter)** : Cadena acumulativa de hasta 10 dígitos que registra el total de dinero despachado históricamente por esa manguera^^.
* **Aforador Volumétrico Total (Total Volume Counter)** : Cadena acumulativa de hasta 10 dígitos que registra el total de litros/galones despachados históricamente^^.

*Factor de Escala y Protocolo de 6 u 8 Dígitos:* Dependiendo del bit de configuración de la tarjeta CPU del surtidor Advantage (Flag de Protocolo de 6 u 8 dígitos), los valores transmitidos por los aforadores pueden requerir un multiplicador por 10 o por 100 para obtener la lectura exacta con decimales^^. Es una regla fundamental en los drivers de integración verificar la resolución del totalizador para evitar discrepancias en las lecturas de auditoría diaria^^.

### 4. Comando de Cambio y Actualización de Precios Unitarios (Change PPU - `'6'`)

El cambio remoto de precios unitarios (Price Per Unit) evita tener que programar manualmente el teclado del surtidor Advantage en la pista^^. Se efectúa enviando una trama de datos extendida con el código de comando `'6'`^^.

#### Estructura de la Trama de Cambio de Precio:

[STX] [DL] [Dirección] ['6'] [Grado/Manguera] [Nivel de Precio] [Nuevo Precio PPU] [ETX] [LRC]
Donde los campos del payload se desglosan de la siguiente manera^^:

* **Comando** : Byte ASCII `'6'` (o identificador de bloque de PPU)^^.
* **Grado / Manguera (Grade)** : Carácter ASCII que indica el producto a modificar (`'1'` para Nafta Súper, `'2'` para Premium, `'3'` para Diesel, etc.)^^.
* **Nivel de Precio (Price Level)** : Byte ASCII que especifica el nivel tarifario:
* `'4'`: Precio Nivel 1 (Crédito / Contado Estándar)^^.
* `'5'`: Precio Nivel 2 (Débito / Descuento / Lista Alternativa)^^.
* **Nuevo Precio PPU** : Cadena de 4 o 5 dígitos numéricos ASCII que representan el nuevo precio unitario por litro sin punto decimal explícito (ejemplo: la cadena `"04500"` se interpreta como $450.00 o $45.00 según la configuración de los displays PPU de la carátula)^^.

#### Momento de Aplicación y Comportamiento Operativo (FC 83.8 / Price Change Option)

El momento exacto en que el nuevo precio se refleja en los displays de PPU del surtidor depende de la configuración interna del firmware (Función de Campo FC 83.8 en modelos Gilbarco)^^:

1. **Opción 1 (Inmediata)** : El precio sube a los displays inmediatamente al procesar la trama, siempre que la manguera esté colgada (surtidor inactivo)^^.
2. **Opción 2 (Post-Despacho)** : Si el surtidor se encuentra con la manguera descolgada o despachando, el nuevo precio queda en cola y se aplica automáticamente una vez que el cliente cuelga la manguera y se asienta el microinterruptor^^.
3. **Opción 3 (Inmediata con Borrado)** : Aplica el cambio de precio de inmediato y resetea las pantallas de venta a cero^^.

## Ciclo de Vida y Máquina de Estados del Surtidor Advantage

El control de un surtidor Gilbarco Advantage exige estructurar el software controlador respetando la máquina de estados finitos que ejecuta la CPU del equipo^^. Las transiciones de estado se derivan del análisis continuo de las respuestas que retorna el surtidor ante cada comando de sondeo (`Status Poll - '0'`)^^.

El ciclo operativo completo se desarrolla a través de las siguientes fases lógicas:

### 1. Estado Inactivo / Reposo (OFF / IDLE)

El surtidor se encuentra con todas las mangueras asentadas en sus soportes y los microinterruptores colgados^^. El controlador maestro envía cíclicamente el comando Status Poll (`'0'`) a la dirección de la posición de carga^^. El surtidor responde con el código de estado indicando condición inactiva. En esta fase es idóneo enviar cambios de precio (`'6'`) o consultar aforadores totalizadores (`'5'`)^^.

### 2. Estado de Llamada (CALL)

Un cliente o playero descuelga la manguera en la cara del surtidor^^. El microinterruptor de la palanca envía una señal a la CPU del Advantage^^. En la siguiente respuesta al Status Poll, el surtidor cambia su respuesta al código de "Llamada" (CALL) e informa el número de la manguera/grado seleccionado por el usuario^^.

### 3. Fase de Configuración y Autorización (PRESET & AUTHORIZATION)

Una vez detectado el estado de CALL, el controlador evalúa la modalidad de venta^^:

* **Venta Libre** : El controlador envía directamente el comando simple Authorize (`'1'`)^^.
* **Venta Predeterminada** : El controlador transmite primero la trama extendida Preset Data (`'2'`) detallando el valor numérico (monto o litros), nivel de precio y manguera^^. Tras recibir la confirmación de la trama por parte del surtidor, el controlador emite el comando Authorize (`'1'`)^^.

Al ser autorizado, el surtidor resetea sus mostradores a cero, habilita el arranque del motor de bombeo y abre la válvula solenoide de bajo flujo.

### 4. Estado de Despacho (DISPENSING)

Una vez que el emisor de pulsos (pulser) detecta paso de fluido, el surtidor entra en el estado de despacho^^. La respuesta ante el Status Poll cambia al código de "Pumping / Dispensing"^^. Durante esta etapa, el controlador mantiene el sondeo periódico para detectar posibles comandos de parada de emergencia (`Pump Stop - '3'`) o cortes involuntarios^^.

### 5. Estado de Venta Finalizada (TRANSACTION COMPLETE)

El despacho concluye cuando se alcanza el volumen/monto prefijado en el Preset o cuando el usuario cuelga la manguera cortando la señal del microinterruptor^^. El surtidor detiene el motor, cierra las válvulas y conmuta su respuesta de estado a "Transacción Completada / Despacho Finalizado"^^. Los mostradores principales permanecen congelados mostrando el volumen y el importe de la venta^^.

### 6. Cierre de Transacción y Lectura de Datos (READOUT & CLEARING)

El controlador maestro emite el comando Request Transaction Data (`'4'`) para descargar el detalle final de la venta (litros, importe, PPU y grado)^^. Una vez que el sistema de control ha registrado e impreso el ticket correspondiente y la manguera ha sido asentada correctamente, el surtidor borra el estado de venta terminada y retorna a la condición inactiva (`OFF / IDLE`), quedando disponible para el siguiente ciclo^^.

## Consideraciones de Integración para Controladores Industriales

Al desarrollar un controlador de pista o implementar un driver de software sobre sistemas basados en microcontroladores, PCs industriales o PLCs para comunicarse con surtidores Gilbarco Advantage a través de convertidores RS-485, es necesario atender las siguientes directivas de diseño:

### Gestión del Tiempo y Tiempos de Respuesta (Timing Specifications)

1. **Intervalos de Sondeo (Polling Rate)** : Se recomienda mantener un tiempo inter-petición de entre 100 ms y 250 ms por cada posición de carga^^. Emitir peticiones a frecuencias excesivamente elevadas (menores a 20 ms) puede saturar la cola de interrupciones de la placa CPU del surtidor Advantage, provocando caídas temporales de la línea de comunicación^^.
2. **Tiempos de Conmutación de Bus en RS-485** : Al trabajar sobre una topología RS-485 semi-dúplex, el software controlador debe habilitar la línea RTS (Request to Send) inmediatamente antes de transmitir la trama y deshabilitarla tan pronto como se haya completado el envío del último byte (o byte de verificación LRC)^^. Esto garantiza que el puerto quede libre para recibir la respuesta del surtidor, la cual suele iniciarse pocos milisegundos después de recibir la petición^^.

### Manejo de Excepciones y Reintentos

1. **Detección de Caídas de Línea** : Si un surtidor no responde tras tres comandos Status Poll consecutivos, la posición de carga debe marcarse temporalmente como fuera de línea (OFFLINE)^^. El controlador debe continuar sondeando la dirección a un intervalo de reintento mayor hasta restablecer el enlace^^.
2. **Rechazo por Error de LRC** : Si la respuesta recibida desde el surtidor presenta una falla en el byte de comprobación LRC, el controlador debe desechar la trama y reemitir la solicitud de datos^^.

## Conclusiones y Recomendaciones de Implementación

El comando y monitoreo de surtidores Gilbarco Advantage mediante buses de comunicación RS-485 es una solución técnica viable, robusta y ampliamente utilizada en la automatización de estaciones de servicio^^. Aunque el equipamiento adquirido a través de distribuidores locales como ARPEC S.A. responde a las necesidades específicas de la región, la arquitectura de comunicación se rige estrictamente por las especificaciones globales del **Gilbarco Two-Wire Protocol**^^.

Para asegurar una integración exitosa, los proyectos de software y automatización deben cumplir con tres pilares fundamentales:

1. **Acondicionamiento Físico Adecuado** : Emplear convertidores de interfaz de grado industrial (RS-485 a Current Loop de 2 hilos) provistos de aislamiento galvánico y ajuste de la corriente del bucle a 45 mA^^.
2. **Enmarcado Serial Exacto** : Configurar los puertos de comunicación a 4800 bps (o 5787 bps según la versión firmware), con formato de 8 bits de datos, paridad par y 1 bit de parada (8-E-1)^^.
3. **Máquina de Estados Rigurosa** : Implementar un controlador que gestione el ciclo de vida de la transacción respetando las etapas de sondeo, predeterminación, autorización, despacho, actualización de precios y lectura de registros de última venta y aforadores acumulativos, garantizando el cálculo preciso del byte de control LRC en cada trama de datos extendida^^.
