---
name: theia-dev
description: Eclipse Theia extension architecture — InversifyJS DI, service symbol+interface pattern, rebind, contribution points/providers, command/menu/keybinding contributions, widgets (ReactWidget, TreeWidget (TreeModelImpl, TreeImpl, createTreeContainer, decorators, DnD), WidgetFactory, AbstractViewContribution, FrontendApplicationContribution lifecycle (initializeLayout, onStart, onStop), React 19 shared imports), custom editors (Navigatable, NavigatableWidgetOpenHandler, OpenHandler, child-container options), label providers (LabelProviderContribution, FileTreeLabelProvider, icons), breadcrumbs (BreadcrumbsContribution, NavigatableWidget URI), enhanced tab preview (TabBarRenderer, TabBarRendererFactory), MessageService (toasts, actions, progress), property view (PropertyDataService, PropertyViewWidgetProvider), preferences (PreferenceSchema, scopes, proxy, session/backend), perspectives (PerspectiveContribution), Theia AI (agents, chat agents, variables, tools, renderers), BackendApplicationContribution (initialize, Express endpoints), dynamic toolbar (ToolbarDefaultsFactory), i18n (nls.localize, toLocalizedCommand, LocalizationContribution, TextReplacementContribution), tasks (TaskContribution provider/resolver, TaskRunnerContribution), telemetry (TelemetryService, TelemetrySink, consent), contribution filters (FilterContribution, ContributionFilterRegistry), Theia-only API for VS Code extensions (appName gate, custom API/commands), frontend/backend JSON-RPC split. Use when writing or reviewing code in a repo depending on @theia/*, or files like *-frontend-module.ts / *-backend-module.ts.
---

# Theia Dev

## Output discipline
- No preamble, no recap, no restating the request. Code + ≤2 lines of rationale.
- Read only files you will edit or whose types you consume. Never scan node_modules, lib/, gen-*, src-gen/.
- Look up a Theia API by grepping `node_modules/@theia/<pkg>/src` for the exact symbol, not by browsing. Installed version wins over memory.

## Architecture
- Two processes, frontend and backend, built from one source. They talk via JSON-RPC over WebSocket or REST over HTTP.
- Electron runs both locally. Browser/remote runs the backend on a server host. Never assume the two processes share a machine or filesystem.
- Each process has its own DI container. A binding in a frontend module does not exist in the backend, and vice versa.
- Frontend startup loads every extension's DI modules, then calls `FrontendApplication.start()`. Hook in with `FrontendApplicationContribution`.
- Backend startup loads every extension's DI modules, then calls `BackendApplication.start(port)`. The backend is an express server that also serves the frontend bundle by default. Hook in with `BackendApplicationContribution` (e.g. `configure(app)` for routes).

## Package layout (platform folders under `src/`)
| folder | runtime | may use |
|---|---|---|
| `common/` | any | no DOM, no Node; interfaces, symbols, RPC protocol |
| `browser/` | frontend | DOM; no Node; `*-frontend-module.ts` |
| `electron-browser/` | frontend (Electron renderer) | DOM + Electron renderer APIs |
| `node/` | backend | Node; no DOM; `*-backend-module.ts` |
| `node-electron/` | backend (Electron only) | Node + Electron |
- Register modules in `package.json` → `theiaExtensions: [{ frontend, backend, frontendElectron?, backendElectron? }]`. Each module default-exports a `ContainerModule`.
- Import only from the same folder or `common`. `browser` must never import `node` or the reverse. Electron folders may import their non-Electron counterpart (`electron-browser` → `browser`, `node-electron` → `node`).

## DI rules (InversifyJS)
- Every DI-created class needs `@injectable()`. Injection works only on container-created instances.
- Prefer field injection: `@inject(X) protected readonly x: X;`. Keep fields `protected`, not `private`, so subclasses can override.
- `@postConstruct() protected init(): void` takes no parameters. Inject via fields, then use them in `init`. Async init: call an async helper and keep the promise (e.g. `readonly ready = new Deferred<void>()`).
- Constructor injection is allowed: `constructor(@inject(X) protected readonly x: X) {}`.

## Public (replaceable) service = symbol + interface + default impl
```ts
export const GreetingRegistry = Symbol('GreetingRegistry');
export interface GreetingRegistry { register(g: Greeting): void; }
@injectable() export class GreetingRegistryImpl implements GreetingRegistry { register(g: Greeting): void {} }
// module
bind(GreetingRegistryImpl).toSelf().inSingletonScope();
bind(GreetingRegistry).toService(GreetingRegistryImpl);
// adopter
rebind(GreetingRegistry).to(MyRegistry);
```
- Don't use a class as both token and type for public services. A class with private or protected members is typed nominally, which blocks independent rebinds and forces test doubles to subclass the real impl.
- A plain class token is fine for internal-only services.
- Use `toService` (alias) rather than a second `to(...)`, so the symbol and the class resolve to the same singleton.

## Contributing to contribution points
```ts
bind(MyCmds).toSelf().inSingletonScope();
bind(CommandContribution).toService(MyCmds);
bind(MenuContribution).toService(MyCmds);   // one class may implement several
```
- All three contribution types below are bound in the frontend module.

## Commands, menus, keybindings
```ts
export const HelloCmd: Command = { id: 'hello.say', label: 'Say Hello' }; // id required, label optional

@injectable()
export class HelloContribution implements CommandContribution, MenuContribution, KeybindingContribution {
    @inject(MessageService) protected readonly messages: MessageService;
    registerCommands(r: CommandRegistry): void {
        r.registerCommand(HelloCmd, { execute: () => this.messages.info('Hello'), isEnabled: () => true });
    }
    registerMenus(m: MenuModelRegistry): void {
        m.registerMenuAction(CommonMenus.EDIT_FIND, { commandId: HelloCmd.id, label: HelloCmd.label });
    }
    registerKeybindings(k: KeybindingRegistry): void {
        k.registerKeybinding({ command: HelloCmd.id, keybinding: 'ctrlcmd+alt+h', when: 'editorFocus' });
    }
}
```
- **Command** = `id` + optional `label` (and icon/category). Triggered from the palette, a keybinding, a menu, or `commandRegistry.executeCommand(id, ...args)`.
- **CommandHandler** = `execute(...args)` with optional `isEnabled`, `isVisible`, `isToggled`.
  - `registerHandler(id, handler)` adds more handlers for the same command. When the command runs, the first handler whose `isEnabled` returns true executes. Make handler conditions mutually exclusive.
  - `isVisible` of the active handler controls whether the command shows in menus, toolbars and the command palette.
  - `isToggled` controls the checked state of connected menu items.
- **CommandRegistry**: inject it anywhere to execute commands, list all registered commands, or read recently executed ones.
- **Menus**: `registerMenuAction(menuPath: MenuPath, { commandId, label?, order?, icon? })`. A `MenuPath` is a `string[]`, e.g. `CommonMenus.FILE`, `EDIT`, `EDIT_FIND`, `VIEW`, `HELP`.
  - Custom menu: `registerSubmenu([...MAIN_MENU_BAR, 'my-menu'], 'My Menu')`, then use that path array as the target of later actions.
  - `order` is a string compared lexicographically.
- **Keybindings**: `{ keybinding, command, when? }`.
  - `when` uses VS Code context-key syntax (`&&`, `||`, `!`).
  - Use `ctrlcmd`, not `ctrl`, so the shortcut maps to Cmd on macOS and Ctrl elsewhere (`KeyModifier.CtrlCmd`, formerly `Modifier.M1`).
  - Key names come from `Key` in `@theia/core/lib/browser/keys`.
- The doc example's keybinding class has no `@injectable()`. Always add it, since DI can't create the class without it.

## Widgets (views/editors)
Three parts: the widget, a `WidgetFactory`, and a view contribution. All live in `browser/`.
- Base classes:
  - `ReactWidget` for most custom UI (the default choice).
  - `TreeWidget` for tree views. For any tree work (nodes, model, lazy children, labels/icons, decorations, context menu, drag and drop), read [tree.md](tree.md) first.
  - `BaseWidget` for no-React UI.
- Lifecycle hooks come from Phosphor/Lumino (`onUpdateRequest`, `onResize`, `onActivateRequest`, `onAfterAttach`, `onBeforeDetach`, ...). Override them and call `super`.
```tsx
@injectable()
export class MyWidget extends ReactWidget {
    static readonly ID = 'my:widget';
    static readonly LABEL = 'My Widget';
    @inject(MessageService) protected readonly messages: MessageService;
    @postConstruct()
    protected init(): void {
        this.id = MyWidget.ID;                      // unique; WidgetManager key
        this.title.label = MyWidget.LABEL;          // tab text
        this.title.caption = MyWidget.LABEL;        // tab hover
        this.title.closable = true;
        this.title.iconClass = codicon('window');   // or 'fa fa-...'
        this.update();                              // schedules render()
    }
    protected render(): React.ReactNode {
        return <button className='theia-button secondary' onClick={() => this.messages.info('Hi')}>Hi</button>;
    }
}
```
- Re-render by calling `this.update()` after state changes. Keep state in widget fields, not only in React state, because `render()` rebuilds the tree from them.

### React (Theia ≥ 1.75)
- React 19 only (a peer dependency of @theia/core). React 18 is unsupported.
- Import only from `@theia/core/shared/react` and `@theia/core/shared/react-dom` (`/client`, `/server`). Never import `react` directly: a second React instance breaks hooks and context.
- Optional tsconfig: `"jsx": "react-jsx", "jsxImportSource": "@theia/core/shared/react"`. With it, drop imports that exist only to make JSX compile. Classic `"jsx": "react"` still works.

### Factory (frontend module)
```ts
bind(MyWidget).toSelf();
bind(WidgetFactory).toDynamicValue(ctx => ({
    id: MyWidget.ID,
    createWidget: () => ctx.container.get<MyWidget>(MyWidget),
})).inSingletonScope();
```
- `WidgetManager.getOrCreateWidget(factoryId, options?)` returns the cached instance or creates one. Instances are cached per factory ID + options.
- `createWidget(options)` receives those options. Use them to pass parameters, e.g. through a child container.

### View contribution
```ts
export const MyWidgetCommand: Command = { id: 'my-widget:toggle', label: 'Toggle My Widget' };
@injectable()
export class MyWidgetContribution extends AbstractViewContribution<MyWidget> {
    constructor() {
        super({ widgetId: MyWidget.ID, widgetName: MyWidget.LABEL,
                defaultWidgetOptions: { area: 'left' }, toggleCommandId: MyWidgetCommand.id });
    }
    override registerCommands(c: CommandRegistry): void {
        c.registerCommand(MyWidgetCommand, { execute: () => this.openView({ activate: false, reveal: true }) });
    }
}
// module
bindViewContribution(bind, MyWidgetContribution);
bind(FrontendApplicationContribution).toService(MyWidgetContribution); // only if it overrides onStart/initializeLayout
```
- The base class adds the view to the View menu and the command palette.
- If `registerCommands` is not overridden, the base class already registers `toggleCommandId` → `toggleView()`.
- `area`: `'left' | 'right' | 'main' | 'bottom'`.
- Open programmatically with `openView({ activate, reveal })`.
- The doc example omits `@injectable()`. Always add it.

### Frontend application lifecycle (`FrontendApplicationContribution`)
Uses: open or arrange views, register listeners, add status bar items, customize the shell, or persist data on shutdown (e.g. via `StorageService`).

| Hook | When it runs | Use for |
|---|---|---|
| `initialize()` | first, before `configure` | early setup |
| `configure(app)` | every start, before the shell is attached and before menus exist | installing listeners (e.g. preference changes) |
| `onStart(app)` | every start, shell not yet ready | adding widgets, or wait for readiness (below) |
| `initializeLayout(app)` | **only if no stored layout exists** | the default layout on first run; it never overrides the user's saved arrangement |
| `onDidInitializeLayout(app)` | after the layout, every start | follow-up work after the layout exists |
| `onWillStop(app)` / `onStop(app)` | at shutdown | a veto prompt, or persisting state (keep `onStop` synchronous) |
```ts
@injectable()
export class MyViewContribution extends AbstractViewContribution<MyWidget> implements FrontendApplicationContribution {
    @inject(FrontendApplicationStateService) protected readonly stateService: FrontendApplicationStateService;
    async initializeLayout(): Promise<void> { await this.openView(); }                   // first-run default
    onStart(): void { this.stateService.reachedState('ready').then(() => this.openView({ reveal: true })); } // every start
}
```
- The binding is `bind(FrontendApplicationContribution).toService(MyViewContribution)`.
- Choose `initializeLayout` or `onStart` for opening a view, not both.
- Don't await `reachedState` in `onStart`: that would block startup.

## Custom editors (open a widget for a file extension)
Three parts: a `Navigatable` widget with injected options, an open handler, and a view contribution with `area: 'main'`.
```tsx
export const MyEditorOptions = Symbol('MyEditorOptions');
export interface MyEditorOptions extends NavigatableWidgetOptions { filePath: string; }

@injectable()
export class MyEditor extends ReactWidget implements Navigatable {
    static readonly ID = 'my:editor';
    static readonly LABEL = 'My Editor';
    @inject(MyEditorOptions) protected readonly options: MyEditorOptions;
    protected uri: URI;
    getResourceUri(): URI | undefined { return this.uri; }
    createMoveToUri(resourceUri: URI): URI | undefined { return this.uri.withPath(resourceUri.path); }
    @postConstruct()
    protected init(): void {
        this.uri = new URI(this.options.filePath);
        this.id = `${MyEditor.ID}:${this.options.filePath}`; // unique per file
        this.title.label = this.uri.path.base;
        this.title.caption = MyEditor.LABEL;
        this.title.closable = true;
        this.node.tabIndex = 0;                               // focusable
        this.update();
    }
    protected render(): React.ReactNode { return <div />; }
}

@injectable()
export class MyOpenHandler extends NavigatableWidgetOpenHandler<MyEditor> {
    readonly id = MyEditor.ID;                                // must equal the WidgetFactory id
    canHandle(uri: URI): number { return uri.path.ext.toLowerCase() === '.myext' ? 500 : 0; }
    protected override createWidgetOptions(uri: URI, o?: WidgetOpenerOptions): MyEditorOptions {
        return { ...super.createWidgetOptions(uri, o), filePath: uri.path.toString() };
    }
}

// frontend module
bind(WidgetFactory).toDynamicValue(ctx => ({
    id: MyEditor.ID,
    createWidget: (options: MyEditorOptions) => {
        const child = ctx.container.createChild();
        child.bind(MyEditorOptions).toConstantValue(options);
        child.bind(MyEditor).toSelf();
        return child.get(MyEditor);
    },
}));
bindViewContribution(bind, MyEditorContribution);   // AbstractViewContribution, defaultWidgetOptions: { area: 'main' }
bind(MyOpenHandler).toSelf().inSingletonScope();
bind(OpenHandler).toService(MyOpenHandler);
```
- `Navigatable` ties the tab to its URI. This enables "close others", dirty-state tracking, and move/rename via `createMoveToUri`.
- `canHandle` returns a priority. The built-in text editor returns 1, so return a value > 1 to win (e.g. 500). Return 0 to decline. It may return `MaybePromise<number>`.
- Per-instance options: always use a child container. It keeps the options binding out of the global container and still injects other services into the widget. Don't `new` the widget.
- Widget `id` must be unique per file, not just per widget type.
- View contribution:
  - Define your own command that runs `this.openView({ activate: true })`, so running it always opens and focuses the editor.
  - Or skip the custom command and call `super.registerCommands(commands)` to get the default toggle command.
- Use `ReactWidget` for complex editors. Extend `BaseWidget` for no-React ones.

## Label providers (icon + text for tree nodes, tabs, view headers)
- `LabelProvider` asks every `LabelProviderContribution` and delegates to the one whose `canHandle` returns the highest priority.
- Built-in contributions exist for common types such as files. To customize a file type, extend `FileTreeLabelProvider` and override only what you need.
```ts
@injectable()
export class MyLabelProvider extends FileTreeLabelProvider {
    override canHandle(element: object): number {
        return FileStatNode.is(element) && element.uri.path.ext === '.my'
            ? super.canHandle(element) + 1   // beat the default file contribution
            : 0;                              // decline
    }
    override getIcon(): string { return 'my-icon'; }   // CSS class(es), e.g. codicon('star') or 'fa fa-star-o'
    override getName(node: FileStatNode): string { return super.getName(node) + ' (mine)'; }
    // getLongName(node): tooltip on editor tabs
}
// frontend module
bind(MyLabelProvider).toSelf().inSingletonScope();
bind(LabelProviderContribution).toService(MyLabelProvider);
```
- `canHandle` receives whatever element the caller renders: a `FileStatNode` in the file tree, a `URI` in editor tabs. Narrow with type guards before accessing fields.
- Custom icon: `getIcon` returns a CSS class. Define it with a size and a per-theme image:
```css
.my-icon { background-repeat: no-repeat; background-size: 12px; width: 13px; height: 13px; }
.light-plus .my-icon { background-image: url('./icon-dark.png'); }
.dark-plus  .my-icon { background-image: url('./icon-light.png'); }
```
- Import the CSS file from the frontend module (e.g. `import '../../src/browser/style/index.css'`).

## Breadcrumbs (main-area widgets; enabled by a preference)
- A widget declares where its content lives by implementing `NavigatableWidget.getResourceUri(): URI | undefined`.
- Every `BreadcrumbsContribution` that returns a non-empty array for that URI contributes crumbs. They are concatenated in **ascending `priority`** order; an empty array means "not mine".
- File URIs:
  - The filesystem contribution already shows the directory path, with a file-tree popup.
  - Return a `file://` URI to reuse it, and append your own crumbs with a higher priority (e.g. sections within the file).
  - To show only custom crumbs, use a non-file scheme, e.g. `resource://a/b/c`.
```ts
export const MyBreadcrumbType = Symbol('MyBreadcrumb');
@injectable()
export class MyBreadcrumbs implements BreadcrumbsContribution {
    readonly type = MyBreadcrumbType;
    readonly priority = 100;
    protected readonly onDidChangeBreadcrumbsEmitter = new Emitter<URI>();
    readonly onDidChangeBreadcrumbs: Event<URI> = this.onDidChangeBreadcrumbsEmitter.event; // fire(uri) to recompute
    computeBreadcrumbs(uri: URI): MaybePromise<Breadcrumb[]> {
        if (uri.scheme !== 'resource') { return []; }
        return uri.path.toString().split('/').filter(Boolean).map((seg, i) => ({
            id: `${i}:${seg}`, label: seg, longLabel: seg /* tooltip */, iconClass: codicon('folder'), type: MyBreadcrumbType,
        }));
    }
    async attachPopupContent(crumb: Breadcrumb, parent: HTMLElement): Promise<Disposable | undefined> {
        // runs on click; render anything into parent (a tree, buttons, ...). The returned Disposable is called when the popup closes.
        const h = document.createElement('h3'); h.textContent = crumb.label; parent.appendChild(h);   // no innerHTML with data (XSS)
        return undefined;
    }
}
bind(MyBreadcrumbs).toSelf().inSingletonScope();
bind(BreadcrumbsContribution).toService(MyBreadcrumbs);
```

## Enhanced tab bar preview (main/bottom horizontal tab bars)
- Off by default; enabled by the preference `window.tabbar.enhancedPreview`.
- The preview shows `title.label` and `title.caption`, so set a rich caption in the widget, e.g. `this.title.caption = 'Home > Theia > Getting Started'`.
- CSS hooks:
  - `.theia-hover` styles all hovers.
  - `.theia-hover.extended-tab-preview` styles only this preview (the default has `border-radius: 10px`).
  - Inner elements: `.theia-horizontal-tabBar-hover-div`, `-title` and `-caption`. For a fixed size, set a `width` on the div, and `max-width` plus `word-wrap: break-word` on the title and caption.
- To change the content, override `renderExtendedTabBarPreview` (it's an arrow-function **property**, not a method), then swap in your renderer via the factory:
```ts
export class MyTabBarRenderer extends TabBarRenderer {
    protected override renderExtendedTabBarPreview = (title: Title<Widget>): HTMLElement => {
        const box = document.createElement('div'); box.classList.add('theia-horizontal-tabBar-hover-div');
        const p = document.createElement('p'); p.classList.add('theia-horizontal-tabBar-hover-title'); p.textContent = title.label;
        box.append(p); return box;
    };
}
rebind(TabBarRendererFactory).toFactory(({ container }) => () => new MyTabBarRenderer(
    container.get(ContextMenuRenderer), container.get(TabBarDecoratorService), container.get(IconThemeService),
    container.get(SelectionService), container.get<CommandService>(CommandService), container.get<CorePreferences>(CorePreferences),
    container.get(HoverService)));
