---
cssclasses:
  - hide-toolbar
  - hide-properties
  - inbox-feed
mode: read_only
modified: 2026-06-12
---
```datacorejsx
// -- Configuration ----------------------------------------------------------
// Folders to scan for new activity. Add or remove paths to match your vault.
// Each entry is a folder prefix � any note whose path starts with one of
// these strings will appear in the feed.
// Examples:
//   "Inbox/"         � a dedicated inbox folder
//   "Projects/"      � all notes under Projects
//   "Journal/"       � daily/weekly journals
const SOURCE_FOLDERS = [
  "Inbox/",
  "Projects/",
  "Notes/",
];

// localStorage key used to persist which groups are open/closed.
const STORAGE_KEY = "activity-feed-collapse-states";

// Group labels shown in the UI. Order matters � listed top-to-bottom.
const GROUPS = ["Recent (Last 3 Days)", "Past 7 Days", "Monthly"];
// ---------------------------------------------------------------------------

const savedStates = (() => {
  try {
    return JSON.parse(localStorage.getItem(STORAGE_KEY) || "{}");
  } catch {
    return {};
  }
})();

const saveState = (groupName, isOpen) => {
  savedStates[groupName] = isOpen;
  localStorage.setItem(STORAGE_KEY, JSON.stringify(savedStates));
};

const toDate = (value) => {
  if (!value) return null;
  if (value instanceof Date) return isNaN(value.getTime()) ? null : value;
  if (typeof value?.toJSDate === "function") {
    const d = value.toJSDate();
    return isNaN(d.getTime()) ? null : d;
  }
  if (typeof value === "number") {
    const d = new Date(value);
    return isNaN(d.getTime()) ? null : d;
  }
  if (typeof value === "string") {
    const d = new Date(value);
    return isNaN(d.getTime()) ? null : d;
  }
  return null;
};

const relativeTime = (date) => {
  const now = new Date();
  const diffMs = date.getTime() - now.getTime();
  const absMs = Math.abs(diffMs);
  const rtf = new Intl.RelativeTimeFormat("en", { numeric: "auto" });
  const minute = 60 * 1000;
  const hour = 60 * minute;
  const day = 24 * hour;
  const week = 7 * day;
  const month = 30 * day;
  const year = 365 * day;

  if (absMs < hour) return rtf.format(Math.round(diffMs / minute), "minute");
  if (absMs < day) return rtf.format(Math.round(diffMs / hour), "hour");
  if (absMs < week) return rtf.format(Math.round(diffMs / day), "day");
  if (absMs < month) return rtf.format(Math.round(diffMs / week), "week");
  if (absMs < year) return rtf.format(Math.round(diffMs / month), "month");
  return rtf.format(Math.round(diffMs / year), "year");
};

return function InboxFeed() {
  const [items, setItems] = dc.useState([]);
  const [openGroups, setOpenGroups] = dc.useState(() => {
    const next = {};
    for (const group of GROUPS) next[group] = savedStates[group] ?? true;
    return next;
  });

  const buildItems = () => {
    const files = app.vault.getMarkdownFiles()
      .filter((file) => SOURCE_FOLDERS.some((prefix) => file.path.startsWith(prefix)));

    const results = files.map((file) => {
      const cache = app.metadataCache.getFileCache(file);
      const fm = cache?.frontmatter ?? {};
      // Priority order for choosing the "date" of each entry.
      // Frontmatter fields are tried first, then file system timestamps.
      const orderedFields = [
        fm.published, fm.date, fm.created, fm.received,
        fm.modified, fm.updated, fm.last_modified, fm.lastUpdated,
        file.stat.mtime, file.stat.ctime,
      ];

      let chosenDate = null;
      for (const candidate of orderedFields) {
        const d = toDate(candidate);
        if (d) { chosenDate = d; break; }
      }

      return {
        path: file.path,
        name: file.basename,
        folder: file.parent?.path ?? "",
        date: chosenDate,
      };
    }).filter((entry) => entry.date);

    results.sort((a, b) => b.date.getTime() - a.date.getTime());
    setItems(results);
  };

  dc.useEffect(() => {
    buildItems();
    const vaultRefs = [
      app.vault.on("create", buildItems),
      app.vault.on("delete", buildItems),
      app.vault.on("rename", buildItems),
      app.vault.on("modify", buildItems),
    ];
    const metadataRef = app.metadataCache.on("resolved", buildItems);

    return () => {
      vaultRefs.forEach((ref) => app.vault.offref(ref));
      app.metadataCache.offref(metadataRef);
    };
  }, []);

  const grouped = {
    "Recent (Last 3 Days)": [],
    "Past 7 Days": [],
    "Monthly": [],
  };

  const now = new Date();
  for (const entry of items) {
    const diffDays = Math.floor((now.getTime() - entry.date.getTime()) / 86400000);
    if (diffDays <= 3) grouped["Recent (Last 3 Days)"].push(entry);
    else if (diffDays <= 7) grouped["Past 7 Days"].push(entry);
    else grouped["Monthly"].push(entry);
  }

  const toggleGroup = (group) => {
    setOpenGroups((current) => {
      const updated = { ...current, [group]: !current[group] };
      saveState(group, updated[group]);
      return updated;
    });
  };

  // Opens note in a new tab
  const openPage = (path) => app.workspace.openLinkText(path, "", true);

  // Derives a short label from the folder path to show as the avatar letter.
  // Customize this to match your SOURCE_FOLDERS.
  const getLabel = (folder) => {
    const lower = folder.toLowerCase();
    if (lower.includes("project")) return "P";
    if (lower.includes("journal")) return "J";
    if (lower.includes("inbox")) return "I";
    return folder.charAt(0).toUpperCase() || "N";
  };

  return (
    <div className="inbox-feed-root">
      {GROUPS.map((groupName) => {
        const groupItems = grouped[groupName] ?? [];
        if (!groupItems.length) return null;

        return (
          <details key={groupName} className="inbox-feed-group" open={openGroups[groupName]}>
            <summary className="inbox-feed-summary" onClick={(event) => {
              event.preventDefault();
              toggleGroup(groupName);
            }}>
              {groupName}
            </summary>

            <div className="inbox-feed-container">
              {groupItems.map((entry) => {
                const label = getLabel(entry.folder);
                const folderPath = entry.folder.replace(/\//g, " / ");

                return (
                  <div
                    key={entry.path}
                    className="inbox-feed-item"
                    onClick={() => openPage(entry.path)}
                    role="button"
                    tabIndex={0}
                    onKeyDown={(event) => {
                      if (event.key === "Enter" || event.key === " ") {
                        event.preventDefault();
                        openPage(entry.path);
                      }
                    }}
                  >
                    <div className="inbox-feed-avatar">{label}</div>
                    <div className="inbox-feed-body">
                      <div className="inbox-feed-header">
                        <span className="inbox-feed-prefix">New entry: </span>
                        <span className="inbox-feed-title">{entry.name}</span>
                      </div>
                      <div className="inbox-feed-subtext">
                        {relativeTime(entry.date)} � {folderPath}
                      </div>
                    </div>
                  </div>
                );
              })}
            </div>
          </details>
        );
      })}
    </div>
  );
}
```
