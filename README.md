# EDS FILE 048 · Dashboard de Ventas y Comisiones

Dashboard web hecho con **Streamlit** para analizar las ventas diarias de una
estación de servicio y calcular las comisiones por producto y por cajero.

La app permite:

- Cargar ventas desde **archivos Excel** locales o desde **Google Sheets** (vía URL de exportación CSV).
- Mantener una tabla de **comisiones** que se lee desde Google Sheets o desde un Excel local de respaldo.
- Filtrar por **mes de negocio (26 → 25)**, rango de fechas, vendedor (cajero), producto y método de pago.
- Ver **KPIs**, gráficos interactivos (Plotly) y tablas resumen.
- **Exportar** un reporte de comisiones a Excel (.xlsx) con varias hojas.

---

## Tabla de contenido

1. [Estructura del proyecto](#estructura-del-proyecto)
2. [Requisitos](#requisitos)
3. [Cómo lanzar la app en local](#cómo-lanzar-la-app-en-local)
4. [Fuentes de datos soportadas](#fuentes-de-datos-soportadas)
5. [Google Sheets: cómo conectar las hojas](#google-sheets-cómo-conectar-las-hojas)
6. [Caché y botón “Recargar datos ahora”](#caché-y-botón-recargar-datos-ahora)
7. [Formato esperado de los datos](#formato-esperado-de-los-datos)
8. [Despliegue en Streamlit Cloud](#despliegue-en-streamlit-cloud)
9. [Solución de problemas (FAQ)](#solución-de-problemas-faq)

---

## Estructura del proyecto

```
dasboard-estacion-pruebas/
├─ app_estacion.py              # Aplicación principal (Streamlit)
├─ requirements.txt             # Dependencias Python
├─ comisiones_guardadas.json    # Persistencia interna de la UI de comisiones
├─ assets/
│   ├─ favicon.png              # Ícono cuadrado para la pestaña del navegador
│   └─ eds_logo.png             # Logo / imagen original
├─ .streamlit/
│   └─ secrets.toml.example     # Plantilla de secretos (no subir secrets reales)
├─ datos/                       # (opcional, NO se versiona) Excels locales de ventas / comisiones
├─ .gitignore
└─ README.md
```

> La carpeta `datos/` y el archivo real `.streamlit/secrets.toml` están ignorados
> por Git para no exponer información sensible.

---

## Requisitos

- **Python 3.10+** (probado en 3.12).
- Conexión a internet (si vas a usar Google Sheets).
- Dependencias listadas en `requirements.txt`:
  - `streamlit`, `pandas`, `openpyxl`, `plotly`, `requests`.

---

## Cómo lanzar la app en local

Desde la carpeta del proyecto, abrir una terminal (PowerShell en Windows) y:

```powershell
# 1) Crear entorno virtual (recomendado, solo la primera vez)
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# 2) Instalar dependencias
pip install -r requirements.txt

# 3) (Opcional) Configurar secrets locales para Google Sheets
copy .streamlit\secrets.toml.example .streamlit\secrets.toml
# y edita .streamlit/secrets.toml con tus URLs reales

# 4) Lanzar Streamlit
streamlit run app_estacion.py
```

La app quedará disponible en `http://localhost:8501`.

> En Linux/Mac los pasos son iguales cambiando `Activate.ps1` por
> `source .venv/bin/activate` y `copy` por `cp`.

---

## Fuentes de datos soportadas

La app puede cargar **ventas** desde 3 sitios (en este orden de prioridad):

1. **Carga manual** (uploader): archivos `.xlsx` arrastrados en la UI.
2. **Carpeta local `datos/`**: todos los `.xlsx` que estén dentro.
3. **Google Sheets (CSV)**: URL pública de exportación CSV.

Para **comisiones** la app intenta, en orden:

1. **Google Sheets (CSV)** vía `COMMISSIONS_CSV_URL` en secrets.
2. **Excel local** `datos/COMISION.xlsx` (fallback para uso offline).

---

## Google Sheets: cómo conectar las hojas

1. En tu Google Sheet, **Compartir → "Cualquiera con el enlace" como Lector**.
2. Selecciona la pestaña que quieres exportar; mira la URL del navegador y copia:
   - **ID**: lo que está entre `/d/` y `/edit`.
   - **gid**: el número después de `gid=` (cada pestaña tiene uno distinto).
3. Arma la URL de exportación CSV:

   ```
   https://docs.google.com/spreadsheets/d/ID/export?format=csv&gid=GID
   ```

4. Pégala en `secrets.toml` (local) o en **Settings → Secrets** (Streamlit Cloud):

   ```toml
   SHEETS_CSV_URL       = "https://docs.google.com/spreadsheets/d/ID_VENTAS/export?format=csv&gid=GID_VENTAS"
   COMMISSIONS_CSV_URL  = "https://docs.google.com/spreadsheets/d/ID_COMISIONES/export?format=csv&gid=GID_COMISIONES"
   ```

> Puedes usar **el mismo spreadsheet con dos pestañas distintas** (ventas y comisiones);
> cambia solo el `gid` en cada URL.

---

## Caché y botón "Recargar datos ahora"

Para que la app sea **fluida** al filtrar, las descargas de Google Sheets están cacheadas:

| Función                                | TTL    |
|----------------------------------------|--------|
| `cargar_ventas_desde_url_csv`          | 1 hora |
| `cargar_comisiones_desde_url_csv`      | 6 horas|

Si editas el Sheet y quieres ver los cambios **en el acto**, en la sección
**📁 Fuente de Datos** pulsa **🔄 Recargar datos ahora**: limpia la caché,
reinicia la tabla de comisiones en memoria y recarga.

---

## Formato esperado de los datos

### Ventas (Excel o Sheet)

Columnas que la app reconoce (no hace falta que estén todas, pero `Fecha` es obligatoria):

- `Fecha` (formato fecha; admite `dd/mm/aaaa`)
- `Hora` (texto `HH:MM`)
- `Cod Producto`
- `Descripcion`
- `Cantidad`
- `Valor`
- `Nombre Cajero`
- `MOP1` (método de pago)

Si el archivo trae filas decorativas al inicio (cabecera unas filas abajo),
la app **busca automáticamente** la primera fila que contenga la palabra
`Fecha` y la toma como encabezado.

### Comisiones (Sheet)

Encabezados mínimos:

| CODIGO | COMISION |
|--------|----------|
| 12345  | 500      |
| 67890  | 1200     |

- La columna de **clave** puede llamarse `CODIGO`, `Cod Producto`, `Producto`, `Descripcion`, etc.
- La columna de **valor** puede llamarse `COMISION`, `COMISIÓN`, `COMMISSION`.
- Los números deben ser **números reales** (sin texto). Se aceptan
  formatos con coma decimal (`1500,5`) y separadores de miles (`1.500,50`).

---

## Despliegue en Streamlit Cloud

1. Sube el repositorio a GitHub (sin `datos/`, sin `secrets.toml`).
2. En [share.streamlit.io](https://share.streamlit.io/) → **New app**.
3. Selecciona el repo, la rama y como archivo principal **`app_estacion.py`**.
4. En **Settings → Secrets** pega los TOML con `SHEETS_CSV_URL` y `COMMISSIONS_CSV_URL`.
5. La app desplegará y será accesible por una URL pública.

Para actualizar: `git push` desde tu equipo (o GitHub Desktop) y Streamlit Cloud
volverá a desplegar automáticamente.

---

## Solución de problemas (FAQ)

**No carga ventas desde Google Sheets**
- Verifica que la URL termine en `/export?format=csv&gid=...`.
- Abre la URL en una ventana de incógnito: debe descargar/mostrar texto CSV.
- Revisa el permiso "Cualquiera con el enlace (Lector)".

**No carga comisiones desde Google Sheets (cae al Excel)**
- Asegúrate de tener `COMMISSIONS_CSV_URL` en secrets (válido TOML, con comillas dobles).
- Encabezados sugeridos: `CODIGO` y `COMISION`.
- Los valores numéricos no deben venir como texto con símbolos raros (`$`, `mil`, etc.).

**"Streamlit secrets file is empty" / "formato inválido"**
- En el panel de Secrets debes pegar TOML válido, ej:
  ```toml
  SHEETS_CSV_URL = "https://..."
  ```
  Una URL suelta sin clave **no** es TOML válido.

**No me aparecen cambios en GitHub Desktop**
- Asegúrate de que GitHub Desktop esté apuntando a la **misma carpeta**
  que estás editando (Repository → Show in Explorer).

**Quiero borrar manualmente la caché**
- Usa el botón **🔄 Recargar datos ahora**, o reinicia la app
  (Streamlit Cloud → **Manage app → Reboot**).

---

## Créditos

Aplicación desarrollada para uso interno de la estación. Streamlit + Pandas + Plotly.
