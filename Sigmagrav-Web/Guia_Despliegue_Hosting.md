# Guía de Despliegue de SigmaGrav-Web en Hostinger / cPanel / VPS

Esta guía detalla los pasos exactos para subir el proyecto **`SigmaGrav-Web/Api`** a tu servicio de hosting (Hostinger, cPanel, Plesk o Servidor VPS) para que funcione al 100%.

---

## 📋 Resumen: ¿Funcionará directo?

**Sí, funcionará perfectamente**, pero debes realizar **4 configuraciones puntuales** en el hosting debido a la naturaleza de los proyectos Laravel y al peso de los archivos APK.

---

## 🛠️ Pasos para el Despliegue

### Paso 1: Configurar la Versión de PHP en el Hosting
En el panel del hosting (hPanel de Hostinger o cPanel):
- Selecciona **PHP 8.2** o **PHP 8.3**.
- Asegúrate de tener activas las extensiones de PHP: `zip`, `openssl`, `mbstring`, `pdo_sqlite` (o `pdo_mysql` si usas MySQL).

---

### Paso 2: Subir los Archivos y Configurar el `.env`
1. Sube el contenido de la carpeta `/home/burger/workspace/repos/SigmaGrav/SigmaGrav-Web/Api` a tu directorio del servidor (ej: `public_html/api` o la raíz de tu dominio).
2. Duplica o crea el archivo **`.env`** en la raíz del servidor y ajusta las siguientes variables clave:

```ini
APP_NAME="SigmaGrav API"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://tu-dominio.com   <-- Pon aquí tu dominio/subdominio HTTPS real

# Configuración de Base de Datos (Si usas SQLite mantén esto):
DB_CONNECTION=sqlite

# Si prefieres usar MySQL en el hosting, ajusta a:
# DB_CONNECTION=mysql
# DB_HOST=localhost
# DB_DATABASE=nombre_bd_hosting
# DB_USERNAME=usuario_bd_hosting
# DB_PASSWORD=password_bd_hosting
```

---

### Paso 3: Crear el Enlace Simbólico de Almacenamiento (Storage Link)
Para que los archivos `.apk` y licencias subidas sean accesibles públicamente para descarga:
- En Hostinger/cPanel, ve a la sección **SSH / Terminal** o usa el programador de tareas (Cron Job) una única vez para ejecutar:
  ```bash
  php artisan storage:link
  ```
- O en su defecto, en Hostinger ve a **Administrador de Archivos** y confirma que la carpeta `public/storage` apunte a `storage/app/public`.

---

### Paso 4: Ajustar Límites de Subida de Archivos en Hostinger (PHP INI)
Como las APKs pesan entre 20MB y 100MB, debes elevar los límites de PHP en Hostinger:
1. Ve a **Hosting → Configuración de PHP → Opciones de PHP**.
2. Modifica los siguientes valores:
   - `upload_max_filesize`: **128M**
   - `post_max_size`: **128M**
   - `memory_limit`: **256M**
   - `max_execution_time`: **300**
3. El archivo `public/.htaccess` del proyecto ya incluye estas reglas preparadas.

---

### Paso 5: Migraciones y Creación del Usuario Administrador
Ejecuta las migraciones en la terminal del hosting:
```bash
php artisan migrate --force
```

---

## 🔗 Conectar el Backend Local con el Hosting

Una vez subida la web a la nube, solo debes actualizar **1 sola línea** en el archivo `.env` del backend local de las estaciones (`SigmaGrav-BackEnd/.env`):

```ini
SIGMAGRAV_WEB_API_URL=https://tu-dominio.com/api/v1/apk/check-update
```

¡Listo! A partir de ese momento, todos los Posnets Android recibirán automáticamente sus licencias y actualizaciones servidas desde tu Hosting Hostinger.
