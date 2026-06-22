# 🏥 MediAgent
### AI-Powered Multi-Agent Healthcare System

![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.x-6DB33F?style=flat&logo=spring-boot)
![Qwen Cloud](https://img.shields.io/badge/Qwen_Cloud-LLM_+_Audio-FF6A00?style=flat)
![Apache Kafka](https://img.shields.io/badge/Apache_Kafka-Messaging-231F20?style=flat&logo=apache-kafka)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat&logo=docker)
![Track](https://img.shields.io/badge/Hackathon-Track_3_Agent_Society-blueviolet?style=flat)

> Voice-first, multilingual AI triage assistant — patients speak in Hindi, Tamil, Telugu, or English and receive personalized medical guidance powered by a society of 5 specialized Spring Boot microservices.

---

## 📋 Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Microservices](#microservices)
- [Tech Stack](#tech-stack)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Environment Variables](#environment-variables)
- [Build Order](#suggested-build-order)
- [Why MediAgent Wins Track 3](#why-mediagent-wins-track-3)

---

## Overview

MediAgent is a **society of 5 AI agents**, each a Spring Boot microservice with a distinct role. A patient speaks in their native language — the system triages symptoms, researches treatment, recalls their medical history, and responds with a synthesized, personalized voice answer.

**Key Differentiators:**
- 🎙️ **Voice-first** — no typing required, accessible to all literacy levels
- 🌐 **Multilingual** — auto-detects and responds in the patient's language
- 🤝 **Agent conflict resolution** — orchestrator escalates when agents disagree
- 📊 **Measurable baseline** — single-agent vs multi-agent diagnostic accuracy
- 🇮🇳 **Real-world impact** — primary care access for 500M+ Indians in local languages

---

## System Architecture

```
Patient Voice Input
        │
        ▼
┌───────────────┐
│  voice-agent  │  ← STT (Whisper/Qwen Audio) + Language Detection
│   Port 8081   │
└───────┬───────┘
        │  text + detected language
        ▼
┌────────────────────┐
│ orchestrator-agent │  ← The Boss — fans out in parallel
│     Port 8085      │
└──────┬──────┬──────┘
       │      │      │
       ▼      ▼      ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ symptom  │ │ research │ │  memory  │
│  agent   │ │  agent   │ │  agent   │
│  :8082   │ │  :8083   │ │  :8084   │
└──────────┘ └──────────┘ └──────────┘
       │      │      │
       └──────┴──────┘
              │  merged + conflict-resolved
              ▼
┌───────────────┐
│  voice-agent  │  ← TTS output in patient's language
└───────────────┘
        │
        ▼
Patient Voice Response
```

**Orchestrator uses `CompletableFuture` to call all 3 agents in parallel, then resolves conflicts before synthesizing the final answer.**

---

## Microservices

| Service | Port | Role | Key Tech |
|---|---|---|---|
| `voice-agent` | 8081 | STT / TTS / Language Detection | Qwen Audio, Whisper, lingua-java |
| `symptom-agent` | 8082 | Triage Brain — severity scoring | Qwen LLM, rule-based engine |
| `research-agent` | 8083 | Real-time medical data lookup | Qwen + web search tool |
| `memory-agent` | 8084 | Patient history & personalization | Redis (session), PostgreSQL |
| `orchestrator-agent` | 8085 | Fan-out, conflict resolution, synthesis | CompletableFuture, Kafka |

---

### 1. `voice-agent` — Language I/O (Port 8081)

- **STT:** Qwen Audio API or OpenAI Whisper → converts Hindi/Tamil/Telugu/English speech to text
- **Language Detection:** `lingua-language-detector` Java lib for auto-detect
- **TTS:** converts final response back to voice in the patient's language

```java
// Example: Qwen API call (OpenAI-compatible)
OpenAiService service = new OpenAiService(
  System.getenv("QWEN_API_KEY"),
  "https://dashscope.aliyuncs.com/compatible-mode/v1"
);
```

---

### 2. `symptom-agent` — Triage Brain (Port 8082)

- Asks adaptive follow-up questions: *"How many days? Fever temperature? Any vomiting?"*
- Scores severity: **Low / Medium / High** using rule-based logic + Qwen prompt
- Outputs a structured `SymptomReport` JSON to the orchestrator

```json
{
  "primarySymptom": "fever",
  "durationDays": 3,
  "severity": "MEDIUM",
  "additionalSymptoms": ["headache", "fatigue"],
  "followUpAnswers": { "temperature": "102F", "vomiting": false }
}
```

---

### 3. `research-agent` — Real-Time Data (Port 8083)

- Calls Qwen Cloud with **web search tool enabled** for up-to-date results
- Fetches: latest doctor advice, WHO guidelines, medicine info
- Returns: root cause, cure steps, medicine name + current Indian market price (₹)

```java
// Enable web search on Qwen Cloud call
ChatCompletionRequest request = ChatCompletionRequest.builder()
    .model("qwen-plus")
    .messages(messages)
    .functions(List.of(webSearchFunction))
    .build();
```

---

### 4. `memory-agent` — Patient History (Port 8084)

- **Redis:** session memory for the current conversation (TTL: 30 min)
- **PostgreSQL:** long-term store — past illnesses, allergies, medications
- Enables personalization: *"You had this same fever last July — here's what helped"*

```java
// Redis session key pattern
String sessionKey = "session:" + patientId + ":" + sessionId;

// PostgreSQL tables (simplified)
// patients(id, name, dob, language_pref)
// medical_history(id, patient_id, date, diagnosis, treatment)
// allergies(id, patient_id, allergen, severity)
```

---

### 5. `orchestrator-agent` — The Boss (Port 8085)

- Receives patient input → fans out to all 4 agents **in parallel** via `CompletableFuture`
- **Conflict resolution:** if symptom agent says `MILD` but research agent says `SERIOUS` → escalates
- Synthesizes all outputs → sends final response back through the voice agent

```java
// Parallel fan-out
CompletableFuture<SymptomReport> symptomFuture =
    CompletableFuture.supplyAsync(() -> symptomClient.analyze(input));

CompletableFuture<ResearchResult> researchFuture =
    CompletableFuture.supplyAsync(() -> researchClient.lookup(input));

CompletableFuture<PatientHistory> memoryFuture =
    CompletableFuture.supplyAsync(() -> memoryClient.getHistory(patientId));

CompletableFuture.allOf(symptomFuture, researchFuture, memoryFuture).join();

// Conflict resolution
if (symptomResult.getSeverity() == LOW && researchResult.getRiskLevel() == HIGH) {
    finalSeverity = MEDIUM; // escalate
}
```

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Agents | Spring Boot 3.x | Each agent as an independent microservice |
| AI / LLM | Qwen Cloud API | OpenAI-compatible LLM + audio endpoint |
| Messaging | Apache Kafka | Async inter-agent communication |
| Session Memory | Redis | Fast in-memory session state per patient |
| Long-term Store | PostgreSQL | Patient history, allergies, medications |
| API Gateway | Spring Cloud Gateway | Single entry point, routing, auth |
| Voice STT | OpenAI Whisper | Multilingual speech-to-text |
| Voice TTS | Qwen Audio TTS | Natural voice output in patient's language |
| Language Detect | lingua-language-detector | Java lib for auto language identification |
| Deployment | Alibaba Cloud ECS + Docker Compose | Containerized multi-service deploy |
| Frontend | React (optional) | Visual interface alongside voice |

---

## Getting Started

### Prerequisites

- Java 17+
- Maven 3.8+
- Docker & Docker Compose
- Qwen Cloud API key (`DASHSCOPE_API_KEY`)

### Step 1 — Bootstrap the Parent Project

```bash
mvn archetype:generate \
  -DgroupId=com.mediagent \
  -DartifactId=mediagent-parent \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false
```

### Step 2 — Create Each Agent Module

```bash
cd mediagent-parent
mkdir voice-agent symptom-agent research-agent memory-agent orchestrator-agent
```

### Step 3 — Add Qwen Dependency to Each `pom.xml`

```xml
<dependency>
  <groupId>com.theokanning.openai-gpt3-java</groupId>
  <artifactId>service</artifactId>
  <version>0.18.2</version>
</dependency>
```

### Step 4 — Start All Services

```bash
# Start Redis, PostgreSQL, Kafka, and all agents
docker-compose up -d

# Verify all services are running
curl http://localhost:8081/actuator/health  # voice-agent
curl http://localhost:8082/actuator/health  # symptom-agent
curl http://localhost:8083/actuator/health  # research-agent
curl http://localhost:8084/actuator/health  # memory-agent
curl http://localhost:8085/actuator/health  # orchestrator-agent
```

### Step 5 — Send a Test Request

```bash
curl -X POST http://localhost:8085/api/triage \
  -H "Content-Type: application/json" \
  -d '{
    "patientId": "P001",
    "audioBase64": "<base64-encoded-audio>",
    "language": "hi"
  }'
```

---

## Project Structure

```
mediagent-parent/
├── pom.xml                          # Parent POM (manages all modules)
├── docker-compose.yml               # All services + Redis + Postgres + Kafka
├── .env                             # Environment variables
│
├── voice-agent/                     # Port 8081
│   ├── pom.xml
│   └── src/main/java/com/mediagent/voice/
│       ├── VoiceAgentApplication.java
│       ├── controller/VoiceController.java
│       └── service/SpeechService.java
│
├── symptom-agent/                   # Port 8082
│   ├── pom.xml
│   └── src/main/java/com/mediagent/symptom/
│       ├── SymptomAgentApplication.java
│       ├── controller/SymptomController.java
│       └── service/TriageService.java
│
├── research-agent/                  # Port 8083
│   ├── pom.xml
│   └── src/main/java/com/mediagent/research/
│       ├── ResearchAgentApplication.java
│       └── service/ResearchService.java
│
├── memory-agent/                    # Port 8084
│   ├── pom.xml
│   └── src/main/java/com/mediagent/memory/
│       ├── MemoryAgentApplication.java
│       ├── repository/PatientRepository.java
│       └── service/MemoryService.java
│
└── orchestrator-agent/              # Port 8085
    ├── pom.xml
    └── src/main/java/com/mediagent/orchestrator/
        ├── OrchestratorAgentApplication.java
        ├── controller/TriageController.java
        └── service/OrchestratorService.java
```

---

## Environment Variables

Create a `.env` file in the project root:

```env
# Qwen Cloud
QWEN_API_KEY=sk-xxxxxxxxxxxxxxxxxxxx
DASHSCOPE_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# PostgreSQL
POSTGRES_URL=jdbc:postgresql://localhost:5432/mediagent
POSTGRES_USER=mediagent
POSTGRES_PASSWORD=your_password_here

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Service Ports
VOICE_AGENT_PORT=8081
SYMPTOM_AGENT_PORT=8082
RESEARCH_AGENT_PORT=8083
MEMORY_AGENT_PORT=8084
ORCHESTRATOR_PORT=8085
```

---

## Suggested Build Order

| Day | Task |
|---|---|
| Day 1 | Bootstrap parent POM, create all 5 modules, verify Spring Boot starts on correct ports |
| Day 2 | Build `orchestrator-agent` — hardcode mock data and wire up `CompletableFuture` fan-out |
| Day 3 | Build `symptom-agent` — Qwen LLM call with follow-up question loop |
| Day 4 | Build `memory-agent` — Redis session + PostgreSQL schema + CRUD |
| Day 5 | Build `research-agent` — Qwen with web search tool enabled |
| Day 6 | Build `voice-agent` — Whisper STT + lingua detection + Qwen TTS |
| Day 7 | End-to-end integration test, Docker Compose, prepare demo |

---

## Why MediAgent Wins Track 3

| Judging Criterion | How MediAgent Delivers |
|---|---|
| **Agent Society (core)** | 5 distinct agents with well-defined roles, parallel execution via `CompletableFuture`, message passing via Kafka |
| **Problem Value (25%)** | Primary care access for 500M+ Indians in local languages — tangible, measurable real-world impact |
| **Conflict Resolution** | Orchestrator escalates severity when symptom and research agents disagree — explicit, demonstrable logic |
| **Measurable Baseline** | Single-agent vs multi-agent accuracy on symptom diagnosis — clear benchmark to show improvement |
| **Technical Depth** | Kafka messaging, `CompletableFuture` fan-out, Redis + PostgreSQL dual-store, Spring Cloud Gateway |
| **Differentiation** | Voice-first multilingual interface — unique among typical text chatbot submissions |

---

## Contributing

1. Fork the repo
2. Create a feature branch: `git checkout -b feature/your-agent-improvement`
3. Commit your changes: `git commit -m 'feat: improve symptom severity scoring'`
4. Push and open a PR

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

*MediAgent — Built for Hackathon Track 3: Agent Society*  
*Healthcare AI · Multilingual · Voice-First · Spring Boot Microservices*