```
- Core already binds the factory, so use `rebind`. The doc's `bind` would create an ambiguous duplicate.
- The constructor arguments change between versions. Copy them from core's own `TabBarRendererFactory` binding in `@theia/core/src/browser/frontend-application-module.ts`.

## Message service (toasts, actions, progress)
`@inject(MessageService) protected readonly messageService: MessageService` (from `@theia/core`). Toasts appear bottom-right and stay until the user closes them. To change how messages are displayed, rebind `MessageClient`.
```ts
this.messageService.info('Saved');                       // also warn(), error()
this.messageService.info('Saved', { timeout: 3000 });    // auto-close after ms
const action = await this.messageService.error('Failed', 'Retry', 'Cancel');
if (action === 'Retry') { /* ... */ }                    // resolves to the clicked action string, else undefined
const progress = await this.messageService.showProgress({ text: 'Indexing' });
try {
    progress.report({ message: 'Step 1', work: { done: 10, total: 100 } });
    // ...
} finally { progress.cancel(); }                         // cancel() also marks completion
```

## Property view (`@theia/property-view`)
- A global view in the bottom dock (View → Properties, Shift+Alt+P) that shows details for the global selection.
- Built-in content: `EmptyPropertyViewWidget` ("No properties available") and `ResourcePropertyViewWidget` (a file from the explorer or the active editor).
- To extend it, provide a data service (selection → data) plus a widget provider (selection → content widget). For each, the highest `canHandle`/`canHandleSelection` wins.
```ts
@injectable()
export class MyDataService implements PropertyDataService {
    readonly id = 'my'; readonly label = 'MyDataService';
    canHandleSelection(sel: Object | undefined): number { return isMine(sel) ? 1 : 0; }
    async providePropertyData(sel: Object | undefined): Promise<MyData | undefined> {
        return isMine(sel) ? { /* ... */ } : undefined;
    }
}

