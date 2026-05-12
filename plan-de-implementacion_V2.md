
# 📐 Plan Maestro de Implementación: MySaveCar
**Rol:** Tech Lead & Arquitecto de Software Senior  
**Stack:** Flutter + Dart + Firebase (Auth, Firestore, Storage) + Provider  
**Package ID:** `com.example.myselftcar`  
**Firebase Project ID:** `dbcrudmyselftcar`  
**Estado:** Fase de Arquitectura y Planificación (Sin código)

---

## 🛠️ 1. Herramientas y Entorno de Desarrollo

### 🎨 Diseño UI/UX y Gestión de Assets
- **Figma (Design System):** Crear un repositorio de componentes atómicos (botones, inputs, cards, modales, states) con tokens de diseño para garantizar consistencia visual y acelerar el handoff a Flutter.
- **Organización de Assets en Proyecto:**
  - `assets/images/` → Ilustraciones, fondos, logos (formato WebP/AVIF para compresión óptima).
  - `assets/icons/` → Iconografía SVG (escala sin pérdida de calidad, integrable con `flutter_svg`).
  - `assets/fonts/` → Familias tipográficas con todos los pesos declarados en `pubspec.yaml`.
  - `assets/lottie/` → Animaciones para onboarding, estados vacíos, éxito/error y skeletons avanzados.
  - `assets/config/` → JSON estáticos para términos legales, FAQs, configuración de pasarelas y metadata de la app.
- **Pipeline de Optimización:** Implementar `flutter_gen` para generación type-safe de rutas de assets. Configurar pre-commit hooks para compresión automática de imágenes y validación de nombres de archivos.

### 🧩 Extensiones Indispensables para VS Code
| Extensión | Propósito Arquitectónico |
|-----------|--------------------------|
| `Flutter` + `Dart` | Soporte oficial, hot reload, debugging, formateo, análisis estático |
| `Error Lens` | Visualización inline de errores y advertencias sin salir del editor |
| `Pubspec Assist` | Búsqueda y gestión de paquetes con validación de versiones compatibles |
| `Firebase Assistant` | Vinculación rápida del proyecto Firebase y despliegue de reglas/índices |
| `Go Router` | Extensiones de navegación declarativa y validación de rutas |
| `GitLens` | Trazabilidad de cambios, gestión de ramas, integración con PRs |
| `Better Comments` | Categorización visual de `TODO`, `FIXME`, `ARCHITECTURE`, `PERF` |
| `Build Runner` | Ejecución de codegen (`freezed`, `json_serializable`) desde el IDE |
| `REST Client` | Pruebas de endpoints externos, webhooks y funciones serverless |

---

## 📦 2. Gestión de Dependencias (`pubspec.yaml`)

| Categoría | Paquetes | Justificación Técnica |
|-----------|----------|------------------------|
| **Core & State** | `provider`, `equatable` | `provider` para inyección y gestión reactiva. `equatable` para comparación eficiente de estados y prevención de rebuilds innecesarios. |
| **Firebase** | `firebase_core`, `firebase_auth`, `cloud_firestore`, `firebase_storage`, `firebase_messaging`, `firebase_analytics`, `firebase_crashlytics` | Stack oficial. Cubre autenticación multi-método, base de datos en tiempo real, almacenamiento multimedia, push notifications, métricas y monitoreo de errores en producción. |
| **Navegación** | `go_router` | Navegación declarativa, soporte para deeplinks, rutas anidadas, guards de autenticación y manejo de estado de navegación. |
| **UI & UX** | `cached_network_image`, `flutter_svg`, `skeletonizer`, `shimmer`, `responsive_builder` | Rendimiento gráfico optimizado, placeholders progresivos, adaptación a breakpoints y manejo de estados de carga/error. |
| **Modelado & Serialización** | `freezed`, `json_serializable`, `json_annotation` | Generación compile-time de modelos inmutables, `copyWith`, factories y mapeo seguro Firestore ↔ Dart. Reduce boilerplate y errores de runtime. |
| **Utilidades & Cache** | `uuid`, `intl`, `logger`, `shared_preferences`, `connectivity_plus`, `device_info_plus` | IDs locales, formateo de monedas/fechas, logging estructurado, persistencia ligera, detección de red e identificación de dispositivo para analytics/fraud prevention. |
| **Dev & Calidad** | `build_runner`, `flutter_lints`, `very_good_analysis`, `mockito`, `test` | Estricto linting, codegen, pruebas unitarias/widget/integration y estándares de calidad tipo enterprise. |

