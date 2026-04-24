# Content Moderator - Claude Code Configuration

**Project:** Content Moderator for Community Platform  
**Architecture:** Two-Confidence LLM-Based Moderation System  
**Status:** Specification Complete - Ready for Implementation

---

## 📋 Project Overview

This project implements an AI-powered content moderation system for a tabletop miniature hobbyist community platform with ~180K active users generating ~12K posts per day.

**Key Innovation:** Two-confidence architecture (compliance_confidence + violation_confidence) that enables trust-first moderation with explicit grey-zone handling for uncertain cases requiring human review.

**Primary Goal:** Protect community trust by ensuring harmful content never gets through (false negative rate <1%), while efficiently automating decisions for clearly compliant or violating content (70-80% automation rate).

---

## 📁 Project Structure

```
FDE_ContentModerator/
├── Specification/
│   ├── ProjectSpecification.md          # ⭐ Main specification (100% complete)
│   ├── policies.md                      # Community policies for LLM validation
│   ├── ProjectSuccessMetrics.md         # Success metrics and KPIs
│   └── [verification reports]           # Architecture verification docs
├── TestData/
│   ├── technical_tests.json             # 9 API validation test cases
│   ├── business_tests.json              # 20 policy compliance test cases
│   └── README.md                        # Test data documentation
├── README.md                            # Project README
└── CLAUDE.md                            # This file (Claude Code configuration)
```

---

## 🎯 Key Documentation

### Primary Specification (MUST READ)
**Location:** `Specification/ProjectSpecification.md`

This is the **authoritative source** for all implementation details. It contains:
- Complete architecture overview with UML diagrams
- Two-confidence delegation logic and thresholds
- All REST API specifications (endpoints, request/response formats)
- Database schema with both confidence fields
- Frontend UI specifications (PyQT)
- LLM prompt template
- Functional requirements (REQ-MD-01 through REQ-MD-09)
- Test approach and success criteria

**Status:** ✅ 100% complete, verified, production-ready

### Community Policies
**Location:** `Specification/policies.md`

LLM uses these policies to classify messages. Covers:
- Harassment (bullying, insults, threats)
- Spam (commercial advertisements)
- Self-promotion limits
- Doxxing (sharing personal info)
- IP disputes
- Off-topic content

### Success Metrics
**Location:** `Specification/ProjectSuccessMetrics.md`

Comprehensive metrics covering:
- Accuracy metrics (95% classification accuracy target)
- False negative rate (<1% - MOST CRITICAL)
- Automation rate (70-80% target)
- Trust & safety metrics (0 trust-damaging incidents)
- Efficiency and cost metrics

### Test Data
**Location:** `TestData/`
- `technical_tests.json` - 9 API validation tests (8 negative, 1 positive)
- `business_tests.json` - 20 policy compliance tests (10 positive, 10 negative)
- `README.md` - Test data documentation with usage examples

---

## 🏗️ Architecture Overview

### Core Components

1. **Content Moderator Backend** (Python)
   - REST API with 4 endpoints
   - LLM integration (Claude Sonnet via Anthropic API)
   - Two-confidence decision engine
   - In-memory database (Python dict)

2. **Content Moderator Frontend** (PyQT)
   - Message review interface
   - Test message submission view
   - Displays both confidence scores and delegation status

3. **External Integration**
   - Message Board component (existing system)
   - REST-based communication only

### Two-Confidence Architecture

**Revolutionary Approach:** Instead of single confidence score, system produces TWO independent scores:

```python
# LLM returns:
{
  "compliance_confidence": 0.95,  # How confident message is compliant
  "violation_confidence": 0.05    # How confident message violates
}

# Decision logic:
if violation_confidence >= DELEGATION_THRESHOLD_BLOCK:  # 0.95
    delegation = false  # Auto-block, no human review
elif compliance_confidence >= DELEGATION_THRESHOLD_ALLOW:  # 0.80
    delegation = false  # Auto-allow, no human review
else:
    delegation = true  # Grey zone, requires human review
```

