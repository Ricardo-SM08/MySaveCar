Como DBA del proyecto **MySaveCar**, aquí está el modelo completo de entidades que necesitas. Te lo presento en capas lógicas para que sea mantenible y escalable.Puedes hacer clic en cualquier entidad para que te la explique a detalle. Aquí el resumen de los 7 dominios:

<img width="750" height="578" alt="image" src="https://github.com/user-attachments/assets/867f079c-56ec-42b8-bd0b-79a3d92b5854" />
<img width="818" height="584" alt="image" src="https://github.com/user-attachments/assets/d2be65c3-1e3a-4b57-8460-19233779be9d" />
<img width="828" height="250" alt="image" src="https://github.com/user-attachments/assets/8d19b5e4-e6b2-4e66-a3b0-7adc97977324" />




**Usuarios y perfiles** — cubre la autenticación, los roles (comprador, vendedor particular, agencia, admin), y la gestión de dispositivos para notificaciones push. El vendedor tiene su propia entidad con datos de KYC y reputación.

**Catálogo de productos** — es el corazón del sistema. Se divide en tres tipos de ítem: vehículos (con VIN, historial, specs), insumos/accesorios (consumibles) y refacciones (con tabla de compatibilidad make/model/year). Las publicaciones son una capa encima de los productos que controla precio, visibilidad y estado.

**Comercio y transacciones** — gestiona el flujo completo desde carrito hasta orden. La entidad `Oferta/Negociación` es clave para la venta de autos (donde el precio no es fijo). Los cupones tienen sus propias reglas de negocio.

**Pagos y finanzas** — separado del comercio intencionalmente. Incluye `Factura/CFDI` para el mercado mexicano y `Comisión` para que la plataforma registre su corte por cada venta.

**Logística y envíos** — aplica principalmente a insumos y refacciones. Los autos tienen su propio flujo con `Cita de inspección` (necesaria antes de la transferencia).

**Comunicación y contenido** — el chat comprador-vendedor es fundamental para negociar autos. Las reseñas son bidireccionales (comprador califica vendedor y viceversa).

**Operaciones y administración** — sin esto no hay backoffice saludable. `Auditoría` registra cada cambio crítico. `Búsqueda guardada` permite alertas tipo "avísame cuando haya una Hilux 2020 por menos de $300k".

¿Quieres que profundice en algún dominio específico, te genere el DDL (SQL), o diseñemos las relaciones y cardinalidades entre entidades?