*Nota:* Todas las versiones se fijarán mediante lockfile y se validarán en CI antes de cada merge. Se evitarán dependencias no mantenidas o con licencias restrictivas.

---

## 🗂️ 3. Arquitectura de Carpetas (Feature-First + Capas Limpias)

Se adopta un enfoque **Feature-First** con separación vertical por dominio, combinado con capas limpias (Data/Domain/Presentation) adaptado a `Provider`. Esto maximiza la cohesión, facilita el escalado paralelo de equipos y aísla la lógica de negocio de la UI.

```
lib/
├── core/
│   ├── constants/          # Rutas, claves, límites, configuraciones globales
│   ├── theme/              # ThemeData, paletas, tipografías, componentes base
│   ├── utils/              # Helpers, formatters, validadores, extensiones
│   └── errors/             # Excepciones personalizadas, manejo de fallos
├── features/
│   ├── auth/
│   ├── catalog/            # Vehículos, insumos, refacciones
│   ├── marketplace/        # Listados, búsqueda, filtros, favoritos
│   ├── orders/             # Carrito, checkout, historial de compras/ventas
│   ├── payments/           # Integraciones pasarelas, estados de pago
│   ├── logistics/          # Envíos, tracking, direcciones
│   ├── communication/      # Chat, notificaciones, soporte
│   └── profile/            # Gestión de cuenta, reputación, configuración
│       ├── data/
│       │   ├── datasources/ # Firestore/Storage clients
│       │   ├── repositories/ # Implementaciones de contratos
│       │   └── models/      # Clases serializables (DTOs ↔ Entities)
│       ├── domain/
│       │   ├── entities/    # Objetos de negocio puros
│       │   └── usecases/    # Casos de uso (reglas de negocio)
│       └── presentation/
│           ├── providers/   # ChangeNotifiers, scoped providers
│           ├── screens/     # Vistas principales
│           └── widgets/     # Componentes específicos del feature
├── shared/
│   ├── routing/            # go_router configuración y redirecciones
│   ├── services/           # Servicios transversales (analytics, crashlytics)
│   └── widgets/            # UI reutilizable entre features
├── firebase_options.dart   # Configuración multiplataforma generada
└── main.dart               # Entry point, inyección global de providers
```

**Flujo de Estado con Provider:**
- Los `ChangeNotifier` se declaran en `presentation/providers/`.
- Se inyectan con `MultiProvider` o `Provider.value` a nivel de `GoRoute` para limitar el scope y evitar rebuilds globales.
- La capa `data` expone streams o métodos asíncronos; `Provider` escucha y transforma a UI states (Loading, Success, Error).
- Se evita acoplar Firestore directamente a la UI; toda interacción pasa por `Repository → UseCase → Provider → Widget`.
- Se implementa patrón `StateClass` (ej. `LoadingState`, `DataState`, `ErrorState`) para tipado estricto y manejo centralizado de errores.

---

## 🗃️ 4. Estrategia de Datos y Migración a Firestore

La migración de un esquema relacional (PostgreSQL) a Firestore requiere **desnormalización controlada**, **referencias explícitas** y **agrupación por patrones de acceso**. Se priorizan lecturas rápidas, consistencia eventual donde aplique, y reducción de operaciones costosas.

