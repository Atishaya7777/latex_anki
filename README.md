# LaTeX → Anki

Generate and verify Anki flashcards directly from theorem-like environments in a LaTeX document.

The script extracts definitions, theorems, propositions, and methods, generates question/answer pairs, asks Codex to verify their correctness and wording, and syncs approved cards to Anki through AnkiConnect.

Your LaTeX document remains the source of truth. No special Anki markers are required.

## Example

Given:

```latex
\begin{definition}[Lipschitz condition]
    A function $f(t,y)$ satisfies a Lipschitz condition in $y$ on
    $D \subset \mathbb{R}^2$ if there exists $L > 0$ such that
    \[
        |f(t,y_1)-f(t,y_2)| \leq L|y_1-y_2|
    \]
    for all $(t,y_1),(t,y_2) \in D$.
\end{definition}
```

the script can generate a card such as:

**Front**

> What does it mean for $f(t,y)$ to satisfy a Lipschitz condition in $y$?

**Back**

> There exists a constant $L>0$ such that
>
> $$
> |f(t,y_1)-f(t,y_2)|\leq L|y_1-y_2|
> $$
>
> for all $(t,y_1),(t,y_2)\in D$.

The mathematics is rendered by Anki's MathJax support.

## How it works

```text
LaTeX document
      │
      ▼
Environment parser
      │
      ▼
Candidate flashcards
      │
      ▼
Codex verification
      │
      ├── warning ──► manual review
      │
      ▼
Approved cards
      │
      ▼
AnkiConnect
      │
      ▼
     Anki
```

The parser currently recognizes:

* `definition`
* `theorem`
* `proposition`
* `method`

The environment type and optional title are used to infer an appropriate question. You do not need to annotate your LaTeX source with flashcard-specific markers.

Codex then reviews the generated cards for grammar, clarity, faithfulness to the source, and possible mathematical errors.

## Requirements

* Python 3.10+
* Anki Desktop
* AnkiConnect
* Codex CLI

No third-party Python packages are required.

### Codex

Install the Codex CLI and authenticate it before running the script.

Verify that it works with:

```bash
codex --version
```

The script invokes `codex exec` non-interactively and requests structured output.

### AnkiConnect

Install the AnkiConnect add-on in Anki Desktop and keep Anki running while syncing cards.

By default, the script connects to:

```text
http://127.0.0.1:8765
```

For security, non-loopback AnkiConnect URLs are rejected unless explicitly enabled.

## Usage

Generate, verify, and sync new cards:

```bash
python latex_anki.py notes.tex
```

The document's LaTeX title is used as the default Anki deck name:

```latex
\title{Ordinary Differential Equations}
```

becomes:

```text
Ordinary Differential Equations
```

You can also verify cards without syncing them to Anki:

```bash
python latex_anki.py notes.tex --no-sync
```

Run the built-in tests with:

```bash
python latex_anki.py --self-test
```

For all available options:

```bash
python latex_anki.py --help
```

## Incremental processing

The script is designed for notes that grow over time.

It keeps local state describing which LaTeX environments have already been processed. On later runs, unchanged environments are skipped.

A typical workflow is therefore:

```bash
# Study and append to notes.tex

python latex_anki.py notes.tex
```

Only new or changed material needs Codex verification.

Generated state is stored locally and should not be committed to Git.

## Mathematical verification

Codex is instructed not to silently repair suspected mathematical mistakes.

For example:

```latex
\begin{definition}[ODE of order $n$]
    An ordinary differential equation of order 1 is ...
\end{definition}
```

may be held with a warning because the title and definition appear inconsistent.

A held card is **not automatically added to Anki**.

List held material with:

```bash
python latex_anki.py notes.tex --list-held
```

A held source will have an identifier such as:

```text
91c2f13ca520
```

After inspecting it, approve it with:

```bash
python latex_anki.py notes.tex --accept-held 91c2f13ca520
```

or reject it with:

```bash
python latex_anki.py notes.tex --reject-held 91c2f13ca520
```

If a rejected LaTeX environment is later changed, its new contents can be reviewed again.

## Card generation

Definitions generally produce questions of the form:

```text
What is ...?
```

Methods generally produce questions such as:

```text
How do you ...?
```

Theorems and propositions are converted into appropriate statement/recall questions.

Codex may split a large environment into multiple cards when it contains several independently useful facts.

For example, a method describing second-order constant-coefficient ODEs may produce separate cards for:

* the characteristic equation;
* distinct real roots;
* repeated roots;
* complex-conjugate roots.

The goal is to prefer focused recall over unnecessarily large cards.

## MathJax

LaTeX mathematics is preserved for Anki's MathJax renderer.

Both inline and display mathematics are supported:

```latex
$f(t,y)$
```

and

```latex
\[
    |f(t,y_1)-f(t,y_2)| \leq L|y_1-y_2|
\]
```

The script safely escapes HTML-sensitive characters inside MathJax expressions before sending fields to Anki.

## Existing cards

Cards created by the script receive stable identifying tags.

This allows subsequent runs to recognize existing cards instead of creating duplicates.

The script can also repair certain formatting problems in cards generated by older versions.

It does **not** automatically delete Anki cards when a corresponding environment disappears from the LaTeX document.

## Generated files

Processing `notes.tex` creates local state/review data similar to:

```text
.notes.latex-anki-state.json
.notes.latex-anki-review/
```

These may contain generated cards or excerpts from your notes.

They should not be committed to a public repository.

A recommended `.gitignore` is included with the project.

## Security

The script does not contain or manage Codex credentials. Authentication is handled by the Codex CLI.

Codex receives the contents of environments that require verification. Do not use LLM verification on material you are not permitted to send to the configured model/service.

AnkiConnect communication is restricted to loopback addresses by default.

The script never modifies the input `.tex` document.

## Philosophy

The LaTeX document is the source of truth.

The deterministic parser decides **what** material is eligible to become a flashcard. Codex is only responsible for improving and verifying the resulting cards.

This keeps the workflow predictable:

```text
Parser       → What should become a card?
Generator    → Produce an initial question and answer.
Codex        → Is the card clear, concise, and faithful?
Anki         → Store and schedule the approved card.
```

## License

Copyright (c) 2026 Atishaya Maharjan

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.