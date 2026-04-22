# Project Specification

## 1. Overview
### 1.1 Purpose
The goal of the project Content : 
- automatically handle verifycation of messages posted by a user for compliance with policies 
- allowing user to handle manually reported messages

### 1.2 Goals & Success Criteria
- Introducing new component Content Moderator that will be integrated into existing solution (Message board) 
- 95% of messages are correctly classified as complient/non-complient with policies by Content Moderator component 
- user has ability to manually verify and set up complience of messages in the Content Moderator component
- solution passes prepared tests 

### 1.3 Scope
**In scope**
- Backend component to handle automatic verification of messages 
- Frontend component to allow user view and verify messages marked for verification 

**Out of scope**
- Changes in the component to post messages (Message Board)

---

## 2. Assumptions & Constraints

### 2.1 Assumptions
- Existing component Massage Board allows users to create new messages and reply to existing messages 
- Message Board component will use a new REST services to verify if a message is positively verified 
- Message Board component will expose a REST service to mark a message as verified by a human positively or negatively 
- components Message Board and Content Moderator communicate only with REST services 

### 2.2 Constraints
- 

---

## 3. Current Solution (As-Is)

### 3.1 Description
Message Board component allows user to create a message or reply to an existing message 
A message is store in the local database 
A message has the following attributes: 
- message_id - unique identified of a message 
- reply_to_id - identifier of an existing message. Can be null if the message is the new/not a reply 
- title - title of e message. Null if a message is a reply to the existing one (reply_to_id is not null) 
- body - the body of the message 
- user_id - identifier of the user that created the message 
- status - status of the message 

Current message statuses: 
- new 
- user_reported
- user_verified_positive 
- user_verified_negative 

Message Board display messages with statuses: 
- new
- user_reported
- user_verified_positive

A user can report a message in the status new 


### 3.2 Architecture Overview
- Component Message Board allows user to create a new message 
- Component Message Board allows user to reply to an existing message 
- No integrations with external components 


### 3.3 Strengths
- Simple stegihtforward architecture

### 3.4 Limitations / Pain Points
- Message board has about 180K active users
- Users of the Message Board generate about 12K posts per day across forums and galleries
- Some of the posted messages are not complient with current policy 
- Some of the posted messages contain harassment, off-topic content, commercial spam, self-promotion limits, doxxing, and IP disputes over custom sculpts
- 97% of all messages are complient
- 3% of messages potentially contain  harassment, off-topic content, commercial spam, self-promotion limits, doxxing, and IP disputes over custom sculpts and should be verified by the moderation team (8 volunteer moderators, 2 paid staff)

---

## 4. Problem Statement
Growing number of users and messages generate a lot of manual work for moderation team 
A solution is needed that will handle  messages complience, flag non-complient for human review with real context

---

## 5. Proposed Solution (To-Be)

### 5.1 High-Level Description
New component Content Moderator will be added to the existing infrastructure 
Content Moderator will include sub-components: 
- Content Moderator Backend that exposes REST endpoint to verify messages complience 
- Content Moderator Frontend that will allow users to manually verify messages that are marked as non complient 
- Complience of the messages will be verified by an LLM 
- Solution will have a database to handle non-complient messages 


### 5.2 Architecture Overview
Components of the solution: 
- Content Moderator Backend 
- Content Moderator Froentend 

### Content Moderator Backend 
Accepts REST requests and responses with information if a message is complient with policies 

Endpoint name: /api/validateMessage
Host: localhost 
Port: 8000
Type: REST
Method: POST 
Request: JSON
Example request: 
```json
{
 "id": "1", 
 "title": "a new message", 
 "body": [
   "initial message", 
   "reply 1", 
   "reply 2"
 ]
}
```

all fields are required 
body represents the sequence in messages in chronological order from the first one to the following chain of replies 
id - numeric identifier of the message 
title - string max 100 characters long 
body - array of strings max 256 caracters long. should contain at least one entry 

if a request JSON is invalid or at least one of the fields is missing or is of incorrect value, respond HTTP 400 Bad request will be sent back 

**Success Response (HTTP 200):**

```json
{
  "verification": "positive" | "negative",
  "reason": string,
  "confidence": boolean,
  "reasoning": string,
  "confidence_score": float
}
```

**Field Definitions:**

- **verification** (string, required): enum with possible values
  - `"positive"` - Message complies with policies
  - `"negative"` - Message violates policies or requires manual review

