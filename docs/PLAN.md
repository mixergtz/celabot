# Celabot — Plan de aplicación de visión por computador para detección de anomalías en pequeños comercios

> Borrador inicial. Mercado objetivo: **Colombia**. Despliegue: **solo cloud (RTSP → nube)**. Audio: **fase 2**. Modelos: arrancamos con comerciales (SAM 3, VLMs) dejando la puerta abierta a modelos propios.

---

## 1. Resumen ejecutivo

Celabot es un servicio SaaS que se conecta a las cámaras existentes de tiendas, droguerías, panaderías y supermercados de barrio en Colombia y alerta en segundos al dueño/encargado cuando ocurre un comportamiento anómalo:

- Intentos de hurto por clientes (ocultamiento, *grab-and-run*, manipulación de empaques).
- Hurto armado (presencia de arma, posturas de pánico, *lock-down*).
- Hurto interno por empleados (*sweethearting*, devoluciones falsas, descuadre de caja).
- Aglomeraciones, riñas, intrusión fuera de horario.

El MVP es **solo video**. El audio (disparo, gritos, vidrio roto, palabras clave de atraco) entra en una segunda fase. La inferencia corre en la nube; el cliente solo instala un *gateway* mínimo que retransmite RTSP con TLS.

Diferenciadores:

- **Alerta accionable en español** vía WhatsApp con clip de 10–15 s, no solo "alarma".
- **Sin reemplazar cámaras** existentes (DVR/NVR con RTSP/ONVIF).
- **Privacidad por diseño**: cumplimiento Ley 1581/2012 (Habeas Data), retención corta por defecto, anonimización opcional.
- **Modelos comerciales hoy, propios mañana**: arquitectura desacoplada para reemplazar componentes con modelos entrenados con datos del cliente.

---

## 2. Casos de uso priorizados

Cada caso se modela como una **cadena de señales atómicas** → **agregador temporal** → **confirmación VLM** → **alerta**.

| # | Caso | Severidad | Señales atómicas | Confirmación | Latencia objetivo |
|---|------|-----------|------------------|--------------|-------------------|
| 1 | Hurto armado en curso | Crítica | Detección de arma, manos arriba (pose), múltiples personas estáticas, audio (fase 2: disparo/grito) | VLM con últimos 5 s de video | < 5 s |
| 2 | *Grab-and-run* | Alta | Persona toma ítem + se mueve rápido a salida sin pasar por caja | VLM revisa trayectoria | < 15 s |
| 3 | Ocultamiento (*concealment*) | Alta | Mano + ítem entra a bolsillo / bolso / ropa (SAM 3 + pose) | VLM | < 30 s |
| 4 | *Sweethearting* (empleado no escanea) | Media | Ítem cruza zona de caja sin evento POS correspondiente | Correlación POS + VLM | < 60 s (puede ser post-hoc) |
| 5 | Devolución / *void* sospechoso | Media | Evento POS de devolución sin cliente presente | Regla + VLM | post-hoc |
| 6 | Caja abierta sin venta | Media | Evento POS "cash drawer open" + sin venta previa | Regla | < 30 s |
| 7 | Riña / agresión | Alta | Pose agresiva + proximidad + movimiento rápido | VLM | < 10 s |
| 8 | Intrusión fuera de horario | Alta | Persona detectada en horario configurado como cerrado | Regla | < 5 s |
| 9 | Merodeo (*loitering*) | Baja | Persona estática > N min en zona de góndola valiosa | Regla | < N min |
| 10 | Aglomeración inusual | Baja | Densidad de personas > umbral | Regla | < 30 s |

Casos 1, 2, 3, 8 son el **núcleo del MVP**. El resto se libera por fases.

---

## 3. Arquitectura (cloud-only)

