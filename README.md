# Informe de Laboratorio 3: Implementación de CoAP para Telemetría y Control de Actuadores

**Proyecto:** AgriSmart Solutions — AquaYield  
**Plataforma de prueba:** nRF52840-DK con sistema operativo Zephyr y pila OpenThread  
**Red:** Thread sobre IEEE 802.15.4 (Canal 25)  
**Fecha:** 23 de octubre de 2026

---

## 1. Introducción

En este laboratorio se evaluó la implementación del Constrained Application Protocol (CoAP) como protocolo de capa de aplicación sobre una red IPv6/Thread para el proyecto AquaYield[cite: 4, 5]. A diferencia de HTTP que utiliza TCP, CoAP opera sobre UDP con una sobrecarga mínima de cabeceras (4 bytes), permitiendo un intercambio eficiente de datos en redes inalámbricas de baja potencia[cite: 4, 5]. Los datos se serializaron utilizando el formato binario CBOR (Concise Binary Object Representation) y se validó la extensión Observe para la notificación asíncrona de eventos térmicos[cite: 4, 5].

## 2. Objetivos

- Desplegar recursos CoAP para la lectura de sensores ambientales y la actuación de válvulas de riego[cite: 4, 5].
- Cuantificar la reducción de tráfico al utilizar el patrón Observe frente al sondeo periódico tradicional (polling)[cite: 5].
- Evaluar los mecanismos de confiabilidad de CoAP mediante mensajes Confirmables (CON) frente a No Confirmables (NON)[cite: 4, 5].
- Medir empíricamente el impacto en la latencia (RTT) al configurar el actuador como un dispositivo durmiente (Sleepy End Device - SED)[cite: 4, 5].
- Definir contratos formales de API utilizando esquemas CDDL[cite: 4, 5].

## 3. Configuración y metodología experimental

La topología de prueba constó de dos tarjetas nRF52840-DK[cite: 4, 5]:
- **Nodo C (Cliente / Leader):** Coordina la red Thread en el Canal 25 e inicia las peticiones CoAP[cite: 5].
- **Nodo S (Servidor / SED):** Expone los recursos de telemetría y control de válvula, operando con la radio apagada la mayor parte del tiempo[cite: 5].

Ambos dispositivos se programaron utilizando el entorno Zephyr RTOS y la línea de comandos de OpenThread (`ot coap`, `ot ping`, `ot pollperiod`)[cite: 4, 5].

## 4. Contratos de API e interfaz de servicio

### 4.1 Recurso `/env/temp` (Telemetría)

| Propiedad | Definición técnica |
|---|---|
| **Métodos permitidos** | `GET`, `GET` + Observe (RFC 7641)[cite: 4] |
| **Content-Format** | `60` (`application/cbor`)[cite: 4] |
| **Payload** | `{"t": float16}` (6 bytes)[cite: 4] |
| **Política Observe** | Notificación NON si delta > 0,5 °C; Heartbeat CON a los 45 s[cite: 4, 5] |

**Decodificación de ejemplo:**
Una petición devolvió el payload `a1 61 74 f9 4c a0`[cite: 5]. El valor hexadecimal `4ca0` en precisión media (IEEE 754) se traduce como **18,5 °C**[cite: 4, 5].

### 4.2 Recurso `/act/valve` (Actuador)

| Propiedad | Definición técnica |
|---|---|
| **Métodos permitidos** | `PUT` (escribir), `GET` (leer)[cite: 4] |
| **Payload CBOR** | `{"v": 0}` (Cerrado) o `{"v": 1}` (Abierto)[cite: 4] |
| **Códigos** | `2.04 Changed`, `4.00 Bad Request`[cite: 4] |
| **Idempotencia** | Sí, múltiples comandos `v=1` no alteran el relé si ya está abierto[cite: 4, 5] |

## 5. Resultados experimentales y análisis

### 5.1 Eficiencia del Patrón Observe vs. Polling

Se midió el tráfico durante 10 minutos (600 segundos) simulando un sensor que varía ligeramente su temperatura[cite: 5].

| Método | Peticiones enviadas | Notificaciones por Umbral | Notificaciones Heartbeat | Total de paquetes en red |
|---|---:|---:|---:|---:|
| **Polling (1 Hz)** | 600 | 0 | 0 | 1200 |
| **Observe** | 1 | 15 | 2 | 18 |

