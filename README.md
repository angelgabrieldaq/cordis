# Cordis · by Zenoxia

> Plataforma de orquestación inteligente para Servicios de Urgencias hospitalarios.  
> Trazabilidad en tiempo real · Reducción de carga cognitiva · Insights operativos.

---

## ¿Qué es Cordis?

Cordis es una capa de gestión operativa que se monta sobre los sistemas existentes del hospital (HIS, gestión de camas, gestión administrativa). No reemplaza ningún sistema — hace accionable en tiempo real lo que hoy es estático.

**Cordis es a los hospitales lo que un sistema de control de tráfico aéreo es a los aeropuertos:** los aviones (pacientes) y las pistas (camas, equipos, médicos) ya existían. Cordis orquesta el flujo.

> Cordis es un producto de **Zenoxia**.  
> El ecosistema quirúrgico de Zenoxia es **Kairos** (producto separado).

---

## El problema que resuelve

Los hospitales hoy tienen múltiples sistemas que no se hablan entre sí:

- El HIS tiene los datos clínicos pero no muestra flujos en tiempo real
- La gestión de camas no sabe qué estudios están pendientes
- El recepcionista no ve qué box está en limpieza
- El técnico de imágenes no sabe la prioridad clínica del paciente
- Nadie tiene visibilidad del tiempo real de espera por nivel de triage

**Cordis conecta todos estos puntos y los hace visibles para cada rol.**

---

## Arquitectura

```
HIS · Gestión de Camas · Gestión Administrativa
           (sistemas legacy — fuentes de verdad)
                        ↓
              Cordis Integration Layer
         (HL7, FHIR, webhooks, polling)
                        ↓
           Cordis Operational Engine
    (cola dinámica, SLA, alertas, IA de triage)
                        ↓
         Interfaces Cordis (este repositorio)
```

### Origen de datos en la UI

Cada dato en la interfaz muestra su origen:

| Badge | Significado |
|-------|-------------|
| `· HIS sync` | Dato sincronizado desde el HIS |
| `· Cordis Engine` | Dato generado por la lógica de Cordis |
| `· Padrón OS` | Dato validado contra padrón de obra social |
| `· Manual` | Dato ingresado manualmente por el operador |

---

## Módulos y Roles

### 🏥 Guardia

| Módulo | Rol | Estado |
|--------|-----|--------|
| Command Center · Guardia | Recepcionista de Guardia | ✅ Prototipo |
| Mapa de Camas | Recepcionista de Guardia | ✅ Integrado en Command Center |
| Enfermeros de Guardia | Enfermero/a de Triage | 🔄 En desarrollo |
| App del Paciente | Paciente | 🔄 Pendiente |

### 🩻 Imágenes

| Módulo | Rol | Estado |
|--------|-----|--------|
| Command Center · Imágenes | Recepcionista de Imágenes | 🔄 Pendiente |
| Worklist Técnico · TC | Técnico Radiólogo | ✅ Prototipo |

### 🚑 Camilleros

| Módulo | Rol | Estado |
|--------|-----|--------|
| Central de Camilleros | Coordinador de Camilleros | 🔄 Pendiente |
| Camilleros Mobile | Camillero | 🔄 Pendiente |

---

## Flujo del paciente

```
ANTES DE LLEGAR
App del Paciente → Pre-triage sintomático → Cola dinámica estimada
                                                      ↓
EN RECEPCIÓN
Command Center · Guardia → Escaneo DNI → Validación cobertura
                        → Episodio oficial → Pulsera identificatoria
                                                      ↓
TRIAGE
Enfermeros de Guardia → Signos vitales → Motivo de consulta
                     → IA sugiere nivel → Enfermero confirma
                                                      ↓
ATENCIÓN MÉDICA
Médico llama al paciente → Llamador automático mueve al paciente
                        → Box asignado visible en Command Center
                                                      ↓
ESTUDIOS (si aplica)
Orden médica → Command Center · Imágenes → Técnico ejecuta
            → Resultado disponible → Médico aprueba → Entrega
                                                      ↓
ALTA / INTERNACIÓN
Médico firma alta → Alta pendiente en Mapa de Camas
                 → Recepción confirma egreso → Limpieza → Box libre
```

