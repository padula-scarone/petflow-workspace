# 1. Contexto del Proyecto: PetFlow (ERP PWA)

## Dominio del Negocio (Mundo Mascotas)
- **Rubro:** Pet shop y forrajería minorista.
- **Catálogo principal:** Alimento balanceado (cerrado y fraccionado a granel), accesorios, productos de higiene y sanidad.
- **Operatoria:** Venta rápida en mostrador y envíos a domicilio de rango corto.

## Arquitectura y Restricciones
- **Tipo de aplicación:** ERP bajo arquitectura PWA (Progressive Web App) en la nube.
- **Restricción de Red:** El local no posee Wi-Fi. El sistema debe estar optimizado para redes de datos móviles (bajo payload, sin recursos pesados).
- **Dispositivos:** Enfoque *Mobile-First*. Debe operar con fluidez en los celulares del personal y computadoras de escritorio.

## Estructura Modular
El sistema se compone de 6 módulos interconectados (operando bajo transacciones atómicas cruzadas):
1. **Ventas:** Procesamiento ágil en mostrador, descuentos dinámicos, gestión de fiados, cambios de mercadería y envíos de comprobantes vía WhatsApp (Deep Links).
2. **Inventario:** Directorio de proveedores, catálogo de productos, conversión automática de bolsa cerrada a stock a granel, y alertas de reposición.
3. **Caja:** Libro mayor del negocio. Controla apertura, cierre, ingresos (ventas), egresos (gastos) y arqueos físicos de efectivo.
4. **Gastos:** Clasificador transaccional de costos fijos y operativos.
5. **Estadísticas:** Dashboard de lectura que cruza datos para mostrar KPIs y rentabilidad neta.
6. **Seguridad:** Autenticación de usuarios, inyección de autoría en registros y control de accesos (RBAC: Vendedor, Gestor Comercial, Gerente).

---

# 2. Entregable 1: Alcance de la Demo

## Flujos Operativos Requeridos
Despliegue Full-Stack (Front, Back, BD) de los siguientes Casos de Uso Base (CUB):
1. **Autenticación (CUB 6.1):** Login/Logout real con credenciales cifradas (JWT/bcrypt). Restricción de pantallas por roles e inyección de autoría en transacciones.
2. **Registrar Venta (CUB 1.1):** Búsqueda manual de catálogo, aplicación de bonificaciones y selección de modalidad (Inmediato vs. Fiado). Ejecuta descuento de stock, suma en caja (si corresponde), guarda costo histórico inmutable y emite comprobante con Deep Link (WhatsApp/Mail).
3. **Historial de Ventas (CUB 1.2):** Grilla con filtros. Permite liquidar fiados. Habilita acciones críticas bajo **ventana estricta de 15 minutos**: Anulación definitiva o Edición ágil (anula y repuebla el formulario).
4. **Cambios de Mercadería (CUB 1.4):** Operaciones vinculadas o aisladas. Si entra una "bolsa abierta", se convierte automáticamente a stock a granel. Toda diferencia a favor suma a la caja diaria. Habilita Anulación y Edición bajo ventana de 15 minutos.

## Fronteras del Sistema (Mocks)
Para esta etapa, los módulos satélites se integrarán de forma simulada:
- **Caja Diaria:** Operará por defecto en estado perpetuo "Abierta".
- **Estadísticas:** La métrica en la cabecera (RNF-GENERAL-05) renderizará información hardcodeada.
- **Usuarios:** Sin ABM. Usuarios semilla creados manualmente en la BD para probar el inicio de sesión de roles.
- **Inventario:** Sin ABM. Catálogo de productos semilla precargado para validar descuentos de stock reales.

---

# 3. Requerimientos Funcionales (RF)