### 📊 Estructura de Colecciones Principales
| Dominio Relacional | Colección Firestore | Estrategia NoSQL |
|-------------------|---------------------|------------------|
| **Usuarios** | `users/{userId}` | Documento raíz con perfil, roles, métricas (rating, ventas activas). Subcolecciones: `favorites/`, `addresses/`, `notifications/`. |
| **Catálogo (Vehículos/Repuestos/Insumos)** | `products/{productId}` | Polimorfismo controlado mediante campos `category`, `type`, `brand`, `specs` (mapa flexible). Imágenes referenciadas como array de URLs en Storage. Índices compuestos en `status`, `price`, `sellerId`, `category`. |
| **Publicaciones/Listados** | Integrado en `products` + `listings_metadata/{listingId}` | Se evita duplicación. `listings_metadata` almacena visibilidad, boosts, estadísticas, y expira automáticamente vía TTL indexado. |
| **Órdenes/Transacciones** | `orders/{orderId}` | Documento plano con `buyerId`, `sellerId`, `items` (array de snapshots), `total`, `status`, `timestamps`. Desnormalización de `sellerName`/`buyerAvatar` para evitar joins en listados. |
| **Pagos** | `payments/{paymentId}` | Colección raíz para auditoría. Vinculada vía `orderId`. Almacena `provider`, `transactionId`, `status`, `receiptUrl`. Lecturas aisladas para conciliación. |
| **Logística** | `shipments/{shipmentId}` | Vinculado a `orderId`. Campos: `carrier`, `trackingCode`, `status`, `estimatedDelivery`, `addressSnapshot`. Subcolección `tracking_events/` para historial. |
| **Comunicación** | `conversations/{conversationId}` → `messages/{messageId}` | Patrón chat optimizado: `conversations` guarda metadatos y `lastMessage`. `messages` es subcolección paginada (20 doc/req). Índices en `createdAt`, `readBy`. |
| **Operaciones/Admin** | `system_config/`, `reports/`, `audit_logs/` | Documentación de reglas de negocio, logs de moderación, métricas agregadas. Acceso restringido vía Firebase Custom Claims. |

### 📉 Optimización de Costos, Índices y Seguridad
- **Paginación estricta:** Uso de `limit()` + `startAfterDocument()` en listados. Evitar `get()` recursivos o `where` sin índices compuestos declarados.
- **Subcolecciones anidadas máximo 2 niveles:** `conversations/messages` o `orders/tracking` son aceptables; evitar 3+ niveles para no disparar costos de lectura.
- **Contadores distribuidos:** Para métricas altas (vistas, likes, ventas totales) usar el patrón de contadores distribuidos de Firestore o Cloud Functions para agregación asíncrona.
- **Caching inteligente:** `Provider` + `shared_preferences` para datos estáticos. Firestore offline cache activado por defecto en móvil. Sincronización en cola al recuperar conectividad.
- **Índices compuestos:** Definir en `firestore.indexes.json` antes de desarrollo para evitar `FAILED_PRECONDITION` en producción. Validar con emulador local.
- **Reglas de seguridad:** Estructura basada en `request.auth.uid`, `resource.data.sellerId == request.auth.uid`, y validación de tipos/tamaños. Validar uploads de Storage con MIME y tamaño máximo. Separar reglas por colección con `match` explícito.

---

## 🗺️ 5. Hoja de Ruta de Implementación (Fase 1 a 7 - Detallada)

### 🔹 FASE 1: Cimientos y Configuración del Entorno
- **Objetivo:** Inicializar proyecto multiplataforma, configurar estándares y conectar Firebase.
- **Actividades Clave:**
  - Crear proyecto Flutter con `flutter create --org com.example --project-name myselftcar`
  - Configurar `firebase_options.dart` para iOS, Android y Web
  - Implementar estructura de carpetas (Feature-First)
  - Configurar `very_good_analysis`, formateo automático y hooks de pre-commit
  - Definir `ThemeData`, tokens de diseño y sistema de tipografía
  - Configurar `go_router` base con rutas placeholder y guards vacíos
  - Configurar CI básico (lint, format, test dry-run)
- **Entregables:** App compilando sin warnings, Firebase inicializado, estructura de carpetas validada, tema global aplicado, pipeline CI activo.
- **Consideraciones Técnicas:** Evitar hardcodear IDs. Usar variables de entorno para configuración sensible. Validar que `flutter doctor` pase en todas las plataformas objetivo.
- **Criterios de Aceptación:** Compilación exitosa en iOS/Android/Web. Firebase console muestra dispositivos conectados. Linting y formateo pasan al 100%.

