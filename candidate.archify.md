# Diccionario de datos extendido: `RES 20.sql` {#diccionario-res20}

El respaldo describe una tienda en línea con **13 tablas**. Esta guía conserva el SQL original, traduce cada campo a lenguaje de negocio y separa lo confirmado por el script de las interpretaciones hechas por nombre, tipo y valores guardados.

- `tb_products` contiene 5,008 productos y es el catálogo operativo.
- `tbd_products_categories` contiene 4,993 asignaciones entre productos y categorías.
- `tb_logs` contiene 5,448 eventos de auditoría; no representa operaciones vigentes.
- Las demás tablas no tienen filas `INSERT` en este respaldo, por lo que su total actual es 0.

@answer scope="Respaldo RES 20.sql · MySQL 8 · conteos calculados desde INSERT"
@meta locale=es style=product quality_profile=showcase
@source dump title="Respaldo analizado" path="RES 20.sql"

## Cómo leer este diccionario {#lectura}

| Marca | Significado para negocio |
| --- | --- |
| 🔢 Número | Cantidad, importe o identificador numérico. |
| 📝 Texto | Nombre, código, detalle o referencia escrita. |
| 📅 Fecha | Momento en que ocurrió, se creó o se actualizó algo. |
| ✅ Sí/No | Indicador guardado como `bit(1)` o `tinyint(1)`. |
| 🔑 PK | Identificador único principal de la tabla. |
| 🔗 FK | Enlace hacia el registro de otra tabla. |
| ◌ Opcional | Puede quedar vacío (`NULL`). |

Los términos como “activo” para `status` y “proveedor” para `provider_ID` son inferencias documentadas; el SQL no incluye comentarios de negocio. `NOT NULL` se muestra como **Obligatorio**. Los conteos reflejan el archivo entregado, no una base de datos en ejecución.

## Mapa de relaciones, explicado sin jerga {#relaciones}

| Tabla origen y campo | Tabla destino y campo | Explicación amigable |
| --- | --- | --- |
| `tb_carts.user_ID` | `tb_users.ID` | Cada carrito pertenece a una persona. |
| `tb_buyer_orders.seller_ID` | `tb_users.ID` | Cada pedido queda asociado a una persona; el rol exacto debe validarse con negocio. |
| `tbd_products_users_providers.user_ID` | `tb_users.ID` | Una oferta identifica quién provee el producto. |
| `tbd_products_users_providers.product_ID` | `tb_products.ID` | Una oferta se refiere a un producto del catálogo. |
| `tbd_carts_items.cart_ID` | `tb_carts.ID` | Cada renglón forma parte de un carrito. |
| `tbd_order_items.seller_order_ID` | `tb_buyer_orders.ID` | Cada renglón forma parte de un pedido. |
| `tbd_user_payment_methods.user_ID` | `tb_users.ID` | Una persona puede registrar medios de pago. |
| `tbd_user_payment_methods.payment_method_type_ID` | `tbc_payment_methods_types.ID` | El medio registrado usa un tipo de pago definido. |
| `tbd_users_wishlist_products.user_ID` y `product_ID` | `tb_users.ID` y `tb_products.ID` | Cada favorito conecta una persona con un producto. |
| `tbd_products_categories.category_ID` | `tbc_categories.ID` | Una asignación coloca el producto en una categoría; su vínculo de producto es inferido. |

Una persona (`tb_users`) puede tener un carrito, medios de pago, favoritos, pedidos y productos que ofrece. Un producto puede estar en varias categorías y puede ser ofrecido por varios usuarios. El respaldo solo declara formalmente la llave externa de categoría en `tbd_products_categories`; `product_ID` se interpreta como vínculo a `tb_products.ID` por su nombre y propósito.

## `tb_users` — Registro de personas con acceso {#tb-users}

**Total actual: 7 registros.** Cada fila identifica una cuenta; correo y apodo no se pueden repetir.

