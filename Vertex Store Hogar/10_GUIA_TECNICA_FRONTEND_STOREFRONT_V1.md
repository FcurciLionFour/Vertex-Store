# 10 · Guía Técnica Frontend — Storefront V1 (Tienda Pública V1)

**Proyecto:** Vertex Commerce
**Componente:** Storefront V1 / Tienda Pública V1 (primera tienda pública real)
**Fecha:** 2 jun 2026
**Modo de este documento:** Solo análisis y guía. NO se implementa código. NO se crea proyecto. NO se modifica backend ni backoffice.
**Audiencia:** Codex (agente de implementación), team técnico, futuros mantenedores.
**Fuentes consultadas:**
- `vertex_commerce_design_package/06_prompt_implementacion/PROMPT_MAESTRO_TIENDA_PUBLICA_VERTEX_COMMERCE_V2.md`
- `vertex_commerce_design_package/01_storefront_mockups/`, `02_customer_account_mockups/`, `03_design_system_boards/`, `04_design_system_pdfs/`
- `Documentacion/09_STOREFRONT_FRONTEND_READINESS_AUDIT.md`
- Auditoría backoffice ↔ backend V2 (este chat)
- Controllers reales de `backend/src/modules/**/*.controller.ts`

---

## 1. Resumen ejecutivo

### 1.1 Qué se va a construir

Storefront V1 es la **primera tienda pública real** de Vertex Commerce. Es una aplicación web Angular 20 con SSR que renderiza la experiencia de compra de una tienda concreta del sistema multi-tenant: home editorial, catálogo, ficha de producto, búsqueda, carrito, checkout, confirmación, seguimiento, páginas CMS y área autoservicio del cliente (login/registro, mi cuenta, pedidos, perfil, wishlist local).

> El nombre correcto es **Storefront V1** o **Tienda Pública V1**. NO se llama “e-commerce V2” porque no existe una V1 pública previa: lo único que ya está construido es el **Backoffice Admin** y el **Backend API**.

### 1.2 Qué NO se va a construir todavía

- Wishlist persistente server-side (V1 vive en `localStorage`).
- Libreta de direcciones del cliente (no hay backend V2).
- Gateways de pago reales (Mercado Pago, etc.).
- Integración real con courier (Andreani API, OCA API). V1 cotiza con reglas declarativas locales del backend.
- Recuperación de contraseña (`forgot/reset`) — postergada hasta que exista endpoint backend.
- Refresh token customer — la cookie de access dura 7 días, no se necesita aún.
- Cupones, reviews, comparador, server-side cart, multi-idioma, recomendaciones AI.

### 1.3 Diferencia entre Backoffice Admin y Storefront V1

| | Backoffice Admin | Storefront V1 |
|---|---|---|
| Propósito | Operación interna | Compra del cliente final |
| Audiencia | Staff (admin/manager/staff/viewer) | Visitante anónimo + customer registrado |
| Auth | JWT admin (`JWT_ACCESS_SECRET`), refresh propio | JWT customer (`CUSTOMER_JWT_ACCESS_SECRET`), cookie httpOnly + CSRF |
| Endpoints | `/admin/*` + `/auth/*` | `/storefront/*` + `/customer/*` |
| Multi-store | Selector explícito (`x-store-key` derivado de sesión) | Resolución implícita por `Host` (subdominio/dominio de la tienda) |
| Estado app | Pesado: dashboards, ABMs | Liviano: catálogo, carrito local, customer |
| SSR | No | Sí (SEO crítico) |
| Repositorio | `backoffice-admin/` | `storefront/` (a crear, fuera de scope ahora) |

### 1.4 Reglas de consumo de API (innegociables)

1. El storefront consume **únicamente** `/storefront/*` y `/customer/*`.
2. El storefront **nunca** consume `/admin/*`. Los admin endpoints son del backoffice.
3. Los endpoints `/public/*` (legacy) **no se usan**: existen solo como red de seguridad de regresión. Codex debe ignorarlos.
4. Si se descubre que falta un endpoint público, **no se sale del paso usando admin**. Se documenta el gap, se propone backend mínimo y se posterga la feature.

### 1.5 Estado del backend

✅ Backend V2 cerrado al 100% (Waves A → E.3): 168 e2e tests verdes, build limpio, 0 lint warnings. La auditoría `09_STOREFRONT_FRONTEND_READINESS_AUDIT.md` y la auditoría backoffice ↔ backend confirman que **todo lo que necesita Storefront V1 ya existe**.

---

## 2. Objetivo del Storefront V1

### 2.1 Alcance funcional incluido

| Área | Funcionalidad |
|---|---|
| Home | Hero editorial, secciones CMS dinámicas, productos destacados, categorías, banners, newsletter, footer rico |
| Categorías / PLP | Lista paginada de productos por categoría, filtros simples (precio, ordenamiento), breadcrumb |
| Producto / PDP | Galería, info, precio, stockStatus, quantity, agregar al carrito, productos relacionados |
| Búsqueda | Resultados full-text, autocomplete header (suggestions ≥ 2 chars) |
| Carrito | Lista de items, modificar cantidades, validar contra backend, preview de envío |
| Checkout | Stepper datos → envío → pago → revisión, cotización shipping, métodos pago, submit |
| Confirmación | Detalle de pedido + número, refrescable vía lookup |
| Seguimiento | Por número de pedido + email (anónimo) o desde la cuenta del cliente |
| CMS pages | Páginas dinámicas: Nosotros, Contacto (form local), FAQ, Envíos, Cambios, Medios de pago, Términos, Privacidad |
| Customer auth | Login, Register, Logout |
| Mi Cuenta | Dashboard con últimos pedidos y atajos |
| Mis Pedidos | Lista paginada |
| Pedido Detalle | Resumen + tracking |
| Perfil | Editar nombre/teléfono, cambiar contraseña |
| Wishlist | Lista local en `localStorage` (V1) |

### 2.2 Alcance funcional excluido (Wave F backend o post-V1)

- Wishlist persistente server-side.
- Libreta de direcciones del cliente.
- Recuperación de contraseña (`forgot/reset-password`).
- Refresh token customer.
- Cupones y promociones aplicables en checkout.
- Reviews y ratings.
- Comparador de productos.
- Carrito persistente server-side.
- Pago real (gateway integrado).
- Courier real (tracking automatizado externo).
- Recomendaciones personalizadas, multi-idioma, multi-currency dinámico.

### 2.3 Criterios de éxito

- Todas las páginas P0 y P1 (ver §8) funcionan en desktop y mobile.
- 168/168 tests backend siguen en verde (no se toca backend).
- Lighthouse Performance ≥ 80 / Accessibility ≥ 90 en Home y PDP móviles.
- Cero llamadas a `/admin/*` desde el storefront (verificado por grep).
- Storefront SSR-able: HTML completo en respuesta inicial para Home, PLP, PDP y CMS.

---

## 3. Stack técnico recomendado

| Tecnología | Versión sugerida | Rol | Cuándo se usa |
|---|---|---|---|
| Angular | 20.x | Framework UI | Toda la app |
| @angular/ssr | 20.x | Server-Side Rendering | Home/PLP/PDP/CMS (críticos SEO) |
| Standalone Components | siempre | Modularidad y lazy | Toda la app, sin NgModules |
| Signals + `computed` + `effect` | siempre | Estado reactivo | Stores (cart, auth, store-context, navigation) |
| Control flow nuevo (`@if`, `@for`, `@let`) | siempre | Templates | Reemplaza `*ngIf`, `*ngFor` |
| TypeScript | 5.5+ con `strict: true` | Tipado | Todo el código |
| HttpClient (`provideHttpClient`) | 20.x | Cliente HTTP | Servicios + interceptores |
| `withFetch()` | recomendado | HTTP backend en SSR | Necesario para SSR moderno |
| `provideClientHydration()` | recomendado | Hydration | Reduce flicker |
| `TransferState` | opcional | Evitar doble fetch SSR/Browser | Home/PLP/PDP |
| CSS tokens propios | obligatorio | Design System | `styles/tokens.css` |
| ESLint + Prettier | última | Calidad | Mismas reglas que `backoffice-admin` |
| Playwright | última | E2E (Wave 6) | Smoke flow checkout |
| Jest o Karma+Jasmine | a elegir | Unit | Servicios y stores |
| Husky + lint-staged | opcional | Calidad pre-commit | Si se usa git hooks |

### 3.1 Justificación de cada decisión

