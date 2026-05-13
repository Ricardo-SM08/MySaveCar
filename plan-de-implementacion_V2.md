# 📘 Plan Maestro de Arquitectura e Implementación: MySaveCar
**Rol:** Tech Lead & Arquitecto de Software Senior  
**Stack:** Flutter (Dart) + Firebase (Auth, Firestore, Storage) + Provider  
**Package:** `com.example.myselftcar` | **Firebase Project:** `dbcrudmyselftcar`  
**Modo MVP:** Firestore Test Mode habilitado | Telemetría/Analytics explícitamente excluida | Sin código de UI en esta fase.

---

## 🏗️ 1. Arquitectura del Proyecto y Gestión de Assets

### 📂 Estructura de Carpetas (`lib/` + raíz)
Se adopta **Feature-First con separación de capas internas (Data/Domain/Presentation)**, optimizada para `Provider`. Cada feature es autónoma, escalable y testable.

```text
myselftcar/
├── assets/
│   ├── images/           # Logos, fondos, ilustraciones, placeholders, iconos rasterizados
│   ├── fonts/            # Familias tipográficas (.ttf/.otf) con pesos declarados
│   ├── icons/            # Iconografía vectorial (.svg)
│   └── config/           # JSON estáticos (categorías seed, feature flags, términos legales)
├── lib/
│   ├── core/
│   │   ├── constants/    # Rutas, claves de config, límites de UI, mensajes estáticos
│   │   ├── theme/        # Definición de tokens (Dark/Purple), escalas de espaciado
│   │   ├── utils/        # Formateadores, validadores, extensiones Dart, helpers de red
│   │   └── errors/       # Excepciones custom, Failure wrapper, manejo de fallos de red/BD
│   ├── features/
│   │   ├── auth/
│   │   ├── profile/
│   │   ├── marketplace/
│   │   ├── seller_center/
│   │   ├── transactions/
│   │   ├── communication/
│   │   └── logistics/
│   │   └── [feature]/
│   │       ├── data/           # datasources/, repositories/, models/dtos/
│   │       ├── domain/         # entities/, repositories_interfaces/, usecases/
│   │       └── presentation/   # providers/, screens/, widgets/
│   ├── shared/
│   │   ├── routing/        # go_router config, guards, redirects
│   │   ├── widgets/        # UI reutilizable (loaders, dialogs, empty states)
│   │   └── services/       # Utilidades transversales (logger, formatter, validator)
│   ├── firebase_options.dart   # Auto-generado por flutterfire configure
│   └── main.dart               # Entry point, inicialización, MultiProvider
├── test/                   # Estructura espejo de lib/ para pruebas unitarias/widget/integration
├── pubspec.yaml
└── analysis_options.yaml   # very_good_analysis + reglas personalizadas
```

---

## 📦 2. Código Completo de `pubspec.yaml`

```yaml
name: myselftcar
description: Marketplace integral especializado en el sector automotriz.
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: '>=3.2.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter

  # Firebase (Core, Auth, Firestore, Storage) - Sin Analytics/Telemetría
  firebase_core: ^3.6.0
  firebase_auth: ^5.3.1
  cloud_firestore: ^5.4.4
  firebase_storage: ^12.3.2

  # Gestión de Estado
  provider: ^6.1.2
  equatable: ^2.0.5

  # Navegación y Ruteo
  go_router: ^14.2.3

  # Modelado y Serialización Segura
  freezed_annotation: ^2.4.4
  json_annotation: ^4.9.0

  # UI, Imágenes y Assets
  cached_network_image: ^3.4.1
  flutter_svg: ^2.0.10+1
  image_picker: ^1.1.2
  path: ^1.9.0
  path_provider: ^2.1.4

  # Utilidades Core
  intl: ^0.19.0
  uuid: ^4.4.2
  collection: ^1.18.0
  shared_preferences: ^2.3.2
  connectivity_plus: ^6.0.5

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^4.0.0
  very_good_analysis: ^6.0.0
  build_runner: ^2.4.12
  freezed: ^2.5.7
  json_serializable: ^6.8.0
  mocktail: ^1.0.4

flutter:
  uses-material-design: true

  assets:
    - assets/images/
    - assets/fonts/
    - assets/icons/
    - assets/config/

  fonts:
    - family: Inter
      fonts:
        - asset: assets/fonts/Inter-Regular.ttf
        - asset: assets/fonts/Inter-Medium.ttf
          weight: 500
        - asset: assets/fonts/Inter-SemiBold.ttf
          weight: 600
        - asset: assets/fonts/Inter-Bold.ttf
          weight: 700
```

