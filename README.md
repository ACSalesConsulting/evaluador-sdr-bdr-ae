# Mapa de Brechas — Evaluador SDR/BDR/AE

App estática de autodiagnóstico basada en [`../framework_competencias_sdr_bdr_ae.md`](../framework_competencias_sdr_bdr_ae.md) y [`../preguntas-evaluador-sdr-bdr-ae.md`](../preguntas-evaluador-sdr-bdr-ae.md). Sin build, sin dependencias: es un único `index.html` con CSS y JS embebidos.

## Correr localmente

Abrí `index.html` directo en el navegador, o serví la carpeta con cualquier servidor estático:

```bash
npx serve .
```

## Deploy en Vercel

1. Subí esta carpeta (`app/`) a un repo de GitHub.
2. En Vercel: **Add New → Project → Import** ese repo.
3. Framework preset: **Other** (es HTML estático, no hay build step). Root directory: la carpeta donde está `index.html`.
4. Deploy. Vercel te da una URL (`*.vercel.app`); podés conectar un dominio propio después en Project Settings → Domains.

Cada push a la rama principal vuelve a deployar automáticamente.

## Configurar n8n (captura de email + puntajes)

El formulario de "enviarme el resultado" (pantalla de resultado) hace un `POST` a la constante `WEBHOOK_URL` definida al principio del `<script>` en `index.html`. Hoy apunta a un placeholder:

```js
const WEBHOOK_URL = "https://TU-INSTANCIA-N8N.example.com/webhook/evaluador-sdr-bdr-ae";
```

Pasos:

1. **Elegí dónde corre n8n.** Dos caminos:
   - **n8n Cloud** (más simple, plan pago desde ~USD 20/mes): creás la instancia desde n8n.io, sin infra que mantener.
   - **Self-host** (gratis, requiere mantenimiento propio): un contenedor Docker de `n8nio/n8n` en Railway, Render o una VPS chica. Cualquiera de las dos alcanza para este volumen — no hace falta nada más grande.
2. Dentro de n8n, creá un workflow nuevo con un nodo **Webhook**:
   - Method: `POST`
   - Path: por ejemplo `evaluador-sdr-bdr-ae`
   - Respond: "Immediately" con status 200 (el formulario no necesita nada en la respuesta, solo que no falle)
3. Copiá la **Production URL** del nodo Webhook (no la de test) y pegala en `WEBHOOK_URL` en `index.html`.
4. A partir del nodo Webhook, encadená lo que quieras hacer con cada envío — típicamente:
   - Un nodo **Google Sheets / Airtable** para guardar la fila (lead + puntajes).
   - Un nodo de **email** (Gmail/SMTP) para mandarle el resultado a la persona, o para avisarte a vos de un nuevo diagnóstico.
5. **CORS:** el navegador va a hacer el POST directo desde el dominio de Vercel hacia n8n. Si n8n corre en un dominio distinto, revisá que el nodo Webhook no esté bloqueando el origen (por defecto n8n acepta cross-origin en sus webhooks; si self-hosteás detrás de un proxy propio, agregá el header `Access-Control-Allow-Origin` para tu dominio de Vercel).

### Payload que manda el formulario

```json
{
  "email": "persona@empresa.com",
  "timestamp": "2026-09-10T14:32:00.000Z",
  "rol_actual": "SDR",
  "rol_objetivo": "BDR",
  "banda": { "nivel": 3, "label": "Cerca del salto" },
  "puntaje_total": 2.43,
  "puntaje_por_categoria": {
    "Prospección y generación de demanda": 2.5,
    "Diagnóstico y calificación": 2.75,
    "Comunicación y persuasión": 2.0,
    "Gestión y método": 2.6,
    "Autogestión y desarrollo": 2.25
  },
  "brechas_ciegas": ["Presentación y demo"],
  "top3_prioridades": [
    { "competencia": "Negociación comercial", "categoria": "Comunicación y persuasión", "brecha": 3, "metodo": "OBS+RES" }
  ],
  "respuestas_detalle": [
    { "competencia": "Investigación de cuentas y armado de listas", "nivel": 3, "evidencia_index": 0 }
  ]
}
```

`respuestas_detalle` trae las 21 respuestas completas (útil si querés armar seguimiento o coaching personalizado a partir del diagnóstico, no solo el resumen).

### Anti-spam

El formulario tiene un campo honeypot oculto (`cap-hp`). Si un bot lo completa, el envío se descarta en el navegador antes de llegar a n8n — no hace falta lógica extra del lado de n8n para esto.

## Qué falta para producción real

- [ ] Reemplazar `WEBHOOK_URL` por la URL real una vez armada la instancia de n8n.
- [ ] Decidir si querés dominio propio en Vercel o alcanza con `*.vercel.app`.
- [ ] (Opcional) sumar el logo de Reunión de Forecast — hoy el crédito es solo texto ("Un diagnóstico de Reunión de Forecast").
- [ ] Pasar por QA y Tester (siguientes etapas del pipeline) antes de compartir el link públicamente.
