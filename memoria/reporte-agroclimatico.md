# Memoria — Reporte agroclimático semanal (especificación para AVI)

Última actualización: 2026-10-28. Usuario: diroperaciones@lynks.com.co.

## Preferencias del usuario (aplican a futuras conversaciones)
- **AVI** es la plataforma que maneja `lynks-backend` (puntos, grupos, variables, eventos, etc.).
- Para quien arma las instrucciones/especificaciones, **de dónde viene la información es indiferente**: no preguntar por fuentes de datos, grupos, puntos ni cómo se obtienen. Decidirlo con la recomendación propia y declararlo como supuesto.
- Cuando se hable de **ventanas** (periodicidad de envío), preguntar solo si el envío es **diario, semanal, mensual, horario o todos**. No preguntar horas, días ni cortes exactos; proponerlos.
- Las preguntas de especificación van en rondas (máx. 7), con la opción recomendada. No repetir preguntas ya respondidas.
- Responder y entregar en español. Reportes de ejemplo: correo HTML a diroperaciones@lynks.com.co (asunto con `[PRUEBA]` si es prueba).
- No exponer IDs internos al usuario (regla de CLAUDE.md); usar nombres de las estaciones.

## Contexto de la tarea
- Skill usada: `especificar-reporte-lynks` (entrega un paquete: `especificacion.md`, `imagenes/`, `maqueta_salida.html`, `datos_ejemplo/`; no código).
- Recomendación: **rutina nueva** en la plataforma `agro` (no es subreporte de una rutina existente).
- Estado: Fase 2 (preguntas). El usuario aún no dijo "genera".
- Respondidas: AVI = lynks-backend; fuentes de datos indiferentes; ventanas → solo preguntar tipo de envío.

## Qué hace el reporte (misma lógica de la rutina creada)
Reporte semanal por estación meteorológica, según el cultivo:
- Tabla diaria: lluvia, ET0 (suma de `ET0_hora`), ETc = ET0 × Kc, balance = lluvia − ETc (y acumulado), T mín/máx, horas con T ≥ 30 y ≥ 35 °C, horas con HR > 85%, horas con DPV alto, alertas ✅ ⚠️ 🔴, totales de la semana.
- Órdenes de trabajo: 💧 Riego (0 h si balance positivo; si hay déficit: mm a reponer y horas = mm ÷ lámina del sistema, tabla para 1/2/3 mm/h si se desconoce), 🛡️ Sanidad, 🚜 Ventana de aplicación (viento < 6 km/h, sin lluvia, T/DPV adecuados; hora de inicio exacta), 🌱 Desarrollo (GDD o floración en caña).
- Supuestos y limitaciones (Kc usado, HR medida/calculada/estimada, registros faltantes, día parcial).
- Entrega: correo HTML (solo tablas con estilos en línea).

## Parámetros y umbrales usados
- Kc (FAO): caña 0.4 inicial / 0.8 desarrollo / 1.1–1.25 gran crecimiento / 0.7 maduración; aguacate 0.6–0.85; uva 0.3 brotación / 0.7–0.85 fruto / 0.45–0.6 maduración; limón y cítricos 0.7.
- GDD: base 10 °C (caña, uva, aguacate); base 12.5 °C (cítricos).
- Mildiu (uva): horas continuas con follaje mojado (lluvia o HR>85%) y T entre 18 y 25 °C; umbral de decisión 6 h continuas.
- DPV (calculado con T y HR): alto > 2.5 kPa, estrés > 3.5 kPa. Viento para aplicar < 6 km/h. Registros mínimos por día: 90% (propuesto).
- HR: prioridad medida → calculada con T y punto de rocío (Magnus) → N/D (usar horas de lluvia como indicador de mojado).
- Caña: floración por fotoperiodo con la latitud (inducción cuando el día baja de ~12.5 h a ~12 h; emergencia visible 2–3 meses después).

## Estaciones de prueba (evidencia del MCP, consulta 2026-10-27/28)
| Estación | Cultivo de prueba | Frecuencia | HR | ET0 | Notas |
|---|---|---|---|---|---|
| Estación La Estrella (`5a62afe6d3d41b8f6199429e`) | caña | 30 min | variable HR existe (`5a62b060d3d41b8f619942a7`) pero NO sale en la lista del equipo ni en datos crudos; solo agregados vía `get_variable_data` | `ET0_hora` | lat ≈ 8.3° N |
| CARTAMA: Estación Vivero la Playita (`60c0dc136868749f3288b4b9`) | aguacate | 30 min | no mide; calcular con T y punto de rocío | `ET0_hora` | 25/10 con 39/48 registros; DPV min/máx son umbrales constantes; variables de floración vacías |
| Estación grajales (`5cdddc7292ce810fcd3a64ae`) | uva | 30 min | no mide ni tiene punto de rocío | `ET0_hora` | usar horas con lluvia como indicador de mojado |
| Estación - Finca Santa Bárbara (`6876d7c5c34eb677cc2ba5e0`) | limón | 10 min | medida | `ET0_hora` (ignorar "ET0 (NO UTILIZAR)") | 27/10 con 94.4% de completitud |

Hallazgos: el MCP no devuelve `options.sensor`; las pruebas mezclaron Kc (caña/aguacate sin Kc, uva/limón con Kc 0.7) → la especificación debe fijar una regla única; la plataforma tiene 228 grupos visibles.
Resultados de la semana 21–27/10/2026 (prueba): La Estrella lluvia 10.9 / ET0 25.7 / balance −14.8; Vivero La Playita lluvia 143.2 / ET0 22.1 / balance +121.2 (dos lluvias de ~66–67 mm); Grajales lluvia 35.8 / ET0 20.1 / balance +15.7 (con Kc 0.7: +21.7); Finca Santa Bárbara lluvia 3.4 / ET0 31.0 / ETc 21.7 / balance −18.3, máx. 40.8 °C el 23/10.

## Rutinas creadas en la nube (claude.ai)
- `trig_01L2pbDJTFyXMEmufvUthhEN` — "Informe agroclimático por estación (bajo demanda)": sin horario, se dispara a mano con ID de estación y cultivo.
- `trig_01LTYsvf8nxEjJdY3Rse1Dy1` — "Informe agroclimático semanal 4 estaciones": lunes ~6:21 a. m. Bogotá (`CRON_TZ=America/Bogota 21 6 * * 1`), correo HTML a diroperaciones@lynks.com.co.
- **Pendiente:** ambas rutinas se crearon sin conectores (la API no permite asignarlos); el usuario debe activar **Lynks y Gmail** en la configuración de cada rutina en claude.ai para que funcionen.

## Preguntas pendientes de la especificación (ronda 2)
Frecuencia de envío (diario/semanal/mensual/horario/todos); parámetros por estación; módulos de la v1 y fuera de alcance; umbrales por cultivo; datos faltantes y precedencia de reglas; recomendaciones por plantilla vs redactadas; salida (un correo por cliente, destinatarios, "sin novedades", comparativo con semana anterior).