---

## 🔄 3. Gestión de Estado (Estrategia de Providers)

Se implementará un esquema **Multi-Provider jerárquico**. Los proveedores globales se inyectan en `main.dart`, mientras que los scoped se inyectan a nivel de `GoRoute` o feature para minimizar rebuilds.

| Provider | Scope | Responsabilidad | Estado Expuesto |
|----------|-------|-----------------|-----------------|
| `AuthProvider` | Global | Sesión, autenticación, roles, tokens JWT/Firebase | `User?`, `AuthStatus`, `rol`, `isLoading`, `error` |
| `UserProfileProvider` | Global | Datos personales, direcciones, configuración, KYC | `ProfileEntity`, `AddressList`, `loadingState` |
| `SellerProvider` | Scoped (`/seller/...`) | CRUD de publicaciones, métricas de ventas, gestión de stock | `List<ListingEntity>`, `PublishingStatus`, `metrics` |
| `MarketplaceProvider` | Scoped (`/marketplace/...`) | Feed, búsqueda, filtros, paginación, detalle de listing | `List<ListingEntity>`, `FilterState`, `hasMore`, `loading` |
| `CartProvider` | Global | Items del carrito, cálculo de subtotales, sync con Firestore | `List<CartItem>`, `subtotal`, `syncState`, `itemCount` |
| `OrderProvider` | Scoped (`/orders/...`) | Checkout, creación de órdenes, historial, tracking | `OrderEntity?`, `OrderList`, `checkoutStep`, `status` |
| `ChatProvider` | Scoped (`/chat/...`) | Hilos de conversación, mensajes en tiempo real, notificaciones | `Stream<List<Message>>`, `conversations`, `unreadCount` |
| `FavoritesProvider` | Scoped | Lista de publicaciones guardadas, toggle rápido | `Set<String>`, `isLoading`, `error` |

**Patrón de Implementación:**
- Cada `Provider` extiende `ChangeNotifier`.
- Estados se modelan con clases inmutables (`freezed`) o `Enum` (`Loading`, `Data`, `Error`, `Empty`).
- Las acciones de negocio llaman a `UseCase` → `Repository` → `Firestore/Storage`. El resultado actualiza el estado y notifica `notifyListeners()`.
- Se usa `Selector<Provider, T>` en UI para escuchar solo propiedades específicas y evitar rebuilds masivos.

---

## 🗃️ 4. Estrategia de Datos Exhaustiva (Firestore NoSQL)

Traducción fiel de los 7 dominios SQL a un modelo **denormalizado, orientado a lectura y sin JOINs**. Las relaciones se resuelven mediante:
1. **Embebido de snapshots** para datos de lectura frecuente.
2. **Referencias por `documentId`** para escritura única.
3. **Subcolecciones** para datos escalables o históricos.

### 🔹 Dominio 1: Usuarios y Perfiles
| SQL Table | Firestore Path | Estrategia NoSQL |
|-----------|----------------|------------------|
| `usuarios` | `users/{userId}` | Documento raíz. Contiene `email`, `rol`, `telefono`, `avatar_url`, `activo`, `verificado`, `creado_en`, `actualizado_en`. |
| `perfiles_vendedor` | `sellers/{userId}` | 1:1 con `users`. Campos: `tipo_negocio`, `nombre_negocio`, `rfc`, `descripcion`, `calificacion_avg`, `total_ventas`, `kyc_estado`. |
| `direcciones` | `users/{userId}/addresses/{addressId}` | Subcolección para escalabilidad. `alias`, `calle`, `colonia`, `ciudad`, `estado`, `cp`, `lat`, `lon`, `es_principal`. |
| `dispositivos_sesiones` | `users/{userId}/devices/{deviceId}` | Subcolección ligera. `push_token`, `plataforma`, `ultima_conexion`. No se almacena JWT (manejado por Auth SDK). |