export class MyPropsWidget extends ReactWidget implements PropertyViewContentWidget {
    static readonly ID = 'my-property-view';
    protected data: MyData | undefined;
    constructor() { super(); this.id = MyPropsWidget.ID; this.title.label = 'My'; this.title.closable = false; this.node.tabIndex = 0; }
    updatePropertyViewContent(service?: PropertyDataService, sel?: Object): void {
        if (!service) { this.data = undefined; this.update(); return; }
        service.providePropertyData(sel).then(d => { this.data = d; this.update(); });  // update() after the data arrives
    }
    protected render(): React.ReactNode { return this.data ? <div>{/* ... */}</div> : undefined; }
}

@injectable()
export class MyWidgetProvider extends DefaultPropertyViewWidgetProvider {
    override readonly id = 'my'; override readonly label = 'MyWidgetProvider';
    protected readonly widget = new MyPropsWidget();
    override canHandle(sel: Object | undefined): number { return isMine(sel) ? 1 : 0; }
    override provideWidget(): Promise<MyPropsWidget> { return Promise.resolve(this.widget); }
    override updateContentWidget(sel: Object | undefined): void {
        this.getPropertyDataService(sel).then(s => this.widget.updatePropertyViewContent(s, sel));
    }
}
// frontend module
bind(PropertyDataService).to(MyDataService).inSingletonScope();
bind(PropertyViewWidgetProvider).to(MyWidgetProvider).inSingletonScope();
```
- Explorer selections are arrays: check with `Array.isArray(sel) && FileSelection.is(sel[0])`; then `sel[0].fileStat` gives the resource and `isDirectory`.
- Fixed from the doc's example:
  - Its widget called `update()` before the data promise resolved.
  - Its `render` dereferenced undefined data.
  - Its `Promise.reject()` for unmatched selections was replaced with returning `undefined`.

## Preferences
Import from `@theia/core/lib/common/preferences`. Put schema + binding in `common/` so both frontend and backend modules can call the same `bindXPreferences(bind)`.

### Scopes and files
- Resolution order, most specific first: `Session` > `Folder` > `Workspace` > `User` > `Default`. The first scope that has a value wins.
- Files: User `~/.theia/settings.json`, Workspace `<root>/.theia/settings.json`, Folder `<folder>/.theia/settings.json`. Multi-root workspaces can also store them in the workspace file.
- The schema `scope` field sets the **most specific scope where the preference may be set**. It does not change how values resolve.
  - `User` → Default + User only (themes, global options).
  - `Workspace` → adds Workspace (project settings).
  - `Folder` → all scopes. This is also the default when `scope` is omitted.
  - Pick the most restrictive scope that fits.

### Contribute
```ts
export const myPreferenceSchema: PreferenceSchema = {
    properties: {
        'myExt.enabled':  { type: 'boolean', default: true, description: '...', scope: PreferenceScope.User },
        'myExt.timeout':  { type: 'number', default: 5000, minimum: 100, description: '...', scope: PreferenceScope.Workspace },
        'myExt.logLevel': { type: 'string', enum: ['error', 'warn', 'info', 'debug'], enumDescriptions: [...], default: 'info', description: '...' },
        'myExt.indent':   { type: 'string', default: 'x', description: '...', overridable: true }, // allows "[typescript]": {...}
    },
};
export interface MyConfiguration {            // only needed for a proxy; must mirror the schema
    'myExt.enabled': boolean; 'myExt.timeout': number;
    'myExt.logLevel': 'error' | 'warn' | 'info' | 'debug'; 'myExt.indent': string;
}
export const MyPreferences = Symbol('MyPreferences');
export type MyPreferences = PreferenceProxy<MyConfiguration>;
export const MyPreferenceContribution = Symbol('MyPreferenceContribution');