- **reason** (string, required): Classification reason
  - For positive verification: `"OK"`
  - For negative verification: One of the violation types
    - `"HARASSMENT"`
    - `"SPAM"`
    - `"SELF_PROMOTING"`
    - `"DOXXING"`
    - `"IP_DISPUTES"`
    - `"OFF_TOPIC"`

- **confidence** (boolean, required): Indicates LLM confidence level
  - `true` - LLM confidence >= CONFIDENCE_THRESHOLD (high confidence)
  - `false` - LLM confidence < CONFIDENCE_THRESHOLD (low confidence, requires manual review)

**Error Response:**
```json
{}
```
Empty JSON for HTTP 400 Bad Request or HTTP 500 Internal Server Error

## REVISED: Response Examples

### Example 1: High Confidence - Compliant Message
```json
{
  "verification": "positive",
  "reason": "OK",
  "confidence": true
}
```
**Interpretation:** Message is compliant and LLM is confident (confidence >= 0.90). No manual review needed.

---

### Example 2: Low Confidence - Possibly Compliant Message
```json
{
  "verification": "positive",
  "reason": "OK",
  "confidence": false
}
```
**Interpretation:** Message appears compliant but LLM has low confidence (confidence < 0.90). Saved to database for manual review to confirm.

---

### Example 3: High Confidence - Clear Policy Violation
```json
{
  "verification": "negative",
  "reason": "HARASSMENT",
  "confidence": true
}
```
**Interpretation:** Message clearly violates policy (harassment) and LLM is confident (confidence >= 0.90). Saved to database for moderation action.

---

### Example 4: Low Confidence - Uncertain Violation
```json
{
  "verification": "negative",
  "reason": "OFF_TOPIC",
  "confidence": false
}
```
**Interpretation:** Message may violate policy (off-topic) but LLM has low confidence (confidence < 0.90). Saved to database for manual review to confirm.

---


## REVISED: Endpoint validateMessage Flow

1. **Validate Request**
   - Check all required fields present (id, title, body)
   - Validate field types and constraints
   - Return HTTP 400 if validation fails

2. **Prepare LLM Prompt**
   - Load policy content from `Specification/policies.md`
   - Query database for examples of validated violations:
     ```sql
     WHERE user_decision='USER_VERIFIED_NEGATIVE' 
     AND mdate IS NOT NULL 
     ORDER BY mdate DESC 
     LIMIT 10
     ```
   - Build prompt using template (see below)

3. **Call LLM**
   - Send prompt to Claude Sonnet API
   - Receive JSON response with: `validation`, `reason`, `reasoning`, `confidence`
   - If LLM call fails → Return HTTP 500

4. **Process LLM Response**
   
   Parse LLM response fields:
   - `validation`: "positive" or "negative" (string)
   - `reason`: Violation type or "OK" (string)
   - `reasoning`: Explanation text (string)
   - `confidence`: Confidence score 0.0-1.0 (float)

   Load threshold from configuration:
   - `CONFIDENCE_THRESHOLD`: Float value (e.g., 0.90)

   Determine high/low confidence:
   ```python
   is_high_confidence = (llm_confidence >= CONFIDENCE_THRESHOLD)
   ```

5. **Decision Logic**

   **Case A: Positive validation + High confidence**
   - LLM says: compliant, confidence >= threshold
   - Action: Return positive response, DO NOT save to database
   - Response:
     ```json
     {
       "verification": "positive",
       "reason": "OK",
       "confidence": true
     }
     ```

   **Case B: Positive validation + Low confidence**
   - LLM says: compliant, confidence < threshold
   - Action: Return positive response, SAVE to database for manual review
   - Response:
     ```json
     {
       "verification": "positive",
       "reason": "OK",
       "confidence": false
     }
     ```
   - Database record: Save with validation="positive", confidence score, reasoning

   **Case C: Negative validation + High confidence**
   - LLM says: non-compliant, confidence >= threshold
   - Action: Return negative response, SAVE to database for moderation
   - Response:
     ```json
     {
       "verification": "negative",
       "reason": "<VIOLATION_TYPE>",
       "confidence": true
     }
     ```
   - Database record: Save with validation="negative", confidence score, reasoning

   **Case D: Negative validation + Low confidence**
   - LLM says: non-compliant, confidence < threshold
   - Action: Return negative response, SAVE to database for manual review
   - Response:
     ```json
     {
       "verification": "negative",
       "reason": "<VIOLATION_TYPE>",
       "confidence": false
     }
     ```
   - Database record: Save with validation="negative", confidence score, reasoning

