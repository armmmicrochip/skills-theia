# TreeWidget (`@theia/core/lib/browser`)
Reference: generate the "TreeWidget View" example with the Theia Extension Generator. For more options read `TreeWidget`, `TreeProps`, `TreeServices` and `createTreeContainer` in `node_modules/@theia/core/src/browser/tree/`.

## Parts
- **Widget**: a subclass of `TreeWidget`, which is a ReactWidget that already handles rendering and interaction.
- **TreeModel facade**: a subclass of `TreeModelImpl`. It initializes the tree and syncs it with the business model.
- **Tree**: a subclass of `TreeImpl`. It holds the nodes and their UI state and resolves children.
- **Node interfaces**:
  - `TreeNode` (`id`, `parent`) for leaves.
  - `CompositeTreeNode` (`+children`) for containers.
  - `ExpandableTreeNode` (`+expanded`) and `SelectableTreeNode` (`+selected`) add expansion and selection.
- **Child container**: built with `createTreeContainer`, so each tree has its own isolated bindings. Don't use named bindings for this.

## Nodes
- `id` must be unique within the tree; duplicate ids break things silently.
- Add children only via `CompositeTreeNode.addChild(parent, child)`, which maintains both `parent` and `children`. Create nodes with `children: []` and `parent: undefined`.
- Add `data: MyItem` to every node to link it back to the business model. Add a `type` discriminator so each node interface can have an `is()` guard.
```ts
export interface MyNode extends ExpandableTreeNode, SelectableTreeNode { data: Item; type: 'node' }
export namespace MyNode { export function is(c: object): c is MyNode { return CompositeTreeNode.is(c) && 'type' in c && c.type === 'node'; } }
export interface MyLeaf extends SelectableTreeNode { data: Item; type: 'leaf' }
export namespace MyLeaf { export function is(c: object): c is MyLeaf { return TreeNode.is(c) && 'type' in c && c.type === 'leaf'; } }
// factory: { id, data, parent: undefined, type, selected: false, expanded: false, children: [], checkboxInfo?: { checked } }
```

## Widget + model
```ts
@injectable()
export class MyTreeWidget extends TreeWidget {
    static readonly ID = 'my-tree'; static readonly LABEL = 'My Tree';
    constructor(
        @inject(TreeProps) props: TreeProps,
        @inject(TreeModel) override readonly model: MyTreeModel,
        @inject(ContextMenuRenderer) contextMenuRenderer: ContextMenuRenderer,
    ) {
        super(props, model, contextMenuRenderer);
        this.id = MyTreeWidget.ID; this.title.label = this.title.caption = MyTreeWidget.LABEL; this.title.closable = true;
    }
}

export const ROOT_ID = 'my-tree-root';
@injectable()
export class MyTreeModel extends TreeModelImpl {
    @inject(MyItemFactory) protected readonly itemFactory: MyItemFactory;
    @postConstruct()
    protected override init(): void {
        super.init();
        const root: CompositeTreeNode = { id: ROOT_ID, parent: undefined, children: [], visible: false };
        DATA.map(i => this.itemFactory.toTreeNode(i)).forEach(n => CompositeTreeNode.addChild(root, n));
        this.tree.root = root;   // set this LAST: it builds the id-lookup map from the current children
    }
}
```
For async or remote data, initialize the model later, e.g. from the widget's `onAfterAttach`.

