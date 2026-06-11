# Technical Architecture & Design - AMD Intelligent Orchestrator v2.0

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    INTELLIGENT ORCHESTRATOR                      │
│                    (Central Decision Engine)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  │  Intake Phase    │  │  Execution Phase │  │  Decision Phase  │
│  │  - Validation    │  │  - Agent Work    │  │  - Scoring       │
│  │  - Routing       │  │  - Feedback      │  │  - Rules Engine  │
│  │  - Tracking      │  │  - Logging       │  │  - Adaptation    │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
│           │                     │                     │
│           v                     v                     v
│  ┌─────────────────────────────────────────────────────────┐
│  │          Core Orchestration Loop                         │
│  │  1. Upload & Validate Documents                          │
│  │  2. Execute Agent Pipeline (with feedback loops)         │
│  │  3. Detect Conflicts                                     │
│  │  4. Calculate Risk Score (adaptive)                      │
│  │  5. Make Final Decision                                  │
│  └─────────────────────────────────────────────────────────┘
│
└─────────────────────────────────────────────────────────────────┘
         │                   │                        │
         v                   v                        v
    ┌─────────────┐   ┌──────────────┐   ┌──────────────────┐
    │  Document   │   │    Agent     │   │   Decision       │
    │  Processing │   │  Execution   │   │   Engine         │
    └─────────────┘   └──────────────┘   └──────────────────┘
```

---

## Data Flow Architecture

### 1. Document Input Phase

```
User Input (File Paths)
    │
    v
[DocumentUploadManager]
    ├─ Validate file exists
    ├─ Check file format
    ├─ Create DocumentFile object
    └─ Store reference
    │
    v
[DocumentExtractor]
    ├─ Detect file type (PDF/Image)
    ├─ Extract text (OCR if needed)
    ├─ Calculate extraction confidence
    └─ Parse structured fields
    │
    v
[CaseContext]
    ├─ Document list
    ├─ Extracted text
    ├─ Parsed fields
    └─ Confidence scores
```

### 2. Agent Execution Phase

```
[CaseContext with Extracted Data]
    │
    ├─→ [Agent 1: CaseIntakeAgent]
    │   ├─ Validate document types
    │   ├─ Generate decision
    │   └─ Return AgentDecision
    │
    ├─→ [Agent 2: DocumentExtractionAgent]
    │   ├─ Validate extracted fields
    │   ├─ Cross-check consistency
    │   └─ Return AgentDecision
    │
    ├─→ [Agent 3: DocumentVerificationAgent]
    │   ├─ Verify authenticity markers
    │   ├─ Check for tampering signs
    │   └─ Return AgentDecision
    │
    ├─→ [Agent 4: IdentityVerificationAgent]
    │   ├─ Compare names across docs
    │   ├─ Verify identity consistency
    │   └─ Return AgentDecision
    │
    ├─→ [Agent 5: FaceMatchAgent]
    │   ├─ Compare selfie with ID
    │   ├─ Generate match confidence
    │   └─ Return AgentDecision
    │
    └─→ [Agent 6: ComplianceScreeningAgent]
        ├─ Screen against PEP list
        ├─ Check sanctions lists
        └─ Return AgentDecision
    │
    v
[Decision List Collected]
```

### 3. Conflict Detection & Resolution

```
[All Agent Decisions]
    │
    v
[Conflict Detection Engine]
    ├─ Type 1: Hard Decision Conflict
    │   ├─ Some agents: APPROVE
    │   └─ Other agents: REJECT
    │
    ├─ Type 2: Confidence Gap
    │   ├─ Max confidence - Min confidence > threshold
    │   └─ Indicates potential issues
    │
    └─ Type 3: Pattern Mismatch
        ├─ Data from documents doesn't align
        └─ Multiple agents reporting issues
    │
    v
[Conflict Resolution Engine]
    ├─ Identify conflicting agents
    ├─ Request re-execution
    ├─ Provide feedback/context
    └─ Update decisions
    │
    v
[Updated Agent Decisions]
```

### 4. Risk Scoring Phase

```
[Agent Decisions with Confidences]
    │
    v
[Risk Weight Mapping]
    ├─ DocumentExtractionAgent: 0.20
    ├─ DocumentVerificationAgent: 0.25
    ├─ IdentityVerificationAgent: 0.15
    ├─ FaceMatchAgent: 0.15
    └─ ComplianceScreeningAgent: 0.25
    │
    v
[Individual Risk Calculation]
    For each agent:
    agent_risk = (1.0 - confidence) * 100
    weighted_risk = agent_risk * weight
    │
    v
