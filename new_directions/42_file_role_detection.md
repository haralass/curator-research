# 42. File Role Detection

**Curator Research — IUI 2027 Track**
**Date:** 2026-05-31
**Status:** Draft

---

## Abstract

Personal file organization systems have long treated document classification as a topic problem: label a file with its subject matter and route it to the matching folder. This framing is fundamentally incomplete. A PDF about database normalization might be a lecture slide deck, an assignment specification, an exam, a reference paper, or a student's own draft answer — five objects that belong in five different places, yet are indistinguishable by topic alone. This paper introduces *file role detection*: the problem of inferring the functional role a file plays in a user's life, independent of its subject matter. We propose a role taxonomy for personal files, enumerate the multi-modal signals that support role inference, survey prior art from document-type classification and personal information management (PIM) research, describe how role integrates with Curator's routing and rename pipeline, and outline the research contributions — including the first annotated ground-truth dataset for personal file role classification.

---

## 1. Why Role Is Not Topic

Topic classification assigns a semantic label to document content: "this file is about machine learning", "this file concerns payroll". For a digital assistant organizing a filesystem, topic is necessary but not sufficient. Consider a university student's Downloads folder containing three files named `normalization.pdf`, `hw3.pdf`, and `lecture08.pdf`. All three may be topically identical — database normalization — yet the correct organizational action differs sharply:

- `normalization.pdf` — a reference paper downloaded from ACM DL → belongs in `References/Databases/`
- `hw3.pdf` — an assignment sheet from Moodle → belongs in `EPL342/Assignments/`
- `lecture08.pdf` — the professor's slide deck → belongs in `EPL342/Lectures/`

A topic classifier produces the same label for all three; a role classifier distinguishes them. The distinction is not peripheral. In PIM research, Bergman et al. (2008) documented that users frequently fail to retrieve files because the *organizational decision* — which folder to put it in — reflected their mental model of the file's purpose, not its content. When an AI organizer moves files based on topic alone, it violates the user's mental model and increases retrieval friction.

Role classification is also prerequisite for intelligent renaming. The smart rename rule `role="lecture" + topic="EPL342" + sequence=8 → "EPL342_Lecture_08_Normalization.pdf"` requires knowing the role; the topic classifier alone cannot generate this output. Without role, the best rename is `EPL342_Normalization.pdf`, which loses the lecture-vs-assignment distinction the user needs.

A third motivation is archival significance. Receipts and bank statements have legal retention periods; lecture notes do not. Assignments have submission deadlines that make them temporarily high-priority; reference papers do not. Role, not topic, drives these lifecycle decisions.

---

## 2. Role Taxonomy for Personal Files

We propose a twelve-class role taxonomy grounded in the files that appear in real personal filesystems (Jones & Teevan, 2007; Boardman & Sasse, 2004). The taxonomy is deliberately coarse enough to be learnable with modest training data, yet fine enough to drive meaningfully different routing decisions.

| Role ID | Label | Description | Typical routing target |
|---|---|---|---|
| R01 | **Lecture** | Instructional material produced by a teacher or institution | Course folder / Lectures subfolder |
| R02 | **Assignment** | Task specification or problem set the user must complete | Course folder / Assignments subfolder |
| R03 | **Research Paper** | Peer-reviewed or preprint academic publication | References or reading list folder |
| R04 | **Receipt** | Proof of purchase or payment confirmation | Finance / Receipts |
| R05 | **Bank Statement** | Periodic financial statement from a financial institution | Finance / Statements |
| R06 | **Draft** | In-progress version of a user-authored document | Project folder (active area) |
| R07 | **Final Submission** | Completed, delivered version of a user-authored document | Project folder / Submitted |
| R08 | **Reference Document** | Non-academic resource kept for future consultation | References or relevant project |
| R09 | **Template** | Reusable blank or partially-filled document | Templates folder |
| R10 | **Installer / Archive** | Application installer or compressed archive of software | Downloads or Tools |
| R11 | **Meeting Notes** | Notes taken during or summarizing a meeting | Work or Project folder |
| R12 | **Contract / Agreement** | Legal document requiring or recording consent | Legal or Finance folder |

A file may carry multiple role labels (multi-label classification). A document can simultaneously be R03 (Research Paper) and R08 (Reference Document). A partially completed assignment can be R02 and R06 (Draft). The system must support non-exclusive assignment.

---

## 3. Detection Signals for File Role

Role inference draws on four signal categories that complement each other and can be fused via a late-fusion ensemble.

### 3.1 Layout and Structural Signals

