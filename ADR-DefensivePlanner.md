# ADR-001: Defensive Planner Multi-Agent System Architecture

## 1. Context and Problem Statement
The "Defensive Planner" is an intelligent personal assistant that proactively forecasts potential failures in an itinerary and prepares mitigation strategies. The system operates in a "Simulation Mode" to stress-test plans before execution. It uses a multi-agent architecture where specialized agents (Planner, Risk Analyst, Simulator, Recovery) collaborate to ensure plan robustness.

## 2. Technical Decisions

### 2.1. Technology Stack
*   **Language:** Python 3.10+ (Targeting Jupyter Notebook environment).
*   **AI Engine:** Google Agent Development Kit (`google-adk`) using Gemini 2.5 Flash Lite model.
*   **Core Components:**
    *   `LlmAgent` - Intelligent agents with tool calling capabilities
    *   `SequentialAgent` - For linear workflows (Plan → Risk)
    *   `ParallelAgent` - For concurrent expert consultation (MoE)
    *   `LoopAgent` - For the defensive retry loop
    *   `Gemini` model class - gemini-2.5-flash-lite configuration
    *   `Runner` - Executes agent conversations
    *   `InMemorySessionService` - Manages agent sessions and state
    *   `types.HttpRetryOptions` - Retry configuration for resilience
*   **Data Storage:** In-memory session state for agent-to-agent communication.
*   **Communication:** Shared Session State via `ToolContext` and `output_key`.

### 2.2. Architecture Pattern: The "Defensive Loop"
The system implements a **Loop-based Orchestrator** using Google ADK's `LoopAgent`. The system moves through distinct phases, with **real LLM agents making all decisions**.

**Phases:**
1.  **Planning Workflow (SequentialAgent):**
    *   **Weather Fetcher:** Establishes canonical weather data in session.
    *   **Expert Consultation (ParallelAgent):** Weather, Transport, and Merchandising experts run concurrently.
    *   **Planner:** Synthesizes expert outputs into a unified plan.
    *   **Risk Assessment:** Risk Analyst critiques the plan.
2.  **Recovery (RecoveryAgent):**
    *   Checks for failures or risks.
    *   Fetches news context if failures occur.
    *   Coordinates outfits (A2A) if plan succeeds.
    *   Decides whether to loop back or exit.

**Key Innovation:** Real LLM agents reason and decide; only tool backends are emulated with chaos engineering.

### 2.3. Agent Communication Strategy
To simulate a robust distributed system within a single notebook, agents communicate via **Shared Session State**.
*   `weather_fetcher` writes to `session.state["weather_data"]`.
*   All other agents read from `session.state["weather_data"]` to ensure consistency.
*   Experts write to their respective `output_key`s.
*   Planner reads all expert outputs to generate the `initial_plan`.

### 2.4. Observability Layer

The system includes a lightweight observability solution using Python's standard `logging` module and custom metrics tracking:

**Components:**
*   **`MetricsCollector` class:** Tracks tool call counts, success/failure rates, execution times, and session metrics.
*   **`@trace_tool` decorator:** Instruments all tool functions with automatic logging of inputs, outputs, timing, and success status.
*   **Structured logging:** Writes to both `agent_execution.log` file and stdout with timestamps and log levels.
*   **Session tracking:** Records session start/completion events with user context.

**Design Principles:**
*   Core components (tools, agents, orchestrator) contain no print statements—only structured logging.
*   Display logic resides exclusively in the evaluation section via the `run_scenario()` wrapper.
*   Metrics are aggregated in-memory and reported at session completion.

**Production Note:** This lightweight approach is suitable for demos and local development. Production deployments should use enterprise observability platforms such as Google Cloud Operations Suite, OpenTelemetry, or APM tools like Datadog.

## 3. Detailed Component Design

### 3.1. Mixture of Experts (MoE) Pattern

The system implements **parallel expert consultation** using the `ParallelAgent` pattern:

**Coordinator:**
- **ParallelAgent (expert_team):** Runs sub-agents concurrently.
- **Planner Agent:** Aggregates outputs from the parallel execution.

**Specialized Experts:**

1. **🚗 Transportation Expert**
   - **Domain:** Routing, traffic, multimodal transport
   - **Tools:** `get_real_time_traffic`, `book_taxi`, `get_transit_info`, `analyze_weather_impact`
   - **Expertise:** Route optimization, surge pricing analysis, weather impact on transport

2. **☀️ Weather Expert**
   - **Domain:** Meteorology, forecasting, activity planning
   - **Tools:** None (Reads directly from session state)
   - **Expertise:** Detailed forecasts, severe weather alerts, safety recommendations