## Módulo Ventas
- **RF-VENTAS-01 (Registro y Cálculo):** Soporte dual de cobro. Pago Inmediato (impacta caja, estado `COMPLETADA`) vs. Pago Diferido/Fiado (exige cliente, omite caja, estado `PENDIENTE_PAGO`). Congelamiento de costo unitario vigente por ítem.
- **RF-VENTAS-02 (Descuento Stock):** Salida de mercadería en tiempo real hacia inventario al confirmar operación (aplica a cobros y fiados).
- **RF-VENTAS-03 & 04 (Comprobantes y Envío):** Generación de ticket digital y construcción dinámica de enlaces profundos (Deep Linking) con `wa.me` / `mailto:` y texto precargado. **Toda venta (inmediata o fiado) genera un comprobante de forma instantánea.**
- **RF-VENTAS-05 & 06 (Anulación y Edición Venta):** Ventana innegociable de 15 min. La anulación ejecuta baja lógica y revierte impactos de caja/stock correspondientes. La edición ejecuta una anulación y repuebla la vista para recobro.
- **RF-VENTAS-08 (Cambios - Granel y Saldos):** El ingreso de devoluciones marcadas como "bolsa abierta" se suma estrictamente a la variante "granel" del producto. Diferencias económicas `>= 0` asientan ingresos en caja.
- **RF-VENTAS-09 & 10 (Anulación y Edición Cambio):** Ventana de 15 min. Reversión atómica de saldos de caja y reposición exacta (unidad o granel) según el formato devuelto originalmente.

## Generales
- **RF-GENERAL-01 (Inmutabilidad):** Prohibido el borrado físico (`DELETE` SQL). Toda eliminación exige baja lógica (cambio de estado) para mantener integridad referencial.
- **RF-GENERAL-02 (Búsqueda):** Filtros y buscadores estandarizados en todos los listados de recursos.
- **RF-GENERAL-03 (Reset):** Mecanismo ágil de limpieza de formularios para operar consecutivamente.

---

# 4. Requerimientos No Funcionales (RNF)

## Rendimiento y Transaccionalidad
- **RNF-GENERAL-06 (Atomicidad de BD):** Operaciones cruzadas (ej. Venta + Stock + Caja) se ejecutan mediante transacciones atómicas (`prisma.$transaction()`). Rollback inmediato ante fallo parcial.
- **RNF-VENTAS-01 (Latencia):** Tiempo máximo de 4 segundos entre el tap en "Confirmar Venta" y la emisión del ticket bajo red 4G.

## UI / UX
- **RNF-GENERAL-01 & 04 (Diseño):** Enfoque *Mobile-First*. Soporte nativo para alternar *Modo Claro/Oscuro*.
- **RNF-VENTAS-02 (Interacciones):** Interfaz optimizada para reducción de taps/clics durante el mostrador.
- **RNF-GENERAL-05 (Layout):** Reserva espacial obligatoria en cabeceras de módulos para inyección de un KPI rápido estático.

## Optimización de Red
- **RNF-GENERAL-02 (Payload Liviano):** Arquitectura minimalista sin carga de *assets* pesados ni animaciones complejas (diseñado para local sin Wi-Fi).
- **RNF-GENERAL-03 (Service Worker):** Implementación de caché de PWA (App Shell) para compilación instantánea de vistas estáticas en aperturas recurrentes.

---

# 5. Casos de Uso Base (CUB) - Demo

## CUB 1.1: Registrar Venta
**Precondiciones:** Sesión validada (Vendedor, Gestor, Gerente), Caja Diaria en estado "Abierta".
**Flujo de Ejecución:**
1. **Configuración inicial:** Selección de origen (mostrador/digital) y modalidad de entrega (retiro/envío).
2. **Carga de carrito:** Búsqueda manual de artículos. Validación de disponibilidad de stock (alerta visual si excede).
3. **Modalidad de Cobro:**
   - **Pago Inmediato:** Selección de medios de pago, aplicación automática de recargos/descuentos o bonificaciones manuales.
   - **Pago Diferido (Fiado):** Selección exclusiva de "Fiado", registro obligatorio del cliente deudor.
4. **Confirmación Transaccional:**
   - Congela de forma inmutable el costo unitario vigente.
   - Descuenta stock en tiempo real.
   - Si es **Inmediato**: Impacta ingreso en caja, estado `COMPLETADA`, emite ticket digital de pago.
   - Si es **Fiado**: Omite impacto en caja, estado `PENDIENTE_PAGO` y emite ticket de deuda. *(Nota BD: `numero_comprobante` SIEMPRE se genera al instante).*