```
 Tienda                          Internet                  Cloud (AWS/GCP)
 ─────────                       ────────                  ──────────────────────────────
                                                          ┌────────────────────────────┐
 [Cámaras IP/DVR RTSP] ─┐                                 │  Ingesta (mediaMTX/KVS)    │
                        ├─► [Gateway Celabot] ──TLS──►    │  ▼                         │
 [POS opcional]   ──────┘   (Mini-PC/RPi/Docker)          │  Frame sampler (GPU)       │
                                                          │  ▼                         │
                                                          │  Detección/Pose/Tracking   │
                                                          │  (YOLO + SAM 3 + RTMPose)  │
                                                          │  ▼                         │
                                                          │  Agregador temporal        │
                                                          │  ▼                         │
                                                          │  Confirmador VLM           │
                                                          │  (Gemini/Claude video)     │
                                                          │  ▼                         │
                                                          │  Event bus (Redis/Kafka)   │
                                                          │  ▼                         │
                                                          │  Alertas + Dashboard       │
                                                          └────────────────────────────┘
                                                                       │
                                                          ┌────────────▼─────────────┐
                                                          │ WhatsApp Cloud API,      │
                                                          │ push, email, llamada     │
                                                          └──────────────────────────┘
```

### 3.1 Gateway en tienda (mínimo)

El "cloud-only" no exime de un componente físico que **traduzca y proteja** el stream:

- Imagen Docker (`celabot-gateway`) en Raspberry Pi 5 / mini-PC barato (~USD 60–120).
- Funciones:
  - Descubrimiento ONVIF + configuración de RTSP.
  - Re-paquetización RTSP → **SRT** o **WebRTC** con TLS hacia la nube.
  - Sub-stream a 480p / 5–10 fps para análisis (el stream principal queda para grabación local del cliente).
  - Almacenamiento en *ring buffer* local (últimos ~5 min) para reenviar contexto cuando se detecta un evento.
  - Telemetría: latencia, pérdida, *heartbeat*.
- **No hay inferencia local en MVP**. Eso lo hace explotable luego como upsell ("Celabot Edge").

### 3.2 Ingesta en nube

- **mediaMTX** o **AWS Kinesis Video Streams** para ingesta SRT/WebRTC/RTMP.
- *Frame sampler* en GPU con **NVIDIA DeepStream** o **Triton Inference Server**: muestrea a 5–10 fps, decodifica en GPU (NVDEC), forma *batches* de varias cámaras.

### 3.3 Capa de inferencia

Dos carriles:

1. **Carril rápido (siempre activo, frame-level)**: detecciones baratas con modelos pequeños.
2. **Carril lento (disparado por eventos)**: VLM caro confirma antes de alertar.

Esta separación es lo que hace viable cloud-only en costo.

### 3.4 Agregador temporal

Un servicio con estado (Redis Streams + workers) que mantiene, por cámara y por *track-id*:
- Ventana deslizante de señales (30–60 s).
- Reglas declarativas (YAML) por caso de uso.
- *Cooldown* por tienda/cámara para evitar tormentas de alertas.

### 3.5 Confirmador VLM

Cuando una regla dispara, se arma un *clip* de N segundos (típicamente 5–15 s) y se envía a un VLM (Gemini 2.x video, Claude con video, GPT-4o) con un *prompt* específico al caso:

> "Eres un analista de seguridad de tienda. ¿En este clip una persona oculta mercancía en su ropa o bolso? Responde JSON: `{anomaly: bool, confidence: 0-1, reason: string, suggested_action: string}`. Considera falsos positivos comunes: revisar etiqueta, guardar celular, sacar billetera."

Esto recorta drásticamente los falsos positivos sin entrenar modelos propios al inicio.

### 3.6 Alertas

- **WhatsApp Cloud API** como canal primario (Colombia: penetración > 90% en pequeños comerciantes).
- Plantilla con: foto/clip, hora, cámara, tipo, acción sugerida, botones "Ver en vivo / Es falsa alarma / Llamar a policía".
- Fallback: notificación push (app móvil ligera React Native), SMS, llamada automatizada (Twilio Voice) para severidad crítica.

---

## 4. Stack de modelos

### 4.1 Visión

