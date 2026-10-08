# Maintenance guide

## Start with the relevant module

| Change or symptom | Workflow / node |
| --- | --- |
| Daily schedule | Dispatcher: `Daily 04:50 Cairo` and `Plan Dispatch` |
| Batch target, parallelism, candidate limit | Dispatcher: `Plan Dispatch` configuration |
| Topic selection, batch overlap, stuck reservations | Dispatcher: `Plan Dispatch`; inspect `Topics` and `Runs` |
| Article settings and editorial prompts | Main worker: `Run Settings`, `Prompt Library` |
| Worker ownership or controlled image recovery | Main worker: `Prepare Worker Claim`, `Verify Worker Claim`, `Prepare Image Recovery Input` |
| Research and source quality | Research module |
| Article generation or automatic content review | Writing/review module |
| Image-plan generation | Main worker: `Plan Featured Image` and its Groq model |
| OpenAI images, throttling, branding, WebP output | Image module |
| Review delivery or approval parsing | Telegram module |
| Media upload, duplicate handling, draft creation, SEO metadata | WordPress module |
| Per-article final status and failure notifications | Main worker |
| Batch counters and batch notifications | Dispatcher |

The schedule is deliberately represented in more than one place. To change 04:50, review the Schedule Trigger, the `canDispatch` expression in `Plan Dispatch`, and its `scheduled_at` label together.

## Module contracts

The dispatcher submits `batch_id`, `topic_id`, `reservation_token`, and an attempt number to `Worker Input`. The main worker reads the reserved row and claims it before performing article work.

The main worker passes article state to the research and writing modules through their dedicated input preparation nodes. Automatic and human revisions return through the writing module.

The image module accepts article `state` and a four-role image plan. It also has paths for already packaged images and saved generation responses. State must survive every image step so that topic, run, and revision ownership remain attached to the assets.

The Telegram module receives the packaged article state and binary attachments. The parent records the review status, and the module returns the human decision. The review deadline and token identify the pending revision; they should not be regenerated accidentally on a retry.

After approval, the main worker validates publication ownership and invokes the WordPress module. Its publication checkpoint belongs to the parent worker's run ID, not the child execution ID. The parent then records the confirmed draft and sends the result link.

The main worker waits for each child. The dispatcher starts workers without waiting for their complete review lifecycle.

## Queue and ledger

`Topics` is the per-article queue. `Runs` is the batch ledger. Their statuses have different meanings:

| Location | State | Meaning |
| --- | --- | --- |
| Topics | blank / `pending` | Potentially eligible; the other eligibility fields must also be clear |
| Topics | `queued` / `processing` | Reserved or owned by a worker |
| Topics | `awaiting_review` / `revising` | Waiting for or applying review |
| Topics | `draft_created` | Confirmed draft; verify post ID and URL are present |
| Topics | `blocked` | Inspect `failure_class`, `last_error`, and ownership before recovery |
| Topics | `rejected` / `timed_out` / `needs_manual_review` | Human outcome, expired review, or automatic-review limit |
| Runs | `running` / `awaiting_review` | Batch remains active |
| Runs | `blocked` | Reconciliation is required |
| Runs | `completed` / `partial` / `skipped_overlap` | Terminal batch state |

The dispatcher compares the sheet headers with the exact arrays in [SHEET_SCHEMA.json](SHEET_SCHEMA.json). Do not reorder or append columns without updating the schema and every positional sheet write together.

Coordination uses Google Sheets named ranges for dispatch commits, worker claims, image request slots, and controlled recovery. Rows alone are not a complete backup of that state.

## Failure investigation

1. Locate the topic row and its `worker_execution_id`, `batch_id`, `stage`, `failure_class`, and `last_error`.
2. Open that worker execution in n8n and follow the child execution for the failed stage.
3. Compare the stored ownership, review token, WordPress ID/URL, and media checkpoint before considering a retry.
4. For an uncertain WordPress write, inspect WordPress before repeating it. For saved paid image output, use an appropriate recovery path rather than creating the images again.
5. Reconcile the ledger with the actual outcome before clearing a blocked state.

Failure classes include `auth`, `configuration`, `quota`, `ambiguous_write`, and `article`. The dispatcher treats the first four as batch-wide blockers. A stalled lease or overdue review is also a reconciliation condition; it is not proof that a worker may safely be duplicated.

## Image repair behavior captured here

- Image-plan parsing handles supported minor formatting defects and rejects incomplete or duplicate image roles.
- Generation responses retain article state.
- The image package checks consistent topic, run, and revision ownership across all four images.
- Terminal image failures propagate to the parent instead of returning an empty successful result.
- Saved generation responses and already packaged images have reuse paths.
- Main-worker recovery requires a matching reservation and prior execution, and uses a single-use recovery claim. It is not a general bypass of ownership checks.

## Restore boundaries

The manifest records source versions and whether they were active at export. Importing this repository does not restore Google credentials, Telegram approval callbacks, n8n execution history, sheet rows/named ranges, media binaries, WordPress posts, or WordPress custom-plugin code.

All seven source workflows were inactive when captured. Activation is a separate operational step.

The proposed create/update operation and existing-post PUT route are future work. This snapshot implements the existing draft-creation pipeline only.
