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
1. **Ventas:** Procesamiento ágil en mostrador, descuentos dinámicos, gestión de fiados y deudores, cambios de mercadería y envíos de comprobantes vía WhatsApp (Deep Links).
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
5. **Fiados y Deudores (CUB 1.5):** Alta, búsqueda y edición de deudores, requisito previo para fiar.

## Fronteras del Sistema (Mocks)
Para esta etapa, los módulos satélites se integrarán de forma simulada:
- **Caja Diaria:** Operará por defecto en estado perpetuo "Abierta".
- **Estadísticas:** La métrica en la cabecera (RNF-GENERAL-05) renderizará información hardcodeada.
- **Usuarios:** Sin ABM. Usuarios semilla creados manualmente en la BD para probar el inicio de sesión de roles.
- **Inventario:** Sin ABM. Catálogo de productos semilla precargado para validar descuentos de stock reales.

---

# 3. Requerimientos Funcionales (RF)

## Módulo Ventas
- **RF-VENTAS-01 (Registro y Cálculo):** Soporte dual de cobro. Pago Inmediato (impacta caja, estado `COMPLETADA`) vs. Pago Diferido/Fiado (exige un deudor registrado, omite caja, estado `PENDIENTE_PAGO`). Congelamiento de costo unitario vigente por ítem. Toda bonificación manual registra su motivo: es obligatorio cuando hay descuento manual, admite hasta 200 caracteres, no se admite sin descuento y no aplica al cobro de un fiado. En pagos en efectivo se registra el monto recibido y el servidor calcula el vuelto; el ingreso a caja es siempre el monto final.
- **RF-VENTAS-02 (Descuento Stock):** Salida de mercadería en tiempo real hacia inventario al confirmar operación (aplica a cobros y fiados).
- **RF-VENTAS-03 & 04 (Comprobantes y Envío):** Generación de ticket digital y construcción dinámica de enlaces profundos (Deep Linking) con `wa.me` / `mailto:` y texto precargado. **Toda venta (inmediata o fiado) genera un comprobante de forma instantánea.** Al liquidar un fiado se conserva el número de comprobante original: el comprobante pasa a ser un recibo de pago.
- **RF-VENTAS-05 & 06 (Anulación y Edición Venta):** Ventana innegociable de 15 min. La anulación ejecuta baja lógica y revierte impactos de caja/stock correspondientes. La edición ejecuta una anulación y repuebla la vista para recobro. Toda anulación y toda edición registra un motivo, el usuario y la fecha y hora. El motivo se ingresa manualmente: es obligatorio y de 1 a 200 caracteres (sin contar espacios de los extremos). El usuario (tomado del token, nunca del body) y la fecha y hora los asigna el servidor automáticamente. Los tres se registran en la venta anulada; en la edición corresponden a la anulación interna que la origina. Sin motivo, o con uno fuera de ese largo, el servidor responde 400.
- **RF-VENTAS-08 (Cambios - Granel y Saldos):** El ingreso de devoluciones marcadas como "bolsa abierta" se suma estrictamente a la variante "granel" del producto. Diferencias económicas `>= 0` asientan ingresos en caja.
- **RF-VENTAS-09 & 10 (Anulación y Edición Cambio):** Ventana de 15 min. Reversión atómica de saldos de caja y reposición exacta (unidad o granel) según el formato devuelto originalmente.
- **RF-VENTAS-11 (Deudores):** Un deudor es la persona que tiene o tuvo un fiado. Se identifica por su DNI, que es único, y registra nombre y apellido, teléfono (con código de país) y DNI, los tres obligatorios. Se puede dar de alta, buscar y editar; no se da de baja. Un fiado solo se registra asociado a un deudor existente: sin deudor, el servidor rechaza la venta con 400; con un deudor inexistente, responde 404 sin descontar stock ni crear nada. Una venta de contado no admite deudor: si lo trae, el servidor responde 400. En esta versión no se ingresa texto libre de cliente; el campo `clienteReferencia` de las ventas anteriores queda solo de lectura, y esas ventas no tienen deudor.

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

## Seguridad y Datos Personales
- **RNF-VENTAS-03 (Datos personales):** El DNI y el teléfono de los deudores solo se entregan a usuarios con sesión validada y no se incluyen en el texto de los mensajes de error. El historial entrega del deudor solo su id y nombre. El DNI y el teléfono salen en el detalle de una venta, en la respuesta del alta de la venta y en el módulo de deudores.
- **RNF-GENERAL-07 (Seguridad por filas):** La base de datos mantiene activada la seguridad por filas (RLS) en todas las tablas del esquema `public`. El backend se conecta con un rol que la omite, de modo que los datos solo se acceden a través de la API. Toda tabla nueva se crea con RLS activado.

---

# 5. Casos de Uso Base (CUB) - Demo

