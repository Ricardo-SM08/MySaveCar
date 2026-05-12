# 📘 Plan Maestro de Arquitectura e Implementación: MySaveCar
**Rol:** Tech Lead & Arquitecto de Software Senior  
**Stack:** Flutter (Dart) + Firebase (Auth, Firestore, Storage) + Provider  
**Package ID:** `com.example.myselftcar`  
**Firebase Project ID:** `dbcrudmyselftcar`  
**Modo de Desarrollo:** MVP (Firestore en Modo Test, sin telemetría, enfoque core-only)

---

## 🏗️ 1. Arquitectura del Proyecto y Entorno

### 📂 Estructura de Carpetas (Enfoque *Feature-First* + Provider)
La estructura prioriza la cohesión por dominio, facilitando el escalado paralelo y el aislamiento de la lógica de negocio. Se separan capas `data`, `domain` (lógica de negocio/contratos) y `presentation` (UI/Providers).

```
lib/
├── core/
│   ├── constants/          # Rutas, claves de config, límites de UI, mensajes estáticos
│   ├── theme/              # ThemeData, paleta, tipografía, componentes base
│   ├── utils/              # Formateadores, validadores, extensiones Dart
│   └── errors/             # Excepciones custom y manejo de fallos de red/BD
├── features/
│   ├── auth/               # Login, registro, recuperación, sesión
│   ├── profile/            # Gestión de cuenta, direcciones, KYC vendedor
│   ├── marketplace/        # Catálogo, feed, búsqueda, detalle, favoritos
│   ├── seller_center/      # CRUD de publicaciones, gestión de stock/listings
│   ├── transactions/       # Carrito, checkout, historial de órdenes
│   ├── communication/      # Chat en tiempo real, ofertas/negociación
│   └── admin_ops/          # (MVP ligero) Moderación básica y configuración
│       ├── data/           # Repositorios, datasources (Firestore/Storage), DTOs
│       ├── domain/         # Entidades puras, contratos de repositorio, casos de uso
│       └── presentation/   # Screens, widgets UI, ChangeNotifiers (Providers)
├── shared/
│   ├── routing/            # Configuración `go_router`, guards, deep links
│   ├── widgets/            # UI transversal (loaders, dialogs, empty states)
│   └── services/           # Utilidades compartidas (formatter, logger local)
├── firebase_options.dart   # Configuración multiplataforma auto-generada
└── main.dart               # Entry point, `MultiProvider` global, inicialización
```

### 📦 Gestión de Dependencias (`pubspec.yaml`)
*Estrategia:* Solo paquetes esenciales para el MVP. Se excluye explícitamente cualquier SDK de telemetría, analytics o crash reporting para cumplir con la restricción de ligereza.

| Categoría | Paquetes | Justificación Arquitectónica |
|-----------|----------|------------------------------|
| **Firebase Core** | `firebase_core`, `firebase_auth`, `cloud_firestore`, `firebase_storage` | Stack oficial para autenticación, base de datos en tiempo real y almacenamiento multimedia. |
| **Gestor de Estado** | `provider`, `equatable` | `provider` para inyección y reactividad ligera. `equatable` para comparación eficiente de estados y prevención de rebuilds innecesarios. |
| **Navegación** | `go_router` | Rutas declarativas, protección por autenticación, manejo de deep links y transiciones fluidas. |
| **Modelado & Mapeo** | `freezed`, `json_serializable`, `json_annotation` | Generación compile-time de entidades inmutables, `copyWith`, y mapeo seguro Firestore ↔ Dart. Reduce boilerplate y errores de runtime. |
| **UI & Assets** | `cached_network_image`, `flutter_svg`, `image_picker`, `path` | Carga optimizada de imágenes, renderizado SVG, selección de medios para Storage, manejo de rutas locales. |
| **Utilidades Core** | `intl`, `uuid`, `collection`, `flutter_lints` | Formateo de moneda/fechas (MXN), generación de IDs locales, algoritmos de filtrado/paginación, y estándares de calidad de código. |

---

## 🗃️ 2. Estrategia de Datos (Firestore NoSQL)

La migración del esquema relacional (PostgreSQL) a Firestore prioriza **lecturas rápidas**, **desnormalización controlada** y **agrupación por patrones de acceso**. Se eliminan joins complejos embebiendo datos frecuentemente consultados y se usan subcolecciones solo para datos escalables (mensajes, historial).

