## Description

A live **activity feed** for your Obsidian vault styled like a social inbox, showing recently created or modified notes grouped by recency. Built with [Datacore](https://github.com/blacksmithgu/datacore) JSX.

---

## What it does

- Scans a configurable list of **source folders** for Markdown notes
- Groups them into **Recent (Last 3 Days)**, **Past 7 Days**, and **Monthly** buckets
- Shows each note with:
  - A **letter avatar** derived from its folder (customisable)
  - The note name and relative timestamp ("2 days ago", "last week", etc.)
  - The folder path as breadcrumb subtext
- Each group is a collapsible `<details>` element; open/closed state **persists across sessions** via `localStorage`
- Clicking any row opens the note in a **new tab**
- Reactively re-renders on vault changes (create / delete / rename / modify)

---

## Requirements

| Dependency | Notes |
|------------|-------|
| [Datacore plugin](https://github.com/blacksmithgu/datacore) | Core renderer enable in Community Plugins |
| `inbox-datacore-feed.css` snippet | Included in `snippets/` copy to `.obsidian/snippets/` |

---

## Installation

1. **Install Datacore** from Community Plugins (search "Datacore").
2. Copy `snippets/inbox-datacore-feed.css` ? your vault's `.obsidian/snippets/` folder.
3. In Obsidian ? **Settings ? Appearance ? CSS Snippets**, enable **inbox-datacore-feed**.
4. Copy `Inbox Datacore Feed.md` anywhere in your vault (e.g. a `Dashboards/` folder).
5. Open the note the feed renders immediately.

---

## Configuration

All settings live at the top of the `datacorejsx` block:

```js
// Folders to scan for activity. Any note whose path starts with
// one of these prefixes appears in the feed.
const SOURCE_FOLDERS = [
  "Inbox/",
  "Projects/",
  "Notes/",
];
```

**Change these to match your vault's structure.** For example:

```js
const SOURCE_FOLDERS = [
  "Journal/",
  "Areas/",
  "Resources/",
];
```

### Avatar labels

The `getLabel(folder)` function maps folder names to the single-letter initials shown in each row's avatar. Customise it to match your folders:

```js
const getLabel = (folder) => {
  const lower = folder.toLowerCase();
  if (lower.includes("project")) return "P";
  if (lower.includes("journal")) return "J";
  if (lower.includes("inbox"))   return "I";
  return folder.charAt(0).toUpperCase() || "N";
};
```

### Date priority

The view picks the "date" of each note by trying frontmatter fields in this order, then falling back to file system timestamps:

```
published ? date ? created ? received ? modified ? updated ?
last_modified ? lastUpdated ? file.mtime ? file.ctime
```

If none of these exist, the note is **excluded** from the feed.

### Group time buckets

Change the cutoffs by editing the grouping logic in the `InboxFeed` component:

```js
if (diffDays <= 3) grouped["Recent (Last 3 Days)"].push(entry);
else if (diffDays <= 7) grouped["Past 7 Days"].push(entry);
else grouped["Monthly"].push(entry);
```

---
## CSS snippet

`snippets/inbox-datacore-feed.css` styles the feed layout. All selectors are scoped to `.inbox-feed` so they will not affect other notes. Theme variables (`--text-accent`, `--background-modifier-hover`, etc.) are used throughout, so the feed adapts automatically to light/dark themes and custom themes.

---
## License

MIT use freely, modify, share.


## Attached Snippets

- `snippets/inbox-datacore-feed.css` - inbox-datacore-feed

## Screenshots

![Screenshot 1](https://i.ibb.co/1GxZ5GWk/14f99012f7d5.png)
## Installation

1. Download the `.md` file(s) from this repo
2. Place them in your vault
3. Copy the attached CSS snippet files into your vault's CSS snippets folder
4. Enable them in Settings > Appearance > CSS Snippets

---
*Published via [Vault Hub](https://obsidianvaulthub.com)*