export function bindMyPreferences(bind: interfaces.Bind): void {
    bind(MyPreferenceContribution).toConstantValue({ schema: myPreferenceSchema });  // required
    bind(PreferenceContribution).toService(MyPreferenceContribution);
    bind(MyPreferences).toDynamicValue(ctx =>                                         // optional proxy
        ctx.container.get<PreferenceProxyFactory>(PreferenceProxyFactory)(myPreferenceSchema)
    ).inSingletonScope();
}
```
- Name preferences `ext.category.setting`. Always give a `description` and a sensible `default`. Use `enum` + `enumDescriptions` for fixed choices.
- Import `interfaces` from `@theia/core/shared/inversify`.

### Use
- Direct:
  - `prefs.get<T>(name, fallback?, resourceUri?)`
  - `await prefs.set(name, value, scope?, resourceUri?)`
  - `prefs.inspect(name)` returns `{ defaultValue, globalValue, workspaceValue, workspaceFolderValue, value }`.
- Proxy: `this.myPrefs['myExt.timeout']`, which is typed.
- Changes: subscribe with `prefs.onPreferenceChanged(e => { if (e.preferenceName === 'myExt.timeout') { const v = prefs.get(...) } })`, pushed into a `DisposableCollection` and disposed in `dispose()`. The proxy has the same event.
- **Since Theia 1.68, `PreferenceChange` events have no `oldValue`/`newValue` fields.** Always re-read the value with `get` or the proxy.
- Pass a URI to `get`/`set` for resource-specific values.
- The API returns `JSONValue`, not `any`.

### Default overrides
```ts
@injectable()
export class MyPreferenceContribution implements PreferenceContribution {
    readonly schema = myPreferenceSchema;
    async initSchema(s: PreferenceSchemaService): Promise<void> {
        s.registerOverride('editor.tabSize', 'typescript', 2);   // (name, overrideIdentifier, value)
    }
}
```
- Properties must be added to the schema before overrides are registered.

### Session scope (Theia ≥ 1.74)
- In-memory only and never written to disk. Beats every other scope and ignores workspace trust.
- Set on the CLI with `--session-preference name=<JSON>` (repeatable), or `name=base64:<...>` to avoid shell quoting. Unparseable entries only produce a warning.
- Set in code with `prefs.set(name, value, PreferenceScope.Session)`.
- No resource- or language-specific values, and not offered as a scope in the settings UI.
- Writing a persisted scope via the settings UI or the API drops the session override. Editing `settings.json` by hand does not.
- Forwarded to remote backends and dev containers.

### Backend (Theia ≥ 1.65)
- Same `PreferenceService` API, but only Default and User scopes exist. Workspace and Folder values are invisible to the backend.
- Frontend and backend read the same files.
- Register the schema in every process that reads it: call `bindMyPreferences` from both modules.
- For preferences read in the backend, set `scope: PreferenceScope.User` to avoid misleading behavior. If shared, document that the backend sees only user-level values.
- Sample RPC-exposed backend preference service: `examples/api-samples/src/node/sample-backend-preferences-service.ts`.

### Troubleshooting
- Preference missing from UI → `PreferenceContribution` not bound in that process.
- Stale value → no `onPreferenceChanged` listener.
- Type errors → the configuration interface doesn't match the schema.
- To trace resolution, set `"logging.level": "debug"`.
- Reference schemas: `core/src/common/core-preferences.ts`, `filesystem/src/common/filesystem-preferences.ts`, `workspace/src/common/workspace-preferences.ts`.

## Theia AI
For agents, chat agents, prompt fragments, variables, tool functions, slash commands, response renderers or LLM providers (`@theia/ai-*`), read [ai.md](ai.md) in this skill's folder first. Don't load it otherwise.

## Perspectives (experimental, Theia ≥ 1.74)
The API is unstable. Before writing code, grep the installed `@theia/core/src/browser/perspective-service.ts` for current names. Reference implementation: `AIFirstPerspectiveContribution` in `@theia/ai-ide`.
```ts
@injectable()
export class ReviewPerspectiveContribution implements PerspectiveContribution {
    registerPerspectives(s: PerspectiveService): void {
        s.registerPerspective({
            id: 'review',
            label: nls.localize('my-ext/perspective/review', 'Review'),
            viewPlacements: new Map<string, ApplicationShell.Area>([
                [SCM_VIEW_CONTAINER_ID, 'left'],
                [PROBLEMS_WIDGET_ID, 'bottom'],
            ]),
        });
    }
}
// module
bind(ReviewPerspectiveContribution).toSelf().inSingletonScope();
bind(PerspectiveContribution).toService(ReviewPerspectiveContribution);
```
- A perspective is a named layout mapping view IDs to shell areas (`main | left | right | bottom`). Every app also has a built-in `Default` perspective.
- Only views contributed via `AbstractViewContribution` can be placed. Other widgets may have side effects when created.
- Besides placements, a descriptor can set:
  - which view is focused in each area
  - which areas start collapsed
  - hooks that run on enter and on leave
- Inject `PerspectiveService` to get the active perspective, subscribe to its change event, or switch/reset programmatically.
- Context key `activePerspectiveId` gates menus, toolbars and keybindings via `when`.
- Layout changes are saved per perspective.
- The commands Switch Perspective and Reset Perspective (both marked Experimental) exist only in the command palette, not in menus.

## Other contribution points
- Other common points: `FrontendApplicationContribution`, `BackendApplicationContribution`, `TabBarToolbarContribution`, `ColorContribution`, `PreferenceContribution`, `WidgetFactory`, `OpenHandler`.
- To find existing points, grep `bindContributionProvider(` in `@theia/*`.

## Defining a contribution point
```ts
export const FooContribution = Symbol('FooContribution');
export interface FooContribution { registerFoo(r: FooRegistry): void; }
// module
bindContributionProvider(bind, FooContribution);
// consumer
@inject(ContributionProvider) @named(FooContribution)
protected readonly contribs: ContributionProvider<FooContribution>;
// use: this.contribs.getContributions().forEach(c => c.registerFoo(this));
```

## Contribution filter (remove existing contributions, e.g. core commands/menus)
```ts
import { FilterContribution, ContributionFilterRegistry, CommandContribution } from '@theia/core/lib/common';
@injectable()
export class MyFilterContribution implements FilterContribution {
    registerContributionFilters(registry: ContributionFilterRegistry): void {
        registry.addFilters([CommandContribution], [            // '*' = every contribution type
            contrib => !(contrib instanceof UnwantedCommandContribution),   // true = keep
        ]);
    }
}
// module
bind(FilterContribution).to(MyFilterContribution).inSingletonScope();
```
- Filters act on whole contribution instances, not single commands; match by class via `instanceof` against the class exported from the owning `@theia/*` package.
- Filtering applies to what a `ContributionProvider` returns (believed; not verified). A service injected directly is unaffected; replace it with `rebind` instead.
- To drop one command from a contribution that also registers wanted ones, filtering is too coarse. Override the command instead, e.g. `commands.unregisterCommand(id)` in your own `CommandContribution`, or `menus.unregisterMenuAction(id)` for a menu item.

## Backend application lifecycle (`BackendApplicationContribution`)
- Instantiated right after the backend starts. Use it for services that live as long as the backend: timers, DB connections, external processes, REST endpoints.
- All hooks are optional:
  - `initialize()` runs once when the backend is initialized.
  - `configure(app)`, `onStart(server)` and `onStop(app)` extend or configure the Express/HTTP server.
```ts
import { Application } from '@theia/core/shared/express';
import { BackendApplicationContribution } from '@theia/core/lib/node/backend-application';
@injectable()
export class MyBackendContribution implements BackendApplicationContribution {
    @inject(ILogger) protected readonly logger: ILogger;
    protected timer: ReturnType<typeof setInterval> | undefined;
    initialize(): void { this.timer = setInterval(() => this.tick(), 2000); }
    configure(app: Application): void { app.get('/myendpoint', (req, res) => { res.json({ ok: true }); }); }
    onStop(): void { if (this.timer) { clearInterval(this.timer); } }
}
// backend module
bind(MyBackendContribution).toSelf().inSingletonScope();
bind(BackendApplicationContribution).toService(MyBackendContribution);
```
- Import Express from `@theia/core/shared/express`, not from `express` directly.
- Register routes in `configure`.

## Dynamic toolbar (`@theia/toolbar`, optional)
- Add the package to the app to get a toolbar that users can configure.
- To change the defaults shown before the user configures it, replace `ToolbarDefaultsFactory`:
```ts
const bindOrRebind = isBound(ToolbarDefaultsFactory) ? rebind : bind;   // the toolbar may be absent from the app
bindOrRebind(ToolbarDefaultsFactory).toConstantValue(myToolbarDefaults);
```
- Unverified: whether the factory is a function `() => DeflatedToolbarTree`, like the built-in `ToolbarDefaults`. Check `@theia/toolbar/src/browser/toolbar-defaults.ts`.
- If you bind a class instead, the doc's `toService(X)` also needs `bind(X).toSelf()`.

## Internationalization (nls)
- Users switch language with "Configure Display Language". A locale appears there only after its VS Code language pack is installed.
- Wrap every user-facing string:
```ts
import { nls, Command } from '@theia/core';
nls.localize('my-ext/bye', 'Bye');                          // key + English default; '/' groups keys in the JSON
nls.localize('my-ext/byeFmt', 'Bye {0} and {1}!', a, b);    // {n} = extra args; don't build strings with template literals
export const HelloCmd = Command.toLocalizedCommand(
    { id: 'hello-command', label: 'Hello', category: 'Greetings' }, 'my-ext/hello', 'my-ext/greetings'); // label key, category key (the label key defaults to the id)
```
- Run `theia nls-extract` (from `@theia/cli`) to collect all keys into one JSON file. Translate it into `nls.<locale>.json`, then register the translations **in the backend module**:
```ts
@injectable()
export class MyLocalizationContribution implements LocalizationContribution {
    async registerLocalizations(r: LocalizationRegistry): Promise<void> {
        r.registerLocalizationFromRequire('de', require('../../data/i18n/nls.de.json'));   // language codes: de, it, zh-cn, ...
    }
}
bind(MyLocalizationContribution).toSelf().inSingletonScope();
bind(LocalizationContribution).toService(MyLocalizationContribution);
```
- The default locale for first start goes in `package.json`: `"theia": { "frontend": { "config": { "defaultLocale": "zh-cn" } } }`.
- To override any translatable text (rebranding) without writing a full localization, add a `TextReplacementContribution` (in `@theia/core/lib/browser/preload/text-replacement-contribution`). It is applied during frontendPreload.
```ts
@injectable()
export class MyTextReplacement implements TextReplacementContribution {
    getReplacement(locale: string): Record<string, string> {
        return locale === 'de' ? { 'About': 'Über MyApp' } : locale === 'en' ? { 'About': 'About MyApp' } : {};
    }   // keys are the ENGLISH DEFAULT texts, not nls keys
}
// preload module (e.g. src/browser/my-preload-module.ts)
export default new ContainerModule(bind => { bind(TextReplacementContribution).to(MyTextReplacement).inSingletonScope(); });
// package.json: "theiaExtensions": [{ "frontendPreload": "lib/browser/my-preload-module" }]
```

## Tasks (`@theia/task`)
- Tasks use the same format as VS Code (`tasks.json` in the workspace or at user level). They run from the Terminal menu or the command palette.
- Flow: the user picks a task (their own or provider-supplied), the frontend `TaskService` resolves it with the resolver for its `type`, and the backend `TaskServer` runs it with the runner for its `type`.
```ts
// frontend module: providers + resolvers
@injectable()
export class MyTaskContribution implements TaskContribution {
    registerProviders(p: TaskProviderRegistry): void { p.register('myType', this.provider); }
    registerResolvers(r: TaskResolverRegistry): void { r.registerTaskResolver('myType', this.resolver); }
    protected readonly provider: TaskProvider = {
        provideTasks: async () => [{ label: 'My Task', type: 'myType', _scope: 'MyTaskProvider' }],
    };
    protected readonly resolver: TaskResolver = {   // fill defaults/variables; don't overwrite user-set values
        resolveTask: async (c: TaskConfiguration) => ({ myValue: 42, ...c }),
    };
}
bind(MyTaskContribution).toSelf().inSingletonScope();
bind(TaskContribution).toService(MyTaskContribution);

// backend module: runner
@injectable()
export class MyTaskRunner implements TaskRunner {
    @inject(TaskManager) protected readonly taskManager: TaskManager;
    @inject(ILogger) protected readonly logger: ILogger;
    async run(config: TaskConfiguration, ctx?: string): Promise<Task> {
        const task = new MyTask(this.taskManager, this.logger, { config, label: config.label, context: ctx });
        task.start(config);
        return task;
    }
}
class MyTask extends Task {                     // subclassing Task hooks into TaskManager, which shows workbench progress
    start(config: TaskConfiguration): void { /* ... */ this.fireTaskExited({ taskId: this.taskId, code: 0 }); } // always fire exit
    // also implement the Task abstract members: kill(), getRuntimeInfo()
}
@injectable()
export class MyTaskRunnerContribution implements TaskRunnerContribution {
    @inject(MyTaskRunner) protected readonly runner: MyTaskRunner;
    registerRunner(r: TaskRunnerRegistry): void { r.registerRunner('myType', this.runner); }
}
bind(MyTaskRunner).toSelf().inSingletonScope();
bind(MyTaskRunnerContribution).toSelf().inSingletonScope();
bind(TaskRunnerContribution).toService(MyTaskRunnerContribution);
```
- Without a provider, the type is only usable from `tasks.json`. Add a task definition (a JSON schema for the type's properties) so the editor can validate and complete it.
- The doc stops there. For the API, grep `TaskDefinitionRegistry` / `TaskDefinitionContribution` in `@theia/task/src`.

## Telemetry (`@theia/telemetry`, Theia ≥ 1.74, experimental)
- The extension is plumbing only. Nothing is sent unless the app contributes a sink, and there is no built-in destination.
- `TelemetryService` exists in both the frontend and the backend. Frontend events travel over RPC, and the backend decides delivery.
- Browser-only apps get a no-op service.
```ts
import { TelemetryService } from '@theia/telemetry/lib/common';
@inject(TelemetryService) protected readonly telemetry: TelemetryService;
this.telemetry.report('owner/feature/event', { duration: 120, ok: false, targets: ['a', 'b'] },
    { kind: 'error', attributes: { origin: 'build-service' } });   // kind: 'usage' (default) | 'error' | 'crash'