- **Angular 20 con SSR:** SEO es crítico para una tienda. Home/PLP/PDP deben servir HTML completo en la primera respuesta. SSR + hydration evita flicker y mejora Core Web Vitals.
- **Standalone components:** alinea con la práctica vigente del backoffice-admin, simplifica lazy-loading y evita módulos boilerplate.
- **Signals:** estado de carrito, auth y store-context se actualiza en muchos lugares; signals dan reactividad granular sin RxJS subjects manuales.
- **TypeScript strict:** bloquea bugs comunes (undefined, any).
- **CSS tokens propios:** evita atar el storefront a una librería UI genérica que choque con el Design System editorial premium.
- **localStorage para cart/wishlist:** explícitamente decidido en el alcance V1 — sin server-side cart.
- **Interceptors:** centralizan store-context, CSRF customer y manejo de errores.
- **Guards customer:** protegen `/cuenta/*`. Nunca usar guards admin.
- **Resolvers:** carga SEO-friendly previa al render (Home/PLP/PDP/CMS).

---

## 4. Arquitectura de carpetas recomendada

```
src/
  app/
    app.config.ts                       ← bootstrapApplication providers
    app.config.server.ts                ← providers SSR
    app.routes.ts                       ← routing top-level
    app.component.ts                    ← root con <router-outlet>

    core/                               ← infraestructura (singleton, providedIn:'root')
      api/
        storefront-api.service.ts       ← wrapper /storefront/*
        customer-api.service.ts         ← wrapper /customer/* + withCredentials
        api-base.token.ts               ← InjectionToken<string>
        api-types.ts                    ← interfaces espejo de DTOs públicos
      interceptors/
        store-context.interceptor.ts
        customer-csrf.interceptor.ts
        error.interceptor.ts
      guards/
        customer-auth.guard.ts
        customer-guest.guard.ts
      stores/                           ← signal stores globales
        store-context.store.ts
        navigation.store.ts
        cart.store.ts
        auth-customer.store.ts
        wishlist.store.ts
      tokens/
        cookie.token.ts                 ← lectura cookie segura SSR/Browser
      types/                            ← types compartidos (PublicStoreDto, etc.)
      utils/
        money.util.ts
        url.util.ts

    shared/                             ← componentes/pipes reutilizables sin lógica de negocio
      ui/                               ← Vtx* base (Button, Input, etc.)
      pipes/
        currency-vtx.pipe.ts
      directives/
        focus-trap.directive.ts

    layout/                             ← shells de la app
      storefront-shell/                 ← layout público (header + footer + outlet)
      storefront-header/
        storefront-header.component.ts
        search-overlay/
      storefront-footer/
      customer-shell/                   ← layout área autenticada (sidebar + outlet)
      account-sidebar/

    storefront/                         ← features públicas (lazy)
      home/
        home.routes.ts
        home-page.component.ts
        home.resolver.ts
      categories/
        categories.routes.ts
        category-page.component.ts
        category.resolver.ts
        components/
          filter-sidebar/
          filter-drawer/
          sort-select/
      products/
        products.routes.ts
        product-page.component.ts
        product.resolver.ts
        components/
          product-gallery/
          product-info/
          related-products/
      search/
        search.routes.ts
        search-page.component.ts
      cms/
        cms.routes.ts
        cms-page.component.ts
        cms-section-renderer/
      tracking/
        tracking.routes.ts
        order-lookup-page.component.ts

    cart/
      cart.routes.ts
      cart-page.component.ts
      components/
        cart-item/
        cart-summary/
        cart-drawer/

    checkout/
      checkout.routes.ts
      checkout-page.component.ts
      checkout.store.ts
      services/
        shipping.service.ts
        payments.service.ts
      components/
        checkout-stepper/
        shipping-method-card/
        payment-method-card/
        order-summary/

    customer/
      auth/
        auth.routes.ts
        login-page.component.ts
        register-page.component.ts
      account/
        account.routes.ts
        account-dashboard-page.component.ts
      orders/
        orders.routes.ts
        orders-list-page.component.ts
        order-detail-page.component.ts
      profile/
        profile.routes.ts
        profile-page.component.ts
        change-password-page.component.ts
      wishlist/
        wishlist.routes.ts
        wishlist-page.component.ts

  styles/
    tokens.css                          ← variables CSS del DS (colores, spacing, type)
    base.css                            ← reset + tipografía
    utilities.css                       ← clases utilitarias mínimas

  environments/
    environment.ts                      ← dev (apiBase=http://localhost:3000, storeKey?)
    environment.prod.ts                 ← prod (apiBase real, sin storeKey)

  main.ts                               ← bootstrap browser
  main.server.ts                        ← bootstrap SSR
  server.ts                             ← express handler SSR
```

### 4.1 Para qué sirve cada carpeta y qué NO debe contener

| Carpeta | Contiene | NO contiene |
|---|---|---|
| `core/api/` | Wrappers HttpClient genéricos, tipos espejo de DTOs backend | Lógica de UI, signals globales |
| `core/interceptors/` | Interceptors funcionales (Angular 20 style) | Servicios de negocio |
| `core/guards/` | Guards funcionales | Lógica de auth (eso va al store) |
| `core/stores/` | Signal stores globales (singleton) | Componentes |
| `core/tokens/` | InjectionTokens reutilizables | Implementaciones |
| `shared/ui/` | Componentes presentacionales puros (sin servicios de negocio) | HttpClient, stores |
| `shared/pipes/` | Pipes puros | Side effects |
| `layout/` | Shells y header/footer/sidebar | Lógica de páginas |
| `storefront/<area>/` | Features públicas con sus resolvers, componentes y subcomponentes | Servicios cross-feature (van a core) |
| `cart/`, `checkout/`, `customer/` | Features con state propio | Mutaciones admin (no existen) |
| `styles/` | Tokens y base CSS | Estilos por componente (van inline o `.css` propio) |
| `environments/` | URLs y flags por entorno | Secretos (no hay en frontend público) |

---

## 5. Configuración inicial (Wave 0)

Wave 0 NO entrega features de negocio. Solo deja la app lista.

### 5.1 Checklist Wave 0

- [ ] App Angular 20 inicializada con `@angular/ssr`.
- [ ] `tsconfig.json` con `strict: true`, `noUncheckedIndexedAccess: true`.
- [ ] ESLint + Prettier configurados (alineados con `backoffice-admin`).
- [ ] `environment.ts` y `environment.prod.ts` con:
  ```ts
  export const environment = {
    production: false,
    apiBase: 'http://localhost:3000',
    storeKey: 'demo-store', // opcional dev — header x-store-key
    cookieDomain: undefined, // se decide en producción
  };
  ```
- [ ] `styles/tokens.css` con paleta y spacing del DS.
- [ ] `styles/base.css` con reset + fuentes.
- [ ] `app.config.ts` con: `provideRouter`, `provideHttpClient(withFetch(), withInterceptors([...]))`, `provideClientHydration()`, `provideAnimations()` (opcional).
- [ ] `app.config.server.ts` con `provideServerRendering()`.
- [ ] Routing top-level con shells lazy.
- [ ] `StorefrontShellComponent` con `<storefront-header>` + `<router-outlet>` + `<storefront-footer>` (placeholders).
- [ ] `StorefrontApiService` y `CustomerApiService` esqueleto.
- [ ] Interceptors instalados (aunque vacíos al principio).
- [ ] `StoreContextStore` hidratando desde `/storefront/store` y `/storefront/config`.
- [ ] Smoke local: `npm run dev:ssr` arranca, header pinta nombre de la tienda, console limpia.
- [ ] CORS verificado (backend acepta `origin` del storefront).

### 5.2 Qué NO hacer en Wave 0

- No implementar Home dinámica.
- No implementar PLP, PDP, Cart, Checkout, Customer.
- No diseñar componentes complejos (basta header/footer placeholder).
- No agregar SSR caching todavía.
- No conectar estados de UI a features de negocio.

---

## 6. Endpoints reales del backend a usar

> Verificados en controllers reales del repo backend. Esta es la fuente de verdad. Los nombres en el prompt maestro V2 algunos son aspiracionales y NO coinciden con la implementación final; ganan estos.

### 6.1 Storefront público (sin auth, store context implícito)

