# n8n Dead Letter Queue (DLQ) System

A resilient, observable workflow error handling system for n8n that automatically captures, logs, and enables recovery from workflow failures through Slack integration.

> **⚠️ Security Notice**: This repository contains workflow templates with placeholder values. Before deploying, you **must** replace all `YOUR_SLACK_BOT_TOKEN_HERE`, `YOUR_N8N_API_KEY_HERE`, and `YOUR_N8N_INSTANCE_URL` placeholders with your actual credentials. Never commit real credentials to version control.

## 🎯 Overview

This Dead Letter Queue (DLQ) system provides comprehensive workflow observability and resilience for n8n automation workflows. When any workflow fails, the system:

1. **Captures** the error with complete execution context
2. **Logs** the failure to a Supabase database for tracking and analytics
3. **Notifies** the team via Slack with interactive action buttons
4. **Enables** one-click replay or manual resolution directly from Slack
5. **Tracks** the entire lifecycle from failure to resolution

## 🏗️ System Architecture

The DLQ system consists of three interconnected n8n workflows:

```
┌─────────────────────────────────────────────────────────────┐
│                     ANY n8n WORKFLOW                         │
│                         (fails)                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Workflow #1: Main Error Logger                             │
│  - Error Trigger catches all failures                       │
│  - Logs to Supabase DLQ database                           │
│  - Posts interactive Slack notification                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
              [Slack Notification with
               "Replay" & "Resolve" buttons]
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Workflow #2: DLQ - Slack Actions                          │
│  - Receives Slack button clicks via webhook                │
│  - Routes to Replay Worker or Resolve handler              │
│  - Updates Supabase status                                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Workflow #3: DLQ - Replay Worker                          │
│  - Fetches DLQ event from Supabase                         │
│  - Checks if error is replayable                           │
│  - Retries execution via n8n API                           │
│  - Updates Slack with success/failure status               │
└─────────────────────────────────────────────────────────────┘
```

## 📋 Workflows

### Workflow #1: Main Error Logger

**File**: `#1 - Main Error Logger.json`

**Purpose**: Global error handler that captures all workflow failures

![Main Error Logger Workflow](./%231%20Error%20logger.png)

**Nodes**:

1. **Error Trigger** - n8n's built-in Error Trigger node
   - Automatically fires when ANY workflow in the n8n instance fails
   - Captures complete execution context including error details, workflow info, and execution URL

2. **If** - Conditional node
   - Filters out retry-mode errors to prevent duplicate notifications
   - Only processes original failures, not subsequent retries

3. **Create a row** (Supabase) - Database insert
   - Stores failure details in `dlq_events` table:
     - `workflow_name`: Name of the failed workflow
     - `error_message`: Error description
     - `payload`: Complete JSON payload with execution context
     - `status`: Initial status set to "Pending"
     - `execution_id`: Unique n8n execution ID
     - `source`: Set to "Main Error Logger"

4. **HTTP Request** (Slack) - Interactive notification
   - Posts message to `#automation-alerts` channel
   - Includes rich formatted blocks with:
     - Workflow name and owner
     - Execution URL (direct link to n8n execution)
     - Error message
     - DLQ status and source workflow link
   - **Interactive buttons**:
     - 🔁 **Replay**: Attempts to re-run the failed workflow
     - ✅ **Resolve**: Marks the failure as manually resolved

5. **No Operation, do nothing** - Terminal node for filtered retries

**Key Features**:
- **Automatic capture**: No manual configuration needed per workflow
- **Deduplication**: Filters retry attempts to avoid spam
- **Rich context**: Preserves full execution state for debugging
- **Immediate notification**: Team is alerted within seconds

---

### Workflow #2: DLQ - Slack Actions

**File**: `#2 - DLQ - Slack Actions.json`

**Purpose**: Handles user interactions from Slack notification buttons

![DLQ - Slack Actions Workflow](./%232%20-%20DLQ%20-%20Slack%20Actions.png)

**Nodes**:

1. **Webhook: Slack Actions** - HTTP webhook endpoint
   - Path: `/dlq/actions`
   - Method: POST
   - Receives Slack's `x-www-form-urlencoded` payload when users click buttons

2. **Parse Slack Payload** (Code node) - Payload parser
   - Extracts and deserializes the nested JSON payload from Slack
   - Parses critical fields:
     - `action_id`: Either "dlq_replay" or "dlq_resolve"
     - `dlq_event_id`: The execution ID to act upon
     - `user_id` & `user_name`: Who triggered the action
     - `response_url`: Slack webhook for message updates
   - Handles errors gracefully with `has_error` flag