### 🔹 FASE 2: Autenticación, Perfiles y Seguridad
- **Objetivo:** Implementar flujo de acceso seguro, gestión de sesión y estructura de perfiles.
- **Actividades Clave:**
  - Configurar `firebase_auth` (Email/Password, Google, Apple)
  - Implementar `AuthProvider` con estados: `Unauthenticated`, `Loading`, `Authenticated`, `Error`
  - Crear pantallas de Login, Registro, Recuperación de contraseña y Onboarding
  - Implementar persistencia de sesión y manejo de refresh tokens
  - Diseñar estructura de perfil básico (nombre, avatar, rol, fecha registro)
  - Preparar borrador de reglas de Firestore para acceso por rol
  - Configurar `firebase_crashlytics` y `analytics` para tracking de flujos de auth
- **Entregables:** Flujo completo de autenticación funcional, proveedor de sesión reactivo, perfiles creados/leídos, reglas de seguridad base, monitoreo activo.
- **Consideraciones Técnicas:** Validar emails en cliente y servidor. Usar `custom claims` para roles (buyer, seller, admin). Evitar almacenar datos sensibles en `shared_preferences`.
- **Criterios de Aceptación:** Login/logout sin pérdida de estado. Recuperación de contraseña funcional. Sesión persistente tras reinicio. Reglas bloquean accesos anónimos a colecciones protegidas.

### 🔹 FASE 3: Arquitectura de Estado y UI Core
- **Objetivo:** Establecer patrones de estado responsivos, navegación declarativa y componentes reutilizables.
- **Actividades Clave:**
  - Implementar `MultiProvider` en `main.dart` con alcance global y scoped
  - Crear patrón `StateClass` (Loading, Success, Error, Empty) para todos los providers
  - Desarrollar componentes core: AppShell, BottomNavigation, SearchBar, FilterDrawer, Skeletons, ErrorBanners
  - Configurar rutas anidadas con `go_router` y guards de autenticación
  - Implementar manejo de errores globales (`FlutterError.onError`, `Zone` custom)
  - Optimizar rebuilds con `Consumer`, `Selector` y `context.watch/select`
- **Entregables:** Navegación fluida con deep linking, UI responsiva, manejo centralizado de errores, proveedores optimizados, componentes base documentados.
- **Consideraciones Técnicas:** Limitar scope de providers a features específicos. Usar `Provider.of(context, listen: false)` para acciones sin rebuild. Validar accesibilidad (contraste, tamaños de texto dinámicos).
- **Criterios de Aceptación:** Transiciones suaves sin jank. Rebuilds localizados. Navegación funciona con/without auth. App escala correctamente en tablets y web.

### 🔹 FASE 4: Catálogo, Marketplace y Búsqueda
- **Objetivo:** Integrar Firestore para listados, gestión de productos y búsqueda optimizada.
- **Actividades Clave:**
  - Modelar `ProductEntity` con `freezed` y mapeo seguro a Firestore
  - Implementar `ProductRepository` con métodos: `getPaginated`, `getById`, `create`, `update`, `delete`
  - Configurar paginación con `limit` + `startAfterDocument`
  - Desarrollar filtros avanzados (precio, categoría, ubicación, estado) usando índices compuestos
  - Integrar `firebase_storage` para subida/optimización de imágenes (comprimir antes de upload)
  - Implementar caché local de listados frecuentemente accedidos
  - Crear pantallas: Home, Category, ProductDetail, SellerProfile, Favorites
- **Entregables:** Marketplace funcional con paginación, filtros, subida de imágenes, detalle de producto, favoritos, caché offline.
- **Consideraciones Técnicas:** Validar tamaños de imagen en cliente. Usar `FieldValue.serverTimestamp()` para ordenamiento. Indexar campos de filtro antes de deploy. Evitar `arrayContains` con +10 elementos.
- **Criterios de Aceptación:** Listados cargan en <2s en 3G. Filtros responden sin timeout. Imágenes se comprimen y suben correctamente. Modo offline muestra caché y sincroniza al reconectar.