### 🔄 Mapeo Dominal: SQL → Firestore

| Dominio Relacional | Colección Firestore | Estrategia NoSQL & Optimización |
|-------------------|---------------------|----------------------------------|
| **Usuarios & Sesiones** | `users/{userId}` | Documento único con `email`, `rol`, `avatar_url`, `verificado`. Subcolección `addresses/` para direcciones. Datos de sesión se delegan a Firebase Auth; `push_tokens` se almacenan en un array dentro del doc para notificaciones (cuando se habiliten). |
| **Perfiles Vendedor** | `sellers/{sellerId}` | Vinculado por `userId`. Contiene `tipo_negocio`, `rfc`, `calificacion_avg`, `total_ventas`, `kyc_estado`. Para el feed, se **desnormaliza** `seller_name` y `seller_avatar` en `listings` para evitar lecturas cruzadas. |
| **Catálogo Técnico** | `vehicles/{vehicleId}` & `products/{productId}` | `vehicles`: ficha técnica estática (VIN, marca, modelo, año, specs). `products`: refacciones/insumos con `marca`, `numero_parte`, `compatibilidades` (array de mapas), `dimensiones`. Actúan como catálogos maestros; no se replican en cada listing. |
| **Publicaciones (Listings)** | `listings/{listingId}` | **Colección central de lectura.** Contiene: `seller_id`, `seller_snapshot` (nombre/avatar), `vehiculo_ref` o `producto_ref` + `specs_snapshot`, `precio`, `estado`, `visitas`, `imagenes_urls` (array), `ubicacion`, `timestamps`. Índices compuestos en `estado`, `tipo`, `precio`, `creado_en`. |
| **Carrito & Favoritos** | `cart/{userId}` & `favorites/{userId}` | Documentos raíz por usuario. `cart` contiene array de `items_carrito` con `listing_ref`, `cantidad`, `precio_snapshot`. `favorites` es mapa/set de `listingId` para `O(1)` lookup. Se sincroniza con cache local para modo offline. |
| **Órdenes & Transacciones** | `orders/{orderId}` | Documento plano con `buyer_id`, `direccion_snapshot`, `subtotal`, `descuento`, `total`, `estado`, `items_array` (cada item lleva `seller_id`, `listing_title`, `precio_unitario`, `estado_envio`). **Inmutabilidad:** Una vez creado, el precio y los items no cambian. Reflejo exacto del momento del checkout. |
| **Chat & Comunicación** | `conversations/{chatId}` → `messages/{msgId}` | `conversations` almacena `participants`, `last_message`, `last_sender`, `unread_counts`, `listing_ref`. Subcolección `messages` paginada (20 docs/consulta) con `remitente_id`, `contenido`, `adjunto_url`, `leido`, `timestamp`. Streams escalables y económicos. |
| **Ofertas & Negociación** | `offers/{offerId}` | Colección independiente vinculada a `listingId` y `buyerId`. Contiene `monto_oferta`, `contraoferta`, `estado`, `expira_en`. Se indexa por `listingId` para que el vendedor vea ofertas activas sin leer el chat. |

### ⚙️ Reglas de Optimización para MVP
- **Modo Test Firestore:** Se activará `allow read, write: if request.time < timestamp.date(2025, 12, 31);` para evitar fricción inicial. Se documentará la transición a reglas basadas en `auth.uid` y `resource.data` para producción.
- **Denormalización estratégica:** `seller_name`, `listing_price_snapshot`, `direccion_envio` se embeben en `orders` para eliminar lecturas adicionales y garantizar auditoría precisa.
- **Paginación obligatoria:** Todas las consultas de `listings`, `messages` y `orders` usarán `limit(20)` + `startAfterDocument()` para controlar costos de lectura.
- **Imágenes:** Solo se almacenán URLs en Firestore. La compresión y redimensionado se manejan en cliente antes del upload a Storage para reducir ancho de banda y costos.

---

## 🗺️ 3. Hoja de Ruta de Implementación Completa (End-to-End)

