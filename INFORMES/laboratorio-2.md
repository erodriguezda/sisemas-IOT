# Informe de laboratorio 2: Thread Mesh — direccionamiento, múltiples saltos y autorrecuperación

**Proyecto:** AgroNet Solutions — SmartCrop  
**Área:** Conectividad de red de campo  
**Tecnología:** Thread sobre IEEE 802.15.4, IPv6 y 6LoWPAN  
**Plataforma de prueba:** nRF52840-DK  
**Canal de operación:** 20  
**Fecha:** 22 de octubre de 2026  

## 1. Objetivos

- Desplegar una red Thread compuesta por tres nodos: A (Líder), B (Router intermedio) y C (Nodo de destino).
- Capturar y clasificar las diferentes direcciones IPv6 asignadas por la pila Thread, determinando el propósito de cada una.
- Cuantificar el impacto en la latencia (RTT) al añadir saltos de enrutamiento en la malla.
- Medir empíricamente el tiempo de convergencia y autorrecuperación de la red ante la falla de un nodo de enrutamiento crítico.
- Alinear el comportamiento de la red con el modelo de referencia ISO/IEC 30141 y generar directrices de despliegue para el equipo de campo.

## 2. Fundamento teórico

Thread es un protocolo de red en malla (mesh) IPv6 diseñado para dispositivos de bajo consumo y baja tasa de bits, operando sobre el estándar IEEE 802.15.4. Elimina el punto único de fallo típico de las topologías en estrella al permitir que cualquier nodo con capacidad de enrutamiento participe en la propagación de los mensajes. 

Para habilitar el tráfico IPv6 sobre tramas MAC de 127 bytes, Thread utiliza la capa de adaptación 6LoWPAN, la cual aplica compresión de cabeceras (IPHC) y fragmentación a nivel de enlace. La topología y el enrutamiento se gestionan dinámicamente mediante el protocolo MLE (Mesh Link Establishment).

El algoritmo de enrutamiento evalúa las rutas utilizando el costo asociado a la calidad del enlace (Link Quality - LQ):
- **LQ 3** (Margen > 20 dB): Costo = 1
- **LQ 2** (Margen > 10 dB): Costo = 2
- **LQ 1** (Margen > 2 dB): Costo = 4

La pila Thread optimiza la transmisión prefiriendo el camino con el menor costo acumulado, lo que puede resultar en la elección de múltiples saltos de alta calidad (ej. 1 + 1 = 2) frente a un único salto degradado (ej. 4).

## 3. Procedimiento y configuración de la red

El laboratorio se realizó utilizando tres tarjetas de desarrollo nRF52840-DK, asignadas de la siguiente manera:

| Nodo | Rol inicial | Propósito en el experimento |
|---|---|---|
| **Nodo A** | Líder | Administra la red y emite los paquetes de diagnóstico (ping) |
| **Nodo B** | Router | Actúa como relevo (relay) de paquetes; se apaga para simular fallos |
| **Nodo C** | Router | Dispositivo final objetivo para las pruebas de latencia |

El canal físico se configuró en 20, con una potencia de transmisión de 0 dBm. La inicialización se realizó desde el Nodo A, exportando el *active dataset* para que los nodos B y C se integraran a la misma red lógica.

Secuencia de comandos base utilizada:

```bash
# En todos los dispositivos
ot txpower 0
ot factoryreset

# Configuración del Líder (Nodo A)
ot dataset init new
ot dataset channel 20
ot dataset commit active
ot ifconfig up
ot thread start
ot dataset active -x

# Asociación de los Routers (Nodos B y C)
ot dataset set active <DATASET_HEX_DE_A>
ot ifconfig up
ot thread start

# Verificación de topología
ot router table
```

Para evaluar el enrutamiento multisalto, se penalizó artificialmente el enlace directo entre el Nodo A y el Nodo C utilizando el comando `ot macfilter rss add-lqi`, asignándole una calidad de recepción fija equivalente a LQ 1.

## 4. Resultados y análisis

### 4.1 Formación de la malla y selección de rutas

Inicialmente, los tres dispositivos establecieron enlaces directos de calidad máxima (LQ 3). Al aplicar el filtrado MAC sobre el enlace A–C, se forzó un recálculo de la tabla de enrutamiento en el Nodo A, que conmutó su salida hacia el Nodo C a través del Nodo B.