**Key Insight:** Allows system to distinguish between:
- High confidence decisions (auto-decide)
- Low confidence / uncertain cases (escalate to human)
- Maintains explicit grey zone for trust-first moderation

---

## 🔧 Technology Stack

| Component | Technology | Notes |
|-----------|-----------|-------|
| **Language** | Python | Primary implementation language |
| **UI Framework** | PyQT | Desktop application for moderators |
| **Database** | In-memory (Python dict) | No persistence for demo version |
| **API** | REST | All inter-component communication |
| **LLM Provider** | Anthropic API | Claude Sonnet model |
| **Authentication** | API Keys | Stored in .env file |

### Configuration (.env)
```ini
# Anthropic API
ANTHROPIC_API_KEY=sk-ant-...

# LLM Settings
LLM_TIMEOUT=30
LLM_MAX_TOKENS=5000
LLM_TEMPERATURE=0.3          # Low temperature for consistent classification

# Delegation Thresholds
DELEGATION_THRESHOLD_ALLOW=0.80   # Auto-allow threshold
DELEGATION_THRESHOLD_BLOCK=0.95   # Auto-block threshold (high bar)

# Server
HOST=localhost
PORT=8000

# External System
MESSAGE_BOARD_ENDPOINT=localhost:9000/api/updateMessage
```

---

## 📡 REST API Endpoints

### 1. POST /api/validateMessage
**Purpose:** Validate message for policy compliance

**Request:**
```json
{
  "id": "123",
  "title": "Message title (max 100 chars)",
  "body": ["message text (max 256 chars each)"]
}
```

**Response (HTTP 200):**
```json
{
  "verification": "positive" | "negative",
  "reason": "OK" | "HARASSMENT" | "SPAM" | "SELF_PROMOTING" | "DOXXING" | "IP_DISPUTES" | "OFF_TOPIC",
  "delegation": boolean,
  "reasoning": "Explanation text",
  "compliance_confidence": 0.95,
  "violation_confidence": 0.05
}
```

**Error Response:** HTTP 400 or 500 with empty JSON `{}`

### 2. GET /api/getMessagesToReview
**Purpose:** Get list of messages requiring human review

**Response:**
```json
{
  "messages": [{
    "id": 123,
    "title": "message title",
    "reason": "OFF_TOPIC",
    "compliance_confidence": 0.55,
    "violation_confidence": 0.65
  }]
}
```

### 3. GET /api/getMessageDetails?id={id}
**Purpose:** Get full message details for review

**Response:**
```json
{
  "message": {
    "title": "message title",
    "body": ["Initial message", "reply"],
    "reason": "OFF_TOPIC",
    "reasoning": "Detailed explanation",
    "compliance_confidence": 0.55,
    "violation_confidence": 0.55
  }
}
```

### 4. PATCH /api/setUserDecision?id={id}&user_decision={decision}
**Purpose:** Set human moderator decision

**Parameters:**
- `id`: Message database ID
- `user_decision`: USER_VERIFIED_POSITIVE | USER_VERIFIED_NEGATIVE

**Action:** Updates database and notifies Message Board system

---

## 🗄️ Database Schema

### Table: messages

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | integer | PRIMARY KEY | Auto-increment record ID |
| external_id | integer | NOT NULL | Message Board message ID |
| title | char[100] | NOT NULL | Message title |
| body | char[1000] | NOT NULL | Message body (JSON array) |
| reason | char[20] | NOT NULL | OK, HARASSMENT, SPAM, etc. |
| reasoning | char[1024] | NOT NULL | LLM explanation |
| **compliance_confidence** | float | NOT NULL | LLM compliance score (0.0-1.0) |
| **violation_confidence** | float | NOT NULL | LLM violation score (0.0-1.0) |
| user_decision | char[20] | NULL | USER_VERIFIED_POSITIVE/NEGATIVE |
| cdate | datetime | NOT NULL | Creation timestamp |
| mdate | datetime | NULL | Modification timestamp |

**Storage Logic:**
- Save when: `verification == "negative"` OR `delegation == true`
- All violations saved for audit trail
- All grey zone cases saved for human review

---

## 🧪 Testing Approach

