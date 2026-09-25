# Theia AI (agents, variables, tools, chat UI)
Packages:
- `@theia/ai-core`: Agent, prompts, variables, tools, LanguageModel.
- `@theia/ai-chat`: ChatAgent, request/response models.
- `@theia/ai-chat-ui`: renderers, input config.

The API evolves quickly. Grep the installed package `src/` for the exact name before writing code. Reference implementations live in the Theia repo: CommandChatAgent, ModeChatAgent, AskAndContinueChatAgent, TodayVariableContribution, FileVariableContribution, FileContentFunction, Claude Code slash commands.

## Agents
- **Agent**: an injectable service with a use-case-specific API, called by widgets, editors or menus. It builds prompts, gets an LLM, and returns results or triggers actions. Start with one agent per use case; agents may delegate to or chain other agents.
- **ChatAgent**: an Agent that is invoked from the default chat UI as `@<id>`.
```ts
export const myPrompt: BasePromptFragment = { id: 'my-agent-system', template: `You ... {{my-var}} ~{myTool} {{productName}}` };

@injectable()
export class MyChatAgent extends AbstractStreamParsingChatAgent {
    id = 'My'; name = 'My'; description = 'Does X';
    languageModelRequirements: LanguageModelRequirement[] = [{ purpose: 'chat', identifier: 'default/universal' }];
    protected defaultLanguageModelPurpose = 'chat';
    override prompts = [{ id: myPrompt.id, defaultVariant: myPrompt }];   // enables the prompt editor + auto variable resolution
    protected override systemPromptId = myPrompt.id;
    modes = [{ id: 'concise', name: 'Concise' }, { id: 'detailed', name: 'Detailed' }]; // optional; UI selector + Ctrl+M
    // in invoke(request: MutableChatRequestModel): request.request.modeId
}
// frontend module
bind(MyChatAgent).toSelf().inSingletonScope();
bind(Agent).toService(MyChatAgent);
bind(ChatAgent).toService(MyChatAgent);
```
- The doc omits `@injectable()` and the `toSelf` binding. Both are required for `toService` to work.
- Prompt fragments are editable at runtime, so iterate on prompts without rebuilding. Prefer structured output when the model supports it.

## Prompt syntax
- `{{var}}` or `{{var:arg}}` inserts a variable.
- `~{toolId}` exposes a tool function.
- `#var` / `#var:arg` references a variable in user chat.
- `{{capability:fragment-id default on|off}}` declares a toggleable chip in the chat input. It defaults to off when `default` is omitted. When enabled, the fragment is inlined into the system message; when disabled it resolves to nothing. The fragment may contain instructions, variables, `~{tool}`, or `~{agentDelegation}`. Chip state is persisted per session.
  - Chip label and tooltip come from YAML frontmatter (`name:`, `description:`) in the `.prompttemplate` file or in the template string. The frontmatter is stripped before the prompt is sent.
- `{{productName}}` resolves to the app's `applicationName`. Use it for white-label-safe prompts.
- `{{contextSummary}}` / `{{contextDetails}}` (`#contextSummary` / `#contextDetails` in user chat) insert the attached chat context as a list or in full. The tools `~{context_ListChatContext}` and `~{context_ResolveChatContext}` fetch it on demand. If the context UI is shown, the agent must actually use the context.

## Variables
**Agent-specific**: the agent resolves them itself.
```ts
const prompt = await this.promptService.getPrompt(myPrompt.id, { 'my-var': data });
this.agentSpecificVariables = [{ name: 'my-var', description: '...', usedInPrompt: true }]; // in constructor; declared for UI visibility
```
**Global**: usable by all agents and in chat as `#name`.
```ts
export const TODAY: AIVariable = { id: 'today-provider', name: 'today', description: '...', args: [{ name: 'iso', description: '...' }] };
@injectable()
export class TodayVariable implements AIVariableContribution, AIVariableResolver {
    registerVariables(s: AIVariableService): void { s.registerResolver(TODAY, this); }
    canResolve(r: AIVariableResolutionRequest): number { return r.variable.name === TODAY.name ? 1 : 0; }
    async resolve(r: AIVariableResolutionRequest, ctx: AIVariableContext): Promise<ResolvedAIVariable | undefined> {
        return r.variable.name === TODAY.name ? { variable: r.variable, value: new Date().toISOString() } : undefined;
    }
}
bind(AIVariableContribution).to(TodayVariable).inSingletonScope();
```
**Context variables**: add `isContextVariable: true`, plus `label` and `iconClasses: codiconArray('file')` on the `AIVariable`.
- The resolver returns a `ResolvedAIContextVariable`:
  - `value` is inserted inline (e.g. a relative path).
  - `contextValue` (e.g. the file content) goes to `ChatRequestModel.context`.
