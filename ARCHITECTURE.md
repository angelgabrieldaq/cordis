# CORDIS · Documento de Arquitectura
## Zenoxia Suite — Fase 2 · Backend & Cordis Engine

**Versión:** 1.0  
**Estado:** Borrador técnico — base para desarrollo  
**Fecha:** Mayo 2026  
**Contexto:** Este documento unifica el frontend completo de la Fase 1 con los contratos de datos, modelo de base de datos y reglas del motor de orquestación. Está diseñado para que cualquier desarrollador o equipo de ingeniería pueda construir el backend sin ambigüedades.

---

## ÍNDICE

1. [Visión general del sistema](#1-visión-general)
2. [Stack tecnológico recomendado](#2-stack)
3. [Modelo de base de datos](#3-base-de-datos)
4. [Máquina de estados por entidad](#4-estados)
5. [Reglas del Cordis Engine](#5-cordis-engine)
6. [Contratos de eventos (JSON Payloads)](#6-eventos)
7. [API REST — Endpoints por módulo](#7-api)
8. [Integración con HIS legacy](#8-his)
9. [Design Tokens — referencia para el frontend](#9-design-tokens)
10. [Métricas operativas y KPIs](#10-kpis)
11. [Datos operativos reales — contexto de validación](#11-datos-reales)

---

## 1. VISIÓN GENERAL

### 1.1 Paradigma

Cordis **no reemplaza** el HIS (Hospital Information System) ni los sistemas legacy existentes. Opera como una capa superior:

```
┌─────────────────────────────────────────────────┐
│                    USUARIOS                     │
│  Recepcionista · Enfermero · Médico · Camillero │
│              Paciente · Técnico                 │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│            CORDIS FRONTEND (Fase 1)             │
│  Command Center · Triage · App Paciente         │
│  Imágenes · Worklist · Camilleros               │
└──────────────────┬──────────────────────────────┘
                   │ WebSocket / REST API
┌──────────────────▼──────────────────────────────┐
│           CORDIS ENGINE (Fase 2)                │
│  Operational Engine · Event Bus · IA            │
│  Reglas de negocio · Cálculo de fatiga          │
│  Asignación inteligente · SLA monitor           │
└──────────────────┬──────────────────────────────┘
                   │ Integration Layer
┌──────────────────▼──────────────────────────────┐
│              HIS LEGACY / SISTEMAS              │
│  Historia clínica · Facturación · PACS          │
└─────────────────────────────────────────────────┘
```

### 1.2 Principios de diseño

| Principio | Descripción |
|-----------|-------------|
| **Eventos inmutables** | El estado actual de cualquier entidad es el último evento registrado. No se actualiza estado — se agrega un evento nuevo. |
| **IA asiste, humano decide** | Toda sugerencia del Engine (triage, asignación, fatiga) requiere confirmación humana. Ninguna acción clínica es automática. |
| **Trazabilidad total** | Cada evento registra: actor, rol, timestamp, dispositivo. Permite auditoría completa y cálculo de KPIs. |
| **Tiempo real** | Cualquier cambio de estado se propaga a todos los clientes conectados vía WebSocket sin recarga de pantalla. |
| **Horizontal** | Cordis no tiene configuraciones específicas de ninguna institución. Los datos maestros (boxes, equipos, personal) son configurables por institución. |
| **Identidad por DNI** | El DNI es el identificador universal del paciente. El código de barras PDF417 del dorso del DNI argentino permite escaneo inmediato. |

---

## 2. STACK TECNOLÓGICO RECOMENDADO

### 2.1 Backend

```
Lenguaje:     Python 3.12+
Framework:    FastAPI (async, alto rendimiento, OpenAPI automático)
Base datos:   PostgreSQL 16+ (JSONB para metadata flexible)
ORM:          SQLAlchemy 2.0 (async) + Alembic (migraciones)
WebSocket:    FastAPI WebSocket nativo + Redis Pub/Sub
Cache:        Redis 7 (estado en tiempo real, colas, pub/sub)
Queue:        Celery + Redis (tareas asíncronas: notificaciones, webhooks)
Auth:         JWT + OAuth2 (FastAPI security)
```

### 2.2 Frontend (Fase 1 — completado)

```
HTML5 puro + CSS nativo (Design Tokens v2.0)
Vanilla JS — sin frameworks
SVG icons inline (sprite kairos-icons.svg)
Sin dependencias externas
```

### 2.3 Infraestructura

```
Containerización:  Docker + Docker Compose
CI/CD:             GitHub Actions
Entorno:           Linux (Ubuntu 22.04 LTS)
Reverse proxy:     Nginx
SSL:               Let's Encrypt (certbot)
```

---

## 3. BASE DE DATOS

### 3.1 Principio de eventos inmutables

```sql
-- El estado actual de un episodio siempre es:
SELECT estado_resultante
FROM hitos_tiempo
WHERE episodio_id = 'EP-88421'
ORDER BY timestamp DESC
LIMIT 1;

-- NUNCA: UPDATE episodios SET estado = 'EN_ESPERA'
-- SIEMPRE: INSERT INTO hitos_tiempo (evento, estado_resultante...)
```

### 3.2 Entidades core

```sql
-- ─────────────────────────────────────────────
-- PACIENTES (datos inmutables de identidad)
-- ─────────────────────────────────────────────
CREATE TABLE pacientes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dni             VARCHAR(8) UNIQUE NOT NULL,   -- sin puntos
    apellidos       VARCHAR(100) NOT NULL,
    nombres         VARCHAR(100) NOT NULL,
    fecha_nacimiento DATE NOT NULL,
    sexo_registral  VARCHAR(20) NOT NULL,         -- RN-01: inmutable
    genero          VARCHAR(40),
    -- Contacto
    pais            VARCHAR(60) DEFAULT 'Argentina',
    provincia       VARCHAR(60),
    ciudad          VARCHAR(80),
    direccion       VARCHAR(200),
    telefono        VARCHAR(30),
    email           VARCHAR(120),
    -- Sistema
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    his_id          VARCHAR(50),                  -- ID en el HIS legacy
    CONSTRAINT dni_formato CHECK (dni ~ '^\d{7,8}$')
);

-- ─────────────────────────────────────────────
-- COBERTURAS (datos mutables — en tabla separada)
-- ─────────────────────────────────────────────
CREATE TABLE coberturas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    paciente_id     UUID REFERENCES pacientes(id),
    obra_social     VARCHAR(100),
    nro_afiliado    VARCHAR(50),
    plan            VARCHAR(50),
    validada        BOOLEAN DEFAULT FALSE,
    valid_en        TIMESTAMPTZ,
    valid_actor_id  UUID,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- ─────────────────────────────────────────────
-- EPISODIOS CLÍNICOS
-- Un episodio = una visita a guardia
-- ─────────────────────────────────────────────
CREATE TABLE episodios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    numero          VARCHAR(20) UNIQUE NOT NULL,  -- ej: GRD-20260502-0042
    paciente_id     UUID REFERENCES pacientes(id) NOT NULL,
    cobertura_id    UUID REFERENCES coberturas(id),
    -- Estado actual (calculado desde hitos_tiempo)
    -- NOTA: Este campo es un cache desnormalizado para
    -- queries rápidas. La fuente de verdad es hitos_tiempo.
    estado_cache    VARCHAR(50) DEFAULT 'EN_TRIAGE',
    -- Origen del episodio
    origen          VARCHAR(20) DEFAULT 'GUARDIA', -- GUARDIA | IMAGENES | AMBULATORIO
    -- Timestamps clave (calculados, para índices)
    ingreso_at      TIMESTAMPTZ DEFAULT NOW(),
    alta_at         TIMESTAMPTZ,
    -- Sistema
    his_episodio_id VARCHAR(50),
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- ─────────────────────────────────────────────
-- HITOS DE TIEMPO — El corazón del sistema
-- Tabla append-only: cada cambio de estado es un nuevo registro
-- ─────────────────────────────────────────────
CREATE TABLE hitos_tiempo (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episodio_id     UUID REFERENCES episodios(id) NOT NULL,
    -- Evento
    hito_codigo     VARCHAR(60) NOT NULL,
    estado_resultante VARCHAR(60) NOT NULL,
    -- Actor
    actor_id        UUID REFERENCES usuarios(id),
    actor_rol       VARCHAR(40),
    actor_nombre    VARCHAR(100),
    -- Contexto
    recurso_id      UUID,                         -- cama, equipo, box
    notas           TEXT,
    metadata        JSONB DEFAULT '{}',           -- datos adicionales flexibles
    -- Sistema
    timestamp       TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    device_id       VARCHAR(100),
    source          VARCHAR(20) DEFAULT 'CORDIS'  -- CORDIS | HIS | MANUAL
);

CREATE INDEX idx_hitos_episodio ON hitos_tiempo(episodio_id, timestamp DESC);
CREATE INDEX idx_hitos_codigo ON hitos_tiempo(hito_codigo);

-- ─────────────────────────────────────────────
-- RECURSOS FÍSICOS
-- Entidad unificada para Camas, Boxes y Equipos
-- ─────────────────────────────────────────────
CREATE TABLE recursos_fisicos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institucion_id  UUID NOT NULL,
    codigo          VARCHAR(20) NOT NULL,         -- ej: SH-1, OBS-04, TC-1
    nombre          VARCHAR(100) NOT NULL,
    tipo            VARCHAR(30) NOT NULL,         -- CAMA | BOX | EQUIPO_IMAGEN
    subtipo         VARCHAR(30),                  -- SHOCK | INFUSION | OBSERVACION | TC | RX | ECO | RM
    ubicacion       VARCHAR(100),                 -- ej: Planta Baja · Ala Sur
    -- Estado (cache desnormalizado, fuente: hitos_tiempo de recurso)
    estado_cache    VARCHAR(30) DEFAULT 'LIBRE',
    activo          BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(institucion_id, codigo)
);

-- ─────────────────────────────────────────────
-- TRIAGE LOGS
-- Separado de hitos para queries de analytics
-- ─────────────────────────────────────────────
CREATE TABLE triage_logs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episodio_id         UUID REFERENCES episodios(id) NOT NULL,
    -- Nivel
    nivel               INTEGER NOT NULL CHECK (nivel BETWEEN 1 AND 5),
    nivel_ia_sugerido   INTEGER CHECK (nivel_ia_sugerido BETWEEN 1 AND 5),
    nivel_confirmado_por UUID REFERENCES usuarios(id),
    -- Vitales
    ta_sistolica        INTEGER,
    ta_diastolica       INTEGER,
    frecuencia_cardiaca INTEGER,
    frecuencia_respiratoria INTEGER,
    temperatura         DECIMAL(4,1),
    saturacion_o2       INTEGER,
    -- Escalas
    escala_dolor        INTEGER CHECK (escala_dolor BETWEEN 0 AND 10),
    escala_ansiedad     INTEGER CHECK (escala_ansiedad BETWEEN 0 AND 10),
    -- Motivo
    motivo_texto        TEXT,
    motivo_transcripcion TEXT,                    -- voz → texto
    cie10_codigo        VARCHAR(10),
    cie10_descripcion   VARCHAR(200),
    -- MTS + ESI
    mts_motivo          VARCHAR(60),
    mts_discriminadores JSONB DEFAULT '[]',       -- lista de discriminadores activos
    esi_nivel           INTEGER CHECK (esi_nivel BETWEEN 1 AND 5),
    -- IA
    ia_razonamiento     TEXT,
    ia_modelo           VARCHAR(50),
    -- Sistema
    created_at          TIMESTAMPTZ DEFAULT NOW()
);

-- ─────────────────────────────────────────────
-- TRASLADOS
-- ─────────────────────────────────────────────
CREATE TABLE traslados (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episodio_id     UUID REFERENCES episodios(id),
    -- Ruta
    origen          VARCHAR(100) NOT NULL,
    destino         VARCHAR(100) NOT NULL,
    -- Equipamiento y fatiga
    equipo          JSONB DEFAULT '[]',           -- ['CAMILLA', 'OXIGENO', 'PREP_TC']
    paciente_pesado BOOLEAN DEFAULT FALSE,
    puntos_fatiga   INTEGER DEFAULT 1,            -- 1=silla, 2=camilla, 3=camilla+O2+pesado
    prioridad       VARCHAR(20) DEFAULT 'NORMAL', -- NORMAL | URGENTE
    -- Asignación
    solicitante_id  UUID REFERENCES usuarios(id),
    camillero_id    UUID REFERENCES usuarios(id),
    -- Estado (cache)
    estado          VARCHAR(30) DEFAULT 'PENDIENTE_ASIGNACION',
    -- Timestamps
    solicitado_at   TIMESTAMPTZ DEFAULT NOW(),
    asignado_at     TIMESTAMPTZ,
    inicio_at       TIMESTAMPTZ,
    llegada_at      TIMESTAMPTZ,
    completado_at   TIMESTAMPTZ,
    -- Origen del traslado
    auto_retorno    BOOLEAN DEFAULT FALSE,        -- generado por Engine al completar estudio
    traslado_ida_id UUID                          -- referencia al traslado de ida
);

-- ─────────────────────────────────────────────
-- TURNOS DE CAMILLEROS (carga laboral)
-- ─────────────────────────────────────────────
CREATE TABLE turnos_camilleros (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    camillero_id    UUID REFERENCES usuarios(id) NOT NULL,
    inicio          TIMESTAMPTZ NOT NULL,
    fin_esperado    TIMESTAMPTZ NOT NULL,
    fin_real        TIMESTAMPTZ,
    duracion_horas  INTEGER DEFAULT 7,            -- 7hs promedio, 14hs fines de semana
    -- Carga acumulada (calculada en tiempo real por Engine)
    puntos_fatiga_acum INTEGER DEFAULT 0,
    traslados_completados INTEGER DEFAULT 0,
    traslados_camilla INTEGER DEFAULT 0,
    traslados_silla INTEGER DEFAULT 0,
    traslados_uti   INTEGER DEFAULT 0,            -- traslados a áreas críticas
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- ─────────────────────────────────────────────
-- ESTUDIOS DE IMÁGENES
-- ─────────────────────────────────────────────
CREATE TABLE estudios_imagen (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episodio_id     UUID REFERENCES episodios(id),
    -- Pedido
    modalidad       VARCHAR(10) NOT NULL,         -- TC | RX | ECO | RM | LAB
    estudio_nombre  VARCHAR(200) NOT NULL,
    protocolo       VARCHAR(100),
    tipo_sla        VARCHAR(20) DEFAULT 'AMBULATORIO', -- URGENTE | AMBULATORIO
    solicitante_id  UUID REFERENCES usuarios(id),
    -- Asignación
    equipo_id       UUID REFERENCES recursos_fisicos(id),
    tecnico_id      UUID REFERENCES usuarios(id),
    -- Timestamps (inmutables — Room Turnover Time)
    solicitado_at   TIMESTAMPTZ DEFAULT NOW(),
    turno_programado TIMESTAMPTZ,
    entrada_equipo_at TIMESTAMPTZ,               -- inicio real en máquina
    salida_equipo_at  TIMESTAMPTZ,               -- fin real en máquina
    aprobado_at     TIMESTAMPTZ,
    entregado_at    TIMESTAMPTZ,
    -- Estado
    estado          VARCHAR(30) DEFAULT 'PENDIENTE',
    notas_tecnico   TEXT,
    -- No-show
    no_show         BOOLEAN DEFAULT FALSE,
    no_show_at      TIMESTAMPTZ,
    no_show_actor_id UUID
);

-- ─────────────────────────────────────────────
-- USUARIOS
-- ─────────────────────────────────────────────
CREATE TABLE usuarios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institucion_id  UUID NOT NULL,
    nombre          VARCHAR(100) NOT NULL,
    apellido        VARCHAR(100) NOT NULL,
    email           VARCHAR(120) UNIQUE NOT NULL,
    rol             VARCHAR(30) NOT NULL,
    -- RECEPCIONISTA | ENFERMERO | MEDICO | TECNICO_IMAGEN
    -- CAMILLERO | COORDINADOR_CAMILLEROS | JEFE_GUARDIA | ADMIN
    activo          BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 4. MÁQUINA DE ESTADOS POR ENTIDAD

### 4.1 Estados del episodio clínico

```
EN_TRIAGE
    │ HITO: TRIAGE_COMPLETADO
    ▼
EN_ESPERA_MEDICA
    │ HITO: MEDICO_LLAMADO (llamador activado)
    ▼
EN_ATENCION
    │ HITO: ALTA_INDICADA
    ├──────────────────────────────────┐
    │ HITO: INTERNACION_INDICADA        │
    ▼                                  ▼
ALTA                          PEND_INTERNACION_{DESTINO}
                              (UTI | IG | UCO | TERAPIA | PEDIA)
                                        │ HITO: CAMA_ASIGNADA + TRASLADO_COMPLETADO
                                        ▼
                                    INTERNADO
```

### 4.2 Estados del recurso físico (cama/box)

```
LIBRE ──► OCUPADA ──► PEND_INTERNACION ──► LIMPIEZA ──► LISTA ──► LIBRE
                             │                    ▲
                     ALTA_MEDICA ────────────────┘
```

Regla RN-09: cada transición registra actor, rol y timestamp en `hitos_tiempo`.

### 4.3 Estados del estudio de imágenes

```
PENDIENTE ──► EN_EJECUCION ──► PEND_APROBACION ──► APROBADO ──► ENTREGADO
                                                              │
                                                        NO_SHOW (bypass)
```

### 4.4 Estados del traslado

```
PENDIENTE_ASIGNACION
    │ Engine sugiere camillero
    ▼
ASIGNADO
    │ Camillero confirma en app
    ▼
EN_TRANSITO_BUSQUEDA  (camillero va a buscar)
    │ Camillero confirma recogida
    ▼
EN_TRANSITO_ENTREGA   (camillero lleva al destino)
    │ Camillero confirma entrega
    ▼
COMPLETADO
    │ Engine dispara traslado de retorno automático (si aplica)
    ▼
[TRASLADO_RETORNO creado automáticamente]
```

### 4.5 Hitos por módulo — catálogo completo

```python
# Guardia
HITOS_GUARDIA = [
    'INGRESO_RECEPCION',        # paciente llega al mostrador
    'PULSERA_EMITIDA',          # recepción valida y emite pulsera
    'TRIAGE_INICIADO',          # enfermero abre evaluación
    'TRIAGE_COMPLETADO',        # enfermero confirma nivel N1-N5
    'MEDICO_LLAMADO',           # médico activa llamador → mueve al tablero
    'ATENCION_INICIADA',        # médico abre el episodio
    'ESTUDIO_SOLICITADO',       # médico pide estudio de imagen/lab
    'INTERNACION_INDICADA',     # médico indica internación con destino
    'CAMA_ASIGNADA',            # se confirma cama en destino
    'ALTA_INDICADA',            # médico da el alta
    'EGRESO_FISICO',            # paciente sale físicamente
    'LIMPIEZA_INICIADA',        # box entra en limpieza
    'LIMPIEZA_COMPLETADA',      # box queda lista
]

# Imágenes
HITOS_IMAGENES = [
    'ESTUDIO_RECIBIDO',         # recepción de imágenes recibe la orden
    'EQUIPO_ASIGNADO',          # se derivó al equipo correspondiente
    'PACIENTE_EN_SALA',         # paciente llegó a la sala del equipo
    'ESTUDIO_INICIADO',         # técnico confirma inicio (timestamp inmutable)
    'ESTUDIO_COMPLETADO',       # técnico confirma fin (RTT calculable)
    'PEND_APROBACION_MEDICA',   # estudio en cola de aprobación
    'APROBADO_POR_MEDICO',      # médico aprueba resultado
    'RESULTADO_ENTREGADO',      # resultado entregado al paciente/médico
    'NO_SHOW_REGISTRADO',       # paciente no se presentó
]

# Traslados
HITOS_TRASLADOS = [
    'TRASLADO_SOLICITADO',
    'TRASLADO_ASIGNADO',
    'CAMILLERO_EN_CAMINO',
    'PACIENTE_RECOGIDO',
    'PACIENTE_ENTREGADO',
    'TRASLADO_COMPLETADO',
    'PROBLEMA_REPORTADO',       # eventualidad en app mobile
]
```

---

## 5. CORDIS ENGINE

### 5.1 Módulo de SLA — Monitor en tiempo real

```python
# SLA por nivel de triage (estándares MTS internacionales)
SLA_TRIAGE = {
    1: 0,    # N1 Resucitación: inmediato
    2: 15,   # N2 Emergencia: 15 minutos
    3: 30,   # N3 Urgencia: 30 minutos
    4: 120,  # N4 Menor: 2 horas
    5: 240,  # N5 No urgente: 4 horas
}

# SLA por tipo de estudio
SLA_IMAGENES = {
    'URGENTE':      0,    # inmediato
    'AMBULATORIO':  360,  # 6 horas mínimo, máximo 48hs según estudio
}

# Lógica de escalamiento
def check_sla(episodio_id: str) -> SLAStatus:
    """
    Corre cada 60 segundos por Celery Beat.
    Si el tiempo transcurrido supera el SLA del nivel,
    emite alerta PUSH al jefe de guardia.
    """
    ultimo_hito = get_ultimo_hito(episodio_id)
    nivel = get_nivel_triage(episodio_id)
    tiempo_espera = (now() - ultimo_hito.timestamp).minutes
    sla_limite = SLA_TRIAGE[nivel]

    if sla_limite > 0 and tiempo_espera > sla_limite:
        emit_event('SLA_VENCIDO', {
            'episodio_id': episodio_id,
            'nivel': nivel,
            'minutos_vencido': tiempo_espera - sla_limite,
        })
```

### 5.2 Módulo de fatiga de camilleros

```python
# Tabla de puntos de fatiga por tipo de traslado
PUNTOS_FATIGA = {
    'SILLA_RUEDAS':         1,
    'SILLA_ORTOPEDICA':     1,
    'CAMILLA':              2,
    'CAMILLA_OXIGENO':      2,
    'CAMILLA_OXIGENO_PESADO': 3,  # máximo esfuerzo
    'PREP_TC':              1,    # traslado especial para tomógrafo
    'PREP_RM':              1,    # traslado especial para resonador
}

UMBRAL_FATIGA_TURNO_7H  = 18   # puntos máximos en turno de 7hs
UMBRAL_FATIGA_TURNO_14H = 36   # puntos máximos en turno de 14hs

def calcular_fatiga(camillero_id: str) -> FatigaStatus:
    turno = get_turno_activo(camillero_id)
    traslados = get_traslados_turno(camillero_id, turno.inicio)
    puntos = sum(PUNTOS_FATIGA.get(t.equipo_tipo, 1) for t in traslados)
    limite = UMBRAL_FATIGA_TURNO_14H if turno.duracion_horas == 14 \
             else UMBRAL_FATIGA_TURNO_7H
    return FatigaStatus(
        puntos=puntos,
        limite=limite,
        porcentaje=round((puntos / limite) * 100),
        nivel='LIMITE' if puntos >= limite * 0.9
              else 'ALTA'   if puntos >= limite * 0.7
              else 'MODERADA' if puntos >= limite * 0.4
              else 'LEVE',
        sugerir_descanso=puntos >= limite * 0.8
    )
```

### 5.3 Módulo de asignación de camilleros

**Principio de equidad primero**: el Engine sugiere, el coordinador humano decide. Los datos históricos informan los KPIs pero no crean sesgos de asignación automática.

```python
def sugerir_camillero(traslado: Traslado) -> list[Sugerencia]:
    """
    Devuelve lista de camilleros ordenados por disponibilidad y carga.
    El coordinador elige — el Engine no asigna automáticamente.
    """
    camilleros_activos = get_camilleros_turno_activo()

    sugerencias = []
    for cam in camilleros_activos:
        fatiga = calcular_fatiga(cam.id)
        estado = get_estado_camillero(cam.id)

        if estado != 'DISPONIBLE':
            continue  # nunca interrumpir un traslado en curso

        sugerencias.append(Sugerencia(
            camillero=cam,
            fatiga=fatiga,
            disponible=True,
            razonamiento=build_razonamiento(cam, fatiga, traslado),
        ))

    # Ordenar por menor carga — equidad primero
    return sorted(sugerencias, key=lambda s: s.fatiga.puntos)

def build_razonamiento(cam, fatiga, traslado) -> str:
    msgs = [f"{cam.nombre_completo} disponible, carga {fatiga.nivel.lower()}."]
    if fatiga.sugerir_descanso:
        msgs.append("⚠ Considerar descanso — carga alta en el turno.")
    return " ".join(msgs)
```

### 5.4 Módulo de retorno automático (eliminación del viaje vacío)

```python
def on_estudio_completado(estudio_id: str, actor_id: str):
    """
    Se dispara cuando el técnico marca 'Estudio completado' en el Worklist.
    Genera automáticamente un traslado de retorno.
    """
    estudio = get_estudio(estudio_id)
    traslado_original = get_traslado_hacia_estudio(estudio.episodio_id)

    # Crear traslado de retorno
    traslado_retorno = Traslado(
        episodio_id=estudio.episodio_id,
        origen=estudio.equipo.ubicacion,
        destino=traslado_original.origen,  # vuelve al box de origen
        equipo=traslado_original.equipo,
        prioridad='NORMAL',
        auto_retorno=True,
        traslado_ida_id=traslado_original.id,
    )
    db.add(traslado_retorno)

    # Emitir evento al Event Bus → Central de Camilleros
    emit_event('TRASLADO_SOLICITADO', traslado_retorno.to_dict())

    # Registrar hito en el estudio
    registrar_hito(estudio_id, 'RETORNO_GENERADO', actor_id)
```

### 5.5 Monitor de reingresos

```python
VENTANA_REINGRESO_HORAS = 72

def check_reingreso(paciente_id: str) -> RegresoStatus:
    """
    Se ejecuta en cada ingreso. Cruza DNI contra episodios anteriores.
    """
    ultimo_episodio = get_ultimo_episodio_previo(paciente_id)
    if not ultimo_episodio:
        return RegresoStatus(es_reingreso=False)

    horas_desde_alta = (now() - ultimo_episodio.alta_at).hours
    if horas_desde_alta < VENTANA_REINGRESO_HORAS:
        emit_event('REINGRESO_DETECTADO', {
            'paciente_id': paciente_id,
            'horas_desde_alta': horas_desde_alta,
            'episodio_anterior': ultimo_episodio.numero,
        })
        return RegresoStatus(
            es_reingreso=True,
            horas_desde_alta=horas_desde_alta,
            episodio_anterior=ultimo_episodio.numero,
        )
    return RegresoStatus(es_reingreso=False)
```

---

## 6. CONTRATOS DE EVENTOS (JSON Payloads)

Toda comunicación entre el Engine y los clientes usa WebSocket con estos formatos.

### 6.1 Hito registrado

```json
{
  "event_type": "HITO_REGISTRADO",
  "timestamp": "2026-05-02T10:47:00Z",
  "payload": {
    "episodio_id": "EP-88421",
    "episodio_numero": "GRD-20260502-0042",
    "paciente_id": "PAC-00028741",
    "paciente_nombre": "Vega, Carlos Alberto",
    "hito_codigo": "TRIAGE_COMPLETADO",
    "estado_resultante": "EN_ESPERA_MEDICA",
    "actor": {
      "user_id": "USR-104",
      "nombre": "López, Gabriela",
      "rol": "ENFERMERO"
    },
    "recurso_id": null,
    "metadata": {
      "nivel_triage": 2,
      "nivel_ia_sugerido": 2,
      "vitales": { "ta": "180/100", "fc": 92, "sat": 97 }
    }
  }
}
```

### 6.2 Traslado solicitado

```json
{
  "event_type": "TRASLADO_SOLICITADO",
  "timestamp": "2026-05-02T10:38:00Z",
  "payload": {
    "traslado_id": "TR-9932",
    "episodio_id": "EP-88421",
    "paciente_id": "PAC-00028741",
    "paciente_nombre": "Vega, Carlos A.",
    "origen": "Box 01",
    "destino": "Tomografía · Planta Baja",
    "nivel_riesgo": "URGENTE",
    "prioridad": "URGENTE",
    "equipo": ["SILLA_RUEDAS", "PREP_TC"],
    "paciente_pesado": false,
    "puntos_fatiga_calculados": 2,
    "auto_retorno": false,
    "estado_traslado": "PENDIENTE_ASIGNACION",
    "solicitante": {
      "user_id": "USR-201",
      "nombre": "Méndez, Roberto",
      "rol": "MEDICO"
    }
  }
}
```

### 6.3 SLA vencido

```json
{
  "event_type": "SLA_VENCIDO",
  "timestamp": "2026-05-02T11:22:00Z",
  "payload": {
    "episodio_id": "EP-88421",
    "episodio_numero": "GRD-20260502-0042",
    "paciente_nombre": "Vega, Carlos A.",
    "nivel_triage": 2,
    "sla_limite_minutos": 15,
    "tiempo_espera_minutos": 33,
    "minutos_vencido": 18,
    "alerta_destino": ["JEFE_GUARDIA", "RECEPCION"]
  }
}
```

### 6.4 Reingreso detectado

```json
{
  "event_type": "REINGRESO_DETECTADO",
  "timestamp": "2026-05-02T09:14:00Z",
  "payload": {
    "paciente_id": "PAC-00028741",
    "paciente_nombre": "Vega, Carlos A.",
    "episodio_actual": "EP-88421",
    "episodio_anterior": "EP-87201",
    "horas_desde_alta": 38,
    "motivo_anterior": "HTA severa · Crisis hipertensiva"
  }
}
```

### 6.5 Estado de cama actualizado

```json
{
  "event_type": "CAMA_ESTADO_ACTUALIZADO",
  "timestamp": "2026-05-02T10:58:00Z",
  "payload": {
    "recurso_id": "REC-OBS-02",
    "codigo": "OBS-02",
    "tipo": "BOX",
    "estado_anterior": "OCUPADA",
    "estado_nuevo": "LIMPIEZA",
    "episodio_cerrado": "EP-88100",
    "actor": {
      "user_id": "USR-104",
      "rol": "ENFERMERO"
    }
  }
}
```

### 6.6 Estudio completado (dispara retorno automático)

```json
{
  "event_type": "ESTUDIO_COMPLETADO",
  "timestamp": "2026-05-02T11:08:00Z",
  "payload": {
    "estudio_id": "EST-4421",
    "episodio_id": "EP-88421",
    "modalidad": "TC",
    "equipo_codigo": "TC-1",
    "entrada_equipo_at": "2026-05-02T10:52:00Z",
    "salida_equipo_at": "2026-05-02T11:08:00Z",
    "room_turnover_calculado_minutos": 16,
    "retorno_traslado_id": "TR-9933",
    "tecnico": {
      "user_id": "USR-301",
      "nombre": "Romero, Luis",
      "rol": "TECNICO_IMAGEN"
    }
  }
}
```

---

## 7. API REST — ENDPOINTS POR MÓDULO

Base URL: `https://api.cordis.zenoxia.com/v1`

### 7.1 Pacientes

```
GET    /pacientes?dni={dni}                  # Buscar por DNI
GET    /pacientes/{id}                       # Ficha completa
GET    /pacientes/{id}/episodios             # Historial de episodios
POST   /pacientes                            # Crear paciente nuevo
```

### 7.2 Episodios (Guardia)

```
GET    /episodios                            # Lista activos con filtros
GET    /episodios/{id}                       # Episodio completo + hitos
POST   /episodios                            # Crear episodio (ingreso)
POST   /episodios/{id}/hitos                 # Registrar hito
GET    /episodios/{id}/hitos                 # Timeline completo
PATCH  /episodios/{id}/estado-cache          # Actualizar cache (Engine only)
```

### 7.3 Triage

```
POST   /triage                               # Crear evaluación de triage
GET    /triage/{episodio_id}                 # Triage del episodio
POST   /triage/ia-sugerir                   # Solicitar sugerencia de nivel a IA
```

### 7.4 Camas / Recursos físicos

```
GET    /recursos?tipo=CAMA&institucion={id}  # Lista con estado actual
GET    /recursos/{id}                        # Recurso + historial de estados
POST   /recursos/{id}/hitos                  # Cambiar estado
GET    /recursos/mapa                        # Vista completa del mapa de camas
```

### 7.5 Traslados

```
GET    /traslados?estado=PENDIENTE           # Cola de traslados
GET    /traslados/{id}                       # Traslado completo
POST   /traslados                            # Crear traslado manual
PATCH  /traslados/{id}/asignar               # Asignar camillero
PATCH  /traslados/{id}/estado                # Avanzar estado (app mobile)
POST   /traslados/{id}/problema              # Reportar eventualidad
GET    /traslados/sugerencias?traslado={id}  # Engine sugiere camillero
```

### 7.6 Camilleros

```
GET    /camilleros/turno-activo              # Camilleros en turno ahora
GET    /camilleros/{id}/fatiga              # Estado de fatiga actual
GET    /camilleros/{id}/historial           # Traslados del turno
```

### 7.7 Estudios de imágenes

```
GET    /estudios?estado=PENDIENTE            # Worklist
GET    /estudios/{id}                        # Estudio completo
POST   /estudios                             # Crear pedido
PATCH  /estudios/{id}/iniciar                # Técnico inicia (timestamp inmutable)
PATCH  /estudios/{id}/completar              # Técnico completa → dispara retorno
PATCH  /estudios/{id}/aprobar                # Médico aprueba resultado
POST   /estudios/{id}/no-show                # Registrar no-show
GET    /estudios/rtt                         # Room Turnover Time del turno
```

### 7.8 Engine / Sistema

```
GET    /engine/sla-status                    # Estado SLA de todos los episodios activos
POST   /engine/check-reingreso?dni={dni}     # Verificar reingreso en tiempo real
GET    /engine/metricas/turno                # KPIs del turno actual
WebSocket /ws/guardia                        # Stream de eventos en tiempo real
WebSocket /ws/imagenes                       # Stream de eventos de imágenes
WebSocket /ws/camilleros                     # Stream de traslados
```

---

## 8. INTEGRACIÓN CON HIS LEGACY

### 8.1 Estrategia

Cordis opera como capa de lectura sobre el HIS existente. No escribe en el HIS salvo excepciones acordadas por contrato con la institución.

```
HIS → Cordis (LECTURA)
  - Datos filiatorios del paciente (RN-01: inmutables)
  - Historia clínica previa
  - Cobertura médica y validación
  - Medicación habitual
  - Alergias registradas

Cordis → HIS (ESCRITURA selectiva, según configuración)
  - Episodio cerrado (alta o internación)
  - Resultado de triage (nivel, vitales)
  - Estudios de imágenes completados
```

### 8.2 Modo de contingencia offline

```
Si el HIS no responde → Cordis opera en modo autónomo:
  - El recepcionista carga datos básicos manualmente (RN-03)
  - Se genera episodio temporal sin validación HIS
  - Al restaurar conectividad → reconciliación automática
  - Formularios físicos imprimibles disponibles como backup
```

---

## 9. DESIGN TOKENS — REFERENCIA FRONTEND

Los tokens están definidos en `/design-tokens-v2.css`. Resumen para el backend (generación de PDFs, emails, reportes):

```python
DESIGN_TOKENS = {
    # Marca
    'primary':      '#2B57D9',
    'primary_dark': '#1A3FAA',
    'primary_light':'#EBF2FF',
    # Fondos
    'bg':  '#ffffff',
    'bg2': '#f5f4f0',
    'bg3': '#f0efea',
    # Texto
    'txt':  '#1a1a18',
    'txt2': '#5f5e5a',
    'txt3': '#888780',
    # Semánticos
    'ok':   '#0F6E56', 'ok_l': '#E1F5EE',
    'warn': '#633806', 'warn_l':'#FAEEDA', 'warn_m':'#EF9F27',
    'err':  '#791F1F', 'err_l': '#FCEBEB', 'err_m': '#E24B4A',
    # Triage MTS (estándares internacionales)
    'triage': {
        1: '#C0392B',  # Resucitación
        2: '#D4680A',  # Emergencia
        3: '#A07C00',  # Urgencia
        4: '#2E7D4F',  # Menor
        5: '#1565A8',  # No urgente
    },
}
```

---

## 10. MÉTRICAS OPERATIVAS Y KPIs

### 10.1 KPIs de Guardia

```sql
-- Door-to-Doctor time (tiempo ingreso → médico llamado)
SELECT
    AVG(atencion.timestamp - ingreso.timestamp) AS door_to_doctor_avg,
    MAX(atencion.timestamp - ingreso.timestamp) AS door_to_doctor_max,
    t.nivel
FROM hitos_tiempo ingreso
JOIN hitos_tiempo atencion
    ON ingreso.episodio_id = atencion.episodio_id
JOIN triage_logs t ON t.episodio_id = ingreso.episodio_id
WHERE ingreso.hito_codigo = 'INGRESO_RECEPCION'
  AND atencion.hito_codigo = 'MEDICO_LLAMADO'
  AND ingreso.timestamp > NOW() - INTERVAL '30 days'
GROUP BY t.nivel;

-- Length of Stay (estadía total)
SELECT
    AVG(egreso.timestamp - ingreso.timestamp) AS los_avg,
    PERCENTILE_CONT(0.8) WITHIN GROUP
        (ORDER BY egreso.timestamp - ingreso.timestamp) AS los_p80
FROM hitos_tiempo ingreso
JOIN hitos_tiempo egreso
    ON ingreso.episodio_id = egreso.episodio_id
WHERE ingreso.hito_codigo = 'INGRESO_RECEPCION'
  AND egreso.hito_codigo = 'EGRESO_FISICO';
```

### 10.2 KPIs de Imágenes

```sql
-- Room Turnover Time por equipo
SELECT
    rf.codigo,
    rf.nombre,
    AVG(proximo.entrada_equipo_at - actual.salida_equipo_at) AS rtt_avg
FROM estudios_imagen actual
JOIN estudios_imagen proximo
    ON proximo.equipo_id = actual.equipo_id
    AND proximo.entrada_equipo_at > actual.salida_equipo_at
WHERE actual.salida_equipo_at IS NOT NULL
  AND proximo.entrada_equipo_at IS NOT NULL
JOIN recursos_fisicos rf ON rf.id = actual.equipo_id
GROUP BY rf.id, rf.codigo, rf.nombre;

-- Tasa de no-show
SELECT
    COUNT(*) FILTER (WHERE no_show = TRUE)::float /
    COUNT(*) AS tasa_no_show,
    modalidad
FROM estudios_imagen
WHERE solicitado_at > NOW() - INTERVAL '30 days'
  AND tipo_sla = 'AMBULATORIO'
GROUP BY modalidad;
```

### 10.3 KPIs de Camilleros

```sql
-- Carga de traslados por camillero en el turno
SELECT
    u.apellido || ', ' || u.nombre AS camillero,
    COUNT(tr.id) AS traslados_totales,
    SUM(tr.puntos_fatiga) AS fatiga_acumulada,
    COUNT(tr.id) FILTER (WHERE tr.paciente_pesado) AS traslados_pesados,
    AVG(tr.completado_at - tr.inicio_at) AS tiempo_promedio
FROM traslados tr
JOIN usuarios u ON u.id = tr.camillero_id
WHERE tr.solicitado_at > NOW() - INTERVAL '1 day'
  AND tr.estado = 'COMPLETADO'
GROUP BY u.id, u.apellido, u.nombre
ORDER BY fatiga_acumulada DESC;
```

---

## 11. DATOS OPERATIVOS REALES — CONTEXTO DE VALIDACIÓN

Cordis fue diseñado y validado contra datos reales de operación. Estos datos orientan el diseño pero **no configuran reglas específicas** de una institución. Cordis es un producto horizontal.

### 11.1 Volúmenes de referencia

| Métrica | Valor real | Fuente |
|---------|-----------|--------|
| Pacientes/mes guardia | ~7,100 (236/día) | Dataset episodios Abr 2026 |
| Día pico | Miércoles | Análisis semanal |
| Pico horario | 11-13hs y 16-18hs | Análisis Abr 2026 |
| Traslados totales analizados | ~44,400 | Planilla 2024-2026 |
| T2 espera promedio | 33 min (SLA: 15 min) | Dataset guardia |
| T2 espera máxima | 390 min | Dataset guardia |
| T3 del volumen total | 60% | Dataset guardia |
| Internaciones/día | ~17 (bloquean boxes) | Dataset guardia |
| Tasa reingreso <72hs | 8% | Dataset guardia |
| No-show estudios ambulatorios | 6.5–8.1% | Dataset imágenes |
| RTT Tomografía (no medido) | 0.1% de registros | Dataset imágenes |

### 11.2 Flujo más crítico validado

```
Ambulatoria → Shock Room → TAC
(ruta más densa del hospital por volumen de traslados)

TAC es simultáneamente:
  - Origen #1: ~4,128 traslados
  - Destino #1: ~4,736 traslados
→ Justifica la regla de retorno automático como prioritaria
```

### 11.3 Turno más cargado

```
Tarde (13:00–20:00): 34% del volumen total
→ El Engine debe ajustar thresholds de alerta
  y disponibilidad de recursos a partir de las 13:00hs
```

### 11.4 Registro de camilleros (contexto histórico)

```
Durante 2024 y 2025 la columna "camillero responsable"
estuvo vacía en el 100% de los registros.
Solo 1,620 traslados en 2026 tienen camillero asignado.

→ Cordis resuelve este problema de raíz:
  cada traslado registra automáticamente quién lo hizo,
  cuándo, con qué equipamiento y cuánto tardó.
```

---

## APÉNDICE A — Reglas de negocio consolidadas

| ID | Regla | Módulo |
|----|-------|--------|
| RN-01 | DNI, nombre, fecha nac. y sexo registral son inmutables | Pacientes |
| RN-02 | Sin cobertura → particular transitorio con circuito de reintegro | Recepción |
| RN-03 | Contingencia offline → formularios imprimibles disponibles | Sistema |
| RN-04 | IA sugiere nivel de triage, enfermero valida siempre | Triage |
| RN-05 | Órdenes médicas en borrador hasta aprobación del médico | Imágenes |
| RN-06 | SLA inmediato para urgencias, 6-48hs para ambulatorios | Imágenes |
| RN-07 | Resultado de estudio no entregable sin aprobación médica | Imágenes |
| RN-08 | Llamador del médico mueve automáticamente al paciente en el tablero | Guardia |
| RN-09 | Ciclo hotelero: Libre→Ocupada→Pend.Internación→Limpieza→Lista→Libre | Camas |
| RN-10 | Toda transición registra actor, rol y timestamp | Sistema |
| RN-11 | Camillero no puede ser interrumpido en un traslado activo | Camilleros |
| RN-12 | El retorno de estudios de imágenes se genera automáticamente | Engine |
| RN-13 | SLA vencido en T2 dispara alerta push al Jefe de Guardia | Engine |
| RN-14 | Reingreso <72hs genera badge de alerta en el tablero médico | Engine |
| RN-15 | Fatiga >80% del umbral activa sugerencia de descanso | Engine |

---

## APÉNDICE B — Estructura de archivos Fase 1

```
cordis-fase1/
├── index.html                          ← Navegador del ecosistema
├── design-tokens-v2.css               ← Design Tokens v2.0 (fuente de verdad)
├── kairos-icons.svg                   ← Sprite SVG 37 íconos
├── README.md
├── guardia/
│   ├── command-center.html            ← Tablero principal de guardia
│   ├── enfermeros-guardia.html        ← Triage asistido por IA
│   └── app-paciente.html              ← App móvil del paciente
├── imagenes/
│   ├── command-center-imagenes.html   ← Recepción y coordinación
│   └── worklist-tc.html               ← Worklist del técnico radiólogo
└── camilleros/
    ├── central-camilleros.html        ← Central de despacho
    └── camilleros-mobile.html         ← App móvil del camillero
```

---

*Documento generado por Zenoxia con Claude (Anthropic) · Cordis Engine v1.0 · Mayo 2026*
