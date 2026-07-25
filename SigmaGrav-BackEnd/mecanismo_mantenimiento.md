# Mecanismo de Mantenimiento y Archivado de Datos (db:archive)

Este documento detalla el funcionamiento interno del comando `php artisan db:archive` (implementado en `ArchiveOldData.php`) y la estructura jerárquica de las tablas que se procesan durante el mantenimiento de la base de datos de **SigmaGrav**.

---

## 1. Mecanismo de Funcionamiento

El proceso de mantenimiento y archivado se ejecuta en lotes (`batches`) y realiza las siguientes acciones de forma secuencial por cada modelo configurado en `config/archive.php`:

```mermaid
graph TD
    A[Inicio: db:archive] --> B[Calcular fecha umbral: Hoy - N meses]
    B --> C[Iterar sobre cada Modelo configurado]
    C --> D[Obtener lote de registros viejos con relaciones]
    D --> E{¿Existen registros?}
    E -- Sí --> F[Iniciar Transacciones en ambas DBs]
    E -- No --> G[Siguiente Modelo / Fin]
    F --> H[Desactivar triggers y FK en DB Archivo]
    H --> I[Replicar registro padre preservando timestamps]
    I --> J[Replicar y eliminar registros hijos/relacionados]
    J --> K[Eliminar físicamente registros padres en DB Principal]
    K --> L[Confirmar transacciones (Commit)]
    L --> D
```

### Detalles del Flujo:

1. **Cálculo del Umbral**: Se calcula la fecha límite según el parámetro `threshold_months` definido en `config/archive.php` (por defecto, 1 o 2 meses antes del día actual).
2. **Batching (Procesamiento por lotes)**: Los registros se obtienen en grupos pequeños (ej. 500 registros por consulta) para evitar el consumo excesivo de memoria del servidor.
3. **Manejo de SoftDeletes**: Si el modelo de origen implementa el trait `SoftDeletes` de Laravel, la consulta incluye los registros eliminados lógicamente (`withTrashed()`).
4. **Transaccionalidad Cruzada**: Para evitar inconsistencias o pérdidas de datos ante un fallo de red, se abren transacciones simultáneas en la base de datos de producción (`pgsql`) y la base de datos de archivo (`pgsql_archive`).
5. **Upsert Safety (ON CONFLICT)**: Para evitar violaciones de clave primaria, el comando comprueba si el `id` ya existe en la base de datos de archivo. Si existe, actualiza los valores existentes; si no, inserta un registro nuevo.
6. **Preservación de Tiempos Históricos**: En lugar de replicar el modelo normalmente (lo cual regeneraría timestamps), se copian los atributos puros mediante `setRawAttributes($record->getAttributes())` y se desactiva temporalmente el actualizador automático del modelo (`$model->timestamps = false;`). Esto garantiza que `created_at` y `updated_at` permanezcan idénticos a los originales.
7. **Eliminación Física Definitiva**: Una vez que un registro se inserta correctamente en la base de datos de archivo, se elimina físicamente de la base de datos principal (`forceDelete()` si tiene `SoftDeletes`, o `delete()` ordinario si no los tiene).

---

## 2. Jerarquía y Relaciones de Tablas Archivadas

Cuando se archiva un registro padre, **todas sus relaciones configuradas se migran y eliminan junto a él**. A continuación, se presenta la jerarquía detallada de dependencias:

### Jerarquías Complejas (Tablas Dependientes)

```
📂 Transacciones (transacciones) [Filtro: created_at]
 ├── 📄 Facturas (facturas)
 ├── 📄 Remitos (remitos)
 └── 📄 Traza de Productos (trazaproductos)
```
> [!NOTE]
> Al migrar una transacción vieja, todas las facturas, remitos y movimientos de stock asociados a esa transacción se mueven al archivo en el mismo paso.

```
📂 Pagos y Gastos (pagos_gastos) [Filtro: created_at]
 └── 📄 Detalles de Pagos (pagos)
```

