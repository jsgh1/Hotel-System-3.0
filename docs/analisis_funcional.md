# Análisis Funcional - Sistema Hotelero

**Fecha:** Mayo 2026  
**Versión:** 1.0  
**Estado:** Completado y Validado

---

## 1. Resumen Ejecutivo

El **Sistema Hotelero** es una solución integral de gestión para cadenas hoteleras, diseñada con una arquitectura modular basada en **dominios funcionales independientes**. La base de datos PostgreSQL está optimizada para escalabilidad, mantenibilidad y auditoría completa.

### Objetivos principales:
- ✅ Gestionar reservas, check-in y check-out de huéspedes
- ✅ Controlar inventario de habitaciones y disponibilidad
- ✅ Administrar facturación, pagos y prefacturas
- ✅ Mantener configuración empresarial y seguridad de acceso
- ✅ Permitir notificaciones, promociones y fidelización
- ✅ Registrar mantenimiento y condiciones de habitaciones

---

## 2. Visión de Arquitectura

### 2.1 Principios Rectores

| Principio | Descripción | Implementación |
| --- | --- | --- |
| **Modularidad por Dominio** | Cada dominio funcional es autónomo | 8 schemas PostgreSQL separados |
| **Nomenclatura Técnica en Inglés** | Evita ambigüedad y mejora internacionalización | Todos los objetos en inglés |
| **Identificadores UUID** | Escalabilidad y distribución | PK en todas las tablas |
| **Auditoría Completa** | Trazabilidad de cambios | Campos `created_by`, `updated_by`, `deleted_by`, timestamps |
| **Control de Versiones** | Migraciones controladas y repetibles | Liquibase con changelogs YAML |
| **Infraestructura Reproducible** | Entorno consistente en desarrollo y producción | Docker + Docker Compose |

### 2.2 Stack Tecnológico

```
┌─────────────────────────────────────────────────────┐
│        APLICACIÓN (Por definir)                     │
├─────────────────────────────────────────────────────┤
│        ORM / SQL Driver (JDBC, Python, etc.)        │
├─────────────────────────────────────────────────────┤
│    PostgreSQL 14+ (Puerto 5432)                     │
│    - 8 Schemas funcionales                          │
│    - 50+ Tablas de negocio                          │
│    - Views y Materialized Views para reportes       │
│    - Functions y Procedures para lógica             │
│    - Triggers para integridad de datos              │
├─────────────────────────────────────────────────────┤
│    Liquibase (Versionamiento de migraciones)        │
│    - Changelogs YAML/SQL                            │
│    - Rollbacks automáticos                          │
│    - Historial auditado en DATABASECHANGELOG        │
├─────────────────────────────────────────────────────┤
│    Docker + Docker Compose (Orquestación)           │
│    - Contenedor PostgreSQL con volumen persistente  │
│    - Scripts de carga y validación                  │
│    - Smoke tests integrados                         │
└─────────────────────────────────────────────────────┘
```

---

## 3. Descripción de Dominios Funcionales

### 3.1 Domain 1: Configuration
**Propósito:** Catalogos maestros y metadatos base del sistema.

**Tablas principales:**
- `customer`: Registro de clientes (personas naturales)
- `person`: Información personal detallada
- `company`: Información de la empresa/sede principal
- `day_type`: Clasificación de días (laboral, festivo, etc.)
- `payment_method`: Métodos de pago disponibles
- `legal_information`: Datos legales y fiscales
- `employee`: Registro de empleados
- `price`: Precios base y configuración

**Características:**
- No contiene operaciones transaccionales
- Datos relativamente estáticos
- Referenciado por la mayoría de dominios
- Soporta soft-delete con `deleted_at`

---

### 3.2 Domain 2: Security
**Propósito:** Control de acceso, usuarios, roles y permisos.

**Tablas principales:**
- `role`: Roles del sistema (admin, instructor, manager, etc.)
- `permission`: Permisos granulares (crear, leer, actualizar, eliminar)
- `module`: Módulos del sistema (reservas, facturación, etc.)
- `screen`: Pantallas/vistas de la aplicación
- `app_user`: Usuarios del sistema
- `app_user_role`: Relación usuario-rol (muchos-a-muchos)
- `role_permission`: Relación rol-permiso (muchos-a-muchos)
- `module_screen`: Relación módulo-pantalla