5. **Acciones Post-Confirmación:** Envío de comprobante vía Deep Link (WhatsApp/Mail) o ejecución de Edición Ágil (anulación interna y repoblado del formulario).
**Postcondiciones:** Venta registrada, comprobante emitido, stock descontado, caja actualizada (solo si es pago inmediato).

## CUB 1.2: Consultar Historial de Ventas
**Precondiciones:** Sesión validada, Caja Diaria "Abierta" (obligatorio para liquidar o anular ventas con cobro).
**Flujo de Ejecución:**
1. **Visualización:** Grilla interactiva con paginación (Server-Side, latencia <2s) y filtrado (fecha, producto, estado).
2. **Edición Ágil (Regla 15 min):** El servidor valida la ventana temporal. Si aprueba, ejecuta anulación lógica, revierte caja/stock y redirige a CUB 1.1 repoblando el formulario. Si expiran los 15 minutos, el servidor debe devolver Error 403 Forbidden.
 3. **Anulación Definitiva (Regla 15 min):** Ejecuta baja lógica de la venta y revierte impactos atómicamente. Si el ticket era un fiado, solo repone stock. Si expiran los 15 minutos, el servidor debe devolver Error 403 Forbidden.
4. **Liquidar Fiado (Sin límite de tiempo):** 
   - Busca ticket en estado `PENDIENTE_PAGO`.
   - Selecciona medio de pago actual (aplica recargos/descuentos del día).
   - Inyecta el ingreso en la Caja Diaria abierta del día en curso.
   - Pasa la transacción a `COMPLETADA` y emite recibo/comprobante de pago.
**Postcondiciones:** Estado de venta actualizado (`ANULADA` o `COMPLETADA`), impactos monetarios y de inventario ajustados.

## CUB 1.4: Administrar Cambios de Mercadería
**Precondiciones:** Sesión validada, Caja Diaria "Abierta".
**Flujo de Ejecución:**
1. **Alta de Cambio:** 
   - Opcional: Vincular ticket original mediante búsqueda.
   - Ingresar artículo devuelto (entrante) y nuevo artículo (saliente).
2. **Conversión a Granel:** Interfaz provee control (toggle/checkbox) para "bolsa abierta". Si se activa, el peso se convierte y suma automáticamente a la variante "granel" del producto devuelto. Si es cerrada, repone unidad entera.
3. **Cálculo de Saldos (Reactivo en UI):** Si la diferencia de precios arroja saldo a favor del establecimiento, se asienta como ingreso en la caja diaria.
4. **Anulación / Edición (Regla 15 min):** Operaciones atómicas (`prisma.$transaction()`). Revierte con exactitud el stock fraccionado o unitario ingresado y resta la caja.
**Postcondiciones:** Stocks actualizados (unidad/granel), ingresos extra asentados en caja.

---

# 6. Stack Tecnológico, Infraestructura e IA

## Frontend (PWA)
- **Base:** React + Vite, TypeScript.
- **Estilos:** Tailwind CSS (maquetación en archivo único).
- **Enrutamiento:** React Router (protección de rutas por roles).
- **Integraciones:** URL Schemes nativos (`wa.me` / `mailto:`) para envíos sin APIs externas de terceros.

## Backend (API REST)
- **Entorno & Lenguaje:** Node.js, Express, TypeScript (unificando stack con Frontend).
- **Validación Transversal:** Zod (inferencia de tipos automáticos a partir de contratos SDD).
- **Seguridad:** JWT (sesiones en headers) y bcryptjs (hashing de contraseñas).

## Capa de Datos (PostgreSQL)
- **Motor:** PostgreSQL alojado en **Supabase** (DB as a Service).
- **ORM:** Prisma (encargado de migraciones y de garantizar transacciones atómicas seguras mediante `prisma.$transaction()`).

