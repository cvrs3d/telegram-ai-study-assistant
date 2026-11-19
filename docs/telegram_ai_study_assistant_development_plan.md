# Telegram AI Study Assistant MVP Development Plan

**November 19, 2026 · Based on specification v0.2**

## Release goal

A single user launches the Mini App from Telegram on a phone or computer. They can start or resume an independent conversation with Opus 5.5 High, upload files, see responses as they stream, work through complex tasks in stages, and download results. The same interface supports web search with sources and interactive English quizzes. The project owner sets a spending threshold and receives a Telegram alert when spending exceeds it.

[Technical specification v0.2](telegram_ai_study_assistant_spec_v0_2.md)

## Proposed architecture

| Component | Starting point |
|---|---|
| UI | React, TypeScript, and Vite. One responsive Mini App with a chat list, messages, attachments, sources, and interactive quizzes. |
| API and bot | FastAPI and aiogram. FastAPI validates Telegram `initData` and issues a session to the authorized user. The bot opens the Mini App and handles the owner's administrative commands. |
| Request execution | A separate Python worker from the same repository processes jobs stored in PostgreSQL. The backend limits concurrent computations and stores state for each run separately. |
| Storage | PostgreSQL stores messages, runs, stages, quizzes, and usage records. A persistent volume holds source files and downloadable results. |
| Event delivery | Server-Sent Events (SSE) carry text, status updates, and file events from the backend to the UI. On reopening, the client loads persisted state and subscribes to new events for the run. |
| Model and computation | Anthropic Messages API, fixed `claude-opus-5-5` with `output_config.effort = "high"`, Code Execution, and Web Search |
| Deployment | Docker Compose, HTTPS through Caddy, and a dedicated VPS. Initial estimate is 2 vCPUs, 4 GiB RAM, and a 40 GiB SSD. |

