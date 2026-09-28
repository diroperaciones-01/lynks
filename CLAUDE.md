# Informe de Monitoreo — Grupos de Salud Lynks

Este archivo define las instrucciones que Claude debe seguir al ejecutar el informe de monitoreo
de los grupos de salud. El informe se ejecuta bajo demanda o de forma semanal según se indique.

---

## Alcance actual (piloto)

Durante la fase de afinamiento, ejecutar **solo el grupo Castellana**.
Una vez validado, extender a los 22 grupos de salud.

Para activar el informe completo de todos los grupos, el usuario debe indicar explícitamente:
> "ejecuta el informe para todos los grupos"

---

## Instrucciones generales

- Nunca expongas IDs internos de puntos, grupos o variables al usuario.
- Presenta los resultados en tablas Markdown con encabezados claros.
- Usa íconos para el estado: ✅ normal · ⚠️ advertencia · 🔴 crítico.
- Si una herramienta no está disponible o retorna error, indícalo en el resultado con nota explicativa.
- Al inicio de cada ejecución imprime la fecha y hora de consulta.

---

## Módulo 1 — Bolsa de correo y SMS consumida

**Objetivo**: mostrar el % de la bolsa de correo y SMS consumida a la fecha para cada grupo/cliente.

**Herramienta**: `get_group_overview` por cada grupo.

**Instrucciones**:
1. Llama `get_group_overview` para obtener el resumen del grupo.
2. Busca en la respuesta campos relacionados con cuota de notificaciones: `sms_quota`, `email_quota`,
   `sms_used`, `email_used`, o similares.
3. Calcula el porcentaje: `(usado / total) * 100`.
4. Si la respuesta no contiene información de cuota, anota "⚠️ Dato no disponible en MCP"
   y continúa con los demás módulos.

**Formato de salida**:
```
## Módulo 1 — Consumo de bolsa (al DD/MM/YYYY)

| Grupo          | Correos usados | % Correo | SMS usados | % SMS |
|----------------|---------------|----------|-----------|-------|
| Castellana     | X / Y         | XX%      | X / Y     | XX%   |
```

---

## Módulo 2 — Señal SSI/BDI (rssi)

**Objetivo**: detectar equipos con señal de radio baja que puedan perder conectividad.

**Reglas**:
- Valor `rssi` positivo → ignorar (no aplica).
- Valor `rssi` negativo entre -1 y -66 → ✅ señal normal.
- Valor `rssi` negativo menor que -66 (ej. -67, -80) → 🔴 **ALERTA: SSI/BDI bajo**.

**Herramientas**:
1. `search_groups` → obtener el grupo.
2. `get_points_by_group` → listar equipos del grupo.
3. `get_point_status` por cada equipo → leer variable con label `rssi` o `SSI` o `BDI` o similar.

**Instrucciones**:
1. Para cada equipo del grupo, obtener su estado actual con `get_point_status`.
2. Buscar en `variables` la entrada cuya `label` contenga "rssi", "ssi", "bdi" o "señal"
   (sin distinguir mayúsculas).
3. Leer `last_value`. Aplicar las reglas de umbral definidas arriba.
4. Construir la tabla de resultados.
5. Si un equipo no tiene variable de señal, marcarlo como "Sin variable rssi".

**Formato de salida**:
```
## Módulo 2 — Señal SSI/BDI

| Equipo                          | rssi (dB) | Estado              |
|---------------------------------|-----------|---------------------|
| Nevera 1 Farmacia principal ... | -61       | ✅ Normal           |
| Nevera 2 Farmacia cirugía ...   | -69       | 🔴 SSI/BDI bajo     |
```

Al final de la tabla, si hay equipos críticos, mostrar:
> ⚠️ **X equipo(s) con señal por debajo de -66 dB requieren revisión física de antena o reubicación.**

---

## Módulo 3 — Eventos del mes en curso

**Objetivo**: cuántos eventos generó cada grupo desde el 1 del mes hasta hoy.

