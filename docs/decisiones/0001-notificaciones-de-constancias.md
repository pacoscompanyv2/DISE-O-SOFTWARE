# Comunicación asíncrona mediante registros persistidos para notificaciones de constancias

* Status: proposed
* Date: 2026-11-03

## Context and Problem Statement

El sistema ConstanciaUV debe permitir a estudiantes autorizados solicitar y obtener constancias digitales. El proceso incluye validar la identidad del usuario, generar/almacenar la constancia y notificar al estudiante vía correo electrónico mediante un proveedor externo.

Se requiere definir cómo estructurar la comunicación entre el core de ConstanciaUV y el servicio externo de notificaciones, asegurando que fallos o lentitudes en el proveedor no afecten el flujo principal de generación de constancias y evitando envíos duplicados.

## Decision Drivers

* **Tolerancia a fallos y disponibilidad del servicio (RT2, R2):** La generación y almacenamiento de constancias no debe detenerse ni fallar aunque el proveedor externo de correo esté fuera de servicio.
* **Persistencia y prevención de duplicados (RT3, RT4):** Las notificaciones pendientes deben almacenarse para reintentos posteriores y debe garantizarse la idempotencia, ya que el proveedor externo no asegura la entrega única.

## Considered Options

* **Alternativa A:** Comunicación directa con recuperación de fallos (Sincrónico con fallback)
* **Alternativa B:** Procesamiento asíncrono mediante eventos/registros persistidos (Outbox Pattern / Worker)

## Decision Outcome

Option chosen: **Alternativa B**, porque garantiza un desacoplamiento completo entre el dominio principal (generación de constancias) y el servicio secundario de notificaciones. Cumple estrictamente con RT2, RT3 y RT4 al almacenar primero la intención de notificación en una estructura persistente antes de delegar el envío a un proceso independiente.

### Consequences

* **Good:** Alta disponibilidad del módulo de constancias; las caídas del servicio de correo no impactan al estudiante al solicitar su documento.
* **Good:** Facilidad para implementar controles de idempotencia (mediante identificadores únicos por constancia) y políticas de reintento (exponential backoff).
* **Bad:** Incremento en la complejidad del sistema al requerir la mantención de un procesador/worker en segundo plano y monitoreo de colas/tablas de pendientes.

## Confirmation

* **Prueba de aislamiento de fallos:** Desconectar el servicio simulado de correo y verificar que la API de constancias responde exitosamente (HTTP 200/201) al usuario, almacenando la constancia y registrando la notificación con estado `PENDING`.
* **Prueba de idempotencia y duplicados:** Simular ejecuciones concurrentes del worker de notificaciones sobre el mismo registro y verificar que el proveedor externo solo recibe una solicitud por `constancia_id`.

## Pros and Cons of the Options

### Alternativa A: Comunicación directa con recuperación de fallos

* **Good:** Menor latencia percibida en el envío de correo cuando el servicio externo opera con normalidad.
* **Bad:** Si el servicio externo presenta alta latencia o caídas, se bloquean los hilos de ejecución principales de la aplicación, violando la restricción RT2.

### Alternativa B: Procesamiento asíncrono mediante eventos/registros persistidos

* **Good:** Desacoplamiento temporal y operativo total; cumple nativamente con RT2, RT3 y RT4.
* **Bad:** La entrega de notificaciones no es estrictamente inmediata en tiempo real, dependiendo de la frecuencia de ejecución del worker.

## More Information

Documento redactado para la evaluación del caso práctico ConstanciaUV.

### Revisión 1. Cambio en el proveedor de notificaciones

* **Cambio recibido:** El proveedor externo de notificaciones modificó su interfaz de comunicación y cambió la estructura de los códigos/mensajes de error de envío.
* **Impacto sobre la alternativa seleccionada:** El impacto se limita exclusivamente al componente independiente (worker/procesador de notificaciones) encargado de consumir los registros pendientes y realizar las llamadas HTTP. El dominio principal de generación de constancias no sufre ninguna modificación.
* **Evaluación de la decisión:** La decisión de mantener la **Alternativa B** permanece **vigente y respaldada**. Este cambio valida la elección, demostrando que el desacoplamiento arquitectónico aisla las modificaciones del proveedor externo sin afectar el núcleo del sistema.
* **Actualizaciones a consecuencias y comprobaciones:** Se debe actualizar la capa de integración del worker para mapear los nuevos códigos de error. Se agregará una prueba unitaria/de integración específica en el worker para validar la interpretación del nuevo formato de respuestas de error del proveedor.