**Características:**
- Control de acceso basado en roles (RBAC)
- Auditoria de cambios de permiso
- Separación entre módulos de negocio y pantallas UI
- No afecta datos de negocio

---

### 3.3 Domain 3: Distribution
**Propósito:** Gestión física de sedes, habitaciones y disponibilidad.

**Tablas principales:**
- `branch`: Sedes/sucursales del hotel
- `room_type`: Tipos de habitación (simple, doble, suite, etc.)
- `room_status`: Estados posibles (disponible, ocupada, mantenimiento, etc.)
- `room`: Registro de habitaciones individuales
- `room_availability`: Calendario de disponibilidad
- `room_catalog`: Catálogo de características y servicios por tipo
- `price_room_type`: Relación con precios (FK a configuration.price)

**Características:**
- Estructura jerárquica: Branch → Room Type → Room
- Auditoria de cambios en disponibilidad
- Soporta disponibilidad por fecha
- Base para reservas

---

### 3.4 Domain 4: Service Delivery
**Propósito:** Ciclo de vida de reservas, estadías y check-in/out.

**Tablas principales:**
- `room_reservation`: Solicitudes de reserva
- `room_cancellation`: Cancelaciones y sus motivos
- `stay`: Estadía efectiva del huésped
- `check_in`: Registro de entrada
- `check_out`: Registro de salida

**Características:**
- Flujo transaccional crítico
- Estados: PENDING → CONFIRMED → CHECKED_IN → CHECKED_OUT
- Validaciones de fechas (end_date > start_date)
- Cálculo de noches y montos
- Relación con Customer y Room
- Auditoría de quien hace cada cambio

---

### 3.5 Domain 5: Inventory
**Propósito:** Gestión de productos, servicios y disponibilidad.

**Tablas principales:**
- `supplier`: Proveedores de productos y servicios
- `product`: Inventario de productos (amenities, etc.)
- `service`: Servicios adicionales (spa, lavandería, etc.)
- `product_sale`: Registro de ventas de productos
- `service_sale`: Registro de ventas de servicios
- `product_tracking`: Seguimiento de stock
- `inventory_availability`: Disponibilidad actual

**Características:**
- Independiente del ciclo de reservas
- Vinculado a facturación y costos
- Control de stock en tiempo real
- Proveedores auditados

---

### 3.6 Domain 6: Billing
**Propósito:** Facturación, pagos e historial de transacciones financieras.

**Tablas principales:**
- `pre_invoice`: Prefacturas (borrador antes de facturación final)
- `invoice`: Facturas emitidas
- `partial_payment`: Pagos parciales o diferidos
- `purchase_detail`: Detalle de items en compra

**Características:**
- Relación con Stay para facturación por estadía
- Validaciones de montos (subtotal, tax, discount, total)
- Estados de factura: ISSUED, PAID, CANCELLED
- Auditoría completa de emisión y cambios
- Integración con Configuration para métodos de pago

---

### 3.7 Domain 7: Notification
**Propósito:** Sistema de comunicaciones, promociones y fidelización.

**Tablas principales:**
- `promotion`: Promociones y descuentos
- `alert`: Alertas y notificaciones
- `terms_condition`: Términos y condiciones aplicables
- `customer_loyalty`: Programa de fidelización

**Características:**
- Independiente de operaciones críticas
- Soporta campañas y comunicaciones masivas
- Auditoría de cambios en T&C
- Vinculación con clientes para personalizacion

---

### 3.8 Domain 8: Maintenance
**Propósito:** Control y seguimiento de mantenimiento de habitaciones.

**Tablas principales:**
- `room_maintenance`: Tickets de mantenimiento
- `maintenance_usage`: Registro de uso y desgaste
- `maintenance_remodeling`: Proyectos de remodelación
- `maintenance_dashboard`: Métrica agregada de estado