## Despliegue (Deploy) & Repositorios
- **Estructura Git:** Polyrepo (1. `petflow-frontend`, 2. `petflow-backend`, 3. `petflow-workspace` para docs y configuraciones de VSCode).
- **Cloudflare Pages:** Hosting de la PWA del Frontend.
- **Render (Free Tier):** Hosting de la API Backend (con script en Cron-job.org haciendo ping cada 10 min para evitar hibernación / *cold start*).
- **Stack IA:** Claude Pro (para generación de código) y Gemini Pro (para planificación). IDE: Antigravity.

---

# 7. Paradigmas y Patrones Arquitectónicos

## Macro-Arquitectura: Cliente-Servidor Desacoplada
Separación física absoluta. Comunicación exclusiva mediante intercambio JSON vía peticiones HTTP.

## Micro-Arquitectura Frontend: Funcional + Custom Hooks
- **Paradigma Funcional:** Prohibidas las mutaciones y uso de clases. Uso obligatorio de inmutabilidad (ej. `setEstado([...viejo, nuevo])`).
- **Segregación Vista/Lógica:** 
  - *Vistas:* Componentes "tontos" enfocados solo en renderizar Tailwind.
  - *Lógica:* Archivos aislados (Custom Hooks) manejan estados, peticiones y errores.

## Micro-Arquitectura Backend: POO Estructural + Capas (CSD)
- **Paradigma POO Estructural:** Moldes de datos puramente tipados (`interface` / `type`). Sin instanciación con clases ni `new`. Funciones externas operan sobre los objetos.
- **Patrón CSD:**
  1. *Controladores:* Reciben petición HTTP y validan esquemas con Zod. Sin lógica comercial.
  2. *Servicios:* Centralizan las reglas de negocio, cálculos y flujo transaccional.
  3. *Datos:* Prisma ORM ejecutando consultas a PostgreSQL.

## Transversal: Screaming Architecture (Feature-Sliced)
Agrupación física de carpetas por **módulo de negocio**, NO por tecnología. 
- *Back:* `/modules/ventas/` (incluye su controller, service, schema).
- *Front:* `/features/ventas/` (incluye sus hooks, vistas, componentes).

---

# 8. Estándares de Trabajo y Nomenclaturas

## 8.1. Gestión de Tareas (Issues en GitHub)

**Formato estricto para tableros:** `[ÁMBITO] Tipo: Acción`  
*(Regla: el valor de `Acción` debe coincidir exactamente con el nombre de la columna "Acción" del Cuadro de Acciones).*

* **`[FRONT] UI: Alta Venta`**  Se crea en el repositorio `petflow-frontend`.
* **`[BACK] API: Alta Venta`**  Se crea en el repositorio `petflow-backend`.
* **`[DISEÑO] UX: Alta Venta`**  Draft Issue en el tablero de GitHub Projects o Issue en `petflow-workspace`.
* **`[DOCS] SDD: Alta Venta`**  Se crea en `petflow-workspace` (redacción de contratos o diagramas).
* **`[QA] E2E: Alta Venta`**  Se crea en `petflow-workspace` (requiere los PRs aprobados de Front y Back para darse por cerrada).

## 8.2. Control de Versiones (Git)
**Ramas (kebab-case):** `tipo/id-nombre-tarea` (Ej: `feat/1-alta-venta`, `fix/calculo-total`).
**Commits (Convencionales):** `tipo(alcance): accion en imperativo y minuscula` (Ej: `feat(ventas): crear custom hook de carrito`).
- *Tipos válidos:* `feat`, `fix`, `style` (comentarios/formateo), `refactor`, `docs`, `chore`.
- *Alcances típicos:* `ventas`, `caja`, `auth`, `ui`, `api`, `db`.

## 8.3. Reglas de Código Transversales
**Regla de Oro:** Cero abreviaciones. Las variables deben usar palabras completas en español para garantizar claridad semántica (Ej: `cantidadProducto`, no `cantProd`).
- **Caracteres Permitidos (ASCII):** Prohibido usar tildes, diéresis o la letra "ñ" en el código fuente (nombres de variables, funciones, archivos, bases de datos o ramas). Escribe en español limitándote al alfabeto inglés. Para palabras con "ñ", utiliza sinónimos semánticos en lugar de reemplazarla por "n" (ej. prohibido usar `año` o `anio`, usa `periodo` o `ejercicio`; en lugar de `tamano`, usa `dimension` o `medida`).

