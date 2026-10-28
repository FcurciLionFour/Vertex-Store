# PROMPT MAESTRO — Implementación Tienda Pública Vertex Commerce V2

## Contexto

Este prompt inicia oficialmente la implementación de la tienda pública de Vertex Commerce V2.

El backoffice/admin ya fue trabajado previamente. Ahora el foco es la tienda pública/storefront.

Vertex Commerce es un sistema multi-ecommerce. Un único backoffice administra múltiples tiendas, y cada tienda tiene catálogo, branding, CMS, menús, páginas, banners, productos publicados, inventario, pedidos, caja, gastos y reportes propios.

La tienda pública debe renderizar la experiencia de una tienda concreta, sin mezclar datos de tiendas distintas.

---

## Assets obligatorios incluidos en el ZIP

Antes de implementar, revisar el paquete de diseño adjunto:

- `00_referencias_originales/`
- `01_storefront_mockups/`
- `02_customer_account_mockups/`
- `03_design_system_boards/`
- `04_design_system_pdfs/`
- `05_admin_backoffice_mockups/`

Estos archivos son referencia obligatoria.  
El diseño, colores, spacing, estilo, componentes, iconografía, jerarquía visual y experiencia premium NO se negocian.

Si una decisión improvisada contradice los mockups o el design system, ganan los mockups y el design system.

---

## Diseño obligatorio

La tienda debe sentirse:

- premium
- cálida
- moderna
- confiable
- clara
- rápida
- mobile-first
- editorial
- no genérica
- no marketplace
- no template barata
- no Amazon
- no MercadoLibre
- no estética de backoffice

Inspiración de calidad:

- Zara Home
- West Elm
- Crate & Barrel
- Restoration Hardware

No copiar directamente, solo tomar claridad, calidez y jerarquía.

---

## Storefront mockups obligatorios

Usar los mockups del ZIP como referencia para:

1. Home desktop
2. Category / PLP desktop
3. Product Detail / PDP desktop
4. Cart desktop
5. Checkout desktop
6. Order Success desktop
7. Login / Register desktop
8. Mi Cuenta / Dashboard cliente
9. Mis Pedidos
10. Pedido Detalle
11. Wishlist / Favoritos
12. Search Results
13. CMS / Nosotros / About page

---

## Design System obligatorio

Usar como referencia:

1. Foundations
2. Buttons & Forms
3. Commerce Components
4. Navigation & Layout
5. Feedback & States
6. Illustrations
7. Icons
8. Vertex UI Spec + Responsive Rules
9. Vertex Design System Book Premium

Paleta base:

- warm ivory
- sand
- stone
- taupe
- charcoal
- copper / terracotta para CTAs

No introducir colores nuevos sin justificación.

---

## Regla crítica de endpoints

No usar endpoints `/admin/*` para el storefront público.

Los endpoints admin son exclusivamente del backoffice.

La tienda pública debe consumir endpoints públicos, storefront o customer.

Si faltan endpoints públicos, no usar admin para salir del paso. Primero auditar, documentar y crear/proponer los endpoints correctos.

---

## Auditoría obligatoria antes de implementar

Antes de escribir código de tienda:

1. Auditar controllers/routes/services del backend.
2. Listar endpoints existentes.
3. Separar endpoints admin vs storefront/customer.
4. Marcar qué endpoints puede usar la tienda.
5. Detectar faltantes.
6. Proponer plan backend mínimo.
7. Recién después implementar storefront shell.

Formato de auditoría:

| Módulo | Endpoint | Método | Público/Admin | ¿Usar en tienda? | Observación |
|---|---|---|---|---|---|

---

## Endpoints públicos esperados

### Storefront / tienda

- `GET /storefront/store`
- `GET /storefront/store/by-domain`
- `GET /storefront/config`
- `GET /storefront/navigation`
- `GET /storefront/footer`

### Home / CMS

- `GET /storefront/home`
- `GET /storefront/pages/:slug`
- `GET /storefront/sections`
- `GET /storefront/banners`

### Categorías

- `GET /storefront/categories`
- `GET /storefront/categories/:slug`
- `GET /storefront/categories/:slug/products`

### Productos

- `GET /storefront/products`
- `GET /storefront/products/:slug`
- `GET /storefront/products/featured`
- `GET /storefront/products/search`
- `GET /storefront/products/:id/related`

El producto público no debe exponer costo interno, margen interno, proveedor interno ni datos administrativos.

### Search

- `GET /storefront/search?q=...`

Debe soportar filtros por categoría, precio, color, material, disponibilidad y promociones.

### Cart

Definir si será local, por sesión o persistente.

Endpoints posibles:

