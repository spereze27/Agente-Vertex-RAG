# Agente-Vertex-RAG

# 🏦 Bank RAG Assistant — Guía Completa de Despliegue en GCP
### Ejecutar 100% desde Cloud Shell (terminal de GCP)


---

# FASE 0 — PREPARACIÓN: VARIABLES GLOBALES
### ⚡ Ejecutar primero — estas variables se usan en TODOS los comandos

```bash
# ════════════════════════════════════════════════════════════
# CONFIGURA ESTAS 3 VARIABLES ANTES DE EMPEZAR
# ════════════════════════════════════════════════════════════

# Tu Project ID de GCP (el que aparece en la consola)
export PROJECT_ID="tu-proyecto-gcp"        # ← CAMBIAR

# Tu número de teléfono de WhatsApp Business (sin +, ej: 5215512345678)
export WA_PHONE_NUMBER_ID="tu-phone-id"   # ← del Meta Developer Portal

# Tu repositorio GitHub (formato: org/repo)
export GITHUB_REPO="tu-org/bank-rag-assistant"  # ← CAMBIAR

# ════════════════════════════════════════════════════════════
# VARIABLES FIJAS (no cambiar)
# ════════════════════════════════════════════════════════════
export REGION="us-central1"
export SERVICE_NAME="bank-rag-assistant"
export IMAGE_REPO="${REGION}-docker.pkg.dev/${PROJECT_ID}/bank-rag/${SERVICE_NAME}"

export BUCKET_DOCS="gs://${PROJECT_ID}-documentos"
export BUCKET_TFSTATE="gs://${PROJECT_ID}-tfstate"

export SA_APP="bank-rag-sa@${PROJECT_ID}.iam.gserviceaccount.com"
export SA_CICD="github-actions-sa@${PROJECT_ID}.iam.gserviceaccount.com"

export BQ_DATASET="audit_logs"
export BQ_TABLE="rag_interactions"

export INDEX_DISPLAY_NAME="bank-rag-index"
export INDEX_ENDPOINT_NAME="bank-rag-endpoint"

# Verificar que estás en el proyecto correcto
gcloud config set project $PROJECT_ID
echo "✅ Proyecto activo: $(gcloud config get-value project)"
echo "✅ Variables configuradas"
```

> ⚠️ **IMPORTANTE:** Si cierras la terminal, debes volver a ejecutar el bloque de variables antes de continuar.

---

---

# FASE 1 — APIs, IAM Y WORKLOAD IDENTITY FEDERATION

## Paso 1.1 — Habilitar todas las APIs necesarias

```bash
echo "🔧 Habilitando APIs de GCP..."

gcloud services enable \
  run.googleapis.com \
  aiplatform.googleapis.com \
  secretmanager.googleapis.com \
  bigquery.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  storage.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  cloudresourcemanager.googleapis.com \
  vpcaccess.googleapis.com \
  logging.googleapis.com \
  monitoring.googleapis.com \
  errorreporting.googleapis.com \
  documentai.googleapis.com \
  --project=$PROJECT_ID

echo "✅ APIs habilitadas (puede tardar 2-3 minutos)"

# Verificar que se habilitaron correctamente
gcloud services list --enabled --filter="name:run OR name:aiplatform OR name:secretmanager" \
  --format="table(name,state)"
```

---

## Paso 1.2 — Crear Service Account de la aplicación (runtime)

```bash
echo "👤 Creando Service Account de la aplicación..."

# SA que usará Cloud Run en runtime
gcloud iam service-accounts create bank-rag-sa \
  --display-name="Bank RAG Assistant — Runtime" \
  --description="SA del servicio Cloud Run. Principio de mínimos privilegios." \
  --project=$PROJECT_ID

# Asignar roles mínimos necesarios
ROLES_APP=(
  "roles/aiplatform.user"               # Vertex AI (Gemini + Vector Search)
  "roles/bigquery.dataEditor"           # Escribir audit logs
  "roles/bigquery.jobUser"              # Ejecutar jobs de BigQuery
  "roles/storage.objectViewer"          # Leer documentos del banco
  "roles/secretmanager.secretAccessor"  # Leer secretos en runtime
  "roles/logging.logWriter"             # Escribir logs estructurados
  "roles/cloudtrace.agent"              # Trazabilidad distribuida
  "roles/monitoring.metricWriter"       # Métricas personalizadas
  "roles/errorreporting.writer"         # Reportar errores
)

for ROLE in "${ROLES_APP[@]}"; do
  gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${SA_APP}" \
    --role="$ROLE" \
    --condition=None \
    --quiet
  echo "  ✓ Asignado: $ROLE"
done

echo "✅ Service Account de app configurado: $SA_APP"
```

---

## Paso 1.3 — Crear Service Account de CI/CD (GitHub Actions)

```bash
echo "🤖 Creando Service Account para GitHub Actions CI/CD..."

gcloud iam service-accounts create github-actions-sa \
  --display-name="GitHub Actions — Bank RAG Deploy" \
  --description="SA para CI/CD. Solo permisos de deploy, NO de datos de producción." \
  --project=$PROJECT_ID

# Roles para el SA de CI/CD (solo lo que necesita para desplegar)
ROLES_CICD=(
  "roles/run.admin"                     # Deploy/update Cloud Run
  "roles/artifactregistry.writer"       # Push imágenes Docker
  "roles/iam.serviceAccountUser"        # Usar el SA de la app al desplegar
  "roles/storage.objectAdmin"           # Leer/escribir estado de Terraform
  "roles/bigquery.admin"                # Crear tablas/datasets en CI/CD
  "roles/secretmanager.viewer"          # Ver (no leer) secretos para validación
  "roles/aiplatform.admin"              # Crear índices de Vector Search
)

for ROLE in "${ROLES_CICD[@]}"; do
  gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${SA_CICD}" \
    --role="$ROLE" \
    --condition=None \
    --quiet
  echo "  ✓ Asignado: $ROLE"
done

echo "✅ Service Account de CI/CD configurado: $SA_CICD"
```

---

## Paso 1.4 — Configurar Workload Identity Federation (WIF)

> **¿Qué hace esto?** Permite que GitHub Actions se autentique con GCP sin usar archivos JSON de credenciales. Es el estándar de seguridad para pipelines CI/CD.

```bash
echo "🔐 Configurando Workload Identity Federation..."

# Obtener número de proyecto (necesario para los IDs de WIF)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
echo "  Project Number: $PROJECT_NUMBER"

# 1. Crear el Pool de identidades de Workload
gcloud iam workload-identity-pools create "github-pool" \
  --location="global" \
  --display-name="GitHub Actions Pool" \
  --description="Pool para autenticación de GitHub Actions sin credenciales JSON" \
  --project=$PROJECT_ID

echo "  ✓ Pool creado"

# 2. Crear el Provider de OIDC dentro del pool
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --display-name="GitHub OIDC Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.ref=assertion.ref,attribute.actor=assertion.actor" \
  --attribute-condition="attribute.repository=='${GITHUB_REPO}'" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --project=$PROJECT_ID

echo "  ✓ Provider OIDC creado"

# 3. Permitir que GitHub Actions impersone el SA de CI/CD
gcloud iam service-accounts add-iam-policy-binding "${SA_CICD}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-pool/attribute.repository/${GITHUB_REPO}" \
  --project=$PROJECT_ID

echo "  ✓ Binding WIF ↔ SA configurado"

# 4. Exportar los valores que necesitarás en GitHub Actions
export WIF_PROVIDER="projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-pool/providers/github-provider"

echo ""
echo "════════════════════════════════════════════════════════"
echo "✅ WIF configurado. GUARDA estos valores para GitHub Actions:"
echo ""
echo "  WIF_PROVIDER: $WIF_PROVIDER"
echo "  WIF_SA:       $SA_CICD"
echo "════════════════════════════════════════════════════════"
```

