# FUTURE_MODULES.md — Módulos futuros de BSAgent

Este archivo documenta los módulos que BSAgent incorporará después del Calendar MVP.
**Ninguno de estos módulos está implementado.** Son diseños preliminares para orientar
la arquitectura futura y evitar decisiones que los hagan incompatibles.

BSAgent está diseñado como sistema modular. Cada módulo añade capacidades nuevas
sin romper los módulos existentes. El Calendar MVP es la base sobre la que se construyen.

---

## Índice de módulos

| Módulo | Prioridad futura | Estado |
| --- | --- | --- |
| Shopping List | Alta | No implementado |
| Wishlist | Media | No implementado |
| Car Maintenance | Media-alta | No implementado |
| Training & Nutrition | Media | No implementado |

---

## Módulo 1 — Shopping List

### Objetivo

Permitir que Jon añada productos a la lista de la compra desde WhatsApp y pueda
consultar qué tiene que comprar cuando vaya al supermercado.

### Problema que resuelve

La lista de la compra mental se pierde. Este módulo permite dictar productos en
cualquier momento desde WhatsApp y consultarlos en el punto de venta sin buscar
en otras apps.

### Ejemplos de mensajes

```
"Añade leche, huevos y yogur griego a la compra"
"Apunta papel de cocina"
"¿Qué tengo que comprar?"
"Marca huevos como comprado"
"Borra la lista, ya he hecho la compra"
```

### Intenciones futuras

| Intent | Descripción |
| --- | --- |
| `ADD_SHOPPING_ITEMS` | Añadir uno o más productos a la lista |
| `QUERY_SHOPPING_LIST` | Consultar productos pendientes |
| `MARK_ITEM_BOUGHT` | Marcar producto como comprado |
| `CLEAR_SHOPPING_LIST` | Vaciar la lista (requiere confirmación) |

### Datos mínimos futuros

```
shopping_items:
  - item_id         uuid
  - item_name       texto
  - quantity        texto libre ("2 litros", "1 docena", null)
  - category        texto (lácteos, limpieza, fruta, etc.)
  - status          pending | bought | cancelled
  - source_message  texto original del usuario
  - created_at      timestamptz
  - bought_at       timestamptz | null
```

### Reglas de comportamiento previstas

- `ADD_SHOPPING_ITEMS` no requiere confirmación — añadir es de bajo riesgo.
- `CLEAR_SHOPPING_LIST` sí requiere confirmación antes de borrar.
- `QUERY_SHOPPING_LIST` devuelve solo los ítems en estado `pending`.
- La categoría puede inferirse por IA o dejarse vacía.

### Persistencia

- n8n Data Store en MVP de módulo.
- Supabase si el volumen crece o se añade historial de listas pasadas.

### Prioridad futura

**Alta.** Módulo sencillo de implementar y con uso diario probable. Candidato a ser
el segundo módulo tras Calendar MVP estabilizado.

### Fuera de alcance inicial del módulo

- Recetas automáticas basadas en la lista.
- Precios o comparación de supermercados.
- Integración con apps de supermercado.
- Compartir lista con otra persona.

---

## Módulo 2 — Wishlist

### Objetivo

Guardar cosas que Jon quiere comprar o investigar en el futuro, separadas de la
compra semanal habitual.

### Problema que resuelve

Las ideas de compra ("quiero mirar una silla buena", "me interesa ese libro") se
pierden si no se anotan en el momento. Este módulo actúa como bloc de notas
de intenciones de compra, con categorías y prioridades.

### Diferencia con Shopping List

- **Shopping List:** productos de consumo recurrente y compra inmediata.
- **Wishlist:** artículos no urgentes, de mayor valor o que requieren investigación previa.

### Ejemplos de mensajes

```
"Guárdame que quiero mirar una silla de oficina buena"
"Añade a deseos: pantalón técnico negro"
"Apunta que quiero mirar una mochila para el portátil"
"¿Qué cosas tenía pendientes de comprar para casa?"
"¿Qué tengo en deseos de tecnología?"
"Descarta la silla, ya no me interesa"
```

### Intenciones futuras

| Intent | Descripción |
| --- | --- |
| `ADD_WISHLIST_ITEM` | Añadir un deseo a la lista |
| `QUERY_WISHLIST` | Consultar deseos por categoría o estado |
| `UPDATE_WISHLIST_ITEM` | Cambiar estado o notas de un deseo |
| `DISCARD_WISHLIST_ITEM` | Marcar deseo como descartado |

### Datos mínimos futuros

```
wishlist_items:
  - item_id           uuid
  - title             texto
  - category          ropa | casa | trabajo | tecnología | coche | deporte | kobo | otro
  - priority          high | medium | low
  - estimated_budget  número | null
  - notes             texto libre
  - status            idea | researching | shortlisted | bought | discarded
  - source_message    texto original del usuario
  - created_at        timestamptz
  - updated_at        timestamptz
```

### Categorías iniciales previstas