## Container + module
```ts
export default new ContainerModule(bind => {
    bindViewContribution(bind, MyTreeViewContribution);
    bind(FrontendApplicationContribution).toService(MyTreeViewContribution);
    bind(WidgetFactory).toDynamicValue(ctx => ({
        id: MyTreeWidget.ID,
        createWidget: () => createMyTreeContainer(ctx.container).get(MyTreeWidget),
    })).inSingletonScope();
    bind(MyTreeLabelProvider).toSelf().inSingletonScope();
    bind(LabelProviderContribution).toService(MyTreeLabelProvider);
});
function createMyTreeContainer(parent: interfaces.Container): Container {
    const child = createTreeContainer(parent, {
        model: MyTreeModel, widget: MyTreeWidget, tree: MyTree,
        props: { contextMenuPath: MY_TREE_CONTEXT_MENU, multiSelect: false, search: true },  // Partial<TreeProps>, merged with defaults
        // decoratorService: MyDecorationService,
    });
    child.bind(MyItemFactory).toSelf().inSingletonScope();
    return child;
}
```
- The doc also binds the model in the parent container as a singleton. That is redundant and would leak one shared model across widgets, so it's omitted here.
- `TreeProps` options:
  - `search`: type-to-filter.
  - `multiSelect`: Ctrl/Shift selection.
  - `expandOnlyOnExpansionToggleClick`: nodes expand only via the toggle, not when selected.
  - `contextMenuPath`: enables the context menu (nodes must also be `SelectableTreeNode`).

## Labels and icons
- Without a label the node renders as `<Unknown>`.
- Either override `toNodeName(node)` in the widget, or (preferred) add a `LabelProviderContribution` with `canHandle` returning e.g. 100 for your nodes, plus `getName` and `getIcon`. See Label providers in SKILL.md.
- `renderIcon` in the base class is empty, so an icon from the label provider is not drawn. Override it:
```tsx
protected override renderIcon(node: TreeNode, props: NodeProps): React.ReactNode {
    const icon = this.getIconClass(this.toNodeIcon(node));
    return icon ? <div className={icon} /> : super.renderIcon(node, props);
}
```
- `toNodeIcon` adds the `fa fa-` prefix to a FontAwesome name like `'folder'`.
- To style nodes, append a class in `createNodeClassNames(node, props)` (`super(...).concat('my-node')`) and import the CSS from the frontend module. Icons carry class `a`, so padding would be `.my-node .a { padding-right: 4px }`.

## Lazy children
- Make container nodes `ExpandableTreeNode` with `expanded: false`, and add only the root's children in `init`.
- `TreeExpansionService` flips `expanded` and calls `tree.refresh(node)`, which calls `resolveChildren`. The node shows a busy spinner if resolution takes longer than 800 ms.
```ts
@injectable()
export class MyTree extends TreeImpl {
    @inject(MyItemFactory) protected readonly itemFactory: MyItemFactory;
    override async resolveChildren(parent: CompositeTreeNode): Promise<TreeNode[]> {
        if (parent.id === ROOT_ID) { return [...parent.children]; }
        if (!MyNode.is(parent)) { return []; }
        if (parent.children.length === parent.data.children?.length) { return [...parent.children]; } // optional cache
        return (await this.fetchChildren(parent.data)).map(i => this.itemFactory.toTreeNode(i));
    }
}
```
- Bind it via `tree: MyTree` in `createTreeContainer`. The doc's class omits `@injectable()`, which it needs.

## Structural edits
Change the business data first, then call `this.tree.refresh(affectedParent)` for every affected parent. The children are then re-resolved.
```ts
addItem(parent: TreeNode): void {
    if (MyNode.is(parent)) { parent.data.children?.push(newItem); this.tree.refresh(parent); }
}
```
Use `this.tree.getNode(id)` to look up a node by id.

## Checkboxes
- Set `checkboxInfo: { checked }` on nodes.
- To react to changes, override `markAsChecked(node, checked)` in the model: write to `node.data` first, then call `super.markAsChecked(...)`.
- Known upstream issue: the UI may not reflect the new state after a click.