3. **🛒 Merchandising Expert**
   - **Domain:** Shopping, deals, inventory management
   - **Tools:** `search_products`, `find_best_deal`, `get_purchase_recommendations`, `purchase_tickets`, `get_outfit_recommendation`
   - **Expertise:** Product search, price comparison, promotion hunting, personalized recommendations

### 3.2. Data Schemas (Session State)

#### `SystemState`
The central source of truth passed between agents via `output_key` and `session.state`.
```json
{
  "weather_data": { ... },      // Canonical weather source
  "weather_analysis": "...",    // Output from Weather Expert
  "transportation_analysis": "...", // Output from Transport Expert
  "merchandising_analysis": "...",  // Output from Merchandising Expert
  "initial_plan": "...",        // Synthesized plan
  "risk_assessment": "...",     // Risk analysis
  "event:{id}:outfits": { ... } // A2A Coordination state
}
```

### 3.3. Agent Implementation (Google ADK LlmAgents)

#### A. The Planner Agent (Aggregator)
*   **Implementation:** `LlmAgent` with gemini-2.5-flash-lite model
*   **Role:** Synthesizes expert analyses into actionable plans
*   **Tools:** None (Aggregates parallel outputs)
*   **Input:** Expert Analyses + Weather Data
*   **Output:** Comprehensive plan incorporating expert advice

#### B. Expert Agents (Transportation, Weather, Merchandising)
*   **Implementation:** 3 specialized `LlmAgent` instances
*   **Role:** Provide domain-specific analysis and recommendations
*   **Tools:** Specialized tool sets per expert
*   **Execution:** Parallel via `ParallelAgent`
*   **Output:** Domain-specific recommendations to session state

#### C. The Risk Analyst Agent
*   **Implementation:** `LlmAgent` with gemini-2.5-flash-lite model
*   **Role:** Critiques the plan using real LLM analysis.
*   **Tools:** None (analysis only)
*   **Input:** Synthesized plan from Planner
*   **Output:** Risk assessment with levels, probabilities, mitigations

#### D. The Recovery Agent
*   **Implementation:** `LlmAgent` with gemini-2.5-flash-lite model
*   **Role:** Fixes failed plans OR coordinates successful ones.
*   **Tools:** `get_event_news`, `get_coordinated_outfit`, `exit_defensive_loop`
*   **Input:** Plan + Risk Assessment
*   **Output:** Revised plan or exit signal
*   **Instruction:** "You are a Recovery Agent. Fix failed plans with news context, or coordinate outfits for successful plans."

### 3.4. Mock Tool Functions (Python Functions for ADK)

All tools are Python functions returning JSON strings, compatible with Google ADK's tool calling. Each tool is instrumented with the `@trace_tool` decorator for observability.

1.  **`get_weather_forecast`**: Canonical source, stores to session (10% failure rate).
2.  **`book_taxi`**: Unified taxi tool with chaos engineering (20% failure).
3.  **`get_transit_info`**: Transit information with service disruptions (8% failure).
4.  **`get_real_time_traffic`**: Traffic conditions and route alternatives.
5.  **`search_products`**: Product search with inventory checks (15% failure).
6.  **`find_best_deal`**: Price comparison with promotions.
7.  **`get_purchase_recommendations`**: Personalized suggestions.
8.  **`purchase_tickets`**: Ticket purchasing with chaos engineering (30% failure).
9.  **`analyze_weather_impact`**: Weather impact on transportation modes.
10. **`get_outfit_recommendation`**: Outfit suggestions based on weather.
11. **`get_event_news`**: Simulates news headlines to explain failures.
12. **`get_coordinated_outfit`**: A2A tool for preventing fashion conflicts.

## 4. Implemented Scenarios

The notebook demonstrates 5 comprehensive scenarios **ordered by increasing complexity**:

### Scenario 1: The Art Exhibition (Happy Path) 
**Complexity:** ⭐ Basic  
*   **Goal:** Attend "Modernist Gala exhibition at 7 PM downtown"
*   **Context:** Clear skies, normal traffic (low-risk environment)
*   **Flow:** Planner creates itinerary -> Risk Analyst approves -> Execution
*   **Demonstrates:** Basic defensive loop, tool usage, LLM decision-making

### Scenario 2: The Flight Connection (Weather Disruption)
**Complexity:** ⭐⭐ Moderate  
*   **Goal:** Catch international flight departing at 5 PM
*   **Context:** Storm warning, heavy traffic
*   **Flow:** Weather Fetcher detects storm -> Transport Expert advises transit -> Planner adjusts
*   **Demonstrates:** Context-aware planning, single disruption handling

### Scenario 3: The Date Night (Inventory/Reservation Failure)
**Complexity:** ⭐⭐ Moderate  
*   **Goal:** Dinner reservation at "The Riverside Restaurant"
*   **Context:** High popularity (demand-driven failure)
*   **Failure Mode:** `purchase_tickets` returns "Sold out"
*   **Recovery:** Recovery Agent suggests alternatives based on news/context
*   **Demonstrates:** Handling sold-out scenarios, alternative selection

