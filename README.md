# ERPNext AI-Powered Meeting Analysis DocType Specification

## 1. Introduction

This document outlines the specifications for a new DocType in ERPNext called "AI Meeting Analyzer." This DocType will allow users to upload meeting recordings (audio or video), which will then be processed by an AI service. After processing, users can interact with an AI chat interface to ask questions and retrieve information from the meeting content.

## 2. DocType Details

*   **DocType Name**: `AI Meeting Analyzer`
*   **Module**: (Suggest a relevant module, e.g., "Tools", "CRM", "Projects", or a new custom module "AI Services")
*   **Purpose**: To store, manage, and facilitate AI-powered analysis of meeting recordings, enabling users to query meeting content via a chat interface.
*   **Naming**: "Meeting Recording - {meeting_title} - {#####}" (Auto-name based on title)

## 3. Fields

| Label                     | Field Name                | Type              | Req'd | Description                                                                                                | Notes                                                                 |
| :------------------------ | :------------------------ | :---------------- | :---- | :--------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------- |
| **Meeting Details**       |                           |                   |       |                                                                                                            |                                                                       |
| Meeting Title             | `meeting_title`           | Data              | Yes   | Title of the meeting.                                                                                      |                                                                       |
| Meeting Date              | `meeting_date`            | Date              | Yes   | Date the meeting took place.                                                                               |                                                                       |
| Participants              | `participants`            | Table (MultiSelect) | No    | Link to `Contact` or `User` DocTypes for attendees.                                                        | Child table: `Meeting Participant` with field `participant` (Link). |
| Related To (Optional)     | `related_doctype`         | Link              | No    | Link to another DocType (e.g., `Lead`, `Opportunity`, `Project`, `Customer`).                              | Dynamic Link based on `related_doctype_name`.                       |
| Related Document          | `related_docname`         | Dynamic Link      | No    | The specific document name for the `related_doctype`.                                                      |                                                                       |
| **Recording & Analysis**  |                           |                   |       |                                                                                                            |                                                                       |
| Recording File            | `recording_file`          | Attach Image / File | Yes   | The audio or video recording of the meeting.                                                               | Supported formats: MP3, WAV, MP4, MOV (configurable).               |
| File Mimetype             | `file_mimetype`           | Data              | No    | Read-only. Mimetype of the uploaded file.                                                                  | Automatically populated on file upload.                             |
| File Size (MB)            | `file_size_mb`            | Float             | No    | Read-only. Size of the uploaded file in MB.                                                                | Automatically populated on file upload.                             |
| Analysis Status           | `analysis_status`         | Select            | Yes   | Status of the AI analysis.                                                                                 | Options: "Pending Upload", "Pending Analysis", "Analyzing", "Completed", "Error". Default: "Pending Upload". Read-only after "Pending Analysis". |
| Analysis Error Message    | `analysis_error_message`  | Small Text        | No    | Stores any error message if the analysis fails.                                                            | Read-only. Visible only if `analysis_status` is "Error".            |
| Transcription             | `transcription`           | Text Editor       | No    | Read-only. The full transcription of the meeting, if provided by the AI.                                   |                                                                       |
| Summary                   | `summary`                 | Text Editor       | No    | Read-only. AI-generated summary of the meeting.                                                            |                                                                       |
| **AI Chat Interface**     |                           |                   |       |                                                                                                            |                                                                       |
| Chat History              | `chat_history`            | Table             | No    | Stores the conversation with the AI.                                                                       | Child Table: `AI Chat Message`.                                       |

### Child Table: `Meeting Participant`

| Label       | Field Name    | Type          | Req'd | Description                  |
| :---------- | :------------ | :------------ | :---- | :--------------------------- |
| Participant | `participant` | Link (User/Contact) | Yes   | Meeting attendee.            |

### Child Table: `AI Chat Message`

| Label        | Field Name   | Type        | Req'd | Description                                           |
| :----------- | :----------- | :---------- | :---- | :---------------------------------------------------- |
| Sender       | `sender`     | Select      | Yes   | "User" or "AI".                                       |
| Message      | `message`    | Text Editor | Yes   | The content of the chat message.                      |
| Timestamp    | `timestamp`  | Datetime    | Yes   | When the message was sent. Default: `Now`.            |
| AI Reference | `ai_reference` | Small Text  | No    | Optional: Pointers/timestamps from the recording AI refers to. |

## 4. Workflow and Process

1.  **Creation & File Upload**:
    *   User creates a new "AI Meeting Analyzer" document.
    *   User fills in "Meeting Title", "Meeting Date", and other relevant details.
    *   User uploads the meeting recording (`recording_file`).
    *   On file upload, `file_mimetype` and `file_size_mb` are populated.
    *   `analysis_status` is initially "Pending Upload". Once file is attached, it becomes "Pending Analysis".

2.  **AI Analysis Trigger**:
    *   Upon saving the document with an attached file and `analysis_status` as "Pending Analysis", a background job should be triggered.
    *   This job sends the `recording_file` (or a secure link to it) to the designated AI service API endpoint.
    *   The `analysis_status` changes to "Analyzing".
    *   The system should handle API authentication and potential file size limits or chunking if required by the AI service.