3. **If: Parsed OK** - Validation check
   - Ensures payload was successfully parsed
   - Routes only valid requests to action handler

4. **Switch: Action** - Action router
   - Routes based on `action_id`:
     - Output 1: "dlq_replay" → Replay flow
     - Output 2: "dlq_resolve" → Resolve flow

5. **Supabase: Mark Replay Requested** - Status update
   - Updates `dlq_events` table:
     - `status`: "replay_requested"
     - `source`: "slack"

6. **Slack: Update Message (Replay)** - Visual feedback
   - Updates the original Slack message to show "🔄 Replaying…"
   - Displays who triggered the replay
   - Provides interim status while replay is in progress

7. **HTTP Request** - Triggers Replay Worker
   - Calls Workflow #3's webhook endpoint
   - Sends payload:
     - `execution_id`: The failed execution ID
     - `channel` & `message_ts`: For updating the Slack message
     - `requested_by`: User who clicked Replay
     - `response_url`: Slack callback URL

8. **Supabase: Mark Resolved** - Manual resolution
   - Updates status to "resolved" when user clicks Resolve
   - Marks event as handled without replay

9. **Slack: Update Message (Resolve)** - Resolution confirmation
   - Updates Slack message to show ✅ Resolved
   - Displays who resolved it
   - No further action required

**Key Features**:
- **User-friendly**: One-click actions directly from Slack
- **Audit trail**: Tracks who performed which action
- **Status tracking**: Updates database and UI in real-time
- **Error handling**: Validates payloads before processing

---

### Workflow #3: DLQ - Replay Worker

**File**: `# 3 DLQ - Replay Worker.json`

**Purpose**: Intelligent replay engine that retries failed workflows

![DLQ - Replay Worker Workflow](./%23%203%20DLQ%20-%20Replay%20Worker.png)

**Nodes**:

1. **Webhook: Replay Worker** - Webhook receiver
   - Path: `/dlq/replay-worker`
   - Method: POST
   - Receives replay requests from Workflow #2

2. **Supabase: Fetch DLQ Event** - Event retrieval
   - Queries `dlq_events` table by `execution_id`
   - Retrieves complete failure record including stored payload

3. **Function: Parse Stored Payload** (Code node) - Payload deserializer
   - Parses the stored JSON payload (handles both string and JSONB formats)
   - Extracts key information:
     - Original execution metadata
     - Error details
     - Execution URL
     - Slack context (channel, message timestamp, requester)
   - Gracefully handles parse errors

4. **IF: Non-Replayable (SyntaxError)?** - Replay viability check
   - Analyzes error type to determine if replay is possible
   - **SyntaxError detection**: These require code fixes, not replay
   - Routes to appropriate handler:
     - **True (SyntaxError)**: Block replay, request manual fix
     - **False (other errors)**: Proceed with replay attempt

5. **Supabase: Mark Needs Fix** - Block non-replayable errors
   - Updates status to "needs_fix"
   - Indicates human intervention required

6. **Slack: Update Message (Replay Blocked)** - User notification
   - Updates Slack message with ⛔ Replay Blocked
   - Explains the SyntaxError requires code fixes
   - Provides link to execution for debugging
   - Instructs user to fix code before retrying

7. **HTTP: Replay Actual (Worker Webhook)** - Execution retry
   - Calls n8n API: `POST /api/v1/executions/{id}/retry`
   - Sends headers with API authentication
   - Includes replay context:
     - `source`: "dlq_replay"
     - `execution_id`: Original failed execution
     - `triggered_by`: Slack user who requested replay
     - `stored_payload`: Original execution data
   - Timeout: 120 seconds

8. **IF: Replay Error?** - Outcome check
   - Examines replay execution result
   - Checks if `executionStatus === "error"`
   - Routes to success or failure handler

9. **Supabase: Mark Replay Failed** - Failed replay logging
   - Updates status to "replay_failed"
   - Logs new error message
   - Increments `retry_count`

10. **Slack: Update Message (Replay Failed)** - Failure notification
    - Updates message with ❌ Replay Failed
    - Shows the new error message
    - Indicates workflow is still broken
    - Suggests fixing root cause before retry

11. **Supabase: Mark Replay Success** - Successful replay
    - Updates status to "replay_succeeded"
    - Clears error message
    - Marks event as resolved

12. **Slack: Update Message (Replay Success)** - Success notification
    - Updates message with ✅ Replay Successful
    - Confirms workflow recovered
    - Provides link to successful execution
    - Marks DLQ event as cleared

