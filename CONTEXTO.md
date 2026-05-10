# BRONCO-COLOR — Contexto del proyecto

## Qué es
App web de seguimiento de bronquiectasias para un estudio piloto clínico.
Pacientes registran semanalmente el color del esputo (escala Murray 1–8) y
el cuestionario CAT. Los clínicos validan las clasificaciones desde un panel
dedicado.

## Stack técnico
- **Un solo archivo**: `index.html` (~2 300 líneas). Sin framework, vanilla JS.
- **Estado global**: objeto `S` con todos los campos de la sesión.
- **Render**: función `render()` → `renderP()` / `renderC()` según rol.
  Cada vista es una función `renderXxx()` que devuelve HTML como string.
- **Firebase Firestore**: colecciones `registros`, `perfiles`, `config`.
- **Firebase Storage**: imágenes de esputo en `esputo/{pid}/{id}.jpg`.
- **EmailJS**: solo alertas **rojas** al correo del investigador (ámbar/naranja
  solo en panel, sin email — Fase 3).
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
- **Fix**: check `aiGrade` con `!= null && >= 1 && <= 8`.

### ✅ Fase 2b — Refactor del wizard CAT (completada 2026-05-10)
- `getPrevPatientRecord(pid)`: helper central que filtra `S.records` por
  `patientId === pid && r.cat != null`, ordena por fecha con null-guard,
  devuelve el registro anterior más reciente (o `null`).
- `refreshCatScore()`: usa el helper; elimina bug de truthiness (`r.cat`
  excluía CAT=0) y condición `t>=3` incorrecta.
- `renderW3()`: usa el helper; elimina filtro sin `patientId`.
- `wizardSubmit()`: sustituye bloque duplicado de 4 líneas por el helper.

### ✅ Fase 3 — Sistema de alertas v2 (completada 2026-05-10; Cambio D pendiente)

#### Cambio A — Nivel naranja y `ALERT_PRIORITY`
- `ALERT_PRIORITY = {rojo:4, naranja:3, ambar:2, verde:1}`: constante única,
  elimina los tres mapas hardcodeados que existían en distintas funciones.
- Panel: stat naranja (🟠 Atención); 5 columnas; colores naranja en tarjetas.
- `getPatientSummary()`: `topAlert` excluye registros con `alertAcknowledged===true`;
  usa `ALERT_PRIORITY`.
- `renderCPatient()`: header/icono/historial con caso naranja.
- `renderCReview()`: sort y `alertBorder` para naranja.

#### Cambio B — `computeAlert()` reescrita según EF v1.0
- Nueva firma: `computeAlert(record, prevRecord) → {level, reasons[]}`.
  Criterios vs visita anterior (`catDelta`), no vs basal.
- **ROJO aislado** (cualquiera dispara): hemoptisis, `catDelta≥5`,
  disnea grave nueva (`catAnswers[3]===5` con previo `≤2`).
- **ROJO combinado** (≥2 de): `catDelta≥3`, esputo grave (`murrayGrade≥6`
  o subida ≥2 grados), fiebre, dolor torácico.
- **ÁMBAR**: `catDelta≥2`.
- **NARANJA**: q1(tos)/q2(flema)/q4(disnea) sube ≥2 puntos con `catDelta<2`.
- `sendAlertEmail()`: solo dispara para `level==='rojo'` (antes: ambar+rojo).
- `wizardSubmit()`: `alertResult` calculado antes del `const record={}`;
  guarda `alertLevel` y `alertReasons` desde el resultado.
- `renderW3()` preview: nueva firma + caso naranja.

#### Cambio C — `ackAlert()` y botón "Marcar como atendida"
- `ackAlert(recordId)`: actualiza `S.records` en memoria + `DB.updateRecord()`
  con `{alertAcknowledged:true, alertAckBy:S.clinicRole, alertAckAt:ISO}`.
- `bindC()`: handler `.ack-btn` con `stopPropagation`.
- `renderCPatient()`: tarjetas de alerta con botón "Marcar como atendida"
  (solo rojo no atendida), badge "✓ Atendida", `alertReasons` visible,
  opacidad 0.65 en atendidas.
- `genDemoRecords()`: `PEND` y `val()` incluyen ack defaults;
  `alertReasons` en registros no-verde; BC-007c → naranja (`q2_flema↑≥2`);
  BC-008b → atendida por neumo.

#### Cambio D — Cloud Function recordatorio 48h *(pendiente, sesión separada)*
- Si `alertLevel ∈ {rojo}` y `alertAcknowledged===false` y `fecha` > 48h:
  enviar recordatorio al investigador.
- Requiere Firebase Cloud Functions + decisión de servicio email
  (EmailJS REST API o Firebase Trigger Email extension).

## Commits relevantes
```
8e6110e  Fase 3 (A/B/C): sistema alertas v2 — naranja, EF v1.0, ack
a79c4c3  docs: añade EF_v1_0_completa.md
5090c98  Fase 2b: refactor wizard CAT — extrae getPrevPatientRecord()
0ec9c0b  Fix: renderCReview crash con registros Firestore sin aiGrade
ab73133  Fase 2a: vista de validación clínica con escala Murray 1-8
2b96548  Fase 1, Cambio 3: añade wellbeingAuto calculado desde catDelta
6bfec45  Fix: añadir 'const record={' que faltaba en wizardSubmit
256f7bb  Fase 1 (2/3): calcular catDelta y catDeltaByQ vs visita previa
b7c5d2d  Fase 1 (1/3): añadir campos clinicianGrade y alertReasons al schema
```