| Endpoint | Método | Auth | Store context | Notas |
|---|---|---|---|---|
| `/storefront/store` | GET | — | sí (Host / `x-store-key` / `?storeKey`) | Datos públicos: nombre, logo, faviconUrl, dominio primario |
| `/storefront/store/by-domain` | GET | — | `?domain=` directo | Resolver SSR si no hay Host confiable |
| `/storefront/config` | GET | — | sí | `currency`, `locale`, `timezone`, `seo`, `theme` |
| `/storefront/navigation/:location` | GET | — | sí | `:location ∈ MenuLocation` (HEADER, FOOTER, etc.) |
| `/storefront/home` | GET | — | sí | Alias de `/pages/home` con `sections[]` |
| `/storefront/pages/:slug` | GET | — | sí | Solo `PageStatus.PUBLISHED` |
| `/storefront/categories` | GET | — | sí | Árbol público de categorías |
| `/storefront/categories/:slug` | GET | — | sí | Categoría + productos públicos |
| `/storefront/products` | GET | — | sí | Query: `search`, `category`, `minPrice`, `maxPrice`, `sort`, `page`, `limit` |
| `/storefront/featured` | GET | — | sí | Mismos filtros que products |
| `/storefront/products/:slug` | GET | — | sí | PDP — `PublicProductDto` |
| `/storefront/products/:slug/related` | GET | — | sí | `?limit` (por defecto 8) |
| `/storefront/search` | GET | — | sí | `?q=` + filtros — full-text contains |
| `/storefront/search/suggestions` | GET | — | sí | `?q=` (mín 2 chars) → top 10 mixto PRODUCT/CATEGORY/PAGE |
| `/storefront/cart/validate` | POST | — | sí | Body: `{ items: [{ productStoreId, quantity }] }` |
| `/storefront/payment-methods` | GET | — | sí | Solo `enabled: true`, sin secrets |
| `/storefront/shipping-methods` | GET | — | sí | Solo `enabled: true`, sin zones/keys |
| `/storefront/shipping/quote` | POST | — | sí | Body: `{ postalCode, items: [{ productStoreId, qty }] }` |
| `/storefront/checkout` | POST | — | sí | Crea pedido público, retorna `{ order }` |
| `/storefront/orders/lookup` | GET | — (throttle 5/min) | sí | `?orderNumber=&customerEmail=` |
| `/storefront/orders/:orderNumber/tracking` | GET | — (throttle 5/min) | sí | Tracking anónimo |

### 6.2 Customer (cookie httpOnly + CSRF double-submit)

| Endpoint | Método | Auth | CSRF | Notas |
|---|---|---|---|---|
| `/customer/auth/register` | POST | — | — | Setea cookies `vtx_customer_at` + `vtx_customer_csrf` |
| `/customer/auth/login` | POST | — | — | Setea cookies |
| `/customer/auth/logout` | POST | JWT | sí | Limpia cookies |
| `/customer/me` | GET | JWT | — | Hidrata `AuthCustomerStore` |
| `/customer/profile` | GET | JWT | — | Alias de `/me` |
| `/customer/profile` | PATCH | JWT | sí | Edita `firstName/lastName/phone` (no email/password) |
| `/customer/profile/change-password` | POST | JWT | sí | Cambio de contraseña |
| `/customer/orders` | GET | JWT | — | `?page=&limit=` paginado |
| `/customer/orders/:orderNumber` | GET | JWT | — | Detalle |
| `/customer/orders/:orderNumber/tracking` | GET | JWT | — | Tracking del cliente |

### 6.3 Endpoints que NO se deben usar

| Endpoint | Por qué no |
|---|---|
| Cualquier `/admin/*` | Es del backoffice. Romper esta regla rompe la separación de seguridad. |
| `/public/checkout`, `/public/products`, `/public/categories`, `/public/menus`, `/public/pages` | Legacy preservado solo por regresión backend. El storefront nuevo usa `/storefront/*`. |
| `/auth/login`, `/auth/refresh`, `/auth/me`, `/auth/logout` | Es la auth del **admin**. La del cliente es `/customer/auth/*`. |

### 6.4 Endpoints que NO existen aún (gaps)

| Esperado (prompt maestro V2) | Estado real | Decisión Storefront V1 |
|---|---|---|
| `/storefront/footer` | NO existe (se sirve vía `/storefront/navigation/FOOTER`) | usar `navigation/FOOTER` |
| `/storefront/banners`, `/storefront/sections` | NO existen | se sirven dentro de `/storefront/home` y `/storefront/pages/:slug` (`sections[]`) |
| `/storefront/categories/:slug/products` | NO existe (existe `/storefront/categories/:slug` o `/storefront/products?category=:slug`) | usar `/storefront/products?category=:slug` |
| `/storefront/cart` (server-side) | NO existe | V1 cart es local (localStorage) |
| `/storefront/cart/items`, `apply-coupon` | NO existen | V1 sin cupones, sin server cart |
| `/storefront/products/featured` | NO existe (real: `/storefront/featured`) | usar `/storefront/featured` |
| `/storefront/products/:id/related` | NO existe (real: `/storefront/products/:slug/related`) | usar slug |
| `/customer/auth/refresh` | NO existe | confiar en cookie 7d |
| `/customer/auth/forgot-password`, `/reset-password` | NO existen | postergar Wave F backend |
| `/customer/wishlist*` | NO existen | wishlist localStorage |

---

## 7. Tabla endpoint → uso frontend

| Endpoint | Cuándo se usa | Página/Componente | Request | Response esperado | Store/Service frontend |
|---|---|---|---|---|---|
| `GET /storefront/store` | Bootstrap SSR + shell global | `StorefrontShell`, `StorefrontHeader` | — | `PublicStoreDto` (`name`, `logoUrl`, `faviconUrl`, `domain`) | `StoreContextStore` |
| `GET /storefront/store/by-domain` | SSR cuando se conoce el `Host` exacto pero no hay middleware de resolución | resolver SSR | `?domain=...` | `PublicStoreDto` | `StorefrontStoreService` |
| `GET /storefront/config` | Bootstrap | shell | — | `PublicConfigDto` (`currency`, `locale`, `timezone`, `seo`, `theme`) | `StoreContextStore` |
| `GET /storefront/navigation/HEADER` | Header carga | `StorefrontHeader` | — | `{ items: [...] }` (con `category?/page?/productStore?`) | `NavigationStore` |
| `GET /storefront/navigation/FOOTER` | Footer carga | `StorefrontFooter` | — | idem | `NavigationStore` |
| `GET /storefront/home` | Home page | `HomePageComponent` (resolver) | — | `{ page, sections[] }` | `CmsService` |
| `GET /storefront/pages/:slug` | Page genérica `/p/:slug` y páginas legales | `CmsPageComponent` (resolver) | `:slug` | `{ page, sections[] }` | `CmsService` |
| `GET /storefront/categories` | Drawer/footer/header tree, sidebar PLP | `CategoryDrawer`, `FilterSidebar` | — | tree | `CategoriesService` |
| `GET /storefront/categories/:slug` | PLP header (datos categoría) | `CategoryPageComponent` (resolver) | `:slug` | categoría + productos | `CategoriesService` |
| `GET /storefront/products` | PLP filtrada + Home grid | `CategoryPageComponent`, `HomePageComponent` | query DTO | `{ items: PublicProductCardDto[], total, page, limit }` | `ProductsService` |
| `GET /storefront/featured` | Home destacados | `HomePageComponent` | — | misma shape | `ProductsService` |
| `GET /storefront/products/:slug` | PDP | `ProductPageComponent` (resolver) | `:slug` | `PublicProductDto` | `ProductsService` |
| `GET /storefront/products/:slug/related` | PDP related | `RelatedProductsComponent` | `?limit` | `{ items, total, page, limit }` | `ProductsService` |
| `GET /storefront/search` | Página `/buscar` | `SearchPageComponent` | `?q=&...` | misma shape products | `SearchService` |
| `GET /storefront/search/suggestions` | Header overlay | `SearchOverlay` | `?q=` (≥2) | `{ suggestions: [...] }` | `SearchService` (debounced) |
| `POST /storefront/cart/validate` | Cart load + agregar item + entrar a checkout | `CartPageComponent`, `CheckoutPageComponent` | `{ items: [...] }` | `CartValidateResponseDto` | `CartStore` |
| `GET /storefront/payment-methods` | Checkout step pago | `CheckoutPageComponent` | — | `{ items: [...] }` | `PaymentsService` |
| `GET /storefront/shipping-methods` | Cart preview + Checkout step envío | `CartPageComponent`, `CheckoutPageComponent` | — | `{ items: [...] }` | `ShippingService` |
| `POST /storefront/shipping/quote` | Cart preview + Checkout cotización | idem | `{ postalCode, items }` | `{ items: [{ code, label, cost, estimatedDays? }] }` | `ShippingService` |
| `POST /storefront/checkout` | Submit final | `CheckoutPageComponent` | `PublicCheckoutRequestDto` | `{ order }` | `CheckoutService` |
| `GET /storefront/orders/lookup` | Página `/seguimiento` (form) | `OrderLookupPageComponent` | `?orderNumber=&customerEmail=` | `PublicOrderDto` | `OrdersService` |
| `GET /storefront/orders/:orderNumber/tracking` | Tracking anónimo | `OrderLookupPageComponent` | `:orderNumber` | `PublicOrderTrackingDto` | `OrdersService` |
| `POST /customer/auth/register` | Página Register | `RegisterPageComponent` | `CustomerRegisterDto` | `{ customer }` | `CustomerAuthService` + `AuthCustomerStore` |
| `POST /customer/auth/login` | Página Login | `LoginPageComponent` | `CustomerLoginDto` | `{ customer }` | idem |
| `POST /customer/auth/logout` | Header / cualquier área autenticada | `StorefrontHeader` | — | `{ ok: true }` | idem |
| `GET /customer/me` | Bootstrap área autenticada | `customerAuthGuard` | — | `CustomerMeDto` | `AuthCustomerStore` |
| `GET /customer/profile` | Página Perfil | `ProfilePageComponent` | — | `CustomerMeDto` | `CustomerProfileService` |
| `PATCH /customer/profile` | Editar perfil | idem | `CustomerUpdateProfileDto` | `CustomerMeDto` | idem |
| `POST /customer/profile/change-password` | Cambio contraseña | `ChangePasswordPageComponent` | `CustomerChangePasswordDto` | `{ ok: true }` | idem |
| `GET /customer/orders` | Mis Pedidos | `OrdersListPageComponent` (resolver) | `?page=&limit=` | `{ items, total, page, limit }` | `CustomerOrdersService` |
| `GET /customer/orders/:orderNumber` | Pedido Detalle | `OrderDetailPageComponent` (resolver) | `:orderNumber` | `PublicOrderDto` | idem |
| `GET /customer/orders/:orderNumber/tracking` | Tracking del cliente | `OrderDetailPageComponent` | `:orderNumber` | `PublicOrderTrackingDto` | idem |

