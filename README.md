GameHull mein live chat integrate karne ke liye aapko existing contact form ko **two-way support conversation system** mein convert karna hoga.

Main aapke existing **PHP/MySQL admin architecture** ke according yeh structure recommend karta hoon.

## Required Workflow

```text
User starts chat
→ Chat goes to Unassigned Queue
→ Support Agent A claims it
→ Only Agent A can reply
→ Agent A resolves or transfers it
→ After transfer, only Agent B can reply
```

Sirf frontend button disable karna enough nahi hai. Backend aur database level par bhi agent ownership enforce karni hogi.

---

# 1. Database Tables

Minimum three new tables banayein.

## `support_conversations`

Har support chat ka main record:

```text
id
user_id
subject
category
priority
status
assigned_agent_id
claimed_at
last_message_at
resolved_at
closed_at
created_at
updated_at
```

Recommended statuses:

```text
open
claimed
waiting_for_user
waiting_for_support
resolved
closed
```

Meaning:

* `open`: kisi agent ne claim nahi ki
* `claimed`: agent assigned hai
* `waiting_for_user`: support ne reply kar diya
* `waiting_for_support`: user ne reply kiya
* `resolved`: issue solve ho gaya
* `closed`: conversation permanently close

---

## `support_messages`

Conversation ke tamam user aur admin messages:

```text
id
conversation_id
sender_type
sender_user_id
sender_agent_id
message
attachment_path
is_internal_note
read_at
created_at
```

`sender_type` values:

```text
user
agent
system
```

Internal notes sirf admin/support ko show hon:

```text
is_internal_note = 1
```

User ko internal notes kabhi expose na karein.

---

## `support_chat_transfers`

Agent-transfer history:

```text
id
conversation_id
from_agent_id
to_agent_id
transferred_by
transfer_reason
created_at
```

Is se audit trail rahega ke chat kis agent se kis agent ko transfer hui.

---

# 2. User Chat Interface

User side par in mein se koi interface create karein:

```text
/support
```

ya floating chat button:

```text
Help & Support
```

User interface mein:

* Start New Conversation
* Subject
* Category
* Message
* Attachment
* Conversation history
* Agent reply
* Unread message count
* Close conversation

Categories example:

```text
Game Account Issue
Deposit Issue
Withdrawal Issue
Affiliate Issue
Password Reset
Technical Problem
Other
```

User sirf apni conversation access kar sake:

```sql
WHERE support_conversations.user_id = logged_in_user_id
```

---

# 3. Admin Support Panel

Admin route:

```text
/admin/support/chats
```

Recommended tabs:

```text
Unassigned
My Chats
Waiting for Support
Waiting for User
Resolved
Closed
```

Each chat row mein show karein:

```text
Chat ID
User
Subject
Category
Priority
Status
Assigned Agent
Last Message
Last Activity
```

Admin actions:

```text
Claim Chat
Reply
Add Internal Note
Transfer Chat
Resolve Chat
Close Chat
Reopen Chat
```

---

# 4. Claim Chat Logic

Yeh sabse important part hai.

Jab Agent A **Claim Chat** click kare, backend ko atomic update karni chahiye:

```sql
UPDATE support_conversations
SET
    assigned_agent_id = :agent_id,
    status = 'claimed',
    claimed_at = NOW()
WHERE id = :conversation_id
  AND assigned_agent_id IS NULL
  AND status = 'open';
```

Phir affected rows check karein:

```text
1 row updated → Agent A successfully claimed
0 rows updated → Another agent already claimed it
```

Agent B ko message show karein:

```text
This chat has already been claimed by another support agent.
```

Is atomic update ki wajah se do agents same time par chat claim nahi kar sakenge.

---

# 5. Agent Reply Permission

Admin reply endpoint par check karein:

```text
Is logged-in admin the assigned agent?
```

Backend rule:

```php
$conversation->assigned_agent_id === $loggedInAgentId
```

Agar match nahi karta:

```text
403 Forbidden
This conversation is assigned to another support agent.
```

Super Admin ke liye aap optionally override permission de sakte hain, lekin normal support agents sirf assigned chats mein reply karein.

---

# 6. Chat Transfer Logic

Route:

```text
POST /admin/support/chats/{id}/transfer
```

Required fields:

```text
to_agent_id
transfer_reason
```

Backend process:

1. Database transaction start karein.
2. Conversation row lock karein.
3. Confirm logged-in agent current assigned agent hai.
4. New agent active support member hai.
5. Transfer-history record create karein.
6. `assigned_agent_id` new agent se update karein.
7. System message add karein.
8. Transaction commit karein.

Example system message:

```text
Chat transferred from Agent A to Agent B.
```

Transfer complete hone ke baad Agent A reply nahi kar sakega. Sirf Agent B reply karega.

---

# 7. Required API Routes

## User routes

```text
POST /support/conversations
GET  /support/conversations
GET  /support/conversations/{id}
POST /support/conversations/{id}/messages
POST /support/conversations/{id}/close
```

## Admin routes

```text
GET  /admin/support/chats
GET  /admin/support/chats/{id}
POST /admin/support/chats/{id}/claim
POST /admin/support/chats/{id}/messages
POST /admin/support/chats/{id}/transfer
POST /admin/support/chats/{id}/resolve
POST /admin/support/chats/{id}/close
POST /admin/support/chats/{id}/reopen
```

