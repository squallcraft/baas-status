# BaaS Status Page

Página pública de estado del servicio BaaS de Trackingtech.

URL final: `https://status.baas.trackingtech.cl` (con custom domain) o
`https://squallcraft.github.io/baas-status` (sin custom domain).

## Cómo funciona

- **Monitoreo automático cada 5 minutos**: GitHub Action que hace fetch al
  `/health` del BaaS y persiste el resultado en `data/uptime.json` (rolling
  window de 30 días).
- **Check en vivo desde el browser cada 30 segundos**: el `index.html` pollea
  el `/health` directo y pinta el estado actual + latencia.
- **Incidentes manuales**: para reportar un incidente, agregar un archivo
  markdown nuevo en `incidents/` (copiar el `TEMPLATE.md`) y reflejarlo en
  `data/incidents.json` para que aparezca en la página.

## Costo

**Cero pesos**. GitHub Pages es gratuito para repos públicos, y GitHub
Actions tiene cuota de 2.000 minutos/mes para repos públicos (este
workflow consume ~24 hs/mes, holgado dentro del límite).

## Setup inicial (una sola vez)

### Paso 1 — Crear el repo en GitHub

```bash
# Desde tu Mac, en la carpeta del kit:
cd /Users/oscarguzman/Library/Application\ Support/Claude/local-agent-mode-sessions/ea2d94d8-4fa3-46d1-b383-78d2184ca4fe/09fa29a7-aaf9-4058-b16a-0680de167950/local_7a22f6c8-332a-4096-9bcb-a791adf3b261/outputs/baas-status

git init
git branch -M main
git add .
git commit -m "feat: status page inicial"
```

Después en GitHub: crear repo nuevo público en
`https://github.com/squallcraft/baas-status`.

```bash
git remote add origin git@github.com:squallcraft/baas-status.git
git push -u origin main
```

### Paso 2 — Activar GitHub Pages

En el repo de GitHub:

1. `Settings` → `Pages` (menú lateral)
2. En "Source" elegir **Deploy from a branch**
3. Branch: **main** / Folder: **/ (root)**
4. Save

GitHub va a tardar 1-2 minutos en publicar. La URL temporal será
`https://squallcraft.github.io/baas-status`. Probala en el browser.

### Paso 3 — (Opcional) Custom domain `status.baas.trackingtech.cl`

#### 3.1 Configurar DNS

En el panel de tu proveedor DNS (donde controles `trackingtech.cl`):

```
Tipo:  CNAME
Host:  status.baas
Valor: squallcraft.github.io
TTL:   3600
```

#### 3.2 Agregar archivo CNAME al repo

```bash
echo "status.baas.trackingtech.cl" > CNAME
git add CNAME
git commit -m "feat: custom domain"
git push
```

#### 3.3 Configurar en GitHub Pages

En el repo: `Settings` → `Pages`:

- Custom domain: `status.baas.trackingtech.cl`
- Click **Save**
- Cuando el DNS propague (5 min a 1 hora), tildar **Enforce HTTPS**

GitHub provee certificado SSL automático vía Let's Encrypt.

### Paso 4 — Habilitar el workflow

El workflow se ejecuta automáticamente cada 5 minutos según el cron. Para
probarlo manualmente la primera vez:

1. En el repo: tab **Actions**
2. Seleccionar workflow **BaaS Health Check**
3. Click **Run workflow** → **Run workflow**

A los 30-60 segundos debería commitear `data/uptime.json` con la primera
entrada. La página la mostrará en la próxima recarga.

### Paso 5 — Verificación final

Abrir la status page en el browser:

- Hero debería pintarse en verde con "Todos los sistemas operacionales".
- Componentes: API, Database, Redis — los tres en verde.
- Métricas: latencia actual en ms. Uptime 24h aparece "—" hasta que pasen
  algunas horas con data acumulada.
- Timeline 24h: barras grises (sin data) o verdes (con data acumulada).
- Incidentes: lista vacía o el incidente de ejemplo del deploy del 2026-06-17.

## Reportar un incidente nuevo

Cuando haya un incidente real (downtime, degradación, deploy con impacto):

### Opción A — Editar desde la UI de GitHub (más rápido para incidentes urgentes)

1. Ir a `data/incidents.json` en GitHub
2. Click ✏️ (editar)
3. Agregar el nuevo incidente al inicio del array:

```json
[
  {
    "date": "2026-06-25T14:30:00-03:00",
    "title": "Haulmer respondió 503 durante 15 minutos",
    "status": "resolved",
    "body": "Haulmer tuvo un incidente de su lado entre 14:30 y 14:45.\nDurante ese período las emisiones quedaron en retry y completaron al volver.\nNingún DTE perdido."
  }
]
```

4. Commit con mensaje "incident: Haulmer 503"

GitHub Pages rebuildea automáticamente. La página actualizada aparece en
~1 min.

### Opción B — Documentación detallada en `/incidents/`

Para incidentes mayores, además de actualizar el JSON, crear un markdown
detallado:

```bash
cp incidents/TEMPLATE.md incidents/2026-06-25-haulmer-503.md
# Editarlo, commitear, push.
```

El markdown queda como historial técnico (postmortem completo) y no se
muestra en la página automáticamente — sirve como referencia interna.

## Severidades de incidente

- **investigating** — incidente en curso, todavía investigando causa.
- **major** — impacto alto, varios componentes afectados.
- **resolved** — ya resuelto, queda en el histórico de transparencia.

## Estructura del repo

```
baas-status/
├── index.html                          # status page (todo inline)
├── README.md                           # este archivo
├── CNAME                               # custom domain (creado en setup)
├── .github/
│   └── workflows/
│       └── check-health.yml            # cron cada 5 min
├── data/
│   ├── uptime.json                     # rolling 30 días (autogenerado)
│   └── incidents.json                  # incidentes manuales
└── incidents/
    ├── TEMPLATE.md                     # plantilla para postmortems
    └── 2026-06-17-sandbox-deploy.md    # ejemplo
```

## Mantenimiento

- **Resetear el histórico**: borrar `data/uptime.json`, reemplazar por `[]`,
  commitear. La próxima ejecución del workflow empezará a llenarlo de nuevo.
- **Cambiar el intervalo de check**: editar el cron en
  `.github/workflows/check-health.yml` línea 12 (default `*/5 * * * *` =
  cada 5 min). El mínimo de GitHub Actions es 5 min, no se puede menos.
- **Cambiar URL monitoreada**: editar el campo `HEALTH_URL` en `index.html`
  (línea ~313) y la línea del curl en `check-health.yml` (~38).

## Troubleshooting

### El workflow falla con "Permission denied"

Verificar en `Settings` → `Actions` → `General` → `Workflow permissions`
que esté seleccionado **Read and write permissions**.

### Custom domain no funciona ("404 No such host")

- Verificar que el DNS propagó: `dig status.baas.trackingtech.cl`. Debería
  responder con CNAME a `squallcraft.github.io`.
- Esperar hasta 24 horas para propagación DNS global.
- Si después de 24h sigue fallando, revisar en `Settings` → `Pages` el
  estado del custom domain (debería decir "DNS check successful").

### La página dice "Servicio no responde" pero el BaaS está arriba

Probablemente CORS. Verificar que `https://baas.trackingtech.cl/health`
responda con header `Access-Control-Allow-Origin: *` o con el dominio
específico de la status page. El BaaS hoy tiene CORS abierto (`*`), así
que no debería pasar — pero si pasa, hay que actualizar
`app/main.py:33-39` para incluir el dominio de la status page.