| Columna exacta | Tipo SQL exacto | Lectura amigable | Atributos |
| --- | --- | --- | --- |
| `ID` | `int unsigned` | 🔢 Identificador interno de la persona. | 🔑 PK · Obligatorio · Autonumérico |
| `email` | `varchar(100)` | 📝 Correo usado para identificar o contactar la cuenta. | Único · Obligatorio |
| `nick` | `varchar(50)` | 📝 Nombre corto mostrado al usuario. | Único · Obligatorio |
| `password` | `varchar(255)` | 📝 Credencial de acceso; debe tratarse como dato sensible. | Obligatorio |
| `creation_date` | `datetime` | 📅 Cuándo se creó la cuenta. | Obligatorio |
| `last_update` | `datetime` | 📅 Última modificación conocida de la cuenta. | ◌ Opcional |
| `last_login` | `datetime` | 📅 Último acceso registrado. | ◌ Opcional |
| `status` | `bit(1)` | ✅ Indica si la cuenta está habilitada; el significado exacto de 0/1 no está comentado. | ◌ Opcional |

## `tb_products` — Catálogo de artículos vendibles {#tb-products}

**Total actual: 5,008 registros.** Los valores de ejemplo incluyen SKU, nombre, precio e inventario de productos.

| Columna exacta | Tipo SQL exacto | Lectura amigable | Atributos |
| --- | --- | --- | --- |
| `ID` | `int unsigned` | 🔢 Identificador interno del producto. | 🔑 PK · Obligatorio · Autonumérico |
| `SKU` | `varchar(50)` | 📝 Código comercial único para reconocer el producto. | Único · Obligatorio |
| `name` | `varchar(250)` | 📝 Nombre visible del producto. | Obligatorio |
| `description` | `text` | 📝 Explicación ampliada de sus características. | ◌ Opcional |
| `current_price` | `decimal(10,2)` | 🔢 Precio vigente del catálogo. | Obligatorio · Predeterminado `0.00` |
| `current_stock` | `int unsigned` | 🔢 Unidades disponibles según el inventario actual. | Obligatorio · Predeterminado `0` |
| `status` | `bit(1)` | ✅ Marca de disponibilidad o activación; la codificación exacta no está documentada. | ◌ Opcional |
| `creation_date` | `datetime` | 📅 Alta del producto. | Obligatorio |
| `last_update` | `datetime` | 📅 Última actualización del producto. | Obligatorio |

## `tbc_categories` — Clasificación jerárquica del catálogo {#tbc-categories}

**Total actual: 100 registros.** Una categoría puede señalar a otra categoría superior, formando una estructura de grupos y subgrupos.

| Columna exacta | Tipo SQL exacto | Lectura amigable | Atributos |
| --- | --- | --- | --- |
| `ID` | `int` | 🔢 Identificador de categoría. | 🔑 PK · Obligatorio |
| `name` | `varchar(100)` | 📝 Nombre de la categoría visible para el negocio. | Obligatorio |
| `description` | `varchar(255)` | 📝 Explicación breve de la categoría. | ◌ Opcional |
| `sup_id_category` | `int` | 🔢 Categoría padre, si esta es una subcategoría. | 🔗 FK a `tbc_categories.ID` · ◌ Opcional |
| `status` | `int` | 🔢 Estado de uso de la categoría; el SQL no fija catálogo de valores. | ◌ Opcional |

## `tbd_products_categories` — Asignación de productos a categorías {#tbd-products-categories}

**Total actual: 4,993 registros.** Es el puente que permite clasificar un producto en una o varias categorías.

| Columna exacta | Tipo SQL exacto | Lectura amigable | Atributos |
| --- | --- | --- | --- |
| `product_ID` | `int` | 🔢 Producto clasificado. | 🔑 PK compuesta · Obligatorio · vínculo inferido a `tb_products.ID` |
| `category_ID` | `int` | 🔢 Categoría asignada al producto. | 🔑 PK compuesta · 🔗 FK a `tbc_categories.ID` · Obligatorio |
| `status` | `bit(1)` | ✅ Marca de vigencia de la asignación. | Obligatorio |
| `creation_date` | `datetime` | 📅 Cuándo se clasificó el producto. | Obligatorio |
| `last_update` | `datetime` | 📅 Último cambio de la clasificación. | Obligatorio |