### Test Data Location
- **Technical tests:** `TestData/technical_tests.json` (9 test cases)
- **Business tests:** `TestData/business_tests.json` (20 test cases)

### Test Categories

#### 1. Technical Validation Tests (9 tests)
**Purpose:** Validate API request validation logic

**Coverage:**
- Missing required fields (id, title, body)
- Invalid data types (non-numeric id)
- Size limits (title > 100 chars, body > 256 chars, total > 10K chars)
- Empty body array
- Valid request baseline

**Expected:** 8x HTTP 400, 1x HTTP 200

#### 2. Business Validation Tests (20 tests)
**Purpose:** Validate LLM policy classification

**Coverage:**
- 10 positive cases (compliant messages)
- 10 negative cases (policy violations):
  - 2x HARASSMENT
  - 2x SPAM
  - 2x SELF_PROMOTING
  - 1x DOXXING
  - 1x IP_DISPUTES
  - 2x OFF_TOPIC

**Target Accuracy:** ≥95% (19/20 correct classifications)

### Running Tests

```bash
# Run all technical tests
python tests/run_tests.py --type technical

# Run all business tests
python tests/run_tests.py --type business

# Run all tests
python tests/run_tests.py --type all
```

### Test Framework Requirements
- Load test data from JSON files
- Execute API calls
- Compare actual vs expected results
- Generate pass/fail report
- Track accuracy metrics

---

## 💡 Implementation Guidelines

### Critical Requirements

1. **Trust-First Principle**
   - False negatives (<1%) are MORE CRITICAL than false positives (<15%)
   - Better to review harmless content than miss harmful content
   - Conservative thresholds (0.95 for block, 0.80 for allow)

2. **Two-Confidence Logic**
   - ALWAYS store both `compliance_confidence` and `violation_confidence`
   - NEVER merge them into single score
   - Use proper threshold comparison logic
   - Maintain explicit grey zone

3. **Delegation Semantics**
   - `delegation = false` → System auto-decides (no human review)
   - `delegation = true` → Requires human review (grey zone or uncertain)
   - This is CRITICAL - logic must be correct

4. **Database Storage**
   - Save ALL negative validations (for audit trail)
   - Save ALL grey zone cases (delegation=true)
   - Do NOT save auto-allowed messages (saves storage)

### LLM Integration

**Prompt Location:** See `Specification/ProjectSpecification.md` lines 525-598

**Key Requirements:**
- Load policies from `Specification/policies.md`
- Include 10 most recent human-verified violations as examples
- Request TWO confidence scores in response
- Use temperature 0.3 (configurable via .env)
- Timeout: 30 seconds
- Max tokens: 5000

**LLM Response Format:**
```json
{
  "validation": "positive" | "negative",
  "reason": "OK" | "HARASSMENT" | "SPAM" | "SELF_PROMOTING" | "DOXXING" | "IP_DISPUTES" | "OFF_TOPIC",
  "reasoning": "1-2 sentence explanation",
  "compliance_confidence": 0.0-1.0,
  "violation_confidence": 0.0-1.0
}
```

### Error Handling

**Demo Version - Basic Error Handling:**
- No retries on LLM API failures
- Return HTTP 500 on errors
- Empty JSON `{}` for error responses
- Log errors but don't expose details to client

### Security Considerations

- API keys in .env file (never in code)
- Validate all input fields (type, length, required)
- Sanitize error messages (no internal details)
- Be aware of OWASP top 10 vulnerabilities
- No SQL injection (using parameterized queries)
- No XSS (JSON responses only)

---

## 🎯 Success Criteria

### Primary Metrics (MUST ACHIEVE)

1. **False Negative Rate: <1%** (CRITICAL)
   - Harmful content that gets through
   - Target: Trend toward 0%

2. **Classification Accuracy: ≥95%**
   - LLM decisions match human review
   - Measured across all reviewed messages

3. **Automation Rate: 70-80%**
   - Messages auto-decided without human review
   - Balance efficiency with safety

4. **Trust-Damaging Incidents: 0 per month**
   - Harmful content that goes viral
   - Zero tolerance metric