- For the frontend UX, add a separate `FrontendVariableContribution.registerVariables(s: FrontendVariableService)` that calls:
  - `s.registerArgumentPicker(VAR, () => Promise<string | undefined>)` (quick pick, typically `QuickInputService`)
  - `s.registerArgumentCompletionProvider(VAR, (model, position) => monaco CompletionItem[])`
  - `s.registerDropHandler((e: DragEvent, ctx) => Promise<AIVariableDropResult | undefined>)`, which returns `{ variables, text }`
- Bind the frontend contribution as another `AIVariableContribution`.
- For the chip label, add a `LabelProviderContribution`: `bind(X).toSelf().inSingletonScope(); bind(LabelProviderContribution).toService(X)`.
- To hide the context UI: `rebind(AIChatInputConfiguration).toConstantValue({ showContext: false, showPinnedAgent: true })`.

## Tool functions
The LLM decides whether to call them. Use them for retrieval or actions, including edits.
```ts
@injectable()
export class FileContentTool implements ToolProvider {
    static ID = 'getFileContent';
    getTool(): ToolRequest {
        return {
            id: FileContentTool.ID, name: FileContentTool.ID, description: 'Get file content',
            parameters: { type: 'object', properties: { file: { type: 'string', description: 'Workspace path' } }, required: ['file'] },
            handler: (argString: string) => this.read(JSON.parse(argString).file),  // args arrive as a JSON string
        };
    }
}
bind(ToolProvider).to(FileContentTool);
```
- Prototype tools without code using `@theia/ai-tool-sketchpad`. Add it to the app's dependencies. Tools are defined in the "AI Tool Sketchpad" view with a fixed return or "Ask At Runtime", saved to `sketchedTools.yml`, referenced as `~{name}`, and hot-reloaded.

## Slash commands
They are prompt fragments with command metadata. Register via `PromptService.addBuiltInPromptFragment`, e.g. in `onStart`:
```ts
this.promptService.addBuiltInPromptFragment({
    id: 'my-compare', template: 'Compare $1 and $2.', isCommand: true,
    commandName: 'compare', commandDescription: 'Compare two things', commandArgumentHint: 'a b', commandAgents: ['My'],
});
```
- `$ARGUMENTS` = all arguments as one string; `$1`, `$2`, ... = positional arguments (quotes group words).
- `commandAgents` restricts which agents offer the command.

## Custom response rendering
Flow: LLM text → agent parses → `ChatResponseContent` parts → the chat UI picks the renderer with the highest `canHandle`.
- Parse whole responses in the agent (e.g. JSON → `new CommandChatResponseContentImpl(cmd)`).
- For inline parts, push a matcher in `@postConstruct`:
  `this.contentMatchers.push({ start: /^<question>.*$/m, end: /^<\/question>$/m, contentFactory: (content, request) => new QuestionResponseContentImpl(...) })`.
- Renderer:
  - `canHandle(c: ChatResponseContent): number`, returning > 0 to claim a part and -1 to decline.
  - `render(c): ReactNode`.
  - Register with `bind(ChatResponsePartRenderer).to(MyRenderer).inSingletonScope()`.
- A response can mix many part types.

## Response state
- State is `isComplete`, `isWaitingForInput`, `isError`. If all are false, the UI shows "Generating..." and only allows cancel.
- Set it with `request.response.complete()`, `.cancel()`, `.error(e)`, or `.waitForInput()`.
- Progress messages:
  - `const m = request.response.addProgressMessage({ content, show: 'untilFirstContent' | 'whileIncomplete' | 'forever' })`
  - later `request.response.updateProgressMessage({ ...m, status: 'completed' })`
- Interactive flows: override `onResponseComplete(request)`. If input is still needed, call `waitForInput()`; otherwise call `super.onResponseComplete(request)`. Pair this with a question renderer (see AskAndContinueChatAgent).

## Reasoning
- The model description declares `reasoningSupport` (levels `off | minimal | low | medium | high | auto` plus a default). The chat input then shows a selector.
- The request carries `request.reasoning?.level` (on `UserRequest` / `CommonChatSessionSettings`).
- Providers translate the level to the native API in `getSettings()` and add `reasoningApi` where applicable.
- `ThinkingModeSettings` / `thinkingMode` were **removed**; use `ReasoningSettings { level }`.
- Defaults come from the preference `ai-features.reasoning.defaults` (keyed by provider, model or agent).
- Resolution order: session override → persisted per-agent selection → most specific preference → model default.

## Custom LLM provider
- Implement `LanguageModel` and register it with `languageModelRegistry.addLanguageModels([new MyModel()])`.
- Built-in providers: OpenAI-compatible, Hugging Face, Ollama, Llamafile.
- Agents request models through `languageModelRequirements` (a purpose + identifier), never by concrete class.
- (The source doc was truncated here. Remaining topics not yet captured: provider configuration, Copilot, Change Sets, Chat Suggestions, Chat Banners, AI Preferences.)
