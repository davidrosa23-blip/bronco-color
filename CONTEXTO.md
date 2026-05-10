# BRONCO-COLOR — Contexto del proyecto

## Qué es
App web de seguimiento de bronquiectasias para un estudio piloto clínico.
Pacientes registran semanalmente el color del esputo (escala Murray 1–8) y
el cuestionario CAT. Los clínicos validan las clasificaciones desde un panel
dedicado.

## Stack técnico
- **Un solo archivo**: `index.html` (~2 200 líneas). Sin framework, vanilla JS.
- **Estado global**: objeto `S` con todos los campos de la sesión.
- **Render**: función `render()` → `renderP()` / `renderC()` según rol.
  Cada vista es una función `renderXxx()` que devuelve HTML como string.
- **Firebase Firestore**: colecciones `registros`, `perfiles`, `config`.
- **Firebase Storage**: imágenes de esputo en `esputo/{pid}/{id}.jpg`.
- **EmailJS**: alertas ámbar/rojo al correo del investigador.
- **IA de esputo**: Claude Haiku a través de Cloudflare Worker
  (`bronco-color-ia.david-rosa23.workers.dev`). Clave Anthropic guardada
  en Firestore `config/keys → anthropic`.

## Roles y acceso
- **Paciente**: login con código `BC-NNN`. Accede al wizard de registro.
- **Clínico / Neumólogo** (clave `NEUMO`): panel completo + exportación CSV.
- **Gestora** (clave `GESTORA`): panel sin exportación.

## Escala Murray (8 grados)
| Grado | Código | Descripción |
|-------|--------|-------------|
| 1–2 | M | Mucoso (blanco/grisáceo) |
| 3–4 | MP | Mucopurulento (amarillo pálido) |
| 5–6 | P | Purulento (verde-oliva medio) |
| 7–8 | P+ | Purulento grave (verde-oliva oscuro) |

## Schema de un registro (`S.records[]`)
```
id, fecha, patientId, tipoVisita, murray, murrayGrade,
cat, catAnswers[8], catDelta, catDeltaByQ{q1..q8}, wellbeingAuto,
symptoms[], wellbeing, alertLevel, baseline, image,
aiGrade, aiCode, aiConfidence, aiReasoning,
clinicianGrade, clinicianCode, clinicianValidatedAt, clinicianNotes,
alertReasons[], alertAcknowledged, alertAckBy, alertAckAt
```

## Fases del proyecto

### ✅ Fase 1 — Infraestructura de datos
- Campos `clinicianGrade/Code/ValidatedAt/Notes` en schema de registros.
- `catDelta` y `catDeltaByQ` (delta vs visita previa pregunta a pregunta).
- `wellbeingAuto` calculado desde `catDelta`.

### ✅ Fase 2a — Vista de validación clínica (completada 2026-05-10)
- `DB.updateRecord(id, fields)`: actualización parcial en Firestore.
- `clinValidate(recordId, grade1to8, notes)`: reemplaza `clinRate()`.
  Actualiza `S.records` en memoria y persiste en Firestore.
- `renderCReview()` reescrita: fuente de datos `S.records` (no el antiguo
  `S.pending[]`). Muestra imagen real, comparativa IA vs paciente,
  CAT+delta MCID, `catDeltaByQ` top-3, `wellbeingAuto`, selector de
  8 swatches Murray, textarea de notas y botón "Validar".
- `bindC()`: conecta `.val-swatch` y `.val-btn` con feedback visual.
- `genDemoRecords()`: campos completos en todos los registros demo
  (8 pendientes de validación + 16 ya validados).
- Panel: concordancia desde `S.records`; `S.pending[]` eliminado.
- CSV ampliado con columnas de validación clínica.
- **Fix**: sort en `renderCReview` usaba `b.fecha.localeCompare()`
  sin guardia → `TypeError` con registros Firestore sin `fecha`.
  Corregido con `(b.fecha||'').localeCompare(a.fecha||'')`.
- **Fix**: check `aiGrade` ahora usa `!= null && >= 1 && <= 8`
  para blindar frente a valores inesperados de Firestore.

### ✅ Fase 2b — Refactor del wizard CAT (completada 2026-05-10)
- `getPrevPatientRecord(pid)`: helper central que filtra `S.records` por
  `patientId === pid && r.cat != null`, ordena por fecha con null-guard,
  devuelve el registro anterior más reciente (o `null`).
- `refreshCatScore()`: usa el helper; elimina bug de truthiness (`r.cat`
  excluía CAT=0) y condición `t>=3` incorrecta.
- `renderW3()`: usa el helper; elimina filtro sin `patientId`.
- `wizardSubmit()`: sustituye bloque duplicado de 4 líneas por el helper.

## Commits relevantes
```
5090c98  Fase 2b: refactor wizard CAT — extrae getPrevPatientRecord()
0ec9c0b  Fix: renderCReview crash con registros Firestore sin aiGrade
ab73133  Fase 2a: vista de validación clínica con escala Murray 1-8
2b96548  Fase 1, Cambio 3: añade wellbeingAuto calculado desde catDelta
6bfec45  Fix: añadir 'const record={' que faltaba en wizardSubmit
256f7bb  Fase 1 (2/3): calcular catDelta y catDeltaByQ vs visita previa
b7c5d2d  Fase 1 (1/3): añadir campos clinicianGrade y alertReasons al schema
c4aa5be  v3.4: prompt sin sesgo iluminación, usa toda escala Murray
```
