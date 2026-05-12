# 📋 Plan de Implementación: `MySaveCar`
> Aplicación multiplataforma (Flutter/Dart + Firebase) para venta de vehículos, insumos y refacciones automotrices.

---

## 🛠 Herramientas Requeridas

| Categoría | Herramienta | Propósito |
|-----------|-------------|-----------|
| **IDE Principal** | VS Code | Desarrollo Flutter/Dart, terminal integrada, debugging |
| **Emuladores/Dispositivos** | Android Studio (opcional) + Xcode (macOS) | Emuladores Android/iOS, gestión de SDKs |
| **Diseño UI/UX** | Figma / Adobe XD | Wireframes, prototipos interactivos, Design System |
| **Backend & Servicios** | Firebase Console + Firebase CLI | Auth, Firestore, Storage, Analytics, Crashlytics |
| **Control de Versiones** | Git + GitHub/GitLab | Historial, colaboración, CI/CD |
| **Diagnóstico** | Flutter DevTools | Profiling, inspector de widgets, red, memoria |
| **Validación de Entorno** | `flutter doctor` | Verificar dependencias del SDK, licencias, emuladores |

> 📌 *Nota sobre "Antigravity"*: No existe un IDE o herramienta oficial con ese nombre en el ecosistema Flutter. Se recomienda VS Code como editor principal y Android Studio/Xcode exclusivamente para emuladores y compilación nativa.

---

## 📦 Dependencias Planificadas (`pubspec.yaml`)

| Categoría | Paquetes | Uso |
|-----------|----------|-----|
| **Firebase** | `firebase_core`, `firebase_auth`, `cloud_firestore`, `firebase_storage`, `firebase_messaging` | Backend, autenticación, BD, archivos, notificaciones |
| **Estado** | `provider` | Gestión de estado reactivo y arquitectura limpia |
| **Navegación** | `go_router` o `auto_route` | Routing declarativo, deep links, guards |
| **UI/Assets** | `flutter_svg`, `cached_network_image`, `image_picker`, `flutter_staggered_grid_view` | Iconos, imágenes optimizadas, galerías, selección de archivos |
| **Utilidades** | `intl`, `flutter_dotenv`, `flutter_secure_storage`, `uuid` | Formato moneda/fecha, variables de entorno, tokens seguros, IDs |
| **Desarrollo** | `flutter_test`, `mockito`, `lints` | Testing, mocking, análisis estático |

> 🔍 Las versiones se fijarán en el momento de la creación del proyecto usando `flutter pub add <paquete>` para garantizar compatibilidad con el canal estable actual de Flutter.

---

## 🧱 Arquitectura & Estructura del Proyecto

Se aplicará **Arquitectura por Features + Patrón Repository**:
```
lib/
├── core/          # Constantes, temas, utilidades, routing, di
├── features/
│   ├── auth/      # UI, provider, repository, models
│   ├── catalog/   # Listado, filtros, detalle de productos
│   ├── cart/      # Carrito, favoritos, checkout
│   ├── profile/   # Datos usuario, historial, configuración
│   └── admin/     # (Opcional) Gestión de inventario, reportes
├── data/          # Repositorios Firebase, DTOs, servicios
├── domain/        # Entidades, casos de uso, interfaces
└── main.dart      # Entry point, MultiProvider, Firebase init
```

---

## 🚀 Plan de Implementación Paso a Paso

### 🔹 Fase 1: Configuración del Entorno
1. Instalar Flutter SDK estable, Dart, Git y VS Code.
2. Instalar extensiones: `Flutter`, `Dart`, `Error Lens`, `Pubspec Assist`, `Firebase`.
3. Ejecutar `flutter doctor` y resolver advertencias (licencias Android, emuladores, herramientas nativas).
4. Crear proyecto: `flutter create my_save_car --platforms android,ios,web`.
5. Inicializar repositorio Git, configurar `.gitignore` estándar para Flutter.
6. Crear proyecto en Firebase Console, habilitar Auth (Email/Password), Firestore y Storage.
7. Instalar `firebase-tools`, ejecutar `firebase login`, `firebase init`, descargar archivos de configuración (`google-services.json` / `GoogleService-Info.plist`).

### 🔹 Fase 2: Diseño UI/UX
1. Definir personas y flujos: comprador, vendedor, invitado, administrador.
2. Crear wireframes de baja fidelidad en Figma: Login, Registro, Home, Catálogo, Detalle, Carrito/Favoritos, Perfil, Checkout/Contacto.
3. Establecer Design System: paleta de colores, tipografía, espaciado, componentes reutilizables (cards, inputs, botones, loaders, snackbar).
4. Exportar assets optimizados (SVG para iconos, WebP para imágenes, placeholders).
5. Validar prototipo interactivo con 3-5 usuarios objetivo; iterar según feedback.
6. Documentar decisiones de UX (jerarquía visual, estados de carga/vacío/error, accesibilidad).

### 🔹 Fase 3: Integración de Firebase & Autenticación
1. Configurar `firebase_core` e inicializar en `main.dart` (antes de `runApp`).
2. Habilitar método Email/Password en Firebase Auth Console.
3. Implementar flujo completo: Registro, Inicio de Sesión, Cierre, Recuperación de Contraseña, Verificación de Email.
4. Crear validaciones de formulario en UI y mapeo de errores de Firebase a mensajes legibles.
5. Configurar persistencia de sesión (manejo automático de `FirebaseAuth.instance.authStateChanges`).
6. Definir reglas de seguridad básicas en Firestore y Storage (solo lectura pública, escritura autenticada).
7. Implementar logging de eventos de Auth para diagnóstico.