## Notification route

```text
GET /support/unread-count
```

---

# 8. Real-Time Messages

Aapke paas do implementation options hain.

## Option A — AJAX Polling

Frontend har 3–5 seconds baad new messages check kare:

```text
GET /support/conversations/{id}/messages?after_id=123
```

Yeh existing PHP/MySQL project aur normal cPanel environment ke liye simplest option hai.

Advantages:

* Easy implementation
* Separate WebSocket server nahi chahiye
* Existing PHP hosting par kaam karega

Disadvantage:

* True instant chat nahi
* Repeated server requests hongi

## Option B — WebSockets

Real-time production chat ke liye private WebSocket channel use karein:

```text
private-support-chat.{conversationId}
```

Laravel currently Reverb, Pusher Channels aur Ably broadcasting drivers support karta hai. Private channels server-side authorization require karte hain, jo ensure karta hai ke sirf allowed user ya assigned support agent conversation subscribe kar sake. ([Laravel][1])

Recommended authorization:

```text
Allow when:
logged-in user owns conversation

OR

logged-in admin is assigned agent

OR

logged-in admin has super-admin permission
```

### GameHull ke liye recommendation

Agar GameHull normal PHP/custom framework aur cPanel par chal raha hai:

```text
Phase 1: AJAX polling
Phase 2: WebSocket real-time upgrade
```

Agar Laravel aur VPS/server access available hai, Laravel Reverb ya private Pusher/Ably channels use kiye ja sakte hain. Realtime presence functionality agent online/offline state show karne ke liye bhi use ho sakti hai. ([Ably Realtime][2])

---

# 9. Existing Contact Form Ka Kya Karna Hai?

Current contact form ko remove karna zaroori nahi.

Contact form submission ke baad:

```text
Create support_conversation
Create first support_message
Show conversation inside user support page
Show it inside admin Unassigned queue
```

Ab admin sirf message dekhega nahi, balki chat claim karke reply bhi kar sakega.

User ko notification mile:

```text
Support has replied to your conversation.
```

---

# 10. Notifications

User notifications:

* Support agent replied
* Chat transferred
* Issue resolved
* Chat closed

Agent notifications:

* New unassigned chat
* Chat transferred to agent
* User replied
* High-priority issue created

Notification channels:

```text
In-app notification
Unread badge
Email notification
```

Email mein sensitive chat content bhejne ke bajaye yeh message bhejna better hai:

```text
You have a new support response. Log in to view it.
```

---

# 11. Security Requirements

Har endpoint par:

```text
Authentication
CSRF protection
Conversation ownership
Agent assignment check
Input validation
Output escaping
Rate limiting
```

Important rules:

* User doosre user ki chat access na kar sake.
* Agent unassigned ya another-agent chat mein reply na kar sake.
* Chat ID modify karke authorization bypass na ho.
* Messages HTML-escaped hon.
* Passwords aur provider credentials chat mein automatically expose na hon.
* Attachments ka MIME type aur size validate ho.
* Executable files upload allow na hon.
* Every claim, transfer, resolve aur close action audit log mein ho.

---

# 12. UI States

## User side

```text
Support is reviewing your issue.
Waiting for your response.
Issue resolved.
Conversation closed.
```

## Admin side

```text
Unassigned
Claimed by You
Claimed by Agent Ahmed
Waiting for User
Waiting for Support
Resolved
```

Jab another agent claimed conversation open kare:

```text
Assigned to Agent Ahmed
Read-only
```

Reply box disabled hona chahiye.

---

# 13. Required QA Tests

Integration ke baad yeh tests karein:

1. User conversation create kar sakta hai.
2. User sirf apni conversations dekh sakta hai.
3. Two agents same time claim karein to sirf one succeeds.
4. Unassigned agent reply nahi kar sakta.
5. Assigned agent reply kar sakta hai.
6. Agent chat successfully transfer kar sakta hai.
7. Old agent transfer ke baad reply nahi kar sakta.
8. New agent transfer ke baad reply kar sakta hai.
9. User reply assigned agent ko show hota hai.
10. Resolve aur close actions properly work karte hain.
11. Closed conversation mein messages block hote hain.
12. Unauthorized chat ID request `403` ya `404` return karti hai.
13. Internal notes user ko show nahi hoti.
14. Attachments secure hain.
15. Unread counts correctly update hote hain.

---

# Recommended Integration Order

```text
1. Database migrations
2. Conversation and message models
3. User conversation APIs
4. Admin unassigned queue
5. Atomic Claim Chat logic
6. Agent reply permissions
7. Transfer functionality
8. User chat interface
9. Admin chat interface
10. AJAX polling or WebSockets
11. Notifications
12. Audit logs
13. Security and QA testing
```

Sabse pehle **conversation, message, claim aur transfer workflow** implement karein. Real-time WebSocket baad mein add kiya ja sakta hai; claim ownership database level par start se implement honi chahiye.

[1]: https://laravel.com/docs/12.x/broadcasting?utm_source=chatgpt.com "Broadcasting | Laravel 12.x - The clean stack for Artisans and agents"
[2]: https://ably.com/docs/presence-occupancy?utm_source=chatgpt.com "Ably Pub/Sub | Presence and occupancy overview"
