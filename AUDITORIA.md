# Reporte de auditoría de documentación — Manager Library

**Repositorio:** `test-manager-library-3145556-docs`  
**Alcance:** completitud, consistencia interna, ausencia de bifurcaciones y ambigüedades.  
**Método:** comparación sistemática de `spec-python/` vs `spec-.net/`, cruce de historias (HU/CA), reglas (RN), modelo de datos, contrato HTTP (§2/§3), ADR y tasks.  
**Importante:** este reporte **no modifica** ningún archivo de la especificación. El repositorio se dejó en su estado original; únicamente se añade este `.md`.

---

## 1. Archivos revisados

- `README.md`
- `spec-python/`: `constitution.md`, `spec.md`, `plan.md`, `architecture.md`, `arquitectura-panomapica.md` → `arquitectura-panoramica.md`, `data-model.md`, `api-contract.md`, `tasks.md`, `adr/adr-001..004.md`
- `spec-.net/`: mismos archivos y `adr/`.
- Comparación archivo-por-archivo entre ambas versiones.

*(Nota: la ruta correcta del archivo es `arquitectura-panoramica.md`; se verificó su presencia y contenido.)*

---

## 2. Resultado general de bifurcación (python vs .NET)

No se encontró bifurcación de **negocio**. Las diferencias entre `spec-python/` y `spec-.net/` son únicamente tecnológicas:
lenguaje/runtime, framework HTTP, ORM+migraciones, hasher de claves, herramienta de pruebas/arquitectura y nombres de carpetas/módulos.

Archivos con diferencias **solo técnicas**:
- `constitution.md` (rutas `src/library/...` vs `src/Library/...`, Alembic vs EF Core, `import-linter` vs NetArchTest).
- `plan.md` (D-01..D-03, D-09, D-12).
- `architecture.md` (FastAPI vs ASP.NET, SQLAlchemy vs EF Core, ruta de composición).
- `arquitectura-panoramica.md` ("backend Python" vs "backend .NET").
- `tasks.md` (T-19: `import-linter` vs `NetArchTest`).
- `adr/*` son idénticos en ambas versiones.
- `spec.md`, `data-model.md`, `api-contract.md` son idénticos de byte a byte.

Por tanto, el riesgo de bifurcación de **negocio** se considera **cerrado** entre las dos versiones, aunque existen inconsistencias internas dentro del spec (sección 3).

---

## 3. Inconsistencias internas detectadas (sin corrección aplicada)

Cada hallazgo incluye **ruta** y **problema**. Las rutas se dan para `spec-python/`; el mismo problema replica en `spec-.net/` cuando se indica.

### H-01 · `DocumentNumber` prometido pero ausente en tarea y árbol
- **Ruta:** `spec-python/plan.md` (D-07); `spec-python/tasks.md` (T-03); `spec-python/architecture.md` (§2 árbol de carpetas).
- **Problema:** `plan.md` declara `Isbn`, `Quantity`, `DocumentNumber`, `DateRange` como objetos de valor. Pero `tasks.md` T-03 solo lista `Isbn`, `Quantity`, `DateRange` y el comentario del árbol en `architecture.md` solo lista `Isbn, Quantity, DateRange`. No se decide de forma unívoca si `DocumentNumber` es objeto de valor (con validación) o un `string` directo del `Member`. (Replica en `.net`.)

### H-02 · ADR-002 exige "testigo de versión" que el modelo de datos no documenta
- **Ruta:** `spec-python/adr/adr-002-concurrencia-optimista.md` (decisión); `spec-python/data-model.md` (§3 tabla `book`, §4 restricciones); `spec-python/tasks.md` (T-13).
- **Problema:** el ADR afirma que "el libro lleva un testigo de versión" y T-13 menciona "testigo de versión", pero `data-model.md` §3 no lista ninguna columna de versión en `book` y §4 no asigna marca (motor/solo aplicación/pendiente) a ese testigo. Un implementador no sabe si usar columna entera, `xmin` de Postgres, o bloqueo pesimista. Es el principal gap "ADR vs modelo".

### H-03 · Endpoint de portada GET no listado en la tabla de §3.4
- **Ruta:** `spec-python/api-contract.md` §3.4 (tabla Libros) y texto posterior ("`cover_key` es una clave opaca: para pintar la portada se usa `GET /books/{id}/cover`…").
- **Problema:** la tabla de endpoints de Libros omite `GET /books/{id}/cover`, aunque el texto lo asume existente y define su comportamiento (binario o 404). Fuente de verdad fragmentada.