### 🔹 Fase 4: Gestión de Estado con Provider
1. Crear `ChangeNotifier` por dominio: `AuthProvider`, `ProductProvider`, `CartProvider`, `UserProfileProvider`.
2. Implementar patrón Repository para aislar llamadas a Firebase (interfaz `IProductRepository`, implementación `FirebaseProductRepository`).
3. Configurar `MultiProvider` en `main.dart` con inyección de dependencias centralizada.
4. Definir estados UI: `loading`, `success`, `error`, `empty`, `noInternet`.
5. Conectar vistas con `Provider.of`, `context.watch`, `Consumer`; evitar rebuilds innecesarios.
6. Implementar manejo de errores global (snackbar/dialog) y reintentos automáticos cuando aplique.

### 🔹 Fase 5: Desarrollo de Funcionalidades Core
1. **Modelado de Datos Firestore**:
   - Colecciones: `users`, `products` (subcolecciones `specs`, `images`), `categories`, `orders`, `inquiries`.
   - Definir índices compuestos para filtros (marca + tipo + precio).
2. **Catálogo & Búsqueda**:
   - Listado paginado (`limit` + `startAfter`).
   - Filtros por categoría, precio, marca, año, condición (nuevo/usado).
   - Barra de búsqueda con debounce.
3. **Detalle de Producto**:
   - Galería de imágenes, ficha técnica, ubicación, contacto del vendedor.
   - Botones: "Me interesa" (inicia chat/email), "Agregar a favoritos".
4. **Carrito / Pedidos / Insumos & Refacciones**:
   - Lógica local con sincronización a Firestore.
   - Gestión de stock básico (marcar como reservado/vendido).
5. **Perfil de Usuario**:
   - Editar datos, historial de publicaciones/compras, gestión de favoritos.
   - Subida de imágenes a Firebase Storage con compresión y progreso.
6. **Rutas & Navegación**:
   - Implementar `go_router` con guards (rutas protegidas, redirección post-login).
   - Manejo de deep links para compartir productos.

### 🔹 Fase 6: Pruebas & Optimización
1. **Unit Testing**: Lógica de repositorios, validaciones, transformadores de modelos.
2. **Widget Testing**: Formularios, listas, estados de UI, navegación básica.
3. **Integration Testing**: Flujo completo (login → buscar → contactar → cerrar sesión).
4. **Performance**:
   - Paginación real en Firestore.
   - `cached_network_image` con política de cache.
   - `ListView.builder` / `GridView.builder` para listas largas.
   - Minimizar `setState` y rebuilds con `Provider` selectivo.
5. **Profiling**: Usar Flutter DevTools para analizar frames, memoria, red y CPU.
6. **Accesibilidad & Responsive**: Escalado de texto, contraste, soporte a tablets y web.

### 🔹 Fase 7: Despliegue & Mantenimiento
1. Configurar builds: `flutter build apk --release`, `flutter build ios --release`, `flutter build web`.
2. Firmar aplicaciones, configurar keystores, perfiles de provisionamiento.
3. Publicar en Google Play Console y App Store Connect (metadatos, capturas, política de privacidad).
4. Implementar CI/CD básico (GitHub Actions o Codemagic) para builds automáticos.
5. Activar Firebase Crashlytics y Analytics para monitoreo post-lanzamiento.
6. Documentar reglas de seguridad de Firestore/Storage en repositorio.
7. Planificar ciclo de actualizaciones: parches de seguridad, nuevas categorías, optimizaciones.

---

## 🔐 Consideraciones de Seguridad & Buenas Prácticas
- ✅ **Validación dual**: Cliente (UI) + Servidor (Firestore Security Rules).
- ✅ **No hardcodear** claves, IDs o URLs sensibles; usar `flutter_dotenv` o configuración segura.
- ✅ **Reglas de Firestore**: Restringir escritura/lectura por `request.auth.uid != null` y validar `resource.data`.
- ✅ **Manejo de errores**: Centralizado, sin exponer stack traces en producción.
- ✅ **Privacidad**: Política de datos, consentimiento para notificaciones, opción de eliminar cuenta.
- ✅ **Código limpio**: `flutter analyze`, lints personalizados, commits semánticos, PR reviews.

---

## ✅ Entregables por Fase & Siguientes Pasos

| Fase | Entregable | Validación |
|------|------------|------------|
| 1 | Proyecto Flutter + Firebase configurado | `flutter run` sin errores, Auth habilitado |
| 2 | Prototipo Figma + Design System | Aprobación de flujos y componentes |
| 3 | Auth funcional + reglas básicas | Login/registro/error handling verificados |
| 4 | Provider estructurado + repositorios | Estados UI sincronizados, sin memory leaks |
| 5 | Catálogo, detalle, perfil, carrito | CRUD Firestore, filtros, paginación |
| 6 | Suite de pruebas + optimización | >80% cobertura, 60fps estables |
| 7 | Builds firmados + stores + monitoreo | App publicada, Crashlytics activo |

---

📌 **Próximo paso**: Una vez revisado y aprobado este plan, procederé a generar la estructura de carpetas, el `pubspec.yaml` base, la configuración de Firebase y los primeros componentes de UI/estado según tu feedback. ¿Deseas ajustar algún flujo, agregar módulos (ej. chat interno, pasarela de pagos, geolocalización) o priorizar una fase específica?
