# Informe de Monitoreo — Grupos de Salud Lynks

Este archivo define las instrucciones que Claude debe seguir al ejecutar el informe de monitoreo
de los grupos de salud. El informe se ejecuta bajo demanda o de forma semanal según se indique.

---

## Alcance actual (piloto — 2 grupos)

Durante la fase de afinamiento, ejecutar los siguientes grupos:

| Grupo       | ID                         | Equipos |
|-------------|----------------------------|---------|
| Castellana  | `6675fa70cefc9df3e4fdd9e6` | 13      |
| Grupo AFIN  | `65afc216fee8ffeae08cd41a` | 33      |

Una vez validado el piloto, extender a los 22 grupos de salud.

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

## Módulo 1 — Bolsa de notificaciones consumida

**Objetivo**: mostrar el % de bolsa de correo, SMS, llamadas y WhatsApp consumida
a la fecha para cada grupo/cliente.

**Fuente de datos**: Google Sheet actualizado diariamente.
- **ID del sheet**: `1d0-FzqTUDBPoBqqaWkU6J6438zfQRHt8RMZyDV4taIo`
- **Hoja**: `GroupsAGRO-SALUD` (204 filas, sectores SALUD y AGRO)
- **Herramienta**: conector Google Drive — `read_file_content` con `fileId: "1d0-FzqTUDBPoBqqaWkU6J6438zfQRHt8RMZyDV4taIo"`

**Columnas del sheet**:
| Columna          | Descripción                          |
|------------------|--------------------------------------|
| `_id`            | ID del grupo en Lynks                |
| `name`           | Nombre del cliente/grupo             |
| `email_shots`    | Correos enviados (usado)             |
| `email_bag`      | Límite de correos del plan           |
| `PORCENTAJE EMAIL` | % consumido (ya calculado)         |
| `sms_shots`      | SMS enviados (usado)                 |
| `sms_bag`        | Límite de SMS del plan               |
| `PORCENTAJE SMS` | % consumido (ya calculado)           |
| `call_shots`     | Llamadas realizadas (usado)          |
| `call_bag`       | Límite de llamadas del plan          |
| `PORCENTAJE LLAMADA` | % consumido (ya calculado)       |
| `wpp_shots`      | WhatsApp enviados (usado)            |
| `wpp_bag`        | Límite de WhatsApp del plan          |
| `PORCENTAJE WPP` | % consumido (ya calculado)           |
| `enable`         | Si el grupo está activo              |
| `points`         | Número de equipos                    |

**Nota sobre la lectura del sheet**:
`download_file_content` exporta la **primera hoja activa** del libro como CSV completo (base64).
La hoja `GroupsAGRO-SALUD` debe ser la primera hoja del libro (ya configurado).
`read_file_content` solo devuelve una muestra parcial — no usar para filtrar filas específicas.

**Instrucciones**:
1. Llamar `download_file_content(fileId: "1d0-FzqTUDBPoBqqaWkU6J6438zfQRHt8RMZyDV4taIo")`.
   La respuesta contiene el campo `content` en base64 — decodificarlo para obtener el CSV.
2. Parsear el CSV. Columnas (en orden): `_id, name, Puntos, Sector, email_shots, email_bag,
   PORCENTAJE EMAIL, sms_shots, sms_bag, PORCENTAJE SMS, call_shots, call_bag,
   PORCENTAJE LLAMADA, wpp_shots, wpp_bag, PORCENTAJE WPP, enable, points`
3. Para el informe piloto (Castellana), buscar la fila con `_id == "6675fa70cefc9df3e4fdd9e6"`.
   Para el informe de todos los grupos de salud, filtrar por `Sector == "SALUD"`.
4. Leer `PORCENTAJE EMAIL`, `PORCENTAJE SMS`, `PORCENTAJE LLAMADA`, `PORCENTAJE WPP`.
4. Por cada fila filtrada, leer `PORCENTAJE EMAIL`, `PORCENTAJE SMS`,
   `PORCENTAJE LLAMADA`, `PORCENTAJE WPP`.
