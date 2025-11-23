# AI/LLM Integration in Staffflow

This document explains how Staffflow integrates Large Language Models (LLMs) to provide intelligent, context-aware staffing recommendations.

---

## Overview

Staffflow uses **Manus LLM API** (powered by Google's Gemini 2.5 Flash) to analyze complex staffing situations and generate natural language recommendations that hospital administrators can understand and act upon immediately.

### When AI is Used

The LLM is invoked **only when balance status is INADEQUATE**, meaning:
- At least one nurse has 5+ patients, OR
- Standard deviation > 1.5 (severe workload imbalance)

This ensures AI recommendations are provided precisely when human decision-making would benefit most from intelligent analysis.

---

## Architecture

### 1. LLM Client (`server/_core/llm.ts`)

The core LLM client provides a type-safe wrapper around the Manus Forge API:

```typescript
export async function invokeLLM(params: InvokeParams): Promise<InvokeResult>
```

**Key Features:**
- **Model**: `gemini-2.5-flash` (fast, cost-effective, high-quality)
- **Max Tokens**: 32,768 (supports long context)
- **Thinking Budget**: 128 tokens (enables chain-of-thought reasoning)
- **Response Format**: Structured JSON with schema validation
- **Authentication**: Bearer token via `BUILT_IN_FORGE_API_KEY`

### 2. Recommendation Engine (`server/recommendationEngine.ts`)

The recommendation engine orchestrates the AI analysis:

```typescript
export async function generateRecommendations(
  assignments: Assignment[],
  suggestions: RebalancingSuggestion[]
): Promise<Recommendation[]>
```

**Input:**
- Current nurse assignments with workload data
- Algorithmic rebalancing suggestions from the simulation engine

**Output:**
- Prioritized recommendations with natural language explanations
- Estimated impact on workload distribution
- Specific patient transfer suggestions

### 3. API Endpoint (`server/routers.ts`)

The tRPC endpoint exposes recommendations to the frontend:

```typescript
getRecommendations: publicProcedure.query(async () => {
  const state = getSimulationState();
  // ... get assignments and suggestions
  const recommendations = await generateRecommendations(assignments, suggestions);
  return { assignments, suggestions, recommendations };
})
```

---

## How It Works: Step-by-Step

### Step 1: Detect Imbalance

The simulation engine calculates balance metrics:

```typescript
const balanceMetric = calculateBalanceMetric(assignments);
// Returns: { status: 'INADEQUATE', stdDev: 2.05, avgPatients: 2.7, maxPatients: 7 }
```

### Step 2: Prepare Context

The recommendation engine gathers detailed staffing data:

**Overloaded Nurses:**
```
David Martinez (ER): 7 patients, 23/20 workload (115% capacity)
Sarah Johnson (ICU): 6 patients, 19/18 workload (106% capacity)
```

**Underutilized Nurses:**
```
Robert Garcia (ER): 1 patient, 3/20 workload (15% capacity) - Qualifications: ER, MEDSURG
Amanda Lee (ICU): 0 patients, 0/18 workload (0% capacity) - Qualifications: ICU, ER, MEDSURG
```

**Available Transfers:**
```
Move Zachary Bush (acuity 4) from David Martinez to Robert Garcia. Skill match: Yes
Move Emily Davis (acuity 3) from Sarah Johnson to Amanda Lee. Skill match: Yes
```

### Step 3: Build LLM Prompt

A structured prompt is constructed with:

1. **Current Situation Summary**
   - Number of overloaded nurses
   - Number of idle nurses
   - Average utilization percentage

2. **Detailed Nurse Status**
   - Each overloaded nurse's capacity and patient count
   - Each underutilized nurse's availability and qualifications

3. **Available Patient Transfers**
   - Specific patient names and acuity levels
   - Source and target nurses
   - Skill match status

4. **Task Instructions**
   - Identify most critical imbalance
   - Recommend specific transfers
   - Target 70-90% capacity for all nurses
   - Explain impact on patient safety

**Example Prompt:**
```
You are a hospital staffing optimization expert. The current staffing situation 
shows INADEQUATE balance and requires immediate rebalancing.

CURRENT SITUATION:
- 2 nurses are OVERLOADED (>100% capacity)
- 2 nurses are IDLE (0 patients)
- Average utilization: 58.8%

OVERLOADED NURSES:
David Martinez (ER): 7 patients, 23/20 workload (115% capacity)
Sarah Johnson (ICU): 6 patients, 19/18 workload (106% capacity)

IDLE/UNDERUTILIZED NURSES:
Robert Garcia (ER): 1 patient, 3/20 workload (15% capacity) - Qualifications: ER, MEDSURG
Amanda Lee (ICU): 0 patients, 0/18 workload (0% capacity) - Qualifications: ICU, ER, MEDSURG

AVAILABLE PATIENT TRANSFERS:
Move Zachary Bush (acuity 4) from David Martinez to Robert Garcia. Skill match: Yes
Move Emily Davis (acuity 3) from Sarah Johnson to Amanda Lee. Skill match: Yes

Your task:
1. Identify the MOST CRITICAL staffing imbalance (which nurse is most overloaded?)
2. Recommend specific patient transfers that will:
   - Reduce overloaded nurses to manageable ratios (target 70-90% capacity)
   - Utilize idle nurses effectively
   - Prioritize skill-matched transfers
3. Explain the immediate impact on patient safety and nurse workload

Provide a clear, actionable recommendation that hospital administrators can implement immediately.
```

### Step 4: Define Response Schema

The LLM is constrained to return structured JSON:

```typescript
{
  type: "json_schema",
  json_schema: {
    name: "staffing_recommendation",
    strict: true,
    schema: {
      type: "object",
      properties: {
        title: { type: "string", description: "Brief title of the recommendation" },
        description: { type: "string", description: "Detailed description" },
        priority: { type: "string", enum: ["critical", "high", "medium", "low"] },
        estimatedImpact: { type: "string", description: "Expected outcome" }
      },
      required: ["title", "description", "priority", "estimatedImpact"],
      additionalProperties: false
    }
  }
}
```

**Why Structured Output?**
- Ensures consistent, parseable responses
- Prevents hallucination or off-topic responses
- Enables type-safe handling in TypeScript
- Guarantees all required fields are present

### Step 5: Invoke LLM

The system calls the Manus Forge API:

```typescript
const response = await invokeLLM({
  messages: [
    { role: "system", content: "You are a hospital staffing optimization expert..." },
    { role: "user", content: prompt }
  ],
  response_format: { type: "json_schema", json_schema: schema }
});
```

**API Request:**
```http
POST https://forge.manus.im/v1/chat/completions
Authorization: Bearer <BUILT_IN_FORGE_API_KEY>
Content-Type: application/json

{
  "model": "gemini-2.5-flash",
  "messages": [...],
  "response_format": {...},
  "max_tokens": 32768,
  "thinking": { "budget_tokens": 128 }
}
```

### Step 6: Parse LLM Response

The structured JSON is parsed and validated:

```typescript
const content = response.choices[0]?.message.content;
const parsed = JSON.parse(content);

return [{
  title: parsed.title,
  description: parsed.description,
  suggestions: suggestions.slice(0, 3), // Top 3 transfers
  priority: parsed.priority,
  estimatedImpact: parsed.estimatedImpact
}];
```

**Example LLM Response:**
```json
{
  "title": "Immediate Workload Rebalancing: ER to ICU Transfer",
  "description": "David Martinez in the ER is critically overloaded at 115% capacity with 7 patients, while Amanda Lee in ICU is completely idle. This creates significant patient safety risks and nurse burnout potential. Immediate transfer of high-acuity patients to qualified underutilized nurses is essential.",
  "priority": "critical",
  "estimatedImpact": "Reduces David Martinez from 115% to 95% capacity, activates Amanda Lee to 17% capacity. Brings all nurses within safe patient-to-staff ratios and eliminates critical overload conditions."
}
```

### Step 7: Display to User

The frontend renders the recommendation in the AI Recommendations panel:

```tsx
<RecommendationsPanel>
  <h3>{recommendation.title}</h3>
  <Badge variant={recommendation.priority}>{recommendation.priority}</Badge>
  <p>{recommendation.description}</p>
  <p><strong>Expected Impact:</strong> {recommendation.estimatedImpact}</p>
  <ul>
    {recommendation.suggestions.map(s => (
      <li>Move {s.patientName} from {s.fromNurseName} to {s.toNurseName}</li>
    ))}
  </ul>
</RecommendationsPanel>
```

---

## Fallback Mechanism

If the LLM API fails (network error, timeout, API key invalid), the system automatically falls back to **rule-based recommendations**:

```typescript
function generateFallbackRecommendations(
  assignments: Assignment[],
  suggestions: RebalancingSuggestion[]
): Recommendation[]
```

**Fallback Logic:**
1. Count overloaded nurses
2. Determine priority based on severity
3. Select top 3 skill-matched suggestions
4. Generate template-based description
5. Calculate estimated impact using formulas

**Example Fallback Output:**
```json
{
  "title": "URGENT: Rebalance 2 Overloaded Nurses",
  "description": "Critical staffing imbalance detected. David Martinez is at 115% capacity with 7 patients (max: 6). 1 other nurse is also overloaded. Immediate patient reassignment needed to prevent burnout and ensure patient safety.",
  "priority": "critical",
  "estimatedImpact": "Reduces David Martinez's workload from 115% to ~85% capacity. Brings all nurses within safe patient-to-staff ratios (target: 70-90% utilization)."
}
```

This ensures the system **always provides recommendations**, even if AI is temporarily unavailable.

---

## API Configuration

### Required Environment Variables

```bash
# Manus Forge API (LLM service)
BUILT_IN_FORGE_API_URL=https://forge.manus.im
BUILT_IN_FORGE_API_KEY=your-api-key-here
```

### API Endpoint

```
POST https://forge.manus.im/v1/chat/completions
```

### Authentication

```http
Authorization: Bearer <BUILT_IN_FORGE_API_KEY>
```

The API key is automatically injected by the Manus platform when deployed. For local development, you need to obtain a key from the Manus dashboard.

---

## Cost & Performance

### Token Usage

**Typical Request:**
- **Prompt tokens**: ~800-1200 (depends on number of nurses and patients)
- **Completion tokens**: ~200-400 (structured JSON response)
- **Total per request**: ~1000-1600 tokens

**Cost Estimate:**
- Gemini 2.5 Flash: $0.075 per 1M input tokens, $0.30 per 1M output tokens
- Per recommendation: ~$0.0001 (negligible)

### Response Time

- **Average latency**: 1-3 seconds
- **P95 latency**: 3-5 seconds
- **Timeout**: 15 seconds (configured in tRPC)

The system is designed for **asynchronous loading** - recommendations load after the main dashboard, so users aren't blocked.

---

## Advantages of LLM Integration

### 1. Natural Language Understanding

The LLM can interpret complex staffing scenarios and explain them in human-readable terms:

**Algorithmic Output:**
```
Transfer patient_42 from nurse_7 to nurse_3. Workload delta: -4.
```

**LLM Output:**
```
"David Martinez is critically overloaded at 115% capacity. Transferring 
high-acuity patient Zachary Bush to Robert Garcia (who is underutilized at 
15% capacity) will reduce Martinez's workload to a safe 95% level while 
better utilizing available staff."
```

### 2. Context-Aware Prioritization

The LLM considers multiple factors simultaneously:
- Severity of overload (115% vs 105%)
- Number of affected nurses
- Skill match quality
- Patient acuity levels
- Unit-specific constraints

This produces **smarter prioritization** than simple rule-based systems.

### 3. Adaptive Recommendations

The LLM adapts its language and urgency based on the situation:

**Minor Imbalance:**
```
Priority: medium
"Consider redistributing 2-3 patients to improve balance."
```

**Critical Imbalance:**
```
Priority: critical
"IMMEDIATE ACTION REQUIRED: Multiple nurses exceed safe capacity limits."
```

### 4. Explainability

Each recommendation includes:
- **Why** it's needed (current problem)
- **What** to do (specific transfers)
- **Expected outcome** (predicted impact)

This builds **trust and confidence** in the AI system.

---

## Limitations & Considerations

### 1. API Dependency

The system requires internet connectivity and valid API credentials. If the Manus Forge API is unavailable, fallback recommendations are used.

### 2. Latency

LLM inference adds 1-5 seconds of latency. The UI handles this gracefully with loading states.

### 3. Cost

While negligible per request (~$0.0001), high-frequency usage could accumulate costs. The system only invokes the LLM when balance is INADEQUATE to minimize unnecessary calls.

### 4. Determinism

LLM responses have slight variability. The structured JSON schema ensures consistency, but exact wording may differ between identical inputs.

### 5. Context Window

The current implementation sends all nurse and patient data in one prompt. For very large hospitals (100+ nurses), this could exceed context limits. Future versions could implement chunking or summarization.

---

## Future Enhancements

### 1. Multi-Turn Conversations

Allow administrators to ask follow-up questions:
```
User: "What if Robert Garcia is unavailable?"
AI: "In that case, transfer to Amanda Lee in ICU, who has matching qualifications..."
```

### 2. Historical Analysis

Train the LLM on historical staffing patterns:
```
"Based on past data, ER typically needs +2 nurses on Friday evenings."
```

### 3. Predictive Recommendations

Forecast future imbalances before they occur:
```
"In 2 hours, the night shift will be understaffed by 3 nurses. Recommend early callback."
```

### 4. Multi-Objective Optimization

Balance competing goals:
- Minimize workload imbalance
- Maximize skill utilization
- Minimize patient transfers (continuity of care)
- Respect nurse preferences

### 5. Explainable AI

Provide detailed reasoning chains:
```
"I recommended this transfer because:
1. David Martinez is 15% over capacity (highest in ER)
2. Robert Garcia has matching ER qualification
3. Zachary Bush (acuity 4) is high-priority but transferable
4. This reduces standard deviation by 0.8 (largest single improvement)"
```

---

## Testing AI Integration

### Unit Tests

The recommendation engine includes fallback tests:

```typescript
describe("generateRecommendations", () => {
  it("returns fallback recommendations when LLM fails", async () => {
    // Mock LLM failure
    const recommendations = await generateRecommendations(assignments, suggestions);
    expect(recommendations).toHaveLength(1);
    expect(recommendations[0].priority).toBe("critical");
  });
});
```

### Manual Testing

1. Initialize simulation with 50 patients (creates INADEQUATE state)
2. Scroll to AI Recommendations panel
3. Verify recommendation appears within 5 seconds
4. Check that title, description, priority, and impact are present
5. Confirm top 3 transfer suggestions are listed

### API Key Testing

```bash
# Test with invalid key
BUILT_IN_FORGE_API_KEY=invalid pnpm dev
# Should fall back to rule-based recommendations

# Test with valid key
BUILT_IN_FORGE_API_KEY=<real-key> pnpm dev
# Should show LLM-generated recommendations
```

---

## Summary

Staffflow's AI integration demonstrates how LLMs can enhance traditional algorithmic systems:

1. **Algorithmic Core**: Fast, deterministic workload calculations and rebalancing
2. **AI Layer**: Natural language explanations and context-aware prioritization
3. **Fallback Safety**: Rule-based recommendations ensure system always works
4. **Type Safety**: Structured JSON schemas prevent hallucination
5. **Cost Efficiency**: Only invoke LLM when needed (INADEQUATE state)

The result is a **hybrid intelligence system** that combines the reliability of algorithms with the flexibility and explainability of large language models.

---

**Technical Stack:**
- **LLM**: Google Gemini 2.5 Flash
- **API**: Manus Forge (https://forge.manus.im)
- **Authentication**: Bearer token
- **Response Format**: Structured JSON with schema validation
- **Fallback**: Rule-based recommendations
- **Integration**: tRPC type-safe API layer
