# Formatting Log — Rapport de Stage

## Session: 2026-08-05 (Loop 2 — Honesty Pass)

### Goal
Make the report honest about what was actually implemented vs claimed. Fix credibility issues.

### Changes Made

#### 1. Honest validation counts (ch1, ch7, ch9, abstract)
- **Before**: 8 Réalisé + 5 Partiel + 3 Non réalisable = 16
- **After**: 7 Réalisé + 5 Partiel + 4 Non réalisé = 16
- Moved objective 14 (SSL Inspection) from "Réalisé" → "Partiel" (cert generated but not applied)
- Moved objective 5 (BGP) from "Partiel" → "Non réalisé" (Docker Router replaced it entirely)

#### 2. Chapter 1 — Restructured tables
- Three clear tables: Réalisé (7), Partiel (5), Non réalisable (4)
- Added "Explication" column to Partiel table with honest reasons
- Added "Raison" column to Non réalisable table
- Updated note with accurate summary

#### 3. Chapter 5 — Honest security profiles section
- Replaced fake test commands with honest explanation of limitations
- Added enumerate listing 3 constraints (signatures factory, 3 policies, SSL not applied)
- Added "Leçon apprise" about importance of FortiGuard license
- Each subsection now explains what's configured vs what's testable

#### 4. Chapter 7 — Honest security tests
- Added "Limites des tests de sécurité" subsection explaining constraints
- Changed test descriptions from "Observé: page de blocage" to honest "depends on signatures factory"
- Updated legend with 4 statuses: Réalisé, Partiel, Non réalisé, Non réalisable

#### 5. Chapter 9 — Updated conclusion
- Fixed counts: 7+5+4
- Updated table with honest descriptions (e.g., "Partiel (certificat généré, non appliqué)")

#### 6. Abstracts — Updated counts
- FR: "7 objectifs sur 16 ont été entièrement atteints, 5 partiellement (profils configurés mais non testables), et 4 n'ont pas été réalisables"
- EN: same

#### 7. Fixed UTF-8 encoding error
- chapter7.tex line 147: `bloqué` → `bloque` inside lstlisting (UTF-8 not supported in lstlisting)

### Final Status
- **0 errors** on compile
- **0 overfull hbox warnings**
- All counts consistent: 7+5+4=16 across ch1, ch7, ch9, abstracts
- Report is now honest about limitations

---

## Session: 2026-08-05 (Loop 1 — Audit & Fixes)

### Goal
Comprehensive audit of all .tex files: remove OCI/VPN references, fix inconsistencies, recompile until clean.

### Changes Made

#### 1. Removed VPN/OCI references (HIGH)
- **chapter5.tex**: Replaced IPSec VPN section with "Contraintes et limitations" explaining why VPN wasn't implemented (eval license + cloud firewall)
- **chapter5.tex**: Removed VPN SSL section, replaced with note about eval license constraints
- **chapter5.tex**: Updated OCI Cloud section → "Endpoints de test — Services Docker" (Flask appServer endpoints)
- **chapter5.tex**: Escaped `IKE_SA_INIT` → `IKE\_SA\_INIT` (fixed LaTeX math mode error)
- **chapter9.tex**: Changed "Oracle Cloud" → "endpoint cloud" in difficulties section
- **chapter9.tex**: Changed "Routage OSPF" → "Routage inter-LAN" in objective 4
- **annexeA.tex**: Removed "OCI Cloud" from caption, topology description, and cabling
- **annexeA.tex**: Removed "webterm-1" from clients list and LAN1 cabling

#### 2. Fixed webterm-1 references (HIGH)
- **chapter4.tex**: Removed webterm-1 from Docker specs table and OVS-LAN1 cabling table
- **chapter7.tex**: Replaced all 4× "# Depuis webterm-1" → "# Depuis PC1"
- **chapter8.tex**: Changed `admin-host "192.168.10.50"` → `admin-host "192.168.10.0/24"`
- **annexeB.tex**: Removed webterm-1 row from address table
- **annexeA.tex**: Removed webterm-1 from clients and LAN1 cabling

