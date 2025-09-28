


```dataviewjs
// ---- Config ---------------------------------------------------------------
const ROOT_FOLDER = "Daily";
const DATE_FMT = "DD/MM/YYYY";
// Section names we’ll accept (case/emoji/spacing insensitive)
const ACCEPT = ["what i did today", "what i did"];
// ---------------------------------------------------------------------------

// Utilities
const normalize = (s) =>
  (s ?? "")
    // remove most punctuation & emoji; keep letters/digits/spaces
    .replace(/[^\p{L}\p{N} ]+/gu, " ")
    .replace(/\s+/g, " ")
    .trim()
    .toLowerCase();

function findSectionByHeading(raw, cache, targets) {
  if (!cache?.headings?.length) return "";
  const lines = raw.split("\n");
  const H = cache.headings;

  // Find the heading index that matches our targets
  let idx = -1;
  for (let i = 0; i < H.length; i++) {
    const htext = normalize(H[i].heading);
    if (targets.includes(htext)) { idx = i; break; }
  }
  if (idx === -1) return "";

  const startLine = H[idx].position.end.line + 1; // first line after heading
  const level = H[idx].level;

  // Find the next heading with level <= current to bound the section
  let endLine = lines.length;
  for (let j = idx + 1; j < H.length; j++) {
    if (H[j].level <= level) { endLine = H[j].position.start.line; break; }
  }
  return lines.slice(startLine, endLine).join("\n").trim();
}


// Fallback: render nested bullets if we can’t use dv.markdown
function renderNestedBullets(md) {
  const container = document.createElement("div");
  const root = document.createElement("ul");
  container.appendChild(root);
  const stack = [{ ul: root, depth: 0 }];

  const lines = md.split("\n");
  for (let raw of lines) {
    if (!raw.trim()) continue;
    const m = raw.match(/^([ \t]*)([-*])\s+(.*)$/);
    if (!m) continue;

    const indent = m[1] ?? "";
    let units = 0;
    for (const ch of indent) units += ch === "\t" ? 2 : 1; // tab ≈ 2 spaces
    let depth = Math.floor(units / 2);
    depth = Math.max(0, depth);

    while (depth > stack[stack.length - 1].depth) {
      const parentLI =
        stack[stack.length - 1].ul.lastElementChild ??
        stack[stack.length - 1].ul.appendChild(document.createElement("li"));
      const newUL = document.createElement("ul");
      parentLI.appendChild(newUL);
      stack.push({ ul: newUL, depth: stack[stack.length - 1].depth + 1 });
    }
    while (depth < stack[stack.length - 1].depth) stack.pop();

    const li = document.createElement("li");
    li.textContent = m[3];
    stack[stack.length - 1].ul.appendChild(li);
  }
  return container;
}

// Collect daily pages under Daily/YYYY/MM where filename is DD
const pages = dv.pages('"' + ROOT_FOLDER + '"')
  .where(p => /^Daily\/\d{4}\/\d{2}$/.test(p.file.folder))
  .where(p => /^\d{1,2}$/.test(p.file.name));

let rows = [];

for (const p of pages) {
  const fm = p.file.folder.match(/^Daily\/(\d{4})\/(\d{2})$/);
  if (!fm) continue;
  const [, Y, M] = fm;
  const D = String(p.file.name).padStart(2, "0");

  const m = window.moment(`${Y}-${M}-${D}`, "YYYY-MM-DD", true);
  const dateObj = m.isValid() ? m : window.moment(p.file.ctime);

  // Date cell: clickable DD/MM/YYYY
  const dateEl = dv.fileLink(p.file.path, false, dateObj.format(DATE_FMT));

  // Read + parse via heading cache
  const af = app.vault.getAbstractFileByPath(p.file.path);
  const raw = await app.vault.cachedRead(af);
  const cache = app.metadataCache.getFileCache(af);

  const sectionMd =
    findSectionByHeading(raw, cache, ACCEPT) ||
    ""; // empty string if not found

  // Render the section (prefer dv.markdown if available)
  let whatEl;
  if (sectionMd) {
    if (typeof dv.markdown === "function") {
      whatEl = dv.el("div", "");
      await dv.markdown(sectionMd, whatEl);
    } else {
      whatEl = renderNestedBullets(sectionMd);
      if (whatEl.querySelectorAll("li").length === 0) {
        whatEl = dv.el("div", sectionMd);
      }
    }
  } else {
    whatEl = dv.el("div", "(no “What I Did” section found)");
  }

  rows.push({
    sortKey: dateObj.valueOf(),
    dateEl,
    whatEl
  });
}

// Newest first (change to a.sortKey - b.sortKey for oldest→newest)
rows.sort((a, b) => b.sortKey - a.sortKey);

dv.table(["Date", "What I Did"], rows.map(r => [r.dateEl, r.whatEl]));

```