| Convención | Aplicación en el Código | Ejemplos |
| :--- | :--- | :--- |
| **`camelCase`** | Variables, Propiedades, Funciones (verbo inicial), Hooks (prefijo `use`), Campos en Prisma. | `montoTotal`, `calcularVuelto()`, `useRegistrarVenta()`, `idVenta` |
| **`PascalCase`** | Componentes React (`.tsx`), Interfaces y Types TS, Modelos Prisma. | `<GrillaVentas />`, `interface Venta`, `model Venta` |
| **`kebab-case`** | Carpetas, Archivos generales/Hooks, Rutas API REST. | `/cambios-mercaderia`, `use-registrar-venta.ts`, `/api/ventas` |
| **`snake_case`** | Tablas y Columnas en PostgreSQL (usar decorador `@@map` en Prisma). | `id_venta`, `@@map("detalles_ventas")` |
| **`UPPER_SNAKE_CASE`** | Constantes globales y Enums (Estados). | `LIMITE_MINUTOS`, `PENDIENTE_PAGO` |

---

# 9. Metodologías de Desarrollo
El proyecto opera bajo la fusión de 3 metodologías orientadas a la eficiencia y el uso de Inteligencia Artificial:

1. **Shape Up (Alcance Variable):** El tiempo de entrega es innegociable. Ante bloqueos o falta de tiempo, se descartan los detalles técnicos complejos en favor de entregar la versión "esencial" operativa.
2. **Kanban (Límite WIP = 1):** Prohibido tener más de una (1) tarea en la columna "En Progreso" por desarrollador. Si hay exámenes o bloqueos externos, el tablero se pausa sin romper ciclos de sprint.
3. **SDD (Spec-Driven Development):** 
   - Prohibido codificar directamente. Se debe diseñar primero un **Contrato de Datos** (tipos/interfaces de entrada y salida en texto plano).
   - Este contrato se inyecta en el **Prompt Maestro SDD** junto con el Caso de Uso Base, delegando a la IA (Claude) la planificación lógica y la posterior generación de código tipado, garantizando que el Frontend y el Backend se ensamblen a la perfección sin tiempos de espera.

---

# 10. Herramienta de Ejecución: Prompt Maestro SDD

Este molde estandarizado se debe copiar, completar con los datos del Caso de Uso Base (CUB) y pegar en Claude Pro al iniciar cualquier nueva funcionalidad. Garantiza que la IA respete la arquitectura y proponga los contratos TypeScript antes de escribir una sola línea de código funcional.

***

Actúa como un Desarrollador Senior experto en [REACT/TAILWIND o NODE/PRISMA].
Trabajamos bajo la metodología Spec-Driven Development (SDD) y Screaming Architecture.

# 1. CONTEXTO
- **Módulo:** [Ej: Ventas / Inventario]
- **Acción:** [Ej: Componente de Búsqueda de Productos]

# 2. EL CONTRATO (ESPECIFICACIÓN EN TEXTO PLANO)
A partir de estas directivas, deberás inferir y generar las estructuras de datos o tipos de TypeScript necesarios.
- **Datos que recibe (Input / Estado inicial):** 
  [Ej: Debe recibir el texto que tipea el usuario en el buscador y el filtro de categoría seleccionada].
- **Datos que devuelve (Output / Resultado):** 
  [Ej: Debe devolver una lista de productos que coincidan, mostrando solo su nombre, precio actual y stock físico].

# 3. REGLAS DE NEGOCIO Y SUB-ACCIONES
El código deberá cumplir estrictamente con esto:
1. [Ej: La búsqueda debe ignorar mayúsculas y minúsculas].
2. [Ej: Si el texto está vacío, no debe mostrar ningún resultado].

