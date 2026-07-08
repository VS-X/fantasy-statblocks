<script lang="ts">
    import { MarkdownRenderer, Notice } from "obsidian";
    import type { Layout, MarkdownableItem } from "src/layouts/layout.types";
    import type StatBlockPlugin from "src/main";

    import { getContext } from "svelte";
    import type StatBlockRenderer from "../statblock";
    import type { Monster } from "index";
    import { Linkifier } from "src/parser/linkify";
    import { parseForDice } from "src/parser/dice-parsing";
    import type { Writable } from "svelte/store";
    import { stringify } from "src/util/util";

    export let property: string;

    property =
        typeof property === "string"
            ? Linkifier.stringifyLinks(property)
            : property;

    const context = getContext<string>("context");
    const renderer = getContext<StatBlockRenderer>("renderer");
    let item = getContext<MarkdownableItem>("item");
    let canDice = getContext<boolean>("dice");

    let parseDice = item.dice;
    const monsterStore = getContext<Writable<Monster>>("monster");
    let monster = $monsterStore;
    monsterStore.subscribe((m) => (monster = m));
    let plugin = getContext<StatBlockPlugin>("plugin");
    let layout = getContext<Layout>("layout");

    /** Authored text for this property (e.g. "11 (2d6 + 4)"), kept so combine
     * mode can show it next to the rolled widget even when a dice path replaces
     * the whole property. */
    const authored = stringify(property);

    let split: Array<
        { text: string; original?: string; combined?: string } | string
    > = [property];
    if (canDice && parseDice) {
        if (
            item.diceProperty &&
            item.diceProperty in monster &&
            typeof monster[item.diceProperty] == "string"
        ) {
            split = [{ text: monster[item.diceProperty] as string }];
        } else {
            const parsed = parseForDice(layout, authored, monster);
            if (Array.isArray(parsed)) {
                split = parsed;
            } else {
                split = [parsed];
            }
        }
    }
    if (canDice && item.diceCallback) {
        try {
            const frame = document.body.createEl("iframe");
            const funct = (frame.contentWindow as any).Function;
            const func = new funct("monster", "property", item.diceCallback);
            const parsed = func.call(undefined, monster, property) ?? property;
            document.body.removeChild(frame);
            if (Array.isArray(parsed)) {
                split = parsed;
            } else {
                split = [parsed];
            }
        } catch (e) {
            new Notice(
                `There was an error executing the provided dice callback for [${item.properties.join(
                    ", "
                )}]\n\n${(e as Error).message}`
            );
            console.error(e);
        }
    }

    const combineDice = plugin.settings.combineDiceDisplay;

    /** When a dice path replaced the whole property with a single roll (e.g. the
     * default HP callback returning `[{ text: hit_dice }]`), keep the authored
     * text as the combine-mode prefix so formula + average stay visible. */
    if (
        combineDice &&
        split.length === 1 &&
        typeof split[0] === "object" &&
        !split[0].combined &&
        !split[0].original
    ) {
        split[0].combined = authored;
    }

    property = "";
    for (const dice of split) {
        if (canDice && typeof dice == "object") {
            let diceString;
            let diceText = plugin.getRollerString(dice.text);
            /** When combining, show the authored formula + average (`combined`,
             * falling back to `original`) next to the live-rolled widget. */
            const prefix = combineDice
                ? dice.combined ?? dice.original
                : dice.original;
            if (prefix) {
                diceString = `${prefix} (\`dice: ${diceText}\`)`;
            } else {
                diceString = `\`dice: ${diceText}\``;
            }
            property += diceString;
        } else {
            property += dice;
        }
    }

    const markdown = (node: HTMLElement) => {
        if (property === "-") {
            property = "\\-";
        }
        MarkdownRenderer.render(plugin.app, property, node, context, renderer);
    };
</script>

<div class="statblock-markdown" use:markdown />

<style>
    .statblock-markdown {
        display: inline;
    }
    .statblock-markdown :global(p) {
        display: inline;
    }
    .statblock-markdown :global(p ~ p) {
        display: inline-block;
    }
</style>
