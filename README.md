# Vapi.ai Lead Qualifier & Analyzer (n8n Workflow)

This n8n workflow automates the post-call analysis for **Vapi.ai** voice agents. It receives call data via a Webhook, utilizes **Google Gemini** to analyze the conversation transcript/summary, determines if the lead is qualified, and syncs the structured results to **Google Sheets**.

## 🚀 Features

  * **Automated Ingestion:** Listens for `POST` webhooks from Vapi containing call analysis.
  * **AI-Powered Analysis:** Uses an AI Agent (Google Gemini) to review the call summary.
      * Determines qualification status (`Qualified`, `Not Qualified`, `Follow-up`).
      * Extracts specific reasons, next steps, interest levels, budgets, and objections.
  * **Smart Filtering:** Includes logic to ensure only calls with valid summaries are processed.
  * **Structured Data Parsing:** Automatically extracts JSON fields from the AI response using Regex.
  * **CRM/Database Sync:** Upserts (Appends or Updates) lead data into Google Sheets based on the lead's Name.

## 📋 Prerequisites

Before importing this workflow, ensure you have the following:

1.  **n8n Instance:** A self-hosted or cloud version of n8n (Version 1.x+ recommended).
2.  **Vapi.ai Account:** To generate the call data and trigger the webhook.
3.  **Google Cloud Credentials:**
      * **Google Gemini (PaLM) API Key:** For the AI Chat Model.
      * **Google Sheets OAuth2 Credential:** To read/write to your spreadsheet.

## 🛠️ Setup & Configuration

### 1\. Import the Workflow

1.  Download the `vapi analysis lead qualifier tool 1.json` file.
2.  In your n8n dashboard, go to **Workflows** \> **Import from File**.
3.  Select the JSON file.

### 2\. Configure Credentials

You must update the nodes with your specific credentials:

  * **Google Gemini Chat Model:** Select your *Google Gemini(PaLM) Api account*.
  * **Update Lead (Google Sheets):** Select your *Google Sheets account*.

### 3\. Configure the Google Sheet

Create a new Google Sheet with the following header row (exact spelling recommended):
| name | phone number | qualified | reason | next steps | interest | budget\_mentioned | timeline | objections | gender |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |

  * **Node Configuration:** Open the "Update Lead" node.
      * **Document:** Select your specific Spreadsheet.
      * **Sheet:** Select the specific Sheet (Tab) name.

### 4\. Configure Vapi Webhook

1.  Open the **Webhook** node in n8n.
2.  Copy the **Production URL**.
3.  Go to your **Vapi Dashboard** \> **Server URL** (or specific Assistant settings).
4.  Paste the n8n Webhook URL.

## 🧠 Workflow Logic

1.  **Webhook:** Receives the payload from Vapi (contains `message.analysis.summary` and `structuredData`).
2.  **If Node:** Checks if `summary` exists and is not empty. If empty, the workflow stops.
3.  **Lead Qualifier Agent (AI):**
      * The agent receives the call summary.
      * It runs a system prompt to determine if the lead is qualified.
      * It outputs a raw JSON string containing `qualified`, `reason`, `next_steps`, etc.
4.  **Edit Fields (Set Node):** Uses Regex to parse the AI's string output into usable n8n JSON variables.
5.  **Update Lead:** Maps the Vapi structured data (Name, Phone) and the AI analysis data to the Google Sheet columns. It matches rows based on the **name** column to prevent duplicates.

## 📦 Expected Payload (Input)

This workflow expects a JSON structure similar to the standard Vapi `end-of-call-report`:

```json
{
  "message": {
    "analysis": {
      "summary": "The user was interested in the pricing tier...",
      "structuredData": {
        "name": "John Doe",
        "phone number": "+15550001234",
        "gender": "male"
      }
    }
  }
}
```

## 📊 Output Data (Google Sheets)

The AI will populate the sheet with analysis like this:

  * **Qualified:** `yes` / `no` / `follow_up`
  * **Reason:** "Customer has budget and authority."
  * **Next Steps:** "Send contract, Schedule demo."
  * **Interest:** "High, asked about API limits."
  * **Objections:** "Concerned about onboarding time."