### Scenario 4: The Concert (Logistics/Transportation Failure)
**Complexity:** ⭐⭐⭐ High  
*   **Goal:** Attend rock concert at stadium at 9 PM
*   **Context:** Heavy traffic, limited parking
*   **Challenges:** Multiple transport modes may fail simultaneously
*   **Recovery Strategies:** Switch from cab to public transit, adjust timing
*   **Demonstrates:** Multi-point failure handling, cascading backup strategies

### Scenario 5: Agent-to-Agent (A2A) Social Conflict ⭐
**Complexity:** ⭐⭐⭐⭐ Advanced  
*   **Context:** Two separate user sessions running against the same `InMemorySessionService`.
*   **Setup:**
    *   User A (Alice): Plans to wear "Red Dress" to wedding
    *   User B (Bob): Also wants to wear "Red Suit"
*   **Communication:** Shared `InMemorySessionService` state (`event:{id}:outfits`)
*   **Flow:**
    1.  Alice executes loop, `get_coordinated_outfit` registers "Red".
    2.  Bob executes loop, `get_coordinated_outfit` detects conflict.
    3.  Bob's Recovery Agent suggests "Navy Blue" instead.
*   **Demonstrates:** Multi-agent coordination, conflict detection, autonomous resolution via shared session state.

## 5. Implementation Structure

### 5.1. Notebook Organization

**Setup & Configuration:**
*   Environment initialization and ADK installation.
*   API key loading and ADK imports.

**Observability Layer:**
*   `MetricsCollector` class for metrics aggregation.
*   `@trace_tool` decorator for tool instrumentation.
*   Logging configuration (file + stdout).

**Core System:**
*   Mock Tools definition with Chaos Engineering (all decorated with `@trace_tool`).
*   Agent definitions (Weather, Experts, Planner, Risk, Recovery).
*   Orchestrator implementation (`DefensiveOrchestrator`) with logging-only output.

**Evaluation & Demonstration:**
*   `run_scenario()` wrapper function for display logic.
*   Scenarios 1-5 demonstrating increasing complexity.
*   A2A scenario using shared `InMemorySessionService`.
*   Metrics report display at session completion.

### 5.2. Key Implementation Details

**Google ADK Integration:**
```python
# Model configuration
gemini_model = Gemini(model="gemini-2.5-flash-lite", retry_options=retry_config)

# Workflow definition
planning_workflow = SequentialAgent(
    sub_agents=[weather_fetcher, expert_team, planner_agent, risk_analyst_agent]
)
defensive_loop = LoopAgent(
    sub_agents=[planning_workflow, recovery_agent],
    max_iterations=2
)

# Execution
orchestrator = DefensiveOrchestrator(root_agent=defensive_loop)
result = orchestrator.execute_loop(goal="...", user_id="User1")
```

**A2A Communication:**
```python
# Shared Session Service
a2a_session_service = InMemorySessionService()
orchestrator = DefensiveOrchestrator(root_agent=defensive_loop, session_service=a2a_session_service)

# Tool Implementation
@trace_tool
def get_coordinated_outfit(tool_context, ...):
    existing_outfits = tool_context.state.get(event_key, {})
    # Logic to check conflicts and update state
```

**Observability Integration:**
```python
# Decorator for tool tracing
@trace_tool
def book_taxi(destination, pickup_time, ...):
    # Tool logic - automatically logged

# Orchestrator with logging
class DefensiveOrchestrator:
    def execute_loop(self, goal, user_id):
        logger.info(f"🚀 SESSION START: user_id={user_id}, goal='{goal[:100]}'")
        metrics_collector.record_session_start()
        # ... execution logic ...
        logger.info(f"🏆 SESSION COMPLETE: user_id={user_id}, success={success}")
        metrics_collector.record_session_completion(success)
```

## 6. Evaluation System

The notebook includes comprehensive self-validation:

*   **Scenario-based evaluation:** Each scenario validates functional correctness, resilience, and coordination capabilities.
*   **`run_scenario()` wrapper:** Centralizes display logic, keeping core components clean.
*   **Metrics reporting:** `metrics_collector.report()` displays tool call statistics, success rates, and execution times.
*   **Observability logs:** Structured logs in `agent_execution.log` provide detailed execution traces.

## 7. Future Considerations
*   Replace Mock Tools with real APIs (Google Maps, Uber API, OpenTable).
*   Implement "Human-in-the-loop" via interactive approval for critical decisions.
*   Scale A2A communication to multiple agents with message queues.
*   Add learning system to optimize recovery strategies based on simulation history.
*   Upgrade observability to production-grade solutions (Google Cloud Operations, OpenTelemetry, Datadog).
