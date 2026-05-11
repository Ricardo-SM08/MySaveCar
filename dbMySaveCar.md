Aquí tienes el modelo completo con todas las tablas, atributos y tipos de datos, organizado por dominio para que puedas navegar fácilmente.El esquema completo cubre **34 tablas** distribuidas en 7 dominios. Algunos puntos clave de diseño que vale la pena destacar:

<img width="703" height="626" alt="image" src="https://github.com/user-attachments/assets/cce9bca3-80d3-4f9b-a26c-e15a509e1aec" />
<img width="720" height="680" alt="image" src="https://github.com/user-attachments/assets/eb89dbfc-e980-45b3-8097-6c3914eb51e4" />
<img width="583" height="669" alt="image" src="https://github.com/user-attachments/assets/3b08f83c-6426-4c3f-a23b-9cf078ac8aff" />
<img width="565" height="527" alt="image" src="https://github.com/user-attachments/assets/2d968254-11a5-4dfc-abbf-cbb7ac6a04fe" />
<img width="641" height="469" alt="image" src="https://github.com/user-attachments/assets/7107899b-b255-4935-bf88-7060c13e304a" />
<img width="587" height="462" alt="image" src="https://github.com/user-attachments/assets/449f89d9-d908-4de5-8fd0-3b0a52f294f9" />
<img width="591" height="438" alt="image" src="https://github.com/user-attachments/assets/e752320d-1715-4f93-8793-10405daaec61" />






**Sobre los tipos de dato elegidos:**
- Se usa `UUID` como PK en todas las tablas principales — ideal para apps móviles que necesitan generar IDs offline antes de sincronizar con el servidor.
- `JSONB` (PostgreSQL) aparece en `dispositivos_sesiones`, `inventario`, `productos` y `auditoria` para campos flexibles o variables (specs del dispositivo, dimensiones, permisos, diffs de auditoría).
- `DECIMAL(12,2)` para todos los montos monetarios — nunca `FLOAT` para dinero.
- `ENUM` para estados y tipos controlados — reduce errores de integridad sin necesitar tabla de catálogo.
- `BIGSERIAL` en `auditoria` y `eventos_analitica` porque son tablas append-only de alto volumen donde UUID sería más costoso.
- `INET` en `auditoria` y `dispositivos_sesiones` para almacenar IPs correctamente (IPv4 e IPv6).

**Notas importantes de integridad:**
- `publicaciones` tiene dos FK opcionales (`vehiculo_id` o `producto_id`) — solo una aplica según el tipo de listing.
- `compatibilidades` actúa como tabla puente entre refacciones y modelos de vehículo sin FK rígida al catálogo de vehículos, para permitir combinaciones no registradas.
- `auditoria` usa `datos_antes` / `datos_despues` en JSONB para un log completo de cambios sin necesitar una tabla por entidad.

¿Quieres que genere el DDL completo en SQL, los índices recomendados, o el diagrama ERD con las relaciones y cardinalidades?
