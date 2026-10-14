# WAL-973 — Autocompletar DNI desde InformacionAdicional

## TLDR
- Prueba en **homo** que, cuando el banco manda el `Dni` vacío, CashIn lo completa extrayendo el CUIT de `InformacionAdicional` (regex + dígito verificador → 8 del medio).
- 12 requests en 5 carpetas: formatos que resuelven, casos borde de la conversión, regresión (el Dni del banco gana), rechazos y fallback a la tabla de Postgres.
- Todos hacen `POST /api/v1.0/Fondos/DepositarMovimientoRecaudadora`; los importes son únicos por escenario y por `runId` (subí `runId` en cada corrida para evitar duplicados).
- Requiere la API de homo (o local con `ASPNETCORE_ENVIRONMENT=HOMOLOGACION`) con las ramas de los PRs #460 y #461 desplegadas.
- La respuesta HTTP no dice de dónde salió el DNI: la prueba real es **mirar la base** (queries abajo).

## Escenarios

| # | Request | CUIT en el texto | Dni resultante | Cuenta (homo) | Esperado |
|---|---|---|---|---|---|
| 1.1 | PROVEEDORES | 20398105002 | 39810500 | CC 10000977 | 200, acreditado |
| 1.2 | VAR | 20398105002 | 39810500 | CC 10000977 | 200, acreditado |
| 1.3 | CUENTAS PROPIAS | 20398105002 | 39810500 | CC 10000977 | 200, acreditado |
| 1.4 | Sin tipo | 20398105002 | 39810500 | CC 10000977 | 200, acreditado |
| 2.1 | DNI de 7 dígitos | 20077677276 | 7767727 | CC 10000097 | 200, acreditado |
| 2.2 | Empresa CUIT 30 | 30590360763 | 59036076 | CC 10001302 | 200, acreditado |
| 2.3 | IOL (entidad excluida) | 33707852459 | 70785245 | — | 400 DNI/CUIT no coincide, no se acredita |
| 3.1 | Dni del banco + otro CUIT en el texto | 30590360763 | 39810500 (del banco) | CC 10000977 | 200, **no** va a la 10001302 |
| 4.1 | Dígito verificador inválido | 20398105003 | — | — | 400, documento del ordenante |
| 4.2 | Texto sin CUIT | — | — | — | 400, documento del ordenante |
| 4.3 | Solo `Etiquetas` con DOCUMENTO | — | — | — | 400 (fuera de alcance) |
| 5.1 | Texto exacto de la tabla | — | 27910950 (tabla) | sin cuenta | 400 DNI/CUIT no coincide (no por documento vacío) |

Los clientes de prueba tienen un único `NumeDocu`, una sola CC de tipo 1, producto 57 y no son blanqueo. Los CUIT tienen el dígito verificador válido (verificado).

## Verificación

**1. Postgres (Banks, homo)** — Dni con el que quedó cada depósito:

```sql
SELECT date, receipt, modified_movement->>'Dni' AS dni_final
FROM audit_logs
WHERE receipt LIKE 'WAL973-R<runId>-%'
ORDER BY date;
```

- Escenarios 1.x, 2.x, 3.1: `dni_final` completo (39810500, 7767727, 59036076, 70785245).
- 5.1: `dni_final` = 27910950.
- 4.x: `dni_final` vacío.
- No sirve comparar `original_movement` contra `modified_movement`: guardan el mismo objeto y el Dni sale igual en los dos. Para saber si el Dni lo puso el banco o la regex, mirar el label `dni_source` de la métrica; en 3.1 la prueba es la cuenta acreditada (query 2).

**2. IOL2 (homo)** — a qué cuenta fue cada depósito:

```sql
SELECT mr.comprobante, mr.dni, mr.id_estado_interno, cu.NumeCuen, cm.MontoXXX
FROM tcl_movimiento_recaudadora mr WITH (NOLOCK)
LEFT JOIN tcl_cuenta_movim cm WITH (NOLOCK) ON cm.ID_Cuenta_Movim = mr.ID_movimiento_cuenta
LEFT JOIN tcl_cuenta cu WITH (NOLOCK) ON cu.ID_Cliente = cm.ID_Cliente AND cu.id_tipo_cuenta = cm.ID_Tipo_Cuenta
WHERE mr.comprobante LIKE 'WAL973-R<runId>-%'
ORDER BY mr.comprobante;
```

- La columna `NumeCuen` tiene que ser la de la tabla de escenarios.
- 2.3: sin cuenta acreditada y estado interno `RechazadoFinalizado`.

**3. Métrica** — si `/metrics` del servicio es accesible: `cash_in_matching_resolution_count` con `dni_source` = `additional_info_regex` (1.x, 2.x), `bank` (3.1), `additional_info_table` (5.1) y `none` (4.x).

## Notas
- **2.3 y 5.1 devuelven 400 `DNI/CUIT no coincide con Cuenta Comitente`**: `FondearCuenta` responde eso cuando el Dni se completó pero no hay cuenta comitente que matchee (IOL es entidad excluida; el DNI 27910950 no existe en homo). 2.3 está confirmado en una corrida real; 5.1 está inferido del mismo camino de código y falta confirmarlo.
- **5.1 depende de datos de homo**: la fila id 3 de `cuit_additional_information`. Si alguien la borra, el escenario pasa a 400.
- Cada corrida acredita plata de prueba en CC de homo; no usar en producción.
- Para repetir la corrida: subir `runId` en el environment (1 a 99).

---
← [Colección CashIn](../opencollection.yml)