### H-04 · Edición de libro (`PUT`) no aclara inmutabilidad del ISBN
- **Ruta:** `spec-python/api-contract.md` §3.4 (nota "`POST /books` (petición)…"); `spec-python/spec.md` HU-03.2 ("ISBN … único entre libros activos"); `data-model.md` (edición no especifica).
- **Problema:** la nota dice que `PUT` "reemplaza título, autor, categoría y total", por omisión deja ambiguo si el ISBN se puede cambiar. Con HU-03.2 (unicidad) y ADR-003 (identidad del libro), no se explicita si el ISBN es inmutable.

### H-05 · Rol `admin` por API: código de error no mapeado
- **Ruta:** `spec-python/spec.md` HU-02.3 ("Pedir el rol `admin` por la API responde 422"); `spec-python/api-contract.md` §2 (códigos de error) y §3.2.
- **Problema:** la tabla de códigos de §2 define `FORBIDDEN_ROLE` (403, CA-01.3) y `VALIDATION_ERROR` (422). HU-02.3 exige 422 sin acotar si se usa `VALIDATION_ERROR` o un código específico. No existe código dedicado (p. ej. `INVALID_ROLE`). El implementador debe inferir.

### H-06 · `days_late` sin tipo ni fórmula
- **Ruta:** `spec-python/api-contract.md` §3.6 ("`GET /loans/overdue` → `items`: `{ …, "days_late" }`"); `spec-python/spec.md` HU-09.
- **Problema:** no se indica si `days_late` es entero o decimal, ni si se truncan/ceil los días, ni la base exacta del cálculo (`due_at` → ahora). Permite implementaciones divergentes.

### H-07 · Rango de fechas de préstamos: inclusividad y alcance no fijados
- **Ruta:** `spec-python/api-contract.md` §3.6 ("Lista: `from`, `to`, `status`, `member_id`, `page`, `size`") y §3.7 (reporte: "ambos inclusive").
- **Problema:** para `GET /loans` no se dice si `from`/`to` son inclusivos ni si filtran por día UTC sobre `loaned_at` o por instante exacto. El reporte sí dice "ambos inclusive" (fechas), pero no la conversión a instantes UTC (00:00 a 23:59:59). Posible diferencia de un día entre implementaciones.

### H-08 · Paginación de vencidos no especificada
- **Ruta:** `spec-python/api-contract.md` §3.6 ("`GET /loans/overdue` → `items`:…"); `spec-python/spec.md` HU-09.
- **Problema:** la respuesta usa la key `items`, pero el contrato no dice si se pagina (`page`/`size`/`total` como en el resto) ni si devuelve todo el listado. HU-09 tampoco lo aclara.

### H-09 · Límite de 3 libros por socio: conteo en operación concurrente
- **Ruta:** `spec-python/spec.md` CA-06.4 (`422 con el conteo actual`); `spec-python/data-model.md` marcas (solo dominio).
- **Problema:** no se explicita si la validación debe sumar los libros solicitados en la misma operación (si un socio tiene 2 sin devolver y pide 2, falla por 4>3) o solo verificar el conteo previo. El mensaje "con el conteo actual" sugiere incluir solo lo previo, pero es ambiguo.

### H-10 · `member.active_loans` en detalle: definición de "sin devolver"
- **Ruta:** `spec-python/api-contract.md` §3.5 (`GET /members/{id}` con `active_loans` (conteo sin devolver)).
- **Problema:** "sin devolver" no distingue ítems vencidos de no vencidos ni aclara si cuenta ítems o préstamos. Menor, pero mejorable.

### H-11 · Límite de ejemplares y `PUT` de total: ajuste de disponibles
- **Ruta:** `spec-python/spec.md` CA-03.3; `spec-python/data-model.md` `ck_book_available_range`.
- **Problema:** CA-03.3 dice que al cambiar el total se ajustan los disponibles "en la misma diferencia" y "nunca deja disponibles < 0" (422). No se define si aplicar el delta al total y a los disponibles en este orden ante concurrencia, y si un total menor a los actualmente prestados (available=0 pero total nuevo < prestados) debe fallar. El `CHECK` solo acota `available <= total`, pero la regla de negocio está incompleta para el caso "reducir total por debajo de prestados".

### H-12 · Marcas "motor / solo dominio / pendiente" incompletas
- **Ruta:** `spec-python/data-model.md` §4; `spec-python/constitution.md` Art. VI.
- **Problema:** varias reglas del listado de §4 no tienen **marca** explícita para restricciones que el texto menciona como de base (p. ej. `cover_key` nunca cadena vacía; ISBN solo dígitos). El lector debe inferir si son motor o solo dominio. Inconsistencia con el propio sistema de marcas.