**Características:**
- Desvinculado de operaciones de reservas
- Soporta preventivo y correctivo
- Remodelaciones planeadas y ejecutadas
- Dashboard de estado por sede

---

## 4. Flujo de Datos Principal: Ciclo de Hospedaje

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FLUJO DE UNA RESERVA Y ESTADÍA                   │
└─────────────────────────────────────────────────────────────────────┘

1. SOLICITUD
   Customer (configuration.customer)
   + Room (distribution.room)
   → room_reservation (service_delivery)
       Estado: PENDING
       Monto estimado calculado

2. CONFIRMACIÓN (Opcional)
   room_reservation
       Estado: CONFIRMED
       Cliente confirmó disponibilidad

3. CHECK-IN
   stay (service_delivery)
   ← room_reservation (vinculación)
   → check_in (service_delivery)
   room_availability (distribution) ← estado actualizado

4. CONSUMO DE SERVICIOS
   product_sale (inventory) | service_sale (inventory)
   → purchase_detail (billing) ← agregación para factura

5. CHECK-OUT
   check_out (service_delivery)
   stay (actualiza duración real)
   room_availability (distribution) ← estado actualizado

6. FACTURACIÓN
   pre_invoice (billing) ← borrador
   → invoice (billing) ← final
   Cálculos: subtotal, taxes, discounts, total
   payment_method (configuration)

7. PAGO
   partial_payment (billing)
   invoice.status → PAID

8. AUDITORÍA
   Todos los cambios con created_by, updated_by, deleted_by
   timestamps de creación y actualización
```

---

## 5. Modelos de Datos Clave

### 5.1 Tabla de Auditoria Universal

Todas las tablas comparten este patrón:

```sql
id                UUID PRIMARY KEY DEFAULT gen_random_uuid()
-- Campos de negocio ...
created_by        UUID            -- Quién creó
created_at        TIMESTAMPTZ     DEFAULT now()
updated_by        UUID            -- Quién modificó
updated_at        TIMESTAMPTZ
deleted_by        UUID            -- Quién eliminó (soft-delete)
deleted_at        TIMESTAMPTZ
status            record_status   -- ACTIVE | INACTIVE (enum)
```

### 5.2 Relaciones Principales

| Relación | Tablas | Tipo | Notas |
| --- | --- | --- | --- |
| Customer → Reservation | configuration.customer ← service_delivery.room_reservation | 1:N | FK con validación |
| Customer → Invoice | configuration.customer ← billing.invoice | 1:N | FK con validación |
| Room → Reservation | distribution.room ← service_delivery.room_reservation | 1:N | FK con validación |
| Reservation → Stay | service_delivery.room_reservation ← service_delivery.stay | 1:1 | FK opcional (reserva no confirmada sin stay) |
| Stay → Cancellation | service_delivery.stay ← service_delivery.room_cancellation | 1:N | Motivos de cancelación |
| Stay → Invoice | service_delivery.stay ← billing.invoice | 1:1 | FK requerido |
| Stay → Product Sales | service_delivery.stay ← inventory.product_sale | 1:N | Consumos durante hospedaje |
| Role ↔ Permission | security.role ↔ security.permission | M:N | Tabla intermedia: role_permission |
| User ↔ Role | security.app_user ↔ security.role | M:N | Tabla intermedia: app_user_role |
| Room Type → Price | distribution.room_type ← configuration.price | 1:N | FK indirecto |
| Branch → Room | distribution.branch ← distribution.room | 1:N | FK directo |

### 5.3 Tipos de Datos Customizados (Domain Types)

```sql
-- Enumeraciones de estado
record_status       -- ACTIVE, INACTIVE
reservation_status  -- PENDING, CONFIRMED, CANCELLED
invoice_status      -- ISSUED, PAID, CANCELLED
-- Extensibles según necesidad
```

---

## 6. Objetos Derivados

### 6.1 Vistas (Views)
- `v_room_availability`: Disponibilidad actual de habitaciones
- `v_reservation_details`: Detalles de reservas con cliente, habitación y monto
- `v_stay_billing`: Detalles de estadía con facturación asociada
- `v_user_roles`: Matriz usuario-rol actual
- `v_current_maintenance_dashboard`: Estado de mantenimiento por sede

### 6.2 Vistas Materializadas (Materialized Views)
- `mv_monthly_revenue`: Ingresos mensuales consolidados
- `mv_branch_occupancy`: Tasa de ocupación por rama

### 6.3 Funciones Principales
- `fn_set_updated_at()`: Actualiza `updated_at` automáticamente en triggers
- `fn_calculate_nights()`: Calcula noches entre check-in y check-out
- `fn_calculate_total()`: Calcula total = subtotal + tax - discount
- `fn_calculate_reservation_price()`: Estima monto de reserva según tipo habitación y noches

### 6.4 Procedimientos (Procedures)
Automatización de operaciones complejas y carga segura de datos.

### 6.5 Triggers (Triggers)
- Auditoría automática de cambios
- Actualización de timestamps
- Validación de integridad antes de inserciones/actualizaciones

### 6.6 Índices
Optimización de búsquedas frecuentes:
- Por ID (UUID)
- Por customer_id (búsquedas de cliente)
- Por room_id (búsquedas de habitación)
- Por fecha de reserva/estadía
- Por status (filtros de estado)
- Índices compuestos para queries complejas

---

## 7. Flujos de Datos Transversales

### 7.1 Seguridad y Acceso
```
app_user (security)
  ↓