### 🔹 Dominio 2: Catálogo de Productos
| SQL Table | Firestore Path | Estrategia NoSQL |
|-----------|----------------|------------------|
| `categorias` | `categories/{catId}` | Documento plano. `nombre`, `slug`, `padre_id`, `icono_url`, `orden`, `activa`. Se cachea en cliente. |
| `vehiculos` | `vehicles/{vehicleId}` | Catálogo maestro. Specs técnicos puros. No se modifica tras publicación. |
| `productos` | `products/{prodId}` | Catálogo maestro refacciones/insumos. `tipo`, `nombre`, `marca`, `numero_parte`, `es_oem`, `dimensiones`, `activo`. |
| `compatibilidades` | Embebido en `products/{prodId}` | Array de objetos: `[{marca, modelo, anio_desde, anio_hasta, motor}]`. Evita colección cruzada para lecturas rápidas. |
| `publicaciones` | `listings/{listingId}` | **Colección central**. Contiene `tipo` (`auto`/`producto`), `titulo`, `descripcion`, `precio`, `moneda`, `estado`, `visitas`, `publicado_en`, `expira_en`. **Desnormalización clave**: `seller_snapshot` (`nombre_negocio`, `calificacion`, `avatar`), `specs_snapshot` (marca, modelo, año o compatibilidades), `cover_url`. |
| `imagenes` | `listings/{listingId}/images/{imageId}` | Subcolección. `url`, `miniatura_url`, `tipo`, `orden`, `es_portada`. Límite 20 por listing. |

### 🔹 Dominio 3: Comercio y Transacciones
| SQL Table | Firestore Path | Estrategia NoSQL |
|-----------|----------------|------------------|
| `cupones` | `coupons/{code}` | Documento por código. `tipo_descuento`, `valor`, `max_descuento`, `min_compra`, `usos_actuales`, `activo`, `vigencia`. |
| `carritos` + `items_carrito` | `carts/{userId}` | Documento único por usuario. `items: [{listingId, titulo, precio_unitario, cantidad, sellerId}]`, `subtotal`, `actualizado_en`. Sync local ↔ Firestore. |
| `ordenes` | `orders/{orderId}` | Documento inmutable post-checkout. `compradorId`, `direccion_snapshot`, `cupon_aplicado`, `subtotal`, `descuento`, `envio`, `iva`, `total`, `estado`, `canal`, `notas`. |
| `items_orden` | Embebido en `orders/{orderId}` | Array `items: [{listingId, sellerId, titulo, cantidad, precio_unitario, estado_item}]`. Garantiza auditoría exacta. |
| `ofertas_negociacion` | `offers/{offerId}` | Colección raíz. `listingId`, `compradorId`, `monto_oferta`, `monto_contra`, `mensaje`, `estado`, `expira_en`. Indexada por `listingId`. |
| `favoritos` | `users/{userId}/favorites/{listingId}` | Subcolección tipo mapa/set. `guardado_en`. Lookup O(1). |

### 🔹 Dominio 4: Pagos y Finanzas
| SQL Table | Firestore Path | Estrategia NoSQL |
|-----------|----------------|------------------|
| `metodos_pago` | `users/{userId}/payment_methods/{methodId}` | Subcolección. `tipo`, `alias`, `ultimos_4`, `token_vault` (provider), `es_principal`. |
| `pagos` | `orders/{orderId}/payments/{paymentId}` | Subcolección. `monto`, `moneda`, `ref_externa`, `estado`, `gateway`, `respuesta_gateway`, `pagado_en`. |
| `reembolsos` | `orders/{orderId}/payments/{paymentId}/refunds/{refundId}` | Subcolección anidada. `monto`, `motivo`, `estado`, `ref_externa`. |
| `facturas` | `orders/{orderId}/invoices/{invoiceId}` | Subcolección. `rfc`, `razon_social`, `uso_cfdi`, `folio`, `xml_url`, `pdf_url`, `estado`. |
| `comisiones` | `commissions/{commissionId}` | Colección admin-only. `orderId`, `sellerId`, `porcentaje`, `monto_venta`, `monto_comision`, `estado`, `liquidada_en`. |

