# Standard events

This catalog defines 43 emission boundaries. Parameter structure is normative in [the schema](schema.md); [the protocol](protocol.md) defines delivery and semantic obligations. Profiles group related events. An integration enables only the events and waiting behavior its adapter can faithfully implement. Even the core group does not require every native harness to expose every event.

Session scope requires sessionId and permits optional runId and agentId. Optional-session scope permits all three, but runId or agentId requires sessionId. Client scope carries no session fields. The required and optional field columns below exclude those shared scope fields. Types name schema definitions; each parameter definition is available under the schema's definitions object.

| Method | Profile | Scope | Emission boundary | Required fields | Optional fields | Parameter definition |
| --- | --- | --- | --- | --- | --- | --- |
| `session.started` | core | session | The session has started or resumed. | None | `reason: string`, `parentSessionId: string` | `SessionStartedParams` |
| `session.ended` | core | session | The session has closed; cleanup cannot veto closure. | None | `reason: string` | `SessionEndedParams` |
| `input.submitted` | core | session | New input is available before the agent consumes it. | `content: ContentBlock[]` | `structuredContent: JsonValue` | `InputSubmittedParams` |
| `run.started` | core | session | One invocation of agent work has begun; resumesRunId names a stable prior invocation when the harness exposes that relation. | None | `input: InputPayload`, `resumesRunId: string` | `RunStartedParams` |
| `run.finishing` | core | session | The agent proposes stopping before the harness accepts completion. | None | `output: Payload` | `RunFinishingParams` |
| `run.finished` | core | session | The invocation has reached its terminal outcome, including interruption awaiting a later invocation. | `outcome: RunCompletion` | None | `RunFinishedParams` |
| `tool.requested` | core | session | A complete proposed invocation exists before execution and potentially before permission checks. | `call: ProposedToolCall` | None | `ToolRequestedParams` |
| `tool.started` | core | session | Tool execution has actually begun. | `call: ObservedToolCall` | None | `ToolStartedParams` |
| `tool.finished` | core | session | The invocation is terminal; unknown reports a terminal call whose outcome the harness did not expose. | `call: ObservedToolCall`, `outcome: ToolCompletion` | None | `ToolFinishedParams` |
| `context.compaction.requested` | context | session | Compaction is about to begin. | None | `reason: string` | `ContextCompactionRequestedParams` |
| `context.compaction.finished` | context | session | Compaction reached a terminal outcome; output is exposed compaction output, not proof of replacement history. | `outcome: Completion` | None | `ContextCompactionFinishedParams` |
| `context.source.loaded` | context | session | The identified source material actually entered the agent context. | `source: Source` | `payload: Payload` | `ContextSourceLoadedParams` |
| `catalog.changed` | catalog | optional-session | The exposed tool or skill catalog changed; entries are the complete current snapshot. | `catalog: "tools" \| "skills"`, `entries: ToolDefinition[] \| SkillDefinition[]` | None | `CatalogChangedParams` |
| `model.requested` | model | session | A model request is about to be sent; an exposed input view declares completeness. | None | `model: ModelRef`, `input: ModelInput`, `tools: ToolDefinition[]`, `modelCallId: string` | `ModelRequestedParams` |
| `model.finished` | model | session | One model invocation reached a terminal outcome; exposed output may contain complete proposed tool calls. | `outcome: ModelCompletion` | `model: ModelRef`, `modelCallId: string`, `usage: Usage` | `ModelFinishedParams` |
| `permission.requested` | permission | session | A permission decision is needed for the described operation. | `operation: Operation`, `options: PermissionOption[]` | `requestId: string` | `PermissionRequestedParams` |
| `permission.resolved` | permission | session | The permission decision is known; this event does not itself grant authority. | `operation: Operation`, `decision: "allowed" \| "denied" \| "cancelled"` | `requestId: string`, `optionId: string`, `reason: string` | `PermissionResolvedParams` |
| `interaction.requested` | interaction | session | The harness is requesting a question, form answer, or external URL interaction. | `interaction: Interaction` | `requestId: string` | `InteractionRequestedParams` |
| `interaction.resolved` | interaction | session | An interaction was answered, declined, or cancelled. | `interaction: Interaction`, `response: Answer \| UrlAnswer` | `requestId: string` | `InteractionResolvedParams` |
| `agent.requested` | agent | session | The parent is about to delegate an invocation to a child agent. | `agent: AgentRef`, `input: InputPayload` | `childRunId: string` | `AgentRequestedParams` |
| `agent.started` | agent | session | A delegated child invocation has begun; Scope identifies the parent. | `agent: AgentRef` | `childRunId: string`, `input: InputPayload` | `AgentStartedParams` |
| `agent.finished` | agent | session | A delegated child invocation reached a terminal outcome, including interruption awaiting a later invocation. | `agent: AgentRef`, `outcome: RunCompletion` | `childRunId: string` | `AgentFinishedParams` |
| `handoff.requested` | handoff | session | A transfer of responsibility between agents is proposed. | `from: AgentRef`, `to: AgentRef` | `input: InputPayload` | `HandoffRequestedParams` |
| `handoff.finished` | handoff | session | The transfer attempt reached a terminal outcome; success does not mean destination work finished. | `from: AgentRef`, `to: AgentRef`, `outcome: Completion` | None | `HandoffFinishedParams` |
| `message.emitting` | message | session | A message is about to be published to its recipient. | `message: Message` | `recipient: AgentRef` | `MessageEmittingParams` |
| `message.emitted` | message | session | The harness committed publication to the recipient; this is not proof of reading. | `message: Message` | `recipient: AgentRef` | `MessageEmittedParams` |
| `run.cancellation.requested` | run-control | session | Cancellation was requested; run.finished reports the actual terminal result. | None | `reason: string` | `RunCancellationRequestedParams` |
| `run.paused` | run-control | session | The same invocation paused without finishing; a terminal interrupted run does not map here. | None | `reason: string`, `interactionId: string` | `RunPausedParams` |
| `run.resumed` | run-control | session | The same paused invocation resumed; a new invocation uses run.started and may name resumesRunId. | None | `reason: string`, `interactionId: string` | `RunResumedParams` |
| `tool.batch.finished` | tool-batch | session | A harness-managed batch reached terminal outcomes for all listed calls. | `calls: ToolFinishedItem[]` | `batchId: string` | `ToolBatchFinishedParams` |
| `plan.updated` | plan | session | The plan snapshot changed; an empty snapshot clears it. | `steps: PlanStep[]` | None | `PlanUpdatedParams` |
| `task.created` | task | session | A tracked task was created in a nonterminal state. | `task: TaskSnapshot` | None | `TaskCreatedParams` |
| `task.updated` | task | session | A tracked task snapshot changed while nonterminal. | `task: TaskSnapshot` | None | `TaskUpdatedParams` |
| `task.finished` | task | session | A tracked task reached a terminal outcome. | `task: TaskRef`, `outcome: Completion` | None | `TaskFinishedParams` |
| `workspace.changed` | environment | optional-session | The workspace was opened, closed, or updated. | `workspace: WorkspaceRef`, `change: "opened" \| "closed" \| "updated"` | None | `WorkspaceChangedParams` |
| `resource.changed` | environment | optional-session | An identified resource changed; renamed requires its previous URI. | `resource: ResourceRef`, `change: "created" \| "modified" \| "deleted" \| "renamed"` | `previousUri: URI` | `ResourceChangedParams` |
| `configuration.changed` | environment | optional-session | Named nonsecret settings changed; omitted values are not exposed. | `changes: SettingChange[]` | None | `ConfigurationChangedParams` |
| `model.changed` | environment | optional-session | The selected model configuration changed; individual requests still report actual model identity when known. | `model: ModelRef` | `previousModel: ModelRef` | `ModelChangedParams` |
| `notification.created` | runtime | optional-session | A runtime notice was created. | `level: "info" \| "warning" \| "error"`, `message: string` | `code: string` | `NotificationCreatedParams` |
| `error.occurred` | runtime | optional-session | A runtime fault occurred outside a represented operation completion. | `error: Failure` | `operation: Operation` | `ErrorOccurredParams` |
| `client.registered` | client | client | The user registered this server and consented to the listed methods. | `client: ClientInfo`, `events: string[]` | None | `ClientRegisteredParams` |
| `client.updated` | client | client | The user changed the consented methods. | `client: ClientInfo`, `events: string[]` | None | `ClientUpdatedParams` |
| `client.unregistered` | client | client | The user removed this server; no further events follow. | `client: ClientInfo` | None | `ClientUnregisteredParams` |

Required fields can have additional conditional constraints. Examples include nonempty submitted input, catalog-specific entries, interaction-specific answers and the previous URI on a resource rename. Consult the referenced parameter schema.

Completion distinguishes success, failure and cancellation. ToolCompletion adds denial and unknown outcome. RunCompletion adds terminal interruption and applies to both run.finished and parent-owned agent.finished. ModelCompletion carries ModelOutput so tool-call-only model responses remain typed.

context.source.loaded classifies ingested material as skill, instructions, memory or resource. catalog.changed classifies available definitions as tools or skills. Catalog snapshots can arrive out of order and cannot establish a required precondition.
