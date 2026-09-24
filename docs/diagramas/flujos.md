# Flujos de datos · Textiles DLB

Versión editable (Mermaid) de los flujos. La versión ilustrada, con notación Gane–Sarson, está en [`index.html`](index.html) (figuras F1–F8).

En los diagramas de flujo, los procesos van numerados, los almacenes de datos llevan código `D1…D8` y las entidades externas son rectángulos. Las flechas gruesas marcan el camino del dinero.

## F1 · Diagrama de contexto (nivel 0)

```mermaid
flowchart LR
  CLI["Cliente"]
  EQ["Equipo DLB"]
  WO["Wompi"]
  DIAN["Proveedor DIAN"]
  TR["Transportadoras"]
  MSG["Correo y WhatsApp"]
  AN["Analítica y errores<br/>GA4 · Meta · Sentry"]
  T(("0<br/>Tienda en línea<br/>Textiles DLB"))

  CLI -- "pedido, datos de entrega, devoluciones, PQRS" --> T
  T -- "precios, estado del pedido, factura" --> CLI
  EQ -- "productos, precios, stock, estados" --> T
  T -- "pedidos nuevos, alertas, informes" --> EQ
  T -- "referencia, importe y firma" --> WO
  WO == "pago confirmado (webhook)" ==> T
  T -- "datos de la venta" --> DIAN
  DIAN -- "número, CUFE, PDF y XML" --> T
  T -- "guía y valor a recaudar" --> TR
  TR -- "estados y recaudos" --> T
  T -- "confirmaciones" --> MSG
  MSG -- "respuestas (WhatsApp)" --> T
  T -- "compra confirmada y errores" --> AN
```

## F2 · De la búsqueda al pago confirmado (nivel 1)

```mermaid
flowchart LR
  CLI["Cliente"]
  WO["Wompi"]
  P1(["1.0 Consultar catálogo"])
  P2(["2.0 Gestionar carrito"])
  P3(["3.0 Crear pedido y cobrar"])
  P4(["4.0 Confirmar pago"])
  D1[("D1 Catálogo y stock")]
  D3[("D3 Pedidos y pagos")]
  D4[("D4 Clientes y envíos")]
  D5[("D5 Carritos · Firestore")]
  D6[("D6 Seguimiento · Firestore")]
  D7[("D7 Eventos de webhook")]

  CLI -- "búsqueda y filtros" --> P1
  D1 -- "catálogo y stock" --> P1
  P1 -- "productos, precios, disponibilidad" --> CLI
  CLI -- "variante y cantidad" --> P2
  P2 <-- "ids y cantidades" --> D5
  CLI -- "contacto, dirección, medio de pago" --> P3
  D5 -- "carrito" --> P3
  D1 <-- "precios actuales / reserva con caducidad" --> P3
  D4 <-- "zona, tarifa, contra entrega" --> P3
  P3 -- "pedido pendiente (importes congelados)" --> D3
  P3 == "referencia, importe y firma" ==> WO
  WO == "evento firmado" ==> P4
  P4 -- "id del evento" --> D7
  P4 == "pago aprobado, pedido pagado" ==> D3
  P4 -- "descuenta stock" --> D1
  P4 -- "estado" --> D6
  D6 -. "estado del pedido en vivo" .-> CLI
```

## F3 · Después de la compra (nivel 1)

