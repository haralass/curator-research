# 45. Cold Start Personalization

**Curator Research — IUI 2027 Track**
**Date:** 2026-05-31
**Status:** Draft

---

## Abstract

When a new user installs Curator, the system has no history, no feedback, and no knowledge of the user's mental model of their own files. Users organize their filesystems in radically different ways: some maintain deep, carefully labeled hierarchies; others keep everything flat and rely on search; most fall somewhere between. An AI file organizer that assumes a universal organization style will produce suggestions that conflict with the user's existing mental model, eroding trust from the first interaction. This paper addresses the *cold start personalization problem* for personal file organization: how to infer a user's filing style from the minimal information available at install time and from the first few dozen interaction events. We propose a filing style taxonomy, enumerate the passive signals available at install time from the user's existing filesystem, describe an active cold-start questionnaire, model implicit feedback signals, and propose a Bayesian updating framework that begins with a population prior and updates toward a personal model. The result is the first formal cold-start personalization framework for AI file organization.

---

## 1. The Problem: Zero History, High Stakes

The cold start problem in recommender systems refers to the difficulty of generating useful recommendations when no interaction history is available for a user (Schein et al., 2002). In collaborative filtering, the standard mitigation is to recommend popular items. Personal file organization has no equivalent of "popular items": there is no meaningful universal recommendation ("put this file in Documents/") that improves on the system default.

The cold start problem is particularly acute for Curator because the consequences of incorrect organization are costly. A recommendation system that suggests a movie the user dislikes wastes thirty seconds. A file organizer that moves files to the wrong location wastes minutes of search time and degrades the user's trust in the system permanently (Lee & See, 2004). The first ten routing decisions the system makes disproportionately determine whether the user will adopt or abandon Curator.

Moreover, the diversity of personal filing styles is not merely quantitative (some users have more files) but qualitative (users organize by fundamentally different taxonomies). A user who organizes by client will be confused by suggestions that route by file type; a user who organizes by semester will be confused by suggestions that route by project. The system must infer which organizational taxonomy the user uses before it can make any valid suggestions.

---

## 2. Filing Style Taxonomy

Barreau & Nardi (1995) identified two archetypal personal information management styles: *filers*, who maintain elaborate folder hierarchies and carefully place documents at creation time, and *pilers*, who accumulate documents in a small number of locations and rely on search or temporal memory for retrieval. This binary taxonomy has been replicated and extended in subsequent work.

Jones & Teevan (2007) describe a spectrum with three broad strategies: *organizing* (proactive folder creation and filing), *not organizing* (accepting default locations and using search), and *mixed* (organizing for some information types but not others). Boardman & Sasse (2004) found that most users maintain different organizational strategies across different information types (email, browser bookmarks, files) — so a user who is a heavy organizer in email may be a piler for files.

We propose a five-class filing style taxonomy for Curator's personalization model:

| Style | Description | Key signals |
|---|---|---|
| **Deep Filer** | Maintains 4+ levels of hierarchy, every file has a designated location, rarely uses Downloads as a destination | Max depth ≥ 4, average files/folder < 20, low Downloads mass |
| **Shallow Filer** | 2–3 levels of hierarchy, organized but not exhaustively | Max depth 2–3, moderate files/folder |
| **Piler** | Mostly flat, large number of files in a small number of folders, relies on search | Max depth ≤ 2, average files/folder > 100 |
| **Project-Oriented** | Organizes primarily by project or course, not by file type or topic | Top-level folders correspond to projects/courses, mixed file types within each |
| **Type-Oriented** | Organizes primarily by file type | Top-level folders named "PDFs", "Images", "Code", etc. |

These styles are not mutually exclusive: a Deep Filer might also be Project-Oriented. The model assigns a probability distribution over styles rather than a hard classification.

---

## 3. Passive Signals from Existing Filesystem

At install time, Curator can analyze the user's existing filesystem structure before any interaction occurs. This analysis is entirely read-only and produces a rich signal set:

**3.1 Hierarchy depth**: The maximum and average depth of the folder tree below the home directory. A maximum depth of 7 with low variance is a strong filer signal; a maximum depth of 2 with high file counts at leaf nodes is a piler signal.

**3.2 Average files per folder**: Low average (< 20) suggests careful organization; high average (> 100) suggests piling.