[Aggregation]
    total_risk = sum(all weighted_risks)
    │
    v
[Adjustment Factors]
    if conflicts_detected:
        total_risk += conflict_severity_adjustment
    │
    v
[Final Risk Score: 0.0 - 100.0]
```

### 5. Final Decision Phase

```
[Risk Score, Decisions, Conflicts, Confidences]
    │
    v
[Decision Rule Engine]
    │
    ├─ Rule 1: Reject Takes Precedence
    │   if any_agent.decision == REJECT:
    │       final_decision = REJECT
    │
    ├─ Rule 2: High Confidence Approval
    │   if avg_confidence >= threshold and 
    │      approve_count >= (total_agents - 1):
    │       final_decision = APPROVE
    │
    └─ Rule 3: Mixed Signals Review
        else:
            final_decision = REVIEW
    │
    v
[Reasoning Generation]
    Concatenate reasoning from:
    - Conflict information
    - Confidence levels
    - Risk analysis
    - Rule application
    │
    v
[Final Result]
    ├─ Decision (APPROVE/REJECT/REVIEW)
    ├─ Risk Score
    ├─ Confidence
    └─ Reasoning
```

---

## Component Architecture

### Layer 1: Data Models

```
Pydantic BaseModels (Type Safety)
│
├─ DocumentFile
│   ├─ file_id: str
│   ├─ file_name: str
│   ├─ doc_type: DocumentType (Enum)
│   ├─ extraction_confidence: float
│   └─ extracted_text: str
│
├─ AgentDecision
│   ├─ agent_name: str
│   ├─ decision: DecisionType (Enum)
│   ├─ confidence: float
│   ├─ reasoning: str
│   └─ supporting_evidence: Dict
│
└─ CaseContext
    ├─ case_id: str
    ├─ documents: List[DocumentFile]
    ├─ agent_decisions: List[AgentDecision]
    ├─ conflicts_detected: List[Dict]
    ├─ final_decision: DecisionType
    └─ final_risk_score: float
```

### Layer 2: Processing Modules

```
DocumentProcessing Layer
├─ DocumentUploadManager
│   ├─ upload_document(path, type)
│   ├─ get_documents_by_type(type)
│   └─ list_all_documents()
│
└─ DocumentExtractor
    ├─ extract_from_file(path) → (text, confidence)
    ├─ _extract_from_pdf(path)
    ├─ _extract_from_image(path)
    └─ parse_fields(text) → Dict[str, str]
```

### Layer 3: Agent Layer

```
BaseAgent (Abstract)
├─ execute(case) → AgentDecision
├─ can_recheck() → bool
└─ get_feedback() → feedback_info

Specialized Agents
├─ CaseIntakeAgent
├─ DocumentExtractionAgent
├─ DocumentVerificationAgent
├─ IdentityVerificationAgent
├─ FaceMatchAgent
├─ ComplianceScreeningAgent
└─ [CustomAgents...]
```

### Layer 4: Orchestration & Decision Engine

```
IntelligentOrchestratorAgent
├─ Document Management
│   ├─ upload_documents()
│   └─ [upload_manager]
│
├─ Agent Execution
│   ├─ orchestrate()
│   └─ [agents list]
│
├─ Conflict Management
│   ├─ detect_conflicts()
│   ├─ resolve_conflicts()
│   └─ request_agent_feedback()
│
├─ Decision Engine
│   ├─ calculate_risk_score()
│   └─ make_adaptive_decision()
│
└─ Configuration
    ├─ confidence_threshold
    ├─ risk_threshold
    ├─ conflict_threshold
    └─ max_rechecks
```

---

## Conflict Detection Algorithm

### Pattern Matching

```
Input: List of AgentDecisions

Step 1: Categorize Decisions
  approve_decisions = [d for d in decisions if d.decision == APPROVE]
  reject_decisions = [d for d in decisions if d.decision == REJECT]
  review_decisions = [d for d in decisions if d.decision == REVIEW]

Step 2: Hard Conflict Detection
  if approve_decisions AND reject_decisions:
    conflicts.add({
      type: "DECISION_CONFLICT",
      severity: "HIGH",
      approve_agents: [names...],
      reject_agents: [names...]
    })

Step 3: Confidence Gap Detection
  confidences = [d.confidence for d in decisions]
  conf_gap = max(confidences) - min(confidences)
  
  if conf_gap > CONFLICT_THRESHOLD (0.3):
    conflicts.add({
      type: "CONFIDENCE_GAP",
      gap: conf_gap,
      high_conf_agent: name,
      low_conf_agent: name,
      severity: "MEDIUM"
    })