**Key Features**:
- **Intelligent filtering**: Detects non-replayable errors (SyntaxError)
- **Full context preservation**: Replays with original payload and metadata
- **Real-time feedback**: Updates Slack throughout the replay lifecycle
- **Comprehensive logging**: Tracks all replay attempts and outcomes
- **Resilience**: Handles both successful and failed replays gracefully

---

## 🗄️ Database Schema

The system uses Supabase with a `dlq_events` table:

```sql
CREATE TABLE dlq_events (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMP DEFAULT NOW(),
  workflow_name TEXT NOT NULL,
  execution_id TEXT UNIQUE NOT NULL,
  error_message TEXT,
  payload JSONB NOT NULL,
  status TEXT DEFAULT 'Pending',
  source TEXT,
  retry_count INTEGER DEFAULT 0
);
```

**Status values**:
- `Pending`: Initial state, awaiting action
- `replay_requested`: User clicked Replay button
- `replaying`: Replay in progress
- `replay_succeeded`: Replay completed successfully
- `replay_failed`: Replay attempted but failed again
- `needs_fix`: Non-replayable error, requires code changes
- `resolved`: Manually marked as resolved

## 🔧 Setup Instructions

### Prerequisites

- n8n instance (self-hosted or cloud)
- Supabase account and database
- Slack workspace with:
  - Bot token with `chat:write` and `chat:write.public` scopes
  - Interactive components enabled
  - Request URL configured for Workflow #2's webhook

### Installation

1. **Create Supabase table**:
   ```sql
   CREATE TABLE dlq_events (
     id SERIAL PRIMARY KEY,
     created_at TIMESTAMP DEFAULT NOW(),
     workflow_name TEXT NOT NULL,
     execution_id TEXT UNIQUE NOT NULL,
     error_message TEXT,
     payload JSONB NOT NULL,
     status TEXT DEFAULT 'Pending',
     source TEXT,
     retry_count INTEGER DEFAULT 0
   );
   ```

2. **Configure Supabase credentials in n8n**:
   - Add Supabase API credentials
   - Note the credential ID for workflow import

3. **Configure Slack credentials in n8n**:
   - Add Slack Bot Token (HTTP Header Auth)
   - Header name: `Authorization`
   - Header value: `Bearer xoxb-YOUR-BOT-TOKEN`

4. **Import workflows in order**:
   - Import `#1 - Main Error Logger.json`
   - Import `#2 - DLQ - Slack Actions.json`
   - Import `# 3 DLQ - Replay Worker.json`

5. **Update configuration** (REQUIRED):
   
   **⚠️ IMPORTANT: Replace all placeholder values before activating workflows**
   
   - **Supabase credentials**: Replace credential IDs in all workflows with your Supabase API credentials
   - **Slack bot token**: Replace `YOUR_SLACK_BOT_TOKEN_HERE` with your actual Slack bot token in:
     - Workflow #2, node "Slack: Update Message (Replay)" 
     - Workflow #2, node "Slack: Update Message (Resolve)"
     - Workflow #3, node "Slack: Update Message (Replay Blocked)"
     - Workflow #3, node "Slack: Update Message (Replay Failed)"
     - Workflow #3, node "Slack: Update Message (Replay Success)"
   - **n8n API configuration** in Workflow #3, node "HTTP: Replay Actual (Worker Webhook)":
     - Replace `YOUR_N8N_INSTANCE_URL` with your n8n instance URL (e.g., `https://n8n.yourdomain.com`)
     - Replace `YOUR_N8N_API_KEY_HERE` with your n8n API key
   - **Webhook URLs**: Replace `YOUR_N8N_INSTANCE_URL` in Workflow #2's HTTP Request node
   - **Slack channel**: Update channel name in Main Error Logger (default: `#automation-alerts`)

6. **Configure Slack Interactive Components**:
   - In Slack API settings, enable Interactive Components
   - Set Request URL to Workflow #2's webhook URL
   - Example: `https://your-n8n.com/webhook/dlq/actions`

7. **Activate workflows**:
   - Activate all three workflows
   - Test by intentionally failing a workflow

## 📸 Slack Notification Example

Here's what the interactive Slack notification looks like when a workflow fails:

![Slack Notification Example](./Slack%20Notification.png)

The notification includes:
- Clear error details and workflow information
- Direct link to the n8n execution
- Interactive **Replay** and **Resolve** buttons
- Status tracking as the event progresses

---

## 💪 Resilience Features

### 1. **Automatic Error Capture**
- Zero-configuration: Works for all workflows automatically
- Error Trigger catches failures across the entire n8n instance
- No need to add error handling to individual workflows

### 2. **Deduplication**
- Filters out retry-mode executions to prevent duplicate notifications
- Prevents alert fatigue from the same failure