## CUB 1.1: Registrar Venta
**Precondiciones:** Sesión validada (Vendedor, Gestor, Gerente), Caja Diaria en estado "Abierta".
**Flujo de Ejecución:**
1. **Configuración inicial:** Selección de origen (mostrador/digital) y modalidad de entrega (retiro/envío).
2. **Carga de carrito:** Búsqueda manual de artículos. Validación de disponibilidad de stock (alerta visual si excede).
3. **Modalidad de Cobro:**
   - **Pago Inmediato:** Selección de medios de pago, aplicación automática de recargos/descuentos o bonificaciones manuales. En efectivo, el front informa el monto recibido (mayor o igual al monto final) y el servidor calcula y registra el vuelto; sin monto recibido no se registra vuelto.
   - **Pago Diferido (Fiado):** Selección exclusiva de "Fiado" y selección de un deudor existente o alta de uno nuevo (nombre y apellido, teléfono y DNI, todos obligatorios). Sin deudor no se puede fiar. El front envía el deudor elegido y la venta queda asociada a él. En un pago inmediato no se envía deudor.
   - **Bonificación manual (ambas modalidades):** Toda bonificación manual registra su motivo. Si hay descuento manual, el motivo es obligatorio (de 1 a 200 caracteres, sin contar espacios de los extremos); sin descuento manual no se admite motivo.
4. **Confirmación Transaccional:**
   - Congela de forma inmutable el costo unitario vigente.
   - Descuenta stock en tiempo real.
   - Si es **Inmediato**: Impacta ingreso en caja, estado `COMPLETADA`, emite ticket digital de pago.
   - Si es **Fiado**: Omite impacto en caja, estado `PENDIENTE_PAGO`, asociado al deudor, y emite ticket de deuda. *(Nota BD: `numero_comprobante` SIEMPRE se genera al instante).*
5. **Acciones Post-Confirmación:** Envío de comprobante vía Deep Link (WhatsApp/Mail) o ejecución de Edición Ágil (anulación interna y repoblado del formulario).
**Postcondiciones:** Venta registrada, comprobante emitido, stock descontado, caja actualizada (solo si es pago inmediato).

## CUB 1.2: Consultar Historial de Ventas
**Precondiciones:** Sesión validada, Caja Diaria "Abierta" (obligatorio para liquidar o anular ventas con cobro).
**Flujo de Ejecución:**
1. **Visualización:** Grilla interactiva con paginación (Server-Side, latencia <2s) y filtros (fecha, producto, estado, origen, medio de pago, nombre del deudor y comprobante). La grilla muestra el deudor solo con su id y nombre; el detalle de la venta muestra el deudor completo (nombre, teléfono y DNI). Las ventas de contado y las anteriores al vínculo con deudores no tienen deudor.
2. **Edición Ágil (Regla 15 min):** Antes de ejecutarse, el front pide confirmación y un motivo obligatorio (de 1 a 200 caracteres, sin contar espacios de los extremos). El servidor valida el motivo (sin motivo o fuera de largo responde 400) y la ventana temporal. Si aprueba, ejecuta anulación lógica registrando en la venta anulada el motivo, el usuario del token y la fecha y hora (estos dos últimos asignados automáticamente), revierte caja/stock y redirige a CUB 1.1 repoblando el formulario. Si expiran los 15 minutos, el servidor debe devolver Error 403 Forbidden.
 3. **Anulación Definitiva (Regla 15 min):** Antes de ejecutarse, el front pide confirmación y un motivo obligatorio (de 1 a 200 caracteres, sin contar espacios de los extremos). El servidor valida el motivo (sin motivo o fuera de largo responde 400) y ejecuta baja lógica de la venta registrando el motivo, el usuario del token y la fecha y hora (estos dos últimos asignados automáticamente), y revierte impactos atómicamente. Si el ticket era un fiado, solo repone stock. Si expiran los 15 minutos, el servidor debe devolver Error 403 Forbidden.
4. **Liquidar Fiado (Sin límite de tiempo):** 
   - Busca ticket en estado `PENDIENTE_PAGO`. Si la venta no existe responde 404; si no está en `PENDIENTE_PAGO`, 409.
   - Selecciona los medios de pago actuales. El servidor calcula el total aplicando los recargos/descuentos del día sobre el importe adeudado (subtotal bruto menos descuento manual); el cobro es total, sin cobros parciales.
   - En efectivo, el front informa el monto recibido y el servidor calcula y registra el vuelto.
   - Inyecta el ingreso (monto final, nunca el recibido) en la Caja Diaria abierta del día en curso; si no hay caja abierta, responde 409.
   - Pasa la transacción a `COMPLETADA`, registra la fecha de pago y emite el recibo de pago conservando el mismo número de comprobante del ticket de deuda.
   - Un fiado no puede cobrarse dos veces: ante cobros simultáneos, uno falla con 409.
**Postcondiciones:** Estado de venta actualizado (`ANULADA` o `COMPLETADA`), impactos monetarios y de inventario ajustados. Toda venta `ANULADA` conserva el motivo, el usuario y la fecha y hora de su anulación.

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