El patrón Observe logró una reducción del **98,5 %** en el tráfico aéreo frente al sondeo periódico[cite: 5].

### 5.2 Control de válvula y tiempo de espera (Timeout)

Al desconectar el Nodo S y enviar un comando de apertura en modo **CON** (`PUT /act/valve con 1`), el cliente realizó retransmisiones con retroceso exponencial, arrojando el error `ResponseTimeout` a los **93 segundos**[cite: 5]. En modo **NON**, la pérdida pasó desapercibida para la capa de aplicación, reafirmando que los comandos de actuación siempre deben ser confirmables[cite: 4, 5].

### 5.3 Latencia RTT en nodos SED

El Nodo S se configuró como durmiente, variando su intervalo de despertar (`pollperiod`)[cite: 4, 5]:

| Poll Period | RTT Mín (ms) | RTT Promedio (ms) | RTT Máx (ms) |
|---:|---:|---:|---:|
| **2 s** | 22 | 1.015 | 2.030 |
| **8 s** | 24 | 4.020 | 8.035 |
| **20 s** | 26 | 10.015 | 20.030 |

## 6. Autonomía energética del actuador SED

Basado en el hardware nRF52840, se estimó el consumo de un actuador configurado con un periodo de sondeo de **8 segundos**:
- Corriente de recepción (RX): $6.4\text{ mA}$[cite: 5]
- Corriente en Sleep: $0.0015\text{ mA}$[cite: 5]
- Tiempo activo por sondeo: $10\text{ ms}$[cite: 5]

El ciclo de trabajo ($D$) se calculó como $10\text{ ms} / 8000\text{ ms} = 0.00125$[cite: 5].
La corriente promedio resultante es:
$$I_{avg} = (0.00125 \times 6.4) + (0.99875 \times 0.0015) = 0.00949\text{ mA}$$
Con una batería de 2000 mAh, este consumo asegura la viabilidad multianual del dispositivo de campo sin requerir paneles solares[cite: 4, 5].

## 7. Respuestas a las preguntas técnicas de evaluación

- **¿Por qué CoAP utiliza UDP y no TCP?** TCP exige intercambios constantes (handshakes, keepalives) que agotan la batería, mientras que UDP sobre CoAP permite enviar datos en un único datagrama, delegando la confiabilidad solo cuando es estrictamente necesaria (mensajes CON)[cite: 4, 5].
- **¿Por qué Observe ahorra batería?** Invierte el rol de la comunicación: el cliente no gasta energía preguntando constantemente; el servidor toma la iniciativa de transmitir solo cuando el dato físico realmente cambia de forma relevante[cite: 4, 5].
- **¿Por qué CBOR?** Porque proporciona una serialización estructurada e interoperable que elimina los pesados caracteres de texto plano de JSON (comillas, corchetes), permitiendo que la respuesta entera ocupe apenas 17 bytes y quepa en una sola trama MAC 802.15.4[cite: 4, 5].

## 8. Mapeo arquitectónico ISO/IEC 30141 y Decisiones ADR

- **ASD (Application & Service Domain):** Incluye las rutas URI de CoAP (`/env/temp`), el modelo de datos CDDL para los payloads CBOR y la lógica de negocio de la API[cite: 4, 5].
- **SCD (Sensing & Controlling Domain):** Abarca el enrutamiento Thread subyacente y el mecanismo de retención de paquetes del nodo Padre hacia el SED[cite: 4, 5].

**ADR-004:** Seleccionar un `pollperiod` de 8 segundos para los actuadores AquaYield, equilibrando una latencia máxima aceptable de ~8 segundos con un drenaje ultra bajo de ~0.009 mA[cite: 5].

## 9. Diseño de seguridad: protección ante pérdida de conectividad

Si una válvula recibe el comando `OPEN` pero el gateway de la granja pierde conectividad posteriormente, existe el riesgo de inundar los cultivos[cite: 4]. Para mitigar esto, se implementó un **Failsafe Timer**[cite: 5]:
1. Al recibir `PUT {"v": 1}`, el actuador inicia una cuenta regresiva local de **5 minutos**[cite: 5].
2. El sistema central debe enviar un `PUT` de renovación antes de que expire el tiempo[cite: 5].
3. Si el temporizador llega a cero (indicando una caída de la red), el relé ejecuta automáticamente un cierre de emergencia (`v=0`)[cite: 5].