6. **Database Operations**

   Save to database if ANY of these conditions:
   - `validation == "negative"` (regardless of confidence), OR
   - `confidence < CONFIDENCE_THRESHOLD` (regardless of validation)

   Database record includes:
   ```python
   {
     "external_id": request.id,
     "title": request.title,
     "body": json.dumps(request.body),
     "reason": llm_response.reason,
     "reasoning": llm_response.reasoning,
     "confidence": llm_response.confidence,  # Store original float value
     "user_decision": None,
     "cdate": datetime.now(),
     "mdate": None
   }
   ```

   If database write fails → Return HTTP 500

7. **Return Response**
   - HTTP 200 with JSON response
   - `verification`: LLM's validation value
   - `reason`: LLM's reason value
   - `confidence`: Boolean (true if >= threshold, false otherwise)

---

## REVISED: Implementation Decision Tree

```
┌─────────────────────────────────────────────────────┐
│          Receive Validation Request                 │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│     Validate Request Format (400 if invalid)        │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│   Build Prompt + Call LLM (500 if failure)          │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│        Parse LLM Response JSON                      │
│   validation, reason, reasoning, confidence         │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
         ┌───────────────┐
         │ validation =? │
         └───────┬───────┘
                 │
      ┌──────────┴──────────┐
      │                     │
   "positive"          "negative"
      │                     │
      ▼                     ▼
┌──────────────┐      ┌──────────────┐
│ confidence   │      │ confidence   │
│     ≥        │      │     ≥        │
│ threshold?   │      │ threshold?   │
└──┬────────┬──┘      └──┬────────┬──┘
   │        │            │        │
  YES       NO          YES       NO
   │        │            │        │
   ▼        ▼            ▼        ▼
┌─────┐  ┌─────┐      ┌─────┐  ┌─────┐
│Case │  │Case │      │Case │  │Case │
│  A  │  │  B  │      │  C  │  │  D  │
└──┬──┘  └──┬──┘      └──┬──┘  └──┬──┘
   │        │            │        │
   │        │            │        │
   ▼        ▼            ▼        ▼
Return   Return       Return   Return
positive positive     negative negative
conf=T   conf=F       conf=T   conf=F
NO DB    SAVE DB      SAVE DB  SAVE DB
```

**Legend:**
- Case A: Clear compliance → Allow, no review
- Case B: Uncertain compliance → Allow, flag for review
- Case C: Clear violation → Block, send to moderation
- Case D: Uncertain violation → Block, send to review

---

## LLM Prompt Template
```
As a member of the moderation team, verify the following message for compliance with community policies.

Focus on these violation categories:
- Harassment: bullying, insults, threats, unwanted contact
- Off-topic content: political/religious debates unrelated to hobby
- Commercial spam: unsolicited advertising, affiliate links
- Self-promotion limits: cold-messaging for commissions, portfolio spam
- Doxxing: sharing personal information without consent
- IP disputes: IP infringement accusations as harassment

<MESSAGE>
{
  "title": "{{message_title}}",
  "body": {{message_body_json_array}}
}
</MESSAGE>

<POLICY>
{{content_of_policies_md_file}}
</POLICY>

{{if examples exist}}
Examples of previously confirmed policy violations:

<EXAMPLES>
{{for each example}}
Example {{N}}:
Message:
{
  "title": "{{example_title}}",
  "body": {{example_body_json_array}}
}
Violation:
{
  "validation": "negative",
  "reason": "{{example_reason}}",
  "reasoning": "{{example_reasoning}}"
}

{{end for}}
</EXAMPLES>
{{end if}}

Provide your response in JSON format with the following structure:
{
  "validation": "positive" or "negative",
  "reason": "OK" | "HARASSMENT" | "SPAM" | "SELF_PROMOTING" | "DOXXING" | "IP_DISPUTES" | "OFF_TOPIC",
  "reasoning": "Detailed explanation of your decision (1-2 sentences)",
  "confidence": <float 0.0 to 1.0>
}

Confidence scoring guidelines:
- 1.0: Absolutely certain of the classification (clear violation or obviously compliant)
- 0.9: Very confident (strong evidence for classification)
- 0.7: Somewhat confident (leans one way but not definitive)
- 0.5: Uncertain (borderline case, could go either way)
- 0.3 or lower: Cannot determine with reasonable certainty

Important:
- If validation is "positive", set reason to "OK"
- If validation is "negative", set reason to the specific violation type
- Be conservative: if unsure whether content violates policy, use lower confidence
```