| Categoría | Ejemplos |
| --- | --- |
| `ropa` | Pantalón técnico, zapatillas, ropa deporte |
| `casa` | Silla, lámpara, electrodoméstico |
| `trabajo` | Periférico, software, accesorio escritorio |
| `tecnología` | Dispositivo, gadget, hardware |
| `coche` | Accesorio, recambio, herramienta |
| `deporte` | Material, equipamiento, suplementos |
| `kobo` | Libros, accesorios de lectura |
| `otro` | Sin categoría clara |

### Prioridad futura

**Media.** Útil pero no urgente. Se implementa después de Shopping List.

### Fuera de alcance inicial del módulo

- Recomendaciones automáticas de productos.
- Búsquedas de precios en tiempo real.
- Comparativa entre tiendas.
- Alertas de bajada de precio.
- Compartir wishlist.

---

## Módulo 3 — Car Maintenance

### Objetivo

Registrar el historial completo del coche de Jon: mantenimientos, reparaciones,
cambios de aceite, neumáticos, ITV, averías, costes y kilometraje.

### Problema que resuelve

El historial del coche se pierde entre facturas, notas sueltas y la memoria. Este
módulo centraliza toda la información del vehículo en un formato consultable,
para saber cuándo toca el próximo mantenimiento, cuánto se ha gastado y qué se
ha hecho en cada intervención.

### Ejemplos de mensajes

```
"Hoy he cambiado el aceite del coche con 142.000 km"
"He cambiado las ruedas delanteras"
"¿Cuándo fue la última vez que cambié el aceite?"
"¿Qué mantenimiento tengo pendiente?"
"¿Cuánto llevo gastado en el coche este año?"
"¿Cuándo es la próxima ITV?"
"El coche hace un ruido raro al frenar"
```

### Datos mínimos futuros

```
vehicles:
  - id          uuid
  - brand       texto
  - model       texto
  - year        número
  - fuel_type   gasolina | diésel | híbrido | eléctrico | otro
  - notes       texto

vehicle_events:
  - id              uuid
  - vehicle_id      uuid → vehicles.id
  - event_type      ver tabla de tipos
  - date            date
  - mileage_km      número
  - description     texto
  - workshop        texto | null
  - cost            número | null
  - documents       referencias a archivos | null
  - next_due_date   date | null
  - next_due_km     número | null
  - notes           texto | null
```

### Tipos de evento previstos

| Tipo | Descripción |
| --- | --- |
| `oil_change` | Cambio de aceite y filtro |
| `filter_change` | Filtro de habitáculo, aire, combustible |
| `tires` | Cambio o rotación de neumáticos |
| `brakes` | Pastillas, discos, líquido de frenos |
| `battery` | Batería principal |
| `inspection` | Revisión de taller general |
| `repair` | Reparación no planificada |
| `insurance` | Renovación o cambio de seguro |
| `itv` | Inspección técnica de vehículos |
| `cleaning` | Limpieza profunda o detailing |
| `other` | Otro tipo de evento |

### Intenciones futuras

| Intent | Descripción |
| --- | --- |
| `LOG_CAR_EVENT` | Registrar un evento de mantenimiento o reparación |
| `QUERY_CAR_HISTORY` | Consultar historial por tipo de evento o fecha |
| `QUERY_NEXT_MAINTENANCE` | Consultar próximos mantenimientos pendientes |
| `QUERY_CAR_COSTS` | Consultar gastos por período |

### Límite de seguridad — importante

BSAgent puede:
- Registrar historial e interpretar documentación del taller.
- Recordar cuándo toca el próximo mantenimiento.
- Responder a preguntas sobre el historial registrado.
- Orientar sobre qué preguntar al mecánico.
- Informar sobre comprobaciones básicas (nivel de aceite, presión de neumáticos).

BSAgent **no debe** dar instrucciones para:
- Reparaciones de frenos, dirección o suspensión crítica.
- Reparaciones del sistema eléctrico complejo.
- Trabajos en el motor que requieran desmontaje.
- Bypass de sistemas de seguridad activa.
- Actuaciones que puedan afectar a la homologación o superar la ITV.

**Motivo:** El riesgo de una instrucción incorrecta en sistemas de seguridad del vehículo
puede causar accidentes graves. BSAgent debe orientar, no sustituir a un mecánico profesional.

### Persistencia

Supabase desde el inicio de este módulo. El volumen de datos y la estructura relacional
(vehicle → events) no encaja bien en el Data Store de n8n.

### Prioridad futura

**Media-alta.** El historial del coche es información sensible que se pierde con el tiempo.
Interesa documentarlo antes de que los registros se pierdan.

### Fuera de alcance inicial del módulo

- Base documental de manuales o fichas técnicas del vehículo.
- Subida y OCR de facturas de taller.
- RAG sobre documentación técnica.
- Alertas automáticas de próximo mantenimiento.
- Gestión de múltiples vehículos en la primera versión.

---

## Módulo 4 — Training & Nutrition

### Objetivo

Registrar entrenamientos, pesos levantados, peso corporal, alimentación aproximada
y evolución semanal de Jon.

### Problema que resuelve