### H-13 · Nota de `.env` versionado en constitución contradictoria con la entrega
- **Ruta:** `spec-python/constitution.md` Art. IX y `spec-python/plan.md` §2; `README.md` (repo docs).
- **Problema:** la constitución afirma que "solo se versiona `.env.example`", pero este repositorio (`docs`) no contiene `.env.example` ni ningún archivo de configuración. En el alcance del repo docs es esperable, pero la redacción induce a buscar el archivo. Ambigüedad menor de alcance.

### H-14 · `tasks.md` estado inicial "pendiente" y ausencia de owner/fechas
- **Ruta:** `spec-python/tasks.md`.
- **Problema:** no es técnica de spec, pero para una entrega SDD no hay campos de evidencia adjunta ni identificación de quién ejecuta. No bloqueante.

### H-15 · `README.md`: "los otros cinco empiezan vacíos" vs deliverables
- **Ruta:** `README.md` (línea 6 y tabla de repos).
- **Problema:** el README dice que este repo es "el único con contenido"; se asume que los otros cinco existen y están vacíos. Como solo se auditó `docs`, no se verificó la existencia/vacío real de los otros repos. Fuera de alcance, pero es una dependencia no verificable desde aquí. (No es inconsistencia documental interna.)

### H-16 · O (residual) Portada: código de éxito y tamaños
- **Ruta:** `spec-python/api-contract.md` §3.4 y §4 (seguridad).
- **Problema:** no se define código HTTP de éxito de `PUT/DELETE /books/{id}/cover` (200/204) ni si `GET /books/{id}/cover` devuelve `Content-Type` derivado o fijo. Menor.

### H-17 · (residual) Conversión fechas en reporte por instante
- **Ruta:** `spec-python/api-contract.md` §3.7; `spec-python/adr/adr-004-reporte-agregado-y-congelado.md`.
- **Problema:** el reporte recibe fechas (ambas inclusivas) pero no especifica el timezone ni la hora de corte. ADR-004 habla de instante, CA-10.1 no. Riesgo de diferencia de un día.

### H-18 · (residual) Normalización del administrador inicial
- **Ruta:** `spec-python/plan.md` D-10; `spec-python/spec.md` HU-02.2.
- **Problema:** plan dice que el admin inicial se crea desde entorno, pero no explicita si aplica la normalización de HU-02.2 (minúsculas y trim) al `ADMIN_USERNAME`. Ambiguo para implementación.

### H-19 · (residual) ISBN con guiones
- **Ruta:** `spec-python/spec.md` CA-03.2; `spec-python/data-model.md` (`isbn char(13)`).
- **Problema:** CA-03.2 dice "13 dígitos"; el tipo `char(13)` y la nota "solo dígitos" implican sin guiones, pero no se dice explícitamente el rechazo de guiones en la validación.

### H-20 · (residual) Búsqueda de libros: sensibilidad y `q` vacío
- **Ruta:** `spec-python/api-contract.md` §3.4 (`q`, `category_id`, `page`, `size`); `spec-python/spec.md` CA-04.1.
- **Problema:** no se especifica sensibilidad a mayúsculas/acentos ni comportamiento cuando `q` va vacío o ausente.

---

## 4. Puntos verificados sin hallazgos

- Historias HU-01..HU-10 y CA tienen correspondencia con códigos de error de §2 del contrato (sin contradicciones excepto H-05).
- Roles (`admin`, `librarian`) y endpoints coinciden con los actores de `spec.md`.
- `data-model.md` §7 (privacidad) coincide con CA-10.3 (sin socios ni bibliotecarios en el reporte).
- ADR-001..ADR-004 son coherentes con `constitution.md` (artículos V a VIII).
- Índices y FKs marcados "motor" tienen verificación en §10 del mismo `data-model.md`.
- Orden de lectura declarado en `README.md` coincide con archivos presentes.
- Las dos versiones de los ADR son idénticas (sin bifurcación).

---

## 5. Conclusión

- **Bifurcación python/.net:** no hallada en negocio (solo diferencias tecnológicas esperadas).
- **Consistencia interna:** existen 12 inconsistencias principales (H-01..H-12) y 8 ambigüedades residuales (H-13..H-20). No se han modificado los documentos fuente; este reporte es el único artefacto añadido.
- Se recomienda aplicar las correcciones H-01..H-07 (al menos) y anotar las decisiones de implementación para H-08..H-20 en el `README` de la API o en tests de contrato.