### 🔹 Dominio 5: Logística y Envíos
| SQL Table | Firestore Path | Estrategia NoSQL |
|-----------|----------------|------------------|
| `almacenes` | `sellers/{sellerId}/warehouses/{warehouseId}` | Subcolección. `nombre`, `direccion_snapshot`, `responsable`, `activo`. |
| `envios` | `orders/{orderId}/shipments/{shipmentId}` | Subcolección por item_orden. `almacenId`, `paqueteria`, `numero_guia`, `estado`, `peso`, `costo`, `entregado_en`. |
| `eventos_rastreo` | `envios/{shipmentId}/tracking/{eventId}` | Subcolección. `descripcion`, `ubicacion`, `codigo_estado`, `timestamp`. Paginada. |
| `inventario` | `sellers/{sellerId}/inventory/{inventoryId}` | Subcolección. `productoId`, `almacenId`, `stock_disponible`, `stock_reservado`, `umbral`. Actualizado vía transactions. |
| `citas_inspeccion` | `listings/{listingId}/inspections/{appointmentId}` | Subcolección. `compradorId`, `inspector`, `fecha`, `lugar`, `estado`, `resultado`, `reporte_url`. |

### 🔹 Dominio 6: Comunicación y Contenido
| SQL Table | Firestore Path | Estrategia NoSQL |
|-----------|----------------|------------------|
| `resenas` | `listings/{listingId}/reviews/{reviewId}` | Subcolección. `autorId`, `calificacion`, `comentario`, `visible`, `creado_en`. `sellerId` indexado para cálculo de promedio asíncrono. |
| `mensajes` | `chats/{chatId}/messages/{msgId}` | `chatId` = `hash(userIdA_userIdB_listingId)`. `chats/` almacena `participants`, `lastMessage`, `unreadCounts`. `messages/` es subcolección paginada (20 docs/req). |
| `notificaciones` | `users/{userId}/notifications/{notifId}` | Subcolección. `tipo`, `titulo`, `cuerpo`, `canal`, `leida`, `ref_id`, `creado_en`. TTL automático vía `expiresAt`. |
| `reportes_denuncias` | `reports/{reportId}` | Colección moderación. `reportadorId`, `ref_id`, `ref_type`, `motivo`, `descripcion`, `estado`. |

### 🔹 Dominio 7: Operaciones y Administración
| SQL Table | Firestore Path | Estrategia NoSQL |
|-----------|----------------|------------------|
| `roles_permisos` | `admin/roles/{roleId}` | Documento estático. `nombre`, `permisos` (mapa). Cacheado en cliente. |
| `auditoria` | `admin/audit_logs/{logId}` | Colección admin-only. `userId`, `tabla`, `registroId`, `accion`, `timestamp`. Escrito vía Cloud Functions (no cliente). |
| `busquedas_guardadas` | `users/{userId}/saved_searches/{searchId}` | Subcolección. `nombre`, `filtros` (mapa), `alerta_activa`, `frecuencia`. |
| `config_plataforma` | `config/{featureFlag}` | Colección raíz. `valor`, `tipo`, `descripcion`. Lectura única al inicio de sesión. |
| `moderacion` | `admin/moderation/{modId}` | Vinculado a `reports/`. `revisorId`, `accion`, `notas`, `resuelto_en`. |

**🔑 Principio Relacional sin JOINs:**
- **Lectura:** `listings` embebe `seller_name`, `cover_url`, `specs_summary`. `orders` embebe `items` con precios congelados. Elimina lecturas cruzadas.
- **Escritura:** Actualizaciones atómicas vía `runTransaction` (ej: stock -1, cart clear, order create).
- **Índices:** Compuestos predefinidos para `(tipo, estado)`, `(precio, creado_en)`, `(sellerId, estado)`, `(chatId, enviado_en DESC)`.

---

## 🗺️ 5. Hoja de Ruta de Implementación (End-to-End)

✅ **Estado:** DocumAquí tienes el **Plan Maestro de Implementación completo (Fases 1 a 10)**, estructurado con precisión arquitectónica, mapeo exhaustivo del DDL a Firestore, y formato optimizado para infografía o documentación técnica. **Sin código de UI**, enfocado 100% en arquitectura, datos, providers y lógica de negocio.

