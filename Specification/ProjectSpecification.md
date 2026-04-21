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
Summary of the proposed approach and key ideas.

### 5.2 Architecture Overview
- Main components
- Data flows
- Key integrations
- Technology stack (if decided)

_(Optional: link to To‑Be architecture diagram)_

### 5.3 Key Design Decisions
- Decision 1 – rationale
- Decision 2 – trade-offs considered

### 5.4 Functional Requirements
- FR-1: …
- FR-2: …

### 5.5 Non-Functional Requirements
- Performance
- Scalability
- Security
- Availability
- Maintainability

---


## 8. Open Questions & Decisions Needed
- Question 1
- Question 2

---