El seguimiento del entrenamiento y la nutrición se dispersa entre apps, notas y
conversaciones de ChatGPT. Este módulo centraliza el registro en BSAgent y permite
consultar la evolución sin salir de WhatsApp.

### Contexto de migración

Cuando se implemente este módulo, se deberá consolidar como base inicial la
información existente en el proyecto Salud de ChatGPT:

- Rutina de entrenamiento actual.
- Ejercicios habituales y músculos trabajados.
- Histórico de pesos registrados por ejercicio.
- Progresión por serie y repetición.
- Registro de peso corporal.
- Hábitos y patrones de alimentación.
- Objetivo actual (superávit calórico, subida de peso).

Esta migración es una tarea explícita antes de activar el módulo.

### Ejemplos de mensajes

```
"Hoy he hecho press banca 55 kg: 10, 9, 8 reps"
"Peso de hoy: 76,8 kg"
"Hoy he comido batido, arroz con pollo y yogur griego"
"¿Cómo voy esta semana con el superávit?"
"¿Cuánto hice de peso muerto la semana pasada?"
"Resumen del entrenamiento de hoy"
"Añade 3 series de curl martillo 14 kg: 12, 12, 10"
```

### Intenciones futuras

| Intent | Descripción |
| --- | --- |
| `LOG_TRAINING_SESSION` | Registrar una sesión de entrenamiento |
| `LOG_EXERCISE_SETS` | Registrar series de un ejercicio específico |
| `LOG_BODY_WEIGHT` | Registrar el peso corporal del día |
| `LOG_NUTRITION` | Registrar ingesta del día en texto libre |
| `QUERY_TRAINING_PROGRESS` | Consultar evolución de un ejercicio |
| `QUERY_WEEKLY_SUMMARY` | Resumen semanal de entrenamiento y nutrición |

### Datos mínimos futuros

```
training_sessions:
  - id            uuid
  - date          date
  - routine_day   texto (ej: "Día A — Empuje")
  - notes         texto | null

training_exercises:
  - id            uuid
  - session_id    uuid → training_sessions.id
  - exercise_name texto
  - muscle_group  texto

training_sets:
  - id            uuid
  - exercise_id   uuid → training_exercises.id
  - weight_kg     número
  - reps          número
  - rpe           número | null (escala 1–10)
  - notes         texto | null

body_weight_logs:
  - id            uuid
  - date          date
  - weight_kg     número

nutrition_logs:
  - id                  uuid
  - date                date
  - meal_text           texto libre (transcripción del mensaje)
  - estimated_protein   número | null
  - estimated_calories  número | null
  - confidence          número | null (0–1)
```

### Notas de diseño

- `meal_text` guarda el texto del mensaje tal cual — la IA estima proteína y calorías
  pero la fuente de verdad es el texto, no la estimación.
- `confidence` refleja cuánta certeza tiene la IA en la estimación nutricional.
  Las estimaciones de baja confianza deben marcarse visualmente.
- La migración de datos desde ChatGPT es una precondición del módulo, no opcional.

### Prioridad futura

**Media.** Puede subir de prioridad cuando Calendar MVP esté estable. El proyecto Salud
ya tiene datos históricos valiosos que conviene no perder.

### Fuera de alcance inicial del módulo

- Generación automática de rutinas de entrenamiento.
- Planes de dieta estructurados.
- Integración con wearables o apps de fitness (Garmin, Apple Health).
- Análisis biomecánico o recomendaciones de técnica.
- Comparación con referencias externas o tablas de fuerza.
- Migración automática desde ChatGPT (es un proceso manual supervisado).

---

## Consideraciones arquitectónicas transversales

Estos puntos aplican a todos los módulos futuros y deben tenerse en cuenta
cuando se diseñe cada uno:

### Principio de confirmación por módulo

Cada módulo define sus propias reglas de confirmación según el riesgo de la acción:

- **Bajo riesgo** (añadir ítem a lista, registrar entrenamiento): sin confirmación.
- **Riesgo medio** (marcar lista como completada, borrar historial): con confirmación.
- **Riesgo alto** (eliminar datos históricos, acciones irreversibles): con doble confirmación.

### Persistencia por módulo

| Módulo | Persistencia recomendada |
| --- | --- |
| Shopping List | n8n Data Store en MVP → Supabase si crece |
| Wishlist | n8n Data Store en MVP → Supabase si crece |
| Car Maintenance | Supabase desde el inicio (estructura relacional) |
| Training & Nutrition | Supabase desde el inicio (volumen de datos alto) |

### Integraciones con el módulo de Calendar

- Car Maintenance puede crear eventos en Calendar para próximos mantenimientos.
- Training puede crear eventos en Calendar para días de entrenamiento planificados.
- Estas integraciones son opcionales y se diseñan en cada módulo, no en Calendar MVP.

### Clasificador de intención unificado

Cuando BSAgent tenga múltiples módulos activos, el clasificador de intención
deberá identificar a qué módulo pertenece cada mensaje antes de clasificar la
intención específica. Esto se diseña cuando el segundo módulo esté listo para
implementarse.