---

## 🔹 FASE 1: Setup y Configuración Inicial
**🎯 Objetivo:** Cimentar el entorno, vincular Firebase y activar estándares de calidad.
| 📂 Archivos & Lógica Clave | 🗃️ Mapeo Firestore (Preparación) | ✅ Entregables | 🎨 Sugerencia Visual Infografía |
|:---|:---|:---|:---|
| `pubspec.yaml` (dependencias base)<br>`firebase_options.dart` (auto-generado)<br>`analysis_options.yaml` (`very_good_analysis`)<br>`firestore.rules` (modo test) | Activación de `dbcrudmyselftcar`<br>Reglas temporales: `allow read, write: if request.time < timestamp.date(2026, 1, 1);`<br>Storage bucket configurado | Proyecto compilando en 3 plataformas<br>Firebase vinculado<br>CI básico activo (lint + format) | 🛠️ Terminal + 🔥 Firebase + 🛡️ Escudo temporal (Test Mode) |

---

## 🔹 FASE 2: Arquitectura Base y Tema Visual
**🎯 Objetivo:** Definir estructura escalable, tokens de diseño y assets.
| 📂 Archivos & Lógica Clave | 🗃️ Mapeo Firestore (Preparación) | ✅ Entregables | 🎨 Sugerencia Visual Infografía |
|:---|:---|:---|:---|
| `lib/core/theme/app_theme.dart`<br>`lib/core/constants/`<br>`assets/images|fonts|icons|config/`<br>`shared/widgets/` (skeletons, dialogs) | Estructura de colecciones definida en `firestore.indexes.json`<br>Configuración de TTL para datos expirables | Sistema de diseño Dark/Purple activo<br>Árbol de carpetas validado<br>Assets organizados y referenciados | 🎨 Swatches `#0A0A0C`, `#121214`, `#7E3AF2` + 🌳 Árbol de carpetas limpio |

---

## 🔹 FASE 3: Core, Modelos y MultiProvider
**🎯 Objetivo:** Implementar capa de datos tipada, inyección y routing.
| 📂 Archivos & Lógica Clave | 🗃️ Mapeo Firestore (Core) | ✅ Entregables | 🎨 Sugerencia Visual Infografía |
|:---|:---|:---|:---|
| `features/*/data/models/` (`freezed` + `json`)<br>`features/*/domain/repositories/` (interfaces)<br>`main.dart` (`MultiProvider` global)<br>`shared/routing/` (`go_router`) | `config_plataforma/{clave}` → Feature flags<br>`roles_permisos/{id}` → RBAC cacheado | Modelos inmutables generados<br>DI configurado<br>Rutas declarativas con guards | 🧩 Bloques `Data → Domain → Presentation` + 🔌 `MultiProvider` inyectado |

---

## 🔹 FASE 4: Autenticación y Gestión de Sesión
**🎯 Objetivo:** Flujos de acceso, persistencia y redirección por estado.
| 📂 Archivos & Lógica Clave | 🗃️ Mapeo Firestore (Dominio 1) | ✅ Entregables | 🎨 Sugerencia Visual Infografía |
|:---|:---|:---|:---|
| `auth/datasources/auth_remote.dart`<br>`auth/repository.dart`<br>`auth/provider.dart` (`AuthStatus` enum)<br>Guards en `go_router` | `users/{userId}`<br>Campos: `email`, `rol`, `activo`, `verificado`, `avatar_url`, `creado_en`<br>`dispositivos_sesiones/{deviceId}` (push tokens) | Login/Signup/Logout funcionales<br>Sesión persistente<br>Routing protegido por rol | 🔐 Candado + 👤 Usuario + 🔄 Stream `userChanges()` |

---