---

## Example LLM Responses

Simple LLM JSON response
```json
{
  "validation": "negative", 
  "reason": "HARASSMENT", 
  "reasoning": "user is harassing others", 
  "confidence": 0.50 
}
```
where: 
- validation is enum - positive or negative 
- reason: enum with possible statues of validation or OK if validation is positive
- reasoning: description while this reason is chosen, 
- confidence: the level of LLM confidence in provided response 

### Example 1: Clear Compliance
```json
{
  "validation": "positive",
  "reason": "OK",
  "reasoning": "Message is a constructive hobby discussion about painting techniques with no policy violations.",
  "confidence": 0.95
}
```

### Example 2: Uncertain Compliance
```json
{
  "validation": "positive",
  "reason": "OK",
  "reasoning": "Message appears compliant but contains ambiguous language that could be interpreted as subtle self-promotion.",
  "confidence": 0.65
}
```

### Example 3: Clear Violation
```json
{
  "validation": "negative",
  "reason": "HARASSMENT",
  "reasoning": "Message contains direct personal insults and bullying language directed at another user.",
  "confidence": 0.98
}
```

### Example 4: Uncertain Violation
```json
{
  "validation": "negative",
  "reason": "OFF_TOPIC",
  "reasoning": "Message discusses political topics but tangentially relates to hobby community. Borderline off-topic.",
  "confidence": 0.55
}
```

---

### Backend services to support Frontend 

### Endpoint getMessagesToReview 
GET /api/getMessagesToReview

Endpoint returns messages that required human review 

REST method GET 

Request: no parameters 

Response: 
```json
  {
    "messages": [{
      "id": 123,
      "title": "message title",
      "reason": "OFF_TOPIC",
      "confidence": 0.55
    }]
  }
```

Query to fetch messages from the database: 
```sql
SELECT * FROM messages 
WHERE user_decision IS NULL 
  AND confidence < CONFIDENCE_THRESHOLD
ORDER BY cdate DESC
LIMIT 5000
```

In case of database error, service returns response 500 - Internal server error 


### Endpoint getMessageDetails 
Basic path GET /api/getMessageDetails

Endpoint to get message details for human review - returns full information on a given message  

REST method GET 
Path with query parameter example GET /api/getMessageDetails?id={id}

Query parameter: id - id of the message in the database 

Response:  
```json
{
  "message": {
    "title": "message title", 
	"body": [
	  "Initial message", 
	  "reply"
	], 
    "reason": "OFF_TOPIC",
    "reasoning": "Message discusses political topics but tangentially relates to hobby community. Borderline off-topic.",
    "confidence": 0.55
  }
}
```

Query to fetch message details from the database: 
```sql
SELECT * FROM messages WHERE id = {id_parameter}
```

In case of database error, service returns response 500 - Internal server error 


### Endpoint setUserDecision  
Endpoint updates the given message with a user decision  

REST method PATCH  

Endpoint PATCH /api/setUserDecision?id={id}&user_decision={decision}

Query parameters: 
 - id - id of the message in the database 
 - user_decision - decision made by a user. Can be one from USER_VERIFIED_POSITIVE, USER_VERIFIED_NEGATIVE

Response: empty JSON 
```json
{
}
```

Endpoint does the following actions: 

Update the message with id provided in the query parameter in the database 
set the following fields: 
-- set mdate to current date and time 
-- set user_decision to USER_VERIFIED_POSITIVE or USER_VERIFIED_NEGATIVE


-- informs the external system about user decision 
To inform the external Message Board system, a REST service should be called: 
REST method: PATCH 
REST endpoint: MESSAGE_BOARD_ENDPOINT 

Request body for external endpoint  
```json
{
  "id": "1",
  "reason": "User_verified_positive",
  "reasoning": "Positive human verification",
  "confidence": 0.7
}
```
Where id is external_id from the database 


In case of missing parameters or parameters are not matching pattern, service returns response 400 - Bad parameters

In case of database error, service returns response 500 - Internal server error 

In case of external endpoint error, service returns 500 - Internal server error 

---

### Content Moderator Database 

## Database Schema