#### 3. Fixed count inconsistencies (HIGH)
- **chapter1.tex**: Added objective 14 (SSL Deep Inspection) and 15 (Authentification locale) to "Réalisés" table
- **chapter1.tex**: Reorganized three tables to show 8 Réalisé + 5 Partiel + 3 Non réalisable = 16
- **chapter9.tex**: Fixed summary from "10+4+2" → "8+5+3"
- **abstract.tex** (FR): Fixed "10 objectifs sur 16... 1 n'a pas été réalisable" → "8... 3 n'ont pas été réalisables"
- **abstract.tex** (EN): Same fix in English
- **abstract.tex** (FR/EN): Removed "du routage OSPF" → "du routage inter-LAN via Docker Router"
- **abstract.tex** (FR/EN): Removed "des tunnels VPN IPSec et SSL" (not implemented)
- **chapter1.tex**: Updated "En attente" table → "Non réalisés" with correct statuses

#### 4. Fixed planning.tex (MEDIUM)
- Replaced "routage OSPF" → "routage inter-LAN via Docker Router"
- Removed "Mise en place des tunnels VPN IPSec et SSL"

#### 5. Fixed annexes inconsistencies (MEDIUM)
- **annexeB.tex**: Changed port3 IPs from `10.0.0.x/30` → `169.254.0.x/30` (HA heartbeat)
- **annexeB.tex**: Changed FGT-Secondary port1 from static `192.168.122.3/24` → DHCP
- **annexeB.tex**: Changed appServer-1 IP from `192.168.10.103` → `192.168.10.10`
- **annexeB.tex**: Replaced "Transit 10.0.0.0/30" → "HA Heartbeat 169.254.0.0/30"
- **chapter3.tex**: Changed "Dans notre topologie, OSPF permet" → "Dans une topologie de type site-à-site, OSPF permettrait" (conditional)

#### 6. Added Gantt chart (NEW)
- **packages.tex**: Added `\usepackage{pgfgantt}`
- **planning.tex**: Replaced table with pgfgantt Gantt chart, scaled to `\textwidth` with `\resizebox`
- Shows 5 weeks (July 2 – August 8, 2026) with task breakdown per week
- Includes milestone for soutenance

#### 7. Expanded remerciements (NEW)
- **acknowledgements.tex**: Expanded from 4 short paragraphs to full acknowledgements
- Added specific thanks to: Mr. Lachhab (detailed), Mme. Fatima Zahra, Mr. Youssef, EMSI department, classmates, family
- Added closing signature block

### Final Status
- **0 errors** on compile
- **0 overfull hbox warnings**
- Only cosmetic underfull warnings (badness 10000 = blank paragraphs, table cells)
- All cross-references valid
- Count consistency verified across ch1, ch7, ch9, abstracts
- No forbidden terms (OCI, webterm-1, etc.) remaining

---

## Session: 2026-08-03 (Original formatting)

### Goal
Fix mise en page issues: abstract positioning, abbreviations table, hyphenation, spacing, overfull/underfull warnings.

### Changes Made

#### 1. Packages (`styles/packages.tex`)
- Added `multicol` for 2-column abbreviations
- Added `ragged2e` for better line breaking
- Added `microtype` for improved typography (character protrusion, font expansion)

#### 2. Formatting (`styles/formatting.tex`)
- Added `\hyphenation{}` rules for FortiGate, security, configuration, etc.
- Added `\setlength{\parskip}{0pt}` and `\setlength{\parindent}{1.5em}`
- Tightened float spacing: `\textfloatsep`, `\floatsep`, `\intextsep` all set to 10pt
- Added widow/orphan control: `\widowpenalty=10000`, `\clubpenalty=10000`
- Added `\raggedbottom`
- Code listing: `frame=none` (removed frame for cleaner look)

#### 3. Abstract (`frontmatter/abstract.tex`)
- Added blank page between acknowledgements and abstract to push abstract to page 3

#### 4. Abbreviations (`frontmatter/abbreviations.tex`)
- Final: single 4-column `tabular*` with `\textwidth`, `\footnotesize`

#### 5. Chapter 2 (`chapters/chapter2.tex`)
- Shortened two paragraphs to remove overfull

#### 6. Chapter 3 (`chapters/chapter3.tex`)
- Added `scale=0.8, transform shape` to first TikZ figure
- Added `scale=0.85, transform shape` to second TikZ figure

#### 7. Chapter 4 (`chapters/chapter4.tex`)
- Added `{\small ...}` around `tabularx` in plan d'adressage IP table
- Replaced `point-à-point` with `point-a-point` (UTF-8 encoding fix)

#### 8. Annexe B (`backmatter/annexeB.tex`)
- Added `{\small ...}` around both `tabularx` tables

### Final Status
- **74 pages**, A4 format
- **0 overfull hbox warnings**
- **15 underfull hbox warnings** (cosmetic)
- **1 harmless microtype warning**
