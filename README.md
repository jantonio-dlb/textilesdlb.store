# Textiles DLB · Tienda en línea

Repositorio de la tienda en línea de Textiles DLB (`textilesdlb.store`).

**Estado: fase de diseño.** Todavía no hay código de la aplicación. Aquí están las propuestas de diseño y la arquitectura para decidir antes de programar.

## Documentación

| Documento | Qué contiene |
|---|---|
| [`docs/disenos/index.html`](docs/disenos/index.html) | Comparador de las cuatro propuestas de diseño: miniaturas, tabla comparativa, requisitos comunes y recomendación |
| [`propuesta-1-muestrario.html`](docs/disenos/propuesta-1-muestrario.html) | **Muestrario** · el color como protagonista |
| [`propuesta-2-taller.html`](docs/disenos/propuesta-2-taller.html) | **Taller** · hecho por nosotros, a tu medida |
| [`propuesta-3-indigo.html`](docs/disenos/propuesta-3-indigo.html) | **Índigo** · profundo, sereno y premium |
| [`propuesta-4-vitrina.html`](docs/disenos/propuesta-4-vitrina.html) | **Vitrina** · claro, directo y pensado para vender |
| [`docs/diagramas/index.html`](docs/diagramas/index.html) | Arquitectura, modelo entidad-relación, flujos de datos, secuencias y ciclo de vida del pedido (19 figuras) |
| [`docs/diagramas/arquitectura.md`](docs/diagramas/arquitectura.md) | Diagramas de arquitectura en Mermaid |
| [`docs/diagramas/modelo-de-datos.md`](docs/diagramas/modelo-de-datos.md) | Modelo entidad-relación en Mermaid, colecciones de Firestore y rutas de Storage |
| [`docs/diagramas/flujos.md`](docs/diagramas/flujos.md) | Flujos de datos, secuencias y estados en Mermaid |

**Cómo ver los HTML.** GitHub muestra el código de los `.html`, no la página: descarga o clona el repositorio y abre los archivos en el navegador. Son autónomos; solo necesitan internet para cargar las tipografías. Los `.md` con Mermaid sí se ven dibujados en GitHub y son la versión fácil de editar.

Cada propuesta de diseño muestra las mismas cinco pantallas (inicio, catálogo, ficha de producto, checkout y móvil) con los mismos productos y precios de ejemplo, anotaciones numeradas y selectores interactivos de color, talla y medio de pago.

## Stack

- **Next.js** (App Router) con **TypeScript** estricto, **Tailwind CSS v4** y shadcn/ui.
- **Firebase App Hosting** para desplegar desde GitHub con renderizado en servidor.
- **Firebase SQL Connect** (antes Data Connect) sobre **Cloud SQL para PostgreSQL**: catálogo, stock, clientes, pedidos, pagos y facturas.
- **Cloud Firestore** para lo concreto: carritos, seguimiento del pedido en vivo, favoritos, contenido editable, avisos del panel y PQRS.
- **Firebase Authentication**, **Cloud Storage** y **Cloud Functions** (webhook de pagos, colas y tareas programadas).
- **Wompi** (PSE, Nequi, tarjetas con cuotas, Daviplata, Bancolombia) y **contra entrega**; factura electrónica con un proveedor autorizado por la DIAN.

## Reglas que no se negocian

1. El precio, los impuestos y el envío los calcula siempre el servidor; el carrito del navegador solo lleva ids y cantidades.
2. Un pago solo se confirma con el webhook firmado de Wompi, nunca con el redireccionamiento.
3. Todo es idempotente: un webhook o una tarea repetidos no duplican nada.
4. El pedido congela nombre, precio, impuesto y dirección; cambiar un precio no reescribe el historial.
5. El dinero se guarda en enteros (centavos de peso colombiano).
6. El stock se reserva con caducidad al empezar el pago y se descuenta al confirmarlo.
7. El enlace de seguimiento usa un token aleatorio, nunca un número correlativo.
8. Toda entrada se valida en el servidor; los datos de tarjeta nunca pasan por la tienda.

## Próximos pasos

1. Elegir la propuesta de diseño (o combinación) y cerrar paleta, tipografías y componentes.
2. Resolver lo pendiente (sección 5 del documento de diagramas): proveedor de factura DIAN, región de Google Cloud, transportadoras, cobertura y tope de contra entrega, tarifas de envío.
3. **Fase 0 · Cimientos:** monorepo, Next.js con TypeScript estricto y Tailwind, proyectos de Firebase de desarrollo y producción, CI en GitHub Actions y un primer despliegue en App Hosting. Prueba de concepto de SQL Connect con el esquema del catálogo y el filtro por talla y color.
4. **Fases 1 a 5:** catálogo, carrito, cobrar, operar (panel y factura) y pulir.
