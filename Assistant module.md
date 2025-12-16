# 📚 Comprehensive Technical Documentation - Assistant Module

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Architecture](#system-architecture)
3. [Technology Stack](#technology-stack)
4. [Project Structure](#project-structure)
5. [Core Components](#core-components)
6. [Message Processing Flow](#message-processing-flow)
7. [OpenAI Integration](#openai-integration)
8. [Intent Classification System](#intent-classification-system)
9. [Communication API with .NET Backend](#communication-api-with-net-backend)
10. [Transaction Confirmation System](#transaction-confirmation-system)
11. [Audio Handling (Speech-to-Text)](#audio-handling-speech-to-text)
12. [Account Linking System](#account-linking-system)
13. [Security](#security)
14. [Error Handling](#error-handling)
15. [Exposed Endpoints](#exposed-endpoints)
16. [Configuration](#configuration)
17. [Metrics and Observability](#metrics-and-observability)
18. [Technical Presentation Guide](#technical-presentation-guide)

---

## Executive Summary

The **Assistant Module** is an intelligent financial chatbot that allows users to manage their personal finances through Telegram using **natural language** and **voice notes**. It leverages GPT-5.2 for language understanding and Whisper for audio transcription.

### Key Features

| Feature | Description |
|---------|-------------|
| 💬 **Advanced NLU** | Natural language processing with GPT-5.2 |
| 🎙️ **Voice Notes** | Automatic transcription with Whisper |
| 💸 **Expense Management** | Create, query, and delete transactions |
| 📊 **Financial Rules** | Budgets by category and period |
| 🔔 **Notifications** | Push alerts via Telegram |
| ✅ **Smart Confirmation** | Verification for high-value transactions |
| 🔗 **Account Linking** | Secure Telegram ↔ User link system |

---

## System Architecture

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         TELEGRAM BOT API                             │
│                     (Webhook: /api/telegram/webhook)                 │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      SPRING BOOT (sb-service)                        │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    CONTROLLER LAYER                            │  │
│  │  ┌─────────────────────┐  ┌─────────────────────────────────┐ │  │
│  │  │ TelegramController  │  │  NotificationController          │ │  │
│  │  │ (Webhook Handler)   │  │  (Push Notifications API)        │ │  │
│  │  └─────────────────────┘  └─────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                    │                                 │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    ORCHESTRATION LAYER                         │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │              MessageProcessorService                     │  │  │
│  │  │   (Main Orchestrator - Coordinates all operations)       │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                    │                                 │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                       AI LAYER                                 │  │
│  │  ┌─────────────────────────┐  ┌─────────────────────────────┐ │  │
│  │  │ IntentClassifierService │  │ AudioTranscriptionService   │ │  │
│  │  │ (GPT-4 NLU)             │  │ (Whisper STT)               │ │  │
│  │  └─────────────────────────┘  └─────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                    │                                 │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    HANDLER LAYER                               │  │
│  │  ┌─────────────────────┐ ┌─────────────────┐ ┌───────────────┐│  │
│  │  │TransactionHandler   │ │ RuleHandler     │ │ QueryHandler  ││  │
│  │  │(CRUD Transactions)  │ │ (CRUD Rules)    │ │ (Balance,Sum) ││  │
│  │  └─────────────────────┘ └─────────────────┘ └───────────────┘│  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                    │                                 │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    INTEGRATION LAYER                           │  │
│  │  ┌─────────────────────────┐  ┌─────────────────────────────┐ │  │
│  │  │     CoreApiService      │  │    UserMappingService       │ │  │
│  │  │  (HTTP Client → .NET)   │  │  (Telegram ↔ User Mapping)  │ │  │
│  │  └─────────────────────────┘  └─────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    .NET BACKEND (Core Service)                       │
│              (REST API - Financial Data Management)                  │
│   ┌──────────┐  ┌──────────────┐  ┌────────────────┐  ┌──────────┐ │
│   │  Users   │  │ Transactions │  │ FinancialRules │  │ Telegram │ │
│   └──────────┘  └──────────────┘  └────────────────┘  └──────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        POSTGRESQL DATABASE                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Design Pattern: Layered Architecture

The module implements a **Layered Architecture** pattern, which organizes the codebase into distinct horizontal layers, each with specific responsibilities. This pattern promotes separation of concerns, maintainability, and testability.

#### Layer Structure

| Layer | Responsibility | Components |
|-------|----------------|------------|
| **Presentation Layer** | Handles incoming HTTP requests and webhooks | `TelegramController`, `NotificationController` |
| **Business Logic Layer** | Orchestrates operations and applies business rules | `MessageProcessorService`, `ConfirmationService` |
| **AI/Processing Layer** | Manages AI integrations for NLU and STT | `IntentClassifierService`, `AudioTranscriptionService` |
| **Domain Services Layer** | Implements domain-specific business operations | `TransactionHandlerService`, `RuleHandlerService`, `QueryHandlerService` |
| **Integration Layer** | Communicates with external systems | `CoreApiService`, `UserMappingService`, `TelegramService` |
| **Data Access Layer** | Manages local data persistence | `ConversationMessageRepository`, `PendingConfirmationRepository` |

#### Key Principles

1. **Separation of Concerns**: Each layer has a well-defined responsibility and only interacts with adjacent layers
2. **Dependency Direction**: Higher layers depend on lower layers, never the reverse
3. **Loose Coupling**: Layers communicate through interfaces, allowing for easy testing and replacement
4. **Single Responsibility**: Each service within a layer focuses on a specific domain task

#### Benefits of Layered Architecture

- ✅ **Maintainability**: Changes to one layer don't affect others
- ✅ **Testability**: Layers can be unit tested in isolation with mocks
- ✅ **Scalability**: Individual layers can be scaled or replaced independently
- ✅ **Team Collaboration**: Different teams can work on different layers simultaneously
- ✅ **Code Reusability**: Services can be reused across multiple flows

---

## Technology Stack

| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| **Runtime** | Java | 21 | Primary language |
| **Framework** | Spring Boot | 3.4.0 | Application framework |
| **AI Integration** | Spring AI | 1.0.0-M4 | OpenAI integration |
| **NLU Model** | GPT-5.2 | Latest | Intent classification |
| **STT Model** | Whisper | whisper-1 | Audio transcription |
| **Database** | PostgreSQL | 15+ | Persistence (via .NET) |
| **Messaging** | Telegram Bot API | Latest | User interface |
| **Build Tool** | Maven | 3.9+ | Dependency management |

### Key Dependencies (pom.xml)

```xml
<!-- Spring AI for OpenAI Integration -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>

<!-- Spring Web for REST APIs -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Spring Data JPA for local persistence -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

---

## Project Structure

```
src/main/java/com/avaricia/sb_service/assistant/
├── controller/
│   ├── TelegramController.java       # Telegram webhook handler
│   └── NotificationController.java   # Push notifications REST API
│
├── dto/
│   ├── IntentResult.java             # AI classification result
│   ├── PendingAction.java            # Pending confirmation actions
│   ├── PendingBatchAction.java       # Batch pending actions
│   ├── NotificationRequest.java      # Notification request DTO
│   ├── NotificationResponse.java     # Notification response DTO
│   └── ConfirmationIntent.java       # Confirmation intent DTO
│
├── entity/
│   ├── ConversationMessageEntity.java    # Conversation history
│   └── PendingConfirmationEntity.java    # Pending confirmations
│
├── exception/
│   ├── AssistantException.java           # Base exception
│   ├── CoreApiException.java             # External API errors
│   ├── IntentClassificationException.java # AI errors
│   ├── TranscriptionException.java       # Audio errors
│   ├── UserNotLinkedException.java       # Unlinked user error
│   ├── ConfirmationException.java        # Confirmation errors
│   ├── PersistenceException.java         # Database errors
│   └── GlobalExceptionHandler.java       # Global handler
│
├── repository/
│   ├── ConversationMessageRepository.java    # History repository
│   └── PendingConfirmationRepository.java    # Confirmations repository
│
└── service/
    ├── MessageProcessorService.java      # 🎯 MAIN ORCHESTRATOR
    ├── IntentClassifierService.java      # 🤖 GPT-4 Classification
    ├── AudioTranscriptionService.java    # 🎙️ Whisper STT
    ├── TransactionHandlerService.java    # 💸 CRUD Transactions
    ├── RuleHandlerService.java           # 📏 CRUD Rules
    ├── QueryHandlerService.java          # 📊 Balance and Summaries
    ├── CoreApiService.java               # 🔗 HTTP Client → .NET
    ├── UserMappingService.java           # 👤 Account linking
    ├── TelegramService.java              # 📱 Message sender
    ├── ConfirmationService.java          # ✅ Confirmation system
    ├── ConversationHistoryService.java   # 📜 Conversation history
    ├── ResponseFormatterService.java     # 🎨 Response formatting
    ├── OpenAIService.java                # 🧠 OpenAI utilities
    └── MockCoreApiService.java           # 🧪 Testing mock
```

---

## Core Components

### 1. MessageProcessorService (Orchestrator)

**Responsibility**: Coordinates the entire message processing flow.

```java
@Service
public class MessageProcessorService {
    
    // Process a message and return the response
    public String processMessage(Long telegramId, String message) {
        // 1. Check for pending confirmations
        String confirmationResponse = handlePendingConfirmation(telegramId, message);
        if (confirmationResponse != null) return confirmationResponse;
        
        // 2. Get system userId
        String userId = userMapping.getUserIdByTelegramId(telegramId);
        
        // 3. Save message to history
        conversationHistory.saveMessage(telegramId, "user", message);
        
        // 4. Classify intent(s) with AI
        List<IntentResult> intents = intentClassifier.classifyIntent(message, telegramId);
        
        // 5. Execute the intents
        String response = executeMultipleIntents(userId, intents, telegramId);
        
        // 6. Humanize response if applicable
        return humanizeIfNeeded(response, message, intents.get(0).getIntent());
    }
}
```

**Key Functions**:
- `processMessage()`: Main entry point
- `handlePendingConfirmation()`: Handles high-value confirmations
- `executeMultipleIntents()`: Executes multiple operations
- `humanizeIfNeeded()`: Makes responses more natural

### 2. IntentClassifierService (AI)

**Responsibility**: Classifies user intent using GPT-5.2.

```java
@Service
public class IntentClassifierService {
    
    private final ChatClient chatClient;
    
    // System Prompt with all rules
    private static final String SYSTEM_PROMPT = """
        You are an intelligent financial assistant...
        Possible intents are:
        1. "validate_expense" - User ASKS if they can spend
        2. "create_expense" - User CONFIRMS they spent
        3. "create_income" - Register income
        4. "list_transactions" - View transactions
        ...
    """;
    
    public List<IntentResult> classifyIntent(String userMessage, Long telegramId) {
        // Build prompt with current date
        String dynamicPrompt = buildDynamicSystemPrompt();
        
        // Call GPT-5.2
        String response = chatClient.prompt()
            .system(dynamicPrompt)
            .user(messageWithContext)
            .call()
            .content();
        
        // Parse JSON response
        return objectMapper.readValue(response, ...);
    }
}
```

**Supported Intents**:

| Intent | Description | Example |
|--------|-------------|---------|
| `validate_expense` | Query if user can spend | "Can I spend $100?" |
| `create_expense` | Register an expense | "I spent $50 on food" |
| `create_income` | Register income | "Received $2000 salary" |
| `list_transactions` | List transactions | "My expenses this month" |
| `list_transactions_by_date` | By specific date | "Yesterday's expenses" |
| `list_transactions_by_range` | By date range | "This week's expenses" |
| `search_transactions` | Search by description | "How much do I pay for Netflix?" |
| `get_balance` | Query balance | "How much money do I have?" |
| `get_summary` | Expense summary | "What do I spend most on?" |
| `delete_transaction` | Delete transaction | "Delete the last expense" |
| `create_rule` | Create rule/budget | "Limit of $500 on food" |
| `list_rules` | List rules | "My budgets" |
| `question` | General questions | "How can I save money?" |

### 3. TransactionHandlerService

**Responsibility**: Manages transaction CRUD operations.

```java
@Service
public class TransactionHandlerService {
    
    private final CoreApiService coreApi;
    private final ConfirmationService confirmationService;
    
    // Threshold for confirmation ($500 USD equivalent)
    private double confirmationThreshold = 500000;
    
    public String handleCreateTransaction(String userId, IntentResult intent, 
                                          String type, Long telegramId) {
        Double amount = intent.getAmount();
        
        // Check if confirmation is required
        if (telegramId != null && amount >= confirmationThreshold) {
            confirmationService.createPendingAction(telegramId, userId, intent, type);
            return String.format(
                "⚠️ You're about to register a %s of $%,.0f\nConfirm? (Yes/No)",
                type.equals("Expense") ? "expense" : "income",
                amount
            );
        }
        
        // Execute transaction
        return executeTransaction(userId, intent, type);
    }
}
```

### 4. AudioTranscriptionService

**Responsibility**: Transcribes voice notes using Whisper.

```java
@Service
public class AudioTranscriptionService {
    
    private static final String WHISPER_API_URL = 
        "https://api.openai.com/v1/audio/transcriptions";
    
    public String transcribeAudio(String fileId) {
        // 1. Get file path from Telegram
        String filePath = getFilePath(fileId);
        
        // 2. Download audio file
        byte[] audioData = downloadFile(filePath, fileId);
        
        // 3. Send to Whisper for transcription
        return transcribeWithWhisper(audioData, filePath);
    }
    
    private String transcribeWithWhisper(byte[] audioData, String filePath) {
        MultiValueMap<String, Object> body = new LinkedMultiValueMap<>();
        body.add("file", fileResource);
        body.add("model", "whisper-1");
        body.add("language", "es");  // Spanish
        body.add("response_format", "text");
        
        // POST to OpenAI Whisper API
        return restTemplate.exchange(WHISPER_API_URL, ...).getBody();
    }
}
```

### 5. CoreApiService

**Responsibility**: HTTP client for communication with .NET backend.

```java
@Service
public class CoreApiService {
    
    @Value("${ms.core.url}")
    private String baseUrl;
    
    @Value("${ms.core.api-key}")
    private String apiKey;
    
    // Create headers with API Key
    private HttpHeaders createHeaders() {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("X-API-Key", apiKey);
        return headers;
    }
    
    // Create transaction
    public Map<String, Object> createTransaction(String userId, Double amount, 
                                                  String type, String category, 
                                                  String description, String source) {
        String url = baseUrl + "/api/Transaction";
        Map<String, Object> body = new HashMap<>();
        body.put("userId", userId);
        body.put("amount", amount);
        body.put("type", type);  // "Expense" | "Income"
        body.put("category", category);
        body.put("description", description);
        body.put("source", source);  // "Telegram"
        
        return postRequest(url, body);
    }
}
```

**Consumed Endpoints from .NET Backend**:

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/User/telegram/{telegramId}` | Get user by Telegram ID |
| POST | `/api/User/link-telegram` | Link Telegram account |
| POST | `/api/Transaction` | Create transaction |
| GET | `/api/Transaction/user/{userId}` | List transactions |
| DELETE | `/api/Transaction/{id}` | Delete transaction |
| GET | `/api/Transaction/user/{userId}/balance` | Get balance |
| POST | `/api/FinancialRule` | Create financial rule |
| GET | `/api/FinancialRule/user/{userId}` | List rules |
| DELETE | `/api/FinancialRule/{id}` | Delete rule |

---

## Message Processing Flow

### Complete Flow

```
User sends message on Telegram
            │
            ▼
    ┌───────────────────┐
    │ TelegramController │ ◄── Webhook receives update
    └───────────────────┘
            │
            ▼
    ┌───────────────────┐     ┌──────────────────────┐
    │ Is it a voice note?│─Yes─▶│ AudioTranscription   │
    └───────────────────┘     │ Service (Whisper)    │
            │                 └──────────────────────┘
            │No                        │
            ▼                          │
    ┌───────────────────┐              │
    │ MessageProcessor  │◄─────────────┘
    │ Service           │
    └───────────────────┘
            │
            ▼
    ┌───────────────────────────────────┐
    │ Is there a pending confirmation?  │
    └───────────────────────────────────┘
            │                    │
            │No                  │Yes
            ▼                    ▼
    ┌───────────────────┐  ┌───────────────────┐
    │ UserMappingService│  │ConfirmationService│
    │ (Get userId)      │  │ (Handle response) │
    └───────────────────┘  └───────────────────┘
            │                    │
            ▼                    │
    ┌───────────────────┐        │
    │ IntentClassifier  │        │
    │ Service (GPT-5.2) │        │
    └───────────────────┘        │
            │                    │
            ▼                    │
    ┌───────────────────────────────────┐
    │         Intent Routing            │
    │  ┌─────────┬─────────┬─────────┐  │
    │  │Transact.│  Rule   │  Query  │  │
    │  │Handler  │ Handler │ Handler │  │
    │  └─────────┴─────────┴─────────┘  │
    └───────────────────────────────────┘
            │
            ▼
    ┌───────────────────┐
    │  CoreApiService   │ ──▶ .NET Backend
    │  (HTTP Requests)  │
    └───────────────────┘
            │
            ▼
    ┌───────────────────┐
    │ ResponseFormatter │
    │ + Humanizer       │
    └───────────────────┘
            │
            ▼
    ┌───────────────────┐
    │ TelegramService   │ ──▶ Response to user
    │ (Send Message)    │
    └───────────────────┘
```

---

## OpenAI Integration

### Models Used

| Model | Use | Approximate Cost |
|-------|-----|------------------|
| GPT-5.2 | Intent classification | ~$0.05/1K tokens |
| GPT-5.2 | Response humanization | ~$0.05/1K tokens |
| Whisper-1 | Audio transcription | ~$0.006/minute |

### Prompt Engineering

The classifier prompt includes:

1. **Intent definitions** (13 types)
2. **Valid categories** (15 categories)
3. **Classification examples**
4. **Dynamic date handling**
5. **JSON format rules**
6. **Bot limitations**
7. **Multiple operations handling**

---

## Intent Classification System

### IntentResult DTO

```java
public class IntentResult {
    private String intent;          // Intent type
    private Double amount;          // Amount (if applicable)
    private String category;        // Category
    private String description;     // Description
    private String type;            // "Expense" | "Income"
    private String period;          // "Monthly" | "Weekly"
    private String startDate;       // Start date (ISO)
    private String endDate;         // End date (ISO)
    private String searchQuery;     // Search text
    private String response;        // Response for user
}
```

### Supported Categories

**Expenses:**
- Food, Transportation, Entertainment, Health, Education
- Home, Clothing, Technology, Services, Rent, Housing, Others

**Income:**
- Salary, Freelance, Investments, Gifts, Others

### Multiple Operations

The classifier can detect multiple operations in a single message:

```
User: "I spent $10 on taxi and received $50 from my mom"

Classifier response:
[
  {"intent": "create_expense", "amount": 10000, "category": "Transportation"},
  {"intent": "create_income", "amount": 50000, "category": "Gifts"}
]
```

---

## Communication API with .NET Backend

### Authentication

```java
private HttpHeaders createHeaders() {
    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.APPLICATION_JSON);
    headers.set("X-API-Key", apiKey);  // Internal API Key
    return headers;
}
```

### Enum Mapping

| Spring Boot | .NET Backend |
|-------------|--------------|
| `"Expense"` | `TransactionType.Expense` |
| `"Income"` | `TransactionType.Income` |
| `"Telegram"` | `TransactionSource.Telegram` |
| `"CategoryBudget"` | `RuleType.CategoryBudget` |
| `"Monthly"` | `RulePeriod.Monthly` |

### Response Handling

```java
@SuppressWarnings("unchecked")
private Map<String, Object> postRequest(String url, Map<String, Object> body) {
    try {
        ResponseEntity<String> response = restTemplate.postForEntity(url, request, String.class);
        
        if (response.getBody() != null) {
            return objectMapper.readValue(response.getBody(), Map.class);
        }
        
        return Map.of("success", true);
    } catch (Exception e) {
        return Map.of("success", false, "error", e.getMessage());
    }
}
```

---

## Transaction Confirmation System

### Purpose

Prevent errors in high-value transactions (≥ $500 equivalent).

### Confirmation Flow

```
1. User: "I spent $800 on the PS5"
2. Bot: "⚠️ You're about to register an expense of $800. Confirm? (Yes/No)"
3. User: "Yes"
4. Bot: "✅ Expense of $800 registered in Technology"
```

### Implementation

```java
@Service
public class ConfirmationService {
    
    @Transactional
    public void createPendingAction(Long telegramId, String userId, 
                                    IntentResult intent, String type) {
        PendingConfirmationEntity pending = new PendingConfirmationEntity();
        pending.setTelegramId(telegramId);
        pending.setUserId(userId);
        pending.setActionData(objectMapper.writeValueAsString(intent));
        pending.setActionType(type);
        pending.setExpiresAt(LocalDateTime.now().plusMinutes(5));
        
        repository.save(pending);
    }
    
    public Optional<PendingConfirmationEntity> getPendingAction(Long telegramId) {
        return repository.findByTelegramIdAndExpiresAtAfter(
            telegramId, LocalDateTime.now()
        );
    }
}
```

### Batch Confirmations

For multiple high-value transactions:

```
User: "I spent $600 on clothes and $700 on technology"
Bot: "⚠️ You're about to register 2 transactions totaling $1,300:
     1. Expense of $600 in Clothing
     2. Expense of $700 in Technology
     Confirm all? (Yes/No)"
```

---

## Audio Handling (Speech-to-Text)

### Processing Flow

```
1. User sends voice note
2. TelegramController detects voice/audio message
3. Bot responds: "🎧 Processing your voice note..."
4. AudioTranscriptionService:
   a. Gets file_path from Telegram API
   b. Downloads the audio file
   c. Sends to Whisper API
   d. Receives text transcription
5. Bot shows: "📝 I understood: '{transcription}'"
6. MessageProcessor processes the transcribed text normally
```

### Supported Formats

Whisper supports: `.mp3`, `.mp4`, `.mpeg`, `.mpga`, `.m4a`, `.wav`, `.webm`, `.ogg`

### Whisper Configuration

```java
MultiValueMap<String, Object> body = new LinkedMultiValueMap<>();
body.add("file", fileResource);
body.add("model", "whisper-1");
body.add("language", "es");           // Spanish
body.add("response_format", "text");  // Text only
```

---

## Account Linking System

### Linking Flow

```
1. User in Frontend: Requests linking code
2. .NET Backend: Generates unique code (e.g., "LINK_ABC123")
3. User: Opens link t.me/AvariciaBot?start=LINK_ABC123
4. Bot receives /start LINK_ABC123
5. Bot sends code to .NET for validation
6. .NET links telegramId with userId
7. Bot confirms: "✅ Account linked successfully!"
```

### Implementation

```java
@Service
public class UserMappingService {
    
    public String getUserIdByTelegramId(Long telegramId) {
        Map<String, Object> response = coreApi.getUserByTelegramId(telegramId);
        
        if (response.containsKey("error")) {
            throw new UserNotLinkedException(telegramId);
        }
        
        return (String) response.get("id");
    }
    
    public boolean linkTelegram(String code, Long telegramId, 
                                String username, String firstName) {
        Map<String, Object> result = coreApi.linkTelegram(
            code, telegramId, username, firstName
        );
        return !result.containsKey("error");
    }
}
```

---

## Security

### Security Layers

| Layer | Mechanism |
|-------|-----------|
| Telegram → Spring Boot | Secret webhook URL |
| Spring Boot → .NET | API Key in header |
| .NET → PostgreSQL | Secure connection string |
| Frontend → Spring Boot | JWT Bearer token |

### Internal API Key

```java
// CoreApiService.java
@Value("${ms.core.api-key}")
private String apiKey;

private HttpHeaders createHeaders() {
    headers.set("X-API-Key", apiKey);
    return headers;
}
```

```yaml
# application.yml
ms:
  core:
    api-key: ${CORE_API_KEY}  # Environment variable
```

### User Validation

Every operation verifies that the `telegramId` is linked to a valid user:

```java
String userId = userMapping.getUserIdByTelegramId(telegramId);
if (userId == null) {
    throw new UserNotLinkedException(telegramId);
}
```

---

## Error Handling

### Exception Hierarchy

```
AssistantException (Base)
├── CoreApiException          # Communication errors with .NET
├── IntentClassificationException  # AI/GPT errors
├── TranscriptionException    # Audio/Whisper errors
├── UserNotLinkedException    # User not linked
├── ConfirmationException     # Confirmation errors
└── PersistenceException      # Local DB errors
```

### User-Friendly Messages

Each exception has a technical message and a user-facing message:

```java
public class CoreApiException extends AssistantException {
    
    public static CoreApiException transactionFailed(String userId, String error) {
        return new CoreApiException(
            "Failed to create transaction for user " + userId + ": " + error,  // Technical
            "❌ Could not register the transaction. " + error,  // User
            "/api/Transaction",
            "POST"
        );
    }
}
```

### Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserNotLinkedException.class)
    public ResponseEntity<?> handleUserNotLinked(UserNotLinkedException ex) {
        return ResponseEntity.ok(Map.of(
            "success", false,
            "message", "Your Telegram account is not linked. " +
                       "Please link it from the web app."
        ));
    }
}
```

---

## Exposed Endpoints

### TelegramController

```java
@RestController
@RequestMapping("/api/telegram")
public class TelegramController {
    
    // Telegram Webhook
    @PostMapping("/webhook")
    public ResponseEntity<?> handleWebhook(@RequestBody Map<String, Object> update) {
        // Process message or voice note
        // Send response via TelegramService
        return ResponseEntity.ok().build();
    }
}
```

### NotificationController

```java
@RestController
@RequestMapping("/api/notifications")
public class NotificationController {
    
    // Send push notification
    @PostMapping("/send")
    public ResponseEntity<NotificationResponse> sendNotification(
            @RequestBody NotificationRequest request) {
        // Get user's telegramId
        // Send message via Telegram
        return ResponseEntity.ok(new NotificationResponse(true, "Sent"));
    }
    
    // Send notification by userId
    @PostMapping("/send-to-user/{userId}")
    public ResponseEntity<NotificationResponse> sendToUser(
            @PathVariable String userId,
            @RequestBody Map<String, String> body) {
        // Resolve telegramId from userId
        // Send message
    }
}
```

### Available Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/telegram/webhook` | Telegram Webhook |
| POST | `/api/notifications/send` | Send notification |
| POST | `/api/notifications/send-to-user/{userId}` | Notify by userId |

---

## Configuration

### Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `OPENAI_API_KEY` | OpenAI API Key | `sk-...` |
| `TELEGRAM_BOT_TOKEN` | Bot token | `123456:ABC...` |
| `CORE_API_URL` | .NET backend URL | `http://api:8080` |
| `CORE_API_KEY` | Internal API Key | `secret-key` |
| `DATABASE_URL` | PostgreSQL (local) | `jdbc:postgresql://...` |

---

## Metrics and Observability

### Structured Logs

```java
log.info("🤖 Processing message for telegramId: {}", telegramId);
log.debug("🎯 Intent classified: {} with confidence", intent.getIntent());
log.error("❌ API call failed: {}", error.getMessage());
```

### Log Emojis

| Emoji | Meaning |
|-------|---------|
| 🤖 | AI processing |
| 🎯 | Successful classification |
| 📤 | Request sent |
| 📥 | Response received |
| ✅ | Successful operation |
| ❌ | Error |
| 🎤 | Audio/Voice |
| 💰 | Transaction |
| 📏 | Financial rule |

---

## Summary

The Assistant Module represents a modern approach to personal finance management through conversational AI. By leveraging state-of-the-art language models (GPT-5.2) and speech recognition (Whisper), it provides users with an intuitive, natural way to track expenses, manage budgets, and gain insights into their spending habits—all through simple text or voice messages on Telegram.

### Key Architectural Decisions

| Decision | Benefit |
|----------|---------|
| **Spring AI for OpenAI** | Simplified integration with standardized APIs |
| **Layered Architecture** | Clean separation of concerns, easy maintainability |
| **Dedicated Handler Services** | Domain-specific logic isolation |
| **External .NET Backend** | Separation of AI layer from data persistence |
| **Confirmation System** | User safety for high-value operations |

### Future Enhancements

- Multi-language support beyond Spanish
- WhatsApp integration
- Advanced analytics and spending predictions
- Recurring transaction detection