| Estado del laboratorio | Enlace A–B | Enlace B–C | Enlace A–C | Ruta activa hacia C | Costo total evaluado |
|---|---:|---:|---:|---|---:|
| **Topología natural** | LQ 3 | LQ 3 | LQ 3 | A → C (Directa) | Directa: 1; vía B: 2 |
| **Topología penalizada** | LQ 3 | LQ 3 | LQ 1 | A → B → C (2 saltos) | Directa: 4; vía B: 1 + 1 = 2 |

Se comprobó que el protocolo responde según el estándar, prefiriendo la ruta indirecta (costo 2) para eludir el segmento de baja calidad (costo 4).

### 4.2 Direccionamiento IPv6 asignado

Mediante las consultas `ot ipaddr` y `ot extaddr`, se registraron las siguientes direcciones de red:

| Tipo | Dirección IPv6 capturada | Alcance | Uso dentro de la red |
|---|---|---|---|
| **Link-local** | `fe80::3a4b:c1d2:e3f4:5678` | Un salto de radio | Intercambio de mensajes MLE y descubrimiento de vecinos inmediatos |
| **RLOC** | `fdc4:1234:5678:0:0:ff:fe00:4400` | Malla interna | Enrutamiento topológico; cambia si el dispositivo cambia de nodo padre |
| **ALOC** | `fdc4:1234:5678:0:0:ff:fe00:fc00` | Malla interna | Identificador Anycast de roles de red (en este caso, líder) |
| **ML-EID** | `fdc4:1234:5678:0:1122:3344:5566:7788` | Malla interna | Identificador de aplicación estático, ideal para los sensores SmartCrop |
| **Multicast local** | `ff02::1` / `ff02::2` | Enlace local | Difusión para todos los dispositivos o routers del entorno inmediato |
| **Multicast mesh** | `ff03::1` / `ff03::2` | Toda la red | Difusión a toda la infraestructura Thread desplegada |

### 4.3 Verificación de la derivación RLOC16

Se validó el mecanismo de asignación de direcciones. El RLOC de prueba termina en el valor hexadecimal `0x4400`. Al aplicar el desplazamiento de bits estipulado por el estándar para extraer el Router ID (6 bits más significativos):

> **Router ID** = `0x4400` >> 10 = **17** (decimal)

La tabla de enrutamiento (`ot router table`) confirmó la existencia de un router con ID 17, demostrando cómo Thread codifica la ubicación lógica del dispositivo directamente dentro de su dirección IPv6 rutable.

### 4.4 Latencia (RTT) y saltos de red

Se inyectaron 20 paquetes ICMPv6 de 64 bytes hacia las direcciones ML-EID objetivo, con una cadencia de 500 ms.

| Escenario | Saltos físicos | RTT mínimo | RTT promedio | RTT máximo | Pérdida de paquetes |
|---|---:|---:|---:|---:|---:|
| **A → B (Directo)** | 1 | 10,0 ms | 14,5 ms | 22,0 ms | 0 % |
| **A → C (A través de B)** | 2 | 19,0 ms | 28,3 ms | 45,0 ms | 5 % (1 pqte) |

El enrutamiento a través de un nodo intermedio incrementó el RTT promedio en **13,8 ms**. Este retardo es consecuencia del tiempo de aire adicional y el procesamiento de la pila en el Nodo B. La ligera pérdida del 5 % obedeció a la resolución inicial de la dirección ML-EID por parte del protocolo, tras lo cual la comunicación se estabilizó al 100 %.

### 4.5 Resiliencia y tiempos de autorrecuperación

Para simular un fallo en terreno, se inició un tráfico constante (1 ping/s) desde A hacia C vía B. En el transcurso de la prueba, se interrumpió abruptamente el suministro eléctrico del Nodo B. Se contabilizaron los segundos de indisponibilidad hasta que el Nodo A reconfiguró su salida por el enlace directo degradado de respaldo.

| Iteración | Pings sin respuesta (segundos) | Siguiente salto adoptado (RLOC16) |
|---:|---:|---|
| **Ensayo 1** | 7 | `0x2100` (ID: 8 - Nodo C directo) |
| **Ensayo 2** | 9 | `0x2100` (ID: 8 - Nodo C directo) |
| **Ensayo 3** | 6 | `0x2100` (ID: 8 - Nodo C directo) |
| **Promedio** | **7,3 s** | **Ruta de respaldo activada con éxito** |