### 🔹 Fase 1: Setup y Configuración Inicial
- **Objetivo:** Preparar entorno local, vincular Firebase y establecer reglas base de desarrollo.
- **Actividades:**
  - Inicializar proyecto Flutter con IDs especificados.
  - Instalar Firebase CLI y ejecutar `flutterfire configure` para generar `firebase_options.dart`.
  - Configurar Firestore en **Modo de Prueba** (reglas abiertas con fecha de expiración).
  - Configurar VS Code (linters, formatters, emuladores opcionales).
  - Estructurar `pubspec.yaml` con dependencias aprobadas y ejecutar `pub get`.
- **Entregable:** Proyecto compilando en iOS/Android/Web, Firebase vinculado, entorno de desarrollo listo.

### 🔹 Fase 2: Arquitectura Base y Modelos
- **Objetivo:** Implementar estructura de carpetas y capas de datos tipadas.
- **Actividades:**
  - Crear árbol de directorios `Feature-First`.
  - Definir `ThemeData`, sistema de diseño y componentes UI base (botones, inputs, loaders).
  - Implementar entidades con `freezed` + `json_serializable` mapeando el esquema SQL a DTOs Dart.
  - Configurar `go_router` con rutas placeholder y guards de autenticación vacíos.
  - Establecer contratos de repositorios (interfaces abstractas) para `Auth`, `Firestore`, `Storage`.
- **Entregable:** Arquitectura escalable, modelos inmutables tipados, sistema de navegación base, contratos de datos definidos.

### 🔹 Fase 3: Autenticación y Gestión de Estado
- **Objetivo:** Implementar flujo de acceso seguro y manejo reactivo de sesión.
- **Actividades:**
  - Configurar `firebase_auth` (Email/Password, Google).
  - Crear `AuthProvider` (`ChangeNotifier`) con estados: `Unauthenticated`, `Loading`, `Authenticated`, `Error`.
  - Desarrollar pantallas de Login, Registro y Recuperación de Contraseña.
  - Implementar persistencia de sesión y redirección automática post-login.
  - Validar flujo con `test mode` activo y manejo de errores de red/credenciales.
- **Entregable:** Auth funcional, provider reactivo estable, flujos de login/register operativos, navegación protegida.

### 🔹 Fase 4: Perfiles y Roles
- **Objetivo:** Segmentar experiencias por rol (comprador vs vendedor) y gestionar datos personales.
- **Actividades:**
  - Crear `ProfileProvider` para CRUD de datos de usuario (`users` collection).
  - Implementar lógica de `perfiles_vendedor`: activación, carga de datos de negocio, KYC stub.
  - Desarrollar gestión de direcciones (`direcciones` → subcolección o array embebido según MVP).
  - Configurar routing condicional post-auth basado en `rol` (comprador → feed, vendedor → dashboard).
  - Validar persistencia y actualización en tiempo real.
- **Entregable:** Perfiles editables, segmentación por rol funcional, gestión de direcciones, routing dinámico.

### 🔹 Fase 5: Core del Marketplace (Vendedores)
- **Objetivo:** Permitir la creación, edición y publicación de listados con gestión multimedia.
- **Actividades:**
  - Implementar `SellerProvider` y formularios para crear `listings` (autos o productos).
  - Integrar `firebase_storage`: selección, compresión previa y upload de imágenes.
  - Almacenar URLs en `listings/{id}.imagenes` y gestionar estados (`borrador`, `activa`, `pausada`, `vendida`).
  - Desarrollar pantalla de "Mis Publicaciones" con filtros y acciones rápidas.
  - Validar validaciones de precio, specs obligatorios y límites de imágenes (MVP: máx 10).
- **Entregable:** CRUD de publicaciones funcional, upload de imágenes estable, gestión de estados de listado.

### 🔹 Fase 6: Core del Marketplace (Compradores)
- **Objetivo:** Experiencia de descubrimiento, búsqueda y detalle de producto.
- **Actividades:**
  - Desarrollar `MarketplaceProvider` con paginación de `listings` (20 items/scroll).
  - Implementar feed principal, navegación por categorías y búsqueda textual básica.
  - Crear filtros avanzados: rango de precio, marca, año, tipo, radio de ubicación.
  - Diseñar pantalla de `ListingDetail` con galería de imágenes, specs desnormalizados y CTA.
  - Agregar sistema de favoritos (`favorites/{userId}`) con UI de toggle.
- **Entregable:** Feed paginado, búsqueda/filtros operativos, detalle completo, favoritos persistentes.

