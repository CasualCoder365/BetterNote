import { useEffect, useState } from "react";
import {
  BaseDirectory,
  mkdir,
  readDir,
  readTextFile,
  writeTextFile,
  remove,
} from "@tauri-apps/plugin-fs";
import "./App.css";

type Note = {
  id: string;
  title: string;
  body: string;
  color: string;
  tag: string;
  pinned?: boolean;
};

const initialNotes: Note[] = [
  {
    id: "welcome-1",
    title: "Ideas worth exploring",
    body: "What if the best software removes decisions instead of adding features? Make capturing thoughts feel effortless.",
    color: "lavender",
    tag: "Ideas",
    pinned: true,
  },
  {
    id: "welcome-2",
    title: "Things I'm learning",
    body: "Rust ownership, borrowing, and how memory safety works. Build something small every day.",
    color: "blue",
    tag: "Learning",
  },
  {
    id: "welcome-3",
    title: "Today's priorities",
    body: "Finish the prototype\nBack up my files\nRead for 20 minutes",
    color: "sage",
    tag: "Checklist",
  },
  {
    id: "welcome-4",
    title: "A thought to remember",
    body: "Make the easy things easy, and the hard things possible.",
    color: "peach",
    tag: "Personal",
  },
  {
    id: "welcome-5",
    title: "BetterNote",
    body: "A calm place for everything I want to remember, without the overhead of organizing my life.",
    color: "cream",
    tag: "Project",
    pinned: true,
  },
  {
    id: "welcome-6",
    title: "Little reminders",
    body: "Drink some water\nStretch a little\nTake a proper break",
    color: "pink",
    tag: "Life",
  },
];

const ROOT = "BetterNote";
const METADATA = `${ROOT}/metadata.json`;

async function ensureNotesFolder() {
  await mkdir(ROOT, {
    baseDir: BaseDirectory.Document,
    recursive: true,
  });
}

