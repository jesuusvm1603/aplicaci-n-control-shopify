# CLAUDE.md — Aplicación Shopify Dropshipping

SPA de un solo archivo (`index.html`, ~3700 líneas). Sin servidor, sin build. Todo el estado vive en `localStorage`.

---

## Librerías

- **Chart.js v4.4.0** (CDN) — único gráfico: `monthlyChart` en Resumen Mensual y `profitChart`/`spendChart` en Dashboard

---

## Módulos de navegación

| ID div | Título | Función que renderiza |
|---|---|---|
| `mod-dashboard` | Dashboard General | `renderDashboard()` ~L1667 |
| `mod-catalog` | Catálogo de Productos | `renderCatalog()` ~L1945 |
| `mod-testing` | Testing de Creativos | `renderTesting()` ~L2525 |
| `mod-scaling` | Escalado de Campañas | `renderScaling()` ~L2885 |
| `mod-finance` | Finanzas | Varias (ver abajo) |

El módulo activo se gestiona con `showModule(name)` que añade `.active` al div y llama al render correspondiente.

---

## Estructura del state

```javascript
state = {
  products: [{ id, title, price, cog, status, category, image, url, sku,
               manualSales: [{date, units, notes}] }],

  creatives: [],          // DEPRECATED — datos migrados a testCampaigns al arrancar

  testCampaigns: [{       // Módulo Testing
    id, name, platform, status, productId, startDate, objective,
    creatives: [{ id, name, format, date, url, budget, status,
                  days: [{spend, revenue}] }]   // 7 días de seguimiento
  }],

  campaigns: [{           // Módulo Escalado
    id, name, platform, status, productId, startDate,
    budget, totalRevenue, roas, objective
  }],

  returns: [{ id, campaignId, date, amount, description }],

  finance: {
    daily: {
      'YYYY-MM': {
        'YYYY-MM-DD': { date, rev, cpc, visits, orders, adSpend, cogs,
                        autoRev, autoCogs, autoOrders, autoAdSpend }
        // campos "auto*" son calculados por syncFinanceFromSales(), NO editar a mano
      }
    },
    fixedCosts: { team, apps, chatgpt, creatives, banking, custom:[{name,amount}] },
    liquidity:  { wise, bank, shopify, cash, paypal, klarna, stripe,
                  pendingInvoices, debts },
    invoices:   [{ id, number, date, dueDate, concept, amount, status, type }]
  },

  csvData: [],            // Datos de importación CSV (Escalado)
  lastSync: null,
  lastImportUrl: null
}
```

---

## Flujo de datos crítico

```
Edición de datos (cualquier módulo)
  └─► onStateChanged(source)
        ├─ syncFinanceFromSales()   ← recalcula todos los campos "auto" en finance.daily
        ├─ saveState()              ← persiste a localStorage
        └─ refreshActiveModule()   ← re-renderiza UI
```

### syncFinanceFromSales() — función más importante del sistema
1. Resetea `autoRev/autoCogs/autoOrders/autoAdSpend` en todos los días existentes
2. Suma ventas manuales de productos (`products[].manualSales`)
3. Suma datos de creativos (`testCampaigns[].creatives[].days`) → cada día[i] = launchDate + i
4. Suma datos de campañas escaladas (`campaigns[]`) → totalRevenue atribuido a startDate

`addToDay(dateStr, rev, cog, orders, spend)` crea el día si no existe.

---

## localStorage

| Key | Contenido |
|---|---|
| `cc_auth_v1` | `{ users:[{id,name,email,pwHash,shopifyUrl}], session:{userId} }` |
| `cc_data_<userId>` | State completo del usuario (el objeto `state`) |
| `cc_control_v1` | Legacy — ignorar |

El hash de contraseña es determinístico simple (no criptográfico — solo demo local).

---

## Funciones por módulo

### Utilidades globales (~L1622)
```
fmt(n, dec=2)          → número con decimales
fmtEur(n)              → "€X.XX"
fmtPct(n)              → "X.X%"
uid()                  → ID único (timestamp36 + random)
today()                → "YYYY-MM-DD"
currentMonth()         → "YYYY-MM"
roasClass(r)           → clase CSS: roas-great/good/ok/low/bad
roasBadge(r)           → HTML badge coloreado
statusBadge(s)         → HTML badge de estado
getProduct(id)         → busca en state.products
getDailyFixedCost()    → sum(fixedCosts) / 30
calcDayProfit(d)       → rev - cogs - ads - com3% - CF/día
```

### Auth (~L1227)
```
bootApp()              → punto de entrada; carga state, migra datos, renderiza
currentUser()          → usuario logueado o null
doLogin/doRegister/doLogout()
saveProfile/savePassword/saveShopifyUrl()
```

### Dashboard (~L1667)
```
renderDashboard()
getProductTotalSales/Revenue/Profit(productId)
getProductCreativeSales(productId)   ← lee getAllTestCreatives()
getProductCampaignSales(productId)   ← lee state.campaigns
```