For the **booking agent**, this is tailored for a **Vapi Server-Side Tool** setup, as this workflow operates differently than the previous analysis tool (it runs *during* the call to perform actions).


# Vapi.ai ↔ Google Calendar Booking Agent

This n8n workflow functions as a **Server-Side Tool** for Vapi.ai agents. It allows your voice assistant to check real-time availability and book appointments directly on **Google Calendar** during a conversation.

## 🚀 Features

  * **Intelligent Intent Detection:** Uses **Google Gemini** to analyze the conversation transcript and decide if the user wants to *check availability* or *finalize a booking*.
  * **Real-Time Availability:** Queries Google Calendar to find free slots between specific times.
  * **Automatic Booking:** Creates calendar events with dynamic Start/End times based on the conversation.
  * **Context Aware:** Aggregates the chat history (`messages`) to ensure the AI understands the full context of the booking request.
  * **Vapi Tool Protocol:** Formats the output specifically for Vapi's `toolCallId` requirements so the voice agent can speak the result back to the user.

## 📋 Prerequisites

1.  **n8n Instance:** Self-hosted or cloud (v1.x+).
2.  **Google Cloud Credentials:**
      * **Google Gemini (PaLM) API:** For the logic agent.
      * **Google Calendar OAuth2:** To read availability and write events.
3.  **Vapi.ai Account:** To configure the Assistant with this tool.

## 🛠️ Setup & Configuration

### 1\. Import the Workflow

1.  Download the `cognivue customer care booking agent.json` file.
2.  Import it into n8n via **Workflows** \> **Import from File**.

### 2\. Configure Credentials

  * **Google Gemini Chat Model:** Select your PaLM API credential.
  * **Google Calendar Nodes:**
      * Update the **Check availability** node credential.
      * Update the **Book appointment** node credential.

### 3\. Update Calendar ID (Crucial)

By default, the workflow points to a placeholder calendar. You must update this to your own.

1.  Open the **check availability** node.
2.  Change the **Calendar** field from `buchinnamani208@gmail.com` to your own Google Calendar ID (usually your email address).
3.  Repeat this step for the **Book appointment** node.

### 4\. Configure Vapi Tool

In your Vapi Dashboard, add a **Tool** to your assistant:

  * **Type:** `Function` (or Server URL depending on your setup).
  * **Server URL:** Paste the **Production URL** from the n8n **Webhook** node.
  * **Function Name:** `booking_tool` (or similar).
  * **Description:** "Used to check availability and book appointments."

## 🧠 Workflow Logic

1.  **Webhook:** Receives the Tool Call payload from Vapi.
2.  **Split & Aggregate:** Extracts the `artifact.messages` (chat history) to give the AI context on what was said previously.
3.  **Booking Agent (AI):**
      * Analyzes the transcript.
      * **If asking for time:** Calls the `check_availability` tool.
      * **If confirming a slot:** Calls the `book_appointment` tool.
      * It generates dynamic start/end times based on the user's request (e.g., "Next Tuesday at 2 PM").
4.  **Google Calendar Tools:**
      * **Check:** Returns busy/free status for the requested window.
      * **Book:** Creates the actual event for 1 hour (default duration).
5.  **Respond to Webhook:** Returns the result in the specific JSON format Vapi requires to continue the conversation:
    ```json
    {
      "results": [
        {
          "toolCallId": "call_12345...",
          "result": "The appointment is booked for Tuesday at 2 PM."
        }
      ]
    }
    ```

## 📦 Expected Input (Vapi Payload)

The workflow expects the Vapi Tool payload to include the message history artifact:

```json
{
  "message": {
    "toolCalls": [{"id": "call_123..."}],
    "artifact": {
      "messages": [
        {"role": "assistant", "content": "When would you like to come in?"},
        {"role": "user", "content": "Do you have anything next Monday?"}
      ]
    }
  }
}
```

## 📄 License

MIT