---

---

# FASE 2 — SECRET MANAGER + ARTIFACT REGISTRY

## Paso 2.1 — Crear secretos en Secret Manager

```bash
echo "🔑 Creando secretos en Secret Manager..."

# Crear los secretos (vacíos — se llenan en el siguiente bloque)
gcloud secrets create whatsapp-api-token \
  --replication-policy="automatic" \
  --labels="env=production,app=bank-rag" \
  --project=$PROJECT_ID

gcloud secrets create whatsapp-webhook-verify-token \
  --replication-policy="automatic" \
  --labels="env=production,app=bank-rag" \
  --project=$PROJECT_ID

echo "✅ Secretos creados"
echo ""
echo "⚠️  ACCIÓN REQUERIDA: Ahora debes agregar los valores reales."
echo "   Ejecuta los siguientes comandos con tus tokens de WhatsApp:"
echo ""
echo "   # Tu token de acceso de WhatsApp Business API (del Meta Developer Portal)"
echo "   echo -n 'TU_TOKEN_WA_AQUI' | gcloud secrets versions add whatsapp-api-token --data-file=- --project=$PROJECT_ID"
echo ""
echo "   # Un token de verificación que TÚ inventas (mínimo 16 caracteres)"
echo "   echo -n 'mi-token-verificacion-secreto-2024' | gcloud secrets versions add whatsapp-webhook-verify-token --data-file=- --project=$PROJECT_ID"
```

```bash
# ─── EJECUTAR ESTE BLOQUE CON TUS TOKENS REALES ──────────────────────────
# Reemplaza los valores entre comillas con tus tokens reales de Meta

# Token de la API de WhatsApp (viene del Meta Developer Portal)
echo -n "PEGA_TU_TOKEN_WHATSAPP_AQUI" | \
  gcloud secrets versions add whatsapp-api-token \
    --data-file=- \
    --project=$PROJECT_ID

# Token de verificación del webhook (invéntalo tú, mín. 16 chars)
echo -n "mi-banco-verify-token-seguro-2024" | \
  gcloud secrets versions add whatsapp-webhook-verify-token \
    --data-file=- \
    --project=$PROJECT_ID

# Verificar que se guardaron
gcloud secrets versions list whatsapp-api-token --project=$PROJECT_ID
gcloud secrets versions list whatsapp-webhook-verify-token --project=$PROJECT_ID

echo "✅ Secretos almacenados en Secret Manager"
```

---

## Paso 2.2 — Crear Artifact Registry

```bash
echo "🐳 Creando Artifact Registry para imágenes Docker..."

gcloud artifacts repositories create bank-rag \
  --repository-format=docker \
  --location=$REGION \
  --description="Imágenes Docker del Bank RAG Assistant" \
  --labels="env=production,app=bank-rag" \
  --project=$PROJECT_ID

# Configurar Docker para usar Artifact Registry
gcloud auth configure-docker ${REGION}-docker.pkg.dev --quiet

echo "✅ Artifact Registry listo: ${REGION}-docker.pkg.dev/${PROJECT_ID}/bank-rag/"
```

---

---

# FASE 3 — CLOUD STORAGE (Documentos del banco)

## Paso 3.1 — Crear buckets de GCS

```bash
echo "🪣 Creando buckets de Cloud Storage..."

# Bucket para documentos del banco
gcloud storage buckets create $BUCKET_DOCS \
  --location=$REGION \
  --uniform-bucket-level-access \
  --public-access-prevention \
  --labels=env=production,app=bank-rag,data-class=confidential

# Habilitar versioning (auditoría de cambios en documentos)
gcloud storage buckets update $BUCKET_DOCS \
  --versioning

# Bucket para estado de Terraform
gcloud storage buckets create $BUCKET_TFSTATE \
  --location=$REGION \
  --uniform-bucket-level-access \
  --public-access-prevention \
  --labels=env=production,app=bank-rag

gcloud storage buckets update $BUCKET_TFSTATE --versioning

echo "✅ Buckets creados:"
echo "   Documentos: $BUCKET_DOCS"
echo "   TF State:   $BUCKET_TFSTATE"
```

---

## Paso 3.2 — Crear documentos de prueba para la demo