### 🔹 FASE 5: Transacciones, Órdenes y Pagos
- **Objetivo:** Implementar flujo de compra/venta, gestión de órdenes y conciliación de pagos.
- **Actividades Clave:**
  - Diseñar entidad `Order` con estado inmutable (`pending`, `paid`, `shipped`, `completed`, `cancelled`)
  - Implementar carrito temporal (local) y persistencia de checkout
  - Integrar pasarela de pago en modo sandbox (simulación vía Cloud Functions o SDK oficial)
  - Crear `PaymentRepository` para registro de transacciones y recibos
  - Implementar historial de órdenes para compradores y vendedores
  - Configurar reglas de transacción (idempotencia, prevención de doble cobro)
  - Desarrollar pantallas: Cart, Checkout, OrderConfirmation, OrderHistory, OrderDetail
- **Entregables:** Flujo de compra completo, registro de pagos, historial de órdenes, conciliación básica, UI de seguimiento.
- **Consideraciones Técnicas:** Usar `runTransaction` para actualizaciones atómicas de stock/pago. Validar precios en servidor antes de confirmar. Nunca confiar en precios calculados en cliente.
- **Criterios de Aceptación:** Órdenes se crean sin inconsistencias. Estados se actualizan correctamente. Historial refleja actividad real. Sandbox de pago procesa sin errores.

### 🔹 FASE 6: Logística, Comunicación y Notificaciones
- **Objetivo:** Implementar tracking de envíos, chat en tiempo real y sistema de notificaciones push.
- **Actividades Clave:**
  - Diseñar colección `shipments` con subcolección `tracking_events`
  - Implementar `ShipmentProvider` con actualización en tiempo real vía streams
  - Crear sistema de chat: `conversations` + `messages` (paginado, read receipts)
  - Configurar `firebase_messaging` para notificaciones push (FCM)
  - Implementar lógica de notificaciones: nuevos mensajes, actualizaciones de orden, promociones
  - Desarrollar pantallas: Tracking, ChatList, ChatRoom, NotificationCenter
  - Configurar manejo de notificaciones en background/terminated state
- **Entregables:** Tracking en tiempo real, chat funcional con historial, push notifications configuradas, centro de notificaciones, manejo offline de mensajes.
- **Consideraciones Técnicas:** Limitar carga de mensajes a 20 por request. Usar `FieldValue.increment` para contadores no leídos. Validar tokens FCM y rotación. Implementar retry logic para envíos fallidos.
- **Criterios de Aceptación:** Tracking se actualiza sin latencia perceptible. Chat sincroniza en tiempo real. Push se reciben en foreground/background. Mensajes no se pierden tras reconexión.

### 🔹 FASE 7: QA, Optimización, Reglas y Despliegue
- **Objetivo:** Validar calidad, optimizar rendimiento, endurecer seguridad y preparar lanzamiento.
- **Actividades Clave:**
  - Implementar pruebas unitarias (usecases, repositorios, mappers)
  - Implementar pruebas widget (UI states, navegación, proveedores)
  - Implementar pruebas de integración (flujos completos: auth → search → order → payment)
  - Ejecutar profiling (memory leaks, frame drops, rebuilds excesivos)
  - Auditar y endurecer reglas de Firestore y Storage
  - Configurar pipeline CI/CD completo (lint, test, build, deploy a stores)
  - Generar release notes, configuración de metadatos, firma de binarios
  - Configurar monitoreo en producción (Crashlytics, Performance, Analytics)
- **Entregables:** Cobertura de pruebas >70%, reglas de seguridad validadas, pipeline automatizado, binarios firmados, dashboard de monitoreo activo.
- **Consideraciones Técnicas:** Validar tamaños de APK/IPA. Usar `flutter build --release` con tree-shaking. Probar en dispositivos de baja gama. Revisar políticas de privacidad y términos legales antes del lanzamiento.
- **Criterios de Aceptación:** 0 crash reports en fase beta. Reglas bloquean accesos no autorizados. Build time <10 min. App pasa store review guidelines. Monitoreo reporta métricas en tiempo real.

---

## 📋 Próximos Pasos y Validación Arquitectónica

