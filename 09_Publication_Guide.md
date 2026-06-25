# How to Publish in a Reputed Journal
## Step-by-Step Guide for the Modular UAV Fleet Paper

---

## Part 1: Choose the Right Journal

Match your journal to your paper's strongest angle. This paper has three strengths: (1) a hardware/systems experimental methodology, (2) multi-domain integration across propulsion, structure, AI, and energy, and (3) a novel fleet-architecture contribution. The best-fit journals, ranked by practicality for a first-time submission:

---

### Tier 1 — Strong Fit, Realistic Entry Point

**IEEE Access**
- Type: Open access
- Impact Factor: ~3.9 (2024)
- APC: $1,995 USD (waivable for low-income countries — India qualifies for partial waiver)
- Review time: 4–8 weeks (fast by IEEE standards)
- Why: Covers all IEEE disciplines; multi-domain engineering papers are a natural fit; faster than Transactions journals; reputable enough for industry and academic audiences
- Submission link: ieeeaccess.ieee.org
- Format: IEEE two-column LaTeX or Word template

**Drones — MDPI**
- Type: Open access (MDPI)
- Impact Factor: ~4.8 (2024)
- APC: CHF 2,600 (~$2,850 USD); waiver/discount available for authors from developing countries
- Review time: 2–5 weeks (very fast; MDPI uses rapid peer review)
- Why: Dedicated drone journal, so every reviewer is domain-expert; UAV fleet architecture papers are highly relevant; good citation visibility
- Submission link: mdpi.com/journal/drones
- Format: MDPI LaTeX or Word template

**Aerospace Science and Technology — Elsevier**
- Type: Subscription (open access optional at extra cost)
- Impact Factor: ~5.6 (2024)
- APC (open access): $3,450 USD (but free to publish without open access)
- Review time: 2–4 months
- Why: Prestigious Elsevier journal for aerospace engineering; strong readership in propulsion and structures; multi-domain papers are well-received
- Submission link: journals.elsevier.com/aerospace-science-and-technology

---

### Tier 2 — Higher Impact, More Competitive

**IEEE Transactions on Aerospace and Electronic Systems (TAES)**
- Impact Factor: ~5.1 (2024)
- APC: $0 (subscription model; open access optional)
- Review time: 3–8 months
- Why: Very prestigious; covers UAV systems, navigation, and control; harder to get in but the credibility is higher
- Note: Requires very strong experimental validation and novel theoretical contribution

**IEEE Robotics and Automation Letters (RA-L)**
- Impact Factor: ~4.6 (2024)
- APC: $1,750 (open access option)
- Review time: 3–5 months
- Why: High-quality robotics journal; the AI navigation and fleet coordination sections are strong fits
- Note: Length limit 8 pages — the paper will need tightening

**Journal of Field Robotics — Wiley**
- Impact Factor: ~4.2 (2024)
- APC: ~$3,000 (open access)
- Review time: 3–6 months
- Why: Specialises in real-world autonomous system deployment; the fleet endurance and docking results are exactly this journal's domain

---

### Recommended Submission Strategy

**If this is a first journal publication:**
Start with **Drones (MDPI)** or **IEEE Access**. Both have fast review cycles, reasonable acceptance rates for well-executed work, and sufficient prestige for industry and academic citations. The multi-domain framing is a strength in both venues.

**If targeting higher impact:**
Submit to **Aerospace Science and Technology** first. If rejected, revise using reviewer feedback and resubmit to IEEE Access.

**Do not** submit to multiple journals simultaneously — this violates publication ethics.

---

## Part 2: Format Your Paper Correctly

Every journal requires a specific template. Using the wrong format is grounds for desk rejection (rejection without review).

### IEEE Journals (Access, TAES, RA-L)

Download the official IEEE template:
- LaTeX: `bare_jrnl.tex` from ieeeauthorcenter.ieee.org/create-your-ieee-article/use-authoring-tools-and-ieee-article-templates/ieee-article-templates/
- Word: IEEE_template.docx from the same page

