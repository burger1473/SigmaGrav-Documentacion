# Configuración de Fidelización (`fidelizacion_configs`)

Este documento especifica la estructura de la tabla `fidelizacion_configs`, los tipos de reglas soportados por el backend (**SigmaGrav-BackEnd**) y el algoritmo necesario para armar y encriptar **Tickets Personalizados / Custom** en el sistema **SigmaGrav-PosApp**.

---

## 1. Estructura de la Tabla `fidelizacion_configs`

La tabla `fidelizacion_configs` almacena parámetros dinámicos de fidelización, timeouts, reglas antifraude y plantillas de impresión para la aplicación POS.

| Campo | Tipo | Nullable | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `bigint` | No | ID autoincremental primario. |
| `tipo` | `string` | No | Clasificador principal del tipo de regla (`PUNTOS`, `DESCUENTO`, `TIMEOUT`, `ANTIFRAUDE`, `TICKET`). |
| `valor` | `string` | No | Clave o identificador de la regla (ej. combustible `SUPER`, parámetro `CUSTOM`, `ESTADO`, `VIEJO`). |
| `valor2` | `string` | Sí | Valor primario de configuración (ej. multiplicador, timeout en minutos, estado activo `'1'`/`'0'`). |
| `valor3` | `string` | Sí | Atributos adicionales o nombre legible (ej. nombre del ticket custom). |
| `valor4` | `text` | Sí | Contenido extenso o trama encriptada (ej. plantilla del ticket custom encriptada). |
| `valor5` | `string` | Sí | Flags secundarios o comportamiento de impresión (ej. `'1'` para reimprimibles, `'0'` para 1° tirada). |
| `created_at` | `timestamp` | Sí | Fecha de creación del registro. |
| `updated_at` | `timestamp` | Sí | Fecha de última actualización. |

---

## 2. Tipos de Configuración Soportados

### 2.1. `PUNTOS`
Define la tasa de acumulación de puntos por cada litro cargado según el tipo de combustible.
* **`valor`**: Nombre del combustible (ej. `'SUPER'`, `'PREMIUM'`, `'DIESEL'`, `'OTROS'`).
* **`valor2`**: Factor multiplicador de puntos por litro (ej. `'1.5'`).

### 2.2. `DESCUENTO`
Establece reglas de descuentos promocionales aplicables en el cobro.
* **`valor`**: Nombre del beneficio/promoción.
* **`valor2`**: Porcentaje o monto fijo de descuento.
* **`valor3`**: Condiciones adicionales de aplicación.

### 2.3. `TIMEOUT`
Determina las ventanas de tiempo válidas para procesar despachos de combustible.
* **`valor = 'VIEJO'`** (`valor2`): Límite máximo de antigüedad en minutos para procesar una carga pasada.
* **`valor = 'FUTURO'`** (`valor2`): Tolerancia máxima en minutos aceptada para desfasajes de reloj en cargas futuras.

### 2.4. `ANTIFRAUDE`
Controla las reglas de bloqueo para prevención de fraude en cargas repetidas.
* **`valor = 'ESTADO'`** (`valor2`): Estado de activación (`'ACTIVO'` u `'INACTIVO'`).
* **`valor = 'TIEMPO'`** (`valor2`): Tiempo mínimo en minutos requerido entre despachos sucesivos.

---

## 3. Especificación de Tickets Custom (`tipo = 'TICKET'`)

Los tickets o talones personalizados se imprimen dinámicamente en el POS tras finalizar una transacción (ej. cupones de sorteos, promociones, vales).

### 3.1. Mapeo de Campos para Tickets Custom

Para que un ticket custom sea reconocido e impreso correctamente por `PosAppController`, la fila en `fidelizacion_configs` debe armarse de la siguiente manera:

| Campo | Valor Requerido | Descripción |
| :--- | :--- | :--- |
| `tipo` | `'TICKET'` | Identificador obligatorio del tipo. |
| `valor` | `'CUSTOM'` | Especifica que se trata de un talón de ticket personalizado. |
| `valor2` | `'1'` o `'0'` | **Estado**: `'1'` = Activo e imprimible, `'0'` = Desactivado. |
| `valor3` | `string` | Nombre descriptivo del ticket (ej. `'CUPÓN PROMO'` o `'TALÓN SORTEO'`). |
| `valor4` | `string (Encriptado)` | Contenido de la plantilla codificado/encriptado con **AES-256-CBC**. |
| `valor5` | `'1'`, `'0'` (o `null`) | **Comportamiento de Reimpresión**: <br/>• `'1'` o `null` = Reimprimible en cada impresión de copia. <br/>• `'0'` = Solo se imprime en la **primera tirada** (no reimprimible en duplicados). |

---

## 4. Algoritmo y Encriptación de la Plantilla (`valor4`)

Por motivos de seguridad e integridad, la plantilla almacenada en `valor4` **debe estar encriptada** mediante el servicio `EncryptDecryptService` utilizando la clave del sistema (`CRYPTO_SECRET_TICKET_CUSTOM`).

### 4.1. Esquema de Encriptación
1. **Algoritmo**: `AES-256-CBC`
2. **Clave**: `hash('sha256', env('CRYPTO_SECRET_TICKET_CUSTOM', '9b3f7e81d4c2a5e0f81d92c3b4a5e6f7d8c9b0a1e2f3a4b5c6d7e8f9a0b1c2d3'))`
3. **Vector de Inicialización (IV)**: 16 bytes aleatorios.
4. **Formato final de almacenamiento (`valor4`)**:
   $$\text{base64\_encode}(\text{texto\_encriptado} \mathbin{\Vert} \text{"::"} \mathbin{\Vert} \text{base64\_encode}(IV))$$

---

## 5. Ejemplos de Implementación

### 5.1. Armar e Insertar Ticket Custom desde PHP / Laravel

```php
use App\Models\fidelizacion_config;
use App\Services\EncryptDecryptService;

// 1. Obtener instancia del servicio de encriptación
$encryptService = app(EncryptDecryptService::class);

// 2. Definir la plantilla del ticket en formato texto / etiquetas de impresión
$plantilla = "{center}{bold}¡GRACIAS POR SU COMPRA!{/bold}{/center}\n"
    . "{center}Conserve este ticket para participar del sorteo.{/center}\n\n"
    . "Cliente: {cliente_nombre}\n"
    . "Puntos Acumulados: {puntos}\n"
    . "Carga: {volumen} Lts\n";

// 3. Encriptar la plantilla
$plantillaEncriptada = $encryptService->encriptarTicketCustom($plantilla);

// 4. Guardar la configuración en fidelizacion_configs
fidelizacion_config::create([
    'tipo'   => 'TICKET',
    'valor'  => 'CUSTOM',
    'valor2' => '1',                       // 1 = Activo
    'valor3' => 'Cupón Promocional',       // Nombre legible
    'valor4' => $plantillaEncriptada,     // Plantilla encriptada
    'valor5' => '1',                       // 1 = Reimprimible
]);
```

### 5.2. Insertar mediante Consulta SQL Directa

Si se requiere realizar la inserción manualmente mediante una sentencia SQL, primero se debe obtener la cadena cifrada y luego ejecutar:

```sql
INSERT INTO fidelizacion_configs (
    tipo, 
    valor, 
    valor2, 
    valor3, 
    valor4, 
    valor5, 
    created_at, 
    updated_at
) VALUES (
    'TICKET',
    'CUSTOM',
    '1',
    'Talón Sorteo Aniversario',
    'WTIyT3ZkT29P...cadena_encriptada_base64...',
    '0',
    NOW(),
    NOW()
);
```

---

## 6. Procesamiento en el POS (`PosAppController`)

Durante la emisión de comprobantes, `PosAppController` realiza las siguientes validaciones:
1. Filtra los registros donde `tipo = 'TICKET'`, `valor = 'CUSTOM'`, `valor2 = '1'` y `valor4` no nulo.
2. Desencripta `valor4` usando `desencriptarTicketCustom()`. Si la desencriptación falla o resulta vacía, la plantilla se descarta automáticamente.
3. Evalúa `valor5`:
   * Si es `'0'` o `'false'`, clasifica el ticket como **no reimprimible** (se excluye si se trata de una reimpresión o copia).
   * Si es `'1'`, `'true'` o `null`, imprime el talón custom normalmente.
