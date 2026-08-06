---
name: report-enhancer
description: Converts a PDF report to LaTeX, enhances structure/formatting/quality, and compiles back to professional PDF. Use when the user says "improve my report", "enhance this PDF", "convert PDF to LaTeX", "make my report better", or provides a PDF to improve.
---

# Report Enhancer

Converts any PDF report into a professional LaTeX document with improved structure, formatting, and quality.

## When to Use

- User provides a PDF report to improve
- User says "improve my report", "enhance this PDF", "make it better"
- User wants to convert a PDF to LaTeX
- User wants professional formatting applied to an existing document

## Workflow

### Step 1: Extract Content from PDF

Extract text:
```bash
pdftotext -layout input.pdf /tmp/report-extracted.txt
```

Extract images:
```bash
mkdir -p /tmp/report-images
pdfimages -all input.pdf /tmp/report-images/img
```

This extracts all images as PNG/JPEG files. Name them sequentially:
```bash
ls /tmp/report-images/
```

If tools are not available, install them:
```bash
sudo pacman -S poppler  # Arch Linux (includes pdftotext + pdfimages)
# or
sudo apt install poppler-utils  # Debian/Ubuntu
```

Read the extracted text and understand:
- Document structure (chapters, sections, subsections)
- Content (text, lists, tables, figures references)
- Language (French, English, Arabic, etc.)
- Type (internship report, thesis, technical report, etc.)
- Where images were in the original (check figure references in text)

### Step 2: Analyze and Plan Improvements

Read the extracted text and identify:
- **Figure references**: search for "Figure", "Fig.", "Figure X.Y", or image descriptions in the text. Note which section each figure belongs to.
- **Table references**: search for "Tableau", "Table", "Table X.Y". Note which section each table belongs to.
- **Structure**: chapters, sections, subsections
- **Language**: French, English, Arabic, etc.
- **Type**: internship report, thesis, technical report, etc.

Create a mapping of where each image should go:
```
Figure 1.1 → chapter1.tex, section 1.2, after paragraph about X
Figure 2.1 → chapter2.tex, section 2.1, right after the subsection heading
Table 3.1 → chapter3.tex, section 3.2, after the methodology text
```

Identify what needs improvement:
- **Structure**: Missing abstract, table of contents, conclusion, bibliography
- **Formatting**: Consistent fonts, margins, spacing, headers/footers
- **Tables**: Proper formatting with `tabularx`, consistent styles
- **Figures**: Proper placement with `[H]`, descriptive captions, correct labels
- **References**: Bibliography management with `biblatex` or `thebibliography`
- **Language**: Grammar, spelling, academic tone
- **Content flow**: Logical progression, clear transitions

### Step 3: Create LaTeX Document

Create the project structure (NEVER put everything in main.tex — max 30 lines in main.tex):

```
report-output/
├── main.tex                  # Only \input and \include — NO content
├── styles/
│   ├── packages.tex          # All \usepackage declarations
│   └── formatting.tex        # Page layout, headers, code styles
├── frontmatter/
│   ├── titlepage.tex         # Title page
│   ├── abstract.tex          # Résumé + Abstract
│   ├── acknowledgements.tex  # Remerciements
│   └── abbreviations.tex     # List of abbreviations (if any)
├── chapters/
│   ├── chapter1.tex          # One file per chapter
│   ├── chapter2.tex
│   └── chapterN.tex
├── backmatter/
│   ├── annexeA.tex           # One file per annex
│   └── annexeB.tex
└── images/
    ├── img-000.png           # Extracted images
    └── img-001.jpg
```

**Rule: Each chapter gets its own file. No exceptions.**

Copy extracted images into the project:
```bash
cp /tmp/report-images/* report-output/images/
```

#### Base Template (main.tex):

```latex
\documentclass[12pt,a4paper,openany]{report}

% --- Packages ---
\usepackage[utf8]{inputenc}
\usepackage[T1]{fontenc}
\usepackage[french,english]{babel}
\usepackage{geometry}
\geometry{margin=2.5cm}
\usepackage{setspace}
\onehalfspacing
\usepackage{graphicx}
\usepackage{float}
\usepackage{tabularx}
\usepackage{booktabs}
\usepackage{listings}
\usepackage{xcolor}
\usepackage{hyperref}
\hypersetup{colorlinks=true, linkcolor=blue, urlcolor=blue}
\usepackage{fancyhdr}
\pagestyle{fancy}
\fancyhf{}
\fancyhead[L]{\leftmark}
\fancyhead[R]{\thepage}
\renewcommand{\headrulewidth}{0.4pt}

% --- Document ---
\begin{document}

% Front matter
\input{frontmatter/titlepage}
\input{frontmatter/abstract}

% Main content
\include{chapters/chapter1}
% ... more chapters

% Back matter
\appendix
% annexes if needed

\end{document}
```

#### Title Page Template:

```latex
\begin{titlepage}
    \centering
    \vspace*{2cm}
    % Add institution logos if provided
    {\Large\textbf{INSTITUTION NAME}}\\
    \vspace{1cm}
    {\Huge\bfseries REPORT TITLE}\\
    \vspace{2cm}
    {\large Author Name}\\
    \vspace{3cm}
    {\large Date}
\end{titlepage}
```