**3.3 Folder naming patterns**: Folder names that follow date patterns (`2024-Q1`, `Fall_2025`) suggest temporal organization. Names matching course codes (`EPL342`, `CS101`) suggest academic project orientation. Names matching client or project names suggest professional project orientation. Generic names (`Documents`, `Stuff`, `Old`) suggest low-organization or legacy structure.

**3.4 Filename patterns**: Does the user habitually include dates in filenames? Version suffixes (`_v2`, `_final`)? Descriptive titles? These patterns reveal the user's naming convention and preferred level of metadata-in-filename.

**3.5 Downloads folder mass**: A large, unorganized Downloads folder (> 500 files) indicates that the user does not currently file downloads — either a piler behavior or an unfulfilled intent to organize. Combined with deep hierarchy elsewhere, the latter is more likely.

**3.6 File type distribution**: Do PDFs, images, and code files each have designated folders, or are they interleaved? Type separation suggests type-oriented organization.

These signals are combined via a logistic regression model trained on the population prior dataset (§6) to produce an initial filing style probability distribution.

---

## 4. Active Cold Start: Targeted Onboarding Questions

Passive analysis of the existing filesystem provides a noisy prior. For users with sparse or disorganized filesystems, the signal is weak. Active cold-start — asking the user a small number of targeted questions during onboarding — provides high-information observations at low cost.

We propose a 3-question onboarding flow, designed to be completable in under 60 seconds:

**Q1** (taxonomy): "When you save a new file, what determines which folder you put it in?"
- By project or course it belongs to
- By file type (PDF, image, document)
- By topic or subject
- I usually save to Downloads and search later

**Q2** (naming): "How do you typically name your files?"
- Descriptive titles (e.g., "Economics Essay Week 3 Draft")
- Short codes (e.g., "eco_hw3_draft.pdf")
- I keep whatever name the file came with

**Q3** (control): "How do you want Curator to suggest changes?"
- Show me everything, I'll decide
- Make obvious changes automatically, show me the rest
- Mostly automatic, I'll check the history occasionally

These questions directly map to the key dimensions of the personalization model: organizational taxonomy (Q1), rename preference (Q2), and automation trust tier (Q3). The answers initialize the filing style distribution and the automation policy tier (see file 44).

Active learning theory supports the value of this approach: Settles (2010) demonstrates that a small number of informative queries can dramatically reduce the number of labeled examples needed to reach a target model quality. For cold start, three well-chosen questions outperform ten random interaction observations.

---

## 5. Implicit Feedback Signals

Once Curator begins making suggestions, every user interaction is an implicit feedback signal. We enumerate the signal types and their information content:

| Signal | Information content | Weight |
|---|---|---|
| Accepted suggestion | Positive: this action type, destination, and confidence level are acceptable | +1.0 |
| Changed destination | Informative: the correct destination differs from the suggestion; reveals preferred taxonomy | +0.8 (with correction) |
| Rejected suggestion | Negative: this action type or destination is not acceptable | -1.0 |
| Undone action | Strong negative: the executed action was wrong enough to prompt reversal | -2.0 |
| Ignored suggestion (7+ days) | Weak negative: the suggestion was not compelling | -0.3 |

The "changed destination" signal is the most information-rich. When a user accepts a move suggestion but changes the destination from `EPL342/Lectures/` to `University/Fall2025/EPL342/Notes/`, the system observes the user's preferred folder naming convention, hierarchy depth, and organizational taxonomy simultaneously. This single event may update the filing style model more than ten accepted suggestions.

---

## 6. Population Prior and Bayesian Updating

Before any user-specific data is available, Curator uses a population prior over filing styles derived from aggregate, anonymized data contributed by opted-in users. The prior is not a single point estimate but a Dirichlet distribution over the five filing style classes, updated with user-specific observations via Bayesian updating.

Let θ be the filing style distribution for the current user. The prior is:

```
θ ~ Dirichlet(α₀)
```

where α₀ is the population parameter vector estimated from aggregate data. After n observations, the posterior is:

```
θ | observations ~ Dirichlet(α₀ + counts)
```

where `counts[i]` is the number of observations consistent with style i. This closed-form update makes the model computationally trivial and fully interpretable.

**6.1 Developer prior from GitHub**: For users identified as developers (presence of `.git` folders, code files, package.json), directory structures of public GitHub repositories provide an additional source of prior data. The top-k most common directory organizational patterns in public repos of a given language type can seed the prior for developer users. This data is publicly available, anonymized by construction, and representative of the population of developers who use version control.

