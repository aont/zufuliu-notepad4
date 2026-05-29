The following is based on a **static analysis**. I have not taken actual performance profiles, but based on the README, Visual Studio project files, menu definitions, and major source files, the largest opportunities for reduction are fairly clear.

## Conclusion: Reduction Candidates by Priority

| Priority | Candidate for Removal/Disabling                                                      | Reason                                                                                                                                            | Expected Impact  |
| -------: | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- |
|        1 | **Multiple language lexers / syntax highlighting / API lists**                       | Dozens of `Lex*.cxx` and `stl*.cpp` files are built. They contribute significantly to binary size, initialization data, and style/API management. | **High**         |
|        2 | **Auto-completion**                                                                  | `EditAutoC.cpp` contains custom word collection, AA trees, and buffer management. It increases input-time processing and temporary memory usage.  | **High**         |
|        3 | **Direct2D / DirectWrite / D3D rendering**                                           | Additional rendering backends, font features, bidirectional text, ligatures, and color emoji support.                                             | **Medium–High**  |
|        4 | **matepath file browser integration**                                                | Included as a separate project in the solution. Unnecessary for a standalone Notepad.                                                             | **Medium**       |
|        5 | **Code folding / fold margin**                                                       | Requires fold state tracking, margin rendering, and lexer integration on the Scintilla side.                                                      | **Medium**       |
|        6 | **Advanced search/replace, Boost regex**                                             | README explicitly states Boost regex usage. Regular expression processing is relatively heavy.                                                    | **Medium**       |
|        7 | **Web Tools: CSS/JS/JSON compress/pretty, JS evaluate, URL encode/decode**           | Independent menu features. Not needed in a simple text editor.                                                                                    | **Medium**       |
|        8 | **Transliteration, Unicode/CJK helpers, ICU/ELS/Hanja/LaTeX input**                  | Involves delayed DLL loading, dictionaries, and conversion logic.                                                                                 | **Medium**       |
|        9 | **Hi-DPI image resources, toolbars, multiple icon sizes**                            | Enabled in `config.h`. Resource size can be reduced.                                                                                              | **Small–Medium** |
|       10 | **Favorites / recent history / autosave / change notification / system integration** | Additional UI and state management. Monitoring features also consume processing resources.                                                        | **Small–Medium** |

---

## Feature Inventory

### 1. Core Editing Features

Notepad4 is a lightweight text editor based on Scintilla. It provides syntax highlighting, code folding, auto-completion, and API lists for many languages.

Primary basic features include:

* New, Open, Save, Save As
* Backup Save, Save Copy, Preserve Original Timestamp
* Read-only files and read-only mode
* Reload with encoding selection
* ANSI / UTF-8 / UTF-8 BOM / UTF-16LE / UTF-16BE
* CRLF / LF / CR line endings
* Printing and page setup
* File properties and open containing folder
* Recent files and Favorites
* Browse/folder open through matepath

Menu definitions confirm the existence of Save, Reload, Encoding, Line Endings, Browse, Favorites, and Recent entries under the File menu.

**Reduction considerations:**

For a minimal Notepad-like editor, the following are candidates:

* Favorites
* Recent History UI
* Browse/matepath
* Create Desktop Link
* Save Backup
* Save Copy
* Preserve Original Timestamp

Keeping only core save functionality simplifies the application considerably.

---

### 2. Multi-language Syntax Highlighting, Lexers, and API Lists

The README lists support for numerous languages and formats including Plain Text, C/C++, CSS, HTML, JavaScript, Python, PHP, Markdown, SQL, XML, YAML, Rust, Go, Java, Kotlin, Swift, Zig, and many others.

The Visual Studio project also compiles a large collection of `scintilla/lexers/Lex*.cxx` files. Examples include APDL, Asm, AutoHotkey, Bash, CMake, CPP, CSS, CSV, Dart, Diff, HTML, Java, JavaScript, JSON, Markdown, Python, SQL, YAML, Zig, and more.

Notepad4 additionally contains extensive style definitions in `src/EditLexers/stl*.cpp`.

**Why this is heavy:**

* Large number of compilation units.
* Each lexer contains keywords, state machines, folding logic, and styling information.
* Integrated with API lists and auto-completion.
* Increases binary size and initialization data.
* Lexers consume CPU whenever documents are analyzed.