Serve the Mini App and the API from the same domain. The bot opens the Mini App, which uploads attachments directly to the backend. The backend stores source files and downloads generated results from Anthropic so they stay available in the conversation. [Telegram Mini Apps](https://core.telegram.org/bots/webapps) · [Claude streaming](https://platform.claude.com/docs/en/build-with-claude/streaming) · [Anthropic Files API](https://platform.claude.com/docs/en/build-with-claude/files)

## Development sequence

### Project setup and first commit

Create a dedicated private repository named `telegram-ai-study-assistant`. The first milestone is a project that can be cloned and set up with one command.

1. Put the current specification and development plan in `docs/`. Describe the project's purpose and document the installation command in `README.md`.
2. Set up Python 3.12 and use uv for dependency management. Add the `anthropic` SDK and commit `pyproject.toml` and `uv.lock`.
3. Add `.env.example` with an empty `ANTHROPIC_API_KEY=`. Add the local `.env`, virtual environment, personal input files, and run outputs to `.gitignore`.
4. Run `uv sync --locked` and `uv run --locked python -c "import anthropic"` from a clean clone. The import check needs neither an API key nor a request to the model.

**Done when** the first commit contains the documentation and a reproducible environment, and the check passes from a clean clone. The VPS and Telegram are needed at stage 1.

### 0. Validate Opus with files

1. Write a local test script. Send text to Opus 5.5 High and capture the response stream and API usage data.
2. Use a PDF assignment and XLSX input in the same scenario. Ask Opus to extract the methodology, perform the calculation, and return an XLSX file. Download the workbook and compare its values with the reference result.
3. Separately test a scan, a PDF of about 70 MiB, web search citations, and XLSX generation with formulas. For the large PDF, verify page-range processing if the full document exceeds the provider's processing limit.
4. Record model and tool version IDs, formats used, typical errors, response times, and actual token usage. Use a real assignment from the intended user as a benchmark if available. Until then, use a synthetic assignment with a known answer.

**Done when** a verified Excel workbook can be downloaded through the Opus API and a repeat run matches the reference result. For scans and large PDFs, document what can be processed directly and when a page range is needed. [Document tools and Skills in the API](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart) · [PDF processing](https://platform.claude.com/docs/en/build-with-claude/pdf-support)

### 1. Launch the Mini App with text chat

1. Run the application, bot, and PostgreSQL with Docker Compose. Add migrations for users, conversations, messages, and runs.
2. Verify the signature and freshness of Telegram `initData` on the server. Issue a session to the authorized Telegram user ID and check conversation access on every read or write.
3. Add a chat list, creation, history search, renaming, and deletion. Reopening the Mini App restores the last chat and its saved history.
4. Save the user message and the generation job before calling Opus. The worker loads the context for that conversation, stores the result, and publishes events by `run_id`.
5. Stream responses to the UI. The request continues when the user closes the app or switches chats. On return, restore the text received so far and the current status.

**Done when** the intended user can create two chats, close Telegram while a response is streaming, return to the result, and continue one conversation while preserving the separate context of the other. Verify launch inside Telegram over HTTPS.

### 2. Add files, previews, and revisions through chat

1. Stream uploads to disk, show upload progress, and enforce limits. Start with 80 MiB per file, 100 MiB in attachments per message, and a provisional 64 KiB text limit. Check actual data size and the disk quota.
2. Associate each file with a conversation and a message. Store the display name, size, format, file source, and Anthropic file ID. Check conversation access before serving downloads.
3. Support PDF, scans, images, DOCX, CSV, TSV, TXT, and XLSX inputs. Cover the specified formats in initial testing. Record in a format matrix which can be sent directly to the provider and which require conversion.
4. Return downloadable XLSX, DOCX, PDF, PPTX, and charts. Store each result version as a separate record in the same conversation. For Excel workbooks with formulas, verify recalculation in Excel or a compatible application.
5. When deleting a conversation, clean up associated working files and remove their copies from the Anthropic Files API once they are no longer in use. Set a separate retention period for backups. Clean up incomplete uploads separately.
6. Add previews for PDFs, charts, and XLSX results tables. On desktop, open a panel beside the chat. On mobile, open a separate screen. Tie the open result to its `artifact_id`, `version_id`, and `conversation_id`. Store the version's links to the file and its preview. Select the viewer based on content type.
7. Include the selected version in revision requests. Update the panel once the new version is saved successfully. Verify that the download matches the displayed version and that the previous result remains available after an error or a successful rerun.

**Done when** the user can view a results table in Telegram, ask for a recalculation in chat, and verify that the updated preview matches the downloaded workbook. A PDF of about 70 MiB uploads alongside an XLSX within the combined limit. The per-file, combined-size, and PDF processing limit errors are distinguishable. The intended user can download the new version after reopening the Mini App and verify its numbers against the reference.

### 3. Add multi-stage tasks with approval in chat

1. Store the plan, current stage, interim result versions, and links to relevant files. Opus shows the plan and automatically starts stage one if it has enough information.
2. After producing the result, the stage waits for the user's message and releases the execution slot. Use the message and conversation history to distinguish a revision, approval, question, or ambiguous reply.
3. Save revisions as new versions of the same stage. On approval, the backend verifies that the user approved the current result and creates a single run of the next stage. Approval uses a regular message and accepts natural phrasing.
4. Build the next stage's input from the approved data, method, and files. Enforce the one-active-run limit on the server when enqueuing a job. Display status separately in each conversation.

**Done when** two successive revisions change only the current stage. A message such as “That's right now. Continue.” starts exactly one run of the next stage using the corrected data. If the user switches chats while a stage awaits approval, they can resume the task when they return.

### 4. Add web search with sources

1. Add Anthropic Web Search to ordinary requests to Opus. Store citations with the response and render them as clickable links in chat.
2. Make the per-message search call limit configurable. Choose a starting value during prototyping based on the tasks and their cost. If the tool is unavailable, explain the error and let the user retry with a regular message.
3. Include web search costs in API usage accounting.

**Done when** a request for current information triggers a real search, returns verifiable links, and keeps those links accessible after reopening the chat. [Web Search in the Claude API](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool)

### 5. Add interactive English quizzes

1. Ask Opus to return a structured quiz with questions, options, an answer key, and explanations. Validate the structure on the backend and send the questions and options to the UI.
2. Save answers as they are selected. When the user returns to the chat, show the unfinished quiz with their previous selections.
3. On a message such as “Check my answers,” calculate the score on the server and show explanations. Handle open-ended answers and requests to explain a mistake in regular chat.

**Done when** the intended user starts a quiz, closes Telegram, returns, changes or adds answers, asks to have the quiz graded, and sees the result. Verify that selecting options and scoring the quiz require no per-action token usage.

### 6. Track spending and send alerts

1. Record API usage once per call ID. Track tokens, caching, and billable tools separately where usage is available. Label costs as estimates.
2. Implement owner commands `/budget` and `/budget <USD>` in a private chat with the bot. Save the new threshold in PostgreSQL and apply it immediately.
3. Create an alert delivery record on the first threshold breach per calendar month. Persist delivery state and retry after temporary failures. The intended user retains access to the app.
4. Test with synthetic amounts and reconcile one real billing period against the Anthropic report.

**Done when** the owner can change the threshold without a restart, another user's command is rejected, only the owner receives the alert, and subsequent spending updates create no additional alert record for that month. [Anthropic Usage and Cost API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api)

### 7. Complete acceptance testing and release the MVP

1. On the VPS, verify migrations, HTTPS, persistent volumes, backups, and a test restore of chat history and files together. Log errors using conversation and run IDs and exclude study material text.
2. Test an expired Anthropic container, a dropped connection, a worker restart, and duplicate message delivery. If the outcome of an external call is unknown, show its status and let the user explicitly retry. Any new billable attempt starts only when the user retries.
3. Run user scenarios in Telegram on a phone and a computer. Check the input with the on-screen keyboard open, switching chats, uploading a large file, and downloading a result.
4. Compare three assignments from the intended user with Claude on the web. Cover analysis of a scan, a multi-stage calculation using a PDF and spreadsheet, and revision of a previously generated file. Compare the method, key numbers, whether the files open, and formulas where required.
5. Share the bot link with the intended user once acceptance testing passes. After the first real tasks, review response times, VPS load, and API costs.

**Done when** the v0.2 acceptance criteria and data restore test pass. Document any deviation, the affected scenario, and the release decision.

## Artifacts after the MVP

The next phase aims to reproduce the Claude Artifacts capabilities that are useful to the Telegram user. Before starting that phase, build a feature matrix for the web app and define acceptance criteria. Evaluate direct editing, version viewing and restoration, interactive apps, and result sharing. Check that editors preserve formulas and formatting. For interactive apps, verify execution isolation and state persistence. Establish the common version and viewer contract in MVP stage 2.

## Keeping the MVP in scope

- Each change should address a specific scenario in the specification or a defect found during acceptance testing. Record the formats the intended user needs and that stage 0 confirms in the format matrix before stage 2 ends.
- The first release targets one user, one model, one VPS, and one active computation. Projects, Research, memory across chats, connectors, model switching, and general-purpose interactive Artifacts are scheduled for later consideration.
- Validate the proposed UI stack, PostgreSQL-backed worker, and SSE in the first end-to-end scenario. Finalize the disk quota, backup retention, and web search limit before release.

## First implementation task

Set up the repository as described in **Project setup and first commit**, then verify it from a clean clone. Build one reproducible flow that takes a **PDF and XLSX**, sends them to **Opus High with Code Execution**, and returns a **downloaded XLSX**. Use a small assignment with a known answer. Record the command, expected numbers, and actual results. Then send a simple text request through the Mini App and connect it to the verified file workflow.