# 4. REGLAS DE ARQUITECTURA Y DOCUMENTACIÓN
- **Arquitectura:** [Ej Front: La lógica debe ir en un Custom Hook, la vista es "tonta". / Ej Back: El controlador valida, el servicio opera en BD].
- **Seccionado de archivos:** Si el código es extenso, debes dividir el archivo en bloques lógicos utilizando comentarios separadores claros (ej. `// --- ESTADOS ---`, `// --- FUNCIONES AUXILIARES ---`, `// --- RENDERIZADO ---`).
- **Comentarios en línea:** Agrega comentarios breves y descriptivos justo arriba de los bloques de lógica explicando el *qué* y el *por qué*, evitando comentar lo que ya es obvio por el nombre de las variables.
- **Documentación de Tipos:** Utiliza el estándar JSDoc (`/** ... */`) para documentar brevemente las interfaces, types y propiedades principales que generes.

# 5. INSTRUCCIÓN DE EJECUCIÓN (FRENO DE MANO)
NO escribas el código funcional todavía.
De acuerdo con el flujo SDD, tu única tarea en esta etapa es:
1. Analizar el contrato y **proponer cómo se llamarán y estructurarán los tipos de datos (TypeScript)** que vas a crear.
2. Escribir un **Plan Lógico** breve de cómo vas a estructurar la solución.
3. Listar las **Tareas Técnicas** paso a paso.

Espero tu Plan y Tareas para revisarlos. Solo cuando yo te dé el "OK" pasaremos a la etapa de Implementación.

---
# 11. Cuadro de Acciones (Hoja de Ruta Maestra)

> **Directiva de Jerarquía y Fuente de Verdad:** 
> Esta tabla dictamina el 100% del alcance de programación. Ninguna funcionalidad fuera de esta lista debe ser desarrollada. Las IAs (Claude/Gemini) deben utilizar estas sub-acciones exactas como guía de pasos para los planes lógicos (SDD) y el desglose de tareas técnicas.

| Módulo | Acción | Sub-acciones (Checklist Técnico) |
| :--- | :--- | :--- |
| **Seguridad** | Iniciar/cerrar sesión | • Validar credenciales (bcrypt/JWT).<br>• Inyectar rol y usuario en contexto global.<br>• Restringir rutas (React Router).<br>• Destruir sesión (Logout). |
| **Ventas** | Alta Venta | • Seleccionar canal y entrega.<br>• Buscar/añadir/modificar/borrar productos del carrito.<br>• Validar stock en memoria.<br>• Seleccionar modalidad (Inmediato vs. Fiado).<br>• Aplicar descuentos manuales o automáticos.<br>• Confirmar y ejecutar transacción en BD.<br>• Visualizar y enviar ticket (Deep Linking). |
| **Ventas** | Modificación Venta | • Validar límite temporal (15 min) en Servidor.<br>• Anular lógicamente transacción original.<br>• Reposición atómica de stock (y revertir caja solo si la venta era Inmediata).<br>• Repoblar formulario inicial con datos originales. |
| **Ventas** | Baja Venta | • Validar límite temporal (15 min).<br>• Confirmación por modal (UI).<br>• Baja lógica de registro.<br>• Reposición atómica de stock (y revertir caja solo si la venta era Inmediata). |
| **Ventas** | Consultar Historial | • Cargar grilla (SSR paginado).<br>• Aplicar filtros (fecha, producto, estado). |
| **Ventas** | Liquidar Fiado (Cobro) | • Seleccionar fiado en estado `PENDIENTE_PAGO`.<br>• Seleccionar medio de pago actual.<br>• Registrar ingreso en Caja activa, pasar a `COMPLETADA` y emitir comprobante. |
| **Ventas** | Consultar Venta | • Mostrar vista de detalle del comprobante/transacción. |
| **Ventas** | Alta Cambio | • Buscador para vinculación de ticket original.<br>• Seleccionar artículo entrante y saliente.<br>• Checkbox "Bolsa abierta" para conversión a granel.<br>• Calcular e ingresar diferencia a favor en caja.<br>• Confirmar transacción. |
| **Ventas** | Modificación Cambio | • Validar límite temporal (15 min).<br>• Anular cambio original revirtiendo saldo y conversiones granel.<br>• Repoblar vista de cambios. |
| **Ventas** | Baja Cambio | • Validar límite temporal (15 min).<br>• Baja lógica y reversión atómica de saldos/stock. |
| **Ventas** | Historial Cambios | • Cargar grilla de cambios con filtros. |