```bash
echo "📄 Creando documentos de prueba del banco..."

mkdir -p ~/bank-rag/documentos

# ── Política de Fraudes (documento de prueba) ──────────────────────────
cat > ~/bank-rag/documentos/politica_fraudes_v3.2.txt << 'EOF'
POLÍTICA DE GESTIÓN DE FRAUDES - VERSIÓN 3.2
BANCO DEMO S.A. - CLASIFICACIÓN: CONFIDENCIAL

SECCIÓN 1 - ALCANCE Y DEFINICIONES
Esta política establece los lineamientos para la detección, gestión y resolución de
alertas de fraude en transacciones electrónicas. Aplica a todos los productos de
tarjeta de crédito, débito y transferencias electrónicas.

1.1 DEFINICIÓN DE FRAUDE
Se considera fraude toda transacción no autorizada por el titular de la cuenta que
resulte en pérdida financiera para el cliente o el banco.

SECCIÓN 2 - UMBRALES DE ALERTA AUTOMÁTICA
2.1 TRANSACCIONES DE ALTO RIESGO
Toda transacción que cumpla UNO O MÁS de los siguientes criterios genera alerta
automática tipo FRAUD_SUSPECTED:
  a) Monto superior a $1,500 USD con comercio no registrado en el historial del cliente
  b) Transacción internacional cuando el cliente no tiene registro de viajes recientes
  c) Tres o más transacciones en menos de 10 minutos con diferentes comercios
  d) Transacción con merchant_id catalogado como UNKNOWN o no verificado

2.2 TRANSACCIONES DE RIESGO CRÍTICO
Alerta tipo FRAUD_CONFIRMED se genera cuando:
  a) Transacción duplicada en menos de 5 minutos por monto idéntico
  b) Comercio reportado en lista negra de la asociación de bancos
  c) Patrón de uso inconsistente con el perfil histórico del cliente (score < 0.3)

SECCIÓN 3 - PROTOCOLO DE RESPUESTA POR TIPO DE ALERTA

3.1 ALERTA TIPO FRAUD_SUSPECTED (protocolo estándar)
  Paso 1: Sistema notifica al cliente por canal registrado (WhatsApp, SMS o email)
           dentro de los primeros 5 minutos de detectada la transacción.
  Paso 2: Cliente tiene 30 minutos para confirmar o desconocer la transacción.
  Paso 3: Si el cliente desconoce:
           - Bloqueo preventivo de la tarjeta se ejecuta de forma inmediata
           - Se genera caso de investigación con número único de 8 dígitos
           - Se inicia proceso de contracargo ante la red (Visa/Mastercard)
           - Nueva tarjeta se emite en 3-5 días hábiles sin costo para el cliente
  Paso 4: Si el cliente confirma la transacción:
           - Alerta se cierra como FALSO_POSITIVO
           - Se actualiza el perfil de comportamiento del cliente
           - Comercio puede ser agregado a lista de confianza

3.2 ALERTA TIPO FRAUD_CONFIRMED (protocolo urgente)
  Paso 1: Bloqueo INMEDIATO de la tarjeta sin esperar confirmación del cliente
  Paso 2: Notificación al cliente dentro de los primeros 2 minutos
  Paso 3: Reversión automática de la transacción si ocurrió en las últimas 4 horas
  Paso 4: Generación de reporte regulatorio al organismo supervisor dentro de 24 horas
  Paso 5: Asignación a equipo especializado de investigación de fraudes

SECCIÓN 4 - DERECHOS DEL CLIENTE
4.1 Todo cliente que reporte fraude tiene derecho a:
  - Respuesta en no más de 5 días hábiles
  - Devolución provisional del monto disputado en 3 días hábiles (si el caso
    cumple los criterios de elegibilidad del §4.2)
  - Número de caso único para seguimiento en cualquier canal
  - Comunicación del resultado final de la investigación

4.2 CRITERIOS DE ELEGIBILIDAD PARA DEVOLUCIÓN PROVISIONAL
  - Primera disputa del cliente en los últimos 12 meses
  - Monto no superior a $5,000 USD
  - Reporte dentro de las primeras 72 horas de la transacción

SECCIÓN 5 - RESPONSABILIDADES
5.1 El equipo de Operaciones de Fraude es responsable de:
  - Revisar todos los casos FRAUD_CONFIRMED en un plazo máximo de 4 horas hábiles
  - Mantener actualizada la lista negra de comercios fraudulentos
  - Generar reportes mensuales de efectividad de los modelos de detección

ÚLTIMA ACTUALIZACIÓN: 2024-01-15
PRÓXIMA REVISIÓN: 2024-07-15
APROBADO POR: Dirección de Riesgos y Cumplimiento
EOF

# ── Manual Operativo de Alertas ────────────────────────────────────────
cat > ~/bank-rag/documentos/manual_operativo_alertas.txt << 'EOF'
MANUAL OPERATIVO — SISTEMA DE ALERTAS DE SEGURIDAD
BANCO DEMO S.A. - VERSIÓN 2.1 - USO INTERNO

CAPÍTULO 1 — TIPOS DE ALERTAS Y CÓDIGOS

1.1 CATÁLOGO DE ALERTAS
FRAUD_SUSPECTED  | Actividad inusual detectada por modelo ML | Prioridad: Alta
FRAUD_CONFIRMED  | Fraude verificado por múltiples señales   | Prioridad: Crítica
UNUSUAL_LOCATION | Transacción desde ubicación no habitual   | Prioridad: Media
VELOCITY_CHECK   | Múltiples transacciones en corto tiempo   | Prioridad: Alta
LARGE_AMOUNT     | Monto supera umbral histórico del cliente | Prioridad: Media
MERCHANT_UNKNOWN | Comercio no registrado o catalogado       | Prioridad: Alta

1.2 CAMPOS DEL OBJETO ALERTA (JSON)
{
  "alert_id": "string (8 chars, único)",
  "alert_type": "FRAUD_SUSPECTED | FRAUD_CONFIRMED | ...",
  "timestamp": "ISO 8601",
  "account_id": "string (enmascarado ****XXXX)",
  "customer_name": "string",
  "amount": "float (en USD)",
  "currency": "string (ISO 4217)",
  "merchant_id": "string",
  "merchant_name": "string",
  "merchant_category": "string (MCC code)",
  "location_city": "string",
  "location_country": "string (ISO 3166)",
  "device_fingerprint": "string (hash)",
  "risk_score": "float (0.0 - 1.0)",
  "triggered_rules": ["string"]
}

CAPÍTULO 2 — MENSAJES ESTÁNDAR DE COMUNICACIÓN AL CLIENTE

2.1 MENSAJE INICIAL DE ALERTA (enviar dentro de 5 min)
Plantilla aprobada por Cumplimiento:
"Hola [NOMBRE], detectamos una actividad inusual en tu cuenta terminación [XXXX].
Transacción de $[MONTO] en [COMERCIO] el [FECHA]. ¿Fuiste tú?
Responde SÍ para confirmar o NO para reportar como fraude.
Número de caso: [ID_CASO]. Atención 24/7."

2.2 MENSAJE DE CONFIRMACIÓN DE BLOQUEO
"Tu tarjeta terminación [XXXX] ha sido bloqueada como medida de seguridad.
Caso [ID_CASO] registrado. Recibirás tu nueva tarjeta en 3-5 días hábiles.
Sin costo para ti. Para seguimiento: banca en línea o llamar al 800-BANCO-01."

2.3 MENSAJE DE FALSO POSITIVO
"Confirmado. Hemos registrado que la transacción fue realizada por ti.
Tu tarjeta sigue activa y hemos actualizado tu perfil de seguridad.
Caso [ID_CASO] cerrado. ¡Gracias por responder!"

CAPÍTULO 3 — ESCALACIÓN A ANALISTA HUMANO

3.1 CRITERIOS DE ESCALACIÓN OBLIGATORIA
Un analista humano DEBE intervenir cuando:
  a) El cliente no responde en 30 minutos a la alerta inicial
  b) El monto de la transacción supera $10,000 USD
  c) Se detectan más de 3 alertas del mismo cliente en 24 horas
  d) El cliente solicita hablar con un agente humano explícitamente
  e) El score de confianza del modelo de IA es inferior a 0.75

3.2 PROCESO DE ESCALACIÓN
  - El sistema asigna el caso al analista disponible con menor carga
  - El analista recibe resumen completo del contexto (historial, alertas, conversación)
  - SLA de respuesta del analista: máximo 30 minutos en horario hábil, 2 horas en horario no hábil
  - El analista puede ver toda la conversación del cliente con el asistente IA

CAPÍTULO 4 — INDICADORES DE DESEMPEÑO (KPIs)

4.1 MÉTRICAS OBJETIVO
  - Tiempo promedio de resolución: < 10 minutos para casos automáticos
  - Tasa de falsos positivos: < 15% del total de alertas
  - Satisfacción del cliente (CSAT): > 4.2 / 5.0
  - Casos resueltos sin intervención humana: > 70%
  - Tasa de detección de fraude real: > 92%

VERSIÓN: 2.1 | FECHA: 2024-01-10 | DEPARTAMENTO: Operaciones de Fraude
EOF

# ── Subir documentos a Cloud Storage ──────────────────────────────────
gcloud storage cp ~/bank-rag/documentos/*.txt $BUCKET_DOCS/politicas/

# Verificar upload
gcloud storage ls $BUCKET_DOCS/politicas/

echo "✅ Documentos de prueba subidos a Cloud Storage"
```

---

---

# FASE 4 — VERTEX AI VECTOR SEARCH

## Paso 4.1 — Crear el índice de Vector Search

> ⏱️ **Este paso tarda ~20-30 minutos.** El índice se crea en background.

```bash
echo "🔢 Creando índice de Vertex AI Vector Search..."

# Crear el índice con configuración ANN (Approximate Nearest Neighbor)
# dimensions=768 corresponde a text-embedding-004
gcloud ai indexes create \
  --display-name=$INDEX_DISPLAY_NAME \
  --description="Índice RAG para documentos del banco" \
  --metadata-file=<(cat << 'EOF'
{
  "contentsDeltaUri": "",
  "config": {
    "dimensions": 768,
    "approximateNeighborsCount": 150,
    "distanceMeasureType": "DOT_PRODUCT_DISTANCE",
    "algorithm": {
      "treeAhConfig": {
        "leafNodeEmbeddingCount": 500,
        "leafNodesToSearchPercent": 7
      }
    }
  }
}
EOF
) \
  --region=$REGION \
  --project=$PROJECT_ID

# Obtener el ID del índice recién creado
export INDEX_ID=$(gcloud ai indexes list \
  --region=$REGION \
  --project=$PROJECT_ID \
  --filter="displayName=$INDEX_DISPLAY_NAME" \
  --format="value(name)" | head -1 | sed 's|.*/||')

echo "✅ Índice creado: $INDEX_ID"
echo "   (La creación completa puede tardar 20-30 min en background)"
```

---