## `tbd_products_users_providers` — Oferta de producto por proveedor {#tbd-products-users-providers}

**Total actual: 0 registros.** Relaciona a quien ofrece un producto con el producto y su tipo de participación.

| Columna exacta | Tipo SQL exacto | Lectura amigable | Atributos |
| --- | --- | --- | --- |
| `user_ID` | `int unsigned` | 🔢 Persona que provee u ofrece el producto. | 🔑 PK compuesta · 🔗 FK a `tb_users.ID` · Obligatorio |
| `product_ID` | `int unsigned` | 🔢 Producto ofrecido. | 🔑 PK compuesta · 🔗 FK a `tb_products.ID` · Obligatorio |
| `type_ownership` | `enum('Owner','Reseller','Distributor')` | 📝 Papel del proveedor: dueño, revendedor o distribuidor. | ◌ Opcional |
| `status` | `bit(1)` | ✅ Indica si la oferta está activa. | Obligatorio |
| `avalaible_date` | `datetime` | 📅 Fecha en que queda disponible; el nombre original conserva el error ortográfico. | Obligatorio |
| `creation_date` | `datetime` | 📅 Alta de la oferta. | Obligatorio |
| `last_update` | `datetime` | 📅 Último cambio de la oferta. | Obligatorio |

## `tb_carts` — Carrito vigente de cada persona {#tb-carts}

**Total actual: 0 registros.** La restricción única sobre `user_ID` indica que una persona puede tener como máximo un carrito guardado.

| Columna exacta | Tipo SQL exacto | Lectura amigable | Atributos |
| --- | --- | --- | --- |
| `ID` | `int unsigned` | 🔢 Identificador del carrito. | 🔑 PK · Obligatorio · Autonumérico |
| `user_ID` | `int unsigned` | 🔢 Dueño del carrito. | 🔗 FK a `tb_users.ID` · Único · Obligatorio |
| `status` | `bit(1)` | ✅ Estado de uso del carrito. | ◌ Opcional |
| `creation_date` | `datetime` | 📅 Creación del carrito. | Obligatorio |
| `last_update` | `datetime` | 📅 Última modificación del carrito. | ◌ Opcional |

## `tbd_carts_items` — Productos dentro de un carrito {#tbd-carts-items}

**Total actual: 0 registros.** Cada renglón conserva producto, proveedor y cantidad solicitada; no admite cantidades menores a 1.

| Columna exacta | Tipo SQL exacto | Lectura amigable | Atributos |
| --- | --- | --- | --- |
| `cart_ID` | `int unsigned` | 🔢 Carrito al que pertenece el renglón. | 🔑 PK compuesta · 🔗 FK a `tb_carts.ID` · Obligatorio |
| `provider_ID` | `int unsigned` | 🔢 Proveedor del producto seleccionado. | 🔑 PK compuesta · 🔗 FK declarada a `tbd_products_users_providers.user_ID` · Obligatorio |
| `product_ID` | `int unsigned` | 🔢 Producto seleccionado. | 🔑 PK compuesta · 🔗 FK declarada a `tbd_products_users_providers.product_ID` · Obligatorio |
| `quantity` | `int unsigned` | 🔢 Unidades solicitadas del producto. | Obligatorio · mínimo 1 |
| `creation_date` | `datetime` | 📅 Momento en que se agregó al carrito. | Obligatorio |

## `tb_buyer_orders` — Cabecera de pedido de compra {#tb-buyer-orders}

**Total actual: 0 registros.** Guarda totales y estado general de un pedido; el campo original `seller_ID` apunta a una persona en `tb_users`.

