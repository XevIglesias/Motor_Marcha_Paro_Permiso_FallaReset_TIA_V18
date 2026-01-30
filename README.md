# Motor_Marcha_Paro_Permiso_FallaReset_TIA_V18
Control de motor en LAD (TIA Portal V18): Marcha/Paro, Permiso, Falla con latch y Reset (S7‑1500/PLCSIM)
# Control de motor con Marcha/Paro, Permiso y Falla con enclavamiento (Latch) en TIA Portal V18 — S7‑1500

Proyecto didáctico de PLC realizado por **Xavier Iglesias** en **Siemens TIA Portal V18 (Student)**.  
Implementa el control de un **motor** con lógica clásica de **Marcha/Paro**, condición de **Permiso**, gestión de **Falla** con **enclavamiento (latch)** y **Reset**. Incluye generación de **pulso de arranque por flanco** (sin usar `R_TRIG`).

---

## 1) Objetivo
- Arrancar/parar un motor mediante pulsadores:
  - **START**: arranque (por pulso)
  - **STOP**: paro (prioritario)
- Permitir el arranque solo si existe **PERMISO**.
- Parar el motor si ocurre **FALLA**.
- Enclavar la falla (`M_FALLA_LATCH`) hasta hacer **RESET**.
- Activar salidas de estado:
  - `Q_MOTOR`: salida al contactor del motor
  - `Q_RUN`: piloto/estado de marcha
  - `Q_ALARM`: alarma por falla

---

## 2) Entorno
- **PLC**: Siemens **S7‑1500** — (simulado con S7‑PLCSIM)
- **Software**: **TIA Portal V18 (Student)**
- **Lenguaje**: **LAD (Ladder)**

---

## 3) Descripción funcional (resumen)
- El motor puede arrancar si:
  - `I_STOP = 1` (STOP no pulsado, contacto NC lógico)
  - `I_PERMISO = 1`
  - `I_FALLA = 1` y `M_FALLA_LATCH = 0`
- Si `I_STOP = 0` → paro inmediato.
- Si `I_FALLA = 0` → se activa `M_FALLA_LATCH` y el motor se detiene.
- Para borrar la falla enclavada:
  - `I_RESET` genera un pulso de reset (por flanco) y desactiva `M_FALLA_LATCH` (según la condición definida en el programa).
- `Q_MOTOR` y `Q_RUN` siguen el estado `M_MOTOR_RUN`.
- `Q_ALARM` indica falla (típicamente `M_FALLA_LATCH`).

---

## 4) Flanco y pulso de START (sin `R_TRIG`)
Se implementa un detector de flanco positivo usando dos marcas:
- `M_START_EDGE`: “memoria” del estado anterior de `I_START`
- `M_START_PULSE`: pulso de 1 scan cuando `I_START` pasa de 0→1

> Nota: `M_START_PULSE` dura 1 ciclo de PLC, por eso puede no verse en la watch table.

---

## 5) Tabla de tags (PLC Tags)

> Tipos: todos **BOOL**.

### Entradas (Inputs)
| Tag | Dirección | Tipo | Descripción |
|---|---:|---|---|
| `I_START` | `I0.0` | BOOL | Pulsador START (arranque) |
| `I_STOP` | `I0.1` | BOOL | Pulsador STOP (paro, NC lógico) |
| `I_PERMISO` | `I0.2` | BOOL | Permiso/habilitación |
| `I_FALLA` | `I0.3` | BOOL | Señal de falla (OK=1 / Falla=0) |
| `I_RESET` | `I0.4` | BOOL | Pulsador RESET (borra latch de falla) |

### Salidas (Outputs)
| Tag | Dirección | Tipo | Descripción |
|---|---:|---|---|
| `Q_MOTOR` | `Q0.0` | BOOL | Salida motor/contactor |
| `Q_RUN` | `Q0.1` | BOOL | Piloto RUN (motor en marcha) |
| `Q_ALARM` | `Q0.2` | BOOL | Alarma de falla |

### Memorias (M)
| Tag | Dirección | Tipo | Descripción |
|---|---:|---|---|
| `M_MOTOR_RUN` | `M0.0` | BOOL | Estado interno de marcha (enclavamiento) |
| `M_FALLA_LATCH` | `M0.1` | BOOL | Falla enclavada (latch) |
| `M_START_EDGE` | `M0.2` | BOOL | Memoria de START (para flanco) |
| `M_RESET_EDGE` | `M0.3` | BOOL | Memoria de RESET (para flanco) |
| `M_START_PULSE` | `M0.4` | BOOL | Pulso de arranque (1 scan) |

---

## 6) Estructura de la programación (OB1)
La lógica está organizada conceptualmente en:

1. **Generación de pulsos (flancos)**
   - `M_START_PULSE` a partir de `I_START` y `M_START_EDGE`
   - (Opcional) pulso equivalente para reset con `M_RESET_EDGE`

2. **Gestión de fallas**
   - Activación de `M_FALLA_LATCH` al detectar `I_FALLA = 0`
   - Reset de `M_FALLA_LATCH` mediante `I_RESET` (según condiciones)

3. **Lógica Marcha/Paro (enclavamiento)**
   - Arranque con `M_START_PULSE`
   - Paro prioritario con `I_STOP`
   - Bloqueo por `I_PERMISO` y por falla (`I_FALLA` / `M_FALLA_LATCH`)

4. **Asignación de salidas**
   - `Q_MOTOR := M_MOTOR_RUN`
   - `Q_RUN := M_MOTOR_RUN`
   - `Q_ALARM := M_FALLA_LATCH` (o la condición de alarma definida)

---

## 7) Simulación y plan de pruebas (Test Plan)

### Prueba 1 — Arranque permitido
- Condición inicial: `I_STOP=1`, `I_PERMISO=1`, `I_FALLA=1`, `M_FALLA_LATCH=0`
- Acción: pulso `I_START`
- Esperado: `M_MOTOR_RUN=1`, `Q_MOTOR=1`, `Q_RUN=1`

### Prueba 2 — Paro prioritario
- Con motor en marcha
- Acción: `I_STOP=0`
- Esperado: `M_MOTOR_RUN=0`, `Q_MOTOR=0`, `Q_RUN=0`

### Prueba 3 — Bloqueo por PERMISO
- Acción: `I_PERMISO=0` y pulso `I_START`
- Esperado: no arranca (`M_MOTOR_RUN=0`)

### Prueba 4 — Falla con enclavamiento
- Con motor en marcha
- Acción: `I_FALLA=0`
- Esperado: `M_FALLA_LATCH=1`, `Q_ALARM=1`, motor parado (`Q_MOTOR=0`)

### Prueba 5 — Reset de falla
- Condición: `I_FALLA=1` (falla retirada)
- Acción: pulso `I_RESET` (y condiciones adicionales si el programa lo requiere, p. ej. STOP activo)
- Esperado: `M_FALLA_LATCH=0`, `Q_ALARM=0`

---

## Notas
Proyecto basado en un ejercicio didáctico visto en clase. Implementación y documentación realizadas por mí.