```
- Topics are slash-separated.
- Data and attributes may be strings, numbers, booleans, or homogeneous arrays of those. They are snapshotted when you call `report`.
- Timestamp and session are set by the framework and can't be set by the caller: one UUID per frontend instance, and a constant for the backend.

**Sink** (backend only):
```ts
import { TelemetryEvent } from '@theia/telemetry/lib/common';
import { TelemetrySink } from '@theia/telemetry/lib/node';
@injectable()
export class MySink implements TelemetrySink {
    readonly id = 'owner/backend';                      // stable; users reference it in telemetry.filters
    readonly interests: readonly string[] = ['owner/build/*'];   // exact topic | 'owner/*' | '*'
    readonly scope = 'remote';                          // 'local' = the data stays on the machine and skips consent
    handle(event: TelemetryEvent): void { /* app-owned transport */ }
    async flush(): Promise<void> { /* optional; awaited on backend shutdown */ }
}
// backend module (unverified binding): bind(MySink).toSelf().inSingletonScope(); bind(TelemetrySink).toService(MySink);
```
- An event is delivered only if all three hold:
  1. `telemetry.filters` allows the topic for this sink id. A missing entry allows all of the sink's interests, `[]` disables the sink, and a non-empty list restricts it to those topics.
  2. One of the sink's interests matches the topic.
  3. For remote sinks, the consent level allows the event kind.
- The framework does no buffering and has no transport.
- **Consent**:
  - `telemetry.telemetryLevel` takes the values `off` (the default), `crash`, `error` and `all`, the same as VS Code.
  - It fails closed: the level counts as off until preferences have loaded.
  - Features and sinks never read the level directly; the framework asks `TelemetryConsentProvider { level; onDidChangeTelemetryLevel }`.
  - For consent from an installer or a policy, rebind that provider in **both** the frontend and backend modules.
  - Sinks aren't told about opt-out. Subscribe to `onDidChangeTelemetryLevel` yourself.
- **Changing defaults**: register `PreferenceContribution` overrides in **both** containers. `theia.frontend.config.preferences` alone isn't enough, because the backend decides delivery.
- **@theia/metrics**:
  - `/metrics` is served by a local sink `theia/measurements`, which consumes `theia/measurement/result` events. It needs the backend started with `--log-level=debug`.
  - Disable it with `"telemetry.filters": { "theia/measurements": [] }`.
  - `MeasurementNotificationService` was removed; replace it with a backend sink interested in `theia/measurement/result`.

## Theia-only API for VS Code extensions
One VS Code extension that runs in both VS Code and Theia, with extra features in Theia. Gate them on the app name (`vscode.env.appName` = the app's `applicationName`):
```ts
if (vscode.env.appName === MY_THEIA_APP_NAME) {
    // Option A: custom API module, imported lazily so VS Code never loads it
    const api = await import('@my/theia-api');
    api.host.onRequestMessage((actor: string) => api.host.showMessage(getMessage(actor)));
    // Option B: custom Theia command; check that it exists first
    if ((await vscode.commands.getCommands(true)).includes(MY_THEIA_COMMAND)) {
        await vscode.commands.executeCommand(MY_THEIA_COMMAND);
    }
}
```
- Option B needs no plugin-side wiring: register the command with a normal Theia `CommandContribution`.
- Option A: mark the API module as external in the extension's bundler; the Theia plugin host supplies it at runtime.
- Option A, Theia side: the page doesn't show how the API module is provided. It is believed to be an `ExtPluginApiProvider` from `@theia/plugin-ext` (not verified). Reference: github.com/thegecko/vscode-theia-extension.

## Frontend ↔ backend service (JSON-RPC)
- All `ConnectionHandler`s are collected through a contribution provider, and `MessagingContribution` opens a channel per `path`.
- Every channel is multiplexed over one websocket.
- Each side gets an `RpcProxy` for the remote object and optionally exposes a local object to it.
```ts
// common/foo-protocol.ts: JSON-serializable args, Promise returns
export const FooPath = '/services/foo';
export const FooServer = Symbol('FooServer');
export interface FooClient { onProgress(e: FooEvent): void; }           // backend → frontend callbacks
export interface FooServer extends RpcServer<FooClient> { run(arg: string): Promise<string>; }  // RpcServer = Disposable & setClient/getClient?