| Columna exacta | Tipo SQL exacto | Lectura amigable | Atributos |
| --- | --- | --- | --- |
| `ID` | `int unsigned` | 🔢 Identificador del pedido. | 🔑 PK · Obligatorio · Autonumérico |
| `seller_ID` | `int unsigned` | 🔢 Persona asociada al pedido; el nombre sugiere vendedor, aunque la tabla se llama buyer orders. | 🔗 FK a `tb_users.ID` · Obligatorio |
| `subtotal` | `decimal(12,2)` | 🔢 Importe de productos antes del envío. | Obligatorio · no negativo |
| `shipping_cost` | `decimal(12,2)` | 🔢 Costo de entrega. | Obligatorio · no negativo |
| `fulfillment_status` | `enum('Pending','Accepted','Preparing','Sent','Delivered','Canceled','Returned')` | 📝 Etapa: pendiente, aceptado, preparando, enviado, entregado, cancelado o devuelto. | Obligatorio |
| `status` | `bit(1)` | ✅ Indicador técnico de vigencia del pedido. | Obligatorio |
| `creation_date` | `datetime` | 📅 Creación del pedido. | Obligatorio |
| `last_update` | `datetime` | 📅 Último cambio del pedido. | ◌ Opcional |

## `tbd_order_items` — Renglones y precios del pedido {#tbd-order-items}

**Total actual: 0 registros.** Conserva el nombre y precio unitario de la venta; `subtotal` se calcula automáticamente como cantidad por precio.

| Columna exacta | Tipo SQL exacto | Lectura amigable | Atributos |
| --- | --- | --- | --- |
| `ID` | `int unsigned` | 🔢 Identificador del renglón. | 🔑 PK · Obligatorio · Autonumérico |
| `seller_order_ID` | `int unsigned` | 🔢 Pedido al que pertenece el renglón. | 🔗 FK a `tb_buyer_orders.ID` · Obligatorio |
| `product_ID` | `int unsigned` | 🔢 Producto comprado. | 🔗 FK declarada a `tbd_products_users_providers.product_ID` · Obligatorio |
| `provider_ID` | `int unsigned` | 🔢 Proveedor que atendió ese producto. | 🔗 FK declarada a `tbd_products_users_providers.user_ID` · Obligatorio |
| `product_name` | `varchar(255)` | 📝 Nombre capturado al momento de comprar. | Obligatorio |
| `quantity` | `int unsigned` | 🔢 Unidades compradas. | Obligatorio · mínimo 1 |
| `unit_price` | `decimal(12,2)` | 🔢 Precio de una unidad en esa compra. | Obligatorio · no negativo |
| `subtotal` | `decimal(12,2)` | 🔢 Importe del renglón. | Calculado: `quantity * unit_price` |
| `creation_date` | `datetime` | 📅 Alta del renglón. | Obligatorio |

## `tbc_payment_methods_types` — Catálogo de tipos de pago {#tbc-payment-methods-types}

**Total actual: 0 registros.** Define las clases de medio de pago que el sistema puede reconocer.

| Columna exacta | Tipo SQL exacto | Lectura amigable | Atributos |
| --- | --- | --- | --- |
| `ID` | `int unsigned` | 🔢 Identificador del tipo de pago. | 🔑 PK · Obligatorio · Autonumérico |
| `code` | `varchar(30)` | 📝 Código corto único del tipo. | Único · Obligatorio |
| `description` | `text` | 📝 Descripción del tipo de pago. | Obligatorio |
| `status` | `bit(1)` | ✅ Disponibilidad del tipo de pago. | Obligatorio |
| `creation_date` | `datetime` | 📅 Alta del tipo. | Obligatorio |
| `last_update` | `datetime` | 📅 Última modificación. | ◌ Opcional |

## `tbd_user_payment_methods` — Medios de pago registrados por persona {#tbd-user-payment-methods}

**Total actual: 0 registros.** Guarda referencias de proveedores de pago, no el número completo de una tarjeta; `last_four` sugiere los cuatro últimos dígitos mostrables.

