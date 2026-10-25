# NovaMart Multi-Agent Customer Support System — Submission Notes

**Udacity Nanodegree: AWS AI Agent Engineering (Course 3.3)**  
**Student:** K. Satya Pranav  
**Project:** Multi-Agent E-commerce RAG with Amazon Bedrock AgentCore & Strands Agents SDK  
**Test Score:** **120 / 120 points (100% Perfect Score)**  

---

## Executive Summary

This project implements a production-grade, enterprise customer support system for **NovaMart**, an e-commerce platform. The system is engineered using the **Strands Agents SDK** and **Amazon Bedrock AgentCore**, deploying an **Orchestrator → Workers** hierarchy with parallel multi-agent Retrieval-Augmented Generation (RAG), strict Bedrock Guardrails, AgentCore Memory, and end-to-end distributed observability via AWS CloudWatch and AWS X-Ray.

All tasks have been rigorously tested against real AWS resources in `us-east-1` and verified via automated test suites.

---

## Rubric Verification Matrix

| Rubric Criterion | Implementation Details & File Reference | Verification Evidence | Status |
| :--- | :--- | :--- | :---: |
| **1. Multi-Agent Graph: Worker Agents** | • `build_inventory_agent()` (lines 284–445): Implements `check_order_status`, `get_customer_tier`, `list_customer_orders` querying DynamoDB composite keys.<br>• `build_refund_agent()` (lines 447–621): Implements `get_inventory_context`, `initiate_refund` applying 30-day (Standard) and 60-day (Premium) windows.<br>• `build_communication_agent()` (lines 834–980): Implements `get_full_workflow_context` for empathetic synthesis.<br>• All workers use `config.WORKER_MODEL_ID` (Claude Sonnet 4.5). Inventory & Refund use `temperature=0.1`; Communication uses `temperature=0.3`. | `tests/test_agent.py task2`<br>`evidence/required/01-tests-all-120.png` | **PASS (100%)** |
| **2. Multi-Agent RAG with Parallel Retrieval** | • `build_policy_agent()` (lines 623–832): Creates three dedicated retriever agents (`ReturnsPolicyRetrieverAgent`, `ShippingPolicyRetrieverAgent`, `WarrantyPolicyRetrieverAgent`) with `temperature=0.0`.<br>• `search_all_policies()` tool invokes all 3 sub-agents concurrently via `concurrent.futures.ThreadPoolExecutor(max_workers=3)` and collects via `as_completed()`.<br>• Policy coordinator uses `temperature=0.2` to synthesize non-hallucinatory policy answers. | `tests/test_agent.py task2`<br>`tests/test_agent.py task5`<br>`evidence/required/01-tests-all-120.png` | **PASS (100%)** |
| **3. OrchestratorAgent & Shared WorkflowState** | • `build_orchestrator_agent()` (lines 1072–1461): Model is `config.ORCHESTRATOR_MODEL_ID` (Claude Haiku 4.5) with `temperature=0.0`.<br>• Enforces all 6 routing rules via explicit system prompt instructions.<br>• Shared state in DynamoDB `WorkflowStateTable` managed with optimistic locking (`_create_workflow_state`, `_read_workflow_state`, `_update_workflow_state` with `expected_version`).<br>• `route_to_communication_agent` is guaranteed to be the final tool call. Orchestrator never writes customer responses directly. | `tests/test_agent.py task2`<br>`evidence/required/01-tests-all-120.png` | **PASS (100%)** |
| **4. Bedrock Guardrail Creation** | • `create_guardrail()` (lines 1645–1685): Creates Bedrock Guardrail with:<br>  - Content Filters: SEXUAL, VIOLENCE, HATE (HIGH); INSULTS, MISCONDUCT (MEDIUM).<br>  - PII Policy: BLOCK SSN & Credit Cards; ANONYMIZE Email & Phone.<br>  - Topic Policy: DENY competitor products, pricing negotiations, legal threats using `STANDARD` tier and `crossRegionConfig={"guardrailProfileIdentifier": "us.guardrail.v1:0"}`.<br>  - Word Policy: Managed profanity list enabled.<br>  - Publishes numbered version via `create_guardrail_version()`, returning `(guardrail_id, guardrail_version)`. | Guardrail ID: `jtjyz5e3odep`<br>Version: `2`<br>`evidence/required/01-tests-all-120.png` | **PASS (100%)** |
| **5. AgentCore Runtime Deployment** | • `deploy_to_agentcore_runtime()` (lines 1687–1767): Stages application to `build/runtime/`, injects runtime environment variables (`AWS_REGION`, `PROJECT_NAME`, `RETURNS_KB_ID`, `SHIPPING_KB_ID`, `WARRANTY_KB_ID`, `AGENT_LOG_GROUP`, `GUARDRAIL_ID`, `GUARDRAIL_VERSION`).<br>• Configures runtime with `PUBLIC` networking, `HTTP` protocol, and `config.AGENTCORE_ROLE_ARN`.<br>• Deploys via `agentcore_cli.deploy()` and resolves ARN via `agentcore_cli.deployed_runtime_arn()`.<br>• `_apply_guardrail()` attaches the guardrail to all agent models. | Runtime ARN: `arn:aws:bedrock-agentcore:us-east-1:053670098006:runtime/udacity_agentcore_runtime-RFDiPt97es`<br>`evidence/required/01-tests-all-120.png` | **PASS (100%)** |
| **6. AgentCore Memory Configuration** | • `configure_memory()` (lines 1769–1819): Creates AgentCore Memory resource using `agentcore_control.create_memory()` with `summaryMemoryStrategy` (`SESSION_SUMMARY`) and `eventExpiryDuration=7` (7-day retention).<br>• Polls until ACTIVE and returns `memoryArn`. | `tests/test_agent.py task4`<br>`evidence/required/01-tests-all-120.png` | **PASS (100%)** |
| **7. Bedrock Knowledge Bases with S3 Vectors** | • Three dedicated Knowledge Bases in `us-east-1` backed by S3 Vectors bucket and matching vector indexes (`returns-policy-index`, `shipping-policy-index`, `warranty-policy-index`).<br>• Embedding model: `amazon.titan-embed-text-v2:0`.<br>• Fully synced data sources under `policies/returns/`, `policies/shipping/`, `policies/warranty/`.<br>• Knowledge Base IDs populated in `.env` (`UOM8KM5NPK`, `VFHKVWDNB8`, `D3ZBNTF9K6`). | `tests/test_agent.py task5`<br>`evidence/console/kb-returns-sync.png`<br>`evidence/required/01-tests-all-120.png` | **PASS (100%)** |
| **8. Observability & Distributed Tracing** | • `configure_observability()` (lines 1821–1861): Configures CloudWatch Logs (`config.AGENT_LOG_GROUP`, `logLevel='INFO'`, `enabled=True`) and AWS X-Ray tracing (`enabled=True`, `samplingRate=1.0`).<br>• Every `@tool` invocation recorded with subsegments; distributed trace map links Orchestrator → Workers → Knowledge Bases. | `tests/test_agent.py task6`<br>`evidence/required/02-xray-service-map.png` | **PASS (100%)** |
| **9. Industry Best Practices & Model Tiering** | • Orchestrator uses cost-efficient, ultra-fast `config.ORCHESTRATOR_MODEL_ID` (Claude Haiku 4.5).<br>• Workers use high-reasoning `config.WORKER_MODEL_ID` (Claude Sonnet 4.5).<br>• Comprehensive docstrings, robust error handling, optimistic locking, and clean separation of concerns throughout. | `src/agent_orchestrator.py` | **PASS (100%)** |
| **10. Standout Achievements** | • **Adversarial Guardrail Red-Teaming**: Full security audit against prompt injection, competitor pricing, and legal threats (`evidence/standout/guardrail_adversarial.md`).<br>• **Cross-Session Storage**: DynamoDB persistent session management.<br>• **CloudWatch Production Dashboard**: Custom metrics dashboard (`evidence/standout/dashboard-novamart-agents.png`).<br>• **Full-Stack Customer UI**: React + Vite frontend connected to AgentCore endpoint (`frontend/`). | `evidence/standout/` folder | **EXEMPLARY** |

---

## Primary Deliverables Included

1. **`src/agent_orchestrator.py`**: Completed multi-agent system implementation (all TODOs fulfilled).
2. **`.env`**: Fully configured environment file with valid AWS IDs.
3. **`evidence/required/01-tests-all-120.png`**: Terminal screenshot showing 120/120 pts across all tasks.
4. **`evidence/required/02-xray-service-map.png`**: AWS CloudWatch X-Ray Service Map screenshot demonstrating full call chain graph.
5. **`SUBMISSION_NOTES.md`**: Complete rubric verification and code mapping.
6. **`TEST_RESULTS.md`**: Comprehensive test report and execution traces.
7. **`REFLECTION.md`**: Architectural decisions, safety considerations, and production insights.
8. **`submission.zip`**: Self-contained submission archive ready for Udacity upload (< 15 MB).
