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
- **Anonimización fuerte obligatoria** para cualquier clip que salga del *tenant* del cliente (etiquetado interno o crowdsourced, ver §17): blur facial + voz + matrículas + tatuajes/uniformes distintivos. El consentimiento del dueño para ceder clips anonimizados se firma como cláusula opt-in del contrato.
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
- **Crowdsourced labeling Tier 1** (ver §17): botones en alerta WhatsApp/dashboard ("Sí era robo / Falsa alarma / Sospechoso") alimentando *active learning*.
- **Meta**: 30 tiendas pagas, < 2 falsos positivos por tienda por día.

### Fase 2 — Expansión (3 meses)

- Audio (gunshot, gritos, vidrio, KWS de atraco).
- Integración POS (Bold + Loyverse para empezar): *sweethearting*, *void* sospechoso.
- App móvil.
- Modelos propios entrenados con datos de Fase 0–1 (arma, *concealment*).
- **Crowdsourced labeling Tier 2** (ver §17): pipeline de anonimización fuerte + portal interno de etiquetado para equipo Celabot y operadores de confianza.
- **Meta**: 200 tiendas, NPS ≥ 50.

### Fase 3 — Plataforma (6 meses)

- Marketplace de "detectores" (alcohol a menores, mascotas, conteo de gente para *queue management*).
- Integración con seguridad privada y línea 123.
- Modo edge (upsell): caja con Jetson Orin Nano para tiendas con mala conectividad.
- **Crowdsourced labeling Tier 3** (ver §17): app pública gamificada "Detective Celabot" con reputación, gold standard y consenso.
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
│   ├── ops/                    # Cron, retención, billing
│   ├── anonymizer/             # Blur facial/voz/matrículas (§17)
│   └── labeling/               # Servicio de etiquetado Tier 1/2/3 (§17)
├── web/                        # Next.js dashboard
├── mobile/                     # React Native app (Fase 1.5)
├── detective/                  # App pública "Detective Celabot" (Fase 3, §17)
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
7. **Incentivos de Tier 3** (§17): ¿puntos canjeables, sorteos mensuales, donación a fundación, descuento en suscripción si el etiquetador es también cliente? Decisión pendiente; evitar pago por etiqueta para no incentivar *click farms*.

---

## 16. Próximos pasos concretos (siguientes 2 semanas)

1. Validar con 3–5 dueños de tienda en Bogotá: entrevistas de 30 min, foco en disposición de pago y casos de uso reales.
2. Prototipo end-to-end con **una sola cámara, un solo caso (arma)** en GCP: RTSP → mediaMTX → frame sampler → YOLO arma → Gemini confirm → WhatsApp.
3. ADRs para: (a) elección de nube, (b) protocolo de relay (SRT vs WebRTC), (c) capa de abstracción de modelos.
4. Setup legal: borrador de contrato de tratamiento de datos + aviso en tienda + política de privacidad **incluyendo cláusula opt-in para ceder clips anonimizados al programa de etiquetado (§17)**.
5. Crear repo con la estructura de §14 y CI básico.
6. Diseñar las **tres preguntas observables** del Tier 1 (§17.2) y las plantillas de botones de WhatsApp asociadas.

---

## 17. Crowdsourced labeling — "¿iba a robar o no?"

> Objetivo: convertir cada alerta y cada clip ambiguo en una etiqueta utilizable para mejorar los modelos, sin sesgar el sistema hacia predicciones de intención ni violar privacidad.

### 17.1 Principio: etiquetar comportamiento observable, no intención

Nunca preguntamos "¿iba a robar?" — esa pregunta no tiene verdad verificable y arrastra sesgos del etiquetador (raza, vestimenta, edad). Preguntamos por **hechos observables** y por el **resultado** cuando se conoce:

- Comportamiento: ¿ocultó un objeto? ¿salió sin pasar por caja? ¿hubo agresión?
- Resultado (gold): ¿el inventario faltó? ¿el dueño confirmó hurto consumado? ¿se mostró arma?

Los modelos se entrenan contra estas etiquetas factuales. Una "predicción de robo" del sistema en producción es la salida del clasificador combinando esas señales — no una etiqueta de entrenamiento.

### 17.2 Tres tiers con audiencias y riesgos distintos

| Tier | Audiencia | Fase | Riesgo legal | Función |
|------|-----------|------|--------------|---------|
| **1** | Dueño / empleado del comercio sobre **sus propias** alertas | **MVP / Fase 1** | Bajo (datos propios) | *Active learning* de alta señal, mejora alertas del mismo cliente |
| **2** | Equipo Celabot + operadores de confianza con NDA | **Fase 2** | Medio (clips anonimizados, contrato) | Etiquetado profesional de clips ambiguos y casos raros |
| **3** | Crowd público gamificado ("Detective Celabot") | **Fase 3** | Alto (público), exige anonimización fuerte + ToS + opt-in del cliente | Volumen masivo de etiquetas, marketing y *brand awareness* |

### 17.3 Tier 1 — Dueño / empleado (MVP)

**Dónde**: botones en la alerta WhatsApp y en el dashboard. También accesible desde "Historial de alertas".

**Preguntas (responde el cliente):**
1. ¿Esta alerta era real? *(Sí / No / No estoy seguro)*
2. *(Si sí)* ¿Qué pasó exactamente? *(Ocultó un objeto / Salió sin pagar / Agresión / Arma / Otro)*
3. *(Opcional, 24 h después)* ¿Confirmaste pérdida de inventario asociada? *(Sí / No / No revisé)*

**Reglas:**
- La etiqueta del dueño se considera de **alta confianza** pero no infalible. Pesa 0.8 vs 1.0 de una etiqueta gold verificada con inventario.
- Si el dueño marca "falsa alarma" tres veces seguidas en el mismo tipo de evento, se sube el umbral de confianza del modelo para ese cliente automáticamente.
- Todo se versiona; el dueño puede corregir su propia etiqueta dentro de 7 días.