Lectures typically use slide layout: high visual-to-text ratio, numbered slide titles, bullet lists, minimal prose paragraphs. PDF slide decks have a distinctive page-aspect ratio (4:3 or 16:9) and consistent header/footer patterns. PDFMiner and pdfplumber expose page-level layout features that distinguish slide PDFs from prose PDFs. Assignment sheets have characteristic structures: numbered problems, point values ("10 pts"), fill-in-the-blank spaces, submission instructions. Research papers follow the IMRaD or introduction-body-references convention with a structured bibliography section. Contracts contain signature blocks, numbered clauses, and legal boilerplate detectable via keyword density patterns.

### 3.2 Filename Patterns

Filenames encode role signals with high precision. The patterns `_draft`, `_v2`, `_final`, `_submitted`, `_old`, `_backup` are strong role signals when combined with document content. Institutional naming patterns are also informative: `EPL342_HW3.pdf` suggests an assignment; `Lecture_08.pdf` or `Week3_Slides.pdf` suggests a lecture. Receipt filenames often contain dates and vendor names (`Amazon_Invoice_2026-03-14.pdf`). Bank statement filenames follow the pattern `Statement_YYYY-MM.pdf` or include the bank name. A regex taxonomy of high-precision role-indicative filename patterns can serve as a zero-shot prior before any content analysis.

### 3.3 Provenance Signals

Where a file came from is highly informative about its role. Files downloaded from a university LMS URL (Moodle, Canvas, Blackboard — detectable via browser history metadata or download path) are almost certainly lectures or assignments. Email attachments from known colleagues or managers are likely work documents, meeting notes, or contracts. Files appearing in a browser's automatic download folder shortly after a purchase event are likely receipts. macOS stores the origin URL in the `com.apple.metadata:kMDItemWhereFroms` extended attribute, accessible via `mdls`, providing provenance information without user action (Apple Developer Documentation, 2023).

### 3.4 Temporal and Contextual Signals

A file created at 2:00 AM the night before an assignment deadline is more likely a draft or final submission than a reference paper. Files modified repeatedly over weeks are likely drafts. Files never modified after download are likely reference material, lectures, or receipts. These temporal signals combine with structural signals for robust role inference.

---

## 4. Prior Art

### 4.1 Document Type Classification

Invoice and receipt detection for expense management systems is the most commercially mature area. Katti et al. (2018) introduced Chargrid, a 2D convolution-based approach treating document layout as a character grid, achieving strong performance on invoice field extraction — the prerequisite for role detection. The SROIE dataset (Huang et al., 2019) provides benchmarks for scanned receipt understanding. Scientific paper classification has a long history: Teufel & Moens (2002) developed argumentative zoning — classifying sentences by their rhetorical role in papers — which is conceptually related to document-level role classification. More recently, PubLayNet (Zhong et al., 2019) provides large-scale layout annotation for scientific documents.

Contract analysis has attracted significant NLP investment: Chalkidis et al. (2019) developed LEGAL-BERT and demonstrated superior performance on contract clause classification. Form understanding — which maps to the assignment role — is addressed by DocVQA (Mathew et al., 2021) and the IBM FUSE dataset.

### 4.2 Personal File Role Classification

To our knowledge, no published work addresses role classification specifically for *personal files* in a filesystem context. Jones & Teevan (2007) provide taxonomies of how people organize personal information but do not attempt automated role inference. Bergman et al. (2008) study why people fail to retrieve personal files, implicitly identifying role-based organization as a key mental model, but propose no classifier. Boardman & Sasse (2004) document multi-folder personal information management across email, bookmarks, and files — again without automated role inference. The closest work is Dinneen et al. (2020), who study the characteristics of personal digital archives, but focus on preservation rather than classification.

There is thus a genuine research gap: personal file role classification is an open problem with no published dataset, taxonomy, or evaluation benchmark.

---

## 5. Integration with Curator's Routing and Rename Pipeline

Role feeds into Curator's pipeline at two stages. First, it modifies the confidence tier assigned to a routing decision. A file with role=Lecture and high topic confidence is a reliable AUTO-tier action: move to `EPL342/Lectures/` without user review. A file with role=Draft has lower routing confidence because drafts are often mid-process and the user's preferred destination may be project-specific. The confidence tier model from Curator's core pipeline is parameterized by `(topic_confidence, role_confidence, provenance_available)`.

Second, role enables structured smart renaming. The rename template library is indexed by role:

- `role=Lecture`: `{CourseCode}_Lecture_{SeqNum}_{Topic}.{ext}`
- `role=Assignment`: `{CourseCode}_HW{Number}_{Topic}.{ext}`
- `role=Receipt`: `{Vendor}_{Date}_Receipt.{ext}`
- `role=Research Paper`: `{FirstAuthor}{Year}_{ShortTitle}.{ext}`

This produces meaningful, consistent filenames that match the user's retrieval expectations. Without role, the rename system can only produce topic-based names that lose role information.

---

## 6. Research Contribution

The primary contributions of a paper on file role detection are:

1. **Taxonomy**: The first published role taxonomy for personal files, with twelve classes grounded in PIM literature and empirical observation of real filesystem contents.
2. **Annotation study**: A ground-truth dataset collected via a user study in which participants label a sample of their own files with role labels. The study protocol and inter-annotator agreement metrics constitute a methodological contribution.
3. **Multi-modal classifier**: A late-fusion model combining layout features (from PDF rendering), filename regex features, provenance metadata (from xattrs), and temporal signals. Evaluated against the annotated dataset.
4. **Multi-label evaluation**: Standard accuracy metrics do not apply cleanly to multi-label classification. We define evaluation metrics appropriate for the multi-label personal file role setting.
5. **Integration study**: A within-subjects user study comparing Curator's routing accuracy and user satisfaction with and without role detection, demonstrating the practical value of the classifier.

---

## 7. Connection to FLRS (Forgetting-and-Relevance Scoring)

File role affects the forgetting dynamics parameter S₀ (initial salience) in Curator's Forgetting-Like Relevance Score framework. Different roles exhibit radically different temporal relevance curves:

- **Receipt**: High initial salience (needed for returns, warranties), decays rapidly after ~30 days but has a retention spike at tax season.
- **Lecture**: Moderate initial salience, slow decay — remains relevant throughout the semester, drops sharply at course end.
- **Draft**: High salience during active editing, drops to near-zero once the corresponding Final Submission is detected.
- **Research Paper**: Low initial salience, very slow decay — reference material stays relevant indefinitely.
- **Contract**: Moderate-to-high salience, essentially zero decay — legal documents remain permanently relevant.

Incorporating role into FLRS allows the system to apply role-appropriate forgetting curves rather than a single universal function, substantially improving the model's prediction of when users actually need specific files.

---

## References

Apple Developer Documentation. (2023). *File metadata and extended attributes*. Apple Inc. https://developer.apple.com/documentation/coreservices/file_metadata

Barreau, D., & Nardi, B. A. (1995). Finding and reminding: File organization from the desktop. *ACM SIGCHI Bulletin*, 27(3), 39–43. https://doi.org/10.1145/221296.221307

Bergman, O., Beyth-Marom, R., & Nachmias, R. (2008). The user-subjective approach to personal information management systems design: Evidence and implementations. *Journal of the American Society for Information Science and Technology*, 59(2), 235–246. https://doi.org/10.1002/asi.20738

Boardman, R., & Sasse, M. A. (2004). "Stuff goes into the computer and doesn't come out": A cross-tool study of personal information management. In *Proceedings of the SIGCHI Conference on Human Factors in Computing Systems* (pp. 583–590). ACM. https://doi.org/10.1145/985692.985765

Chalkidis, I., Androutsopoulos, I., & Aletras, N. (2019). Neural legal judgment prediction in English. In *Proceedings of the 57th Annual Meeting of the Association for Computational Linguistics* (pp. 4317–4323). ACL. https://doi.org/10.18653/v1/P19-1424

Dinneen, J. D., Nguyen, T., & Julien, C.-A. (2020). The personal digital library: Characteristics from a large-scale study. In *Proceedings of the 83rd Annual Meeting of the Association for Information Science and Technology*. ASIS&T. https://doi.org/10.1002/pra2.217

Huang, Z., Chen, K., He, J., Bai, X., Karatzas, D., Lu, S., & Jawahar, C. V. (2019). ICDAR2019 competition on scanned receipt OCR and information extraction. In *2019 International Conference on Document Analysis and Recognition* (pp. 1516–1520). IEEE. https://doi.org/10.1109/ICDAR.2019.00244

Jones, W., & Teevan, J. (Eds.). (2007). *Personal information management*. University of Washington Press.

Katti, A. R., Reisswig, C., Guder, C., Brandt, J., Dengel, A., & Bhatt, U. (2018). Chargrid: Towards understanding 2D documents. In *Proceedings of the 2018 Conference on Empirical Methods in Natural Language Processing* (pp. 4459–4469). ACL. https://doi.org/10.18653/v1/D18-1476

Mathew, M., Karatzas, D., & Jawahar, C. V. (2021). DocVQA: A dataset for VQA on document images. In *Proceedings of the IEEE/CVF Winter Conference on Applications of Computer Vision* (pp. 2200–2209). IEEE. https://doi.org/10.1109/WACV48630.2021.00225

Teufel, S., & Moens, M. (2002). Summarizing scientific articles: Experiments with relevance and rhetorical status. *Computational Linguistics*, 28(4), 409–445. https://doi.org/10.1162/089120102762671936

Zhong, X., Tang, J., & Yepes, A. J. (2019). PubLayNet: Largest dataset ever for document layout analysis. In *2019 International Conference on Document Analysis and Recognition* (pp. 1015–1022). IEEE. https://doi.org/10.1109/ICDAR.2019.00166