## 🔹 FASE 5: Perfiles, Roles y Direcciones
**🎯 Objetivo:** Segmentación comprador/vendedor/agencia y gestión de ubicaciones.
| 📂 Archivos & Lógica Clave | 🗃️ Mapeo Firestore (Dominio 1) | ✅ Entregables | 🎨 Sugerencia Visual Infografía |
|:---|:---|:---|:---|
| `profile/provider.dart`<br>`seller_center/onboarding/`<br>Lógica de `kyc_estado` | `sellers/{userId}`<br>Campos: `tipo`, `nombre_negocio`, `rfc`, `calificacion_avg`, `kyc_estado`<br>`users/{userId}/addresses/{addressId}` | Perfiles editables<br>Onboarding vendedor<br>CRUD direcciones + `es_principal` | 🏷️ Tags `comprador`/`vendedor`/`agencia` + 📍 Mapa + 📄 KYC flow |

---

## 🔹 FASE 6: Marketplace (Vendedores)
**🎯 Objetivo:** CRUD de publicaciones, gestión de catálogo y upload multimedia.
| 📂 Archivos & Lógica Clave | 🗃️ Mapeo Firestore (Dominio 2) | ✅ Entregables | 🎨 Sugerencia Visual Infografía |
|:---|:---|:---|:---|
| `seller_center/listings/`<br>`storage_service.dart` (compresión + upload)<br>Validación de `publicacion_tipo` y `estado` | `listings/{listingId}`<br>Campos: `titulo`, `precio`, `tipo`, `estado`, `acepta_oferta`<br>Embebidos: `seller_snapshot`, `specs_snapshot`<br>`listings/{id}/images/` (subcolección) | Publicación creada/editada<br>Imágenes en Storage con URLs en Firestore<br>Estados: `borrador → activa → vendida` | 📝 Formulario + 🖼️ Imágenes + 📦 Catálogo `vehiculos`/`productos` |

---

## 🔹 FASE 7: Marketplace (Compradores)
**🎯 Objetivo:** Feed paginado, búsqueda, filtros y detalle con specs.
| 📂 Archivos & Lógica Clave | 🗃️ Mapeo Firestore (Dominio 2) | ✅ Entregables | 🎨 Sugerencia Visual Infografía |
|:---|:---|:---|:---|
| `marketplace/provider.dart` (paginación)<br>`marketplace/filters/`<br>`users/{userId}/favorites/{listingId}` | `listings` (lectura principal)<br>Índices compuestos: `(tipo, estado)`, `(precio, creado_en)`<br>`categories/` cacheado<br>`compatibilidades` embebido en specs | Feed infinito estable<br>Búsqueda + filtros avanzados<br>Detalle con galería y compatibilidades | 🔍 Lupa + 📶 Paginación `limit(20)` + ❤️ Favoritos |

---

## 🔹 FASE 8: Transacciones (Carrito, Checkout y Órdenes)
**🎯 Objetivo:** Gestión de carrito, flujo de pago simulado y creación atómica de órdenes.
| 📂 Archivos & Lógica Clave | 🗃️ Mapeo Firestore (Dominio 3) | ✅ Entregables | 🎨 Sugerencia Visual Infografía |
|:---|:---|:---|:---|
| `transactions/cart_provider.dart`<br>`transactions/checkout_usecase.dart`<br>Validación de stock y cupones | `carts/{userId}` → `items: [{listingId, precio_snapshot, cantidad}]`<br>`orders/{orderId}` → Inmutable post-creación<br>`items_orden` embebidos en `orders` | Carrito sincronizado<br>Checkout fluido<br>Órdenes con snapshot de precios y estados | 🛒 Carrito + 🧾 Checkout + 🔒 Transacción atómica (`runTransaction`) |

---

## 🔹 FASE 9: Comunicación (Chat y Ofertas)
**🎯 Objetivo:** Chat en tiempo real, sistema de negociación y gestión de conversaciones.
| 📂 Archivos & Lógica Clave | 🗃️ Mapeo Firestore (Dominios 3 & 6) | ✅ Entregables | 🎨 Sugerencia Visual Infografía |
|:---|:---|:---|:---|
| `communication/chat_provider.dart`<br>`offers/negotiation_usecase.dart`<br>Paginación inversa de mensajes | `chats/{chatId}` → `participants`, `lastMessage`, `unreadCounts`<br>`chats/{id}/messages/{msgId}` → Stream paginado<br>`offers/{offerId}` → `monto_oferta`, `estado`, `expira_en` | Chat bidireccional estable<br>Ofertas con ciclo de vida completo<br>Sincronización offline/online | 💬 Burbujas de chat + ⏳ Stream + 💰 Flujo oferta/counter |