---

### Tablas Independientes y Soporte de Registros Huérfanos

Para asegurar la integridad de la base de datos principal y evitar que queden registros antiguos abandonados (huérfanos), las tablas dependientes (`facturas`, `remitos` y `pagos_realizados`) también se configuran como modelos independientes al final del proceso de mantenimiento. 

De esta forma, si existe alguna fila que no esté relacionada a una transacción o gasto principal, el motor la procesará y archivará de manera independiente.

Las tablas independientes procesadas son:

*   **Facturas Huérfanas** (`facturas`) | Filtro: `created_at`
*   **Remitos Huérfanas** (`remitos`) | Filtro: `created_at`
*   **Detalles de Pagos Huérfanos** (`pagos_realizados`) | Filtro: `created_at`
*   **Cierres de Turno** (`cierresturnos`) | Filtro: `created_at`
*   **Cobros** (`cobros`) | Filtro: `created_at`
*   **Notas de Crédito** (`notacreditos`) | Filtro: `created_at`
*   **Notas de Débito** (`notadebitos`) | Filtro: `created_at`
*   **Fichadas** (`fichadas`) | Filtro: `fecha_registro`
*   **Cargas** (`cargas`) | Filtro: `fecha`
*   **Asistencias y Justificaciones** (`justificaciones`) | Filtro: `fecha`
*   **Movimientos de Fidelización** (`fidelizacion_movimientos`) | Filtro: `created_at`
*   **Purgas** (`purgas`) | Filtro: `created_at`
*   **Movimientos de Tanques** (`tanques_mov`) | Filtro: `created_at`
*   **Tiradas al Buzón** (`tiradasalbuzon`) | Filtro: `created_at`
*   **Adelantos de Sueldos** (`sueldos_adelantos`) | Filtro: `created_at`
*   **Liquidaciones de Sueldos** (`sueldos_liquidaciones`) | Filtro: `created_at`
*   **Vacaciones** (`vacaciones`) | Filtro: `created_at`
*   **Ejecuciones de Runners** (`runner_executions`) | Filtro: `created_at`

---

## 3. Configuración y Columnas de Fecha

La siguiente tabla resume la definición física del archivado para cada modelo según `config/archive.php`:

| Clase de Modelo / Tabla | Columna de Fecha Usada para Filtrado | Relaciones Archivadas |
| :--- | :--- | :--- |
| `App\Models\Transacciones` | `created_at` | `facturas`, `remitos`, `trazaproductos` |
| `App\Models\cierresturnos` | `created_at` | Ninguna |
| `App\Models\AsistenciaJustificacion` | `fecha` | Ninguna |
| `App\Models\cargas` | `fecha` | Ninguna |
| `App\Models\Cobros` | `created_at` | Ninguna |
| `App\Models\Fichada` | `fecha_registro` | Ninguna |
| `App\Models\fidelizacion_movimientos` | `created_at` | Ninguna |
| `App\Models\NotaCredito` | `created_at` | Ninguna |
| `App\Models\NotaDebito` | `created_at` | Ninguna |
| `App\Models\PagosGastos` | `created_at` | `pagos` |
| `App\Models\purgas` | `created_at` | Ninguna |
| `App\Models\RunnerExecution` | `created_at` | Ninguna |
| `App\Models\SueldosAdelantos` | `created_at` | Ninguna |
| `App\Models\SueldosLiquidaciones` | `created_at` | Ninguna |
| `App\Models\tanques_mov` | `created_at` | Ninguna |
| `App\Models\tiradasalbuzon` | `created_at` | Ninguna |
| `App\Models\trazaproductos` | `created_at` | Ninguna |
| `App\Models\Vacacion` | `created_at` | Ninguna |
| `App\Models\facturas` | `created_at` | Ninguna |
| `App\Models\remitos` | `created_at` | Ninguna |
| `App\Models\PagosRealizados` | `created_at` | Ninguna |