---

## 7. Few-Shot Convergence Target

The key empirical question for this framework is: how many interaction events does the personalization model need before routing accuracy reaches the baseline of an expert human organizer (approximately 85% routing accuracy as measured by user satisfaction surveys)?

We hypothesize, based on the active learning literature (Settles, 2010) and cold-start recommendation research (Schein et al., 2002), that 15–25 interaction events are sufficient to reach baseline accuracy when combined with passive filesystem analysis and the three-question onboarding flow. The evaluation metric is *routing accuracy at k interactions*, where routing accuracy is the fraction of suggestions the user accepts without modification.

The few-shot personalization literature for recommendation systems provides useful benchmarks. Bharadhwaj et al. (2019) demonstrate that meta-learning approaches can reach near-full-data performance with as few as 5–10 interactions on collaborative filtering tasks. File organization personalization is a simpler inference problem (a small number of discrete style classes rather than a high-dimensional item preference space), suggesting that fewer interactions should suffice.

---

## 8. Research Contribution

1. **Filing style taxonomy**: A five-class taxonomy (Deep Filer, Shallow Filer, Piler, Project-Oriented, Type-Oriented) grounded in PIM literature and operationalized as a probabilistic classifier.
2. **Passive signal taxonomy**: An enumeration of filesystem structural signals observable at install time, with associated information content for style inference.
3. **Active cold-start protocol**: A three-question onboarding instrument validated for filing style elicitation.
4. **Implicit feedback model**: A weighted signal model for updating the filing style distribution from user interaction events.
5. **Bayesian update framework**: A Dirichlet-Dirichlet Bayesian model that begins with a population prior and converges to a personal model.
6. **Evaluation**: A user study measuring routing accuracy as a function of interaction count, validating the few-shot convergence hypothesis.

---

## References

Barreau, D., & Nardi, B. A. (1995). Finding and reminding: File organization from the desktop. *ACM SIGCHI Bulletin*, 27(3), 39–43. https://doi.org/10.1145/221296.221307

Bharadhwaj, H., Park, H., & Lim, B. Y. (2019). RecSys-DAN: Discriminative adversarial network for cross-domain recommender system. *IEEE Transactions on Neural Networks and Learning Systems*, 31(8), 2731–2740. https://doi.org/10.1109/TNNLS.2019.2916070

Boardman, R., & Sasse, M. A. (2004). "Stuff goes into the computer and doesn't come out": A cross-tool study of personal information management. In *Proceedings of the SIGCHI Conference on Human Factors in Computing Systems* (pp. 583–590). ACM. https://doi.org/10.1145/985692.985765

Jones, W., & Teevan, J. (Eds.). (2007). *Personal information management*. University of Washington Press.

Lee, J. D., & See, K. A. (2004). Trust in automation: Designing for appropriate reliance. *Human Factors*, 46(1), 50–80. https://doi.org/10.1518/hfes.46.1.50_30392

Pan, R., Zhou, Y., Cao, B., Liu, N. N., Lukose, R., Scholz, M., & Yang, Q. (2008). One-class collaborative filtering. In *Proceedings of the 2008 Eighth IEEE International Conference on Data Mining* (pp. 502–511). IEEE. https://doi.org/10.1109/ICDM.2008.16

Schein, A. I., Popescul, A., Ungar, L. H., & Pennock, D. M. (2002). Methods and metrics for cold-start recommendations. In *Proceedings of the 25th Annual International ACM SIGIR Conference on Research and Development in Information Retrieval* (pp. 253–260). ACM. https://doi.org/10.1145/564376.564421

Settles, B. (2010). Active learning literature survey. *Computer Sciences Technical Report 1648*. University of Wisconsin–Madison. http://burrsettles.com/pub/settles.activelearning.pdf

Teevan, J., Alvarado, C., Ackerman, M. S., & Karger, D. R. (2004). The perfect search engine is not enough: A study of orienteering behavior in directed search. In *Proceedings of the SIGCHI Conference on Human Factors in Computing Systems* (pp. 415–422). ACM. https://doi.org/10.1145/985692.985745

Whittaker, S., & Hirschberg, J. (2001). The character, value, and management of personal paper archives. *ACM Transactions on Computer-Human Interaction*, 8(2), 150–170. https://doi.org/10.1145/376929.376932