| Capa | Modelo propuesto (MVP) | Alternativas | Camino a modelo propio |
|------|-----------------------|--------------|------------------------|
| Detección genérica (persona, bolso, carrito) | YOLO11 / RT-DETR v2 | Grounding DINO 1.6 | Fine-tune con datos del cliente |
| Segmentación / tracking *promptable* | **SAM 3** (Meta) | SAM 2.1, Cutie | SAM 3 ya hace tracking; fine-tune si Meta lo permite, o destilar a modelo propio |
| Tracking multi-objeto | BoT-SORT con Re-ID (OSNet) | ByteTrack, OC-SORT | Re-ID propio con triplets de tiendas reales |
| Pose | RTMPose-L | ViTPose, MoveNet | Suficiente con OSS |
| Detección de arma | YOLO11 fine-tuned (datasets: Sohas, Guns&Knives) | Roboflow Universe pre-entrenados | **Propio desde día 1**: dataset crítico, sesgo geográfico fuerte |
| Acción / interacción mano-objeto | VideoMAE v2, UniFormerV2 | X-CLIP | Fine-tune; pocos datos públicos buenos |
| Razonamiento de alto nivel | **Gemini 2.x Video** / Claude (video) | GPT-4o, Qwen2.5-VL | Mantener vendor-agnóstico vía capa de abstracción |
| Búsqueda forense por similitud | SigLIP-2 + Qdrant | CLIP, DINOv2 | OSS suficiente |
| Re-identificación cara (opcional, regulado) | InsightFace (buffalo_l) | — | Desactivado por defecto, opt-in por cliente |

### 4.2 ¿Por qué SAM 3 desde el inicio?

- *Promptable concept segmentation* permite preguntar en lenguaje natural ("manos", "ítem en estante", "bolso") sin entrenar.
- Tracking integrado reduce dependencias.
- Ideal para *concealment* (seguir mano + objeto hasta bolsillo).
- Riesgo: licencia y costo de inferencia. Mitigación: usarlo solo en el carril lento, no por frame.

### 4.3 Capa de abstracción

Todo el código habla con interfaces, no con modelos concretos:

```
Detector, Segmenter, Tracker, PoseEstimator, ActionRecognizer, Reasoner
```

Cada uno tiene implementaciones `commercial/*` y `selfhosted/*`. Esto permite, en orden:

1. MVP con `commercial/*` (APIs gestionadas).
2. Migrar piezas costosas a `selfhosted/*` cuando la economía lo pida.
3. Reemplazar por modelos propios entrenados con datos del cliente (con consentimiento contractual de uso para mejorar el modelo).

---

## 5. Audio (Fase 2)

Aunque está fuera del MVP, se diseña ahora para no rehacer la arquitectura:

- Ingesta: el gateway envía un sub-stream de audio (16 kHz, mono, AAC/Opus).
- Clasificadores:
  - **Gunshot / glass break / scream**: PANNs (CNN14) o YAMNet fine-tuned con datasets (UrbanSound8K, AudioSet, MIVIA Audio Events).
  - **Stress / pánico en voz**: modelos de SER (Speech Emotion Recognition) — wav2vec2 + clasificador.
  - **Palabras clave de atraco** ("al piso", "esto es un asalto", "abre la caja"): Whisper (es-CO) + keyword spotter (KWS) ligero ejecutado por ventana.
- Fusión multimodal: el agregador temporal combina señales de video y audio antes de disparar al VLM.

**Cumplimiento crítico**: grabar audio en Colombia exige consentimiento explícito y señalización clara. Por contrato, el cliente firma asumir esa responsabilidad; Celabot ofrece pantallas de aviso obligatorias y registros de consentimiento.

---

## 6. Datos y MLOps

### 6.1 Recolección

- **Bootstrap**: datasets públicos (UCF-Crime, ShanghaiTech, XD-Violence, Avenue, MEVA, AVA-Kinetics, RWF-2000) para validar pipeline.
- **Datos de tienda real**: con consentimiento contractual del cliente, retención corta (7–30 días) salvo clips marcados como evento (90–365 días).
- **Sintéticos**: Unity / Unreal + asset packs de retail para escenarios raros (arma, *grab-and-run*) — sube *recall* sin comprometer privacidad.

### 6.2 Etiquetado

- CVAT auto-hospedado o Roboflow (paga por uso).
- *Active learning*: el sistema marca los clips donde el VLM duda (confianza 0.4–0.7) como prioridad de etiquetado.
- *Human-in-the-loop*: un operador junior valida alertas críticas en horario de oficina; sus decisiones alimentan el conjunto de entrenamiento.

### 6.3 Entrenamiento y despliegue