| Columna exacta | Tipo SQL exacto | Lectura amigable | Atributos |
| --- | --- | --- | --- |
| `ID` | `bigint unsigned` | 🔢 Identificador del medio registrado. | 🔑 PK · Obligatorio · Autonumérico |
| `user_ID` | `int unsigned` | 🔢 Persona titular. | 🔗 FK a `tb_users.ID` · Obligatorio |
| `payment_method_type_ID` | `int unsigned` | 🔢 Clase de pago elegida. | 🔗 FK a `tbc_payment_methods_types.ID` · Obligatorio |
| `provider` | `varchar(60)` | 📝 Empresa o plataforma que procesa el pago. | Obligatorio |
| `provider_method_ref` | `varchar(255)` | 📝 Referencia segura entregada por el proveedor. | Única junto con `provider` · Obligatorio |
| `label` | `varchar(100)` | 📝 Nombre que la persona da al medio de pago. | ◌ Opcional |
| `last_four` | `char(4)` | 📝 Últimos cuatro caracteres para reconocerlo. | ◌ Opcional |
| `is_default` | `tinyint(1)` | ✅ Señala si es el medio preferido. | Obligatorio |
| `status` | `bit(1)` | ✅ Indica si continúa disponible. | Obligatorio |
| `creation_date` | `datetime` | 📅 Registro inicial. | Obligatorio |
| `last_update` | `datetime` | 📅 Último cambio. | ◌ Opcional |

## `tbd_users_wishlist_products` — Productos marcados como favoritos {#tbd-users-wishlist-products}

**Total actual: 0 registros.** Une personas y productos guardados para revisar después.

| Columna exacta | Tipo SQL exacto | Lectura amigable | Atributos |
| --- | --- | --- | --- |
| `user_ID` | `int unsigned` | 🔢 Persona que guarda el favorito. | 🔑 PK compuesta · 🔗 FK a `tb_users.ID` · Obligatorio |
| `product_ID` | `int unsigned` | 🔢 Producto guardado. | 🔑 PK compuesta · 🔗 FK a `tb_products.ID` · Obligatorio |
| `status` | `bit(1)` | ✅ Marca de vigencia del favorito. | Obligatorio |
| `creation_date` | `datetime` | 📅 Cuándo se guardó. | Obligatorio |
| `last_update` | `datetime` | 📅 Último cambio. | Obligatorio |

## `tb_logs` — Bitácora de acciones sobre la base {#tb-logs}

**Total actual: 5,448 registros.** Sus ejemplos registran altas, cambios y eliminaciones de usuarios y productos. Es una fuente de trazabilidad, no una tabla operativa del catálogo.

| Columna exacta | Tipo SQL exacto | Lectura amigable | Atributos |
| --- | --- | --- | --- |
| `ID` | `int unsigned` | 🔢 Identificador del evento de auditoría. | 🔑 PK · Obligatorio · Autonumérico |
| `table_name` | `varchar(80)` | 📝 Tabla donde ocurrió la acción. | Obligatorio |
| `table_operation` | `enum('Create','Read','Update','Delete')` | 📝 Acción: crear, consultar, modificar o eliminar. | ◌ Opcional |
| `db_user` | `varchar(80)` | 📝 Usuario técnico que ejecutó la acción. | Obligatorio |
| `operation_date` | `datetime` | 📅 Cuándo se registró la acción. | Obligatorio |
| `table_description` | `text` | 📝 Detalle narrativo de lo que ocurrió. | Obligatorio |
| `operation_status` | `bit(1)` | ✅ Resultado o vigencia del evento; el SQL no define el significado de 0/1. | ◌ Opcional |

## Especificación de interfaz para Archify {#diseno}

@workflow id=experiencia-diccionario title="Recorrido recomendado de la interfaz"
| start | Abrir resumen | frontend | explorar |
| filter | Buscar tabla o campo | frontend | explorar |
| select | Abrir tarjeta de tabla | frontend | explorar |
| fields | Ver campos y atributos | database | detalle |
| relation | Resaltar conexiones | backend | detalle |
| start > filter | buscar |
| filter > select | coincidencia |
| select > fields | desplegar |
| fields > relation | ver relación |