**Key IEEE formatting rules:**
- Two-column layout (11pt Times New Roman)
- Abstract: 150–250 words, no citations, no abbreviations unexplained
- Keywords: 5–10 terms
- Equations: numbered on the right margin, referenced as "Eq. (1)"
- Tables: numbered with Roman numerals (Table I, Table II)
- Figures: numbered with Arabic numerals (Fig. 1, Fig. 2)
- References: IEEE format [1], [2], etc. — not author-year
- Page limit: IEEE Access has no page limit; TAES 12 pages typical; RA-L 8 pages strict

### MDPI Drones

Download from mdpi.com/authors/word — includes Word and LaTeX templates.

**Key MDPI rules:**
- Single-column layout for submission
- Abstract: 200–300 words
- Keywords: 5–10
- Section numbering: 1, 2, 3 (not Roman numerals)
- Equations: LaTeX-style numbering
- Figures must be submitted separately as high-resolution files (300 DPI minimum for print, 600 DPI for line art)

### Elsevier (Aerospace Science and Technology)

Download from author.elsevier.com/Templates — "Elsevier article" LaTeX class.

---

## Part 3: Prepare Required Materials

Every journal submission requires:

**1. Cover Letter (1 page)**
Write directly to the Editor-in-Chief. Include:
- What the paper is about (2–3 sentences)
- Why it belongs in this specific journal (1 sentence)
- Statement of novelty — what has not been published before
- Confirmation that the work is original, not under review elsewhere, and not published before
- Suggested reviewers (optional but helps) — name 3–5 domain experts with contact details

Sample opening:
> "Dear Editor, We submit for your consideration our manuscript titled 'Synergistic Multi-Domain Optimization of Autonomous UAV Fleet Systems...' This paper addresses three gaps in the UAV literature [state them]. We believe the manuscript is well-suited to [journal name] because [1 sentence reason]. The primary novel contributions are [list 3 briefly]. We confirm the manuscript has not been published or submitted elsewhere."

**2. Highlights (Elsevier only, 3–5 bullet points)**
Example:
- W^1.5 hover power scaling verified to <0.1% across all payload configurations
- FOC reduces vibration by 11.3×, improving AI navigation accuracy by 44.7%
- CFRP × FOC interaction improves SOC estimation by 57%, extending mission endurance 6.3%
- SPF autonomous navigation achieves 92.7% real-world success (2.5 pp gap to human)
- Fleet rotation yields 7+ hour continuous coverage from 45-minute per-drone windows

**3. Author Contribution Statement (most journals)**
Since this is a single-author paper:
> "S. Kumar: Conceptualisation, Methodology, Software, Formal Analysis, Investigation, Data Curation, Writing – Original Draft, Writing – Review & Editing, Visualisation."

**4. Data Availability Statement**
> "All simulation scripts, experimental raw data, and processing pipelines are archived at github.com/uav-systems-lab/advanced-uav-research-2026 under CC BY 4.0."

**5. Conflict of Interest Statement**
> "The author declares no conflict of interest."

**6. Funding Statement**
> "This research received no external funding." (or state if funded)

**7. Ethics Statement (if human subjects involved)**
Not required for this paper.

---

## Part 4: The Submission Process Step by Step

### Step 1: Register on the Journal's Submission Portal
- IEEE Access: mc.manuscriptcentral.com/ieee-access
- MDPI Drones: susy.mdpi.com
- Elsevier (AST): ees.elsevier.com/ast

### Step 2: Prepare Your Files
- Main manuscript (LaTeX PDF or Word DOCX) — typically without author information for double-blind review
- Title page with author information (separate file)
- Figures as separate high-resolution files if required
- Cover letter (PDF)
- Any supplementary materials

### Step 3: Submit and Track
After submission you will receive:
- Immediate confirmation email with a manuscript ID
- Editor assignment notification (1–5 days)
- "Under Review" status when sent to peer reviewers
- Decision email (accept / minor revision / major revision / reject)

**Common status meanings:**
- "With Editor" — awaiting assignment or desk review
- "Under Review" — at least one reviewer has agreed; typically takes 4–8 weeks
- "Required Reviews Completed" — reviews received, editor deciding
- "Decision in Process" — editor writing decision

### Step 4: Respond to Reviewers

If you receive revisions (minor or major), this is normal and expected — not a rejection. Write a **Response to Reviewers** document:
- Address every comment, numbered, point by point
- Quote the original comment, then write your response
- Indicate every change made to the manuscript with page/line numbers
- Be polite — "We thank the reviewer for this insightful comment..."
- Never argue without evidence; if you disagree, cite data or literature