### Catálogo (~L1945)
```
renderCatalog()
syncProducts()                       ← llama Shopify URL
handleCatalogCSV(file)
openImportPreview(title, items)
confirmImport()
editProduct(id) / saveProduct()
addManualSale() / deleteManualSale()
calcProductMargin()
```

### Testing de Creativos (~L2519)
```
getAllTestCreatives()                 ← aplana testCampaigns[].creatives[], inyecta productId/platform
renderTesting()
toggleTestCampaign(campId)           ← acordeón expand/collapse

openTestCampaignModal(campId?)
saveTestCampaign()
deleteTestCampaign(campId)

openCreativeModal(campId, creativeId?)
saveCreative()
deleteTestCreative(campId, creativeId)

openCreativeDays(campId, creativeId) ← modal de detalle 7 días (solo lectura)
updateDayRoas(i)                     ← actualiza ROAS/Margen% en tiempo real en el modal

calcCreativeROAS/Spend/Revenue/Units/Margin(c)
calcUnitsFromRev(rev, price)
```

### Escalado de Campañas (~L2885)
```
renderScaling()
openCampaignModal(id?) / saveCampaign() / deleteCampaign(id)
calcBEROAS() / calcBEROASValue(productId)  ← Break-Even ROAS
applyCSVToFinance()                        ← importa CSV Meta Ads → finance.daily
setupDragDrop()
```

### Finanzas
```
financeTab(tab)                      → cambia pestaña (daily/fixed/liquidity/invoices/monthly)

renderFinanceDaily()    ~L3135       → tabla días: rev+auto / CPC / visitas / órdenes / AOV /
                                       CR% / COGS / Ads / Com3% / CF / Profit / ROAS / Profit%
updateDay(ym, date, field, value)
deleteDay(ym, date)
addFinanceDay()
toggleTableFullscreen()              → modo pantalla completa tabla diaria

renderFixedCosts()      ~L3236
renderLiquidity()       ~L3310
renderMonthlySummary()  ~L3422       → gráfico Chart.js + tabla resumen por mes
renderInvoices()        ~L3564
```

---

## Modales

| ID | Descripción |
|---|---|
| `settings-modal` | Perfil, contraseña, URL Shopify |
| `import-preview-modal` | Preview productos antes de importar |
| `product-modal` | Editor de producto (con ventas manuales) |
| `creative-modal` | Editor de creativo (nombre, formato, fecha, URL, presupuesto, estado, tabla 7 días) |
| `test-campaign-modal` | Editor de campaña de test (nombre, producto, plataforma, fecha, estado, objetivo) |
| `creative-days-modal` | Vista detalle 7 días de un creativo (solo lectura + botón editar) |
| `campaign-modal` | Editor de campaña escalada |

Todos se abren con `.classList.add('open')` y se cierran con `closeModal(id)`.

---

## CSS — clases y variables clave

**Variables de color:**
```css
--bg / --bg2 / --bg3      fondos oscuros (oscuro → claro)
--text / --text2           texto principal / secundario
--accent / --accent2       índigo (primario)
--green / --yellow / --orange / --red
--border
```

**Componentes:**
```
.card / .card-label / .card-value / .card-sub / .card-sub.neutral
.table-wrap / .chart-wrap / .form-group / .form-grid
.btn / .btn-primary / .btn-secondary / .btn-danger / .btn-sm
.badge-green/yellow/red/blue/orange/gray
.roas-great/good/ok/low/bad
.text-green / .text-red / .text-yellow
.semaphore + .sem-green / .sem-yellow / .sem-red
.auto-cell / .auto-tag     (celdas con valor auto + manual en finanzas)
.finance-table / .finance-table-wrap / .fullscreen
.module / .module.active
.modal-overlay / .modal-overlay.open
.empty-state / .empty-icon
.flex / .flex-between
.page-header / .search-row / .table-header
```

---

## Notas para futuras sesiones

- **Producto vinculado a creativos**: no se guarda en el creativo, sino en la campaña. `getAllTestCreatives()` lo inyecta al aplanar.
- **Migración automática**: si hay datos en `state.creatives` (versión anterior), `bootApp()` los migra a `testCampaigns` la primera vez.
- **Campos auto en finanzas**: nunca editar `autoRev/autoCogs/autoAdSpend` directamente — se sobreescriben en cada `syncFinanceFromSales()`.
- **Resumen mensual**: usa `requestAnimationFrame` al renderizar para que el canvas de Chart.js tenga dimensiones correctas.
- **Com.6% eliminada**: la columna fue quitada de la tabla diaria (era redundante). Solo existe Com.3%.
- **Tabla de creativos compacta**: la tabla principal muestra resumen (ROAS, gasto, ingresos, margen%, dots de 7 días). El detalle completo de días está en `openCreativeDays()`.