app_user_role (security)
  ↓
role (security) → role_permission (security)
  ↓
permission (security)
```
El sistema valida permisos antes de cualquier operación CRUD.

### 7.2 Auditoría y Compliance
- Todas las tablas registran `created_by` (usuario que crea)
- Todas registran `updated_by` (usuario que modifica)
- Soft-delete con `deleted_at` y `deleted_by` para cumplimiento
- Liquibase mantiene historial en `DATABASECHANGELOG`

### 7.3 Configuración Centralizada
Configuration domain alimenta el resto:
- Métodos de pago → Billing
- Clientes → Todas las operaciones
- Empleados → Auditoría
- Precios → Billing y Reservation
- Día tipo → Cálculos de tarifa

---

## 8. Consideraciones de Seguridad

### 8.1 Control de Acceso (DCL)
- Role `ariel5253` (instructor) con permisos limitados
- Grants granulares por schema y tabla
- Políticas de seguridad (RLS) preparadas para extensión futura
- No se permite DDL a usuarios no administradores

### 8.2 Validaciones a Nivel de Base de Datos
```sql
CONSTRAINT ck_reservation_date_values CHECK (end_date > start_date)
CONSTRAINT ck_reservation_persons CHECK (guest_count > 0)
CONSTRAINT ck_invoice_amounts CHECK (subtotal >= 0 AND tax >= 0 AND discount >= 0 AND total >= 0)
-- Y muchas más según tabla
```

### 8.3 Integridad Referencial
- Foreign keys en todas las relaciones
- Cascadas de DELETE configuradas según necesidad del negocio
- Soft-delete para preservar historia

---

## 9. Consideraciones de Performance

### 9.1 Indexación Estratégica
- PKs (UUID) con índice B-tree automático
- FKs para evitar sequential scans
- Índices en columnas de filtrado frecuente
- Índices compuestos para JOIN complejos

### 9.2 Particionamiento (Futuro)
- Posibilidad de particionar `room_reservation` por rango de fechas
- Particionar `invoice` por año fiscal
- Particionar `stay` por branch

### 9.3 Materialized Views para Reportes
- `mv_monthly_revenue`: Cálculos precalculados
- `mv_branch_occupancy`: Agregaciones diarias/mensuales
- Refresh manual o programado (no transaccional)

### 9.4 Caché a Nivel de Aplicación
- Configuración (cambia raramente) → Cache
- Disponibilidad de habitaciones → Invalidar en check-in/out
- Roles y permisos → Cache con TTL

---

## 10. Flujo de Versionamiento (Liquibase)

```
liquibase.properties (configuración de conexión)
  ↓
changelog/changelog-master.yaml (entrada maestra)
  ↓