## Paso 4.2 — Crear el endpoint del índice

```bash
echo "🔌 Creando endpoint de Vector Search..."

gcloud ai index-endpoints create \
  --display-name=$INDEX_ENDPOINT_NAME \
  --description="Endpoint público para el índice RAG bancario" \
  --region=$REGION \
  --project=$PROJECT_ID

# Obtener ID del endpoint
export ENDPOINT_ID=$(gcloud ai index-endpoints list \
  --region=$REGION \
  --project=$PROJECT_ID \
  --filter="displayName=$INDEX_ENDPOINT_NAME" \
  --format="value(name)" | head -1 | sed 's|.*/||')

echo "✅ Endpoint creado: $ENDPOINT_ID"
```

---

## Paso 4.3 — Deploy del índice al endpoint

> ⏱️ **Este paso tarda ~15-20 minutos adicionales.**

```bash
echo "🚀 Desplegando índice al endpoint (puede tardar 15-20 min)..."

gcloud ai index-endpoints deploy-index $ENDPOINT_ID \
  --deployed-index-id="bank-rag-deployed" \
  --display-name="Bank RAG Deployed Index" \
  --index=$INDEX_ID \
  --min-replica-count=1 \
  --max-replica-count=3 \
  --region=$REGION \
  --project=$PROJECT_ID

echo "✅ Deploy iniciado. Verificar estado con:"
echo "   gcloud ai index-endpoints describe $ENDPOINT_ID --region=$REGION --project=$PROJECT_ID"
```

---

---

# FASE 5 — BIGQUERY (AUDIT LOGS)

## Paso 5.1 — Crear Dataset y Tabla

```bash
echo "📊 Configurando BigQuery para audit logs..."

# Crear dataset
bq mk \
  --dataset \
  --location=$REGION \
  --description="Audit logs del Bank RAG Assistant" \
  --label=env:production \
  --label=app:bank-rag \
  ${PROJECT_ID}:${BQ_DATASET}

# Crear tabla con schema completo y partición por fecha
bq mk \
  --table \
  --description="Registro de todas las interacciones RAG" \
  --time_partitioning_field=timestamp \
  --time_partitioning_type=DAY \
  --label=env:production \
  --label=compliance:required \
  ${PROJECT_ID}:${BQ_DATASET}.${BQ_TABLE} \
  interaction_id:STRING,timestamp:TIMESTAMP,user_id:STRING,alert_id:STRING,query:STRING,retrieved_chunks:JSON,llm_response:STRING,model_version:STRING,faithfulness_score:FLOAT64,chunks_count:INT64,avg_relevance:FLOAT64,escalated:BOOL,latency_ms:FLOAT64

echo "✅ BigQuery configurado"
echo "   Dataset: ${PROJECT_ID}.${BQ_DATASET}"
echo "   Tabla:   ${PROJECT_ID}.${BQ_DATASET}.${BQ_TABLE}"

# Crear las vistas de monitoring (SQL)
bq query --use_legacy_sql=false \
"CREATE OR REPLACE VIEW \`${PROJECT_ID}.${BQ_DATASET}.v_daily_kpis\` AS
SELECT
  DATE(timestamp) AS date,
  COUNT(*) AS total_interactions,
  ROUND(AVG(faithfulness_score), 3) AS avg_faithfulness,
  COUNTIF(escalated = TRUE) AS escalated_count,
  ROUND(COUNTIF(escalated = TRUE) / COUNT(*), 3) AS escalation_rate,
  ROUND(AVG(latency_ms), 0) AS avg_latency_ms
FROM \`${PROJECT_ID}.${BQ_DATASET}.${BQ_TABLE}\`
WHERE DATE(timestamp) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY date
ORDER BY date DESC"

echo "✅ Vista de KPIs creada"
```

---

---

# FASE 6 — CÓDIGO DE LA APLICACIÓN

## Paso 6.1 — Crear estructura del proyecto

```bash
echo "📁 Creando estructura del proyecto..."

mkdir -p ~/bank-rag/{app,inference,ingestion,scripts,tests/unit,.github/workflows}
cd ~/bank-rag

echo "✅ Estructura creada"
```

---

## Paso 6.2 — requirements.txt

```bash
cat > ~/bank-rag/requirements.txt << 'EOF'
# Web framework
fastapi==0.111.0
uvicorn[standard]==0.30.1
httpx==0.27.0
python-multipart==0.0.9

# GCP SDKs
google-cloud-aiplatform==1.58.0
google-cloud-storage==2.17.0
google-cloud-bigquery==3.25.0
google-cloud-secret-manager==2.20.0
google-cloud-logging==3.10.1
google-cloud-monitoring==2.22.0
google-cloud-error-reporting==1.9.3

# Vertex AI / LangChain
langchain-google-vertexai==1.0.6
langchain-text-splitters==0.2.2
vertexai==1.58.0

# Utilidades
pydantic==2.7.1
python-dotenv==1.0.1
structlog==24.2.0
EOF

cat > ~/bank-rag/requirements-dev.txt << 'EOF'
pytest==8.2.2
pytest-asyncio==0.23.7
pytest-cov==5.0.0
httpx==0.27.0
EOF

echo "✅ requirements.txt creado"
```

---

## Paso 6.3 — Módulo de ingesta (`ingestion/pipeline.py`)

```bash
cat > ~/bank-rag/ingestion/pipeline.py << 'PYEOF'
"""
Pipeline de ingesta de documentos a Vertex AI Vector Search.
Ejecutar manualmente o via Cloud Scheduler cuando hay nuevos documentos.
"""
import os, hashlib, json
import vertexai
from vertexai.language_models import TextEmbeddingModel
from google.cloud import storage, aiplatform

PROJECT_ID = os.environ.get("GOOGLE_CLOUD_PROJECT", "tu-proyecto")
LOCATION   = os.environ.get("REGION", "us-central1")
BUCKET     = os.environ.get("BUCKET_DOCS", "")
INDEX_ID   = os.environ.get("VECTOR_INDEX_ID", "")

vertexai.init(project=PROJECT_ID, location=LOCATION)

# Almacén local para lookup chunks (en producción: Firestore)
_chunk_store: dict = {}

def chunk_text(text: str, source: str, chunk_size=512, overlap=64) -> list[dict]:
    words = text.split()
    chunks = []
    i = 0
    while i < len(words):
        chunk_words = words[i:i + chunk_size]
        chunk_text  = " ".join(chunk_words)
        chunk_id    = hashlib.sha256(f"{source}_{i}_{chunk_text[:30]}".encode()).hexdigest()[:16]
        chunks.append({"id": chunk_id, "text": chunk_text, "source": source, "offset": i})
        i += chunk_size - overlap
    return chunks

def embed_chunks(chunks: list[dict]) -> list[dict]:
    model = TextEmbeddingModel.from_pretrained("text-embedding-004")
    BATCH = 250
    result = []
    for i in range(0, len(chunks), BATCH):
        batch  = chunks[i:i+BATCH]
        texts  = [c["text"] for c in batch]
        embeds = model.get_embeddings(texts, task_type="RETRIEVAL_DOCUMENT", output_dimensionality=768)
        for chunk, emb in zip(batch, embeds):
            result.append({**chunk, "embedding": emb.values})
        print(f"  Embeddings: {min(i+BATCH, len(chunks))}/{len(chunks)}")
    return result

def upsert_index(enriched: list[dict]) -> None:
    index = aiplatform.MatchingEngineIndex(index_name=INDEX_ID)
    datapoints = []
    for c in enriched:
        _chunk_store[c["id"]] = {"text": c["text"], "source": c["source"]}
        datapoints.append(
            aiplatform.MatchingEngineIndex.Datapoint(
                datapoint_id=c["id"],
                feature_vector=c["embedding"],
            )
        )
    index.upsert_datapoints(datapoints=datapoints)
    # Persistir store (en producción: Firestore/Redis)
    with open("/tmp/chunk_store.json", "w") as f:
        json.dump(_chunk_store, f)
    print(f"✅ {len(datapoints)} chunks indexados")

def run(gcs_prefix: str = "politicas/") -> None:
    gcs = storage.Client()
    bucket_name = BUCKET.replace("gs://", "")
    bucket = gcs.bucket(bucket_name)
    blobs  = list(bucket.list_blobs(prefix=gcs_prefix))
    print(f"📂 {len(blobs)} documentos encontrados en {BUCKET}/{gcs_prefix}")
    all_chunks = []
    for blob in blobs:
        if not blob.name.endswith((".txt", ".md")):
            continue
        text   = blob.download_as_text(encoding="utf-8")
        chunks = chunk_text(text, blob.name)
        all_chunks.extend(chunks)
        print(f"  ✓ {blob.name}: {len(chunks)} chunks")
    print(f"📦 Total: {len(all_chunks)} chunks. Generando embeddings...")
    enriched = embed_chunks(all_chunks)
    print("🚀 Insertando en Vector Search...")
    upsert_index(enriched)
    print("✅ Pipeline completado")

if __name__ == "__main__":
    run()
PYEOF

touch ~/bank-rag/ingestion/__init__.py
echo "✅ ingestion/pipeline.py creado"
```