5. Si el valor es `#DIV/0!` → significa que la bolsa tiene límite 0 (no contratada).
   Reportar como "— Sin bolsa" y excluir del semáforo.
6. Si el valor es un porcentaje numérico, aplicar semáforo:
   - < 70% → ✅ Normal
   - 70–89% → ⚠️ Advertencia
   - ≥ 90% → 🔴 Crítico (bolsa casi agotada — notificar al cliente)
   - > 100% → 🔴🔴 **EXCEDIDO** (el grupo ya superó su bolsa contratada)
7. Si el conector Google Drive no está disponible en la sesión, indicar:
   > ⚠️ **Conector Google Drive no activo en esta sesión.**
   > Actívalo en Configuración → Conectores y vuelve a ejecutar el informe.

**Formato de salida**:
```
## Módulo 1 — Consumo de bolsa (al DD/MM/YYYY)

| Grupo      | Correo        | % Email | SMS         | % SMS | Llamadas    | % Call | WhatsApp    | % WPP |
|------------|--------------|---------|------------|-------|------------|--------|------------|-------|
| Castellana | 450 / 1,000  | 45% ✅  | 120 / 500  | 24% ✅ | 10 / 100   | 10% ✅ | 80 / 200   | 40% ✅ |
```

> 🔴 Grupos con cualquier canal ≥ 90% requieren renovación de bolsa urgente.

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

## Módulo 4 — Completitud de datos / Pérdida de transmisión

**Objetivo**: identificar equipos que enviaron significativamente menos registros de los
esperados según su frecuencia configurada, indicando posible pérdida de datos en el mes.

**Por qué no usar `check_group_data_completeness` día por día**:
En la práctica, todos los equipos retornan `overall_status: "partial"` cada día (nunca
"complete") porque la plataforma exige el 100% exacto de transmisiones en la ventana.
Esto hace que el indicador sea inútil para diferenciar equipos con buen vs mal comportamiento.

**Herramienta**: `get_point_data` con `range: "this_month"` por cada equipo.

**Instrucciones**:
1. Para cada equipo del grupo, llamar `get_point_data` con `range: "this_month"`.
2. Leer de la respuesta:
   - `data_quality.count_variables`: número de registros recibidos en el mes.
   - `point.frequency`: frecuencia de transmisión configurada (en minutos).
3. Calcular registros esperados:
   - `dias_transcurridos` = día actual del mes (ej. 28 para sep-28).
   - `transmisiones_por_dia` = 1440 / frecuencia_minutos (ej. 15 min → 96/día, 60 min → 24/día).
   - `esperado` = dias_transcurridos × transmisiones_por_dia.
4. Calcular:
   - `% recibido` = `(registros_reales / esperado) × 100`
5. Clasificar:
   - ≥ 90% → ✅ Normal
   - 70–89% → ⚠️ Parcial
   - < 70% → 🔴 Crítico (pérdida significativa)
6. Si `point.frequency` no está disponible o es `null` → reportar como
   "Sin frecuencia configurada" y excluir del cálculo.

**Nota**: Si `get_point_data` no retorna `count_variables`, usar
`check_point_data_completeness` con `start` = primer día del mes y `end` = hoy
para obtener el resumen mensual de un equipo en una sola llamada.

**Formato de salida**:
```
## Módulo 4 — Completitud de datos (mes en curso, N=28 días evaluados)

| Equipo                          | Frecuencia | Registros reales | Esperados | % Recibido | Estado      |
|---------------------------------|-----------|-----------------|-----------|-----------|-------------|
| Nevera 1 Farmacia principal...  | 15 min    | 2,688 / 2,688   | 2,688     | 100%      | ✅ Normal   |
| Nevera 2 Farmacia cirugía...    | 15 min    | 1,450 / 2,688   | 2,688     | 54%       | 🔴 Crítico  |
| Analizador Redes...             | —         | —               | —         | —         | Sin frecuencia |
```

Al final: resumen con conteo por estado y % global de transmisión del grupo.

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