5. **Workload Reduction: ≥50%**
   - Reduction in manual review volume
   - From ~360/day to <180/day

### Performance Targets

- API latency (p95): <2 seconds
- API latency (p99): <5 seconds
- LLM call time: <1.5 seconds average
- Throughput: ≥200 messages/minute
- System uptime: ≥99.9%

---

## 🚀 Implementation Roadmap

### Phase 1: Backend Core
1. Request validation logic
2. LLM integration (Anthropic API)
3. Two-confidence decision engine
4. In-memory database

### Phase 2: Backend Endpoints
1. POST /api/validateMessage
2. GET /api/getMessagesToReview
3. GET /api/getMessageDetails
4. PATCH /api/setUserDecision

### Phase 3: Frontend
1. Message list view
2. Message detail view with expand
3. Confirm/Decline buttons
4. Test message submission view

### Phase 4: Testing
1. Implement test framework
2. Run technical validation tests (9 tests)
3. Run business validation tests (20 tests)
4. Verify accuracy metrics

### Phase 5: Integration
1. External API integration (Message Board)
2. End-to-end testing
3. Performance testing
4. Security review

---

## 📝 Code Organization Suggestions

```
src/
├── api/
│   ├── __init__.py
│   ├── validate_message.py       # POST /api/validateMessage
│   ├── get_messages.py           # GET /api/getMessagesToReview
│   ├── get_details.py            # GET /api/getMessageDetails
│   └── set_decision.py           # PATCH /api/setUserDecision
├── core/
│   ├── __init__.py
│   ├── llm_client.py             # Anthropic API integration
│   ├── decision_engine.py        # Two-confidence logic
│   ├── database.py               # In-memory database
│   └── config.py                 # .env configuration loading
├── models/
│   ├── __init__.py
│   ├── message.py                # Message model
│   └── response.py               # API response models
├── ui/
│   ├── __init__.py
│   ├── main_window.py            # PyQT main window
│   ├── message_list.py           # Message list view
│   └── test_view.py              # Test message view
├── utils/
│   ├── __init__.py
│   ├── validation.py             # Request validation
│   └── logging.py                # Logging utilities
└── main.py                       # Application entry point

tests/
├── __init__.py
├── run_tests.py                  # Test framework
├── test_technical.py             # Technical tests
├── test_business.py              # Business tests
└── test_utils.py                 # Test utilities

.env                              # Configuration (not in git)
.gitignore                        # Git ignore patterns
requirements.txt                  # Python dependencies
```

---

## 🔒 Security & Privacy

### Configuration Security
- Store all secrets in `.env` file
- Add `.env` to `.gitignore`
- Never commit API keys or passwords

### Input Validation
- Validate all required fields present
- Check data types (id must be numeric)
- Enforce length limits (title ≤ 100, body elements ≤ 256)
- Sanitize JSON input