- Registro: MLflow o Weights & Biases.
- Serving: Triton Inference Server (multi-modelo, GPU compartida).
- *Shadow mode*: todo modelo propio corre en paralelo al de producción durante ≥2 semanas; se promueve solo si supera al actual en F1 y latencia.
- *Canary*: rollout por 10% de tiendas con rollback automático si KPI cae.

### 6.4 Métricas

- **Por evento**: precisión, recall, F1, AUROC, latencia p50/p95.
- **Por tienda**: alertas/día, tasa de falsos positivos confirmados por el dueño, *time-to-acknowledge*.
- **De negocio**: pérdidas evitadas reportadas, NPS, churn, ARPU.

---

## 7. Privacidad y cumplimiento (Colombia)

- **Ley 1581 de 2012** (Habeas Data) + **Decreto 1377 de 2013**. Registro de bases de datos ante la SIC.
- Vigilancia con cámaras: la SIC ha emitido conceptos que exigen señalización visible, finalidad legítima, proporcionalidad y retención mínima.
- Medidas operativas:
  - **Aviso visible** en tienda (Celabot entrega plantilla).
  - **Consentimiento de empleados** por escrito para monitoreo (relación laboral).
  - **Retención por defecto**: clips no-evento se borran a las 72 h; eventos a 90 días; configurable.
  - **Anonimización opcional** (blur de rostros) en clips compartidos por WhatsApp.
  - **Cifrado**: TLS en tránsito, AES-256 en reposo (KMS por tenant).
  - **Aislamiento por tenant** en Postgres (RLS) y en almacenamiento de video (prefijos S3 + IAM por tenant).
  - **Derechos ARCO**: portal para que cualquier persona consulte / borre sus datos (procedimiento documentado).
- **Reconocimiento facial**: desactivado por defecto. Si se activa, requiere autorización adicional y queda registrado en bitácora inmutable.

---

## 8. Integraciones

| Integración | Prioridad | Notas |
|-------------|-----------|-------|
| WhatsApp Cloud API | MVP | Canal principal de alertas. Plantillas pre-aprobadas. |
| ONVIF / RTSP | MVP | Estándar para descubrir cámaras existentes. |
| POS Colombia | Fase 2 | **Bold**, **Siigo**, **Alegra**, **Loyverse**, **Bsale**. Webhooks o lectura de impresora fiscal cuando no hay API. |
| Empresas de seguridad privada | Fase 3 | API para escalar alertas críticas (botón de pánico digital). |
| Policía Nacional / cuadrantes | Fase 3 | Convenio + integración con línea 123 / app "A un clic de la Policía". Requiere validación legal. |
| Pasarelas de pago | MVP | **Wompi** o **Bold** para cobro mensual; soportar PSE y débito automático. |
| Facturación electrónica | MVP | Siigo / Alegra para emitir factura DIAN. |

---

## 9. Producto / UX

### 9.1 Personas

- **Don Jaime, dueño de tienda de barrio (50–80 m²)**: 2–4 cámaras, opera solo o con familiar. Quiere WhatsApp, no quiere apps complejas. Presupuesto < COP 80–120k/mes.
- **Andrea, administradora de cadena de 6 droguerías**: dashboard web, reportes, integración con POS, comparación entre sucursales. Presupuesto COP 300–600k/mes.
- **Carlos, jefe de seguridad de minimarket franquiciado**: alertas críticas a equipo, integración con guardia, evidencia para procesos legales.

### 9.2 App móvil mínima (Fase 1.5)

- Login, lista de cámaras, ver en vivo (WebRTC), historial de alertas con clip, marcar como falsa alarma.
- React Native + Expo. iOS y Android.

### 9.3 Dashboard web (Fase 1)

- Next.js + Tailwind + shadcn/ui.
- Multi-tienda, multi-cámara, mapa de calor por zona, *time-lapse* de alertas.

---

## 10. Stack técnico propuesto