async function loadNotes(): Promise<Note[]> {
  await ensureNotesFolder();

  let metadata: Record<string, Partial<Note>> = {};

  try {
    metadata = JSON.parse(
      await readTextFile(METADATA, {
        baseDir: BaseDirectory.Document,
      })
    );
  } catch {
    // A missing metadata file is normal on first launch.
  }

  const entries = await readDir(ROOT, {
    baseDir: BaseDirectory.Document,
  });

  const files = entries.filter(
    (entry) => entry.name?.endsWith(".md")
  );

  const loaded: Note[] = [];

  for (const file of files) {
    if (!file.name) continue;

    try {
      const id = file.name.slice(0, -3);
      const markdown = await readTextFile(`${ROOT}/${file.name}`, {
        baseDir: BaseDirectory.Document,
      });

      const match = markdown.match(/^# (.*)(?:\r?\n|$)/);
      const title = match?.[1] || "Untitled note";
      const body = match
        ? markdown.slice(match[0].length).replace(/^\r?\n/, "").replace(/\n$/, "")
        : markdown;

      loaded.push({
        id,
        title,
        body,
        color: metadata[id]?.color || "cream",
        tag: metadata[id]?.tag || "Note",
        pinned: metadata[id]?.pinned || false,
      });
    } catch (error) {
      console.error(`Could not load ${file.name}`, error);
    }
  }

  if (loaded.length === 0) {
    return initialNotes;
  }

  return loaded.sort((a, b) => {
    if (a.pinned !== b.pinned) return a.pinned ? -1 : 1;
    return a.title.localeCompare(b.title);
  });
}

async function saveNotes(notes: Note[]) {
  await ensureNotesFolder();

  const entries = await readDir(ROOT, {
    baseDir: BaseDirectory.Document,
  });

  const existingFiles = new Set(
    entries
      .filter((entry) => entry.name?.endsWith(".md"))
      .map((entry) => entry.name!)
  );

  const currentFiles = new Set(
    notes.map((note) => `${note.id}.md`)
  );

  for (const note of notes) {
    const safeTitle = note.title.replace(/\r?\n/g, " ").trim();
    const markdown = `# ${safeTitle || "Untitled note"}\n\n${note.body}\n`;

    await writeTextFile(`${ROOT}/${note.id}.md`, markdown, {
      baseDir: BaseDirectory.Document,
    });
  }

  // Remove files for notes deleted inside BetterNote.
  for (const filename of existingFiles) {
    if (!currentFiles.has(filename)) {
      await remove(`${ROOT}/${filename}`, {
        baseDir: BaseDirectory.Document,
      });
    }
  }

  const metadata = Object.fromEntries(
    notes.map((note) => [
      note.id,
      {
        color: note.color,
        tag: note.tag,
        pinned: Boolean(note.pinned),
      },
    ])
  );

  await writeTextFile(METADATA, JSON.stringify(metadata, null, 2), {
    baseDir: BaseDirectory.Document,
  });
}

function App() {
  const [notes, setNotes] = useState<Note[]>([]);
  const [query, setQuery] = useState("");
  const [activeView, setActiveView] = useState("All notes");
  const [ready, setReady] = useState(false);
  const [saveStatus, setSaveStatus] = useState("Loading notes…");

  useEffect(() => {
    let cancelled = false;

    loadNotes()
      .then((loaded) => {
        if (cancelled) return;
        setNotes(loaded);
        setReady(true);
        setSaveStatus("Saved locally");
      })
      .catch((error) => {
        console.error("Could not load BetterNote", error);
        if (cancelled) return;
        setSaveStatus("Could not load notes");
      });

    return () => {
      cancelled = true;
    };
  }, []);

  useEffect(() => {
    if (!ready) return;

    setSaveStatus("Saving…");

    const timeout = window.setTimeout(() => {
      saveNotes(notes)
        .then(() => setSaveStatus("Saved locally"))
        .catch((error) => {
          console.error("Could not save BetterNote", error);
          setSaveStatus("Save failed");
        });
    }, 500);

    return () => window.clearTimeout(timeout);
  }, [notes, ready]);

  const visibleNotes = notes.filter((note) => {
    const matchesQuery =
      `${note.title} ${note.body} ${note.tag}`
        .toLowerCase()
        .includes(query.toLowerCase());

    const matchesView =
      activeView === "All notes" ||
      (activeView === "Pinned" && note.pinned);

    return matchesQuery && matchesView;
  });

  function createNote() {
    const note: Note = {
      id: crypto.randomUUID(),
      title: "Untitled note",
      body: "",
      color: "cream",
      tag: "Note",
    };

    setNotes((current) => [note, ...current]);
    setActiveView("All notes");
    setQuery("");
  }

  function updateNote(id: string, changes: Partial<Note>) {
    setNotes((current) =>
      current.map((note) =>
        note.id === id ? { ...note, ...changes } : note
      )
    );
  }

  async function deleteNote(id: string) {
    try {
      await remove(`${ROOT}/${id}.md`, {
        baseDir: BaseDirectory.Document,
      });
    } catch (error) {
      // The file might not exist yet if it has just been created.
      console.warn("Could not remove note file yet", error);
    }

    setNotes((current) => current.filter((note) => note.id !== id));
  }

  return (
    <div className="app-shell">
      <aside className="sidebar">
        <div className="brand">
          <div className="brand-icon">b.</div>
          <span>betternote</span>
        </div>

        <button className="new-note-button" onClick={createNote}>
          <span>＋</span> New note
          <kbd>Ctrl N</kbd>
        </button>

        <div className="nav-label">LIBRARY</div>

        <button
          className={`nav-item ${activeView === "All notes" ? "active" : ""}`}
          onClick={() => setActiveView("All notes")}
        >
          <span>▦</span> All notes
          <span className="nav-count">{notes.length}</span>
        </button>

        <button
          className={`nav-item ${activeView === "Pinned" ? "active" : ""}`}
          onClick={() => setActiveView("Pinned")}
        >
          <span>♧</span> Pinned
        </button>

        <div className="sidebar-bottom">
          <div className="sync-indicator">
            <span className="status-dot" />
            <div>
              <strong>Your space</strong>
              <small>{saveStatus}</small>
            </div>
          </div>
          <div className="version-label">BETTERNOTE · PREVIEW</div>
        </div>
      </aside>

      <main className="main-content">
        <header className="topbar">
          <div className="breadcrumb">
            <span>Library</span><span className="slash">/</span>
            <strong>{activeView}</strong>
          </div>
          <div className="topbar-right">
            <span className="local-badge">
              <span className="status-dot" /> Local workspace
            </span>
            <div className="avatar">B</div>
          </div>
        </header>

        <section className="workspace">
          <div className="welcome-row">
            <div>
              <div className="eyebrow">YOUR PERSONAL SPACE</div>
              <h1>{activeView === "Pinned" ? "Your favorites." : "A place to think."}</h1>
              <p className="subtitle">
                All your thoughts, in one place. Nothing to organize.
              </p>
            </div>
            <button className="create-button" onClick={createNote}>
              <span>＋</span> Create a note
            </button>
          </div>

          <div className="toolbar">
            <div className="search-box">
              <span className="search-icon">⌕</span>
              <input
                value={query}
                onChange={(event) => setQuery(event.target.value)}
                placeholder="Find anything in your notes..."
                aria-label="Search notes"
              />
              <kbd>/</kbd>
            </div>
            <span className="note-total">
              {visibleNotes.length} {visibleNotes.length === 1 ? "note" : "notes"}
            </span>
          </div>

          {!ready ? (
            <div className="empty-state">
              <h2>Opening your notes…</h2>
            </div>
          ) : visibleNotes.length > 0 ? (
            <div className="notes-grid">
              {visibleNotes.map((note) => (
                <article className={`note-card ${note.color}`} key={note.id}>
                  <div className="card-top">
                    <span className="note-tag">{note.tag}</span>
                    <button
                      className={`pin-button ${note.pinned ? "is-pinned" : ""}`}
                      title={note.pinned ? "Unpin note" : "Pin note"}
                      aria-label={note.pinned ? "Unpin note" : "Pin note"}
                      onClick={() =>
                        updateNote(note.id, { pinned: !note.pinned })
                      }
                    >
                      {note.pinned ? "◆" : "◇"}
                    </button>
                  </div>

                  <input
                    className="card-title"
                    value={note.title}
                    onChange={(event) =>
                      updateNote(note.id, { title: event.target.value })
                    }
                    aria-label="Note title"
                  />

                  <textarea
                    className="card-body"
                    value={note.body}
                    onChange={(event) =>
                      updateNote(note.id, { body: event.target.value })
                    }
                    placeholder="Start writing..."
                    aria-label="Note content"
                    rows={4}
                  />

                  <div className="card-footer">
                    <select
                      value={note.color}
                      onChange={(event) =>
                        updateNote(note.id, { color: event.target.value })
                      }
                      aria-label="Card color"
                    >
                      <option value="cream">Cream</option>
                      <option value="lavender">Lavender</option>
                      <option value="blue">Blue</option>
                      <option value="sage">Sage</option>
                      <option value="peach">Peach</option>
                      <option value="pink">Pink</option>
                    </select>
                    <button
                      className="delete-button"
                      onClick={() => deleteNote(note.id)}
                      title="Delete note"
                      aria-label="Delete note"
                    >
                      Delete
                    </button>
                  </div>
                </article>
              ))}
            </div>
          ) : (
            <div className="empty-state">
              <div className="empty-icon">⌕</div>
              <h2>Nothing here just yet.</h2>
              <p>Try another search, or create a new note.</p>
              <button className="create-button" onClick={createNote}>
                ＋ Create a note
              </button>
            </div>
          )}

          <footer className="workspace-footer">
            <span>Made for the way your mind works.</span>
            <span>Less organizing. More thinking.</span>
          </footer>
        </section>
      </main>
    </div>
  );
}

export default App;