### 🔹 Fase 7: Transacciones
- **Objetivo:** Flujo de carrito, checkout y generación de órdenes inmutables.
- **Actividades:**
  - Implementar `CartProvider` (estado local + sync a Firestore cuando hay auth).
  - Diseñar carrito UI: ajuste de cantidades, cálculo de subtotal, validación de stock/disponibilidad.
  - Crear flujo de checkout: selección de dirección, resumen, confirmación.
  - Implementar creación atómica de `orders` (snapshot de precios/items, asignación de `orderId`, estado `pendiente`).
  - Desarrollar `OrdersProvider` con historial y detalle por estado (`pendiente` → `pagada` → `enviada`).
- **Entregable:** Carrito funcional, checkout completo, generación de órdenes con snapshot de datos, historial visible.

### 🔹 Fase 8: Comunicación
- **Objetivo:** Chat en tiempo real y flujo de ofertas/negociación.
- **Actividades:**
  - Crear `ChatProvider` con streams de Firestore para `conversations` y `messages`.
  - Implementar UI de chat: lista de conversaciones, burbuja de mensajes, indicador de "no leídos".
  - Configurar paginación inversa de mensajes y scroll automático.
  - Integrar flujo de `offers`: botón "Hacer oferta", formulario, estado (`enviada`, `contraoferta`, `aceptada`, `rechazada`).
  - Validar sincronización en tiempo real y manejo de desconexiones/reconexión.
- **Entregable:** Chat bidireccional en tiempo real, sistema de ofertas funcional, estados sincronizados, UI responsiva.

### 🔹 Fase 9: Testing y QA
- **Objetivo:** Validar estabilidad, corrección de errores y preparación de código para release.
- **Actividades:**
  - Pruebas unitarias: Providers (estados, transiciones), Repositorios (mapeo, manejo de errores), Utilidades.
  - Pruebas de Widget: Navegación, formularios, loaders, empty states, flujos críticos (auth → checkout).
  - Pruebas manuales: Offline/online sync, límite de imágenes, paginación, rotación de pantalla, baja conectividad.
  - Profiling: Detección de rebuilds innecesarios, optimización de `Consumer`/`Selector`, validación de memory leaks.
  - Corrección de bugs MVP, limpieza de código, eliminación de logs de depuración.
- **Entregable:** Suite de pruebas funcional, estabilidad validada, rendimiento optimizado, código listo para producción.

### 🔹 Fase 10: Preparación y Despliegue
- **Objetivo:** Generar artefactos finales, configurar metadatos y desplegar a tiendas/web.
- **Actividades:**
  - Configurar íconos (`flutter_launcher_icons`), splash screens, y permisos de plataforma (iOS/Android).
  - Ejecutar `flutter build` para cada plataforma: App Bundle (Android), Archive (iOS), Web (JS/CSS/HTML).
  - Configurar metadatos de tiendas: capturas de pantalla, descripciones, políticas de privacidad, categorías.
  - Preparar checklist de seguridad: revisión de `firebase_options`, eliminación de modo test, configuración de reglas de producción.
  - Desplegar a Firebase Hosting (Web), Google Play Console, App Store Connect.
- **Entregable:** Binarios firmados, metadatos completos, apps publicadas en revisión, web desplegada, documentación de release.

---

## ✅ Cierre Arquitectónico
Este plan establece una base **escalable, mantenible y estrictamente enfocada en el MVP**. Cumple con las restricciones críticas (Firestore modo test, cero telemetría, sin código en esta etapa) y traduce fielmente la lógica relacional del DDL a un modelo NoSQL optimizado para lecturas frecuentes y costos controlados.

**Próximo paso recomendado:** Validar la estructura de carpetas y la estrategia de desnormalización de `listings` y `orders`. Una vez aprobado, se puede proceder a la Fase 1 con generación de código modular y configuración de emuladores locales.

----
---
----
-----

# Prompt:

**Rol y Contexto:**
Actúa como un Tech Lead y Arquitecto de Software Senior experto en Flutter, Dart y ecosistemas de Firebase. Tu objetivo es diseñar un plan de implementación maestro y exhaustivo para una aplicación móvil multiplataforma llamada "MySaveCar".

