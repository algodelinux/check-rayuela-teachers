# Check-Rayuela-Teachers 🏫🔍

Herramienta de administración para centros educativos de Extremadura. Permite sincronizar las cuentas de usuario de los servidores del centro con la exportación oficial de datos de **Rayuela**, automatizando la detección y limpieza del profesorado que ya no pertenece a la plantilla.

---

## 📖 Tabla de Contenidos
- [Descripción](#-descripción)
- [Requisitos](#️-requisitos)
- [Instalación](#-instalación)
- [Configuración](#-configuración)
- [Modo de Uso](#-modo-de-uso)
- [Flujo de Trabajo Seguro](#-flujo-de-trabajo-seguro)
- [Seguridad y Advertencias](#-seguridad-y-advertencias)

---

## 📝 Descripción

El script `check-rayuela-teachers` es una utilidad en Perl diseñada para mantener la higiene del servidor LDAP y el sistema de archivos del centro educativo. 

### Funciones principales:
* **Contraste de Datos**: Compara los logins activos en el XML de Rayuela con las cuentas existentes en el servidor LDAP.
* **Generación de Bajas LDAP**: Crea un archivo `.ldif` para eliminar la entrada del usuario (`ou=People`) y su grupo personal asociado (`ou=Group`).
* **Limpieza de Almacenamiento**: Genera un script Bash para eliminar físicamente los directorios `/home/profesor/`.
* **Sistema de Excepciones**: Permite definir una "lista blanca" de UIDs para evitar el borrado accidental de cuentas especiales que no aparecen en Rayuela.

---

## 🛠️ Requisitos

El sistema debe tener instaladas las siguientes dependencias de Perl:

```bash
sudo apt update
sudo apt install libnet-ldap-perl libxml-libxml-perl
```

---

## 📥 Instalación

Descarga el script directamente en tu servidor o clona el repositorio:

```bash
cd /usr/local/sbin/
wget https://raw.githubusercontent.com/algodelinux/check-rayuela-teachers/refs/heads/main/check-rayuela-teachers
```

Otorga permisos de ejecución al archivo:

```bash
chmod +x check-rayuela-teachers
```

---

### ⚙️ Configuración

Antes de la primera ejecución, es obligatorio editar las variables de configuración al inicio del script para que coincidan con la red y estructura de tu centro:

```perl
# Configuración en las líneas 7-10 del script:
my $ldap_server = "ip-servidor-ldap";           # IP de tu servidor LDAP (ej. 192.168.1.250)
my $base_dn     = "dc=instituto,dc=extremadura,dc=es"; 
my $ou_people   = "ou=People,$base_dn";
my $ou_group    = "ou=Group,$base_dn";
```

---

### 🚀 Modo de Uso

1. Preparación de archivos
* Archivo XML: Exporta el fichero ExportacionDatosProfesorado.xml desde la plataforma Rayuela.

* Archivo de Excepciones (Opcional): Crea un archivo de texto (ej: excepciones.txt) con los logins que quieres proteger, un usuario por línea (ej: admin, coordinadortic, tic).

2. Ejecución del Script
Ejecuta el script pasando el XML como primer argumento y el archivo de excepciones como segundo:

```bash
./check-rayuela-teachers ExportacionDatosProfesorado.xml excepciones.txt
```

3. Resultados generados
El script mostrará un resumen por pantalla y creará automáticamente:

* bajas_profesores.ldif: Instrucciones formateadas para borrar las entradas en LDAP.

* borrar_homes_profesores.sh: Script ejecutable para borrar las carpetas de usuario en el servidor de archivos.

### 🧹 Flujo de Trabajo Seguro

El script no realiza cambios directos de forma automática para evitar errores fatales. Sigue este orden para aplicar los cambios una vez revisados los ficheros generados:

* Sincronización de LDAP:
Ejecuta el comando ldapmodify para eliminar los usuarios del directorio:

```bash
ldapmodify -x -H ldap://ip-servidor-ldap \
-D "cn=admin,ou=People,dc=instituto,dc=extremadura,dc=es" \
-W -f bajas_profesores.ldif
```

* Limpieza del sistema de archivos:
Una vez borrados de LDAP, procede a liberar espacio en el disco:

```bash
# Se recomienda revisar el contenido antes de ejecutar
cat borrar_homes_profesores.sh

# Ejecutar con privilegios de superusuario
sudo ./borrar_homes_profesores.sh
```

---

### ⚠️ Seguridad y Advertencias

* Protección de Alumnos: El script incluye un filtro estricto que solo selecciona usuarios cuya ruta de inicio (homeDirectory) esté dentro de /home/profesor/. Las cuentas de alumnos (/home/alumnos/) están protegidas y no se verán afectadas.

* Cuentas Críticas: Es fundamental incluir en la lista de excepciones cuentas como admin, tic, secretaria o cualquier usuario genérico que deba permanecer activo aunque no figure en la exportación oficial de Rayuela.

* Backup de Seguridad: Se recomienda realizar un volcado de seguridad del LDAP (slapcat > backup_previo.ldif) antes de proceder con cualquier borrado masivo.

* Responsabilidad: Esta herramienta realiza acciones de borrado irreversibles. Úsala con precaución y siempre bajo la supervisión del administrador de red.