3.  **AI Processing**:
    *   The external AI service processes the recording. This may include:
        *   Speech-to-text transcription.
        *   Speaker diarization (identifying different speakers).
        *   Content summarization.
        *   Indexing the content for querying.
    *   The specifics depend on the chosen AI service capabilities.

4.  **Receiving Analysis Results**:
    *   Once the AI service completes processing, it should send a callback to an ERPNext API endpoint or ERPNext should poll for results.
    *   The system updates the "AI Meeting Analyzer" document:
        *   `analysis_status` changes to "Completed" (or "Error" if failed).
        *   If an error occurs, `analysis_error_message` is populated.
        *   Populate `transcription` and `summary` fields if the AI provides them.
    *   If successful, the AI Chat interface is enabled for this document.

5.  **AI Chat Interaction**:
    *   A new section or custom button "Chat with AI" becomes visible/enabled on the DocType form once `analysis_status` is "Completed".
    *   This interface will have:
        *   An input field for the user to type their questions.
        *   A display area for the chat history (`chat_history` child table).
    *   When a user sends a message:
        *   The message is logged in `chat_history` with `sender`="User".
        *   The message is sent to the AI service API (along with context, like the ID of the analyzed meeting or the full chat history for conversational context).
        *   The AI's response is received.
        *   The AI's response is logged in `chat_history` with `sender`="AI".
        *   The AI response is displayed to the user.

## 5. AI Integration Details

*   **AI Service Endpoint**: Configuration for the AI service API URL needs to be stored securely (e.g., in "Custom Settings" or a dedicated AI Settings DocType).
*   **API Key Management**: Secure storage and handling of API keys for the AI service.
*   **Data Transfer**:
    *   For sending the recording: Secure transfer of the media file. Consider direct upload from ERPNext server or signed URLs if the AI service supports it.
    *   For chat: JSON payloads for questions and answers.
*   **Error Handling**: Robust error handling for API communication failures, AI processing errors, etc.
*   **Asynchronous Processing**: Analysis of large files can take time. The process must be asynchronous (background jobs) to avoid freezing the UI.

## 6. User Interface (UI) - Chat Section

*   The chat interface should be embedded within the "AI Meeting Analyzer" DocType view, likely below the main details or in a separate tab.
*   It should only be active/visible when `analysis_status` is "Completed".
*   **Components**:
    *   **Chat Display Area**: Shows messages from `chat_history`. User messages aligned to one side, AI messages to the other. Timestamps should be visible.
    *   **Message Input Field**: A text area for users to type their questions.
    *   **Send Button**: To submit the question.
*   **Real-time Updates (Optional but Recommended)**: Consider using WebSockets or similar for a more responsive chat experience, though polling can be a simpler first step.

## 7. Permissions (Standard ERPNext Role Permissions Manager)

*   Roles that can create/read/write/delete "AI Meeting Analyzer" documents.
*   Consider if there are any restrictions on who can trigger AI analysis or interact with the chat.

## 8. Custom Scripts / Server Logic

*   **Client Scripts**:
    *   Populate `file_mimetype` and `file_size_mb` on upload.
    *   Control UI elements based on `analysis_status` (e.g., disable chat input if not "Completed").
*   **Server Scripts (Python)**:
    *   `on_update` or `on_submit` (if DocType is submittable) hooks to trigger the AI analysis background job.
    *   API endpoints for the AI service callback (if used).
    *   Logic to interact with the AI service API (sending requests, handling responses).
    *   Logic to update the DocType with analysis results.
    *   Controller methods for handling chat messages (sending to AI, saving history).

## 9. Future Enhancements / Considerations

*   **Speaker Identification Display**: If the AI supports speaker diarization, display speaker labels in the transcription and chat.
*   **Action Item Detection**: AI to identify and list potential action items from the meeting.
*   **Sentiment Analysis**: Analyze and display the overall sentiment or sentiment per speaker.
*   **Topic Modeling**: Identify key topics discussed.
*   **Search within Transcription**: Allow users to search for keywords in the generated transcription.
*   **Cost Tracking**: If the AI service is usage-based, consider tracking costs per analysis.
*   **Multi-language Support**: Support for recordings in different languages.
*   **Editing Transcripts**: Allow users to manually correct AI-generated transcripts.
*   **Periodic Re-analysis**: Option to re-analyze if AI models improve.

## 10. Technical Stack Considerations (for Dev Team)

*   **Backend**: Python (Frappe framework).
*   **Background Jobs**: Frappe's built-in background jobs (RQ).
*   **API Integration**: Python `requests` library or Frappe's `frappe.utils.data.execute_json` / `make_post_request`.
*   **AI Service**: Specify the intended AI service provider if known (e.g., OpenAI, AssemblyAI, Google Cloud Speech-to-Text & Vertex AI, AWS Transcribe & Bedrock). This will heavily influence API specifics. If not specified, design for a generic API interface that can be adapted.

This specification should provide a solid foundation for your ERPNext development team. Remember to discuss the choice of AI service with them, as it will impact the implementation details of the AI integration part.