**Herramienta**: `count_group_events`

**Instrucciones**:
1. Determinar el primer día del mes en curso a las 00:00:00 (ej. `2026-09-01T00:00:00`).
2. Determinar hoy a las 23:59:59 (ej. `2026-09-28T23:59:59`).
3. Llamar `count_group_events` con esos valores, `bucket_by_month: false`.
4. Llamar también con `reviewed: false` para obtener el conteo de pendientes.

**Formato de salida**:
```
## Módulo 3 — Eventos del mes (01/MM/YYYY – DD/MM/YYYY)

| Grupo      | Total eventos | Revisados | Pendientes de revisar |
|------------|--------------|-----------|----------------------|
| Castellana | 21           | 15        | 6                    |
```

---

## Módulo 4 — Completitud de datos / Frecuencia de transmisión

**Objetivo**: identificar equipos que transmitieron menos días de los esperados en el mes,
calculando qué porcentaje de días tuvieron datos.

**Herramienta principal**: `check_group_data_completeness` — una llamada por día del mes.

**Instrucciones**:
1. Calcular N = número de días transcurridos en el mes hasta hoy (incluyendo hoy).
2. Para cada día d (desde día 1 hasta hoy), llamar `check_group_data_completeness` con:
   - `start`: `YYYY-MM-ddT00:00:00`
   - `end`: `YYYY-MM-ddT23:59:59`
3. Por cada equipo, acumular:
   - `dias_completos` = días con `overall_status == "complete"`
   - `dias_parciales` = días con `overall_status == "partial"`
   - `dias_sin_datos` = días con `overall_status == "no_data"`
4. Calcular:
   - `% días con datos` = `(dias_completos + dias_parciales) / N * 100`
   - `% días completos` = `dias_completos / N * 100`
5. Clasificar:
   - ≥ 95% días con datos → ✅ Normal
   - 80–94% → ⚠️ Parcial
   - < 80% → 🔴 Crítico
6. Equipos con `overall_status == "unknown"` → excluir del cálculo, reportar como
   "Sin frecuencia configurada".

**Nota de eficiencia**: Si el mes tiene muchos días y el número de equipos es grande,
evalúa primero con `get_point_data(range=this_month)` para ver si `data_quality.count_variables`
ofrece suficiente información antes de hacer N llamadas diarias.

**Formato de salida**:
```
## Módulo 4 — Completitud de datos (mes en curso, N=28 días evaluados)

| Equipo                         | Días con datos | % Datos | % Completo | Estado   |
|--------------------------------|---------------|---------|-----------|----------|
| Nevera 1 Farmacia principal... | 28 / 28       | 100%    | 62%       | ✅ Normal |
| Nevera 2 Farmacia cirugía...   | 22 / 28       | 79%     | 45%       | 🔴 Crítico|
```

Al final: resumen con conteo por estado y % global del grupo.

---

## Módulo 5 (futuro) — Límites configurados vs alertas activas

Este módulo se desarrollará en una segunda fase. Verificará si cada equipo tiene
la alerta correctamente configurada según el límite definido para su tipo de variable.

Herramientas a usar: `get_point_alerts`, `get_point_raw_alerts`.

---

## Orden de ejecución del informe

1. Imprimir encabezado con fecha/hora.
2. Ejecutar Módulo 1 (bolsa).
3. Ejecutar Módulo 2 (señal rssi) — en paralelo para todos los equipos del grupo.
4. Ejecutar Módulo 3 (eventos del mes).
5. Ejecutar Módulo 4 (completitud) — optimizar llamadas agrupando días en paralelo.
6. Imprimir resumen ejecutivo al final con los puntos críticos encontrados.

---

## Grupos de salud (22 en total — fase futura)

Buscar con `search_groups` sin argumento para listar todos los grupos disponibles.
Filtrar solo aquellos que correspondan a grupos de salud según nomenclatura conocida.
Durante el piloto, usar únicamente **Castellana**.