├─ 01_ddl/
│  ├─ 00_extensions/ → Habilitar extensiones PostgreSQL
│  ├─ 01_schemas/ → Crear 8 schemas funcionales
│  ├─ 02_types/ → Crear tipos/enumeraciones
│  ├─ 03_tables/ → Crear 50+ tablas por dominio
│  ├─ 04_views/ → Crear vistas
│  ├─ 05_materialized_views/ → Crear vistas materializadas
│  ├─ 06_functions/ → Crear funciones
│  ├─ 07_procedures/ → Crear procedimientos
│  ├─ 08_triggers/ → Crear triggers
│  └─ 09_indexes/ → Crear índices
├─ 02_dml/ (Datos iniciales)
│  └─ 00_inserts/ → 8 archivos por dominio
├─ 03_dcl/ (Control de acceso)
│  ├─ 00_roles/ → Crear roles
│  ├─ 01_grants/ → Otorgar permisos
│  └─ 02_policies/ → Políticas futuras
├─ 04_tcl/ (Transacciones)
│  ├─ 00_transaction_blocks/ → Carga segura
│  └─ 01_manual_recoveries/ → Recuperación
└─ 05_rollbacks/ → Scripts de reversión por bloque
```

Cada cambio:
1. Se registra en `DATABASECHANGELOG`
2. Queda idempotente (seguro ejecutar múltiples veces)
3. Tiene rollback asociado
4. Es auditado

---

## 11. Validación y Testing

### 11.1 Smoke Test (`checks/001_smoke_test.sql`)
Verifica:
- ✅ Existencia de todos los 8 schemas
- ✅ Tabla de auditoria Liquibase
- ✅ Número esperado de tablas por schema
- ✅ Usuario instructor `ariel5253` existe
- ✅ Grants para instructor son correctos

### 11.2 CI/CD Workflow (`.github/workflows/db-ci.yml`)
Automático en cada PR:
1. Levanta PostgreSQL en contenedor
2. Ejecuta Liquibase completo
3. Corre smoke test
4. Verifica rollback seguro
5. Valida vistas y funciones

### 11.3 Carga Local (`scripts/load-database.ps1`)
Script PowerShell para desarrolladores:
```powershell
1. Levanta docker-compose.yml
2. Espera a PostgreSQL listo
3. Ejecuta liquibase
4. Corre smoke test
5. Reporyta si es exitoso
```

---

## 12. Estructura de Entrega

```
infra/db/
├── README.md                      # Este es el análisis técnico
├── liquibase.properties           # Configuración de Liquibase
├── docker/
│   └── docker-compose.yml         # Contenedor PostgreSQL
├── scripts/
│   └── load-database.ps1          # Script de carga local
├── checks/
│   └── 001_smoke_test.sql         # Prueba de salud
├── 01_ddl/                        # Data Definition Language
│   ├── 00_extensions/             # Extensiones PostgreSQL
│   ├── 01_schemas/                # 8 schemas funcionales
│   ├── 02_types/                  # Tipos y enumeraciones
│   ├── 03_tables/                 # Tablas por dominio (8 carpetas)
│   ├── 04_views/                  # Vistas
│   ├── 05_materialized_views/     # Vistas materializadas
│   ├── 06_functions/              # Funciones
│   ├── 07_procedures/             # Procedimientos
│   ├── 08_triggers/               # Triggers
│   └── 09_indexes/                # Índices
├── 02_dml/                        # Data Manipulation Language
│   └── 00_inserts/                # Datos iniciales por dominio
├── 03_dcl/                        # Data Control Language
│   ├── 00_roles/                  # Definición de roles
│   ├── 01_grants/                 # Otorgamiento de permisos
│   └── 02_policies/               # Políticas de fila
├── 04_tcl/                        # Transactional Control Language
│   ├── 00_transaction_blocks/     # Bloques transaccionales
│   └── 01_manual_recoveries/      # Scripts de recuperación
└── 05_rollbacks/                  # Scripts de reversión
    ├── 01_ddl/
    ├── 02_dml/
    ├── 03_dcl/
    └── 04_tcl/
