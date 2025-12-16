# Riwi Wallet - N8N Automation Service

**[English](#english) | [Español](#español)**

---

## English

### What is the N8N Automation Service?

The N8N Automation Service is a workflow automation platform that enables RiwiWallet to automatically process financial transactions from multiple sources. It reads bank notification emails, parses the transaction data, and saves it directly to the database without manual intervention.

### Purpose

This service was created to:

- **Automatically parse bank emails** from Bancolombia and Nequi
- **Extract transaction data** (amount, type, category) from email content
- **Create transactions automatically** in the RiwiWallet system
- **Execute AI-driven actions** based on structured intent commands
- **Reduce manual data entry** for users by 100%

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         EMAIL PROVIDERS                              │
│                  (Gmail, Bancolombia, Nequi)                        │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         N8N PLATFORM                                 │
│                  (https://n8n.avaricia.crudzaso.com)                │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    TRIGGER LAYER                               │  │
│  │  ┌─────────────────────┐  ┌─────────────────────────────────┐ │  │
│  │  │   Gmail Trigger     │  │   Webhook Trigger               │ │  │
│  │  │   (Every Minute)    │  │   (AI Intent Receiver)          │ │  │
│  │  └─────────────────────┘  └─────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                    │                                 │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    PROCESSING LAYER                            │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │              IF Node (Filter by Bank)                    │  │  │
│  │  │   Checks if email is from Bancolombia or Nequi           │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  │                              │                                 │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │              JavaScript Code Node                        │  │  │
│  │  │   - Detects bank (Bancolombia, Nequi, Davivienda)       │  │  │
│  │  │   - Extracts amount ($150.000 → 150000)                 │  │  │
│  │  │   - Classifies type (Income/Expense)                    │  │  │
│  │  │   - Extracts description/merchant                        │  │  │
│  │  │   - Gets user email for ID lookup                        │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                    │                                 │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    API LAYER                                   │  │
│  │  ┌─────────────────────────┐  ┌─────────────────────────────┐ │  │
│  │  │   HTTP Request 1        │  │   HTTP Request 2            │ │  │
│  │  │   GET /User/email/{x}   │  │   POST /Transaction         │ │  │
│  │  │   (Fetch User ID)       │  │   (Create Transaction)      │ │  │
│  │  └─────────────────────────┘  └─────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    .NET BACKEND (Core Service)                       │
│              (https://api.avaricia.crudzaso.com)                     │
│   ┌──────────┐  ┌──────────────┐  ┌────────────────┐               │
│   │  Users   │  │ Transactions │  │ FinancialRules │               │
│   └──────────┘  └──────────────┘  └────────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        POSTGRESQL DATABASE                           │
│                     (157.90.251.124:5432)                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Technology Stack

| Technology | Purpose |
|------------|---------|
| n8n | Workflow automation platform |
| Node.js | JavaScript runtime for code nodes |
| Gmail OAuth2 | Email access authentication |
| REST API | Communication with .NET backend |
| PostgreSQL | Database storage (via backend) |
| Docker | Containerization |

### Workflows

#### 1. Parse Email - Bancolombia/Nequi

**Trigger**: Gmail Trigger (Every Minute)

**Purpose**: Automatically read and parse bank notification emails.

| Node | Type | Function |
|------|------|----------|
| Gmail Trigger | Trigger | Polls Gmail every 60 seconds for new emails |
| If | Condition | Filters emails from Bancolombia or Nequi |
| Code in JavaScript | Code | Parses email content and extracts transaction data |
| HTTP Request | API | Fetches user ID by email address |
| HTTP Request1 | API | Creates the transaction in the backend |

**Supported Banks**:

| Bank | Detection Method | Example Format |
|------|------------------|----------------|
| Bancolombia | From contains "bancolombia" | `$150.000` |
| Nequi | From contains "nequi" | `$150.000` |
| Davivienda | From contains "davivienda" | `$150.000` |

**Transaction Type Detection**:

| Keywords | Type |
|----------|------|
| "recibiste", "te enviaron", "abono", "transferencia recibida" | Income |
| All other transactions | Expense |

#### 2. AI Financial Assistant

**Trigger**: Webhook (`POST /webhook/ai-action`)

**Purpose**: Execute financial operations based on AI intent classification.

| Intent | API Endpoint | Description |
|--------|--------------|-------------|
| `validate_expense` | POST /SpendingValidation/validate | Check if user can afford expense |
| `create_expense` | POST /Transaction | Create expense transaction |
| `create_income` | POST /Transaction | Create income transaction |
| `get_balance` | GET /User/{id}/balance | Get user balance |
| `list_transactions` | GET /Transaction/user/{id} | List user transactions |
| `create_rule` | POST /FinancialRule | Create financial rule |
| `list_rules` | GET /FinancialRule/user/{id} | List financial rules |
| `get_summary` | GET /Transaction/user/{id}/category-summary | Get spending summary |

**Expected JSON Payload**:

```json
{
  "intent": "create_expense",
  "userId": "e491896e-bc5c-4a2c-a20e-493aa5972281",
  "amount": 50000,
  "category": "Comida",
  "description": "Almuerzo",
  "type": "Expense"
}
```

### Email Parsing Logic

The JavaScript code node implements the following parsing logic:

```javascript
// 1. DETECT BANK
let banco = 'Otro';
if (from.toLowerCase().includes('bancolombia')) {
  banco = 'Bancolombia';
} else if (from.toLowerCase().includes('nequi')) {
  banco = 'Nequi';
}

// 2. DETECT TYPE (Income or Expense)
let type = 'Expense'; // Default is expense
if (body.includes('recibiste') || body.includes('abono')) {
  type = 'Income';
}

// 3. EXTRACT AMOUNT (Colombian format $150.000)
const montoMatch = snippet.match(/\$\s?([\d.]+(?:,\d{2})?)/);
let amount = 0;
if (montoMatch) {
  amount = parseFloat(montoMatch[1].replace(/\./g, '').replace(',', '.'));
}

// 4. EXTRACT USER EMAIL
let userEmail = to;
const emailMatch = to.match(/<([^>]+)>/);
if (emailMatch) {
  userEmail = emailMatch[1];
}
```

### API Integration

#### Backend Endpoints Used

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/User/email/{email}` | Get user by email address |
| POST | `/api/Transaction` | Create a new transaction |
| GET | `/api/User/{id}/balance` | Get user balance |
| POST | `/api/SpendingValidation/validate` | Validate spending against rules |
| POST | `/api/FinancialRule` | Create financial rule |

#### Data Types

| Field | Type | Values |
|-------|------|--------|
| Type | String | "Income", "Expense" |
| Source | String | "Manual", "Telegram", "WhatsApp", "Automatic" |
| RuleType | String | "SpendingLimit", "SavingsGoal", "CategoryBudget" |
| RulePeriod | String | "Daily", "Weekly", "Biweekly", "Monthly", "Yearly" |

### Configuration

#### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `BACKEND_URL` | Backend API base URL | `https://api.avaricia.crudzaso.com` |
| `N8N_HOST` | N8N instance URL | `https://n8n.avaricia.crudzaso.com` |

#### Workflow Settings

| Setting | Recommended Value |
|---------|-------------------|
| Execution Order | v1 (recommended) |
| Save failed production executions | Save |
| Save successful production executions | Do not save |
| Timeout Workflow | 2 minutes |

### Deployment

#### Production URLs

| Service | URL |
|---------|-----|
| n8n Instance | https://n8n.avaricia.crudzaso.com |
| Backend API | https://api.avaricia.crudzaso.com |
| Swagger UI | https://api.avaricia.crudzaso.com/swagger |

#### Import Workflow

```bash
# Clone repository
git clone https://github.com/Team-Avaricia/riwiwallet-n8n-service.git
cd riwiwallet-n8n-service

# Export N8N API key
export N8N_API_KEY="your-api-key"

# Import workflows
./scripts/import-workflows.sh
```

### Troubleshooting

#### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Execution stuck in "Queued" | n8n worker overloaded | Restart n8n container |
| 400 Bad Request | Invalid User ID | Check if user exists in database |
| Gmail Trigger not working | OAuth token expired | Re-authenticate Gmail credentials |
| Amount = 0 | Email format not recognized | Update regex pattern |

#### Verify Workflow Execution

1. Check n8n Executions tab
2. Look for Success/Error status
3. Inspect node outputs for data flow
4. Check database for created transactions

---

## Español

### Qué es el Servicio de Automatización N8N?

El Servicio de Automatización N8N es una plataforma de automatización de flujos de trabajo que permite a RiwiWallet procesar automáticamente transacciones financieras desde múltiples fuentes. Lee correos de notificación bancaria, analiza los datos de la transacción y los guarda directamente en la base de datos sin intervención manual.

### Propósito

Este servicio fue creado para:

- **Analizar automáticamente correos bancarios** de Bancolombia y Nequi
- **Extraer datos de transacciones** (monto, tipo, categoría) del contenido del correo
- **Crear transacciones automáticamente** en el sistema RiwiWallet
- **Ejecutar acciones impulsadas por IA** basadas en comandos de intención estructurados
- **Reducir la entrada manual de datos** para los usuarios en un 100%

### Arquitectura

```
┌─────────────────────────────────────────────────────────────────────┐
│                      PROVEEDORES DE EMAIL                            │
│                  (Gmail, Bancolombia, Nequi)                        │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       PLATAFORMA N8N                                 │
│                  (https://n8n.avaricia.crudzaso.com)                │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    CAPA DE DISPARADORES                        │  │
│  │  ┌─────────────────────┐  ┌─────────────────────────────────┐ │  │
│  │  │   Gmail Trigger     │  │   Webhook Trigger               │ │  │
│  │  │   (Cada Minuto)     │  │   (Receptor de Intenciones IA)  │ │  │
│  │  └─────────────────────┘  └─────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                    │                                 │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    CAPA DE PROCESAMIENTO                       │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │              Nodo IF (Filtrar por Banco)                 │  │  │
│  │  │   Verifica si el correo es de Bancolombia o Nequi        │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  │                              │                                 │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │              Nodo de Código JavaScript                   │  │  │
│  │  │   - Detecta banco (Bancolombia, Nequi, Davivienda)      │  │  │
│  │  │   - Extrae monto ($150.000 → 150000)                    │  │  │
│  │  │   - Clasifica tipo (Ingreso/Gasto)                      │  │  │
│  │  │   - Extrae descripción/comercio                          │  │  │
│  │  │   - Obtiene email del usuario para buscar ID            │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                    │                                 │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    CAPA DE API                                 │  │
│  │  ┌─────────────────────────┐  ┌─────────────────────────────┐ │  │
│  │  │   HTTP Request 1        │  │   HTTP Request 2            │ │  │
│  │  │   GET /User/email/{x}   │  │   POST /Transaction         │ │  │
│  │  │   (Obtener ID Usuario)  │  │   (Crear Transacción)       │ │  │
│  │  └─────────────────────────┘  └─────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    BACKEND .NET (Servicio Core)                      │
│              (https://api.avaricia.crudzaso.com)                     │
│   ┌──────────┐  ┌──────────────┐  ┌────────────────┐               │
│   │ Usuarios │  │ Transacciones│  │ ReglasFinanc.  │               │
│   └──────────┘  └──────────────┘  └────────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      BASE DE DATOS POSTGRESQL                        │
│                     (157.90.251.124:5432)                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Stack Tecnológico

| Tecnología | Propósito |
|------------|-----------|
| n8n | Plataforma de automatización de flujos de trabajo |
| Node.js | Runtime de JavaScript para nodos de código |
| Gmail OAuth2 | Autenticación de acceso a correo |
| REST API | Comunicación con backend .NET |
| PostgreSQL | Almacenamiento en base de datos (vía backend) |
| Docker | Contenedorización |

### Flujos de Trabajo

#### 1. Parse Email - Bancolombia/Nequi

**Disparador**: Gmail Trigger (Cada Minuto)

**Propósito**: Leer y analizar automáticamente correos de notificación bancaria.

| Nodo | Tipo | Función |
|------|------|---------|
| Gmail Trigger | Disparador | Consulta Gmail cada 60 segundos por nuevos correos |
| If | Condición | Filtra correos de Bancolombia o Nequi |
| Code in JavaScript | Código | Analiza contenido del correo y extrae datos de transacción |
| HTTP Request | API | Obtiene ID de usuario por dirección de correo |
| HTTP Request1 | API | Crea la transacción en el backend |

**Bancos Soportados**:

| Banco | Método de Detección | Formato de Ejemplo |
|-------|---------------------|-------------------|
| Bancolombia | From contiene "bancolombia" | `$150.000` |
| Nequi | From contiene "nequi" | `$150.000` |
| Davivienda | From contiene "davivienda" | `$150.000` |

**Detección de Tipo de Transacción**:

| Palabras Clave | Tipo |
|----------------|------|
| "recibiste", "te enviaron", "abono", "transferencia recibida" | Ingreso |
| Todas las demás transacciones | Gasto |

#### 2. Asistente Financiero IA

**Disparador**: Webhook (`POST /webhook/ai-action`)

**Propósito**: Ejecutar operaciones financieras basadas en clasificación de intenciones de IA.

| Intención | Endpoint API | Descripción |
|-----------|--------------|-------------|
| `validate_expense` | POST /SpendingValidation/validate | Verificar si el usuario puede costear el gasto |
| `create_expense` | POST /Transaction | Crear transacción de gasto |
| `create_income` | POST /Transaction | Crear transacción de ingreso |
| `get_balance` | GET /User/{id}/balance | Obtener saldo del usuario |
| `list_transactions` | GET /Transaction/user/{id} | Listar transacciones del usuario |
| `create_rule` | POST /FinancialRule | Crear regla financiera |
| `list_rules` | GET /FinancialRule/user/{id} | Listar reglas financieras |
| `get_summary` | GET /Transaction/user/{id}/category-summary | Obtener resumen de gastos |

**Payload JSON Esperado**:

```json
{
  "intent": "create_expense",
  "userId": "e491896e-bc5c-4a2c-a20e-493aa5972281",
  "amount": 50000,
  "category": "Comida",
  "description": "Almuerzo",
  "type": "Expense"
}
```

### Lógica de Análisis de Correos

El nodo de código JavaScript implementa la siguiente lógica de análisis:

```javascript
// 1. DETECTAR BANCO
let banco = 'Otro';
if (from.toLowerCase().includes('bancolombia')) {
  banco = 'Bancolombia';
} else if (from.toLowerCase().includes('nequi')) {
  banco = 'Nequi';
}

// 2. DETECTAR TIPO (Ingreso o Gasto)
let type = 'Expense'; // Por defecto es gasto
if (body.includes('recibiste') || body.includes('abono')) {
  type = 'Income';
}

// 3. EXTRAER MONTO (formato colombiano $150.000)
const montoMatch = snippet.match(/\$\s?([\d.]+(?:,\d{2})?)/);
let amount = 0;
if (montoMatch) {
  amount = parseFloat(montoMatch[1].replace(/\./g, '').replace(',', '.'));
}

// 4. EXTRAER EMAIL DEL USUARIO
let userEmail = to;
const emailMatch = to.match(/<([^>]+)>/);
if (emailMatch) {
  userEmail = emailMatch[1];
}
```

### Integración de API

#### Endpoints del Backend Utilizados

| Método | Endpoint | Propósito |
|--------|----------|-----------|
| GET | `/api/User/email/{email}` | Obtener usuario por dirección de correo |
| POST | `/api/Transaction` | Crear una nueva transacción |
| GET | `/api/User/{id}/balance` | Obtener saldo del usuario |
| POST | `/api/SpendingValidation/validate` | Validar gasto contra reglas |
| POST | `/api/FinancialRule` | Crear regla financiera |

#### Tipos de Datos

| Campo | Tipo | Valores |
|-------|------|---------|
| Type | String | "Income", "Expense" |
| Source | String | "Manual", "Telegram", "WhatsApp", "Automatic" |
| RuleType | String | "SpendingLimit", "SavingsGoal", "CategoryBudget" |
| RulePeriod | String | "Daily", "Weekly", "Biweekly", "Monthly", "Yearly" |

### Configuración

#### Variables de Entorno

| Variable | Descripción | Por Defecto |
|----------|-------------|-------------|
| `BACKEND_URL` | URL base del Backend API | `https://api.avaricia.crudzaso.com` |
| `N8N_HOST` | URL de instancia N8N | `https://n8n.avaricia.crudzaso.com` |

#### Configuración del Workflow

| Configuración | Valor Recomendado |
|---------------|-------------------|
| Execution Order | v1 (recomendado) |
| Guardar ejecuciones fallidas | Guardar |
| Guardar ejecuciones exitosas | No guardar |
| Timeout del Workflow | 2 minutos |

### Despliegue

#### URLs de Producción

| Servicio | URL |
|----------|-----|
| Instancia n8n | https://n8n.avaricia.crudzaso.com |
| Backend API | https://api.avaricia.crudzaso.com |
| Swagger UI | https://api.avaricia.crudzaso.com/swagger |

#### Importar Workflow

```bash
# Clonar repositorio
git clone https://github.com/Team-Avaricia/riwiwallet-n8n-service.git
cd riwiwallet-n8n-service

# Exportar API key de N8N
export N8N_API_KEY="tu-api-key"

# Importar workflows
./scripts/import-workflows.sh
```

### Solución de Problemas

#### Problemas Comunes

| Problema | Causa | Solución |
|----------|-------|----------|
| Ejecución atascada en "Queued" | Worker de n8n sobrecargado | Reiniciar contenedor n8n |
| 400 Bad Request | ID de Usuario inválido | Verificar si el usuario existe en la base de datos |
| Gmail Trigger no funciona | Token OAuth expirado | Re-autenticar credenciales de Gmail |
| Monto = 0 | Formato de correo no reconocido | Actualizar patrón regex |

#### Verificar Ejecución del Workflow

1. Revisar pestaña de Ejecuciones en n8n
2. Buscar estado de Éxito/Error
3. Inspeccionar salidas de nodos para flujo de datos
4. Verificar base de datos para transacciones creadas

---

**License**: This project is part of the Riwi educational program.