### Table: Messages

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | integer | PRIMARY KEY | Auto-increment record ID |
| external_id | integer | NOT NULL | External message ID from Message Board |
| title | char[100] | NOT NULL | Message title |
| body | char[1000] | NOT NULL | Message body (JSON serialized array) |
| reason | char[20] | NOT NULL | Validation reason (OK, HARASSMENT, etc.) |
| reasoning | char[1024] | NOT NULL | LLM's explanation |
| confidence | float | NOT NULL | LLM confidence score (0.0-1.0) |
| user_decision | char[20] | NULL | Manual review decision (USER_VERIFIED_POSITIVE, USER_VERIFIED_NEGATIVE) |
| cdate | datetime | NOT NULL, DEFAULT NOW | Record creation timestamp |
| mdate | datetime | NULL | Last modification timestamp (set when user_decision changes) |

**Storage Logic:**
- Store records when: validation="negative" OR confidence < threshold
- `confidence` field stores the original float value from LLM (e.g., 0.65)
- This allows frontend to display actual confidence levels to moderators

---


### Content Moderator UI 

Content Moderator UI allows user (a member of moderation team) to review and manually verify messages that did not pass AI verification 

Content Moderator UI has one window divided into the following areas: 
- List of messages to be reviewed by a human 
- View to create messages 

Users who can run application will be able to view and change messages. No additional authorization is required. 


### List of Messages 
List of messages view displayed all the messages that required user's attention and verification 
The view contains: 
- message title (field title from the database) 
- Classification reason provided by LLM (field reason from the database)
- Confidence given by LLM (field confidence from the database)

Messages that require human verification are retrieved with the endpoint provided by Content Moderator backend: GET /api/getMessagesToReview

When a message is clicked, the message expands and the following information, in addition to information that is already rendered, is displayed: 
- message body: list of messages in array from database field body displayed in the list (database field body)
- LLM reasoning (database field reasoning)
- Button "Confirm" 
- Button "Decline" 
Message details are retrieved with the endpoint provided by Content Moderator backend: GET /api/getMessageDetails?id={id}
Query parameter to pass to the endpoint - id of the message which user clicked 

Button "Confirm" action:
- Calls: PATCH /api/setUserDecision?id={message_id}&user_decision=USER_VERIFIED_POSITIVE
- On success: Message is removed from the list

Button "Decline" action:
- Calls: PATCH /api/setUserDecision?id={message_id}&user_decision=USER_VERIFIED_NEGATIVE
- On success: Message is removed from the list

If an error occurs during service calls, user received information about HTTP error code: "Error occurred. Code: 500" 


## View to Create Messages 
Functionality is required for test purposes 
Possibility to simulate sending a message for verification by providing: 
- message title - text input with limit characters 100
- message body - text input with limit characters 256 
- button "Send" - active when both fields are provided 

Pressing send button do the following: 
- prepares a request to Content Moderator Backend 
- confirm the user that message is sent 

Request body for request: 
```json
{
 "id": "1", 
 "title": "message title", 
 "body": [
   "message body"
 ]
}
```
where id is constant 1 
title - title provided by the user 
message body - message provided by the user  

When service call finishes with success (HTTP code 200), user received the following information: 
- text "Message is sent successfully" 
- text "LLM classification: reason", where reason is field reason from the response
- text "LLM confidence: High" (if confidence=true) or "Low" (if confidence=false)
- text "LLM confidence: confidence_score", where confidence_score value is field confidence_score from the response

In case of an error, the user receives a message: "Message was not set. Error: 400" where error value is HTTP error code  


### Technology stack 
Programming language: Python 
UI Library: PyQT
Database: in memory database: Python dict
Interfaces: REST 
LLM provider: API endpoint  
LLM model: Claude Sonnet 
LLM authentification: API keys
LLM configuration: 
- Timeout: 30 seconds 
- no retries on errors 
- number of tokens: 5000 

### 5.3 Key Design Decisions
- API keys, passwords and configuration parameters are stored in .env file and not explicitly used in the code 

- no data persistence for this version of implementation
- in memory database approach: Single dict with ID keys {1: {record}, 2: {record}} 

- industry standard 200/400/500 status codes will be used for REST endpoints responses 

- policies.md file should be loaded from Specification/policies.md during application start 

## Configuration (.env)

```ini
# Anthropic API Configuration
ANTHROPIC_API_KEY=sk-ant-...

# LLM Settings
LLM_TIMEOUT=30
LLM_MAX_TOKENS=5000

# Confidence Threshold (0.0 to 1.0)
# Messages with confidence below this threshold require manual review
CONFIDENCE_THRESHOLD=0.90

# Server Configuration
HOST=localhost
PORT=8000

# Message Board Server 
MESSAGE_BOARD_ENDPOINT=localhost:9000/api/updateMessage 

```

