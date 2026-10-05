# Banco XYZ - Optimización de Procesos Batch

## Descripción del proyecto

Modernización de los procesos batch del sistema legacy del Banco XYZ usando
Spring Batch. El objetivo es migrar datos de forma más eficiente, resistente a
errores y escalable, con procesamiento por lotes, validación de información,
manejo de excepciones y ejecución paralela.

Los procesos leen archivos CSV y guardan la información en una base de datos
relacional. Además, el proyecto expone APIs por canal (BFF), autenticación con
JWT, mensajería de eventos con Kafka y tolerancia a fallos con Resilience4j.

## Tecnologías utilizadas

- Java 17
- Spring Boot 3.3.5
- Spring Batch
- Spring Data JPA
- Spring Security + JWT (jjwt)
- Apache Kafka (spring-kafka)
- Resilience4j (Circuit Breaker y Retry)
- H2 (desarrollo) y PostgreSQL (driver incluido para producción)
- Maven y Lombok

## Arquitectura

### Procesos batch

Cada job sigue el esquema Reader → Processor → Writer, con chunks de 5 registros:

| Job | Archivo de entrada | Qué hace |
|---|---|---|
| `reporteTransaccionesDiarias` | `transacciones.csv` | Valida que el monto sea mayor a 0 y marca la transacción como `PROCESADA` |
| `calculoInteresesMensuales` | `cuentas.csv` | Valida el saldo y le aplica el interés |
| `generacionEstadosCuentaAnuales` | `estado_cuenta.csv` | Valida el año y los montos, y recalcula el saldo final |

Características:
- **Ejecución paralela:** los steps usan un `ThreadPoolTaskExecutor`. Como
  `FlatFileItemReader` no es thread-safe, la lectura se sincroniza con
  `SynchronizedItemStreamReader`; el procesamiento y la escritura sí son paralelos.
- **Tolerancia a fallos:** se reintentan solo errores transitorios de base de
  datos (hasta 3 veces) y los registros inválidos se saltan (límite de 10).
- **Registro de errores:** `ErrorListener` informa los registros rechazados
  y el motivo. En los datos de ejemplo, `Pedro,-100` se rechaza por monto inválido.

### BFF por canal

| Canal | Ruta base | Rol requerido |
|---|---|---|
| Web | `/api/web/**` | `WEB` |
| Mobile | `/api/mobile/**` | `MOBILE` |
| ATM | `/api/atm/**` | `ATM` |

Los BFF devuelven datos de ejemplo fijos; todavía no leen lo que carga el batch.
El ATM rechaza retiros con monto menor o igual a 0.

### Seguridad

Autenticación JWT sin sesión (stateless). Los usuarios, contraseñas y el secreto
se configuran en `application.properties` (o con variables de entorno
`JWT_SECRET`, `AUTH_WEB_PASSWORD`, `AUTH_MOBILE_PASSWORD`, `AUTH_ATM_PASSWORD`).

| Usuario | Contraseña por defecto | Rol |
|---|---|---|
| `web` | `web123` | `WEB` |
| `mobile` | `mobile123` | `MOBILE` |
| `atm` | `atm123` | `ATM` |

Las rutas `/api/batch/**` y `/api/eventos/**` requieren el rol `WEB`.
Los valores por defecto son solo para desarrollo local.

### Eventos con Kafka

Arquitectura orientada a eventos (ver `ARQUITECTURA.md`):

- Tópicos con 3 particiones: `transaccion-creada`, `estado-cuenta-generado`, `alerta-fraude`.
- `EventoProducer` publica y `EventoConsumer` procesa mediante grupos de consumidores.
- Por ahora solo `transaccion-creada` se publica desde la API. Los eventos de estado
  de cuenta y de fraude están definidos pero aún no se emiten.

### Resilience4j

`ServicioExternoResiliente` combina Retry (3 intentos), Circuit Breaker y Fallback.
La configuración está en `application.properties` bajo la instancia `bancoExterno`.

## Cómo ejecutar

Requisitos: Java 17, Maven y Docker (para Kafka).

```bash
docker compose up -d
mvn clean package
mvn spring-boot:run
```

Los jobs no se ejecutan al arrancar (`spring.batch.job.enabled=false`); se lanzan
por la API.

##