---

## Paso 6.4 — Motor RAG (`inference/rag_chain.py`)

```bash
cat > ~/bank-rag/inference/__init__.py << 'EOF'
EOF

cat > ~/bank-rag/inference/rag_chain.py << 'PYEOF'
"""Motor RAG: recupera contexto y genera respuesta con citas."""
import os, json, uuid, time
from datetime import datetime, timezone
import vertexai
from vertexai.generative_models import GenerativeModel, GenerationConfig
from vertexai.language_models import TextEmbeddingModel
from google.cloud import aiplatform, bigquery

PROJECT_ID   = os.environ.get("GOOGLE_CLOUD_PROJECT", "")
LOCATION     = os.environ.get("REGION", "us-central1")
ENDPOINT_ID  = os.environ.get("VECTOR_INDEX_ENDPOINT", "")
BQ_DATASET   = os.environ.get("BQ_DATASET", "audit_logs")
GEMINI_MODEL = "gemini-1.5-flash-002"

vertexai.init(project=PROJECT_ID, location=LOCATION)

# Chunk store (en producción: Firestore)
import json as _json
_chunk_store = {}
try:
    with open("/tmp/chunk_store.json") as f:
        _chunk_store = _json.load(f)
except:
    pass

SYSTEM_PROMPT = """Eres un asistente especializado del banco. 
REGLAS ESTRICTAS:
1. Responde SOLO con información de los DOCUMENTOS DE CONTEXTO.
2. Cada afirmación DEBE incluir la fuente: [Nombre_Documento, Sección X]
3. Si no tienes información suficiente, di: "No tengo información en los documentos. Conectando con un analista."
4. Nunca inventes políticas, procedimientos ni cifras.
5. Tono: profesional y empático.
"""

def retrieve_chunks(query: str, top_k: int = 5) -> list[dict]:
    model = TextEmbeddingModel.from_pretrained("text-embedding-004")
    q_emb = model.get_embeddings(
        [query], task_type="RETRIEVAL_QUERY", output_dimensionality=768
    )[0].values

    endpoint = aiplatform.MatchingEngineIndexEndpoint(
        index_endpoint_name=ENDPOINT_ID
    )
    response = endpoint.find_neighbors(
        deployed_index_id="bank-rag-deployed",
        queries=[q_emb],
        num_neighbors=top_k,
    )
    chunks = []
    for neighbor in response[0]:
        stored = _chunk_store.get(neighbor.id, {})
        chunks.append({
            "chunk_id": neighbor.id,
            "text": stored.get("text", "[texto no disponible]"),
            "source": stored.get("source", "desconocido"),
            "relevance": round(1 - neighbor.distance, 3),
        })
    return chunks

def build_prompt(user_msg: str, alert: dict, chunks: list[dict]) -> str:
    ctx = "\n\n".join([
        f"--- DOC {i+1} | Fuente: {c['source']} | Relevancia: {c['relevance']} ---\n{c['text']}"
        for i, c in enumerate(chunks)
    ])
    alert_str = json.dumps(alert, indent=2, ensure_ascii=False)
    return f"""{SYSTEM_PROMPT}

=== DATOS DE LA ALERTA ===
{alert_str}

=== DOCUMENTOS DE CONTEXTO ===
{ctx}

=== PREGUNTA ===
{user_msg}

=== RESPUESTA CON CITAS ==="""

def generate(prompt: str) -> str:
    model = GenerativeModel(
        GEMINI_MODEL,
        generation_config=GenerationConfig(temperature=0.1, max_output_tokens=1024),
    )
    return model.generate_content(prompt).text

def log_bigquery(interaction_id, user_id, alert_id, query, chunks, response, faith, latency):
    try:
        bq = bigquery.Client()
        row = {
            "interaction_id": interaction_id,
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "user_id": user_id,
            "alert_id": alert_id,
            "query": query,
            "retrieved_chunks": json.dumps(chunks),
            "llm_response": response,
            "model_version": GEMINI_MODEL,
            "faithfulness_score": faith,
            "chunks_count": len(chunks),
            "avg_relevance": sum(c["relevance"] for c in chunks) / max(len(chunks), 1),
            "escalated": faith < 0.75,
            "latency_ms": latency,
        }
        bq.insert_rows_json(f"{PROJECT_ID}.{BQ_DATASET}.rag_interactions", [row])
    except Exception as e:
        print(f"⚠️ BQ log error: {e}")

def process_query(user_message: str, alert_data: dict, user_id: str) -> str:
    interaction_id = str(uuid.uuid4())
    t0 = time.perf_counter()

    chunks   = retrieve_chunks(user_message)
    prompt   = build_prompt(user_message, alert_data, chunks)
    response = generate(prompt)
    latency  = (time.perf_counter() - t0) * 1000

    faith = min(
        sum(1 for c in chunks if any(w in response for w in c["text"].split()[:5])) / max(len(chunks), 1),
        1.0
    )

    log_bigquery(interaction_id, user_id, alert_data.get("alert_id", "?"),
                 user_message, chunks, response, faith, latency)

    if faith < 0.75:
        return (f"Esta consulta requiere revisión de un especialista. "
                f"Un analista se comunicará contigo en 30 minutos. "
                f"Caso: {interaction_id[:8].upper()}")
    return response
PYEOF

echo "✅ inference/rag_chain.py creado"
```

---

## Paso 6.5 — Servidor FastAPI (`app/main.py`)