1. **Revisión del Plan:** Validar alcance, fases, dependencias y estrategia de datos con stakeholders.
2. **Configuración de Emuladores:** Montar entorno local de Firebase (Auth, Firestore, Storage) para desarrollo seguro y pruebas offline.
3. **Sprint 0 (Setup):** Implementar Fase 1 y 2 con documentación técnica interna y métricas de éxito definidas.
4. **Iteración por Fases:** Cada fase se entregará con pruebas, documentación y review de arquitectura antes de avanzar.
5. **Gestión de Deuda Técnica:** Registrar en backlog cualquier decisión temporal (hotfixs, workarounds) para refactor en fase 7.

✅ **Este documento está listo para ser utilizado como blueprint de desarrollo.** No contiene código ejecutable, cumple con el principio de "planificar antes de implementar", y establece bases escalables para un marketplace automotriz profesional.







---

PROMPT

Rol y Contexto:
Actúa como un Tech Lead y Arquitecto de Software Senior experto en Flutter, Dart y ecosistemas de Firebase. Tu objetivo es diseñar un plan de implementación maestro para una aplicación móvil multiplataforma llamada "MySaveCar".

Descripción del Proyecto:
"MySaveCar" es un marketplace integral (estilo MercadoLibre) especializado en el sector automotriz. Permitirá la compra y venta de automóviles, insumos vehiculares y refacciones. La base de datos ya está diseñada en PostgreSQL y migraremos su lógica estructural a Firestore.

Stack Tecnológico:

Framework: Flutter (Dart) multiplataforma (iOS, Android, Web).

Entorno: VS Code.

Backend/BaaS: Firebase (Autenticación, Firestore, Storage).

Gestor de Estado: Provider.

Package Name: com.example.myselftcar

Firebase Project ID: dbcrudmyselftcar

Restricción Importante:
NO ESCRIBAS CÓDIGO TODAVÍA. En esta etapa inicial, requiero exclusivamente un documento de diseño y planificación de alto nivel en formato Markdown, estructurado para un desarrollo escalable, mantenible y óptimo.

Entregables Requeridos:
Por favor, genera el plan de implementación abordando secuencialmente los siguientes puntos:

Herramientas y Entorno de Desarrollo:

Extensiones indispensables para VS Code que optimicen el desarrollo con Flutter y Firebase.

Gestión de Dependencias (pubspec.yaml):

Lista clasificada de las dependencias clave que necesitaremos (UI, State Management, Firebase, Routing, Utilidades), justificando brevemente su uso.

Arquitectura de Carpetas (Estructura del Proyecto):

Diseña un árbol de directorios escalable (recomendable enfoque Feature-First o Clean Architecture adaptado a Provider) que separe claramente UI, lógica de negocio, servicios, modelos y rutas.

Estrategia de Datos (Firestore):

Basándote en el script DDL de PostgreSQL adjunto al final, propón cómo estructurar las colecciones y subcolecciones principales en NoSQL (Firestore) para optimizar lecturas y costos, especialmente para Usuarios, Vehículos, Productos, Publicaciones y Órdenes.

Hoja de Ruta de Implementación (Step-by-Step):

Define las fases del desarrollo paso a paso (Fase 1: Setup y Configuración, Fase 2: Autenticación, Fase 3: Core y UI, Fase 4: Integración de Base de Datos, etc.) para tener un backlog claro.

Contexto de Datos (Esquema SQL de Referencia):
A continuación, te proporciono el script DDL original que dicta las reglas de negocio y entidades que debemos replicar conceptualmente en la app:

SQL
-- =============================================================
--  MySaveCar — Script DDL completo (PostgreSQL)
--  Versión 1.0
--  Descripción: App móvil de venta de autos, insumos y refacciones
-- =============================================================

-- Extensiones necesarias
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- [NOTA: Asume que aquí va todo el script SQL proporcionado anteriormente, con los DOMINIOS 1 al 7: Usuarios, Catálogo, Transacciones, Pagos, Logística, Comunicación y Operaciones]

-- =============================================================
-- FIN DEL SCRIPT
-- =============================================================
Espero el plan de implementación detallado en formato Markdown.


/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-/*-
