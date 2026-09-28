# Análisis de un incidente de pagos + alerta automática con n8n

Mini proyecto de soporte técnico y datos: investigar por qué fallan los pagos, clasificar la gravedad, avisar al cliente y automatizar la detección para que el equipo se entere en minutos.

> Los nombres de proveedores, billers y partners están anonimizados. Los datos son de un ejercicio práctico.

## Qué contiene

| Archivo | Descripción |
|---|---|
| `README.md` | Análisis del incidente y explicación del workflow |
| `payment_failure_alert_workflow.json` | Workflow de n8n (sin credenciales) |

## El caso

Un partner avisa de que hay pagos fallando. Con una tabla de transacciones y un log de la aplicación, hay que responder: ¿qué falla, desde cuándo, a quién afecta y a quién se escala?

## Parte 1: Análisis del incidente

**Cuándo empezó:** a las 14:09:33 fallaron los primeros pagos. Todas las transacciones anteriores (12:31 a 14:07) fueron exitosas, así que el corte fue abrupto, no gradual.

**Qué está fallando:** un único proveedor (Proveedor A) con un único biller (electricidad), con el mismo error: `PROVIDER_TIMEOUT` (30 segundos sin respuesta).

```sql
SELECT provider, biller_id, status, error_code, COUNT(*) AS total
FROM transactions
GROUP BY provider, biller_id, status, error_code
ORDER BY provider, biller_id, status;
```

| provider | biller | status | total |
|---|---|---|---|
| Proveedor A | Electricidad | success | 6 |
| Proveedor A | Electricidad | failed | 23 |
| Proveedor B | Telefonía/Internet (3 billers) | success | 11 |

El 100% de los fallos es Proveedor A + electricidad. El Proveedor B no tiene ningún fallo en el mismo periodo, lo que descarta un problema general del sistema.

**A quién afecta:** a más de un partner.

```sql
SELECT partner_id, COUNT(*) AS failed_txns
FROM transactions
WHERE provider = 'proveedor_a' AND status = 'failed'
GROUP BY partner_id;
```

| partner | fallos |
|---|---|
| Partner 1 | 20 |
| Partner 2 | 3 |

**Por qué importa:** si solo fallara un partner, habría que revisar su configuración. Como el mismo proveedor falla con partners distintos, la causa está en el proveedor.

**Qué confirma el log:** el mismo error en todos los casos, con una latencia constante de unos 30 000 ms y tres reintentos por transacción. Un proveedor externo que deja de responder suele dar este patrón, mientras que un bug propio daría errores más variados.

### Clasificación: P1 (crítico)

- Afecta a varios partners a la vez.
- Tasa de fallo cercana al 79% (23 de 29 transacciones).
- Servicio esencial para el usuario final (pago de electricidad).
- Sin señales de recuperación hasta el último registro del log.

**Escalado:** al equipo que gestiona la integración con el Proveedor A, no al equipo del partner. En paralelo, abrir un canal interno de incidente para coordinar las actualizaciones.

## Parte 2: Mensaje al cliente

Mensaje corto y sencillo, en inglés:

> **Subject: We're looking into the electricity payment issues**
>
> Hi team,
>
> Thanks for the message. We already started checking this.
>
> We can confirm there are more failed payments on this biller since around 14:09. This is not only happening with your account, we also see it with another partner, so we think the problem is on one of our payment providers, not on your side.
>
> Other billers are working fine.
>
> We are talking to the provider team now to fix this. We will send you an update in the next 15 minutes.
>
> Sorry for the trouble, we are working on it.

## Parte 3: Alerta automática con n8n

Un workflow que vigila los fallos sin que nadie tenga que mirar logs.

```
Schedule Trigger → Edit Fields → SQL query → If → Claude API → Slack
```

1. **Schedule Trigger:** se ejecuta cada 5 minutos.
2. **Edit Fields:** aquí se configuran los tres parámetros del workflow:
   - `window_minutes`: cuántos minutos hacia atrás se analizan (15).
   - `error_rate_threshold_pct`: a partir de qué porcentaje de fallos salta la alerta (30).
   - `min_sample_size`: mínimo de transacciones para que el dato cuente (5), para evitar falsas alarmas con muy pocos datos.
3. **SQL query:** calcula, por proveedor y biller, el total de transacciones, los fallos y el porcentaje de error dentro de la ventana, y devuelve solo los que superan el umbral y el mínimo.
4. **If:** comprueba si la consulta devolvió algún resultado. Si no hay incidente, el workflow se detiene aquí.
5. **Claude API:** convierte los datos en bruto en un resumen legible: qué proveedor falla, cuántos fallos y qué revisar.
6. **Slack:** envía ese resumen al canal del equipo.

**Por qué esta idea:** automatizar la detección para que el equipo se entere del incidente en minutos y no cuando un cliente se queje.

### Qué mejoré sobre la primera versión

En la primera versión, el umbral (30%) y la ventana (15 minutos) estaban escritos directamente dentro del SQL y el mínimo de transacciones no se aplicaba. Ahora los tres valores se leen desde el nodo `Edit Fields`, así que se cambian en un solo sitio. Lo probé cambiando cada valor y comprobando que el resultado cambiaba (por ejemplo, con el umbral en 60 ya no salta la alerta).

### Para usarlo

1. Importar `payment_failure_alert_workflow.json` en n8n.
2. Crear y asignar tus propias credenciales de PostgreSQL y Slack.
3. Poner tu propia API key de Anthropic en el nodo HTTP Request.
4. Adaptar el canal de Slack y los parámetros del nodo `Edit Fields`.

## Lo que aprendí

- Usar SQL (`GROUP BY`, `FILTER`, CTEs) para encontrar la causa de un incidente.
- Distinguir un problema del proveedor de un problema de un cliente concreto.
- Clasificar la gravedad y decidir a quién escalar.
- Comunicar un incidente en un inglés claro y sencillo.
- Construir mi primera automatización con n8n desde cero y corregirla al revisarla.