**Descripción del Proyecto:**
"MySaveCar" es un marketplace integral especializado en el sector automotriz para la compra y venta de automóviles, insumos vehiculares y refacciones. La base de datos original fue diseñada en PostgreSQL (script adjunto al final) y migraremos su lógica estructural a Firestore.

**Stack Tecnológico:**

* **Framework:** Flutter (Dart) multiplataforma (iOS, Android, Web).
* **Entorno:** VS Code.
* **Backend/BaaS:** Firebase (Authentication, Firestore, Storage).
* **Gestor de Estado:** Provider.
* **Package Name:** `com.example.myselftcar`
* **Firebase Project ID:** `dbcrudmyselftcar`

**Restricciones Técnicas Críticas (Etapa MVP):**

1. Configurar Firebase Firestore en **Modo de Prueba (Standard/Test Mode)** para agilizar el desarrollo inicial sin bloqueos de reglas de seguridad complejas.
2. **ESTRICTAMENTE PROHIBIDO implementar Firebase Analytics** u otras herramientas de telemetría. El sistema debe ser ligero y centrado exclusivamente en la funcionalidad core.
3. **NO ESCRIBAS CÓDIGO TODAVÍA.** En esta etapa, requiero exclusivamente un documento de diseño arquitectónico y planificación en formato Markdown.

**Entregables Requeridos:**
Genera el plan de implementación estructurado, abordando los siguientes puntos:

1. **Arquitectura del Proyecto y Entorno:**
* Estructura de carpetas escalable bajo el enfoque *Feature-First* en `lib/`.
* Gestión de dependencias en `pubspec.yaml` (UI, Firebase, Provider, enrutamiento, utilidades).


2. **Estrategia de Datos (Firestore NoSQL):**
* Basándote en el script SQL, diseña las colecciones y subcolecciones clave en NoSQL. Optimiza para evitar lecturas innecesarias (desnormalización de datos embebidos para Usuarios, Publicaciones, Órdenes y Chats).


3. **Hoja de Ruta de Implementación Completa (End-to-End):**
Desglosa el desarrollo en las siguientes fases lógicas y secuenciales, detallando qué se hace en cada una:
* **Fase 1: Setup y Configuración Inicial:** (Firebase CLI, VS Code, inicialización).
* **Fase 2: Arquitectura Base y Modelos:** (Estructura de carpetas, clases de modelos en Dart con mapeo JSON).
* **Fase 3: Autenticación y Gestión de Estado:** (Login/Signup, AuthProvider).
* **Fase 4: Perfiles y Roles:** (Lógica para separar compradores de vendedores y sus datos).
* **Fase 5: Core del Marketplace (Vendedores):** (Creación de publicaciones, carga de imágenes a Storage).
* **Fase 6: Core del Marketplace (Compradores):** (Feed principal, búsqueda, filtrado de categorías y detalle de producto).
* **Fase 7: Transacciones:** (Carrito de compras, flujo de checkout y generación de órdenes).
* **Fase 8: Comunicación:** (Chat en tiempo real entre comprador y vendedor usando streams).
* **Fase 9: Testing y QA:** (Pruebas unitarias de los providers, pruebas de widgets y corrección de bugs del MVP).
* **Fase 10: Preparación y Despliegue:** (Configuración de íconos, splash screens, compilación de bundles/APKs y despliegue a App Store, Google Play y Web).



**Contexto de Datos (Esquema SQL de Referencia):**
A continuación, te proporciono el script DDL original que dicta las reglas de negocio que debemos replicar conceptualmente:

```sql
-- =============================================================
--  MySaveCar — Script DDL completo (PostgreSQL)
--  Versión 1.0
--  Descripción: App móvil de venta de autos, insumos y refacciones
-- =============================================================

-- Extensiones necesarias
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- [NOTA PARA LA IA: Asume que aquí va todo el script SQL proporcionado anteriormente, con los DOMINIOS 1 al 7: Usuarios, Catálogo, Transacciones, Pagos, Logística, Comunicación y Operaciones]

-- =============================================================
-- FIN DEL SCRIPT
-- =============================================================

```

Espero el plan maestro detallado.

---

¿Qué te parece esta estructura? Ya abarca desde el "Hola Mundo" hasta el momento en que subes la app a las tiendas. En cuanto estés listo para comenzar con la primera fase de código, solo tienes que decírmelo.