---

## 8. Tabla página → endpoints

| # | Página | Ruta frontend | Endpoints principales | Endpoints secundarios | Prioridad | Observaciones |
|---|---|---|---|---|---|---|
| 1 | Home | `/` | `GET /storefront/home` | `/storefront/config`, `/storefront/store`, `/storefront/navigation/HEADER`, `/storefront/navigation/FOOTER`, `/storefront/featured`, `/storefront/categories` | P0 | SSR fuerte |
| 2 | PLP / Categoría | `/categorias/:slug` | `GET /storefront/categories/:slug`, `GET /storefront/products?category=:slug&...` | `/storefront/categories` (sidebar) | P0 | SSR fuerte, paginación |
| 3 | PDP | `/productos/:slug` | `GET /storefront/products/:slug` | `/storefront/products/:slug/related`, `POST /storefront/cart/validate` | P0 | SSR + JSON-LD |
| 4 | Search | `/buscar?q=` | `GET /storefront/search?q=` | `GET /storefront/search/suggestions` | P1 | overlay header |
| 5 | Cart | `/carrito` | `POST /storefront/cart/validate` | `GET /storefront/shipping-methods`, `POST /storefront/shipping/quote` | P0 | local + revalidate |
| 6 | Checkout | `/checkout` | `POST /storefront/checkout` | `POST /storefront/cart/validate`, `GET /payment-methods`, `GET /shipping-methods`, `POST /shipping/quote` | P0 | flujo crítico |
| 7 | Order Success | `/pedido/exito/:orderNumber` | response del checkout (state) | `GET /storefront/orders/lookup` (refresh) | P0 | guardar `orderNumber+email` para refresh |
| 8 | Order Lookup / Tracking | `/seguimiento` | `GET /storefront/orders/lookup` | `GET /storefront/orders/:orderNumber/tracking` | P1 | throttled 5/min |
| 9 | CMS Page | `/p/:slug` | `GET /storefront/pages/:slug` | — | P1 | SSR |
| 10 | Nosotros | `/nosotros` | `GET /storefront/pages/nosotros` | — | P2 | depende de contenido CMS |
| 11 | Contacto | `/contacto` | `GET /storefront/pages/contacto` | — | P2 | form local; sin endpoint backend (gap) |
| 12 | FAQ | `/faq` | `GET /storefront/pages/faq` | — | P2 | depende de contenido |
| 13 | Envíos | `/envios` | `GET /storefront/pages/envios` | `GET /storefront/shipping-methods` | P2 | mostrar métodos reales |
| 14 | Cambios y devoluciones | `/cambios-y-devoluciones` | `GET /storefront/pages/cambios-y-devoluciones` | — | P2 | depende de contenido |
| 15 | Medios de pago | `/medios-de-pago` | `GET /storefront/pages/medios-de-pago` | `GET /storefront/payment-methods` | P2 | listado real |
| 16 | Términos | `/terminos` | `GET /storefront/pages/terminos` | — | P2 | |
| 17 | Privacidad | `/privacidad` | `GET /storefront/pages/privacidad` | — | P2 | |
| 18 | Login | `/cuenta/ingresar` | `POST /customer/auth/login` | — | P1 | guard guest |
| 19 | Register | `/cuenta/crear` | `POST /customer/auth/register` | — | P1 | guard guest |
| 20 | Mi Cuenta | `/cuenta` | `GET /customer/me` | `GET /customer/orders?limit=5` | P1 | guard auth |
| 21 | Mis Pedidos | `/cuenta/pedidos` | `GET /customer/orders` | — | P1 | paginado |
| 22 | Pedido Detalle | `/cuenta/pedidos/:orderNumber` | `GET /customer/orders/:orderNumber` | `GET /customer/orders/:orderNumber/tracking` | P1 | timeline |
| 23 | Perfil | `/cuenta/perfil` | `GET /customer/profile`, `PATCH /customer/profile` | `POST /customer/profile/change-password` | P1 | requiere CSRF en mutaciones |
| 24 | Wishlist | `/cuenta/favoritos` | (localStorage) | — | P2 | sin backend V1 |

> Forgot/Reset password se omiten en el routing inicial: el botón “¿Olvidaste tu contraseña?” muestra mensaje “Próximamente disponible”.

---

## 9. Services frontend recomendados

> Convención: cada service vive en `core/api/` o en su feature. Stateless por defecto. Los stores con signals tienen state.

| Service | Responsabilidad | Endpoints que llama | NO debe hacer | Stateless? |
|---|---|---|---|---|
| `StorefrontApiService` | Wrapper HttpClient para `/storefront/*` con base URL y `withCredentials: false` por defecto | (genérico — `get<T>(path)`, `post<T>(path,body)`) | Tener lógica de negocio | sí |
| `CustomerApiService` | Wrapper HttpClient para `/customer/*` con `withCredentials: true` | (genérico) | Servir endpoints `/admin/*` o `/storefront/*` | sí |
| `StorefrontStoreService` | Resolver y obtener tienda + config | `GET /storefront/store`, `/store/by-domain`, `/config` | Mantener state (eso lo hace `StoreContextStore`) | sí |
| `NavigationService` | Obtener menús por location | `GET /storefront/navigation/:location` | Cachear en service (cachea el store) | sí |
| `CmsService` | Home + páginas CMS | `GET /storefront/home`, `/pages/:slug` | Renderizar HTML | sí |
| `ProductsService` | PLP, PDP, featured, related | `GET /storefront/products`, `/featured`, `/products/:slug`, `/products/:slug/related` | Validar carrito | sí |
| `CategoriesService` | Tree y categoría por slug | `GET /storefront/categories`, `/categories/:slug` | — | sí |
| `SearchService` | Search + suggestions con debounce | `GET /storefront/search`, `/search/suggestions` | Renderizar overlay | sí |
| `CartStore` (no service) | Estado del carrito + persistencia | `POST /storefront/cart/validate` | Mutar pedidos server-side | conectado a store |
| `CheckoutService` | Submit checkout | `POST /storefront/checkout` | Persistir datos sensibles | sí |
| `ShippingService` | Métodos + cotización | `GET /storefront/shipping-methods`, `POST /storefront/shipping/quote` | — | sí |
| `PaymentsService` | Métodos | `GET /storefront/payment-methods` | Procesar pago real | sí |
| `OrdersService` (anónimo) | Lookup + tracking sin login | `GET /storefront/orders/lookup`, `/storefront/orders/:n/tracking` | — | sí |
| `CustomerAuthService` | Register/login/logout/me | `POST /customer/auth/*`, `GET /customer/me` | Guardar tokens en localStorage | sí (state vive en store) |
| `CustomerProfileService` | Perfil + change-password | `GET/PATCH /customer/profile`, `POST /customer/profile/change-password` | Editar email | sí |
| `CustomerOrdersService` | Pedidos del cliente | `GET /customer/orders*` | — | sí |

---

## 10. Stores con Signals

### 10.1 Lista

