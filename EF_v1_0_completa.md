# BRONCO-COLOR — Especificación Funcional v1.0

## 1. Sistema de alertas

### 1.1 Jerarquía de niveles
| Nivel | Disparador | Panel | Email |
|-------|-----------|-------|-------|
| 🟢 Verde | Sin cambios significativos | Sí | No |
| 🟠 Naranja | Subida ≥2 pts en cat_q1/q2/q4 con catDelta < 2 | Sí | No |
| 🟡 Ámbar | catDelta ≥2 puntos | Sí | No |
| 🔴 Rojo | Criterios de agudización | Sí (destacado) | Sí |

### 1.2 Filosofía
Solo alertas rojas generan email. Naranja y ámbar solo en panel.
Objetivo: 0-2 emails/semana en cohorte de 50 pacientes.

### 1.3 Criterios naranja
Disparador: subida ≥2 puntos en cat_q1(tos), cat_q2(flema)
o cat_q4(disnea), CON catDelta < 2 (CAT total estable).
Las preguntas q5-q8 NO disparan naranja (reflejan estado
anímico/sociofamiliar, no enfermedad respiratoria activa).

### 1.4 Criterios rojo — aislados (cualquiera dispara solo)
- Hemoptisis (hemoptysis === true)
- catDelta ≥5 puntos respecto a visita previa
- Disnea grave nueva: cat_q4 === 5 en paciente con cat_q4 habitual ≤2

### 1.5 Criterios rojo — combinados (≥2 simultáneos)
- catDelta ≥3 puntos
- Esputo: murrayGrade ≥6 nuevo, o subida ≥2 grados Murray
- Fiebre (fever === true)
- Dolor torácico (chestPain === true)

### 1.6 alertReasons
computeAlert() debe devolver array con las razones detectadas,
no solo el nivel. Se almacena en Firestore campo alertReasons[].

### 1.7 Marcar como atendida
- ackAlert(recordId) → DB.updateRecord({alertAcknowledged: true,
  alertAckBy: S.clinicRole, alertAckAt: new Date().toISOString()})
- Botón "Marcar como atendida" en panel para alertas rojas
- Alertas atendidas: excluir de topAlert
- Recordatorio 48h si alertAcknowledged === false → Cambio D (futuro)
- Sin más emails tras 96h (evita spam)

## 2. Vista de validación clínica (Fase 2a — completada)
- renderCReview() usa S.records (no S.pending)
- Selector 8 swatches Murray (grados 1-8)
- clinValidate(recordId, grade1to8, notes) actualiza S.records + Firestore
- Stats: pendientes / validados / % concordancia IA vs clínico
- Sección validados colapsable al final

## 3. Refactor wizard CAT (Fase 2b — completada)
- getPrevPatientRecord(pid): única fuente de verdad para delta CAT
- Filtra por patientId, ordena por fecha con null-guard
- Usado en refreshCatScore(), renderW3() y wizardSubmit()

## 4. Schema Firestore — campos añadidos en Fase 1
clinicianGrade, clinicianCode, clinicianValidatedAt, clinicianNotes,
alertReasons[], alertAcknowledged, alertAckBy, alertAckAt,
catDelta, catDeltaByQ{q1..q8}, wellbeingAuto, aiGrade, aiCode,
aiConfidence, aiReasoning, murrayGrade

## 5. Decisiones clínicas pendientes
- Modo ciego en vista validación: ¿desde el inicio?
- Validación retroactiva de fotos antiguas: ¿se permite?
- Frecuencia recomendada de revisión del panel
- Umbral exacto para disnea grave nueva: cat_q4=5 con basal ≤2
  (confirmado en EF v1.0)

## 6. Roadmap fases
- Fase 1 ✅ Schema datos + catDelta + wellbeingAuto
- Fase 2a ✅ Vista validación clínica Murray 1-8
- Fase 2b ✅ Refactor wizard CAT
- Fase 3 🔄 Sistema alertas v2 (Cambios A/B/C en curso, D pendiente)
- Fase 4 ⏸ Métricas ICC en tiempo real
- Fase 5 ⏸ Pruebas piloto 30-50 registros
- Fase 6 ⏸ Documentación + dossier CEIC