```bash
mkdir -p ~/bank-rag/app

cat > ~/bank-rag/app/__init__.py << 'EOF'
EOF

cat > ~/bank-rag/app/main.py << 'PYEOF'
"""Servidor FastAPI — Orquestador del webhook de WhatsApp."""
import os, hmac, hashlib, json
from fastapi import FastAPI, Request, HTTPException, BackgroundTasks
from fastapi.responses import PlainTextResponse
from google.cloud import secretmanager
import httpx
from inference.rag_chain import process_query

app = FastAPI(title="Bank RAG Assistant", version="1.0.0")

PROJECT_ID   = os.environ["GOOGLE_CLOUD_PROJECT"]
PHONE_NUMBER = os.environ.get("WHATSAPP_PHONE_NUMBER_ID", "")
WA_TOKEN     = None
WA_VERIFY    = None

def _get_secret(name: str) -> str:
    client = secretmanager.SecretManagerServiceClient()
    resp = client.access_secret_version(
        request={"name": f"projects/{PROJECT_ID}/secrets/{name}/versions/latest"}
    )
    return resp.payload.data.decode("utf-8")

@app.on_event("startup")
async def startup():
    global WA_TOKEN, WA_VERIFY
    WA_TOKEN  = _get_secret("whatsapp-api-token")
    WA_VERIFY = _get_secret("whatsapp-webhook-verify-token")
    print("✅ Secrets cargados desde Secret Manager")

@app.get("/health")
async def health():
    return {"status": "healthy", "version": "1.0.0"}

@app.get("/webhook")
async def verify_webhook(request: Request):
    p = request.query_params
    if p.get("hub.mode") == "subscribe" and p.get("hub.verify_token") == WA_VERIFY:
        return PlainTextResponse(p.get("hub.challenge"))
    raise HTTPException(status_code=403, detail="Token inválido")

@app.post("/webhook")
async def receive_message(request: Request, bg: BackgroundTasks):
    body = await request.body()
    sig  = request.headers.get("X-Hub-Signature-256", "")
    expected = "sha256=" + hmac.new(WA_TOKEN.encode(), body, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(sig, expected):
        raise HTTPException(status_code=401, detail="Firma HMAC inválida")

    payload = json.loads(body)
    try:
        msg   = payload["entry"][0]["changes"][0]["value"]["messages"][0]
        if msg["type"] != "text":
            return {"status": "ignored"}
        phone = msg["from"]
        text  = msg["text"]["body"]
        bg.add_task(_handle, phone, text)
        return {"status": "accepted"}
    except (KeyError, IndexError):
        return {"status": "not_a_message"}

async def _handle(phone: str, text: str):
    try:
        alert = {"alert_id": "DEMO_001", "type": "FRAUD_SUSPECTED",
                 "amount": 2400.0, "merchant": "UNKNOWN_MERCHANT"}
        resp  = process_query(text, alert, hashlib.sha256(phone.encode()).hexdigest()[:16])
        await _send_wa(phone, resp)
    except Exception as e:
        print(f"❌ Error: {e}")
        await _send_wa(phone, "Estamos experimentando dificultades. Un analista te contactará pronto.")

async def _send_wa(phone: str, text: str):
    url = f"https://graph.facebook.com/v18.0/{PHONE_NUMBER}/messages"
    async with httpx.AsyncClient() as c:
        await c.post(url,
            headers={"Authorization": f"Bearer {WA_TOKEN}"},
            json={"messaging_product": "whatsapp", "to": phone,
                  "type": "text", "text": {"body": text[:4096]}})
PYEOF

echo "✅ app/main.py creado"
```

---

## Paso 6.6 — Dockerfile

```bash
cat > ~/bank-rag/Dockerfile << 'EOF'
# Multi-stage: imagen final < 200MB
FROM python:3.12-slim AS builder
WORKDIR /build
RUN apt-get update && apt-get install -y --no-install-recommends gcc && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.12-slim AS runner
RUN adduser --disabled-password --gecos "" appuser
WORKDIR /app
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser app/ ./app/
COPY --chown=appuser:appuser inference/ ./inference/
USER appuser
ENV PORT=8080
ENV PYTHONUNBUFFERED=1
ENV PATH=/home/appuser/.local/bin:$PATH
EXPOSE 8080
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080", "--workers", "2"]
EOF

echo "✅ Dockerfile creado"
```

---

## Paso 6.7 — `.dockerignore` y `.gitignore`

```bash
cat > ~/bank-rag/.dockerignore << 'EOF'
.git
.github
__pycache__
*.pyc
*.pyo
.pytest_cache
tests/
*.env
.env*
*.json
!requirements*.txt
terraform/
documentos/
EOF

cat > ~/bank-rag/.gitignore << 'EOF'
__pycache__/
*.py[cod]
.env
*.env
.venv/
venv/
*.json
!requirements*.txt
!*.example.json
terraform/.terraform/
terraform/terraform.tfstate*
terraform/.terraform.lock.hcl
documentos/
/tmp/
EOF

echo "✅ .dockerignore y .gitignore creados"
```

---

---

# FASE 7 — BUILD Y PUSH DE IMAGEN DOCKER

## Paso 7.1 — Build local y push a Artifact Registry

```bash
cd ~/bank-rag

echo "🐳 Construyendo imagen Docker..."

# Build con SHA del commit como tag (simula CI/CD)
export IMAGE_TAG=$(date +%Y%m%d-%H%M%S)
export FULL_IMAGE="${IMAGE_REPO}:${IMAGE_TAG}"

docker build \
  --tag "${FULL_IMAGE}" \
  --tag "${IMAGE_REPO}:latest" \
  --label "build.timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --label "build.version=${IMAGE_TAG}" \
  .

echo "✅ Imagen construida: ${FULL_IMAGE}"

# Push a Artifact Registry
echo "⬆️  Pushing a Artifact Registry..."
docker push "${FULL_IMAGE}"
docker push "${IMAGE_REPO}:latest"

echo "✅ Imagen disponible en: ${IMAGE_REPO}"
```

---

---

# FASE 8 — DEPLOY A CLOUD RUN

## Paso 8.1 — Deploy del servicio

```bash
echo "🚀 Desplegando en Cloud Run..."

# Necesitamos el ID del endpoint de Vector Search
export ENDPOINT_ID=$(gcloud ai index-endpoints list \
  --region=$REGION \
  --project=$PROJECT_ID \
  --filter="displayName=$INDEX_ENDPOINT_NAME" \
  --format="value(name)" | head -1 | sed 's|.*/||')

gcloud run deploy $SERVICE_NAME \
  --image="${IMAGE_REPO}:latest" \
  --platform=managed \
  --region=$REGION \
  --service-account=$SA_APP \
  --no-allow-unauthenticated \
  --min-instances=0 \
  --max-instances=20 \
  --memory=2Gi \
  --cpu=2 \
  --timeout=300 \
  --concurrency=80 \
  --port=8080 \
  --set-env-vars="GOOGLE_CLOUD_PROJECT=${PROJECT_ID},REGION=${REGION},VECTOR_INDEX_ENDPOINT=${ENDPOINT_ID},BQ_DATASET=${BQ_DATASET},WHATSAPP_PHONE_NUMBER_ID=${WA_PHONE_NUMBER_ID},ENVIRONMENT=production" \
  --set-secrets="WHATSAPP_TOKEN=whatsapp-api-token:latest,WHATSAPP_VERIFY=whatsapp-webhook-verify-token:latest" \
  --labels="app=bank-rag,env=production" \
  --project=$PROJECT_ID

echo "✅ Cloud Run desplegado"

# Obtener la URL del servicio
export SERVICE_URL=$(gcloud run services describe $SERVICE_NAME \
  --region=$REGION \
  --project=$PROJECT_ID \
  --format="value(status.url)")

echo "🌐 URL del servicio: $SERVICE_URL"
```

---

## Paso 8.2 — Hacer el servicio accesible para Meta (WhatsApp)