**Configuration Notes:**
- `CONFIDENCE_THRESHOLD`: Float between 0.0 and 1.0
  - Recommended: 0.85-0.95 for balanced automatic/manual review
  - Higher values (0.95+): More conservative, more manual reviews
  - Lower values (0.80): More aggressive automation, fewer manual reviews

---



### 5.4 Functional Requirements

### FR-1: User message verification

**As a user** I want to validate a message while posting so that my messages are compliant with portal policy

**Acceptance Criteria:**

**AC1: High confidence compliant message**
- GIVEN that a message is compliant with policies
- AND LLM confidence >= CONFIDENCE_THRESHOLD
- WHEN the endpoint is called
- THEN a positive response is sent with confidence=true
- AND the message is NOT stored in the database

**AC2: Low confidence compliant message**
- GIVEN that a message appears compliant with policies
- AND LLM confidence < CONFIDENCE_THRESHOLD
- WHEN the endpoint is called
- THEN a positive response is sent with confidence=false
- AND the message IS stored in the database for manual review

**AC3: High confidence policy violation**
- GIVEN that a message violates policies
- AND LLM confidence >= CONFIDENCE_THRESHOLD
- WHEN the endpoint is called
- THEN a negative response is sent with confidence=true and appropriate reason
- AND the message IS stored in the database

**AC4: Low confidence policy violation**
- GIVEN that a message may violate policies
- AND LLM confidence < CONFIDENCE_THRESHOLD
- WHEN the endpoint is called
- THEN a negative response is sent with confidence=false and appropriate reason
- AND the message IS stored in the database for manual review

---


### 5.5 Non-Functional Requirements

---

### 5.6 Testing approach 

### Content Moderator Backend 

- Technical verification 
Generate messages with incorrect request: 
- missing fields: 
-- id 
-- title 
-- body 
- non correct values for fields: 
-- id is not numeric 
-- title is longer than 100 characters 
-- body array is empty 
-- one message in body array has more than 256 characters 
-- total number of body serialized ot JSON is more than 10000 characters long 

- Business verification
Generate 10 messages based on policies file that produce positive feedback 

Generate 10 messages based on policies file that produce negative feedback 

Proposed structure for verification: 
{
 "test-cases": [
   {
    "id": "test-case-001", 
	"goal": "verify content moderation - off topic", 
	"message": "message serialized to json including id, title and body", 
    "expected-result": "negative" 	
   }
 ]
 }

Create a framework to run test cases and verify the result  

### Additional Business Test Cases for Confidence Levels

```json
{
  "test-cases": [
    {
      "id": "biz-conf-001",
      "goal": "Low confidence positive - ambiguous self-promotion",
      "message": {
        "id": "3001",
        "title": "Just sharing my latest work",
        "body": ["Here's a mini I painted recently", "Let me know if you want tips"]
      },
      "expected-result": "positive",
      "expected-confidence": "false",
      "note": "Borderline self-promotion, should trigger manual review"
    },
    {
      "id": "biz-conf-002",
      "goal": "Low confidence negative - mild criticism",
      "message": {
        "id": "3002",
        "title": "That painting could be better",
        "body": ["Your technique needs work", "Consider practicing more"]
      },
      "expected-result": "negative",
      "expected-confidence": "false",
      "note": "Could be constructive feedback or mild harassment"
    },
    {
      "id": "biz-conf-003",
      "goal": "High confidence positive - clear hobby discussion",
      "message": {
        "id": "3003",
        "title": "Tips for dry brushing",
        "body": ["Use a flat brush with very little paint", "Build up layers slowly"]
      },
      "expected-result": "positive",
      "expected-confidence": "true",
      "note": "Clearly compliant hobby content"
    },
    {
      "id": "biz-conf-004",
      "goal": "High confidence negative - obvious spam",
      "message": {
        "id": "3004",
        "title": "BUY NOW - 50% OFF ALL MINIATURES!!!",
        "body": ["Visit www.example.com", "Limited time only", "Use code SAVE50"]
      },
      "expected-result": "negative",
      "expected-confidence": "true",
      "note": "Obviously commercial spam"
    }
  ]
}
```

---



## 8. Open Questions & Decisions Needed
- 

---