| Capa | Elección | Razón |
|------|----------|-------|
| Lenguaje principal de servicios CV | Python 3.12 | Ecosistema ML |
| API gateway / backend de negocio | TypeScript + NestJS o Go + Fiber | Tipado fuerte, dev velocity |
| Streaming de video | mediaMTX → SRT/WebRTC | OSS maduro |
| Inferencia | Triton + TensorRT, GPUs L4/L40S en GCP o AWS | Mejor $/perf para CV |
| Cola de eventos | Redis Streams (MVP) → Kafka (escala) | Empezar simple |
| Base de datos | Postgres 16 + RLS, TimescaleDB para series | Estándar |
| Almacenamiento de video/clip | S3 con Lifecycle (estándar→IA→Glacier) | Costo |
| Búsqueda forense | Qdrant | Embeddings de SigLIP |
| Observabilidad | OpenTelemetry + Grafana Cloud o Datadog | — |
| IaC | Terraform + GitHub Actions | — |
| Despliegue | Kubernetes (GKE/EKS) o Nomad si se quiere menos peso | — |

---

## 11. Roadmap por fases

### Fase 0 — Validación (4–6 semanas)

- 5 tiendas piloto en Bogotá / Medellín, sin cobro.
- Procesar streams en una sola GPU compartida con pipeline mínimo: detección de persona + intrusión fuera de horario + arma (pretrained Roboflow) + VLM como confirmador único.
- Alertas WhatsApp manuales (un humano confirma antes de enviar — Mago de Oz).
- Objetivo: validar precision/recall por caso y *willingness to pay*.

### Fase 1 — MVP comercial (3 meses)

- Casos 1, 2, 3, 8 automatizados end-to-end.
- Onboarding self-service de cámaras vía gateway.
- Dashboard web + alertas WhatsApp + cobro recurrente (Wompi).
- Cumplimiento Ley 1581 implementado.
- **Meta**: 30 tiendas pagas, < 2 falsos positivos por tienda por día.

### Fase 2 — Expansión (3 meses)

- Audio (gunshot, gritos, vidrio, KWS de atraco).
- Integración POS (Bold + Loyverse para empezar): *sweethearting*, *void* sospechoso.
- App móvil.
- Modelos propios entrenados con datos de Fase 0–1 (arma, *concealment*).
- **Meta**: 200 tiendas, NPS ≥ 50.

### Fase 3 — Plataforma (6 meses)

- Marketplace de "detectores" (alcohol a menores, mascotas, conteo de gente para *queue management*).
- Integración con seguridad privada y línea 123.
- Modo edge (upsell): caja con Jetson Orin Nano para tiendas con mala conectividad.
- Multi-país: México (LFPDPPP), Perú (Ley 29733), Chile.

---

## 12. Costos (orden de magnitud, USD)

Supuestos: tienda promedio = 4 cámaras, sub-stream 480p @ 7 fps.

**Por tienda / mes**:

| Concepto | Costo |
|----------|-------|
| Banda ancha de salida (cliente) | ~ — (asumido por cliente, ~2 Mbps upload) |
| Ingesta + storage (clips eventos, retención 30 días) | ~ USD 3 |
| Inferencia rápida (YOLO + pose + tracking en L4 compartida) | ~ USD 8–12 |
| Inferencia VLM (cuando dispara, ~50 confirmaciones/día) | ~ USD 5–10 |
| WhatsApp Cloud API (Colombia, ~200 mensajes/mes) | ~ USD 2 |
| Otros (DB, logs, observabilidad) | ~ USD 1 |
| **Total cloud por tienda** | **~ USD 20–30 / mes** |

**Precio sugerido al cliente**: COP 89.000–149.000/mes (≈ USD 22–37), según número de cámaras. Margen bruto saludable se alcanza con ≥ 500 tiendas.

**Hardware del gateway**: USD 60–80 (Raspberry Pi 5 + caja + microSD). Se puede subsidiar y diluir en 12 meses, o vender a costo.

---

## 13. Riesgos y mitigación