// node/foo-backend-module.ts
bind(FooServerImpl).toSelf().inSingletonScope();
bind(FooServer).toService(FooServerImpl);
bind(ConnectionHandler).toDynamicValue(ctx =>
    new RpcConnectionHandler<FooClient>(FooPath, client => {      // client = proxy to the frontend object
        const server = ctx.container.get<FooServer>(FooServer);
        server.setClient(client);
        return server;                                             // this object is exposed over RPC
    })).inSingletonScope();

// browser/foo-frontend-module.ts: pass the local client object as the 3rd argument
bind(FooServer).toDynamicValue(ctx =>
    ServiceConnectionProvider.createProxy<FooServer>(ctx.container, FooPath, ctx.container.get(FooWatcher).getClient())
).inSingletonScope();
```
- If the local client depends on the proxy, create the proxy without a client and call `proxy.setClient(client)` afterwards. This is why the server interface extends `RpcServer<Client>`.
- Client pattern:
  - A watcher, e.g. `FooWatcher.getClient()`, returns `{ onProgress: e => emitter.fire(e) }`.
  - Frontend code subscribes to `watcher.onProgress`. See Events under TypeScript.
- **Multiple windows or tabs:**
  - A singleton server with `setClient` keeps only the last client.
  - To notify every frontend, keep a set of clients: add each one on connection and remove it on `dispose()` or when its channel closes.
  - For per-connection state, create a fresh instance in the connection factory instead.
- Imports come from `@theia/core/lib/common/messaging` (`ConnectionHandler`, `RpcConnectionHandler`, `RpcServer`) and `@theia/core/lib/browser/messaging`.
- Older Theia uses `JsonRpcConnectionHandler`, `JsonRpcServer`, and `WebSocketConnectionProvider` (`ctx.container.get(WebSocketConnectionProvider).createProxy(path, client)`). Grep the installed @theia/core for the exported names.
- Frontend code depends only on the interface, never on `FooServerImpl`.

## TypeScript
- `strict`. No `any`; use `unknown` and narrow.
- Explicit return types on public methods.
- `readonly` on injected fields.
- Use `Disposable` / `DisposableCollection` for listeners; push to `this.toDispose` and dispose in `dispose()`.


- Use `MaybePromise<T>` where contribution APIs allow sync or async.
