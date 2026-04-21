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
{
 "id": "1", 
 "title": "a new message", 
 "body": [
   "initial message", 
   "reply 1", 
   "reply 2"
 ]
}
all fields are required 
body represents the sequence in messages in chronological order from the first one to the following chain of replies 
id - numeric identifier of the message 
title - string max 100 characters long 
body - array of strings max 256 caracters long. should contain at least one entry 

if a request JSON is invalid or at least one of the fields is missing or is of incorrect value, respond HTTP 400 Bad request will be sent back 

Response: JSON
Example response - positive verification: 
{
 "verification": "positive", 
 "reason": "OK" 
}
Example response - negative verification: 
{
 "verification": "negative", 
 "reason": "OFF_TOPIC" 
}
verification - enum with possible values "positive" or "negative" 
reason - enum with information why verification is negative or value "OK" for positive verification  
possible values for reason enum: 
- HARASSMENT
- SPAM
- SELF_PROMOTING
- DOXXING
- IP_DISPUTES
- OFF_TOPIC

Empty JSON "{}" response in case of errors Bad request or Internal server error 

Endpoint validateMessage flow: 
- prepare a prompt using a predefined template based on: 
-- received message title 
-- received message body 
-- current user policy from the external file 
-- existing examples of user negatively validated messages from the Content Moderator database (10 top messages confirmed by a user as negative based on modification date, send examples if they are returned by the query, send no examples if query returns empty set). Condition in SQL query: WHERE user_decision='USER_VERIFIED_NEGATIVE' AND mdate IS NOT NULL ORDER BY mdate DESC LIMIT 10
- send a prompt to LLM 
- LLM returns a JSON that includes: validation, reason, reasoning
- if a message is complient, endpoint returns positive verification to requestor 
- if a message is not complient, endpoint: 
-- returns negative verification to requestor
-- add a record to Content Moderator database for further processing  
- if LLM returns a failure then 500 Internal server error should be returned
- if the database returns an error then 500 Internal server error should be returned

Initial prompt: 
As a member of moderation team verify the message for complience with the policy, especially under the following criterias: harassment, off-topic content, commercial spam, self-promotion limits, doxxing, and IP disputes over custom sculpts
<MESSAGE>
{
 "title": "a new message", 
 "body": [
   "initial message", 
   "reply 1", 
   "reply 2"
 ]
}
</MESSAGE>
<POLICY>
  content of the Specification/policies.md 
</POLICY>
example messages with negative validation
<EXAMPLE>
Message 1: 
{
 "title": "a new message", 
 "body": [
   "initial message", 
   "reply 1", 
   "reply 2"
 ]
}
Response: 
{
 "validation": "negative", 
 "reason": "HARASSMENT", 
 "reasoning": "user is harassing others" 
}
</EXAMPLE>
Result provide in JSON format: 
{
 "validation": "negative", 
 "reason": "HARASSMENT", 
 "reasoning": "user is harassing others" 
}
where reason is one of (the same as in the endpoint response): 
- HARASSMENT
- SPAM
- SELF_PROMOTING
- DOXXING
- IP_DISPUTES
- OFF_TOPIC

Example LLM JSON response
{
 "validation": "negative", 
 "reason": "HARASSMENT", 
 "reasoning": "user is harassing others" 
}
where: 
- validation is enum - positive or negative 
- reason: enum with possible statues of validation or OK if validation is positive
- reasoning: description while this reason is chosen 


### Content Moderator Database 
database structure
Table Messages:
- id: integer - primary key  
- external_id: integer not null - external id of a Message
- title: char[100] not null - message title 
- body: char[255] not null - message body - serialized to JSON list of messages 
- reason: char[20] not null - message validation status 
- reasoning: char[1024] not null - reasoning of the LLM 
- user_decision: char[20] default null - decision of a user - one of USER_VERIFIED_POSITIVE, USER_VERIFIED_NEGATIVE
- cdate: datetime not null default sysdate - timestamp of the entry creation. Set up with a current data/time when a record is created 
- mdate: datetime - timestamp of user's decision - when the record is modified. Set up with a current data/time when a record is modified  


### Technology stack 
Programming language: Python 
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
Example of .env file:
ANTHROPIC_API_KEY=sk-...
 LLM_TIMEOUT=30
 LLM_MAX_TOKENS=5000

- no data persistence for this version of implementation
- in memory database approach: Single dict with ID keys {1: {record}, 2: {record}} 

- industry standard 200/400/500 status codes will be used for REST endpoints responses 

- policies.md file should be loaded from Specification/policies.md during application start 

### 5.4 Functional Requirements
- FR-1: User message verification 
As a user I want to validate a message while posting so that my messages are complient with the portal policy 

Acceptance criteria: 
AC1: postive scenario 
GIVEN that a message is complient with the policies THAN endpoint is called THEN positive response is sent 
AC2: negative scenario 
GIVEN that a message is not complient with the policies THAN endpoint is called THEN negative response is sent AND the message is stored in the database


### 5.5 Non-Functional Requirements


---


## 8. Open Questions & Decisions Needed
- 

---