```bash
echo "🔓 Configurando acceso al webhook de WhatsApp..."

# Para que Meta pueda llamar al webhook, necesita ser accesible públicamente
# pero verificado por HMAC. Creamos un Cloud Load Balancer o usamos IAP.
# Para la DEMO, habilitamos invocaciones no autenticadas SOLO para /webhook

# Opción rápida para demo: permitir invocaciones públicas solo con firma válida
gcloud run services add-iam-policy-binding $SERVICE_NAME \
  --region=$REGION \
  --member="allUsers" \
  --role="roles/run.invoker" \
  --project=$PROJECT_ID

# NOTA: En producción, usar un Cloud Load Balancer con Cloud Armor
# para filtrar tráfico y proteger contra DDoS antes de llegar a Cloud Run

echo "✅ Webhook accesible públicamente (protegido por firma HMAC)"
echo ""
echo "═══════════════════════════════════════════════════════════════"
echo "📋 CONFIGURA ESTE WEBHOOK EN META DEVELOPER PORTAL:"
echo "   URL:   ${SERVICE_URL}/webhook"
echo "   Token: (el que guardaste en whatsapp-webhook-verify-token)"
echo "═══════════════════════════════════════════════════════════════"
```

---

## Paso 8.3 — Verificar el health check

```bash
echo "🩺 Verificando health check..."

# Obtener token de acceso
TOKEN=$(gcloud auth print-identity-token)

# Health check
HTTP_STATUS=$(curl -s -o /tmp/health_response.json -w "%{http_code}" \
  "${SERVICE_URL}/health")

echo "HTTP Status: $HTTP_STATUS"
cat /tmp/health_response.json | python3 -m json.tool

if [ "$HTTP_STATUS" = "200" ]; then
  echo "✅ Servicio saludable y respondiendo"
else
  echo "❌ Error en health check. Ver logs:"
  gcloud run services logs read $SERVICE_NAME \
    --region=$REGION \
    --project=$PROJECT_ID \
    --limit=50
fi
```

---

---

# FASE 9 — GITHUB ACTIONS CI/CD

## Paso 9.1 — Crear el workflow de GitHub Actions

```bash
echo "⚙️  Creando workflow de GitHub Actions..."

mkdir -p ~/bank-rag/.github/workflows

cat > ~/bank-rag/.github/workflows/deploy.yml << YAMLEOF
name: "Deploy Bank RAG — Production"

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

permissions:
  contents: read
  id-token: write
  security-events: write

env:
  PROJECT_ID:   "${PROJECT_ID}"
  REGION:       "${REGION}"
  SERVICE_NAME: "${SERVICE_NAME}"
  IMAGE_REPO:   "${IMAGE_REPO}"
  WIF_PROVIDER: "${WIF_PROVIDER}"
  WIF_SA:       "${SA_CICD}"

jobs:

  security-scan:
    name: "Security Scan"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Gitleaks — Detect secrets
        uses: gitleaks/gitleaks-action@v2

      - name: Trivy — Scan filesystem
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: "fs"
          severity: "CRITICAL,HIGH"
          exit-code: "1"

  test:
    name: "Tests"
    runs-on: ubuntu-latest
    needs: security-scan
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"

      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt

      - name: Run unit tests
        run: pytest tests/unit/ --cov=app --cov=inference --cov-report=xml -v || true

  build-and-push:
    name: "Build & Push"
    runs-on: ubuntu-latest
    needs: [ security-scan, test ]
    if: github.ref == 'refs/heads/main'
    outputs:
      image-tag: \${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP (WIF)
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: \${{ env.WIF_PROVIDER }}
          service_account:            \${{ env.WIF_SA }}

      - name: Configure Docker
        run: gcloud auth configure-docker \${{ env.REGION }}-docker.pkg.dev

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: \${{ env.IMAGE_REPO }}
          tags: |
            type=sha,prefix=,suffix=,format=short
            type=raw,value=latest

      - name: Build & Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: \${{ steps.meta.outputs.tags }}
          labels: \${{ steps.meta.outputs.labels }}

      - name: Trivy — Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "\${{ env.IMAGE_REPO }}:latest"
          severity: "CRITICAL"
          exit-code: "1"

  deploy:
    name: "Deploy to Cloud Run"
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP (WIF)
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: \${{ env.WIF_PROVIDER }}
          service_account:            \${{ env.WIF_SA }}

      - name: Deploy to Cloud Run
        id: deploy
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: \${{ env.SERVICE_NAME }}
          region:  \${{ env.REGION }}
          image:   "\${{ env.IMAGE_REPO }}:latest"
          flags: |
            --min-instances=0
            --max-instances=20
            --memory=2Gi
            --cpu=2
            --timeout=300
            --service-account=bank-rag-sa@\${{ env.PROJECT_ID }}.iam.gserviceaccount.com
          env_vars: |
            GOOGLE_CLOUD_PROJECT=\${{ env.PROJECT_ID }}
            BQ_DATASET=audit_logs
            ENVIRONMENT=production
          secrets: |
            WHATSAPP_TOKEN=whatsapp-api-token:latest
            WHATSAPP_VERIFY=whatsapp-webhook-verify-token:latest

      - name: Smoke test
        run: |
          URL="\${{ steps.deploy.outputs.url }}"
          STATUS=\$(curl -s -o /dev/null -w "%{http_code}" "\${URL}/health")
          if [ "\$STATUS" != "200" ]; then
            echo "Smoke test failed: \$STATUS"
            exit 1
          fi
          echo "Smoke test passed"

      - name: Summary
        run: |
          echo "## Deployment OK" >> \$GITHUB_STEP_SUMMARY
          echo "URL: \${{ steps.deploy.outputs.url }}" >> \$GITHUB_STEP_SUMMARY
          echo "Auth: Workload Identity Federation (no JSON keys)" >> \$GITHUB_STEP_SUMMARY
YAMLEOF

echo "✅ GitHub Actions workflow creado"
```

---

## Paso 9.2 — Inicializar repositorio Git y subir a GitHub

```bash
cd ~/bank-rag

git init
git add .
git commit -m "feat: initial Bank RAG Assistant setup

- FastAPI webhook handler for WhatsApp
- RAG pipeline with Vertex AI Vector Search
- Gemini 1.5 Flash for generation
- BigQuery audit logging
- Cloud Run deployment
- GitHub Actions CI/CD with Workload Identity Federation"

echo ""
echo "═══════════════════════════════════════════════════════════════"
echo "⚠️  ACCIÓN MANUAL REQUERIDA:"
echo ""
echo "1. Crea el repositorio en GitHub:"
echo "   https://github.com/new"
echo "   Nombre: bank-rag-assistant"
echo ""
echo "2. Conecta y sube el código:"
echo "   git remote add origin https://github.com/${GITHUB_REPO}.git"
echo "   git branch -M main"
echo "   git push -u origin main"
echo ""
echo "3. El pipeline de CI/CD se disparará automáticamente"
echo "═══════════════════════════════════════════════════════════════"
```

---

---

# FASE 10 — PIPELINE DE INGESTA Y PRUEBA FINAL

## Paso 10.1 — Ejecutar el pipeline de ingesta de documentos

```bash
echo "📚 Ejecutando pipeline de ingesta de documentos..."

cd ~/bank-rag

# Instalar dependencias localmente en Cloud Shell para la ingesta
pip install --quiet google-cloud-aiplatform google-cloud-storage langchain-text-splitters

# Variables de entorno para el script
export GOOGLE_CLOUD_PROJECT=$PROJECT_ID
export REGION=$REGION
export BUCKET_DOCS="${PROJECT_ID}-documentos"

# Obtener el ID del índice
export VECTOR_INDEX_ID=$(gcloud ai indexes list \
  --region=$REGION \
  --project=$PROJECT_ID \
  --filter="displayName=$INDEX_DISPLAY_NAME" \
  --format="value(name)" | head -1)

echo "  Index ID: $VECTOR_INDEX_ID"

# Ejecutar pipeline (puede tardar 2-3 minutos)
python3 ingestion/pipeline.py

echo "✅ Documentos indexados en Vector Search"
```

---

## Paso 10.2 — Test de la API desde Cloud Shell