| Store | Estado | Hidratación | Endpoints | Persistencia local | Errores que maneja |
|---|---|---|---|---|---|
| `StoreContextStore` | `store: PublicStoreDto`, `config: PublicConfigDto`, `loading`, `error` | SSR resolver al boot | `/storefront/store`, `/storefront/config` | — (cookie storeKey opcional) | network, 4xx, fallback a placeholder |
| `NavigationStore` | `header`, `footer` (caches por location) | después de StoreContext | `/storefront/navigation/:location` | — (memory + opcional TransferState) | silenciar y mostrar shell vacío |
| `CartStore` | `items: CartLine[]`, `validated: CartValidateResponseDto?`, `validating`, `error` | hidrata desde `localStorage['vtx_cart_v1']` | `POST /storefront/cart/validate` | sí: `vtx_cart_v1` (items + qty) | 400 → marca línea inválida; offline → mantener local |
| `AuthCustomerStore` | `customer: AuthenticatedCustomer? \| null \| 'unknown'`, `loading` | `GET /customer/me` al boot si existe cookie | `/customer/auth/*`, `/customer/me` | NO guardar token (cookie httpOnly) | 401 → set null, no redirect automático aquí (lo hace guard) |
| `CheckoutStore` | `step: 'data' \| 'shipping' \| 'payment' \| 'review'`, `data`, `selectedShipping`, `selectedPayment`, `submitting` | nuevo cada visita | (delega en services) | sólo en memoria | error de validación + 400 backend |
| `WishlistStore` | `slugs: string[]` | hidrata desde `localStorage['vtx_wishlist_v1']` | — | sí | — |
| `SearchStore` (opcional) | `q`, `suggestions`, `results`, `loading` | sólo cuando se abre overlay | `/storefront/search/suggestions` | — | silenciar y mostrar empty |

### 10.2 Reglas críticas

- **CartStore:**
  - V1 usa `localStorage` key `vtx_cart_v1`. Estructura: `{ version: 1, items: [{ productStoreId, slug, quantity, addedAt }] }`.
  - Toda interacción del usuario muta el array y dispara `validate()` con debounce (300 ms).
  - Antes de entrar a `/checkout` se vuelve a llamar `validate()` y si hay líneas `unavailable` se bloquea el avance.
  - **No persiste carrito server-side.** Si el usuario cierra la sesión o cambia de dispositivo pierde el carrito (aceptado V1).
- **WishlistStore:**
  - V1 usa `localStorage` key `vtx_wishlist_v1`. Estructura: `{ version: 1, slugs: string[] }`.
  - Operaciones: `add(slug)`, `remove(slug)`, `has(slug)`, `clear()`.
  - **No hay backend wishlist todavía.**
- **AuthCustomerStore:**
  - Sesión basada en cookie httpOnly `vtx_customer_at` (7 días) + cookie no-httpOnly `vtx_customer_csrf`.
  - **No guardar token JWT en `localStorage` ni en memoria expuesta.** El JWT lo gestiona la cookie, el frontend solo necesita saber si está logueado (vía `/customer/me` 200/401).
  - Estado inicial `'unknown'` (server-side aún no sabe). Tras `GET /customer/me`: `customer: AuthenticatedCustomer` o `null`.
  - El refresh no existe en V2; cuando expire la cookie, el siguiente `/customer/me` da 401 → store pasa a `null` → guard redirige.

---

## 11. Interceptors

Todos los interceptors son **funcionales** (Angular 20 `HttpInterceptorFn`).

### 11.1 `store-context.interceptor`

**Propósito:** asegurar que toda petición al backend resuelva la tienda correcta.

```ts
// Pseudocódigo descriptivo, NO implementación
export const storeContextInterceptor: HttpInterceptorFn = (req, next) => {
  if (!req.url.startsWith(environment.apiBase)) return next(req);

  // En SSR el Host ya viene; en browser no se necesita header.
  // En dev, si environment.storeKey existe, se inyecta como x-store-key.
  const headers: Record<string, string> = {};
  if (environment.storeKey && !req.headers.has('x-store-key')) {
    headers['x-store-key'] = environment.storeKey;
  }
  return next(req.clone({ setHeaders: headers }));
};
```

**Reglas:**
- En dev puede mandar `x-store-key` desde `environment.storeKey`.
- En prod NO se hardcodea storeId/storeKey en cliente: el backend resuelve por `Host`.
- Nunca enviar headers admin (`Authorization` para auth admin).

### 11.2 `customer-csrf.interceptor`

**Propósito:** double-submit CSRF en mutaciones a `/customer/*`.

```ts
export const customerCsrfInterceptor: HttpInterceptorFn = (req, next) => {
  const isCustomer = req.url.includes('/customer/');
  const isMutation = req.method !== 'GET';
  if (!isCustomer || !isMutation) return next(req.clone({ withCredentials: req.url.includes('/customer/') }));

  const csrf = readCookie('vtx_customer_csrf'); // util SSR-safe
  const headers: Record<string, string> = {};
  if (csrf) headers['x-csrf-token'] = csrf;

  return next(req.clone({ setHeaders: headers, withCredentials: true }));
};
```

**Reglas:**
- Aplica a `POST/PATCH/PUT/DELETE` de `/customer/*`.
- Requiere `withCredentials: true` (cookies).
- Si no hay cookie CSRF (sesión expirada), backend devuelve 403 → `error.interceptor` redirige a login.

### 11.3 `error.interceptor`

| Status | UX esperada |
|---|---|
| 400 | Mostrar mensajes de validación campo por campo (`response.error.details`). Toast genérico si no hay detalle. |
| 401 | Si es endpoint `/customer/*`: limpiar `AuthCustomerStore`, redirigir a `/cuenta/ingresar` con `returnUrl`. Si no, toast “Iniciá sesión para continuar”. |
| 403 | Si CSRF: invitar a recargar. Si permiso: toast “No autorizado”. |
| 404 | Si es resolver de página: redirigir a `/404`. Si es API ad-hoc: empty state contextual. |
| 429 | Toast “Demasiadas solicitudes, reintentá en un minuto.” (lookup/tracking). |
| 500/502/503 | Toast genérico “Algo salió mal, intentalo de nuevo.” + log a console (Sentry futuro). |
| Network error | Toast “Sin conexión.” + retry button. |

---

## 12. Guards

### 12.1 `customerAuthGuard`

```ts
// Pseudocódigo
export const customerAuthGuard: CanActivateFn = async () => {
  const auth = inject(AuthCustomerStore);
  await auth.ensureHydrated(); // GET /customer/me si está en 'unknown'
  if (auth.customer()) return true;
  const router = inject(Router);
  return router.createUrlTree(['/cuenta/ingresar'], { queryParams: { returnUrl: location.pathname } });
};
```

- Protege todo `/cuenta/*` excepto `/cuenta/ingresar` y `/cuenta/crear`.
- Tolera SSR: si no se puede saber, devuelve `true` y el lado cliente re-evalúa post-hidratación (o redirige).

### 12.2 `customerGuestGuard`

- Protege `/cuenta/ingresar` y `/cuenta/crear`.
- Si ya hay customer hidratado, redirige a `/cuenta` (o `returnUrl`).

### 12.3 Reglas

- **No usar guards admin** (no existe `permissionGuard`/`globalAccessGuard` en storefront).
- **No mezclar roles admin con customer.** El JWT admin y el JWT customer son universos disjuntos.
- Guards funcionales, no clases.

---

## 13. Diseño y Design System

### 13.1 Reglas innegociables

- Seguir **estrictamente** los mockups del paquete (`vertex_commerce_design_package/`).
- **No** inventar colores, tipografías ni iconografía.
- Aplicar tokens del DS:
  - `--vtx-color-ivory: #F7F2EA;` (warm ivory)
  - `--vtx-color-sand: #E9DFCD;`
  - `--vtx-color-stone: #C9BFAE;`
  - `--vtx-color-taupe: #8C7E6A;`
  - `--vtx-color-charcoal: #2A2723;`
  - `--vtx-color-copper: #B86E4A;` (CTA principal)
  - `--vtx-color-terracotta: #C8794A;` (CTA hover/secundario)
  - Spacing: scale 4-8-12-16-24-32-48-64-96 px.
  - Type: tipografía editorial (serif headings, sans body) — exacta según `Vertex_Design_System_Book_V2_Premium_Visual.pdf`.
- Cards suaves (radius 8-12, shadow muy sutil).
- Estética **editorial luxury, cálida, mobile-first**.
- **Prohibido:**
  - Estética marketplace (Mercado Libre, Amazon).
  - Estética admin/backoffice (sidebar denso, tablas).
  - Templates genéricos.
  - Colores fluor o saturados ajenos a la paleta.
  - Inventar layouts cuando hay mockup explícito.
- Si una decisión improvisada contradice mockup o DS, **gana el DS**.
- Si una pantalla no tiene mockup explícito, se construye derivando del DS y de patrones de pantallas hermanas — nunca improvisar libremente.

### 13.2 Componentes base (shared/ui/)