---

## 🔹 FASE 10: Pulido, QA y Despliegue
**🎯 Objetivo:** Validación final, optimización de rendimiento, builds y preparación de tiendas.
| 📂 Archivos & Lógica Clave | 🗃️ Mapeo Firestore (Hardening) | ✅ Entregables | 🎨 Sugerencia Visual Infografía |
|:---|:---|:---|:---|
| `test/` (unitarias + widget)<br>`flutter_launcher_icons.yaml`<br>Profiling (memory, rebuilds)<br>Builds: `appbundle`, `ipa`, `web` | Reglas de producción (`auth.uid`, `resource.data`)<br>Índices compuestos validados<br>Eliminación de `test mode` | Cobertura >70%<br>Binarios firmados<br>Web desplegada<br>Stores en revisión | 🧪 Tests + 🚀 Build + 📱 Stores + 🛡️ Reglas Production-Ready |

---

## 📐 Resumen de Mapeo Exhaustivo SQL → Firestore (Para Referencia en Infografía)
| Dominio SQL | Tablas Clave | Colección Firestore | Estrategia NoSQL |
|:---|:---|:---|:---|
| **1. Usuarios** | `usuarios`, `perfiles_vendedor`, `direcciones`, `dispositivos_sesiones` | `users/{id}`, `sellers/{id}`, `users/{id}/addresses/`, `users/{id}/devices/` | 1:1 por usuario. Direcciones y dispositivos como subcolecciones escalables. |
| **2. Catálogo** | `categorias`, `vehiculos`, `productos`, `compatibilidades`, `publicaciones`, `imagenes` | `listings/{id}`, `categories/`, `listings/{id}/images/` | `listings` embebe `seller_snapshot` + `specs_snapshot`. `compatibilidades` como array JSON en producto. |
| **3. Comercio** | `cupones`, `carritos`, `items_carrito`, `ordenes`, `items_orden`, `ofertas_negociacion`, `favoritos` | `carts/{uid}`, `orders/{id}`, `offers/{id}`, `users/{uid}/favorites/` | Carrito y favoritos por usuario. `orders` inmutable con items embebidos y precios congelados. |
| **4. Pagos** | `metodos_pago`, `pagos`, `reembolsos`, `facturas`, `comisiones` | `users/{uid}/payment_methods/`, `orders/{id}/payments/` | Subcolecciones por orden/usuario. `comisiones` se calculan vía CF (no cliente). |
| **5. Logística** | `almacenes`, `envios`, `eventos_rastreo`, `inventario`, `citas_inspeccion` | `sellers/{uid}/warehouses/`, `orders/{id}/shipments/`, `envios/{id}/tracking/` | Tracking como subcolección paginada. Inventario se gestiona con `FieldValue.increment`. |
| **6. Comunicación** | `resenas`, `mensajes`, `notificaciones`, `reportes_denuncias` | `chats/{id}/messages/`, `users/{uid}/notifications/`, `reports/` | `chats` agrupa hilos. `messages` paginados. `notifications` con TTL. |
| **7. Admin** | `roles_permisos`, `auditoria`, `busquedas_guardadas`, `config_plataforma` | `config/`, `users/{uid}/saved_searches/`, `admin/` | `config` caché local. `auditoria` vía Cloud Functions. `busquedas` subcolección. |

---
**Próximo paso:** Validar estructura de carpetas, estrategia de denormalización y scope de providers. Una vez aprobado, se generará el código modular fase por fase comenzando con `lib/core/`, `pubspec.yaml` aplicado y configuración de emuladores Firebase.




----
----
----
# PROMPT
Rol y Contexto:
Actúa como un Tech Lead y Arquitecto de Software Senior experto en Flutter, Dart y ecosistemas de Firebase. Tu objetivo es diseñar un plan de implementación maestro, exhaustivo y paso a paso para una aplicación móvil multiplataforma llamada "MySaveCar".

Descripción del Proyecto:
"MySaveCar" es un marketplace integral especializado en el sector automotriz para la compra y venta de automóviles, insumos vehiculares y refacciones. La base de datos original fue diseñada en PostgreSQL (script adjunto al final) y migraremos su lógica estructural a Firestore.