**Reduction strategy:**

For a minimal configuration:

* Plain Text
* Optional: INI/Config
* Optional: Markdown
* Optional: JSON
* Optional: C/C++ or Python if required

Primary removal targets:

* `scintilla/lexers/Lex*.cxx`
* `src/EditLexers/stl*.cpp`
* Scheme menu entries
* Extension mappings in `doc/FileExt.txt`
* Related API and keyword lists

**This is the highest-priority reduction target.**

---

### 3. Code Folding

The README explicitly mentions fold-level controls, current-block folding, and fold shortcuts.

The View menu includes:

* Show Code Folding
* Toggle Folds
* Fold levels 1–10

**Why it is heavy:**

* Lexer must calculate fold levels.
* Scintilla maintains fold state per line.
* Fold margin rendering is required.
* State management scales with line count.

**Reduction strategy:**

A simple text editor does not require folding. If syntax highlighting is removed, folding should typically be removed as well.

---

### 4. Auto-completion

The README lists:

* Enhanced word and function auto-completion
* Context-based auto-completion
* Brace/bracket/quote auto-completion

Implementation details show `EditAutoC.cpp` managing:

* Custom word lists
* AA trees
* Sort-key caches
* Word extraction
* Heap-allocated completion structures

Menu items include:

* Complete Word
* Auto Completion Settings
* Ignore Case
* LaTeX Input Method

**Why it is heavy:**

* Candidate generation during typing.
* Scanning document and API lists.
* Temporary memory allocation.
* Language-specific logic and style checks.
* Dependency on Scintilla's `AutoComplete.cxx`.

**Reduction strategy:**

Strong candidate for complete removal.

If retained:

* Keep document-word completion only.
* Remove API list completion.
* Remove context-sensitive completion.
* Remove LaTeX and emoji input.
* Replace automatic brace insertion with simpler logic.

---

### 5. CallTip and Color Preview

The README references color preview through CallTip and color-dialog integration.

View menu options include:

* Show CallTip
* RGBA/ARGB/BGRA/ABGR formats

Scintilla's `CallTip.cxx` is also compiled.

**Why it is heavy:**

* Color literal detection near the caret.
* CallTip UI management.
* Color parsing and dialog integration.

**Reduction strategy:**

Remove alongside auto-completion for a cleaner architecture.

---

### 6. Direct2D, DirectWrite, D3D, and Advanced Rendering

The README lists:

* GDI and Direct2D/DirectWrite switching
* Font ligatures
* Color fonts and emoji
* RTL layout
* Bidirectional text support
* Fractional font sizes

Menus expose:

* Legacy GDI
* Direct2D
* Direct2D Retain
* Direct2D GDI DC
* Direct3D
* Font quality options
* RTL and bidi options

Build targets include:

* `SurfaceD2D.cxx`
* `SurfaceGDI.cxx`
* `PlatWin.cxx`
* `ScintillaWin.cxx`

**Why it is heavy:**

* Additional rendering paths.
* Font fallback and shaping logic.
* Ligatures, emoji rendering, bidi layout.
* More dependencies and resource management.

**Reduction strategy:**

For maximum simplicity:

* Use GDI only.
* Remove Direct2D/DirectWrite/Direct3D options.
* Remove ligatures, color fonts, bidi, RTL, and fractional sizing.
* Keep basic IME support if needed.

---

### 7. Large File Mode

The File menu includes an x64-only Large File Mode.

Implementation creates alternative Scintilla documents using:

* `SC_DOCUMENTOPTION_TEXT_LARGE`
* `SC_DOCUMENTOPTION_STYLES_NONE`

Conversion routines recreate the document and reload its contents.

**Why it matters:**

* Adds complexity for large-file support.
* Requires temporary buffers during conversion.
* More about complexity than idle memory usage.

**Reduction decision:**

* Remove if targeting the smallest possible editor.
* Keep if large-log viewing remains a use case.

If forced to choose, removing syntax highlighting while retaining large-file support is often the better trade-off.

---

### 8. Advanced Search/Replace and Boost Regex

The README explicitly mentions Boost Regex support.

Build definitions include:

* `BOOST_REGEX_STANDALONE`
* `NO_CXX11_REGEX`

**Why it is heavy:**