### Data Privacy
- Confidence scores are internal (don't send to Message Board)
- Only send verification result and reason externally
- Store messages temporarily (in-memory only for demo)

---

## 🤝 Working with Claude Code

### When Asking for Help

**Provide context:**
- Reference specification sections by line numbers
- Mention which component (Backend/Frontend/Database)
- Quote relevant test cases from TestData/

**Examples:**
- "Implement the validation logic from lines 380-383 in ProjectSpecification.md"
- "Create the decision engine based on the logic at lines 428-436"
- "Write the LLM prompt template from lines 525-598"

### Verification Requests

**Before implementation:**
- "Review the specification at lines X-Y for completeness"
- "Verify the delegation logic is correct"
- "Check if this matches the requirements in REQ-MD-01"

**During implementation:**
- "Does this match the API contract at lines 288-298?"
- "Verify this logic matches the two-confidence decision flow"
- "Check if this handles all test cases in technical_tests.json"

### Code Review Requests

**Focus areas:**
- "Review for delegation logic correctness"
- "Check if all confidence scores are stored correctly"
- "Verify error handling matches specification"
- "Ensure no security vulnerabilities (OWASP top 10)"

---

## 📚 Additional Resources

### Key Specification Sections

| Topic | Lines | Description |
|-------|-------|-------------|
| Two-confidence logic | 184-230 | Agent delegation rules |
| API validateMessage | 331-520 | Request/response/flow |
| LLM prompt template | 525-598 | Exact prompt to use |
| Database schema | 789-812 | Table structure |
| Frontend UI | 816-887 | UI requirements |
| Functional requirements | 960-1010 | REQ-MD-01 through REQ-MD-09 |
| Test approach | 1014-1104 | Testing strategy |

### Verification Reports
- `TWO_CONFIDENCE_PERFECT_REPORT.md` - Architecture verification (all issues resolved)
- `MARKDOWN_PERFECT_VERIFICATION.md` - Documentation quality (100/100 score)

---

## ⚠️ Common Pitfalls to Avoid

### 1. Delegation Logic Inversion ❌
```python
# WRONG - inverted logic
if high_confidence:
    delegation = true  # This means "needs human review" - WRONG!

# CORRECT
if high_confidence:
    delegation = false  # System auto-decides, no human review needed
```

### 2. Merging Confidence Scores ❌
```python
# WRONG - losing information
confidence = (compliance_confidence + violation_confidence) / 2

# CORRECT - keep both separate
response = {
    "compliance_confidence": 0.95,
    "violation_confidence": 0.05
}
```

### 3. Wrong Database Storage Logic ❌
```python
# WRONG - missing violations
if delegation == true:
    save_to_database()

# CORRECT - save violations AND grey zone
if verification == "negative" or delegation == true:
    save_to_database()
```

### 4. Threshold Comparison Errors ❌
```python
# WRONG - backwards thresholds
DELEGATION_THRESHOLD_BLOCK = 0.80  # Too low
DELEGATION_THRESHOLD_ALLOW = 0.95  # Too high

# CORRECT - from specification
DELEGATION_THRESHOLD_BLOCK = 0.95  # High bar for auto-blocking
DELEGATION_THRESHOLD_ALLOW = 0.80  # Lower bar for auto-allowing
```

---

## 📞 Support & Questions

### Specification Questions
- Check `Specification/ProjectSpecification.md` first (1109 lines, comprehensive)
- Line numbers are provided for easy reference
- All verification reports available in `Specification/` folder

### Implementation Questions
- Reference specific test cases from `TestData/`
- Quote specification sections with line numbers
- Mention which component (Backend/Frontend/Testing)

### Architecture Clarifications
- Two-confidence system is the CORE innovation
- Delegation semantics are critical (false = auto-decide, true = human review)
- Trust-first principle guides all decisions

---

## ✅ Pre-Implementation Checklist

Before starting implementation, verify:

- [ ] Read `Specification/ProjectSpecification.md` (main spec)
- [ ] Understand two-confidence architecture (lines 184-230)
- [ ] Review API contracts (lines 261-520)
- [ ] Check database schema (lines 789-812)
- [ ] Read policies.md (for LLM context)
- [ ] Review test data structure (TestData/README.md)
- [ ] Understand delegation logic (lines 428-436)
- [ ] Know success criteria (lines 252-324)
- [ ] Set up .env file with required config
- [ ] Review functional requirements (REQ-MD-01 through REQ-MD-09)

---

## 🎓 Learning Resources

### Understanding Two-Confidence Architecture
Read specification lines 184-230 for detailed explanation of:
- Why two scores instead of one
- How thresholds work
- Grey zone concept
- Trust-first principle

### LLM Integration Best Practices
- Temperature 0.3 for consistent classification
- Include recent examples for better accuracy
- Request structured JSON output
- Handle API failures gracefully

### Testing Strategy
- Technical tests verify API validation
- Business tests verify policy understanding
- Target 95% classification accuracy
- Focus on false negative rate (<1%)

---

**Last Updated:** 2026-04-24  
**Specification Version:** 1.0 (100% complete)  
**Status:** Ready for Implementation  
**Team:** Content Moderator Development Team

---

*This CLAUDE.md file is the guide for working with Claude Code on the Content Moderator project. It points to all key resources, explains the architecture, and provides implementation guidelines.*