### Step 5: Proofing and Publication
After acceptance:
- Check proofs carefully (publisher PDF) for errors
- Complete copyright transfer or open access license agreement
- Pay APC if applicable (waiver applications should be submitted at this stage if needed)
- Article publishes online first (often within 1–2 weeks of proof approval)

---

## Part 5: Before You Submit — Checklist

### Content Checklist
- [ ] Every performance figure cross-referenced against first principles or cited experimental data
- [ ] All equations numbered and referenced in text
- [ ] All tables titled with Roman numerals and captioned above the table
- [ ] All figures titled with Arabic numerals and captioned below the figure
- [ ] References formatted in correct journal style (IEEE format for IEEE journals)
- [ ] Abstract is standalone — can be understood without reading the paper
- [ ] Novelty clearly stated in Introduction (what does this paper add that is not already published?)
- [ ] Limitations section present and honest
- [ ] Future work identified

### Ethics Checklist
- [ ] All data is real — no fabricated results
- [ ] Any figures or tables from other papers are cited and permission obtained if required
- [ ] No portion of the paper is submitted elsewhere simultaneously
- [ ] All co-authors (if any) have approved the submission
- [ ] No plagiarism — use iThenticate or similar before submission (most journals check automatically)

### Technical Checklist
- [ ] PDF compiles without LaTeX errors
- [ ] Figures are at 300 DPI minimum (600 DPI for line art)
- [ ] File size is within submission limits (typically 50 MB)
- [ ] All fonts embedded in PDF
- [ ] Supplementary data archived with stable URL (GitHub, Zenodo, or similar)

---

## Part 6: Timeline Expectations

| Journal | Submission to First Decision | Revision Period | Publication After Acceptance |
|---|---|---|---|
| MDPI Drones | 2–5 weeks | 2–4 weeks | 1–2 weeks |
| IEEE Access | 6–10 weeks | 4–6 weeks | 2–4 weeks |
| Aerospace Science & Technology | 8–16 weeks | 4–8 weeks | 4–8 weeks |
| IEEE TAES | 12–24 weeks | 4–8 weeks | 4–8 weeks |
| IEEE RA-L | 10–16 weeks | 4–8 weeks | 2–4 weeks |

**Realistic timeline from submission to published paper:** 3–8 months for first-tier journals; 1.5–3 months for MDPI.

---

## Part 7: After Publication

**Maximise your paper's reach:**

1. **Post to arXiv** (preprint server) — do this *before* or *on the day of* submission to the journal. arXiv posts are free, immediately indexed, and widely read. Submit to cs.RO (Robotics) or eess.SY (Systems and Control).
   - arXiv link: arxiv.org/submit

2. **Post to ResearchGate** — upload the accepted manuscript (not the publisher PDF, which is copyrighted) after publication.

3. **Share on Devfolio and social platforms** — link to the arXiv preprint from project pages and blog posts.

4. **Add to Google Scholar** — if your author profile isn't auto-indexed, add it manually at scholar.google.com.

5. **Cross-reference your blog posts** — the three Devfolio posts can link to the arXiv preprint, building citation pathways from the builder community.

---

## Part 8: Common Rejection Reasons and How to Avoid Them

| Rejection Reason | How to Avoid |
|---|---|
| "Out of scope for this journal" | Read 10 recent papers in the journal first; cite 3–5 of them |
| "Insufficient novelty" | State explicitly in Introduction what has NOT been done before; cite the closest related work |
| "Results not validated" | Every claim needs either: (a) experimental measurement, (b) FEA confirmation, or (c) physics derivation |
| "Wrong format / template not used" | Always use official template; never submit a raw LaTeX or Word with non-standard fonts |
| "Abstract promises what the paper doesn't deliver" | Write abstract last, after all results are final |
| "References are out of date" | Include papers from 2023–2025; reviewers notice if you ignore recent work |
| "Self-plagiarism" | Don't copy verbatim from your own previous papers; paraphrase and cite |

---

*This guide was compiled in May 2026 for submission of the paper "Synergistic Multi-Domain Optimization of Autonomous UAV Fleet Systems." APC costs and impact factors are current as of May 2026 and should be verified at time of submission.*