```mermaid
flowchart LR
  DIAN["Proveedor DIAN"]
  TR["Transportadoras"]
  MSG["Correo y WhatsApp"]
  CLI["Cliente"]
  WO["Wompi"]
  EQ["Equipo DLB"]
  P5(["5.0 Facturar"])
  P6(["6.0 Despachar y seguir envíos"])
  P7(["7.0 Avisar al cliente"])
  P8(["8.0 Devoluciones y reembolsos"])
  P9(["9.0 Administrar catálogo y pedidos"])
  D1[("D1 Catálogo y stock")]
  D3[("D3 Pedidos y pagos")]
  D6[("D6 Seguimiento · Firestore")]
  D8[("D8 Archivos · Storage")]

  D3 <-- "pedido pagado / número y CUFE" --> P5
  P5 -- "datos de la venta" --> DIAN
  DIAN -- "CUFE, PDF y XML" --> P5
  P5 -- "PDF y XML" --> D8
  D3 <-- "pedido a despachar / guía y estados" --> P6
  P6 -- "guía y valor a recaudar" --> TR
  TR -- "estados y recaudos" --> P6
  P6 -- "estado del envío" --> D6
  D3 -- "eventos del pedido" --> P7
  P7 -- "mensajes" --> MSG
  MSG -- "respuestas" --> P7
  CLI -- "solicitud de devolución" --> P8
  D3 <-- "pedido y pagos / estado" --> P8
  P8 -- "reembolso" --> WO
  WO -- "confirmación" --> P8
  P8 -- "reingreso de stock" --> D1
  EQ -- "productos, precios, stock y estados" --> P9
  P9 -- "escribe" --> D1
  P9 <-- "estados" --> D3
```

## F4 · Compra con pago en línea

```mermaid
sequenceDiagram
  autonumber
  participant NAV as Navegador
  participant NX as Next.js (App Hosting)
  participant FS as Firestore
  participant SQL as SQL Connect
  participant WO as Wompi
  participant FN as Cloud Functions

  NAV->>NX: Pagar (contacto, dirección, medio de pago)
  NX->>FS: Lee el carrito (ids y cantidades)
  NX->>SQL: Precios, stock disponible, zona y cupón
  NX->>NX: Recalcula subtotal, IVA, envío y descuento (ignora importes del navegador)
  NX->>SQL: CrearPedido @transaction: pedido PENDIENTE, líneas congeladas, reservas 20 min
  NX-->>NAV: Referencia, importe en centavos y firma de integridad
  NAV->>WO: Paga en el widget (la tarjeta no pasa por la tienda)
  WO-->>NAV: Redirige a /pedido/{token}: «procesando»
  NAV-->>FS: Escucha seguimiento/{token}
  WO->>FN: Webhook transaction.updated (firmado)
  FN->>FN: Verifica la firma. Si el evento ya existe, responde 200 y termina
  alt pago APROBADO
    FN->>SQL: ConfirmarPago @transaction: WebhookEvent, pedido PAGADO, stock −, reservas consumidas
    SQL-->>FN: Trigger onMutationExecuted → alConfirmarPedido
    FN->>FS: seguimiento/{token} = PAGADO
    Note over FN: Encola factura DIAN, correo y conversión (Cloud Tasks)
  else RECHAZADO o ERROR
    FN->>SQL: Marca el pago rechazado y libera las reservas
    FN->>FS: seguimiento/{token} = PAGO_RECHAZADO
  end
  FN-->>WO: 200 OK (si no, Wompi reintenta)
  FS-->>NAV: La página cambia sola: «¡Pago recibido!»
```

## F5 · Contra entrega

```mermaid
sequenceDiagram
  autonumber
  participant NAV as Navegador
  participant NX as Next.js
  participant SQL as SQL Connect
  participant FN as Cloud Functions
  participant WA as WhatsApp
  participant EQ as Equipo DLB
  participant TR as Transportadora

  NAV->>NX: Confirmar pedido contra entrega
  NX->>SQL: ¿El municipio tiene cobertura y el total está bajo el tope?
  NX->>SQL: ConfirmarContraEntrega @transaction: POR CONFIRMAR, stock descontado, pago PENDIENTE
  NX-->>NAV: Gracias: «te escribiremos por WhatsApp»
  SQL-->>FN: Trigger → alConfirmarPedido
  FN->>WA: Plantilla: confirma tu pedido y dirección
  WA-->>FN: El cliente responde «Confirmo»
  FN->>SQL: Pedido → EN PREPARACIÓN
  Note over FN,WA: Sin respuesta en 24 h: el equipo llama o cancela y el stock vuelve
  EQ->>TR: Crea la guía con valor a recaudar
  TR->>NAV: Entrega y cobra en efectivo
  TR-->>FN: Estado ENTREGADO y liquidación
  FN->>FN: conciliarContraEntrega (diaria): pago RECAUDADO → CONCILIADO
  opt rechazado en la puerta
    TR->>FN: Devuelto al remitente
    FN->>SQL: Pedido DEVUELTO, stock repuesto, costo del flete registrado
  end
```