* Larger binary footprint.
* Potentially expensive processing on large documents.
* Additional UI and history management.

**Reduction strategy:**

* Keep plain text search.
* Remove regex search and replace.
* Remove search history persistence.
* Remove advanced options.

---

### 9. Editing Utilities

The Edit menu contains numerous transformation commands, including:

* Duplicate selection
* Comment/uncomment
* Indent/unindent
* Strip trailing spaces
* Remove blank lines
* Merge duplicate lines
* Sort lines
* Modify lines
* Align lines
* Join/split lines
* Column wrap
* Case conversion
* Tabify/untabify
* Numeric conversions
* Selection enclosure
* HTML/XML helpers
* Unicode helpers
* Escape/unescape tools

**Heavier features include:**

* Sort Lines
* Modify Lines
* Align Lines
* Duplicate-line processing
* Column Wrap
* Escape/Unescape
* Numeric conversion
* Character inspection

**Reduction strategy:**

A minimal editor only needs:

* Undo / Redo
* Cut / Copy / Paste
* Delete
* Select All
* Find

Most transformation features can be removed.

---

### 10. Display and UI Helpers

The View menu includes:

* Word Wrap
* Long Line Marker
* Indentation Guides
* Whitespace/EOL/Wrap Symbols
* Unicode Control Character Display
* Brace Matching
* Current Block Highlight
* Current Line Highlight
* Mark Occurrences
* Line Numbers
* Bookmark Margin
* Change History Margin
* Code Folding
* Zoom
* Full Screen

**Potentially expensive items:**

* Mark Occurrences
* Brace Matching
* Current Block Highlight
* Change History Markers
* Margins and indicators
* Whitespace/EOL visualization

**Reduction strategy:**

Keep only:

* Word Wrap
* Zoom
* Optional line numbers

Everything else is removable.

---

### 11. Bookmarks and Mark Occurrences

The README highlights both bookmarks and occurrence marking.

Search menu items include bookmark navigation and management.

**Why they are heavy:**

* Marker and indicator storage.
* Potential rescanning on caret movement.
* Larger files amplify the cost.

**Reduction strategy:**

Remove both, especially occurrence marking.

---

### 12. Web Tools, Base64, URL Utilities, and JS Evaluation

The Tools menu contains:

* Base64 encode/decode
* URL-safe Base64
* Embedded image encoding
* Hex decoding
* CSS/JS/JSON compression
* CSS/JS/JSON formatting
* JavaScript expression evaluation
* HTML/XML escaping
* URL encoding/decoding
* Online search

**Reduction strategy:**

A minimal editor does not need these features. The entire Tools menu is a strong candidate for removal.

---

### 13. Transliteration, CJK, and Unicode Helpers

The README references:

* CJK improvements
* ANSI art support
* Transliteration

Tools include:

* Half-width/full-width conversion
* Chinese simplification/traditional conversion
* Japanese Kana conversion
* Korean Hanja/Hangul conversion
* Various script-to-Latin conversions

`Edit.cpp` dynamically loads:

* ELS libraries
* ICU libraries
* Property-system libraries

Scintilla also builds:

* `HanjaDic.cxx`
* `LaTeXInput.cxx`

**Reduction strategy:**

Remove:

* Hanja support
* Transliteration
* Unicode character information
* LaTeX input
* Emoji input

Retain basic IME support only.

---

### 14. Hi-DPI Resources, Toolbars, and Appearance

`config.h` enables Hi-DPI image resources.

Resources include:

* 16px
* 24px
* 32px
* 40px
* 48px

toolbar variants and related assets.

Appearance settings include:

* Toolbar customization
* Toolbar scaling
* Large toolbar
* Status bar
* Transparency
* Full-screen options

**Reduction strategy:**

* Set `NP2_ENABLE_HIDPI_IMAGE_RESOURCE 0`
* Remove toolbar customization
* Remove transparency
* Use only a single icon size
* Optionally remove the toolbar entirely

---

### 15. Localization

`config.h` enables satellite resource DLL localization and lexer-style localization.

The README lists support for multiple languages.

**Reduction strategy:**

* Fixed English or Japanese UI
* Remove satellite DLL support
* Remove lexer-style localization
* Exclude locale resources

---

### 16. System Integration and External Execution