**Volumen esperado**: 100% de las alertas, ~50 al mes por tienda = ~5 000 etiquetas/mes con 100 tiendas.

### 17.4 Tier 2 — Equipo interno (Fase 2)

**Quién**: 3–5 operadores Celabot con contrato, NDA y entrenamiento. Pueden ser contractors externos (empresas de etiquetado tipo iMerit, Sama, o local).

**Qué reciben**: clips **ya anonimizados** (§17.6), priorizados por *active learning* (incertidumbre del VLM en 0.3–0.7) o clips donde el dueño marcó "no estoy seguro".

**Interfaz**: portal web propio con teclas rápidas, similar a CVAT pero con flujo simplificado de pregunta sí/no/no se ve, tres veces (las tres preguntas observables).

**Calidad**:
- **3 etiquetadores por clip**, consenso por mayoría (algoritmo Dawid-Skene cuando hay >3).
- **5% de gold standard intercalado** (clips con verdad conocida por inventario) para medir reputación.
- Etiquetadores por debajo de 85% de acierto en gold se reentrenan o se reemplazan.

**Volumen**: ~10 000 clips/mes en Fase 2.

### 17.5 Tier 3 — App pública "Detective Celabot" (Fase 3)

**Producto**: app móvil + web gamificada. El usuario ve un clip de 8 s anonimizado y contesta una pregunta observable. Mecánicas:

- **Tinder-style**: swipe izquierda (no pasó) / derecha (sí pasó) / arriba (no se ve).
- **Rachas y niveles**: "Detective Junior" → "Detective Senior" → "Inspector".
- **Leaderboards** semanales por ciudad.
- **Incentivos** (ver §15, decisión 7): puntos canjeables, sorteos mensuales (mercado, bonos), donación a fundación, descuento si el etiquetador también es cliente. **Sin pago por etiqueta** para evitar *click farms*.

**Calidad**:
- Cada clip se muestra a **5 etiquetadores** mínimo; se requiere consenso ≥ 4/5 para considerar etiqueta firme.
- Reputación con peso bayesiano; etiquetadores nuevos pesan poco hasta acumular gold.
- Detección de bots y patrones de respuesta uniforme.
- Cuarentena automática si la tasa de acierto en gold baja de 70%.

**Onboarding**: tutorial obligatorio con 10 clips de práctica antes de etiquetar producción.

**Marketing**: la app es también canal de adquisición. Slogan tipo "Ayuda a proteger las tiendas de tu barrio. Conviértete en Detective Celabot."

### 17.6 Pipeline de anonimización (requisito para Tier 2 y 3)

Antes de que un clip salga del *tenant* del cliente:

1. **Detección + blur facial** (RetinaFace o YOLO-face) con seguimiento para mantener blur a lo largo del clip.
2. **Blur de matrículas** (LP detector).
3. **Voz**: silenciado total o sustitución por audio sintético neutral (Fase 2 cuando exista pipeline de audio).
4. **Tatuajes / uniformes con texto / placas de empleado**: detector específico + blur.
5. **Metadatos limpiados**: sin nombre de tienda, dirección, hora exacta (solo franja horaria), ni ID de cámara identificable.
6. **Verificación humana** en muestra aleatoria del 1% antes de enviar a Tier 3, hecha por Tier 2.

Si el pipeline falla en cualquier paso, el clip **no sale** del tenant.

### 17.7 Loop de aprendizaje cerrado

```
Alerta → Tier 1 (dueño) ──┬─► etiqueta confirmada → gold (peso 0.8)
                          │
                          └─► "no estoy seguro" / VLM incierto → Tier 2 → consenso 3/3 (peso 0.9)
                                                              │
                                                              └─► sigue incierto → Tier 3 → consenso 4/5 (peso 0.6)

Inventario / dueño reporta hurto consumado → gold absoluto (peso 1.0) → re-etiqueta clips relacionados
```

Cada etiqueta entra al *feature store* con: peso, tier, identidad anónima del etiquetador, timestamp, consenso. El entrenamiento usa pesos como *sample weights*.

### 17.8 Riesgos específicos del crowdsourcing

| Riesgo | Mitigación |
|--------|------------|
| Filtración de clip identificable | Verificación humana del pipeline de anonimización; auditoría externa anual; clip *kill switch* (botón para retirar clip del programa) |
| Sesgo demográfico introducido por crowd | Auditoría periódica de tasas de etiquetado por demografía aparente; balanceo de exposición de clips |
| Etiquetadores recrean intención ("se ve sospechoso") | Preguntas estrictamente observables; gold con verdad de inventario domina el peso |
| Adversarios etiquetando para sabotear modelo | Reputación + gold + límite de etiquetas por usuario/día + detección de patrones |
| Cliente revoca consentimiento de ceder clip | Tombstone que propaga borrado a copias en Tier 2/3 y datasets de entrenamiento; *retraining* programado |
| Ley 1581: derecho de supresión de la persona grabada | Hash perceptual del clip permite localizar y eliminar; portal ARCO ya cubierto en §7 |

### 17.9 Métricas del programa

- **Cobertura**: % de alertas con al menos una etiqueta humana.
- **Latencia de etiqueta**: tiempo desde alerta hasta consenso firme.
- **Calidad por tier**: acuerdo con gold (precisión vs verdad de inventario).
- **Impacto en modelo**: ΔF1 del modelo entre versión sin/con etiquetas crowd.
- **Costo por etiqueta utilizable** (después de filtros de calidad).
- **Engagement Tier 3**: DAU, etiquetas por sesión, retención D7/D30.