#### Chapter Template:

```latex
\chapter{Chapter Title}
\label{chap:label}

\section{Section Title}

Content here...

\subsection{Subsection Title}

More content...

\begin{figure}[H]
    \centering
    % \includegraphics[width=0.8\textwidth]{images/figure.png}
    \caption{Figure caption}
    \label{fig:label}
\end{figure}

\begin{table}[H]
    \centering
    \caption{Table caption}
    \label{tab:label}
    \begin{tabularx}{\textwidth}{|l|X|}
        \hline
        \textbf{Header1} & \textbf{Header2} \\
        \hline
        Content & Content \\
        \hline
    \end{tabularx}
\end{table}
```

### Step 4: Apply Enhancements

#### Formatting Rules:
- Use `openany` in documentclass to avoid blank pages between chapters
- Use `float` package with `[H]` for figure/table placement
- Use `tabularx` for tables that need to fill width
- Use `booktabs` for professional table rules (`\toprule`, `\midrule`, `\bottomrule`)
- Use `listings` for code blocks with syntax highlighting
- Use `hyperref` for clickable links and PDF bookmarks
- Use `\label{}` and `\ref{}` for cross-references (never hardcode page numbers)
- Use `\noindent` before paragraphs that follow headings if needed
- Consistent spacing: `\vspace{}` for vertical spacing

#### Figure Placement Rules:

- **Always use `[H]`** from the `float` package to force exact placement
- Place figures **immediately after** the first reference in the text
- If the text says "comme montré en Figure 3.1", the figure must appear before or right after that sentence
- Use descriptive captions: "Figure 3.1 — Architecture du laboratoire" not "Figure 3.1"
- Labels follow pattern: `\label{fig:descriptive-name}` (e.g., `fig:topo-gns3`, `fig:ha-status`)
- Reference with `\ref{fig:descriptive-name}` in text

```latex
\begin{figure}[H]
    \centering
    \includegraphics[width=0.8\textwidth]{images/topology.png}
    \caption{Topologie du laboratoire --- Vue d'ensemble}
    \label{fig:topology}
\end{figure}
```

#### Code Listings Style:

```latex
\lstset{
    basicstyle=\ttfamily\small,
    backgroundcolor=\color{gray!10},
    frame=single,
    breaklines=true,
    captionpos=b,
    numbers=left,
    numberstyle=\tiny\color{gray}
}
```

#### Academic Writing Rules:
- Use formal register (no contractions, no colloquialisms)
- Use passive voice where appropriate
- Cite sources with `\cite{}` or `\footnote{}`
- Number figures, tables, and equations
- Use consistent terminology throughout
- French: use `\og` and `\fg{}` for guillemets (« »)
- French: use `---` for em dashes
- Avoid orphaned section headers (check with `openany`)

### Step 5: Compile and Verify

```bash
cd report-output/
xelatex -interaction=nonstopmode main.tex
xelatex -interaction=nonstopmode main.tex  # Second pass for references
```

Check for errors:
```bash
python3 -c "
with open('main.log') as f:
    log = f.read()
errors = [l for l in log.splitlines() if l.startswith('!')]
overfull = [l for l in log.splitlines() if 'Overfull' in l]
undefined = [l for l in log.splitlines() if 'undefined' in l.lower()]
print(f'Errors: {len(errors)}')
print(f'Overfull: {len(overfull)}')
print(f'Undefined refs: {len(undefined)}')
for e in errors: print(e)
"
```

### Step 6: Output

- Save the final PDF in the user's working directory
- Report what was improved (structure, formatting, content fixes)
- List any issues found and fixed
- Provide the LaTeX source for future edits

## Quality Checklist

- [ ] `main.tex` is under 30 lines (only `\input` and `\include`)
- [ ] Each chapter is a separate `.tex` file
- [ ] Each annex is a separate `.tex` file
- [ ] Styles separated into `packages.tex` and `formatting.tex`
- [ ] Frontmatter in separate files (title, abstract, acknowledgements)
- [ ] Title page with all required info
- [ ] Abstract/Résumé (FR + EN if bilingual)
- [ ] Table of contents
- [ ] Consistent chapter/section numbering
- [ ] All figures placed where originally referenced (not random)
- [ ] All figures have `[H]` placement
- [ ] All figures have descriptive captions and labels
- [ ] All tables have captions and labels
- [ ] Cross-references work (`\ref{}` shows correct numbers)
- [ ] No orphaned headers
- [ ] No overfull hbox warnings (or minimal)
- [ ] Bibliography formatted consistently
- [ ] Code blocks properly formatted
- [ ] No placeholder text remaining
- [ ] Professional formatting throughout
- [ ] PDF compiles without errors
- [ ] Images extracted from original PDF are all included

## Tips

- `pdfimages` extracts all images from the PDF — use `-all` flag to keep original formats (PNG/JPEG)
- If the PDF has scanned images of text, use `ocrmypdf` to make it searchable first
- For complex tables, consider converting to `tabularx` or `longtable`
- Always do a second xelatex pass for cross-references
- Use `pdffonts input.pdf` to check what fonts the original uses
- Use `pdfinfo input.pdf` to check page count and metadata
- Preserve the original content — only improve formatting and structure
- Match image placement to the original where possible (check figure references in extracted text)
