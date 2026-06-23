# 🧩 Discord.js Components V2 — The Complete Guide

> Learn how to build modern, layout-based Discord bot UIs with **Components V2** in `discord.js` v14.

-----

## 📑 Table of Contents

1. [What Is Components V2?](#what-is-components-v2)
1. [Setup & Requirements](#setup--requirements)
1. [How CV2 Works](#how-cv2-works)
1. [Builder Reference](#builder-reference)
- [ContainerBuilder](#containerbuilder)
- [TextDisplayBuilder](#textdisplaybuilder)
- [SeparatorBuilder](#separatorbuilder)
- [SectionBuilder & ThumbnailBuilder](#sectionbuilder--thumbnailbuilder)
- [MediaGalleryBuilder](#mediagallerybuilder)
- [ActionRowBuilder, ButtonBuilder & Selects](#actionrowbuilder-buttonbuilder--selects)
1. [Sending & Updating CV2 Messages](#sending--updating-cv2-messages)
1. [Worked Examples](#worked-examples)
- [Example 1: Server Stats Card](#example-1-server-stats-card)
- [Example 2: Ticket Panel With Buttons](#example-2-ticket-panel-with-buttons)
- [Example 3: Paginated Leaderboard](#example-3-paginated-leaderboard)
- [Example 4: Settings Menu With a Select](#example-4-settings-menu-with-a-select)
1. [Gotchas & Best Practices](#gotchas--best-practices)
1. [Limitations Cheat Sheet](#limitations-cheat-sheet)

-----

## What Is Components V2?

Components V2 (CV2) is Discord’s layout-first message system. Instead of stuffing everything into a single embed object, you compose a message out of small building blocks — text, separators, images, and interactive rows — all nested inside a `Container`.

|                    |Classic Embeds                   |Components V2                              |
|--------------------|---------------------------------|-------------------------------------------|
|Structure           |One rigid embed object           |Composable blocks you stack freely         |
|Buttons/selects     |Bolted on below the embed        |Live inside the layout itself              |
|Images              |`image` / `thumbnail` fields only|Dedicated gallery + thumbnail components   |
|Text                |Title/description/fields         |Any number of free-form markdown blocks    |
|Side-by-side content|Not possible                     |`SectionBuilder` puts text next to an image|

If you’ve used `discord.py`’s `LayoutView`, the concept is the same — `discord.js` just exposes it through builder classes instead of a Python class hierarchy.

-----

## Setup & Requirements

```bash
npm install discord.js@latest
```

CV2 needs **discord.js v14.16 or newer**. Check your version with `npm list discord.js` and update if needed.

Minimum intents for most bots using CV2:

```js
const { Client, GatewayIntentBits } = require('discord.js');

const client = new Client({
    intents: [
        GatewayIntentBits.Guilds,
        GatewayIntentBits.GuildMessages,
        GatewayIntentBits.MessageContent,
    ],
});
```

Common imports you’ll reach for:

```js
const {
    ContainerBuilder,
    TextDisplayBuilder,
    SeparatorBuilder,
    SeparatorSpacingSize,
    SectionBuilder,
    ThumbnailBuilder,
    MediaGalleryBuilder,
    MediaGalleryItemBuilder,
    UnfurledMediaItemBuilder,
    ActionRowBuilder,
    ButtonBuilder,
    ButtonStyle,
    StringSelectMenuBuilder,
    StringSelectMenuOptionBuilder,
    MessageFlags,
    ComponentType,
} = require('discord.js');
```

-----

## How CV2 Works

The single most important rule in this entire guide:

> **Every CV2 message must include `flags: MessageFlags.IsComponentsV2`.** Without it, Discord ignores the layout builders entirely.

```js
const container = new ContainerBuilder()
    .addTextDisplayComponents(
        new TextDisplayBuilder().setContent("Hey there 👋")
    );

await message.reply({
    components: [container],
    flags: MessageFlags.IsComponentsV2,
});
```

**Shape of a CV2 message:**

```
components: [ Container ]            ← only Container goes at the top level
  ├─ TextDisplay(s)                  ← markdown text blocks
  ├─ Separator(s)                    ← dividers / spacing
  ├─ Section                         ← text (left) + Thumbnail (right)
  ├─ MediaGallery                    ← up to 10 images
  └─ ActionRow                       ← Buttons OR one Select
```

You can never put `embeds: [...]` in the same payload as CV2 `components`. Pick one system per message.

-----

## Builder Reference

### ContainerBuilder

The outer shell. Every other component lives inside one.

```js
new ContainerBuilder()
    .setAccentColor(0x5865F2)   // colored strip on the left edge
    .setSpoiler(false)          // true = blurred until clicked
    .addTextDisplayComponents(/* ... */)
    .addSeparatorComponents(/* ... */)
    .addSectionComponents(/* ... */)
    .addMediaGalleryComponents(/* ... */)
    .addActionRowComponents(/* ... */);
```

A quick colored-border example:

```js
const success = new ContainerBuilder()
    .setAccentColor(0x57F287) // Discord "green"
    .addTextDisplayComponents(
        new TextDisplayBuilder().setContent("## ✅ Done\nYour changes were saved.")
    );
```

-----

### TextDisplayBuilder

Free-form markdown text. Every line of text in a CV2 layout goes through this.

```js
new TextDisplayBuilder().setContent("# Heading 1")
new TextDisplayBuilder().setContent("## Heading 2")
new TextDisplayBuilder().setContent("**bold**, *italic*, __underline__")
new TextDisplayBuilder().setContent("> a quote")
new TextDisplayBuilder().setContent("-# tiny subtext")
```

Interpolate variables like you normally would:

```js
new TextDisplayBuilder().setContent(`**Balance:** \`${coins.toLocaleString()}\` coins`)
new TextDisplayBuilder().setContent(`**Next payout:** <t:${nextPayout}:R>`)
```

-----

### SeparatorBuilder

A horizontal rule or blank gap between blocks.

```js
new SeparatorBuilder()                                   // visible line, small gap
new SeparatorBuilder().setDivider(false)                 // invisible spacer only
new SeparatorBuilder().setSpacing(SeparatorSpacingSize.Large)
```

-----

### SectionBuilder & ThumbnailBuilder

`SectionBuilder` places one or more `TextDisplay` blocks next to a small image, perfect for profile cards.

```js
const section = new SectionBuilder()
    .addTextDisplayComponents(
        new TextDisplayBuilder().setContent(`# ${member.displayName}`),
        new TextDisplayBuilder().setContent(`Joined <t:${joinedTs}:R>`),
    )
    .setThumbnailAccessory(
        new ThumbnailBuilder()
            .setMedia(new UnfurledMediaItemBuilder().setURL(member.displayAvatarURL({ size: 256 })))
            .setDescription(`${member.user.username}'s avatar`)
    );
```

> `ThumbnailBuilder` only works as a `.setThumbnailAccessory()` inside a `SectionBuilder` — it cannot stand on its own.

-----

### MediaGalleryBuilder

Shows up to **10 images** in one block.

```js
const gallery = new MediaGalleryBuilder()
    .addItems(
        new MediaGalleryItemBuilder()
            .setMedia(new UnfurledMediaItemBuilder().setURL("https://example.com/banner.png"))
            .setDescription("Server banner"),
    );
```

For local files, use `attachment://` and send the file alongside the message:

```js
const file = new AttachmentBuilder(buffer, { name: "chart.png" });

const gallery = new MediaGalleryBuilder()
    .addItems(
        new MediaGalleryItemBuilder()
            .setMedia(new UnfurledMediaItemBuilder().setURL("attachment://chart.png"))
    );

await message.reply({
    components: [container],
    flags: MessageFlags.IsComponentsV2,
    files: [file],
});
```

-----

### ActionRowBuilder, ButtonBuilder & Selects

Rows hold the interactive stuff — **up to 5 buttons, or exactly 1 select menu, never both.**

```js
new ActionRowBuilder().addComponents(
    new ButtonBuilder().setLabel("Approve").setStyle(ButtonStyle.Success).setCustomId("approve"),
    new ButtonBuilder().setLabel("Reject").setStyle(ButtonStyle.Danger).setCustomId("reject"),
);
```

Button styles at a glance:

|Style      |Color       |Typical use                               |
|-----------|------------|------------------------------------------|
|`Primary`  |Blue        |Main call to action                       |
|`Secondary`|Grey        |Neutral / “back”                          |
|`Success`  |Green       |Confirm                                   |
|`Danger`   |Red         |Destructive                               |
|`Link`     |Grey + arrow|External URL (no `customId`, no collector)|

Select menus:

```js
new StringSelectMenuBuilder()
    .setCustomId("pick_role")
    .setPlaceholder("Pick a role...")
    .addOptions(
        new StringSelectMenuOptionBuilder().setLabel("Gamer").setValue("gamer").setEmoji("🎮"),
        new StringSelectMenuOptionBuilder().setLabel("Artist").setValue("artist").setEmoji("🎨"),
    );
```

-----

## Sending & Updating CV2 Messages

**Sending:**

```js
await channel.send({
    components: [container],
    flags: MessageFlags.IsComponentsV2,
});
```

**Replying ephemerally (combine flags with bitwise OR):**

```js
await interaction.reply({
    components: [container],
    flags: MessageFlags.IsComponentsV2 | MessageFlags.Ephemeral,
});
```

**Updating in place after a button/select click:**

```js
await interaction.update({
    components: [newContainer],
    flags: MessageFlags.IsComponentsV2,
});
```

-----

## Worked Examples

### Example 1: Server Stats Card

A snapshot card using a `Section` for the icon + a `Separator` to break up stats.

```js
const {
    ContainerBuilder, TextDisplayBuilder, SeparatorBuilder,
    SectionBuilder, ThumbnailBuilder, UnfurledMediaItemBuilder,
    MessageFlags,
} = require("discord.js");

module.exports = {
    name: "serverstats",
    async execute(message) {
        const guild = message.guild;
        const members = await guild.members.fetch();
        const humans = members.filter(m => !m.user.bot).size;
        const bots = members.filter(m => m.user.bot).size;

        const section = new SectionBuilder()
            .addTextDisplayComponents(
                new TextDisplayBuilder().setContent(`# ${guild.name}`),
                new TextDisplayBuilder().setContent(`-# Created <t:${Math.floor(guild.createdTimestamp / 1000)}:R>`),
            )
            .setThumbnailAccessory(
                new ThumbnailBuilder()
                    .setMedia(new UnfurledMediaItemBuilder().setURL(guild.iconURL({ size: 256 }) ?? "https://discord.com/assets/icon.png"))
                    .setDescription("Server icon")
            );

        const container = new ContainerBuilder()
            .setAccentColor(0x5865F2)
            .addSectionComponents(section)
            .addSeparatorComponents(new SeparatorBuilder())
            .addTextDisplayComponents(
                new TextDisplayBuilder().setContent(`**Members:** ${members.size}`),
                new TextDisplayBuilder().setContent(`**Humans:** ${humans} • **Bots:** ${bots}`),
                new TextDisplayBuilder().setContent(`**Roles:** ${guild.roles.cache.size}`),
                new TextDisplayBuilder().setContent(`**Boost Tier:** ${guild.premiumTier || "None"}`),
            );

        await message.reply({
            components: [container],
            flags: MessageFlags.IsComponentsV2,
        });
    }
};
```

-----

### Example 2: Ticket Panel With Buttons

A support-ticket opener that swaps the layout once a ticket is created.

```js
const {
    ContainerBuilder, TextDisplayBuilder, SeparatorBuilder,
    ActionRowBuilder, ButtonBuilder, ButtonStyle,
    MessageFlags, ComponentType,
} = require("discord.js");

function buildPanel() {
    return new ContainerBuilder()
        .setAccentColor(0x5865F2)
        .addTextDisplayComponents(
            new TextDisplayBuilder().setContent("## 🎫 Support Tickets"),
            new TextDisplayBuilder().setContent("Need help? Open a private ticket and our team will assist you."),
        )
        .addSeparatorComponents(new SeparatorBuilder())
        .addActionRowComponents(
            new ActionRowBuilder().addComponents(
                new ButtonBuilder().setLabel("Open Ticket").setStyle(ButtonStyle.Success).setCustomId("open_ticket").setEmoji("🎫"),
            )
        );
}

function buildConfirmation(channel) {
    return new ContainerBuilder()
        .setAccentColor(0x57F287)
        .addTextDisplayComponents(
            new TextDisplayBuilder().setContent("## ✅ Ticket Created"),
            new TextDisplayBuilder().setContent(`Head over to ${channel.toString()} — a staff member will be with you shortly.`),
        );
}

module.exports = {
    name: "tickets",
    async execute(message) {
        const reply = await message.reply({
            components: [buildPanel()],
            flags: MessageFlags.IsComponentsV2,
        });

        const collector = reply.createMessageComponentCollector({
            componentType: ComponentType.Button,
            filter: i => i.customId === "open_ticket",
            time: 600_000,
        });

        collector.on("collect", async i => {
            const ticketChannel = await message.guild.channels.create({
                name: `ticket-${i.user.username}`,
                permissionOverwrites: [
                    { id: message.guild.id, deny: ["ViewChannel"] },
                    { id: i.user.id, allow: ["ViewChannel", "SendMessages"] },
                ],
            });

            await i.reply({
                components: [buildConfirmation(ticketChannel)],
                flags: MessageFlags.IsComponentsV2 | MessageFlags.Ephemeral,
            });
        });
    }
};
```

-----

### Example 3: Paginated Leaderboard

Pagination through buttons, rebuilding the container per page.

```js
const {
    ContainerBuilder, TextDisplayBuilder, SeparatorBuilder,
    ActionRowBuilder, ButtonBuilder, ButtonStyle,
    MessageFlags, ComponentType,
} = require("discord.js");

const PAGE_SIZE = 5;

function buildLeaderboard(entries, page) {
    const totalPages = Math.ceil(entries.length / PAGE_SIZE);
    const start = page * PAGE_SIZE;
    const slice = entries.slice(start, start + PAGE_SIZE);

    const lines = slice.map((entry, idx) =>
        new TextDisplayBuilder().setContent(`**#${start + idx + 1}** — ${entry.name} • \`${entry.score} pts\``)
    );

    return new ContainerBuilder()
        .setAccentColor(0xFEE75C)
        .addTextDisplayComponents(new TextDisplayBuilder().setContent("## 🏆 Leaderboard"))
        .addSeparatorComponents(new SeparatorBuilder())
        .addTextDisplayComponents(...lines)
        .addSeparatorComponents(new SeparatorBuilder())
        .addTextDisplayComponents(new TextDisplayBuilder().setContent(`-# Page ${page + 1} of ${totalPages}`))
        .addActionRowComponents(
            new ActionRowBuilder().addComponents(
                new ButtonBuilder().setCustomId("lb_prev").setLabel("◀ Prev").setStyle(ButtonStyle.Secondary).setDisabled(page === 0),
                new ButtonBuilder().setCustomId("lb_next").setLabel("Next ▶").setStyle(ButtonStyle.Secondary).setDisabled(page >= totalPages - 1),
            )
        );
}

module.exports = {
    name: "leaderboard",
    async execute(message, args, entries /* [{name, score}, ...] */) {
        let page = 0;

        const reply = await message.reply({
            components: [buildLeaderboard(entries, page)],
            flags: MessageFlags.IsComponentsV2,
        });

        const collector = reply.createMessageComponentCollector({
            componentType: ComponentType.Button,
            filter: i => i.user.id === message.author.id,
            time: 60_000,
        });

        collector.on("collect", async i => {
            page += i.customId === "lb_next" ? 1 : -1;
            await i.update({
                components: [buildLeaderboard(entries, page)],
                flags: MessageFlags.IsComponentsV2,
            });
        });
    }
};
```

-----

### Example 4: Settings Menu With a Select

A toggleable settings panel driven by a dropdown.

```js
const {
    ContainerBuilder, TextDisplayBuilder, SeparatorBuilder,
    ActionRowBuilder, StringSelectMenuBuilder, StringSelectMenuOptionBuilder,
    MessageFlags, ComponentType,
} = require("discord.js");

const SETTINGS = {
    welcome_messages: "Welcome Messages",
    auto_role: "Auto Role",
    level_ups: "Level-Up Alerts",
};

function buildSettings(state) {
    const select = new StringSelectMenuBuilder()
        .setCustomId("toggle_setting")
        .setPlaceholder("Toggle a setting...")
        .addOptions(
            Object.entries(SETTINGS).map(([key, label]) =>
                new StringSelectMenuOptionBuilder()
                    .setLabel(label)
                    .setValue(key)
                    .setDescription(state[key] ? "Currently ON" : "Currently OFF")
                    .setEmoji(state[key] ? "🟢" : "🔴")
            )
        );

    const lines = Object.entries(SETTINGS).map(([key, label]) =>
        new TextDisplayBuilder().setContent(`${state[key] ? "🟢" : "🔴"} **${label}**`)
    );

    return new ContainerBuilder()
        .setAccentColor(0x5865F2)
        .addTextDisplayComponents(new TextDisplayBuilder().setContent("## ⚙️ Server Settings"))
        .addSeparatorComponents(new SeparatorBuilder())
        .addTextDisplayComponents(...lines)
        .addSeparatorComponents(new SeparatorBuilder())
        .addActionRowComponents(new ActionRowBuilder().addComponents(select));
}

module.exports = {
    name: "settings",
    async execute(message) {
        const state = { welcome_messages: true, auto_role: false, level_ups: true };

        const reply = await message.reply({
            components: [buildSettings(state)],
            flags: MessageFlags.IsComponentsV2,
        });

        const collector = reply.createMessageComponentCollector({
            componentType: ComponentType.StringSelect,
            filter: i => i.user.id === message.author.id,
            time: 120_000,
        });

        collector.on("collect", async i => {
            const key = i.values[0];
            state[key] = !state[key];

            await i.update({
                components: [buildSettings(state)],
                flags: MessageFlags.IsComponentsV2,
            });
        });
    }
};
```

-----

## Gotchas & Best Practices

- **Always set the flag.** Forgetting `flags: MessageFlags.IsComponentsV2` is the #1 source of “why isn’t this rendering” bugs.
- **No mixing with embeds.** A message is either classic embeds or CV2 — never both.
- **Combine flags with `|`**, not separate keys: `MessageFlags.IsComponentsV2 | MessageFlags.Ephemeral`.
- **Extract builder functions.** Define `buildXContainer(...)` helpers so you can reuse the same layout for the initial send and every later edit/update.
- **Disable expired components.** On collector `"end"`, edit the message to a disabled-button state so users don’t click dead interactions.
- **Use `attachment://`** for local image buffers in a `MediaGalleryItemBuilder`/`ThumbnailBuilder`, and pass the matching file in `files: [...]`.
- **Discord timestamps still work great inside `TextDisplayBuilder`** — `<t:TIMESTAMP:R>`, `:F>`, `:D>`, `:t>` all render normally.

-----

## Limitations Cheat Sheet

|Limitation               |Detail                                                          |
|-------------------------|----------------------------------------------------------------|
|No nested containers     |A `ContainerBuilder` can’t contain another `ContainerBuilder`   |
|Thumbnail is Section-only|`ThumbnailBuilder` only works via `.setThumbnailAccessory()`    |
|ActionRow capacity       |5 buttons **or** 1 select — never mixed                         |
|Gallery cap              |10 images max per `MediaGalleryBuilder`                         |
|Flag required            |`flags: MessageFlags.IsComponentsV2` must always be set         |
|No embeds alongside CV2  |Pick one system per message                                     |
|Top-level restriction    |Only `ContainerBuilder` goes in the top-level `components` array|

-----

> Made by **void** & **Codez Dev** — [discord.gg/codez](https://discord.gg/codez)