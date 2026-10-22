# Building a Desktop App in Rust — A Hands-On Guide

This guide tells you **what to do**, step by step. It does not give you code to copy.
You write every line yourself — that's how Rust sticks.

**Stack:** Rust + [`eframe`/`egui`](https://github.com/emilk/egui) (pure-Rust, immediate-mode GUI).
**Platform focus:** Windows 11 (works the same on macOS/Linux except setup).
**End result:** a data-visualization desktop app that loads a CSV file, plots it, remembers settings, and ships as a `.exe`.

---

## Table of Contents

0. [Setup](#phase-0--setup)
1. [Create the Project](#phase-1--create-the-project)
2. [First Window](#phase-2--first-window)
3. [State & Interaction](#phase-3--state--interaction)
4. [Layout](#phase-4--layout)
5. [Code Organization](#phase-5--code-organization)
6. [Loading Data](#phase-6--loading-data)
7. [Plotting](#phase-7--plotting)
8. [Responsive UI (Background Work)](#phase-8--responsive-ui-background-work)
9. [Error Handling](#phase-9--error-handling)
10. [Persistence](#phase-10--persistence)
11. [Testing](#phase-11--testing)
12. [Polish](#phase-12--polish)
13. [Release & Distribution](#phase-13--release--distribution)
- [Rust Concepts Checklist](#rust-concepts-checklist)
- [Troubleshooting](#troubleshooting)
- [Resources](#resources)

---

## How to Use This Guide

- Do the phases **in order**. Each one builds on the last.
- After **every step**, run `cargo run`. Small steps = small errors.
- Read compiler errors **fully**. Rust's errors usually tell you the fix.
- Run `cargo clippy` often — it teaches idiomatic Rust.
- Keep [docs.rs/egui](https://docs.rs/egui) and [docs.rs/eframe](https://docs.rs/eframe) open.
- Each phase ends with a **✅ Checkpoint**. Don't move on until you hit it.
- Commit to git at every checkpoint.

---

## Phase 0 — Setup

**Goal:** a working Rust toolchain.

1. Go to <https://rustup.rs> and download `rustup-init.exe`.
2. Run it. When prompted, let it install **Visual Studio C++ Build Tools** (workload: *Desktop development with C++*). Rust uses the MSVC linker on Windows.
3. Accept the default installation (stable toolchain).
4. **Open a new terminal** (so PATH refreshes) and verify:
   - `rustc --version`
   - `cargo --version`
5. Install components:
   - `rustup component add clippy rustfmt`
6. Editor: install **VS Code** + the **rust-analyzer** extension. Optional: **Even Better TOML**, **Error Lens**, **CodeLLDB** (debugger).

✅ **Checkpoint:** both version commands print a version number.

---

## Phase 1 — Create the Project

**Goal:** a compiling Rust binary.

1. In this repo folder, run `cargo new app` (or any name you like), then `cd app`.
2. Explore what Cargo created:
   - `Cargo.toml` — project metadata and dependencies.
   - `src/main.rs` — the entry point (`fn main`).
   - `.gitignore` — already ignores `target/`.
3. Run `cargo run`. You should see `Hello, world!`.
4. Add the GUI framework: `cargo add eframe`.
5. Open `Cargo.toml` and notice `eframe` was added under `[dependencies]`.
6. Run `cargo build` once — the first build downloads and compiles many crates; it takes a while. Later builds are fast.

✅ **Checkpoint:** `cargo build` succeeds with `eframe` as a dependency.

---

## Phase 2 — First Window

**Goal:** an empty native window with a title and a heading.

**Concepts:** structs, traits, closures, `Box`.

In `src/main.rs`:

1. Define a **struct** (e.g. `MyApp`). Leave it empty for now. Derive `Default` on it.
2. **Implement the `eframe::App` trait** for your struct.
   - The trait needs one method: `update`. It receives `&mut self`, an `egui::Context`, and an `eframe::Frame`.
   - `update` is called **every frame** (whenever something changes or the user interacts).
3. Inside `update`, show an `egui::CentralPanel` (use its `default()` then `show(...)`), and inside the closure call `ui.heading(...)` with some text.
4. In `main`:
   - Create `eframe::NativeOptions` (default is fine).
   - Call `eframe::run_native` with: a window title string, the options, and a closure that returns your app wrapped in a `Box` (check the docs for the exact return type expected — it changes slightly between eframe versions).
   - `main` should return what `run_native` returns (an `eframe::Result`).
5. `cargo run`.

**Hint:** look at the official example `examples/hello_world` in the egui GitHub repo. *Read* it, close it, then write yours from memory.

✅ **Checkpoint:** a window opens showing your heading. Closing it ends the program.

---

## Phase 3 — State & Interaction

**Goal:** widgets that read and change your app's state.

**Concepts:** `&mut self`, ownership of `String`, immediate-mode UI.

The key idea of egui: **you don't create widgets once and keep them.** Every frame you describe the UI from your state. Change the state → the next frame looks different.

1. Add fields to your struct:
   - a counter (integer),
   - a name (`String`),
   - a boolean (e.g. "show details").
2. In `update`:
   - Add a `ui.button(...)`. It returns a `Response`; check `.clicked()` and increment the counter.
   - Add a `ui.label(...)` that displays the counter (use `format!`).
   - Add a `ui.text_edit_singleline(...)` bound to `&mut self.name`.
   - Add a `ui.checkbox(...)` bound to the boolean.
   - If the boolean is true, show an extra label (conditional UI is just an `if`).
3. Add a `ui.add(egui::Slider::new(...))` bound to a numeric field with a range.
4. Add a `ui.separator()` between sections.

✅ **Checkpoint:** clicking, typing, toggling, and sliding all update what's shown instantly.

---

## Phase 4 — Layout

**Goal:** a real app shell: menu bar, side panel, main area, status bar.

**Order matters:** in egui you must add **side/top/bottom panels before the `CentralPanel`**.

1. **Top menu bar:** `egui::TopBottomPanel::top("some_id")`. Inside, use `egui::menu::bar` (or `egui::MenuBar` in newer versions — check docs). Add a **File** menu with:
   - *Open…* (does nothing yet)
   - *Quit* — closes the window (look up `ViewportCommand::Close` and `ctx.send_viewport_cmd`).
2. **Left side panel:** `egui::SidePanel::left("id")`. Put controls here (your slider, checkbox). Make it `resizable`.
3. **Bottom status bar:** `TopBottomPanel::bottom("id")` with a small label like "Ready".
4. **Central panel:** your main content.
5. Inside panels, try `ui.horizontal(|ui| ...)` and `ui.vertical(...)` to group widgets, and `egui::ScrollArea::vertical()` for long content.
6. Try `egui::Grid` for a label/value form layout.
7. Set an initial window size via `NativeOptions` → `viewport` → `with_inner_size`.

✅ **Checkpoint:** menu, side panel, status bar and main area all visible; *Quit* closes the app.

---

## Phase 5 — Code Organization

**Goal:** separate UI from logic before the project grows.

**Concepts:** modules, `mod`, `pub`, `use`.

Create this structure:

```
src/
  main.rs     ← only launches the window
  app.rs      ← your App struct + eframe::App impl (UI only)
  model.rs    ← data types and pure logic (NO egui imports)
  loader.rs   ← reading files (added in Phase 6)
```

1. Move your struct and its `impl eframe::App` into `app.rs`. Mark the struct `pub`.
2. In `main.rs`, declare `mod app;` (and the others) and `use` your struct.
3. Rule of thumb: **`model.rs` must never import `egui`.** If logic needs the UI, your design is leaking.
4. If `update` gets long, split it into private methods: `fn menu_bar(&mut self, ctx)`, `fn side_panel(...)`, `fn central(...)`.

✅ **Checkpoint:** app still runs exactly as before, but `main.rs` is under ~20 lines.

---

## Phase 6 — Loading Data

**Goal:** let the user pick a CSV file and load it into memory.

**Concepts:** `Result`, `?`, `serde` derive, `Vec<T>`, `PathBuf`, `Option`.

1. Add crates:
   - `cargo add rfd` (native file dialogs)
   - `cargo add csv`
   - `cargo add serde --features derive`
2. Create a test CSV in the repo (e.g. `sample_data/sales.csv`) with a header and 2 numeric columns like `x,y` — 20+ rows.
3. In `model.rs`, define a struct for **one row** (e.g. `Record { x: f64, y: f64 }`) and derive `serde::Deserialize` and `Debug`, `Clone`.
4. In `loader.rs`, write a function that takes a path and returns a `Result` containing a `Vec` of records:
   - Open a `csv::Reader` from the path.
   - Iterate `deserialize()` and collect into a `Vec`, using `?` to propagate errors.
5. In your app state, add:
   - `Option<PathBuf>` for the selected file,
   - `Vec<Record>` for loaded data.
6. Wire **File → Open…**: call `rfd::FileDialog::new()`, add a filter for `csv`, call `pick_file()`. If it returns `Some(path)`, call your loader and store the result.
7. In the central panel, display: file name, row count, and the first ~10 rows in an `egui::Grid`.

✅ **Checkpoint:** you can open your sample CSV and see its rows in the window.

> Stretch: use the `egui_extras` crate's `TableBuilder` for a proper scrollable table.

---

## Phase 7 — Plotting

**Goal:** visualize the loaded data.

**Concepts:** iterators, `map`, `collect`.

1. `cargo add egui_plot` — **make sure its version matches your `egui` version** (check `Cargo.lock` or `cargo tree | findstr egui`). Mismatched versions are the #1 compile error here.
2. Convert your `Vec<Record>` into a list of `[f64; 2]` points using an iterator (`iter().map(...).collect()`).
3. In the central panel, create an `egui_plot::Plot` with an id, and inside its `show` closure add a `Line` built from `PlotPoints`.
4. Add controls in the side panel:
   - a dropdown (`egui::ComboBox`) to choose chart type: Line / Points / Bars,
   - a checkbox for "show grid" / "show legend",
   - a color picker (`ui.color_edit_button_srgba`) for line color.
5. Store those choices in your app state and use them when drawing (`match` on an enum for the chart type).
6. Compute simple statistics in **`model.rs`** (min, max, mean of `y`) as a pure function, and show them in the side panel.

✅ **Checkpoint:** opening a CSV draws a chart; changing chart type/color updates it live; stats are shown.

> Stretch: let the user pick which CSV columns are X and Y (needs reading headers and storing rows as `Vec<f64>` per column instead of a fixed struct).

---

## Phase 8 — Responsive UI (Background Work)

**Goal:** loading a big file must not freeze the window.

**Concepts:** threads, `move` closures, channels, `Send`.

1. Generate a large CSV (hundreds of thousands of rows) and open it — notice the window freezes. That's the problem.
2. Add to your app state: a `Option<std::sync::mpsc::Receiver<...>>` and a `bool` "loading".
3. When the user picks a file:
   - Create a channel (`mpsc::channel()`).
   - `std::thread::spawn` a `move` closure that calls your loader and **sends** the `Result` through the sender.
   - Store the receiver in state; set loading = true.
4. In `update`, every frame: if you have a receiver, call `try_recv()` (non-blocking). If you got a result, store the data, drop the receiver, set loading = false.
5. While loading, show a `ui.spinner()` and disable the Open button (`ui.add_enabled(...)`).
6. Call `ctx.request_repaint()` while loading so `update` keeps running even if the mouse doesn't move. Better: clone the `egui::Context` into the thread and call `request_repaint()` from there after sending.

✅ **Checkpoint:** opening a huge file shows a spinner; the window stays movable and responsive.

---

## Phase 9 — Error Handling

**Goal:** no panics, no `unwrap()` on user input; errors shown nicely.

**Concepts:** `anyhow`, `thiserror`, `?`, `Context`.

1. `cargo add anyhow`. Make your loader return `anyhow::Result<...>`.
2. Use `.with_context(|| ...)` to add human-readable messages ("Failed to read file X").
3. In app state, add `Option<String>` for the last error.
4. When a load fails, store the error message (use `format!("{err:#}")` to include the full chain).
5. Show it: either a red label (`ui.colored_label`) in the status bar, or an `egui::Window` popup with an OK button that clears it.
6. Test it: open a non-CSV file, a CSV with text in a number column, an empty file.
7. Search your code for `unwrap()` and `expect()` — replace any that can fail due to user input.

> Optional: in `model.rs`, define your own error enum with `thiserror` (`cargo add thiserror`) for domain errors like "empty dataset".

✅ **Checkpoint:** bad files produce a friendly message, never a crash.

---

## Phase 10 — Persistence

**Goal:** the app remembers settings (last file, chart type, color, panel width) between runs.

**Concepts:** `Serialize`/`Deserialize`, `#[serde(skip)]`.

1. Enable eframe's persistence feature: edit the `eframe` line in `Cargo.toml` to include `features = ["persistence"]` (keep the default features too).
2. Derive `serde::Serialize` and `serde::Deserialize` on your app struct (and on the enums it contains).
3. Mark fields that should **not** be saved with `#[serde(skip)]`: the loaded data `Vec`, the receiver, the error, the loading flag.
4. Implement the `save` method of `eframe::App`: call `eframe::set_value` with the storage and `eframe::APP_KEY`.
5. In your app's constructor (the closure in `run_native` receives a `CreationContext`), check `cc.storage`; if present, load with `eframe::get_value`, otherwise use `Default`.
6. Bonus: on startup, if a "last file" path was restored, auto-load it (in the background thread from Phase 8).

✅ **Checkpoint:** change settings, close, reopen — everything is as you left it.

---

## Phase 11 — Testing

**Goal:** confidence that logic is correct, independent of the GUI.

**Concepts:** `#[cfg(test)]`, `#[test]`, `assert_eq!`.

1. In `model.rs`, add a `#[cfg(test)] mod tests` block.
2. Write tests for your statistics function: normal data, single value, empty input (what should happen? decide and test it).
3. In `loader.rs`, test loading `sample_data/sales.csv` and a deliberately broken CSV (put test fixtures in `tests/fixtures/`).
4. Run `cargo test`.
5. Run `cargo clippy -- -D warnings` and fix everything.
6. Run `cargo fmt`.

This is why Phase 5 mattered: pure logic in `model.rs` is trivially testable.

✅ **Checkpoint:** `cargo test`, `cargo clippy`, `cargo fmt --check` all pass.

---

## Phase 12 — Polish

Pick what you like:

- **Dark/light toggle:** `ctx.set_visuals(egui::Visuals::dark()/light())`; persist the choice.
- **Keyboard shortcuts:** check `ctx.input(|i| i.key_pressed(...))` with modifiers, e.g. Ctrl+O to open.
- **Drag & drop files:** read `ctx.input(|i| i.raw.dropped_files)` and load the dropped path.
- **Fonts:** load a custom font via `egui::FontDefinitions` in the constructor.
- **Export chart data / stats** to a file using `rfd::FileDialog::save_file()`.
- **Logging:** `cargo add log env_logger` (or `tracing`), log load times and errors.
- **About window:** `egui::Window` showing version from `env!("CARGO_PKG_VERSION")`.
- **Window icon:** `NativeOptions.viewport.with_icon(...)` (decode a PNG with the `image` crate).

---

## Phase 13 — Release & Distribution

1. Build optimized: `cargo build --release`. Output: `target/release/<name>.exe`.
2. Hide the console window in release builds: put this crate-level attribute at the very top of `main.rs`:
   `#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]`
   (This is a config attribute, not app code — it's fine to copy.)
3. Shrink the binary — in `Cargo.toml` add a `[profile.release]` section with `lto = true`, `codegen-units = 1`, `strip = true`, and optionally `opt-level = "z"`. Compare sizes.
4. Embed a Windows `.exe` icon & metadata: look at the `winres` or `embed-resource` crate (needs a `build.rs`).
5. Create an installer:
   - `cargo install cargo-packager` → configure in `Cargo.toml` → produces `.msi`/NSIS `.exe`, **or**
   - `cargo install cargo-wix` → `cargo wix init` → `cargo wix`.
6. **Code signing** (optional, costs money) avoids Windows SmartScreen warnings.
7. **CI:** add a GitHub Actions workflow that runs `cargo fmt --check`, `cargo clippy`, `cargo test` on push, and builds release binaries on tags. Attach the `.exe` to a GitHub Release.

✅ **Checkpoint:** someone else can download your installer and run the app.

---

## Rust Concepts Checklist

Tick these off as you meet them in the phases:

| Concept | Where you use it |
|---|---|
| Structs & `impl` | Phase 2 |
| Traits (`eframe::App`) | Phase 2 |
| Closures | Phases 2–4 (every `show(|ui| ...)`) |
| `&` vs `&mut` borrows | Phase 3 (binding widgets to fields) |
| `String` vs `&str` | Phase 3 |
| Enums + `match` | Phase 7 (chart type) |
| Modules & visibility | Phase 5 |
| `Option` / `Result` / `?` | Phases 6, 9 |
| Iterators (`map`, `collect`) | Phase 7 |
| Derive macros (`serde`) | Phases 6, 10 |
| Threads, `move`, channels | Phase 8 |
| Error crates (`anyhow`, `thiserror`) | Phase 9 |
| Unit tests | Phase 11 |
| Cargo profiles & features | Phases 10, 13 |

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `link.exe not found` | C++ Build Tools missing — rerun the VS Build Tools installer. |
| Many errors after adding `egui_plot` / `egui_extras` | Version mismatch with `egui`. Align versions so `cargo tree` shows only one `egui`. |
| "cannot borrow `self` as mutable more than once" | You're holding a reference to a field while calling a `&mut self` method. Copy/clone the value first, or restructure into smaller methods that take only the fields they need. |
| Panel appears in wrong place / overlaps | Side/top/bottom panels must be added **before** `CentralPanel`. |
| UI doesn't update after background work finishes | Call `ctx.request_repaint()` (from the thread, via a cloned `Context`). |
| Window freezes | Long work on the UI thread → move it to a thread (Phase 8). |
| Settings not saved | Persistence feature not enabled, or `save` not implemented, or field is `#[serde(skip)]`. |
| API in an example doesn't exist | egui changes between versions — read docs.rs **for your exact version** (shown in `Cargo.lock`). |
| First build is very slow | Normal. Subsequent builds are incremental. |

---

## Resources

- **The Rust Book** — <https://doc.rust-lang.org/book/> (chapters 1–10, 13, 16 are most relevant)
- **Rust by Example** — <https://doc.rust-lang.org/rust-by-example/>
- **Rustlings** (exercises) — <https://github.com/rust-lang/rustlings>
- **egui repo & examples** — <https://github.com/emilk/egui>
- **egui live demo** (see every widget, with source) — <https://www.egui.rs>
- **eframe template** — <https://github.com/emilk/eframe_template>
- **docs.rs** — <https://docs.rs> (egui, eframe, egui_plot, rfd, csv, serde, anyhow)

---

## Alternative Frameworks (if egui isn't a fit)

- **Tauri** — web frontend (HTML/JS/React) + Rust backend. Best if you know web dev or want a "native-looking" modern UI.
- **Iced** — Elm architecture (State → View → Message → Update). Great for learning clean architecture.
- **Slint** — declarative `.slint` markup with a visual designer. Check licensing for commercial use.
- **Dioxus** — React-like components in pure Rust; also targets web/mobile.

The phases above (organization, data loading, background work, errors, persistence, testing, release) apply to all of them — only Phases 2–4 change.