The README references Windows integration.

Menus include:

* System Integration
* Run as Administrator
* Restart
* Execute Document
* Open With
* Run Command
* Online Search
* Custom Actions

**Why remove them:**

* Increased UI complexity.
* Additional process-launching paths.
* Larger attack surface.
* Limited value in a minimal editor.

**Reduction strategy:**

Remove:

* Run Command
* Execute Document
* Open With
* Run as Administrator
* Online Search
* Custom Actions
* System Integration

---

## Recommended Minimal Feature Sets

### A. Minimal Notepad Configuration

Keep:

* New / Open / Save / Save As
* Cut / Copy / Paste / Delete / Select All
* Undo / Redo
* Find / Find Next
* Optional basic Replace
* Word Wrap
* Font Selection
* UTF-8 (optionally ANSI and UTF-16)
* CRLF/LF line endings
* Optional drag-and-drop opening

Remove:

* All lexers and syntax highlighting
* Code folding
* Auto-completion
* CallTips
* API lists
* Bookmarks
* Mark Occurrences
* Direct2D/D3D
* Transliteration
* Web Tools
* Base64 utilities
* Advanced editing tools
* matepath
* Localization
* Hi-DPI multi-resource assets
* System integration

### B. Lightweight Code Editor Configuration

Keep:

* Plain Text
* JSON
* Markdown
* INI/Config
* One or two programming languages
* Syntax highlighting
* Line numbers
* Find/Replace
* Word Wrap
* GDI rendering
* UTF-8-focused support

Remove:

* Unused lexers
* Folding
* API lists
* Context-sensitive auto-completion
* CallTips
* Direct2D/D3D
* Web Tools
* Transliteration
* matepath
* Localization

---

## Suggested Implementation Order

### Step 1: Remove matepath

The solution contains two projects:

* Notepad4
* matepath

Remove matepath first.

---

### Step 2: Reduce Lexers

Trim `Lex*.cxx` and `stl*.cpp` entries in `Notepad4.vcxproj`.

A safe initial set:

* `LexNull.cxx`
* `stlDefault.cpp`
* Optional: `LexJSON.cxx` + `stlJSON.cpp`
* Optional: `LexMarkdown.cxx` + `stlMarkdown.cpp`
* Optional: `LexCPP.cxx` + `stlCPP.cpp`

---

### Step 3: Disable AutoComplete and CallTip

Remove:

* `EditAutoC.cpp`
* `AutoComplete.cxx`
* `CallTip.cxx`

Then remove menu items, settings, and command handlers.

---

### Step 4: Drop Direct2D/DirectWrite/D3D

Keep only GDI rendering.

Remove rendering-selection menus and configuration paths.

---

### Step 5: Remove the Tools Menu

This eliminates:

* External execution
* Base64 utilities
* Transliteration
* Web tools

and significantly simplifies the application.

---

## Reduction Candidate Rankings

### Largest Memory Footprint Contributors

1. Document data and undo history
2. Syntax highlighting styles, indicators, decorations
3. Large-file mode conversion buffers
4. Auto-completion candidate structures
5. Lexers, keywords, and API lists
6. Direct2D/DirectWrite resources
7. Hi-DPI image resources
8. Localization DLLs

### Highest CPU Consumers

1. Syntax highlighting and lexing
2. Code folding
3. Regex search/replace
4. Mark Occurrences
5. Auto-completion generation
6. Sorting, duplicate removal, alignment tools
7. CSS/JS/JSON formatting/compression
8. Transliteration and Unicode inspection
9. DirectWrite shaping, bidi, and ligatures

---

## Recommended Direction

The highest-return approach is to **keep Scintilla but remove the majority of Notepad4's code-editor-specific functionality**.

Specifically:

1. Move toward **Plain Text + UTF-8 + GDI + basic editing**.
2. Keep only the minimum necessary lexers.
3. Remove `EditAutoC.cpp`, CallTips, API lists, folding, and occurrence marking.
4. Remove the Tools menu, transliteration, Web Tools, Base64 utilities, and external execution features.
5. Remove matepath and localization.
6. Treat Hi-DPI resource reduction as a later optimization.

Following this sequence tends to produce the clearest dependency cuts while delivering meaningful reductions in memory usage, binary size, and runtime complexity.