La red demostró una capacidad de convergencia promedio de **7,3 segundos**. El descarte continuado de ACKs a nivel MAC por parte del nodo caído desencadena un fallo rápido de enlace (link failure), permitiendo que la capa de enrutamiento deseche la ruta defectuosa antes de que el router expire formalmente (lo cual puede tomar hasta 100 segundos).

## 5. Análisis técnico adicional

- **Encapsulamiento IPv6/802.15.4:** La inclusión de una cabecera IPv6 de 40 bytes en un MTU de 127 bytes es viable gracias al mecanismo de compresión dependiente del contexto de 6LoWPAN (IPHC, RFC 6282). Los prefijos de red son deducidos a nivel de malla y los identificadores de interfaz a menudo se reconstruyen directamente desde las direcciones MAC extendidas, reduciendo dramáticamente el peso de la cabecera.
- **Redundancia necesaria:** La prueba exige degradar artificialmente (LQ 1) el enlace A–C en lugar de bloquearlo al 100%. Un bloqueo total eliminaría la ruta física de contingencia, haciendo imposible probar la conmutación y autorrecuperación de la topología.

## 6. Integración con ISO/IEC 30141

El despliegue de Thread cumple el rol de interconexión local dentro del **Sensing and Controlling Domain (SCD)** de la arquitectura IoT. 

| Perfil en la malla | Intervención en el esquema | Consumo estimado |
|---|---|---|
| **Líder / Router** | Propagación de datos e interconexión multisalto entre sensores y el gateway. | Alto (Rx en estado encendido continuo; requiere panel solar en campo). |
| **Sleepy End Device (SED)** | Sensor de humedad de suelo u hoja final. Solo se comunica con su Router padre. | Mínimo (El radio entra en estado de suspensión profundo, habilitando varios meses/años de batería). |

Se establece que el hardware alimentado por batería en los cultivos de SmartCrop opere estrictamente como **Sleepy End Devices (SED)**, dependiendo de nodos alimentados permanentemente para el enrutamiento.

## 7. Decisión de Arquitectura: ADR-003 - Topología Field Network

**Resolución:** Implementar **Thread Mesh** como base de conectividad en los cultivos.

| Atributo | Thread Mesh | Estrella (Alternativa LPWAN) |
|---|---|---|
| **Alcance efectivo** | Escalable: Ampliación dinámica agregando routers | Fijo: Restringido al rango de radio del Gateway |
| **Resiliencia ante fallos** | Alta: Recuperación automática probada (~7,3 s) | Nula: El colapso del Gateway desactiva todo el cultivo |
| **Penalización por distancia** | ~13,8 ms de retardo adicional por cada salto de radio | Retardo base fijo, pero sin posibilidad de esquivar obstáculos |

**Sustento:** Las pruebas de laboratorio confirman una latencia en torno a 30 ms para rutas de dos saltos y tiempos de conmutación inferiores a 10 segundos ante fallos de hardware. Estos parámetros aseguran confiabilidad y son transparentes para la recolección de métricas agrícolas.

## 8. Informe ejecutivo para Carlos (Líder de Operaciones)

- **Capacidad de supervivencia de la red:** Si un tractor destruye un nodo intermedio, la red restaura las comunicaciones por vías alternas en un promedio de **7,3 segundos**, cumpliendo el objetivo de recuperar el servicio en menos de 2 minutos.
- **Protocolo en terreno:** Cuando se incorporen nuevos equipos, es mandatorio confirmar que operan en el canal 20 con la clave de red correcta. Se debe verificar con `ot router table` que cada sensor cuente con al menos dos enlaces de calidad LQ 2 o LQ 3 hacia routers diferentes para garantizar la redundancia probada hoy.

## 9. Informe ejecutivo para Laura (Arquitectura de Software)

- **Resolución de direcciones:** Las aplicaciones de recolección de datos y actuadores de válvulas deben apuntar invariablemente a las direcciones **ML-EID**. Las direcciones RLOC cambian si un dispositivo cambia su ruta en la malla, mientras que la ML-EID permanece constante, asegurando que la telemetría nunca pierda su objetivo.
- **Presupuesto de latencia:** El enrutamiento de la malla añade aproximadamente **13,8 ms** de demora (RTT) por cada salto adicional; los sistemas de control deben programar sus tiempos de espera (timeouts) en función de la profundidad máxima en saltos esperada en la red de cultivo.
