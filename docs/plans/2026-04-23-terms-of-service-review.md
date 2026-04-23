# Terms of Service Review & Recommendations

**Date:** 2026-04-23
**Scope:** `frontend/src/components/pages/terms-of-service/terms-of-service.page.tsx` and
`frontend/src/components/shared/terms-of-service/table.tsx` (the CC BY summary table).

## Critical gaps (high priority)

### 1. No DMCA / copyright takedown procedure
For a hosted UGC + generative-AI service, the absence of a DMCA § 512(c)
notice-and-takedown policy and designated agent is a material omission.
Without it you likely lose safe-harbor protection for infringing user uploads.
Add a section with the designated agent's contact and the standard
counter-notice procedure, and register the agent with the U.S. Copyright
Office.

### 2. No Privacy Policy reference
ToS references no Privacy Policy, yet you collect emails, WorkOS identities,
and content. GDPR, CCPA/CPRA, and most app-store policies require a linked,
conspicuous privacy notice. Link one, and reference it in §3 (Communication).

### 3. Content Policy is dangerously narrow (§6)
For a generative AI platform, the prohibited-content list is missing:
- **CSAM / sexual content involving minors** (must be explicit and zero-tolerance)
- Deepfakes / non-consensual synthetic likenesses of real people (NCII is
  mentioned but not deepfakes generally)
- Content that infringes third-party IP (copyright, trademark, right of publicity)
- Violent extremism, terrorism, incitement
- Fraud, deception, election/medical/financial disinformation
- Malware, phishing, doxxing, spam
- Illegal goods/services

Also: §6.a.iii prohibits any content containing "logos, watermarks, bugs,
titles, or credits" — written broadly, this bans any branded content, which
is almost certainly overbroad. Tighten to *"overlaid/baked-in watermarks or
credits"*.

### 4. No liability cap for direct damages (§10.b)
You disclaim indirect/consequential damages but never cap aggregate
direct-damage liability (typical: *"greater of fees paid in prior 12 months
or US$100"*). Free services almost always need this.

### 5. Arbitration clause is skeletal (§14)
Missing:
- Class-action and jury-trial waiver (without these, arbitration gives the
  company very little)
- Arbitration provider and rules (AAA/JAMS + Consumer Rules)
- Seat / venue
- Carveout for IP / injunctive relief
- 30-day informal dispute-notice precondition
- 30-day opt-out right (required for enforceability under recent case law in
  several states)
- Mass-arbitration protocol

### 6. "Survival" clause is actually a severability clause (§15)
The heading says Survival but the text is identical to §10.d severability.
True survival lists which sections persist after termination (license grant,
disclaimers, liability, arbitration, governing law, etc.). Rename and rewrite.

### 7. No termination / account-closure clause
Neither the user's right to delete their account nor the company's right to
suspend or terminate (with or without cause, for violations) is defined.
Also no consequence for ToS breach.

### 8. EULA is referenced but not defined (§1.a)
"These Terms of Service, which include our End User License Agreement" — but
no EULA text is linked or embedded. Either drop the reference or attach it.

## Significant issues (medium priority)

### 9. "Non-commercial use only" (§4) is undefined and paired with an informal aside
"(b) We hope to make this an option in the future." doesn't belong in a legal
document. Define "non-commercial" (typically: not primarily intended for or
directed toward commercial advantage or monetary compensation, borrowing CC's
phrasing) or delete the restriction. As written, this ambiguously prohibits a
huge range of legitimate creator/pro use.

### 10. Change-of-terms notice mechanism is inadequate (§2)
"We will post announcements through social media and discord" is not legally
adequate notice and not reliable evidence of consent. Industry standard:
email registered users, post a "Last Updated" date, require re-acceptance
for material changes, and possibly maintain a public changelog. There is
also **no Effective Date / Last Updated date anywhere** in the document —
add one.

### 11. License grant in §7.c is broader than needed
"worldwide, nonexclusive, royalty-free, **sublicensable and transferable**" —
combined with permission to use content for marketing, "transferable" can
alarm creators and is unnecessary for operating the service. Narrow to
*"sublicensable solely to service providers acting on the Company's behalf
and to operate and promote the Service"*.

### 12. "No generative training" carveout is a loophole (§7.f)
"The company may analyze the content with AI for the purpose of running the
service, eg search and recommendations, and training models for these
purposes." A plaintiff could argue a recommendation model is a generative
model, or that "running the service" covers almost anything. Tighten: list
permitted model categories (embeddings, classifiers, moderation,
recommendation) and expressly exclude text-to-image/text-to-video and other
generative models.

### 13. "Commercially reasonable period" for post-deletion license (§7.e) is vague
Define a number (e.g., 30 days after deletion for cached/CDN copies; backups
retained up to 12 months, not displayed).

### 14. User warranties and indemnification are absent
§7.b only has the user "attest" their content is legal. Standard practice:
- Representations: user owns or has rights to content; has obtained all
  necessary releases (including right-of-publicity releases for depicted
  persons); content does not infringe, defame, or violate privacy
- Indemnification: user indemnifies the company for third-party claims
  arising from their content or breach of ToS

### 15. NSFW policy has enforcement holes (§6)
"NSFW content must be declared as such on upload" — no stated consequence
for failure to declare, and no mention of automated or human moderation.
Add a consequence (removal, account action) and mention that the company
may classify/re-classify content at its discretion.

### 16. Governing law lacks venue and the Brazil carveout is unexplained (§11)
Add an exclusive-venue provision (e.g., state or federal courts in [County],
NY for non-arbitrable matters). The Brazil-only carveout suggests someone
tried to address consumer law but stopped there — consider a general
"subject to mandatory local consumer-protection laws" clause instead of
singling out one country.

### 17. Typographical errors in §8
- §8.a: `"citing you by name with link)"` — stray closing paren, missing open paren
- §8.d: `"(e.g., by citing you by name with link) or other appropriate attribution"` — same issue

## Nice-to-have additions

- **AI-content labeling**: EU AI Act Art. 50 and California AB-2013 impose
  AI-content disclosure obligations; reference your disclosure/watermarking
  practice.
- **Acceptable Use clause** covering scraping, reverse engineering,
  rate-limit abuse, automated account creation, model extraction, and API
  misuse (relevant given `python-api` / `edream_sdk`).
- **Responsible disclosure / security bug reporting** contact.
- **Export controls / sanctions** (OFAC, dual-use AI).
- **Assignment, force majeure, entire agreement, notices** — standard closers.
- **Data retention** statement.
- **Claims-limitation period** ("any claim must be brought within one year").
- **Minors' data** language (even with 13+ gate).

## Recommended order of operations

1. **Now**: Add DMCA takedown + designated agent; expand §6 to include CSAM,
   deepfakes, IP infringement, and illegal content; link a Privacy Policy;
   fix the "Survival" vs severability bug; add Last-Updated date.
2. **This quarter**: Rewrite arbitration (class-waiver, provider, opt-out,
   carveouts); add liability cap; add termination, warranties, and
   indemnification; define "non-commercial" or remove; tighten the §7.f
   AI-training carveout and §7.c "transferable" language.
3. **Ongoing**: AI-disclosure, acceptable use, assignment, force majeure, and
   general polish. Have an attorney review once structural fixes are in.