| Componente | Usado en | Notas |
|---|---|---|
| `VtxButton` | global | variantes: `primary` (copper), `secondary`, `ghost`, `link`; sizes; loading; iconSlot |
| `VtxInput` | forms | label flotante, error inline, mask opcional |
| `VtxSelect` | forms / SortSelect | options input |
| `VtxCheckbox` | forms / filtros | |
| `VtxRadio` | forms / shipping/payment | |
| `VtxBadge` | ProductCard, OrderTimeline | tones: `info`, `success`, `warning`, `danger`, `neutral` |
| `VtxPrice` | ProductCard, PDP, Cart | maneja `price` + `compareAtPrice` (tachado) + currency |
| `VtxSkeleton` | loading states | shapes: `text`, `card`, `image` |
| `VtxEmptyState` | búsqueda/pedidos/cart vacíos | icon + title + cta |
| `VtxToast` | feedback | tones |
| `VtxModal` | confirmaciones | accesible (focus trap) |
| `VtxDrawer` | filtros mobile / cart drawer | side `left/right`, focus trap |

### 13.3 Componentes commerce

| Componente | Usado en | Datos | Endpoint relacionado |
|---|---|---|---|
| `StorefrontHeader` | shell | navigation, store, customer? | `/storefront/navigation/HEADER`, `/storefront/store`, `/customer/me` |
| `StorefrontFooter` | shell | footer nav, store | `/storefront/navigation/FOOTER` |
| `ProductCard` | Home/PLP/Search/Related/Wishlist | `PublicProductCardDto` | — |
| `ProductGrid` | múltiple | items + loading | — |
| `ProductGallery` | PDP | images[] | — |
| `ProductInfo` | PDP | producto | `POST /storefront/cart/validate` (CTA) |
| `CategoryCard` | Home/drawer | categoría | — |
| `HeroSection` | Home (CMS) | section.configJson | — |
| `PromoBanner` | Home (CMS) | section.configJson | — |
| `CMSSectionRenderer` | Home/CMS pages | sections[] | — |
| `Breadcrumb` | PLP/PDP/customer | trail | — |
| `FilterSidebar` | PLP desktop | filters config | — |
| `FilterDrawer` | PLP mobile | idem | — |
| `SortSelect` | PLP/Search | options | — |
| `QuantityStepper` | PDP/Cart | min/max/value | — |
| `CartItem` | Cart/drawer | item | — |
| `CartSummary` | Cart/Checkout | totals | — |
| `CheckoutStepper` | Checkout | step | — |
| `ShippingMethodCard` | Cart/Checkout | método + selección | `/shipping-methods`, `/shipping/quote` |
| `PaymentMethodCard` | Checkout | método + selección | `/payment-methods` |
| `OrderSummary` | Cart/Checkout/Success/Detalle | pedido | — |
| `OrderTimeline` | Tracking, Pedido detalle | shipments + statusHistory | `/orders/:n/tracking` |
| `SearchOverlay` | Header | suggestions | `/search/suggestions` |
| `AccountSidebar` | Customer area | items | — |

---

## 14. Routing propuesto

### 14.1 Rutas públicas (sin auth)

```
/                                       → HomePageComponent (resolver)
/categorias/:slug                       → CategoryPageComponent (resolver)
/productos/:slug                        → ProductPageComponent (resolver)
/buscar                                 → SearchPageComponent
/carrito                                → CartPageComponent
/checkout                               → CheckoutPageComponent
/pedido/exito/:orderNumber              → OrderSuccessPageComponent
/seguimiento                            → OrderLookupPageComponent
/p/:slug                                → CmsPageComponent (resolver)
/nosotros                               → CmsPageComponent (slug: 'nosotros')
/contacto                               → CmsPageComponent (slug: 'contacto')
/faq                                    → CmsPageComponent (slug: 'faq')
/envios                                 → CmsPageComponent (slug: 'envios')
/cambios-y-devoluciones                 → CmsPageComponent (slug: 'cambios-y-devoluciones')
/medios-de-pago                         → CmsPageComponent (slug: 'medios-de-pago')
/terminos                               → CmsPageComponent (slug: 'terminos')
/privacidad                             → CmsPageComponent (slug: 'privacidad')
/404                                    → NotFoundPageComponent
**                                      → redirect → /404
```

### 14.2 Rutas customer (con `customerAuthGuard`/`customerGuestGuard`)

```
/cuenta/ingresar                        → LoginPageComponent (guest)
/cuenta/crear                           → RegisterPageComponent (guest)
/cuenta                                 → AccountDashboardPageComponent (auth)
/cuenta/pedidos                         → OrdersListPageComponent (auth)
/cuenta/pedidos/:orderNumber            → OrderDetailPageComponent (auth)
/cuenta/perfil                          → ProfilePageComponent (auth)
/cuenta/perfil/cambiar-password         → ChangePasswordPageComponent (auth)
/cuenta/favoritos                       → WishlistPageComponent (auth opcional)
```

### 14.3 Reglas

- Lazy loading por feature: cada `<feature>.routes.ts` se carga con `loadChildren`.
- Forgot/Reset NO se incluye hasta que exista endpoint backend.
- `/cuenta/favoritos` puede ser accesible sin auth (wishlist es local), pero el `AccountSidebar` se renderiza igual y sugiere login para sincronizar a futuro.

---

## 15. SSR y SEO

### 15.1 Páginas con SSR fuerte (HTML completo en respuesta inicial)

| Página | SSR | Razón |
|---|---|---|
| Home | sí | landing principal, performance crítica |
| PLP | sí | indexable, descubrimiento orgánico |
| PDP | sí | rich snippets, JSON-LD `Product` |
| CMS pages | sí | contenido institucional |
| Search | parcial | shell SSR + resultados client-side |
| Cart, Checkout, área customer | no | requieren cookies/cliente, `noindex` |

### 15.2 Title y Meta

- Por ruta usar `Title` (`@angular/platform-browser`) y `Meta`.
- Plantilla: `${pageTitle} · ${storeName}` donde `storeName` viene de `StoreContextStore`.
- Para PDP: title = `${product.title} · ${storeName}`, description = `product.shortDescription || product.metaDescription`.
- Para PLP: title = `${category.name} · ${storeName}`, description = `category.metaDescription`.
- `<link rel="canonical">` por ruta.
- Open Graph (`og:title`, `og:description`, `og:image`) en Home, PLP, PDP, CMS.
- JSON-LD `Product` en PDP, `BreadcrumbList` en PLP/PDP, `Organization` en Home.

### 15.3 Datos de `/storefront/config`

- `seo.defaultTitle`, `seo.defaultDescription`, `seo.ogImage`.
- `theme` (colores, logo) se inyectan en CSS variables a nivel root al boot SSR.

### 15.4 Skeleton/loading post-hydration

- En SSR ya viene contenido renderizado: el skeleton solo aparece en navegaciones cliente.
- Evitar “double flicker” usando `provideClientHydration()` y, si hace falta, `TransferState`.

### 15.5 TransferState (opcional pero recomendado)

- Cachear en TransferState la respuesta SSR de Home/PLP/PDP/CMS para que el cliente NO repita el `GET` tras hidratar.
- Implementar utilitario `withTransferCache()` o usar interceptor genérico.

### 15.6 Sitemap y robots.txt

- `sitemap.xml` generado en build con rutas estáticas + slugs de categorías + productos + pages CMS (`GET /storefront/categories`, `GET /storefront/products?limit=...`, `GET /storefront/pages`).
- `robots.txt`: `Disallow: /carrito`, `/checkout`, `/cuenta/*`, `/seguimiento` (privacidad).

---

## 16. Manejo de carrito

### 16.1 Flujo

1. Usuario click “Agregar al carrito” en PDP/PLP.
2. `CartStore.add({ productStoreId, slug, quantity })` muta el array en memoria.
3. `CartStore` persiste en `localStorage['vtx_cart_v1']`.
4. `CartStore.validate()` (debounced 300 ms) llama `POST /storefront/cart/validate` con `{ items: [...] }`.
5. Backend devuelve por línea: `unitPrice`, `subtotal`, `taxTotal: '0.00'`, `unavailable?`, `stockStatus`.
6. UI muestra subtotal real y warnings si `unavailable === true` (producto despublicado o sin stock).
7. Antes de entrar a `/checkout`, `CartStore.validate()` se llama nuevamente y bloquea el avance si hay líneas inválidas.

### 16.2 Datos locales vs autoridad backend

| Dato | Fuente | Notas |
|---|---|---|
| Items + cantidad | `localStorage` | autoridad cliente |
| `productStoreId`, `slug` | `localStorage` | snapshot |
| Precio unitario, subtotal, impuestos, totales | **backend** (`/cart/validate`) | jamás confiar en localStorage para precios |
| Disponibilidad (`stockStatus`, `unavailable`) | **backend** | re-evaluar en cada validate |
| Selección de envío y costo en cart preview | `ShippingService` (memoria) | no persistir en localStorage |

