# Fraud Detection Workflow - Quick Start Guide

## 🚀 5-Minute Setup

### 1. Configure Environment

```bash
# Copy sample env file
cd agentic_ai/workflow/fraud_detection
cp .env.sample .env

# Edit .env with your Azure OpenAI credentials
nano .env
```

### 2. Start MCP Server

```bash
# Terminal 1
cd mcp
python mcp_service.py
```

### 3. Run Workflow

```bash
# Terminal 2
cd agentic_ai/workflow/examples/fraud_detection
python fraud_detection_workflow.py
```

## 📋 Key Commands

### Test with Different Alerts

Edit `fraud_detection_workflow.py` and modify the `alerts` list:

```python
# High-risk multi-country login
SuspiciousActivityAlert(
    alert_id="ALERT-001",
    customer_id=1,
    alert_type="multi_country_login",
    severity="high"
)

# Medium-risk data spike
SuspiciousActivityAlert(
    alert_id="ALERT-002",
    customer_id=2,
    alert_type="data_spike",
    severity="medium"
)
```

### View Logs

```bash
# Real-time log viewing
tail -f workflow.log

# Search for specific events
grep "FraudRiskAggregator" workflow.log
```

## 🎯 Common Tasks

### Change Risk Threshold

**File**: `fraud_detection_workflow.py`  
**Line**: ~810

```python
# Current: 0.6 (60% risk)
(lambda assessment: assessment.overall_risk_score >= 0.6, analyst_review)

# Make more sensitive (catches more):
(lambda assessment: assessment.overall_risk_score >= 0.5, analyst_review)

# Make less sensitive (fewer alerts):
(lambda assessment: assessment.overall_risk_score >= 0.7, analyst_review)
```

### Simulate Analyst Decisions

**File**: `fraud_detection_workflow.py`  
**Line**: ~890

```python
analyst_decision = AnalystDecision(
    alert_id=alerts[0].alert_id,
    approved_action="lock_account",  # Change: "clear", "refund_charges", "both"
    analyst_notes="Your notes here",
    analyst_id="analyst_001",
)
```

### Test with Custom Customer

```python
# Add to MCP database first via mcp/data/create_db.py
# Then use in alert:
SuspiciousActivityAlert(
    alert_id="ALERT-999",
    customer_id=999,  # Your custom customer
    alert_type="custom_test",
    description="Testing with custom customer",
    timestamp=datetime.now().isoformat(),
    severity="medium",
)
```

## 🐛 Troubleshooting

### Error: "Cannot connect to MCP server"
```bash
# Check if MCP server is running
ps aux | grep mcp_service

# Restart MCP server
cd mcp
python mcp_service.py
```

### Error: "Azure OpenAI authentication failed"
```bash
# Verify environment variables
echo $AZURE_OPENAI_API_KEY
echo $AZURE_OPENAI_ENDPOINT

# Re-export if needed
export AZURE_OPENAI_API_KEY="your-key"
```

### Workflow Hangs at Analyst Review
```bash
# Check logs for RequestInfoEvent
grep "ANALYST REVIEW" workflow.log

# Verify analyst decision is being sent
# Check around line 890 in fraud_detection_workflow.py
```

### MCP Tools Not Found
```bash
# Verify MCP connection
curl http://localhost:8000/list_tools

# Check filtered tools in logs
grep "filtered tools" workflow.log
```

## 📊 Understanding Output

### Successful Flow (Low Risk)
```
[AlertRouter] → [3 Analysts] → [Aggregator] 
→ [AutoClear] → [Notification] ✅
```

### Successful Flow (High Risk)
```
[AlertRouter] → [3 Analysts] → [Aggregator] 
→ [AnalystReview] ⏸️ → [FraudAction] → [Notification] ✅
```

### Risk Score Interpretation
- **0.0 - 0.3**: Low risk (auto-clear)
- **0.3 - 0.6**: Medium risk (auto-clear by default)
- **0.6 - 0.8**: High risk (requires analyst review)
- **0.8 - 1.0**: Critical risk (requires analyst review)

## 🔧 Quick Customizations

### Add Custom Risk Indicator

**Location**: `UsagePatternExecutor.handle_alert()`

```python
# Add to prompt:
"""
Additional checks:
- Look for X pattern
- Check Y threshold
- Flag Z behavior
"""
```

### Change Analyst Instructions

**Location**: `create_fraud_detection_workflow()`

```python
analyst_review = RequestInfoExecutor(
    name="analyst_review",
    request_info={
        "type": "fraud_analyst_review",
        "instructions": "YOUR CUSTOM INSTRUCTIONS HERE"
    },
)
```

### Modify Action Logic

**Location**: `FraudActionExecutor.handle_decision()`

```python
if "lock" in decision.approved_action:
    # Add custom lock logic
    custom_lock_account(decision)
```

## 📖 Learning Path

### Beginner
1. ✅ Run the example as-is
2. ✅ Change risk threshold
3. ✅ Modify alert severity
4. ✅ Review logs to understand flow

### Intermediate
1. ✅ Add custom risk indicators
2. ✅ Modify analyst instructions
3. ✅ Create custom alert types
4. ✅ Add fourth specialist agent

### Advanced
1. ✅ Implement persistent checkpoint storage
2. ✅ Add real-time UI for analyst review
3. ✅ Integrate with external systems
4. ✅ Add OpenTelemetry tracing

## 🎓 Key Concepts Checklist

- [ ] Understand fan-out pattern (1 → N)
- [ ] Understand fan-in pattern (N → 1)
- [ ] Understand switch/case routing
- [ ] Understand human-in-the-loop
- [ ] Understand checkpointing
- [ ] Understand MCP tool filtering
- [ ] Understand LLM agent executors
- [ ] Understand type-safe messaging

## 📞 Getting Help

### Check Documentation
- [Full README](README.md)
- [Implementation Details](IMPLEMENTATION.md)
- [Workflow Architecture](../../WORKFLOW_README.md)

### Review Examples
- [Human-in-the-Loop](../human-in-the-loop/)
- [Control Flow](../control-flow/)
- [Parallelism](../parallelism/)

### Common Issues
See [Troubleshooting](#-troubleshooting) section above

---

**Pro Tip**: Start with the default configuration and make one change at a time to understand each component's impact!