Stack Tecnológico y Estándares de Diseño:

Framework: Flutter (Dart) multiplataforma (iOS, Android, Web).

Entorno: VS Code.

Backend/BaaS: Firebase (Authentication, Firestore, Storage).

Gestión de Estado: Provider (esquema multi-provider).

Package Name: com.example.myselftcar

Firebase Project ID: dbcrudmyselftcar

Diseño UI/UX: Tema oscuro (Dark Mode) utilizando una paleta de colores donde el color principal sea Morado (Purple) combinado con tonos oscuros (grises profundos, negros o azules noche) que contrasten de manera elegante y profesional.

Restricciones Técnicas Críticas (Etapa MVP):

Configurar Firebase Firestore en Modo de Prueba (Standard/Test Mode) para agilizar el desarrollo inicial.

ESTRICTAMENTE PROHIBIDO implementar Firebase Analytics u otras herramientas de telemetría.

NO ESCRIBAS CÓDIGO DE INTERFAZ (UI) TODAVÍA. Sin embargo, SÍ DEBES generar el código completo y listo para copiar/pegar del archivo pubspec.yaml.

Entregables Requeridos:
Genera el documento arquitectónico abordando los siguientes puntos con máxima especificidad:

Arquitectura del Proyecto (Carpetas y Assets):

Define una estructura de carpetas escalable (Feature-First o Clean Architecture adaptada) dentro de lib/.

Incluye explícitamente la creación de un directorio dedicado en la raíz para imágenes: assets/images.

Código Completo del pubspec.yaml:

Genera el código completo del archivo integrando dependencias actualizadas para: Firebase, Provider, enrutamiento, fuentes, diseño y la configuración de la carpeta de assets.

Gestión de Estado (Providers):

Lista y explica exactamente qué Providers se van a necesitar (ej. AuthProvider, CartProvider, ListingsProvider) y cómo se estructurarán en la raíz de la app.

Estrategia de Datos Exhaustiva (Firestore NoSQL):

Basándote en el script SQL, traduce TODAS Y CADA UNA de las tablas de los 7 dominios a una estructura óptima para NoSQL (Colecciones, Subcolecciones y Datos Embebidos). Explica cómo se relacionarán sin usar JOINs.

Hoja de Ruta de Implementación (End-to-End Detallado):
Desglosa el desarrollo paso a paso, especificando qué archivos y lógica se crean en cada etapa:

Fase 1: Setup y Configuración: (Creación de proyecto, VS Code, assets, y Firebase CLI).

Fase 2: Arquitectura y Tema: (Estructura de carpetas, configuración de la paleta morada/oscura en ThemeData, y pubspec.yaml).

Fase 3: Core y Providers: (Implementación de clases modelo y configuración del MultiProvider).

Fase 4: Autenticación: (Login/Signup con Firebase Auth y redirecciones).

Fase 5: Perfiles y Roles: (Lógica para separar compradores, vendedores y agencias).

Fase 6: Marketplace (Vendedores): (Flujo para subir publicaciones de autos/refacciones y fotos a Storage).

Fase 7: Marketplace (Compradores): (Feed, búsqueda, filtros y vista de detalles).

Fase 8: Transacciones: (Carrito de compras, flujo de pago simulado y registro de órdenes).

Fase 9: Comunicación: (Chat en tiempo real mediante Firestore).

Fase 10: Pulido y Despliegue: (Pruebas, iconos de app, splash screen y builds finales).

Contexto de Datos (Esquema SQL de Referencia):
A continuación, el script DDL original que debes traducir exhaustivamente a Firestore:

SQL
-- =============================================================
--  MySaveCar — Script DDL completo (PostgreSQL)
--  Versión 1.0
--  Descripción: App móvil de venta de autos, insumos y refacciones
-- =============================================================

-- Extensiones necesarias
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- [NOTA PARA LA IA: Asume que aquí va todo el script SQL proporcionado anteriormente]

-- =============================================================
-- FIN DEL SCRIPT
-- =============================================================
Espero el plan maestro detallado y el código del pubspec.yaml. y todaa la estructura de carpetas 