### 3. **Intelligent Replay Logic**
- Detects non-replayable errors (SyntaxError) that require code fixes
- Prevents wasted replay attempts on unfixable errors
- Provides clear guidance on next steps

### 4. **Graceful Failure Handling**
- If replay fails, logs the new error and notifies team
- Maintains complete audit trail of all retry attempts
- Never loses failure information

### 5. **Idempotent Operations**
- Database updates use unique execution_id constraints
- Prevents duplicate DLQ entries for same failure
- Safe to re-run without side effects

## 📊 Observability Features

### 1. **Complete Execution Context**
- Stores full payload including:
  - Workflow ID and name
  - Execution ID with direct URL
  - Error message and stack trace
  - Node that failed
  - Execution mode (webhook, manual, trigger)
  - Timestamp and duration

### 2. **Real-Time Notifications**
- Immediate Slack alerts on failure
- Rich formatted messages with all key details
- Direct links to n8n execution for debugging

### 3. **Status Tracking**
- Every DLQ event has a status field
- Status updates tracked through lifecycle:
  - Pending → Replay Requested → Replaying → [Success/Failed/Needs Fix]
  - Alternative: Pending → Resolved
- Audit trail of who performed actions and when

### 4. **Slack Message Updates**
- Original notification message updates as status changes
- Team sees the full history in a single thread
- No duplicate notifications for same failure

### 5. **Analytics-Ready Data**
- Supabase table enables queries like:
  - Most common errors
  - Workflows with highest failure rates
  - Average time to resolution
  - Replay success rates
  - Error trends over time

### 6. **Retry Counter**
- Tracks number of replay attempts per event
- Helps identify chronically problematic workflows
- Can set policies (e.g., escalate after 3 failed replays)

## 🚀 Usage

### When a Workflow Fails

1. **Automatic Capture**: The Error Trigger fires immediately
2. **Notification**: You receive a Slack message in `#automation-alerts`
3. **Review**: Click "Open Execution" to view failure details in n8n
4. **Choose Action**:

#### Option A: Replay
- Click **🔁 Replay** if the error was transient (network issue, API timeout, etc.)
- The system will:
  - Update Slack to "🔄 Replaying…"
  - Check if error is replayable
  - Retry the execution with original payload
  - Update Slack with ✅ Success or ❌ Failed result

#### Option B: Resolve
- Click **✅ Resolve** if you've manually fixed the issue or determined it's not critical
- Confirm the resolution
- The system marks the event as resolved

### Monitoring DLQ Health

Query Supabase to monitor your DLQ:

```sql
-- View all pending failures
SELECT * FROM dlq_events WHERE status = 'Pending';

-- View replay success rate
SELECT 
  COUNT(CASE WHEN status = 'replay_succeeded' THEN 1 END) as successes,
  COUNT(CASE WHEN status = 'replay_failed' THEN 1 END) as failures
FROM dlq_events
WHERE status IN ('replay_succeeded', 'replay_failed');

-- Most common errors
SELECT error_message, COUNT(*) as count
FROM dlq_events
GROUP BY error_message
ORDER BY count DESC
LIMIT 10;
```

## 🔐 Security Considerations

- **Slack Token**: Store Slack bot token securely as n8n credential
- **n8n API Key**: Use API keys with minimal required permissions
- **Webhook URLs**: Use HTTPS for production deployments
- **Supabase**: Enable Row Level Security (RLS) policies
- **Payload Data**: Be mindful of sensitive data in error payloads

## 📈 Metrics & KPIs

Track these metrics to measure workflow reliability:

- **MTTR** (Mean Time To Recovery): Time from failure to resolution
- **Replay Success Rate**: Percentage of successful replays
- **Failure Rate by Workflow**: Identify problematic workflows
- **Error Categories**: Group by error type (API, syntax, network, etc.)
- **Manual vs Automated Resolution**: Track replay vs resolve button usage

## 🛠️ Customization Ideas

- **Severity Levels**: Add priority field (P1/P2/P3) based on workflow importance
- **PagerDuty Integration**: Escalate critical failures to on-call engineer
- **Auto-Replay**: Automatically retry certain error types without human intervention
- **Retention Policy**: Archive old DLQ events after 30 days
- **Email Notifications**: Add email alerts for high-priority failures
- **Metrics Dashboard**: Build Grafana dashboard from Supabase data

## 📝 License

MIT License - Feel free to use and modify for your needs

## 🤝 Contributing

Contributions welcome! Please open an issue or PR.

## 📧 Support

For questions or issues, please open a GitHub issue.

---

**Built with** ❤️ **using n8n, Supabase, and Slack**