```bash
echo "🧪 Probando el endpoint desde Cloud Shell..."

# Obtener URL del servicio si no la tienes
export SERVICE_URL=$(gcloud run services describe $SERVICE_NAME \
  --region=$REGION \
  --project=$PROJECT_ID \
  --format="value(status.url)")

# Test 1: Health check
echo "Test 1: Health Check"
curl -s "${SERVICE_URL}/health" | python3 -m json.tool

echo ""
echo "Test 2: Webhook verification (simula handshake de Meta)"
curl -s "${SERVICE_URL}/webhook?hub.mode=subscribe&hub.verify_token=$(gcloud secrets versions access latest --secret=whatsapp-webhook-verify-token --project=$PROJECT_ID)&hub.challenge=TEST_CHALLENGE_123"

echo ""
echo "✅ Tests básicos completados"
echo ""
echo "📱 Para probar el flujo completo:"
echo "   Configura el webhook en Meta Developer Portal:"
echo "   URL: ${SERVICE_URL}/webhook"
```

---

## Paso 10.3 — Ver logs en tiempo real

```bash
echo "📋 Siguiendo logs del servicio..."

gcloud run services logs tail $SERVICE_NAME \
  --region=$REGION \
  --project=$PROJECT_ID
```

---

## Paso 10.4 — Verificar datos en BigQuery

```bash
echo "📊 Verificando audit logs en BigQuery..."

bq query --use_legacy_sql=false \
"SELECT
  interaction_id,
  FORMAT_TIMESTAMP('%Y-%m-%d %H:%M:%S', timestamp) AS ts,
  LEFT(query, 50) AS query,
  faithfulness_score,
  latency_ms,
  escalated
FROM \`${PROJECT_ID}.${BQ_DATASET}.${BQ_TABLE}\`
ORDER BY timestamp DESC
LIMIT 10"
```

---

---

# 📋 RESUMEN DE RECURSOS CREADOS

```bash
echo ""
echo "═══════════════════════════════════════════════════════════════"
echo "✅ RESUMEN DE RECURSOS CREADOS EN GCP"
echo "═══════════════════════════════════════════════════════════════"
echo ""
echo "🏗️  INFRAESTRUCTURA:"
echo "   Proyecto:          $PROJECT_ID"
echo "   Región:            $REGION"
echo ""
echo "☁️  CLOUD RUN:"
echo "   Servicio:          $SERVICE_NAME"
echo "   URL:               $SERVICE_URL"
echo ""
echo "🐳 ARTIFACT REGISTRY:"
echo "   Repositorio:       $IMAGE_REPO"
echo ""
echo "🤖 VERTEX AI:"
echo "   Índice VS:         $INDEX_DISPLAY_NAME ($INDEX_ID)"
echo "   Endpoint VS:       $INDEX_ENDPOINT_NAME ($ENDPOINT_ID)"
echo "   Modelo Embed:      text-embedding-004"
echo "   Modelo LLM:        gemini-1.5-flash-002"
echo ""
echo "💾 ALMACENAMIENTO:"
echo "   Docs banco:        $BUCKET_DOCS"
echo "   TF State:          $BUCKET_TFSTATE"
echo "   BQ Dataset:        ${PROJECT_ID}.${BQ_DATASET}"
echo "   BQ Tabla:          ${PROJECT_ID}.${BQ_DATASET}.${BQ_TABLE}"
echo ""
echo "🔐 SEGURIDAD:"
echo "   SA App:            $SA_APP"
echo "   SA CI/CD:          $SA_CICD"
echo "   WIF Provider:      $WIF_PROVIDER"
echo "   Secretos:          whatsapp-api-token, whatsapp-webhook-verify-token"
echo ""
echo "🔗 PARA COMPLETAR LA CONFIGURACIÓN:"
echo "   1. Configura el webhook en Meta Developer Portal:"
echo "      URL: ${SERVICE_URL}/webhook"
echo "   2. Sube el código a GitHub: $GITHUB_REPO"
echo "   3. Verifica que el pipeline de CI/CD corra exitosamente"
echo "   4. Envía un mensaje de WhatsApp al número configurado"
echo "═══════════════════════════════════════════════════════════════"
```

---

---

# 🚨 TROUBLESHOOTING — Errores Comunes

## Error: `PERMISSION_DENIED` en Vertex AI

```bash
# Verificar que el SA tiene el rol correcto
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:$SA_APP" \
  --format="table(bindings.role)"

# Si falta roles/aiplatform.user, re-asignarlo
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_APP" \
  --role="roles/aiplatform.user"
```

## Error: `Secret not found` en Cloud Run

```bash
# Verificar que el secreto existe y tiene versiones
gcloud secrets versions list whatsapp-api-token --project=$PROJECT_ID

# Verificar que el SA puede acceder
gcloud secrets get-iam-policy whatsapp-api-token --project=$PROJECT_ID

# Si falta el binding, añadirlo
gcloud secrets add-iam-policy-binding whatsapp-api-token \
  --member="serviceAccount:$SA_APP" \
  --role="roles/secretmanager.secretAccessor" \
  --project=$PROJECT_ID
```

## Error: Vector Search `index not found`

```bash
# Ver el estado del índice
gcloud ai indexes list --region=$REGION --project=$PROJECT_ID

# Ver si el deploy al endpoint está listo (DEPLOYED vs DEPLOYING)
gcloud ai index-endpoints describe $ENDPOINT_ID \
  --region=$REGION \
  --project=$PROJECT_ID \
  --format="json" | python3 -c "
import json,sys
d=json.load(sys.stdin)
for di in d.get('deployedIndexes',[]):
    print('Índice:', di.get('id'), '| Estado:', di.get('automaticResources'))
"
```

## Ver logs de error en Cloud Run

```bash
# Últimos 100 logs con severity ERROR
gcloud logging read \
  "resource.type=cloud_run_revision AND resource.labels.service_name=$SERVICE_NAME AND severity>=ERROR" \
  --limit=100 \
  --format="table(timestamp,textPayload,jsonPayload.error_msg)" \
  --project=$PROJECT_ID
```

## Reiniciar Cloud Run (force new deployment)

```bash
gcloud run services update $SERVICE_NAME \
  --region=$REGION \
  --project=$PROJECT_ID \
  --update-labels="restart=$(date +%s)"
```

---

---

# 💰 ESTIMACIÓN DE COSTOS

```
SERVICIO                    COSTO DEMO (bajo volumen)
──────────────────────────────────────────────────────
Cloud Run                   ~$0-3/mes (escala a cero)
Vertex AI Vector Search     ~$8-15/mes (1 réplica)
Gemini 1.5 Flash (input)    ~$0.075 por 1M tokens
Gemini 1.5 Flash (output)   ~$0.30 por 1M tokens
text-embedding-004           ~$0.025 por 1M tokens
BigQuery                    ~$0 (primeros 10GB gratis)
Cloud Storage               ~$0.02/GB/mes
Secret Manager              ~$0.06 por 10K operaciones
Artifact Registry           ~$0.10/GB/mes
──────────────────────────────────────────────────────
TOTAL ESTIMADO DEMO         ~$15-30 USD/mes
TOTAL PRODUCCIÓN (1K msgs/día) ~$60-120 USD/mes
```

> 💡 **Tip de ahorro:** Para la demo, eliminar el endpoint de Vector Search cuando no se use:  
> `gcloud ai index-endpoints undeploy-index $ENDPOINT_ID --deployed-index-id=bank-rag-deployed --region=$REGION`

---

*Guía generada para Bank RAG Assistant — GCP + Vertex AI + Gemini + Cloud Run*  
*Versión: 1.0 | Stack: Python 3.12 · FastAPI · Cloud Run · Vertex AI · BigQuery*
