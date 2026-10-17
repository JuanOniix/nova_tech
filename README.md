# 🚀 ERP Novatech — Administración y Control de Acceso Granular en MySQL

Este repositorio contiene la configuración completa de la base de datos `erp_novatech` implementada sobre **Ubuntu (Linux)**, cumpliendo con estrictas directrices de seguridad, control de acceso basado en roles (RBAC), seguridad granular a nivel de columnas y auditoría de privilegios en **MySQL**.

---

## 📋 Tabla de Contenidos

1. [Descripción del Proyecto](#-descripción-del-proyecto)
2. [Arquitectura de Base de Datos](#-arquitectura-de-base-de-datos)
3. [Estructura de Usuarios y Hosts](#-estructura-de-usuarios-y-hosts)
4. [Matriz de Roles y Privilegios (RBAC)](#-matriz-de-roles-y-privilegios-rbac)
5. [Seguridad Granular a Nivel de Columnas](#-seguridad-granular-a-nivel-de-columnas)
6. [Guía de Ejecución y Scripts SQL](#-guía-de-ejecución-y-scripts-sql)
7. [Pruebas y Evidencias de Seguridad](#-pruebas-y-evidencias-de-seguridad)
8. [Consultas de Auditoría](#-consultas-de-auditoría)

---

## 🎯 Descripción del Proyecto

El objetivo principal es implementar un modelo de seguridad robusto para el sistema ERP de **Novatech**. El proyecto aplica el **Principio de Mínimo Privilegio (POLP)** para garantizar que cada departamento y usuario del sistema cuente únicamente con las operaciones necesarias para su labor diaria, evitando la exposición no autorizada o modificación ilícita de los datos.

### 🛠️ Tecnologías Utilizadas
* **Sistema Operativo:** Ubuntu (WSL2 / Linux)
* **Motor de Base de Datos:** MySQL 8.0+
* **Cliente GUI:** MySQL Workbench
* **Lenguaje:** SQL (DDL, DML, DCL, DQL)

---

## 🗄️ Arquitectura de Base de Datos

La base de datos `erp_novatech` se compone de **4 tablas interrelacionadas**:

1. **`departamentos`**: Almacena las áreas organizacionales de la empresa, su ubicación física y su presupuesto asignado.
2. **`empleados`**: Contiene la información del personal, salarios, cargos y la relación con su departamento correspondiente (`FOREIGN KEY`).
3. **`clientes`**: Registra la información de contacto y localización de los compradores.
4. **`ventas`**: Registra las transacciones comerciales vinculando clientes y empleados vendedores (`FOREIGN KEY`).

```
  +------------------+         +------------------+
  |  departamentos   |         |    empleados     |
  +------------------+         +------------------+
  | id_departamento  |<--------| id_departamento  |
  | nombre_dept      | 1     N | id_empleado      |<-------+
  | presupuesto      |         | nombre, salario  |        |
  +------------------+         +------------------+        |
                                                           |
  +------------------+         +------------------+        |
  |     clientes     |         |      ventas      |        |
  +------------------+         +------------------+        |
  | id_cliente       |<--------| id_cliente       | 1    N |
  | nombre, email    | 1     N | id_venta, monto  |--------+
  +------------------+         | id_empleado      |
                               +------------------+
```

---

## 👤 Estructura de Usuarios y Hosts

Para garantizar la correcta autenticación en entornos locales y remotos, se configuraron las siguientes cuentas de usuario:

| Usuario | Host Autorizado | Descripción / Área |
| :--- | :--- | :--- |
| `director_dba` | `localhost` | Administrador general de la base de datos (`WITH GRANT OPTION`). |
| `carlos_dev` | `%`, `JuanDesk.mshome.net` | Desarrollador de software con permisos DDL y DML. |
| `ana_ventas` | `localhost`, `%` | Ejecutiva de ventas para gestión de clientes y transacciones. |
| `rrhh_user` | `localhost` | Analista de Gestión Humana con acceso restringido a nómina. |

---

## 🛡️ Matriz de Roles y Privilegios (RBAC)

Se definieron roles reutilizables asignados de forma automática al iniciar sesión (`activate_all_roles_on_login = ON`):

### 1. `rol_auditor`
* **Permisos:** `SELECT`, `SHOW VIEW`.
* **Propósito:** Auditorías de lectura sin posibilidad de alteración de datos.

### 2. `rol_desarrollador`
* **Permisos:** `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `ALTER`, `DROP`, `REFERENCES`, `CREATE TEMPORARY TABLES`.
* **Propósito:** Gestión completa de estructuras DDL y manipulación DML sobre `erp_novatech.*`.

### 3. `rol_ventas`
* **Permisos:** `SELECT`, `INSERT`.
* **Propósito:** Consulta de catálogo/clientes e inserción de nuevas ventas. Se restringe `DELETE` y `UPDATE`.

---

## 🔒 Seguridad Granular a Nivel de Columnas

Para el departamento de Recursos Humanos (`rrhh_user`), se aplicó una restricción de granularidad estricta para proteger los datos personales de los empleados:

```sql
-- Lectura general sobre la tabla empleados
GRANT SELECT ON erp_novatech.empleados TO 'rrhh_user'@'localhost';

-- Edición EXCLUSIVA para las columnas salario y cargo
GRANT UPDATE (salario, cargo) ON erp_novatech.empleados TO 'rrhh_user'@'localhost';
```

---

## 📜 Guía de Ejecución y Scripts SQL

### 1. Creación de la Base de Datos y Estructura (DDL)

```sql
CREATE DATABASE IF NOT EXISTS erp_novatech 
CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

USE erp_novatech;

CREATE TABLE departamentos (
    id_departamento INT AUTO_INCREMENT PRIMARY KEY,
    nombre_departamento VARCHAR(100) NOT NULL UNIQUE,
    ubicacion VARCHAR(100),
    presupuesto DECIMAL(12, 2) DEFAULT 0.00,
    fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE empleados (
    id_empleado INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    apellido VARCHAR(100) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    telefono VARCHAR(20),
    cargo VARCHAR(100) NOT NULL,
    salario DECIMAL(10, 2) NOT NULL,
    fecha_contratacion DATE NOT NULL,
    id_departamento INT,
    estado ENUM('activo', 'inactivo') DEFAULT 'activo',
    FOREIGN KEY (id_departamento) REFERENCES departamentos(id_departamento) ON DELETE SET NULL
) ENGINE=InnoDB;

CREATE TABLE clientes (
    id_cliente INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    apellido VARCHAR(100) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    telefono VARCHAR(20),
    ciudad VARCHAR(100),
    fecha_registro TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE ventas (
    id_venta INT AUTO_INCREMENT PRIMARY KEY,
    fecha_venta DATETIME DEFAULT CURRENT_TIMESTAMP,
    monto DECIMAL(10, 2) NOT NULL,
    metodo_pago ENUM('Efectivo', 'Tarjeta', 'Transferencia') NOT NULL,
    id_cliente INT NOT NULL,
    id_empleado INT NOT NULL,
    FOREIGN KEY (id_cliente) REFERENCES clientes(id_cliente) ON DELETE CASCADE,
    FOREIGN KEY (id_empleado) REFERENCES empleados(id_empleado) ON DELETE CASCADE
) ENGINE=InnoDB;
```

---

## 🧪 Pruebas y Evidencias de Seguridad

### Prueba 1: Bloqueo de Borrado para `ana_ventas`
* **Acción:** `ana_ventas` intenta ejecutar `DELETE FROM ventas WHERE id_venta = 1;`.
* **Resultado:** ❌ **Bloqueado por el motor.**
* **Error:** `Error Code: 1142. DELETE command denied to user 'ana_ventas'@'localhost' for table 'ventas'`.

### Prueba 2: Modificación de Campos Autorizados para `rrhh_user`
* **Acción:** `rrhh_user` ejecuta:
  ```sql
  UPDATE empleados SET salario = 5200000.00, cargo = 'Líder Técnico' WHERE id_empleado = 1;
  ```
* **Resultado:** ✅ **Exitoso.** `1 row(s) affected`.

### Prueba 3: Bloqueo de Granularidad para `rrhh_user`
* **Acción:** `rrhh_user` intenta modificar el nombre de un empleado:
  ```sql
  UPDATE empleados SET nombre = 'Carlos Mario' WHERE id_empleado = 1;
  ```
* **Resultado:** ❌ **Bloqueado por el motor.**
* **Error:** `Error Code: 1143. UPDATE command denied to user 'rrhh_user'@'localhost' for column 'nombre' in table 'empleados'`.

---

## 🔍 Consultas de Auditoría

Para verificar el estado del servidor y la distribución de privilegios:

```sql
-- 1. Listar todos los usuarios y sus hosts autorizados
SELECT user, host, plugin FROM mysql.user ORDER BY user ASC;

-- 2. Verificar la asignación de roles a usuarios
SELECT FROM_USER AS 'Rol', TO_USER AS 'Usuario', TO_HOST AS 'Host' FROM mysql.role_edges;

-- 3. Comprobar los permisos efectivos de un usuario específico
SHOW GRANTS FOR 'carlos_dev'@'%' USING 'rol_desarrollador'@'localhost';
```

---

## ✒️ Autor
* **Proyecto ERP Novatech**
* **Materia:** Base de Datos / Administración de BD
* **Fecha:** Octubre 2026
