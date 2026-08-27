# Mi Cartera — plantilla

PWA de finanzas personales para **una sola persona**, pensada para clonarse: cada usuario
tiene su propia copia del código y su propio proyecto de Firebase. Sin datos de nadie
precargados.

## Stack

- **HTML/CSS/JS puro en un solo `index.html`.** Sin frameworks, sin build, sin npm.
- **Firebase Firestore** (sincronización) y **Firebase Authentication** (email/password).
- Se publica tal cual en GitHub Pages.
- Archivos: `index.html`, `manifest.json`, `sw.js`, `icon-192.png`, `icon-512.png`,
  `firestore.rules`.

## Antes de usarla: 3 cosas que hay que personalizar

1. `index.html` → `firebaseConfig`: los 6 valores del proyecto de Firebase propio.
2. `index.html` → `ALLOWED_EMAIL`: el correo que puede entrar.
3. `firestore.rules` → el mismo correo, y publicar las reglas en Firebase.

Ver `README.md` para el paso a paso completo.

## Qué trae

- **Inicio**: patrimonio total, tarjetas por cuenta, "cómo vas este mes", actualizar saldos,
  transferencias, tarjetas de crédito, registro de compras de acciones, temas de color.
- **Movimientos**: alta de gastos/ingresos e historial filtrable por tipo y periodo.
- **Metas**: metas de fin de año, proyección "camino a diciembre" y apartados de ahorro.
- **Perfil**: foto, datos personales, estándar financiero personalizado y calificación de hábitos.
- **Noticias**: titulares de tus acciones y de bancos vía RSS público.
- **Deudas**: quién te debe y a quién le debes, con historial por persona.
- **Resumen**: cierre del mes/año con coach, utilidades ("cuánto ganó tu dinero"), pasteles
  interactivos, categorías y gráficas de evolución.

## Conceptos centrales del código

- `calcCompoundAt(base, fechaStr, tasaPct, ms)`: interés compuesto diario (tasa nominal /365,
  días calendario completos). **Toda** cuenta que genere rendimiento pasa por aquí; nunca
  hardcodear un monto fijo por día.
- `parseFechaLocal()` / `hoyLocal()` / `msFechaMov()`: manejo de fechas en horario local para
  evitar el clásico bug de que `YYYY-MM-DD` se lea como UTC y muestre el día anterior.
- **Patrón de escritura**: mutación optimista + `writeBatch`/`runTransaction` + rollback en
  `catch` con `backupCuentas()`/`restoreCuentas()`. Si agregas un campo nuevo al estado que
  se mute en un flujo, agrégalo también al respaldo.
- `calcularCierre()` es un **modelo puro** (no toca el DOM); `textoCoach()`/`textoAnalisis()`
  generan texto; `cierreMesHTML()` pinta. No mezclar esas capas.
- `ESTANDARES` + `estandarDe(perfil)`: única fuente de umbrales financieros (ahorro objetivo,
  gasto máximo, meses de emergencia). Coach, cierre, racha y score los consumen; no
  hardcodear porcentajes sueltos.
- **Cuentas**: el usuario las configura desde la app, en el acordeón "mis cuentas" de
  Inicio. El catálogo de slots (`SLOTS`) y sus capacidades por tipo (`CAPS`) son fijos e
  inmutables: los gastos, ingresos, transferencias y deudas guardan la CLAVE del slot, no
  su nombre, así que **un slot nunca se reasigna a otro banco una vez que tiene historia**
  (renombrarlo reescribe cómo se lee todo su historial). Lo configurable es si la cuenta
  está activa, cómo se llama y, en las tarjetas, el día de corte y si es normal o
  garantizada. Vive en el documento `<NS>/cuentas` y las reglas de Firestore ya lo cubren
  (`match /<NS>/{documento=**}` es recursivo).
  - `aplicarConfigCuentas()` recalcula los nombres (`CUENTAS`) y **todas** las listas
    derivadas, y refleja en el DOM qué cuentas están activas. Es idempotente y reversible
    (`hidden`/`disabled`, nunca `remove()`), y corre en cada `render()` más dos veces en el
    arranque. Si agregas una lista de cuentas nueva, **derívala ahí**; no la quemes.
  - Distingue listas de **alta** (solo cuentas activas: `FUENTES_GASTO`,
    `DESTINOS_INGRESO`, `CTAS_STOCK_OP`, `CUENTAS_DETALLE`, `DEUDA_CUENTAS_ALTA`) de las de
    **lectura y reversión** (todos los slots: `CUENTAS`, `DEUDA_CUENTAS`, las claves de
    `METAS_CUENTAS`). Un movimiento viejo de una cuenta apagada debe poder leerse y
    borrarse siempre.
  - Apagar una cuenta significa "no admite movimientos nuevos", **nunca** "sale del
    patrimonio": por eso no se deja apagar una con saldo distinto de cero, y una apagada
    que aún tenga dinero sigue mostrando su tarjeta.
  - Los nombres los escribe el usuario: al DOM van con `textContent` (vía `data-nom`), y
    a cualquier `innerHTML` **siempre** con `escapeHtml()`.
  - Límite conocido: 2 tarjetas de crédito, 2 cajitas de ahorro y 2 cuentas de banco por
    persona. Para más habría que añadir slots (con su HTML), no reescribir el modelo.
- **Día de corte**: `diaCorteMes()` y `cicloTarjeta(dia)`. Nunca uses `new Date(y,m,31)`
  para un corte: se desborda al mes siguiente. La barra de progreso mide el ciclo real
  (que no siempre son 30 días).

## Reglas de trabajo

1. **Un solo commit por tanda de cambios** — GitHub Pages atasca su cola de deploys si se
   suben varios commits seguidos.
2. Si cambias `sw.js`, **sube también la versión de `CACHE`** para invalidar el caché viejo.
3. `ALLOWED_EMAIL` es solo un gate de UI: la seguridad real está en `firestore.rules`.
4. Validar siempre lo que viene de Firestore (`Number.isFinite`, arrays, regex) — un dato
   corrupto no debe romper la app ni propagar `NaN` a los saldos.
5. Escapar con `escapeHtml()` cualquier texto del usuario que se interpole en HTML.