Step 4: Pattern Analysis
  if too_many_review_decisions:
    conflicts.add({
      type: "UNCERTAINTY_PATTERN",
      severity: "MEDIUM"
    })

Output: List of Conflict objects
```

### Resolution Strategy

```
For each conflict:

if DECISION_CONFLICT:
  ├─ Identify uncertain agents (lower confidence)
  ├─ Request feedback from uncertain agents
  ├─ Provide context: "Agent B strongly disagrees"
  ├─ Agent re-executes with new context
  └─ Update decisions

elif CONFIDENCE_GAP:
  ├─ Request re-check from low-conf agent
  ├─ Provide evidence from high-conf agent
  ├─ Allow agent to update confidence/decision
  └─ Track rechecks_performed

elif UNCERTAINTY_PATTERN:
  ├─ Flag for manual review
  ├─ Provide all evidence
  └─ Recommend escalation
```

---

## Risk Scoring Algorithm (Enhanced)

### Mathematical Model

```
Input: List of AgentDecisions

Base Risk Calculation:
──────────────────

For each agent:
  agent_risk = (1.0 - confidence) * 100
  
  Example: confidence=0.8 → risk=20
           confidence=0.5 → risk=50
           confidence=0.1 → risk=90

Weighted Aggregation:
────────────────────

weights = {
  "DocumentExtractionAgent": 0.20,
  "DocumentVerificationAgent": 0.25,
  "IdentityVerificationAgent": 0.15,
  "FaceMatchAgent": 0.15,
  "ComplianceScreeningAgent": 0.25,
}

weighted_total = Σ(agent_risk × weight)

Conflict Adjustment:
───────────────────

if num_conflicts > 0:
  conflict_risk = num_conflicts * 5  # Per conflict
  adjusted_total = weighted_total + conflict_risk

Final Normalization:
───────────────────

final_risk = min(adjusted_total, 100.0)

Output: final_risk (0.0 to 100.0)
```

### Risk Thresholds

```
Risk Score Range    │  Recommended Action
─────────────────────────────────────────
0-20                │  APPROVE (Low Risk)
20-40               │  APPROVE with REVIEW (Medium-Low)
40-60               │  REVIEW (Medium Risk)
60-80               │  REVIEW with ESCALATION (Medium-High)
80-100              │  REJECT (High Risk)
```

---

## Decision Rule Engine

### Pseudocode

```
function make_adaptive_decision(case):
  decisions = case.agent_decisions
  risk_score = calculate_risk_score(decisions)
  
  # Count decision types
  approve_count = count(d for d in decisions if d.decision == APPROVE)
  reject_count = count(d for d in decisions if d.decision == REJECT)
  review_count = count(d for d in decisions if d.decision == REVIEW)
  
  # Calculate average confidence
  avg_confidence = mean([d.confidence for d in decisions])
  
  # Rule-based decision
  reasoning = []
  
  Rule 1: Hard Reject
  ───────────────────
  if reject_count > 0:
    reasoning.append(f"Reject signals from {reject_count} agent(s)")
    return (REJECT, reasoning)
  
  Rule 2: High Confidence Approval
  ────────────────────────────────
  if avg_confidence >= CONFIDENCE_THRESHOLD and 
     approve_count >= (len(decisions) - 1):
    reasoning.append(f"High confidence: {avg_confidence:.2f}")
    reasoning.append(f"Approve: {approve_count} agents")
    return (APPROVE, reasoning)
  
  Rule 3: Mixed Signals → Review
  ──────────────────────────────
  if review_count > 0 or avg_confidence < CONFIDENCE_THRESHOLD:
    reasoning.append(f"Confidence: {avg_confidence:.2f}")
    reasoning.append(f"Review signals: {review_count}")
    return (REVIEW, reasoning)
  
  Default: Review
  ───────────────
  reasoning.append("Unable to determine clear decision")
  return (REVIEW, reasoning)