- `GET /storefront/cart`
- `POST /storefront/cart/items`
- `PATCH /storefront/cart/items/:id`
- `DELETE /storefront/cart/items/:id`
- `POST /storefront/cart/apply-coupon`

### Checkout

- `POST /storefront/checkout`
- `POST /storefront/orders`
- `GET /storefront/payment-methods`
- `GET /storefront/shipping-methods`
- `POST /storefront/shipping/quote`
- `POST /storefront/payments/init`

### Customer / Orders

- `GET /customer/orders`
- `GET /customer/orders/:id`
- `GET /customer/orders/:id/tracking`
- `GET /storefront/order-lookup`

### Customer Auth

- `POST /customer/auth/login`
- `POST /customer/auth/register`
- `POST /customer/auth/logout`
- `GET /customer/auth/me`
- `POST /customer/auth/forgot-password`
- `POST /customer/auth/reset-password`

No mezclar auth admin con auth cliente.

### Wishlist

- `GET /customer/wishlist`
- `POST /customer/wishlist/items`
- `DELETE /customer/wishlist/items/:id`

---

## Roadmap de implementación

### Wave 1 — Auditoría y base

- Auditar endpoints.
- Crear mapa Storefront API.
- Detectar faltantes.
- Definir estrategia de store context.
- No empezar Home todavía.

### Wave 2 — Storefront shell

- Layout público.
- Header.
- Footer.
- Navegación.
- Store context.
- Theme tokens.
- Responsive base.

### Wave 3 — Home

- Hero.
- Categorías.
- Beneficios.
- Productos destacados.
- Banners editoriales.
- Colecciones.
- Newsletter.
- Footer.

Todo DB-driven o preparado para DB.

### Wave 4 — PLP / Category

- Breadcrumb.
- Hero categoría.
- Filtros.
- Ordenamiento.
- Grid.
- Load more / paginación.
- Empty state.

### Wave 5 — PDP

- Gallery.
- Variantes.
- Precio.
- Stock.
- Cantidad.
- Add to cart.
- Beneficios.
- Descripción.
- Relacionados.
- Sticky CTA mobile futuro.

### Wave 6 — Cart

- Items.
- Cantidades.
- Eliminar.
- Subtotal.
- Descuentos.
- Envío estimado.
- Total.
- CTA checkout.

### Wave 7 — Checkout

- Contacto.
- Envío.
- Pago.
- Revisión.
- Crear orden real.

### Wave 8 — Order Success + tracking

- Número real.
- Resumen real.
- Tracking inicial.
- Soporte.
- CTA seguir comprando / ver pedido.

### Wave 9 — Customer account

- Login / registro.
- Mi cuenta.
- Mis pedidos.
- Pedido detalle.
- Wishlist.
- Direcciones.
- Perfil.

### Wave 10 — Search + CMS pages

- Search results.
- Nosotros.
- Contacto.
- FAQ.
- Envíos.
- Devoluciones.
- Medios de pago.
- Términos.
- Privacidad.

### Wave 11 — Mobile polish

- Home mobile.
- PLP mobile.
- PDP mobile.
- Cart mobile.
- Checkout mobile.

---

## Componentes reutilizables obligatorios

Crear componentes reutilizables:

- ProductCard
- CategoryCard
- HeroBanner
- PromoBanner
- Header
- Footer
- Breadcrumb
- PriceBlock
- QuantityStepper
- CartItem
- CheckoutStepper
- OrderTimeline
- EmptyState
- StatusBadge
- SearchBar
- FilterDrawer
- ProductGallery

Naming sugerido:

- `VtxButton`
- `VtxInput`
- `VtxProductCard`
- `VtxCategoryCard`
- `VtxHeader`
- `VtxFooter`
- `VtxPriceBlock`
- `VtxCartItem`
- `VtxOrderTimeline`

---

## Reglas de contenido

No hardcodear contenido que debe venir de DB:

- categorías
- banners
- productos destacados
- menús
- footer links
- páginas CMS
- promociones
- beneficios comerciales

Se puede usar mock data temporal solo si está claramente marcado como temporal.

---

## Criterio de aceptación

Una pantalla solo está terminada si:

1. Respeta mockup aprobado.
2. Usa componentes reutilizables.
3. Usa endpoints correctos.
4. No consume endpoints admin.
5. Está preparada para mobile.
6. Maneja loading/error/empty states.
7. No hardcodea datos que deben venir del backend.
8. No rompe multi-store.
9. Mantiene código limpio.
10. No introduce deuda visual ni técnica innecesaria.

---

## Nota final

Este proyecto empieza la tienda pública real de Vertex Commerce V2.

El objetivo es construir una experiencia cliente final premium, multi-store, mobile-first, limpia y productiva, respetando los mockups y el Design System incluidos en este ZIP.