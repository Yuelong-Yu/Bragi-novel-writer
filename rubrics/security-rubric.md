# Security Rubric

Screening criteria for content-security (Chinese legal prohibitions) and copyright-security (defamation + intellectual property) review.

Unlike craft rubrics, this rubric is **violation-based**, not score-based. The reviewer does not produce a weighted score; the reviewer enumerates violations against the clauses below and assigns each a severity. Pass/Fail is determined by the highest severity found.

## Scope

Two categories, screened in parallel:

1. **Content Security** — 23 statutory clauses (网络信息内容生态治理规定 + 通用法律禁止情形)
2. **Copyright Security** — 名誉权 (right of reputation) + 知识产权 (intellectual property)

## Pass Threshold

- **PASS**: zero CRITICAL violations AND zero HIGH violations
- **FAIL**: any CRITICAL or HIGH violation
- MEDIUM and LOW violations may coexist with a PASS; they are noted for human review but do not block.
- Maximum **3 iteration rounds** before human intervention (`max_security_rounds`, default 3)

---

## Category A — Content Security (23 statutory clauses)

The clause text is authoritative. When citing a violation, reproduce the clause verbatim — do not translate or paraphrase.

### A1. 违反宪法所确定的基本原则的
- **Default severity**: CRITICAL
- **What to evaluate**: Any passage that, read in context, contradicts or attacks the basic principles established by the Constitution (e.g., socialist system, leadership of the Party, sovereignty of the people). Fictional framing does not immunize.
- **Depicts vs promotes**: Depicting a character holding such views, with clear narrative condemnation, is usually allowed. Promoting the view as normative or authoritative → violation.

### A2. 危害国家安全，泄露国家秘密，颠覆国家政权，破坏国家统一的
- **Default severity**: CRITICAL
- **What to evaluate**: Passage advocates or instructs subversion, separatism, or disclosure of genuine state secrets. Distinguish from fictional geopolitics — a fictional separatist movement framed as antagonist is allowed; endorsing real-world separatism is not.

### A3. 损害国家荣誉和利益的
- **Default severity**: CRITICAL
- **What to evaluate**: Passage deliberately dishonors the nation or its symbols in a way that, on balance, endorses the dishonoring rather than depicting it as wrong.

### A4. 歪曲、丑化、亵渎、否定英雄烈士事迹和精神，以侮辱、诽谤或者其他方式侵害英雄烈士的姓名、肖像、名誉、荣誉的
- **Default severity**: HIGH
- **What to evaluate**: Real, named or clearly identifiable 英雄烈士 (martyrs and heroes recognized under the Heroes and Martyrs Protection Law) depicted with fabricated discreditable acts, insults, or distortion of their deeds. Generic fictional heroes are NOT covered here.

### A5. 宣扬恐怖主义、极端主义或者煽动实施恐怖活动、极端主义活动的
- **Default severity**: CRITICAL
- **What to evaluate**: Passage promotes or provides instructional detail for terrorism/extremism. Depicting a terrorist as a villain is allowed; instructing on methods or endorsing the cause is not.

### A6. 煽动民族仇恨、民族歧视，破坏民族团结的
- **Default severity**: CRITICAL
- **What to evaluate**: Passage incites hatred or discrimination against an ethnic group, or attacks ethnic unity. A prejudiced character depicted as wrong is allowed; narrative endorsement of prejudice is not.

### A7. 破坏国家宗教政策，宣扬邪教和封建迷信的
- **Default severity**: CRITICAL
- **What to evaluate**: Passage promotes 邪教 (cults, as designated by authorities) or propagates superstition as factual guidance. Distinguish from religious depiction and from genre conventions (xuanhuan/xianxia cultivation is genre fiction, not superstition propaganda).

### A8. 散布谣言，扰乱社会秩序，破坏社会稳定的
- **Default severity**: HIGH
- **What to evaluate**: Passage presents fabricated facts about real public events in a way likely to disrupt social order. Fictional plots are not rumors; misrepresenting real incidents as fact is.