## Decorations
Decorations add prefix/suffix text, font or colors, or an icon overlay, without restyling the widget. Each tree defines its own contribution point:
```ts
export const MyTreeDecorator = Symbol('MyTreeDecorator');
@injectable()
export class MyDecorationService extends AbstractTreeDecoratorService {
    constructor(@inject(ContributionProvider) @named(MyTreeDecorator) protected readonly contributions: ContributionProvider<TreeDecorator>) {
        super(contributions.getContributions());
    }
}
@injectable()
export class MyDecorator implements TreeDecorator {
    readonly id = 'my-decorator';
    protected readonly emitter = new Emitter<(tree: Tree) => Map<string, WidgetDecoration.Data>>();
    get onDidChangeDecorations(): Event<(tree: Tree) => Map<string, WidgetDecoration.Data>> { return this.emitter.event; } // fire to re-decorate
    decorations(tree: Tree): MaybePromise<Map<string, WidgetDecoration.Data>> {           // may be async
        const result = new Map<string, WidgetDecoration.Data>();
        if (!tree.root) { return result; }
        for (const n of new DepthFirstTreeIterator(tree.root)) {                          // also BreadthFirst/TopDown/BottomUp
            if (MyLeaf.is(n)) {
                result.set(n.id, {
                    iconOverlay: { position: WidgetDecoration.IconOverlayPosition.BOTTOM_RIGHT, iconClass: ['fa', 'fa-check-circle'], color: 'green' },
                    // backgroundColor, captionSuffixes: [{ data: 'text', fontData: { style: 'italic' } }], ...
                });
            }
        }
        return result;
    }
}
// Bindings: bindContributionProvider(bind, MyTreeDecorator); bind(MyTreeDecorator).to(MyDecorator).inSingletonScope();
// and pass decoratorService: MyDecorationService to createTreeContainer.
```
The doc doesn't show how the decorator service is wired up; the `decoratorService` option in `createTreeContainer` is unverified, so check `TreeContainerProps` in the installed version.

## Open (double-click)
In the widget, subscribe with `this.toDispose.push(this.model.onOpenNode(node => ...))`, or override `doOpenNode` in the model. With field injection, subscribe in a `@postConstruct` method, not in the constructor.

## Context menu
- Set `props.contextMenuPath` and make the nodes selectable.
- Register the actions under the path, e.g. `menus.registerMenuAction([...MY_TREE_CONTEXT_MENU, '_1'], { commandId, label })`.
- In handlers, read `widget.model.selectedNodes[0]`, and gate `isVisible`/`isEnabled` on the node type.

## Drag and drop (not built in)
```tsx
protected readonly toCancelNodeExpansion = new DisposableCollection();
protected override createNodeAttributes(node: TreeNode, props: NodeProps): React.Attributes & React.HTMLAttributes<HTMLElement> {
    return {
        ...super.createNodeAttributes(node, props),
        draggable: MyLeaf.is(node),
        onDragStart: e => { e.stopPropagation(); e.dataTransfer.setData('tree-node', node.id); },
        onDragEnter: e => { e.preventDefault(); e.stopPropagation(); this.toCancelNodeExpansion.dispose(); this.selectDropTarget(node); },
        onDragLeave: e => { e.preventDefault(); e.stopPropagation(); this.toCancelNodeExpansion.dispose(); },
        onDragOver: e => {
            e.preventDefault(); e.stopPropagation(); e.dataTransfer.dropEffect = 'move';
            if (!this.toCancelNodeExpansion.disposed) { return; }            // an expansion is already pending
            const t = setTimeout(() => { if (MyNode.is(node) && !node.expanded) { this.model.expandNode(node); } }, 500);
            this.toCancelNodeExpansion.push(Disposable.create(() => clearTimeout(t)));
        },
        onDrop: e => {
            e.preventDefault(); e.stopPropagation();
            const target = MyLeaf.is(node) ? node.parent : node;               // dropping on a leaf goes to its parent
            if (target && MyNode.is(target)) { this.model.reparent(e.dataTransfer.getData('tree-node'), target); }
        },
    };
}
```
- `reparent` in the model:
  1. Look up the node with `tree.getNode(id)`.
  2. Remove its data from the source parent's `data.children` and push it to the target's.
  3. Refresh both parents.
- `selectDropTarget` resolves the target the same way (a leaf means its parent) and calls `this.model.selectNode(target)` to highlight it.