---

## Reglas de negocio implementadas

| ID | Regla | Módulo |
|----|-------|--------|
| RN-01 | Datos filiatorios inmutables (DNI, nombre, fecha nac., sexo) | Recepción |
| RN-02 | Paciente sin cobertura ingresa como particular transitorio con circuito de reintegro | Recepción |
| RN-03 | Modo contingencia offline — formularios imprimibles para carga posterior | Recepción |
| RN-04 | La IA solo sugiere nivel de triage — el enfermero valida y confirma | Triage |
| RN-05 | Las órdenes médicas quedan en borrador hasta aprobación del médico | Triage / Médico |
| RN-06 | SLA inmediato para urgencias · SLA 6-48hs para estudios ambulatorios | Imágenes |
| RN-07 | Ningún estudio pasa a "A entregar" sin aprobación médica previa | Imágenes |
| RN-08 | El llamador del médico mueve automáticamente al paciente en el tablero | Command Center |
| RN-09 | Ciclo hotelero de camas: Libre → Ocupada → Alta → Limpieza → Lista → Libre | Mapa de Camas |
| RN-10 | Toda transición de estado registra actor, rol y timestamp | Todos |

---

## Design System

| Archivo | Descripción |
|---------|-------------|
| `design-tokens-v2.css` | Fuente de verdad de colores, tipografía, espaciado e iconografía |
| `kairos-icons.svg` | Sprite SVG con 37 íconos del sistema (outline, `currentColor`) |

### Paleta de triage

| Nivel | Color | Uso |
|-------|-------|-----|
| N1 · Resucitación | `#C0392B` Rojo | Band, pill, badge |
| N2 · Emergencia | `#D4680A` Naranja | Band, pill, badge |
| N3 · Urgencia | `#A07C00` Amarillo oscuro | Band, pill, badge |
| N4 · Menor | `#2E7D4F` Verde | Band, pill, badge |
| N5 · No urgente | `#1565A8` Azul | Band, pill, badge |

### Estados de cama

| Estado | Token | Significado |
|--------|-------|-------------|
| Libre y limpia | `--cama-libre` | Disponible para asignar |
| Ocupada | `--cama-ocup` | Paciente activo |
| Alta pendiente | `--cama-alta` | Médico firmó, paciente aún en cama |
| En limpieza | `--cama-limp` | Personal de limpieza trabajando |
| Lista para usar | `--cama-lista` | Limpieza hecha, esperando confirmación |
| Bloqueada | `--cama-bloq` | Mantenimiento o falla reportada |

---

## Estructura del repositorio

```
cordis-zenoxia/
│
├── README.md
├── index.html                        ← Navegador del ecosistema
├── design-tokens-v2.css              ← Design Tokens (fuente de verdad)
├── kairos-icons.svg                  ← Icon Sprite
│
├── guardia/
│   ├── command-center.html           ✅ Command Center · Guardia
│   ├── enfermeros-guardia.html       🔄 En desarrollo
│   └── app-paciente.html             🔄 Pendiente
│
├── imagenes/
│   ├── command-center-imagenes.html  🔄 Pendiente
│   └── worklist-tc.html              ✅ Worklist Técnico · TC
│
└── camilleros/
    ├── central-camilleros.html       🔄 Pendiente
    └── camilleros-mobile.html        🔄 Pendiente
```

---

## Convenciones de código

- **Sin frameworks externos** — HTML, CSS nativo y Vanilla JS
- **Sin colores hardcodeados** — siempre `var(--token)`
- **Sin tamaños de ícono hardcodeados** — siempre `var(--icon-md)` etc.
- **SVG íconos** — siempre `stroke="currentColor"` `fill="none"`
- **Datos con origen** — cada dato muestra su fuente (HIS, Cordis, Manual)
- **IA como asistente** — nunca toma decisiones clínicas sin validación humana

---

## Estado del proyecto

**Versión:** 0.1.0 — Prototipo de ideación  
**Fase actual:** Diseño de interfaces y flujos operativos  
**Próximo hito:** Triage de Enfermería + App del Paciente

---

*Cordis · by Zenoxia — Todos los derechos reservados*