### 16.3 Edge cases

- Cliente cierra y vuelve: `localStorage` rehidrata, se llama `validate()` al boot del cart page.
- Producto despublicado: línea marcada `unavailable: true` → UI muestra warning y permite quitarla.
- Cambio de tienda (subdominio): `CartStore` se vacía si `storeKey` cambia (key separada por store key: `vtx_cart_v1__<storeKey>`).
- `localStorage` deshabilitado (modo privado): cart vive solo en memoria, se limpia al cerrar tab.

---

## 17. Manejo de checkout

### 17.1 Flujo

1. **Validar carrito** (`POST /storefront/cart/validate`). Si hay líneas inválidas, redirigir a `/carrito`.
2. **Cargar shipping methods** (`GET /storefront/shipping-methods`).
3. **Cotizar envío** (`POST /storefront/shipping/quote` con `postalCode + items`).
4. **Cargar payment methods** (`GET /storefront/payment-methods`).
5. **Completar datos** (cliente, dirección, contacto).
6. **Submit** (`POST /storefront/checkout` con DTO completo).
7. **Redirigir** a `/pedido/exito/:orderNumber` con state `{ order }`.
8. **Guardar temporalmente** `orderNumber + customerEmail` en `sessionStorage['vtx_last_order']` para refresh / lookup.
9. **Limpiar** `CartStore` al éxito.

### 17.2 Reglas

- Nunca guardar datos sensibles (tarjetas, CVV) en frontend — V1 no procesa pagos online.
- El form mantiene state en `CheckoutStore` mientras navega entre steps.
- Validación cliente antes de submit (Reactive Forms o signals + valibot/zod).
- Tras submit exitoso, **no permitir “volver atrás”**: si el usuario hace back, `CheckoutStore` está vacío y redirige a `/`.
- Al fallar el submit con 400, mostrar errores por campo.
- Loading state en el botón “Confirmar pedido” (deshabilitado, spinner).

### 17.3 Stepper

- Step 1 — Datos: email, nombre, teléfono.
- Step 2 — Envío: dirección + método + cotización.
- Step 3 — Pago: método (transferencia / efectivo / etc. según `paymentSettingsJson` del store).
- Step 4 — Revisión: resumen + términos + submit.

---

## 18. Manejo de customer auth

### 18.1 Flujos

- **Register:** `POST /customer/auth/register` → setea cookies → store hidratado → redirect a `returnUrl` o `/cuenta`.
- **Login:** `POST /customer/auth/login` → setea cookies → idem.
- **Hidratar `me`:** al boot del shell autenticado, o al entrar a `/cuenta/*`, se llama `GET /customer/me`. Si 200 → store con customer; si 401 → store null.
- **Logout:** `POST /customer/auth/logout` → backend limpia cookies → store reset → redirect a `/`.
- **Profile update:** `PATCH /customer/profile` con CSRF.
- **Change password:** `POST /customer/profile/change-password` con CSRF; actualizar UI con toast “Contraseña actualizada”.
- **Orders:** `GET /customer/orders?page=&limit=`.
- **Order detail + tracking:** `GET /customer/orders/:n` + `/tracking`.

### 18.2 Reglas críticas

- **Cookie httpOnly** `vtx_customer_at`: el frontend NUNCA la lee.
- **Cookie no-httpOnly** `vtx_customer_csrf`: el frontend la lee y la envía en header `x-csrf-token` solo en mutaciones a `/customer/*`.
- **`withCredentials: true`** en todas las llamadas a `/customer/*`.
- **NO guardar token JWT** en `localStorage`, `sessionStorage` ni memoria expuesta. La cookie es la única fuente.
- **NO usar Authorization Bearer** para customer (es para admin).
- En SSR, leer cookies desde `Request` server-side (helper `core/tokens/cookie.token.ts`).
- En logout, además de la llamada al backend, limpiar `AuthCustomerStore` y redirigir.

---

## 19. Roadmap de implementación frontend

> **Regla de oro:** completar una wave entera con build/lint/tests verdes antes de avanzar. Detenerse al final de cada wave y reportar.

### Wave 0 — Setup + Shell

- **Objetivo:** app Angular SSR navegable, sin features de negocio.
- **Páginas:** placeholder Home + 404.
- **Endpoints:** `/storefront/store`, `/storefront/config`, `/storefront/navigation/HEADER`, `/storefront/navigation/FOOTER`.
- **Componentes:** `StorefrontShell`, `StorefrontHeader` (placeholder), `StorefrontFooter` (placeholder), `VtxButton`, `VtxInput`, `VtxSkeleton`, `VtxBadge`, `VtxPrice`.
- **Criterio de salida:**
  - `npm run build` OK, `npm run dev:ssr` arranca, console limpia.
  - Header pinta nombre de la tienda real.
  - Tokens CSS aplicados en root.
  - Lint + tests unit base verdes.
- **NO hacer:** Home dinámica, PLP, cart, customer.

### Wave 1 — Home + CMS

- **Objetivo:** Home conectada y páginas CMS genéricas.
- **Páginas:** Home (`/`), CMS genérica (`/p/:slug`), 404.
- **Endpoints:** `/storefront/home`, `/storefront/pages/:slug`, `/storefront/featured`, `/storefront/categories`, `/storefront/navigation/*`.
- **Componentes:** `CMSSectionRenderer` (HERO, BANNER, PRODUCT_GRID, RICH_TEXT, CATEGORY_GRID), `HeroSection`, `PromoBanner`, `CategoryCard`, `ProductCard`, `ProductGrid`, footer real.
- **Criterio de salida:**
  - Home renderiza secciones reales del backend.
  - SSR con title/meta correctos.
  - Páginas CMS legales accesibles.
  - Mobile y desktop pulidos según mockup `01_storefront_mockups/`.
- **NO hacer:** PLP avanzado, PDP, cart.

### Wave 2 — Catalog

- **Objetivo:** PLP, PDP, search.
- **Páginas:** `/categorias/:slug`, `/productos/:slug`, `/buscar`.
- **Endpoints:** `/storefront/products`, `/storefront/categories/:slug`, `/storefront/products/:slug`, `/storefront/products/:slug/related`, `/storefront/search`, `/storefront/search/suggestions`.
- **Componentes:** `Breadcrumb`, `FilterSidebar`, `FilterDrawer`, `SortSelect`, `ProductGallery`, `ProductInfo`, `RelatedProducts`, `SearchOverlay`.
- **Criterio de salida:**
  - PLP con filtros simples (precio, sort) + paginación.
  - PDP con galería + info + related.
  - Header search overlay con suggestions debounced.
  - JSON-LD `Product` en PDP.
- **NO hacer:** facets dinámicos avanzados, comparador.

### Wave 3 — Cart

- **Objetivo:** carrito local validado contra backend.
- **Páginas:** `/carrito`.
- **Endpoints:** `POST /storefront/cart/validate`, `GET /storefront/shipping-methods`, `POST /storefront/shipping/quote` (preview opcional).
- **Componentes:** `CartItem`, `CartSummary`, `QuantityStepper`, `CartDrawer`.
- **Criterio de salida:**
  - Agregar desde PLP/PDP, modificar cantidad, eliminar.
  - Validate en cada cambio (debounced) + al cargar la página.
  - Preview de costo de envío si hay CP guardado.
  - Empty state.
- **NO hacer:** server-side cart, cupones.

### Wave 4 — Checkout

- **Objetivo:** flujo de compra completo + confirmación + tracking anónimo.
- **Páginas:** `/checkout`, `/pedido/exito/:orderNumber`, `/seguimiento`.
- **Endpoints:** `POST /storefront/checkout`, `GET /storefront/payment-methods`, `GET /storefront/shipping-methods`, `POST /storefront/shipping/quote`, `GET /storefront/orders/lookup`, `GET /storefront/orders/:orderNumber/tracking`.
- **Componentes:** `CheckoutStepper`, `ShippingMethodCard`, `PaymentMethodCard`, `OrderSummary`, `OrderTimeline`.
- **Criterio de salida:**
  - Submit exitoso crea pedido real.
  - Order Success carga datos por state o fallback lookup.
  - `/seguimiento` funciona con orderNumber + email.
  - 429 manejado con UX amigable.
- **NO hacer:** integrar gateway real, MP/PSP, courier real.

### Wave 5 — Customer

