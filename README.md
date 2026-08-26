# Incidente y corrección: reasignación de sesiones de mensajería (RappiPay)

## Resumen

Se desarrolló un feature en Salesforce para que, cuando un asesor (Personal Banker) pierde su sesión de mensajería mientras atiende a un cliente con un caso ya asociado, la conversación se reasigne **con prioridad** al asesor disponible más cercano (o a una cola de desborde si no hay nadie libre), en vez de esperar el enrutamiento normal de Omni-Channel (que puede tardar horas).

El primer despliegue a producción (25/08/2026) causó una incidencia real: sesiones de mensajería saltando entre varios asesores sin razón aparente. Se desactivó de inmediato, se investigó la causa raíz, y se rediseñó la solución dos veces hasta llegar a un mecanismo de detección validado contra datos reales de producción.

**Estado actual:** corrección validada en el ambiente de pruebas (UAT), pendiente de una prueba final en vivo de punta a punta antes de reactivar en producción.

---

## Línea de tiempo

| Fecha/hora | Evento |
|---|---|
| 2026-08-25 (mañana) | Despliegue inicial a producción del feature (detección basada en la presencia del asesor en Omni-Channel) |
| 2026-08-25 (tarde) | Se reportan 4 síntomas: mensajería y caso con dueños distintos, mensajería llegando a 2 asesores a la vez, sesiones con hasta 3 propietarios, asesores uniéndose a mensajerías de otros |
| 2026-08-25 | Se desactiva el feature en producción (interruptor de configuración) para detener el daño |
| 2026-08-25 | Diagnóstico: causa raíz identificada (ver Iteración 1) |
| 2026-08-25 | Corrección 1 desplegada y reactivada en producción |
| 2026-08-25 (varias horas después) | **Segundo incidente**: una sesión saltó entre 6 asesores distintos en menos de 2 horas, terminando bloqueada sin que nadie pudiera gestionarla |
| 2026-08-25 | Se desactiva de nuevo el feature en producción |
| 2026-08-25/26 | Investigación más profunda; rediseño (Iteración 2, luego Iteración 3) exclusivamente en UAT, sin volver a tocar producción |
| 2026-08-26 | Iteración 3 validada contra datos reales históricos de producción; pendiente prueba en vivo final |

---

## Iteración 1: detección por presencia del asesor (causó el primer incidente)

**Mecanismo:** un trigger en `UserServicePresence` detectaba cuando el registro "actual" de presencia de un asesor desaparecía (`IsCurrentState` pasando de `true` a `false`), interpretándolo como una desconexión real, y reasignaba todas las sesiones de mensajería activas de ese asesor.

**Por qué falló:** ese mismo evento (`IsCurrentState` pasando a `false`) ocurre **cada vez que un asesor cambia de estado normalmente** (por ejemplo, de "Activo" a "Almuerzo"), no solo cuando se desconecta de verdad. Como los asesores cambian de estado constantemente durante su turno, esto generaba reasignaciones indebidas de forma frecuente.

**Corrección intentada 1.1:** verificar, en el mismo instante, si el asesor todavía tenía algún registro de presencia "actual" antes de reasignar. Falló en una prueba real: Salesforce cierra el registro de presencia anterior *antes* de abrir el nuevo, dejando una ventana en la que ambos casos (cambio de estado normal vs. desconexión real) se ven idénticos.

**Corrección intentada 1.2 (la que causó el segundo incidente):** reasignar de inmediato en todos los casos (para no perder una desconexión real, ya que la plataforma reacciona en el mismo instante) y verificar unos segundos después, de forma asíncrona, si hay que revertir. Funcionó para desconexiones aisladas, pero cuando **varios asesores cambiaban de estado casi al mismo tiempo alrededor de la misma sesión**, se generaba una cadena de reasignaciones que terminó con una sesión saltando entre 6 personas.

**Conclusión:** la presencia del asesor (`UserServicePresence`) es una señal demasiado "ruidosa" — cambia con mucha frecuencia por razones que no tienen nada que ver con una desconexión real.

---

## Iteración 2: detección por estado de la sesión (descartada antes de llegar a producción)

**Hipótesis:** en vez de mirar la presencia del asesor, reaccionar directamente a que la propia `MessagingSession` cambie su campo `Status` a `'Inactive'`. Se verificó empíricamente en datos reales de producción: de 50 sesiones activas muestreadas, el 100% tenía al asesor propietario actualmente en línea; las únicas 2 sesiones en estado `Inactive` en ese momento correspondían a un asesor genuinamente desconectado.

**Por qué se descartó:** en una prueba real en UAT (desconexión real de un asesor con una conversación genuina), la sesión pasó a `Status = 'Waiting'`, **no** a `'Inactive'` como se esperaba. Es decir, la señal elegida no se disparaba en el escenario real que se quería resolver. Además, se confirmó que `MessagingSession.Status` no se puede establecer a `'Inactive'` mediante DML de Apex bajo ninguna circunstancia (ni insert ni update) — es un estado completamente gestionado por la plataforma, lo cual también complicaba las pruebas automatizadas.