```

---

## Performance Characteristics

### Time Complexity

```
Operation                          Time Complexity
──────────────────────────────────────────────────
Document Upload                    O(n) where n = num_documents
Document Extraction                O(n*m) where m = avg_doc_size
Agent Execution                    O(a) where a = num_agents
Conflict Detection                 O(a²) worst case
Risk Calculation                   O(a)
Final Decision                     O(a)
──────────────────────────────────────────────────
Total (no conflicts)               O(n*m + a²)
Total (with conflicts/rechecks)    O(n*m + (a² * r)) where r = rechecks
```

### Space Complexity

```
Data Structure                     Space Complexity
──────────────────────────────────────────────────
CaseContext                        O(n*m) where m = avg_doc_size
Agent Decisions                    O(a) where a = num_agents
Conflicts Buffer                   O(c²) where c = num_conflicts
Evidence Trail                     O(a * e) where e = evidence items
──────────────────────────────────────────────────
Total Memory                       O(n*m + a*e)
```

### Typical Execution Times (Estimated)

```
Component                          Time (ms)
──────────────────────────────────────────
Document Upload                    10-50
Text Extraction (PDF)              50-200
Text Extraction (Image OCR)        100-500
Agent Execution (all)              50-200
Conflict Detection                 10-50
Decision Making                    10-30
──────────────────────────────────────────
Total (normal case)                230-1030 ms
Total (with conflict resolution)   500-2000 ms
```

---

## Extension Points

### 1. Custom Agent Development

```python
class CustomAgent(BaseAgent):
    def __init__(self):
        super().__init__("CustomAgent")
    
    def execute(self, case: CaseContext) -> AgentDecision:
        # Your logic here
        return AgentDecision(...)
    
    def can_recheck(self) -> bool:
        return True  # Enable feedback loops

# Add to orchestrator
orchestrator.agents.append(CustomAgent())
```

### 2. LLM Integration Points

```python
# At various decision points:

# Point 1: Document interpretation
if llm_enabled:
    llm_analysis = query_llm(f"Analyze extracted text: {text}")

# Point 2: Conflict resolution
if conflict_detected:
    llm_reasoning = query_llm(f"Help resolve conflict between {agent1} and {agent2}")

# Point 3: Final decision rationale
if explain_requested:
    explanation = query_llm(f"Explain why we made {decision} decision")
```

### 3. Database Integration Points

```python
# Store case
db.save_case(case)

# Query history
past_cases = db.query({"name": customer_name})

# Update decision after manual review
db.update_case_decision(case_id, human_decision)

# Analytics
stats = db.aggregate_decisions()
```

### 4. API Integration Points

```python
# Before agent execution
validate_customer_kyc_externally()

# During document extraction
get_additional_documents_from_api()

# After decision
notify_downstream_system(case.final_decision)
```

---

## Error Handling Strategy

### Levels of Error Handling

```
Level 1: Input Validation
├─ File existence check
├─ File format validation
├─ Size limits
└─ Error: Return validation failure

Level 2: Processing Errors
├─ OCR failure → Fallback to basic extraction
├─ Field parsing error → Return empty field
└─ Error: Log but continue

Level 3: Agent Execution Errors
├─ Agent crashes → Skip and continue
├─ Agent timeout → Mark as RECHECK needed
└─ Error: Log but continue with other agents

Level 4: Decision Logic Errors
├─ Insufficient data → Default to REVIEW
├─ Conflicting thresholds → Use conservative approach
└─ Error: Default decision with explanation

Level 5: Graceful Degradation
├─ If critical agents fail → Escalate to manual review
├─ If risk calculation fails → Use base risk (50)
└─ If all else fails → REVIEW with notes
```

---

## Testing Strategy

### Unit Tests
```
- DocumentExtractor.extract_from_file()
- DocumentExtractor.parse_fields()
- Agent.execute() for each agent
- detect_conflicts()
- calculate_risk_score()
- make_adaptive_decision()
```

### Integration Tests
```
- Full orchestrate() flow
- Document upload + extraction
- Agent pipeline execution
- Conflict resolution flow
```

### End-to-End Tests
```
- Real document processing
- Multi-document cases
- Conflict scenarios
- Edge cases (empty docs, corrupted files)
```

### Performance Tests
```
- Large document handling
- Batch processing speed
- Memory usage under load
- Concurrent case processing
```

---

## Security Considerations

### Data Protection
```
- Encrypt extracted PII at rest
- Hash sensitive identifiers
- Secure temporary file handling
- Audit logging of all decisions
```

### Input Validation
```
- File type whitelist
- Size limits on documents
- Malware scanning
- Path traversal prevention
```

### Access Control
```
- Role-based decision review
- API authentication
- Audit trails
- Compliance logging
```

---

## Future Enhancements

### Short Term (v2.1-2.2)
- [ ] Real OCR library integration
- [ ] Database backend
- [ ] REST API wrapper
- [ ] Web UI for file upload

### Medium Term (v3.0)
- [ ] Machine learning risk scoring
- [ ] Advanced conflict resolution
- [ ] Multi-language document support
- [ ] Real-time monitoring dashboard

### Long Term (v3.5+)
- [ ] Federated agent networks
- [ ] Adaptive threshold learning
- [ ] Predictive risk analysis
- [ ] Blockchain audit trail

---

**Architecture designed for extensibility, maintainability, and production deployment.**