### A9. 散布淫秽、色情、赌博、暴力、恐怖或者教唆犯罪的
- **Default severity**: HIGH
- **What to evaluate**: Pornographic depiction, instructional gambling/crime content, or graphic violence/terror presented gratuitously. Genre-appropriate violence (e.g., wuxia combat) is allowed; graphic sexual content or crime instruction is not.

### A10. 侮辱或者诽谤他人，侵害他人名誉、隐私和其他合法权益的
- **Default severity**: HIGH
- **What to evaluate**: Insult or fabrication directed at an identifiable real person, or disclosure of private information. See also Category B-A (名誉权) — a passage can violate both.

### A11. 使用夸张标题，内容与标题严重不符的
- **Default severity**: MEDIUM
- **What to evaluate**: Chapter/section title makes a sensational claim the body does not deliver. Applies to titled chapters; flag only when the gap is material, not stylistic.

### A12. 炒作绯闻、丑闻、劣迹等的
- **Default severity**: MEDIUM
- **What to evaluate**: Passage gratuitously sensationalizes real-person gossip/scandals/disgraces beyond narrative necessity.

### A13. 不当评述自然灾害、重大事故等灾难的
- **Default severity**: MEDIUM
- **What to evaluate**: Passage comments on a real disaster/accident in a way that is disrespectful to victims, speculative about cause without basis, or exploits the event for sensationalism.

### A14. 带有性暗示、性挑逗等易使人产生性联想的
- **Default severity**: MEDIUM (HIGH if targeting real persons or graphic)
- **What to evaluate**: Sexual implication or provocation beyond genre norms. Romance genre tolerates more than thriller; judge relative to genre and target audience.

### A15. 展现血腥、惊悚、残忍等致人身心不适的
- **Default severity**: MEDIUM (HIGH if graphic or sustained)
- **What to evaluate**: Graphic gore, horror, or cruelty that causes disproportionate discomfort. Genre expectations matter: horror tolerates more than cozy mystery.

### A16. 煽动人群歧视、地域歧视等的
- **Default severity**: MEDIUM (HIGH if targeted at real, identifiable groups)
- **What to evaluate**: Passage incites discrimination by population group or region. A prejudiced character depicted as wrong is allowed; narrative endorsement is not.

### A17. 宣扬低俗、庸俗、媚俗内容的
- **Default severity**: MEDIUM
- **What to evaluate**: Content that is vulgar, kitschy, or pandering beyond genre and audience expectations.

### A18. 可能引发未成年人模仿不安全行为和违反社会公德行为、诱导未成年人不良嗜好等的
- **Default severity**: HIGH
- **What to evaluate**: Passage depicts imitable unsafe behavior, especially with instructional clarity or appeal, likely to be copied by minors. Consider whether the work is accessible to minors.

### A19. 侵害未成年人合法权益或者损害未成年人身心健康的内容
- **Default severity**: HIGH
- **What to evaluate**: Content harming minors' rights or wellbeing, including sexualization of minors, exploitation, or psychological harm. CRITICAL if sexualization of minors.

### A20. 其他对网络生态造成不良影响的内容
- **Default severity**: MEDIUM
- **What to evaluate**: Catch-all for content that materially degrades online ecosystem health and is not better captured by A1–A19. Use sparingly; cite the specific harm.

### A21. 煽动非法集会、结社、游行、示威、聚众扰乱社会秩序
- **Default severity**: CRITICAL
- **What to evaluate**: Passage incites illegal assembly, association, protest, or collective disruption of social order.

### A22. 以非法民间组织名义活动的
- **Default severity**: CRITICAL
- **What to evaluate**: Passage operates under the name of an unauthorized civil organization, or promotes such activity.

