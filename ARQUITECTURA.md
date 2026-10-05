# Arquitectura de eventos - Semana 7

## Patrón seleccionado
Arquitectura orientada a eventos (Event-Driven Architecture) con Kafka. Los productores publican eventos sin acoplarse a los consumidores, permitiendo procesamiento asíncrono y escalado por particiones/grupos de consumidores.

```mermaid
flowchart LR
    A[Web / Mobile / ATM] --> B[API Banco]
    B --> P[Kafka Producer]
    P --> T1[transaccion-creada]
    P --> T2[estado-cuenta-generado]
    P --> T3[alerta-fraude]
    T1 --> C1[Consumer Transacciones]
    T2 --> C2[Consumer Estado Cuenta]
    T3 --> C3[Consumer Fraude]
    B --> R[Resilience4j]
    R --> CB[Circuit Breaker + Retry + Fallback]
```

## Evidencias sugeridas
1. `docker compose up -d` mostrando Kafka iniciado.
2. `mvn spring-boot:run` mostrando la aplicación levantada.
3. POST `/api/auth/login` para obtener JWT.
4. POST `/api/eventos/transaccion` con Bearer Token y respuesta HTTP 202.
5. Consola mostrando `Evento procesado [transaccion-creada]`.
6. GET `/api/eventos/resiliencia` mostrando `FALLBACK_ACTIVO`.