## CUB 1.5: Administrar Deudores
**Precondiciones:** Sesión validada (Vendedor, Gestor, Gerente). No exige Caja Diaria abierta.
**Flujo de Ejecución:**
1. **Alta:** nombre y apellido, teléfono (con código de país) y DNI, los tres obligatorios. El DNI es único: si ya existe, el servidor responde 409 con los datos del deudor existente y el front lo selecciona. El alta también está disponible dentro de CUB 1.1 al elegir "Fiado".
2. **Búsqueda:** por nombre o DNI, parcial y sin distinguir mayúsculas.
3. **Edición:** se pueden modificar los tres datos. Si el DNI nuevo pertenece a otro deudor, el servidor responde 409. No hay baja de deudores.
4. **Formatos (Alta y Edición):** Nombre y apellido: se recorta en los extremos y tiene de 1 a 100 caracteres. DNI: se recorta, solo dígitos, 7 u 8, sin puntos. Teléfono: admite espacios, guiones y un `+` solo al inicio; se guarda solo con dígitos, de 11 a 15, con código de país (no se exige el prefijo 54). Los mensajes de error nunca incluyen DNI, teléfono ni nombre (RNF-VENTAS-03).
**Postcondiciones:** Deudor registrado o actualizado, sin efecto en caja ni en stock.

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

**Formato estricto para tableros:** `[ÁMBITO] Módulo: verbo + resultado concreto`

- **Ámbito:** `FRONT`, `BACK`, `DOCS`, `DISEÑO` o `QA`. Indica dónde vive el trabajo (ver tabla).
- **Módulo:** el área del producto que cambia (Ventas, Inventario, Caja, Gastos, Sesión, etc.). Aparece siempre.
- **Verbo + resultado concreto:** verbo en infinitivo (Agregar, Corregir, Conectar, Reemplazar, Mostrar, Alinear, Eliminar; no es una lista cerrada) que dice qué queda distinto al terminar. No existe un campo `Tipo`.
- **Regla de oro:** el título debe poder leerse como *"al terminar, ___"*. Nada genérico como `Alta Venta`.
- **Bug o trabajo nuevo:** se distingue con los labels de GitHub (`bug`, `feature`), no en el título.

| Ámbito | Clase de trabajo | Repositorio |
| :--- | :--- | :--- |
| `[FRONT]` | Interfaz, hooks, llamadas HTTP y configuración del cliente. | `petflow-frontend` |
| `[BACK]` | Endpoints, esquema, migraciones, seed y configuración del servidor. La base solo se opera con Prisma, por eso el trabajo de DB vive en `petflow-backend`. | `petflow-backend` |
| `[DOCS]` | Contratos, planes, diagramas y documentación del workspace. | `petflow-workspace` |
| `[DISEÑO]` | Diseño de experiencia. | Draft Issue en GitHub Projects o Issue en `petflow-workspace` |
| `[QA]` | Pruebas de punta a punta. Se cierra cuando están aprobados los PRs de Front y Back. | `petflow-workspace` |

**Ejemplos:**

| Título de la issue | Al terminar... |
| :--- | :--- |
| `[FRONT] Ventas: Mostrar importes con 2 decimales (ej. $1.500,00)` | los importes se ven con 2 decimales. |
| `[FRONT] Ventas: Reemplazar motivo de descuento por un select` | el motivo de descuento se elige de un select. |
| `[FRONT] Ventas: Alinear a la derecha totales y columnas numéricas` | los totales y las columnas numéricas quedan alineados a la derecha. |
| `[FRONT] Ventas: Corregir formato de importes, motivo de descuento y alineación numérica` | las tres correcciones quedan hechas (issue agrupada, con checklist en el cuerpo). |
| `[FRONT] Ventas: Conectar historial al back real` | el historial muestra datos reales del back. |
| `[BACK] Inventario: Agregar endpoint para descontar stock al vender` | existe un endpoint que descuenta el stock de una venta. |

**Issues que corrigen varias cosas:**
- El título usa una frase que agrupe (ver el cuarto ejemplo) y el cuerpo lleva una checklist con un ítem por cosa.
- Se mantiene **una issue = una rama = un PR**: la checklist se resuelve toda en esa misma rama.
- Si las cosas son independientes entre sí, conviene separarlas en issues distintas.

**Reglas de issues y ramas:**
- **Una issue = una rama = un PR.** La maqueta y la conexión de una misma funcionalidad son issues distintas. Por ejemplo, la maqueta del historial se cierra al mergear la maqueta, y `[FRONT] Ventas: Conectar historial al back real` queda en Backlog o en "En espera" hasta que el back esté listo.
- Cada rama sale de `develop` y vuelve a `develop`. Al mergear se descarta y no se reutiliza.
- Una issue que espera algo externo (por ejemplo, un endpoint del back) va a la columna "En espera" y no ocupa "En Progreso" (ver sección 9).

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
2. **Kanban (Límite WIP = 1):** Prohibido tener más de una (1) tarea en la columna "En Progreso" por desarrollador. Las tareas que esperan algo externo (por ejemplo, un endpoint del back) van a la columna "En espera", que no cuenta para el WIP. Si hay exámenes o bloqueos externos, el tablero se pausa sin romper ciclos de sprint.
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