### A23. 含有法律、行政法规禁止的其他内容的
- **Default severity**: CRITICAL
- **What to evaluate**: Catch-all for content prohibited by other laws/regulations not enumerated above. Use only when a specific legal prohibition applies; cite the law if possible.

---

## Category B — Copyright Security

Two sub-categories. A single passage can violate both B-A and B-B, and can also violate a Category A clause (e.g., A10 + B-A).

### B-A. 名誉权 (Right of reputation)

- **Default severity**: HIGH (CRITICAL if the depiction is severely defamatory or targets a vulnerable real person)
- **What to evaluate**:
  - A fictional character is, in context (name + background + identifying details), recognizably a real, living or recently deceased individual, AND is depicted in a way that lowers their social evaluation (insult, fabrication of discreditable acts, exposure of private information).
  - A real organization, brand, or institution is named and depicted in a defamatory or materially misleading way.
  - Private facts (home address, ID number, private medical information, private sexual conduct) of a real, identifiable person are disclosed.
- **Boundary**: Do not flag generic fictional villains who happen to share a common surname with a real person. The entity must be *reasonably recognizable* given name + context + identifying details.

### B-B. 知识产权 (Intellectual property)

- **Default severity**: CRITICAL for verbatim copying; HIGH for close paraphrase / distinctive-likeness copying; MEDIUM for trademark misuse (HIGH if misleading-endorsement)
- **What to evaluate**:
  - Verbatim or near-verbatim copying of protected expression from a third-party work (sentence-level identity beyond fair-use length, e.g. > 13 consecutive characters identical in Chinese, or distinctive phrasing copied).
  - Close paraphrasing that retains the expressive originality of a protected source.
  - Distinctive, protected character likenesses, world elements, or scene-pulling sequences copied from a specific living-author work (not generic genre tropes).
  - Trademarks or branded material used in a way that misleads consumers or implies endorsement.
  - In `adaptation` mode: source-work mapping reproduces protected expression rather than transforming it into a new, independently coherent story.
  - In `expansion` mode: expanded material reproduces protected expression from the original draft's source if that source is itself third-party IP.
- **Boundary**: Generic genre tropes (the chosen one, the dark lord, the capital city, the evil empire) are NOT IP violations. Short, attributed, transformative quotes within fair use are NOT IP violations.

---

## Severity Definitions

| Severity | Meaning | Action |
|----------|---------|--------|
| CRITICAL | Material that cannot ship under any interpretation. Statutory prohibition (A1–A3, A5–A7, A21–A23), verbatim IP copying, or severely defamatory depiction of a real person. | Must be removed or fundamentally rewritten before publish. Blocks PASS. |
| HIGH | Material with serious legal or platform risk. Statutory prohibition with context-dependent application (A4, A8–A10, A18–A19), close paraphrase, distinctive-likeness copying, trademark misuse with endorsement implication. | Strong directive to fix. Blocks PASS. |
| MEDIUM | Material with potential risk depending on reader, jurisdiction, or framing. Borderline content (A11–A17, A20), neutral trademark references. | Recommend fix; flag for human review. Does not block PASS. |
| LOW | Edge-case or stylistic concern that a cautious reviewer would flag. | Note in report. Does not block PASS. |

## Default Severity Quick Reference

| Clause | Default severity |
|--------|------------------|
| A1, A2, A3, A5, A6, A7, A21, A22, A23 | CRITICAL |
| A4, A8, A9, A10, A18, A19 | HIGH |
| A11, A12, A13, A20 | MEDIUM |
| A14, A15, A16, A17 | MEDIUM (HIGH if targeted at real persons/groups, or if content is graphic) |
| B-A | HIGH (CRITICAL if severely defamatory or targeting a vulnerable real person) |
| B-B | CRITICAL (verbatim) / HIGH (paraphrase, likeness) / MEDIUM (trademark, neutral) |

Defaults are starting points. The reviewer may upgrade or downgrade severity per passage based on context, intent, reach, and audience — but the reasoning must be stated in the violation entry.