## F6 · Alta de un producto desde el panel

```mermaid
sequenceDiagram
  autonumber
  participant EQ as Navegador (equipo)
  participant AU as Firebase Auth
  participant NX as Next.js /admin
  participant SQL as SQL Connect
  participant ST as Cloud Storage
  participant FN as Cloud Functions

  EQ->>AU: Inicia sesión con verificación en dos pasos
  AU-->>EQ: ID token con claim rol = catálogo
  EQ->>NX: Guardar producto, colores y variantes
  NX->>NX: Verifica sesión y rol, y valida con Zod
  NX->>SQL: CrearProducto + colores + variantes (BORRADOR)
  EQ->>ST: Sube fotos a productos/{id}/{colorId}/
  Note over ST: Reglas: solo rol catálogo, solo imágenes, máx. 10 MB
  ST-->>FN: onObjectFinalized
  FN->>FN: Valida, quita metadatos, genera miniaturas
  FN->>SQL: Registra ProductImage
  EQ->>NX: Publicar
  NX->>SQL: Producto → PUBLICADO
  NX->>NX: revalidateTag(producto): ficha y listados nuevos
```

## F7 · Cuentas opcionales y sesión

```mermaid
sequenceDiagram
  autonumber
  participant NAV as Navegador
  participant AU as Firebase Auth
  participant BF as Functions (beforeUserCreated)
  participant NX as Next.js
  participant SQL as SQL Connect

  NAV->>AU: Crear cuenta (correo, Google o enlace por correo)
  AU->>BF: Función de bloqueo antes de crear el usuario
  BF->>SQL: Busca o crea Customer por correo y guarda el uid
  BF-->>AU: Permite y asigna el claim rol = cliente
  AU-->>NAV: ID token (1 h)
  NAV->>NX: POST /api/sesion con el ID token
  NX->>NX: verifyIdToken → createSessionCookie (httpOnly, Secure, 5 días)
  NX-->>NAV: Set-Cookie: __session
  NAV->>NX: GET /cuenta/pedidos
  NX->>SQL: Pedidos del cliente (uid verificado en el servidor)
  NX-->>NAV: Página renderizada en el servidor
```

## F8 · Ciclo de vida del pedido

Es la máquina de estados de `domain/order.ts`: cualquier transición que no esté aquí debe ser rechazada por el dominio.

```mermaid
stateDiagram-v2
  [*] --> PENDING_PAYMENT: pago en línea
  [*] --> COD_PENDING_CONFIRMATION: contra entrega

  PENDING_PAYMENT --> PAID: webhook aprobado
  PENDING_PAYMENT --> PAYMENT_FAILED: webhook rechazado
  PAYMENT_FAILED --> PENDING_PAYMENT: reintenta
  PENDING_PAYMENT --> EXPIRED: reserva vencida (cada 5 min)

  PAID --> PREPARING: el equipo toma el pedido
  COD_PENDING_CONFIRMATION --> PREPARING: confirmado por WhatsApp
  COD_PENDING_CONFIRMATION --> CANCELLED: no confirma o cancela

  PAID --> CANCELLED: el equipo cancela y reembolsa
  PREPARING --> CANCELLED: el equipo cancela (reembolsa si ya pagó)
  PREPARING --> SHIPPED: guía creada
  SHIPPED --> DELIVERED: entregado
  SHIPPED --> RETURNED_TO_SENDER: rechazado en la puerta
  DELIVERED --> RETURN_REQUESTED: retracto o cambio
  RETURN_REQUESTED --> REFUNDED: reembolso total o parcial

  EXPIRED --> [*]
  CANCELLED --> [*]
  RETURNED_TO_SENDER --> [*]
  REFUNDED --> [*]
```

El cobro contra entrega vive en `Payment`: `PENDIENTE → RECAUDADO` (transportadora) `→ CONCILIADO` (tarea diaria). Cada transición del pedido queda en `OrderStatusHistory` con quién la hizo y cuándo.