- **Objetivo:** área autenticada del cliente.
- **Páginas:** `/cuenta/ingresar`, `/cuenta/crear`, `/cuenta`, `/cuenta/pedidos`, `/cuenta/pedidos/:n`, `/cuenta/perfil`, `/cuenta/perfil/cambiar-password`, `/cuenta/favoritos`.
- **Endpoints:** `/customer/auth/*`, `/customer/me`, `/customer/profile*`, `/customer/orders*`.
- **Componentes:** `AccountSidebar`, `OrderTimeline`, `WishlistPage`.
- **Criterio de salida:**
  - Login + register + logout funcionando con cookies.
  - Guards activos.
  - Mis Pedidos paginado.
  - Pedido Detalle + tracking.
  - Perfil edit + cambio password con CSRF.
  - Wishlist local funcional.
- **NO hacer:** forgot/reset, direcciones server, refresh token.

### Wave 6 — Mobile polish + QA

- **Objetivo:** producción-ready.
- **Trabajo:**
  - Responsive completo (validar contra `Vertex_UI_Spec_Responsive_Rules.pdf`).
  - A11y: focus visible, contrastes, ARIA, navegación teclado.
  - Performance: lazy images (`loading="lazy"`, `srcset`), bundle size, hydration cost.
  - SEO: sitemap, robots, JSON-LD, canonicals, OG completos.
  - Lighthouse Mobile: Performance ≥ 80, A11y ≥ 90.
  - Playwright e2e: smoke flow checkout completo.
  - Empty/loading/error states pulidos en todas las pantallas.
- **Criterio de salida:**
  - Lighthouse OK en Home/PLP/PDP móviles.
  - 1 test e2e Playwright que cubre PLP → PDP → Cart → Checkout → Success.
  - Lint + unit + e2e verdes.
- **NO hacer:** features nuevas.

---

## 20. Gaps y decisiones pendientes

### 20.1 Tabla de gaps

| # | Gap | Tipo | Bloquea Wave 0 | Bloquea Wave 4 | Bloquea productivo | Acción |
|---|---|---|---|---|---|---|
| 1 | Seeds reales `paymentSettingsJson` (MP, transferencia, efectivo) | data | ❌ | ⚠️ sí | sí | crear seed declarativo en `prisma/seed.demo.ts` antes de Wave 4 |
| 2 | Seeds reales `shippingSettingsJson` (zonas AR, Andreani flat, retiro) | data | ❌ | ⚠️ sí | sí | idem |
| 3 | Contenido CMS real (home, páginas legales) | data | ❌ | parcial | sí | seed Pages + sections antes de Wave 1/2 |
| 4 | Dominio local por tienda demo (`bazar.localhost`) | infra | ❌ | no | sí | usar `x-store-key` en dev; hosts opcional |
| 5 | URL final API (`apiBase`) y dominio storefront | decisión | ⚠️ | sí | sí | definir antes de Wave 0 |
| 6 | Decisión cookie domain customer (`.vertex.com.ar` vs por tienda) | decisión | no | no | ⚠️ sí | resolver antes de Wave 5 / productivo |
| 7 | CORS origins permitidos en backend | infra backend (no cambia código frontend) | ⚠️ | sí | sí | env `CORS_ORIGINS` agregar dominios storefront |
| 8 | Endpoint `POST /contact` | backend gap | no | no | parcial | V1: form local con `mailto:` o nada server; postergar |
| 9 | `/customer/auth/forgot-password`, `/reset-password` | backend gap | no | no | ⚠️ sí | postergar Wave F backend; UI muestra “Próximamente” |
| 10 | `/customer/auth/refresh` | backend gap | no | no | no | cookie 7d alcanza V1 |
| 11 | `/storefront/categories/:slug/facets` | backend gap | no | no | no | filtros estáticos del query DTO en V1 |
| 12 | Búsqueda full-text avanzada / sinónimos | backend gap | no | no | no | `contains` alcanza V1 |
| 13 | Wishlist server backend | backend gap | no | no | no | `localStorage` V1 |
| 14 | Direcciones del customer (libreta) | backend gap | no | no | no | scope V2 confirmó exclusión |
| 15 | Mockups mobile dedicados (cart, lookup, forgot) | diseño | no | parcial | no | derivar de DS responsive rules |
| 16 | Mockups forgot/reset | diseño | no | no | depende de gap 9 | derivar del DS si habilita Wave F |

### 20.2 Decisiones requeridas antes de Wave 0

1. **`apiBase`** definitivo (puede ser temporal: `http://localhost:3000` en dev).
2. **CORS origins** en backend env (`CORS_ORIGINS`).
3. **Aceptación** de gaps 8–16 como “fuera de Wave 0–5”.
4. **Confirmación** del nombre del repo storefront y ubicación.
5. **Nombre de tienda demo** y `storeKey` para desarrollo.

---

## 21. Reglas de implementación para Codex

> Codex debe leer este documento completo **antes** de tocar código.

### 21.1 Restricciones absolutas

1. **NO modificar `backend/`.** Cero cambios en controllers, services, DTOs, schema.prisma, migraciones o seeds del backend. Si falta un endpoint, documentar gap y postergar.
2. **NO modificar `backoffice-admin/`.** Es otra app. Si se descubre desalineación, reportar — no editar.
3. **NO consumir `/admin/*`.** Verificable por grep: `grep -r "/admin/" storefront/src/` debe dar 0 matches en código de runtime.
4. **NO consumir `/public/*`** legacy desde storefront nuevo.
5. **Usar las rutas reales auditadas** en este documento (§6). Si una ruta del prompt maestro V2 contradice este documento, gana este documento.
6. **No hardcodear** `storeId`, JWT, secrets, ni datos que deban venir del backend (categorías, productos, banners, menús, páginas).
7. **No guardar JWT customer en `localStorage`/`sessionStorage`.** La cookie httpOnly es la única fuente.

### 21.2 Restricciones de proceso

1. **Implementar por waves** (§19). No saltarse waves ni mezclar features de wave futura.
2. **Detenerse al final de cada wave** y reportar:
   - Archivos creados/modificados (lista).
   - Comandos corridos (build, lint, test).
   - Resultado de build/lint/tests (verde/rojo + counts).
   - Endpoints consumidos en esa wave.
   - Componentes creados.
   - Gaps encontrados nuevos.
   - Decisiones tomadas que no estaban en este documento.
3. **No avanzar de wave** sin build limpio, lint limpio y tests verdes.
4. **Respetar mockups y Design System** (§13). Si falta mockup, derivar del DS — nunca improvisar.
5. **No agregar dependencias** sin justificación documentada.
6. **No modificar este documento** sin que el humano lo apruebe; sí se puede agregar un changelog separado en `Vertex Store Hogar/CHANGELOG_FRONTEND.md` si fuera necesario.

### 21.3 Restricciones técnicas

1. Standalone components siempre. Sin NgModules.
2. Signals para state. Evitar BehaviorSubject manual salvo necesidad puntual.
3. SSR habilitado en Home/PLP/PDP/CMS desde Wave 0.
4. TypeScript `strict: true` y `noUncheckedIndexedAccess: true`.
5. ESLint + Prettier con config alineada al backoffice-admin.
6. Lazy loading por feature.
7. CSS tokens en `styles/tokens.css`. No introducir librería UI externa (Material, PrimeNG, Tailwind) sin aprobación.
8. Imports relativos prohibidos en cross-feature: usar paths absolutos `@app/core/...`.

### 21.4 Restricciones de seguridad

1. CSP estricta (a configurar en server.ts SSR).
2. Sanitizar HTML CMS si se usa `innerHTML` (Angular ya lo hace por default; no usar `bypassSecurityTrustHtml` salvo justificación).
3. Validar entradas de formulario en cliente y confiar en backend para validación final.
4. No loguear datos sensibles (passwords, emails completos, números de pedido) en console en producción.

---

## 22. Criterio de aceptación del documento

| # | Criterio | Cumplido |
|---|---|---|
| 1 | Explica arquitectura Storefront V1 (qué es, qué no es, cómo se diferencia del backoffice) | §1, §2 ✅ |
| 2 | Usa nombres correctos: Storefront V1 / Tienda Pública V1; no llama “e-commerce V2” | todo el documento ✅ |
| 3 | Lista endpoints reales (validados contra controllers) | §6, §7 ✅ |
| 4 | Mapea páginas a endpoints | §8 ✅ |
| 5 | Define services / stores / interceptors / guards | §9, §10, §11, §12 ✅ |
| 6 | Define componentes base + commerce | §13 ✅ |
| 7 | Define routing público + customer | §14 ✅ |
| 8 | Define roadmap por waves con criterios de salida | §19 ✅ |
| 9 | Define gaps con clasificación de bloqueo | §20 ✅ |
| 10 | Sirve como guía directa para Codex | §21 ✅ |

**Estado:** documento completo. Listo para revisión humana.

**Próximo paso:** esperar aprobación. NO implementar nada hasta confirmación explícita.