@steps id=experiencia-pasos
| resumen | Abrir resumen | Mostrar 13 tarjetas con nombre técnico, nombre amigable y total actual. | experiencia-diccionario | start |
| buscar | Buscar tabla o campo | Filtrar por nombre exacto, palabra de negocio, tipo SQL o clave. No ocultar el texto buscado. | experiencia-diccionario | filter |
| detalle | Abrir tarjeta de tabla | Expandir con transición corta y conservar el foco de teclado. | experiencia-diccionario | select |
| campos | Ver campos y atributos | Ordenar por definición SQL y mostrar badges de tipo, clave y obligatoriedad. | experiencia-diccionario | fields |
| conexiones | Resaltar conexiones | Al seleccionar una FK, enfocar tabla y campo de destino con explicación breve. | experiencia-diccionario | relation |

El agente debe construir cada tabla como una tarjeta expandible con: nombre exacto, subtítulo amigable, contador, finalidad, columnas, y relaciones. Use color solo como apoyo: texto y símbolo deben comunicar el estado por sí mismos. Mantenga el tipo SQL literal, incluyendo `unsigned`, `enum`, valores por defecto y campos calculados.

| Componente | Comportamiento y contenido requerido |
| --- | --- |
| Cabecera | Título, fuente `RES 20.sql`, fecha de análisis y leyenda de badges. |
| Buscador | Búsqueda instantánea por tabla, columna, explicación y tipo SQL; resaltar coincidencias. |
| Filtros | Dominio: núcleo, venta, pago, auditoría; y atributos: PK, FK, obligatorio, opcional, texto, número, fecha y sí/no. |
| Tarjeta de tabla | Nombre técnico sin alteraciones, explicación humana y badge de conteo actual. Una tabla vacía debe decir “0 registros en este respaldo”, no “sin uso”. |
| Vista de campos | Filas con columna, tipo SQL exacto, significado, badges de dato y atributos. Priorizar lectura móvil con tarjetas apiladas. |
| Vista de relaciones | Diagrama navegable y frases como “Cada renglón de pedido pertenece a un pedido”. Al hacer clic, abrir ambas tablas. |
| Estado de inferencia | Mostrar “Inferido del nombre/tipo/datos” donde el SQL no confirme el significado. Mostrar “Declarado en SQL” para PK, FK, `UNIQUE`, `NOT NULL`, `CHECK` y valores por defecto. |

## Interacciones y movimiento {#interacciones}

- Use expansión y colapso de 180 a 240 ms, con altura y opacidad; respete `prefers-reduced-motion`.
- Al filtrar, anime únicamente la entrada y salida de tarjetas; no reordene el foco sin una acción del lector.
- Permita “vista tarjetas” y “vista relaciones”; conserve búsqueda y filtros al cambiar.
- Mantenga contraste accesible y navegación completa por teclado. El color de PK, FK, obligatorio y opcional debe tener etiqueta textual.
- No exponga valores de `password` ni `provider_method_ref` en muestras; son campos sensibles o referencias de pago.

## Validación antes de publicar {#validacion}

- [ ] El número de tablas visibles es 13 y cada una conserva su nombre técnico exacto.
- [ ] Los conteos mostrados son 5,008 productos, 4,993 relaciones producto-categoría, 5,448 bitácoras, 100 categorías, 7 usuarios y 0 para las demás tablas.
- [ ] Cada campo muestra su tipo SQL, explicación, clave y si acepta vacío.
- [ ] Cada relación declarada en `FOREIGN KEY` conserva tabla y campo de origen y destino.
- [ ] Los vínculos inferidos están diferenciados de las restricciones declaradas.
- [ ] La búsqueda, filtros, expansión, cambio de vista y modo de movimiento reducido funcionan con teclado.