**Este diseño nunca se desplegó a producción** — se detectó el problema en las pruebas de UAT.

---

## Iteración 3: detección por cambio de propietario hacia la cola de desborde (diseño actual)

**Mecanismo:** reaccionar cuando el **propietario** (`OwnerId`) de una `MessagingSession` que ya tiene un `Case` asociado cambia desde un propietario real hacia la cola de desborde configurada (`Desborde_mensajeria`, con etiqueta "PB Desborde mensajería"). En ese momento se marca la sesión con prioridad (`Prioridad__c`) y se busca de inmediato un asesor disponible; si no hay nadie libre, se deja en la cola marcada con prioridad para que se le asigne al primer asesor que quede libre.

**Validación:**
- Confirmado contra **varios ejemplos reales de producción** (usando el historial de campos de `MessagingSession`) que efectivamente, cuando un asesor pierde una conversación con caso asociado, el sistema nativo de Salesforce la mueve a través de un usuario de sistema ("Automated Process") hacia esa misma cola, antes de que otro asesor la reclame.
- El campo `OwnerId` no tiene ninguna restricción de la plataforma (a diferencia de la presencia y del estado), por lo que esta vez el código se puede probar con datos normales de prueba, sin los rodeos que fueron necesarios en las iteraciones 1 y 2.
- Cobertura de pruebas automatizadas: 100% en los componentes nuevos de detección (`MessagingSessionRequeuedHandler`, `MessagingSessionRequeuedTrigger`), 90%+ en el resto.
- Desplegado y probado en UAT sin errores.

**Pendiente antes de producción:**
- Prueba en vivo de punta a punta con una conversación real y una desconexión real (bloqueada temporalmente por un problema de conectividad de Omni-Channel en UAT, sin relación con este cambio).

---

## Componentes técnicos (estado actual, iteración 3)

| Componente | Tipo | Propósito |
|---|---|---|
| `MessagingSessionRequeuedTrigger` | Trigger (MessagingSession, after update) | Detecta el cambio de propietario hacia la cola |
| `MessagingSessionRequeuedHandler` | Clase Apex | Filtra el evento genuino (propietario cambió, tenía un propietario real antes, tiene caso asociado) |
| `OverflowReassignmentService` | Clase Apex | Lógica principal: verifica la cola de desborde, busca un asesor con cupo disponible, marca prioridad, reasigna |
| `AgentDisconnectOverflowHandler` | Clase Apex | Maneja el lado de "un asesor quedó libre" (para reclamar sesiones prioritarias ya parqueadas en la cola) |
| `Overflow_Reassignment_Config__mdt` | Custom Metadata Type | Configuración: cola, canal de servicio, valor de prioridad, interruptor de activación (`Is_Active__c`) |
| `MessagingSession.Prioridad__c` | Campo personalizado | Marca la sesión como prioritaria |
| `MessagingSession.Fecha_transferencia_a_cola__c` | Campo personalizado | Registra cuándo se marcó la prioridad |

**Componentes retirados durante el proceso** (ya no existen en el código): `UserServicePresenceTrigger` (lado de detección de desconexión), `DisconnectFalsePositiveCorrector`, `MessagingSessionInactivityHandler`, `MessagingSessionInactivityTrigger`, `InactiveSessionFalsePositiveCorrector`.

---

## Interruptor de seguridad

Todo el feature se activa/desactiva con un solo campo: `Overflow_Reassignment_Config__mdt.Is_Active__c`. Se usó dos veces durante este proceso para detener el daño de forma inmediata sin necesitar un despliegue de código. **Actualmente está desactivado en producción.**

---

## Descubrimientos adicionales (contexto, no bugs de este feature)

- Existe un Flow activo en producción (`Agent_Work_Propiedad_casos_asociados_a_sesiones_mensajeria`) que sincroniza automáticamente el propietario del Caso para que coincida con el de la sesión de mensajería, cada vez que un asesor nuevo queda realmente asignado — por lo que la discrepancia "mensajería con un asesor, caso con otro" se autocorrige sola en la mayoría de los casos.
- Se identificó (sin resolver) un caso donde un asesor perdió una sesión después de solo 42 segundos de tenerla, sin que su presencia cambiara ni hubiera indicios de lentitud en responder — causa desconocida, no relacionada con este feature (que estaba desactivado en ese momento).
- Problema de conectividad de Omni-Channel en UAT (agentes que la interfaz muestra como "Activo" pero que el backend ya no ve conectados) — no relacionado con este feature, requiere cerrar sesión y volver a entrar.

---

## Próximos pasos

1. Resolver el problema de conectividad de Omni-Channel en UAT.
2. Ejecutar la prueba en vivo de punta a punta (mensaje real → asignación → desconexión real → verificar reasignación con prioridad).
3. Desplegar a producción en dos pasos: código primero (sin activar), luego activar el interruptor como paso separado con monitoreo inmediato.