```

---

## 13. Matriz de Responsabilidades por Dominio

| Dominio | Responsable Primario | Responsable de Datos | Responsable Técnico |
| --- | --- | --- | --- |
| Configuration | Equipo Administrativo | Jefe Operativo | DBA |
| Security | Jefe de Seguridad | Jefe de Seguridad | DBA |
| Distribution | Gerente de Operaciones | Gerente de Operaciones | DBA |
| Service Delivery | Gerente de Reservas | Gerente de Reservas | DBA |
| Inventory | Administrador de Bodega | Administrador de Bodega | DBA |
| Billing | Controller Financiero | Controller Financiero | DBA |
| Notification | Jefe de Marketing | Jefe de Marketing | DBA |
| Maintenance | Jefe de Mantenimiento | Jefe de Mantenimiento | DBA |

---

## 14. Puntos de Extensión y Futuro

### 14.1 Preparados para Crecer
- ✅ Políticas de seguridad a nivel de fila (RLS) → Definir en 03_dcl/02_policies/
- ✅ Procedimientos adicionales de negocio → Agregar en 01_ddl/07_procedures/
- ✅ Vistas adicionales para reportes → Agregar en 01_ddl/04_views/
- ✅ Particionamiento de grandes tablas → Configurar en 01_ddl/09_indexes/
- ✅ Replicación o backup → Configurar en infra/db/docker/

### 14.2 Escalabilidad Horizontal
- UUID permite shard en múltiples bases
- Dominios independientes pueden replicarse selectivamente
- Liquibase facilita sincronización de cambios DDL

### 14.3 Migración y Disaster Recovery
- Liquibase permite rollback rápido
- Soft-delete permite recuperación de datos eliminados
- Auditoría completa permite forensics
- DATABASECHANGELOG permite auditar cambios

---

## 15. Conclusiones y Recomendaciones

### 15.1 Fortalezas del Diseño
✅ **Modularidad:** 8 dominios independientes facilitan mantenimiento y escalabilidad  
✅ **Auditoria:** Trazabilidad completa de cambios  
✅ **Versionamiento:** Liquibase garantiza reproducibilidad  
✅ **Reproducibilidad:** Docker asegura consistencia entre ambientes  
✅ **Documentación:** Análisis dominios + ADRs + Historias de usuario  
✅ **Nomenclatura:** Inglés técnico evita ambigüedad  

### 15.2 Áreas de Vigilancia
⚠️ **Rendimiento:** Validar índices y queries complejas en producción  
⚠️ **Concurrencia:** Pruebas de carga con múltiples usuarios simultáneos  
⚠️ **Backups:** Estrategia de backup y recovery ante disaster  
⚠️ **Monitoreo:** Alertas en tiempo real para anomalías  

### 15.3 Recomendaciones Inmediatas
1. Completar datos semilla realistas para cada dominio
2. Implementar stored procedures complejos para operaciones críticas
3. Definir políticas de seguridad granulares (RLS)
4. Establecer SLA de performance y monitoreo
5. Crear runbooks de operación y disaster recovery
6. Validar con carga realista el número de conexiones soportadas
7. Documentar la estrategia de backup (point-in-time recovery)

---

## 16. Referencias Cruzadas

| Documento | Propósito | Ubicación |
| --- | --- | --- |
| Historias de Usuario | Requerimientos detallados | `docs/Historias de usuario.md` |
| Análisis de Dominios | Definición de 8 dominios | `docs/analisis_dominios.md` |
| ADRs (Architecture Decision Records) | Justificación de decisiones | `docs/ADR/` |
| Matriz de Trazabilidad | Mapeo HU ↔ Archivos | `docs/matriz_trazabilidad_hu.md` |
| Guía de Ejecución | Cómo levantar la base | `docs/guia_ejecucion_y_validacion.md` |
| Orden de Carga | Secuencia de migraciones | `docs/orden_carga.md` |
| Plan de Trabajo | Asignación y cronograma | `docs/responsabilidades/plan_trabajo_inicial.md` |

---

**Fin del Análisis Funcional**

Este documento constituye la base arquitectónica y funcional del Sistema Hotelero y debe ser considerado junto con los ADRs, historias de usuario y documentación técnica en `infra/db/README.md`.