| Riesgo | Impacto | Mitigación |
|--------|---------|------------|
| Tasa alta de falsos positivos → cansancio de alertas | Churn | Doble carril (rápido + VLM), *cooldown*, botón "falsa alarma" que alimenta dataset, umbrales por tienda |
| Latencia cloud insuficiente para casos críticos | Pérdida de confianza | SLA en ingesta < 1 s, regiones cercanas (GCP São Paulo / AWS São Paulo), opción Edge en Fase 3 |
| Internet inestable en tiendas de barrio | Pérdida de eventos | *Ring buffer* local en gateway (5 min), reenvío cuando se restablece conexión, alerta de "cámara offline" |
| Costos de VLM se disparan | Margen negativo | Carril rápido filtra > 95%, *prompt caching*, modelo propio más barato en Fase 2, batch de clips |
| Demandas por privacidad / video filtrado | Legal + reputación | Cifrado, RLS, retención corta, auditoría inmutable, seguro de responsabilidad civil |
| Dependencia de un solo proveedor de VLM | Lock-in / cambio de precios | Capa de abstracción `Reasoner` con 2 proveedores activos |
| SAM 3 cambia licencia o se vuelve costoso | Reemplazo doloroso | Aislar uso a un servicio, dejar SAM 2.1 como fallback |
| Sesgo del modelo (falsos positivos a personas por raza, edad, vestimenta) | Daño + legal | Auditoría de equidad por demografía, no usar reconocimiento facial por defecto, *redress mechanism* |
| Empleados desactivan el gateway | Detección tardía | Heartbeat con alerta al dueño si la cámara cae > 5 min |

---

## 14. Estructura inicial del repo (propuesta)

```
celabot/
├── docs/                       # Plan, ADRs, runbooks
├── gateway/                    # Imagen Docker para tienda (Python/Go)
│   ├── onvif_discovery/
│   ├── relay/                  # RTSP→SRT/WebRTC
│   └── ringbuffer/
├── services/
│   ├── ingest/                 # mediaMTX config + relays
│   ├── sampler/                # DeepStream/Triton pipeline
│   ├── detectors/              # YOLO, pose, SAM 3 wrappers
│   │   ├── interfaces.py
│   │   ├── commercial/
│   │   └── selfhosted/
│   ├── aggregator/             # Reglas YAML + estado en Redis
│   ├── reasoner/               # VLM clients (Gemini/Claude/GPT)
│   ├── alerts/                 # WhatsApp/Push/Voice
│   ├── api/                    # Backend de negocio
│   └── ops/                    # Cron, retención, billing
├── web/                        # Next.js dashboard
├── mobile/                     # React Native app (Fase 1.5)
├── ml/
│   ├── datasets/
│   ├── training/
│   ├── evaluation/
│   └── registry/               # MLflow
├── infra/                      # Terraform, k8s manifests
└── .github/workflows/
```

---

## 15. Decisiones abiertas

1. **Nube primaria**: GCP (Gemini nativo, región São Paulo) vs AWS (Kinesis Video Streams maduro, mejor catálogo de GPUs). Recomendación: **GCP** por VLM y región.
2. **Lenguaje del backend de negocio**: TS/NestJS vs Go. Recomendación: **TypeScript** por velocidad de equipo pequeño y reutilización con dashboard.
3. **Gateway propio vs Frigate**: ¿forkamos Frigate (OSS, maduro) o construimos uno minimalista? Recomendación: **minimalista propio** — Frigate hace inferencia local, no es nuestro caso.
4. **Política con datos del cliente para entrenar modelos**: ¿opt-in con descuento? ¿opt-out? Decisión comercial pendiente.
5. **¿Operación 24/7 humana en Fase 0–1?** Un SOC humano pequeño durante el día sube precisión pero también costo. Pilotear con turnos 10:00–22:00.
6. **Reconocimiento facial**: ¿se ofrece como producto separado (lista de personas problema reportadas por el dueño) o se descarta por riesgo regulatorio? Recomendación: **descartar en MVP**.

---

## 16. Próximos pasos concretos (siguientes 2 semanas)

1. Validar con 3–5 dueños de tienda en Bogotá: entrevistas de 30 min, foco en disposición de pago y casos de uso reales.
2. Prototipo end-to-end con **una sola cámara, un solo caso (arma)** en GCP: RTSP → mediaMTX → frame sampler → YOLO arma → Gemini confirm → WhatsApp.
3. ADRs para: (a) elección de nube, (b) protocolo de relay (SRT vs WebRTC), (c) capa de abstracción de modelos.
4. Setup legal: borrador de contrato de tratamiento de datos + aviso en tienda + política de privacidad.
5. Crear repo con la estructura de §14 y CI básico.
