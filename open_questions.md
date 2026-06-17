# Open Questions & Ambiguities — Project Clarification Log

---

## 1. File Numbering *(Critical — touches Create File, Finalise File, both registers)*

- What is the format of a File Number? *(e.g. `SU30-2025-001` or just `001`?)*
- Is it the same number for Inward and Outward, or separate sequences?
- What triggers a yearly reset — Jan 1 automatically, or a manual admin action?
- Who assigns File Numbers — does the system auto-generate, or does Admin pre-create a list that users pick from a dropdown?

---

## 2. Outward Register Number vs. File Number

These appear to be two different things. File Number is picked at Create File; Register Number is assigned at Finalise.

- Confirm: are they always different?
- Does the Register Number format need to match any official scheme?

---

## 3. Draft Files

- Can one draft be finalised multiple times (revised versions), or is it one-shot — finalise once, done?
- Can a finalised file ever be re-opened or modified?

---

## 4. Inward — Scan Path

- Is the scanned file attached/uploaded through the browser, or is it always placed manually into a folder beforehand and the officer simply picks the filename?

---

## 5. Address Book — Type Field

- **FR-037** mentions a "Type" column. What are the valid values? *(e.g. Internal / External / Others?)*
- **FR-039** states the default is "Others" — what are the other options?

---

## 6. CC Sent To (Inward) and CC (Outward)

- Is CC stored as a list of address IDs (multi-value), or as free text?

---

## 7. Referred To (Inward)

- Is this a dropdown from the address book, a free-text field, or both?

---

## 8. Users / Officers

- Does "Created By" in forms always mean the logged-in user, or can an officer file on behalf of someone else?
- Is Department a free-text field or a dropdown (another master list)?

---

## 9. Admin Panel Placeholders *(FR-046 to FR-049)*

| Area | Open Question |
|---|---|
| Audit Log | Any filters needed? *(by user / by date range / by action type)* |
| Backup | Manual button only, or scheduled? *(e.g. daily at a fixed time)* |
| Yearly Reset | Does it reset both Inward and Outward sequences, or just one? |
| Master List Management | Can items be deleted, or only deactivated? |

---

## 10. Remarks Field

- Appears in both Inward and the draft queue *(FR-012)*.
- Is this free text with no length limit, or is there a character cap?

---

## 11. Legacy XLSX Migration ⚠️ *Highest Priority for Today*

- What columns exist in each of the **3 XLSX files**?
- The schema needs to map them to the new tables exactly.

> **If you're getting those XLSX files today — bring them.**  
> Everything else above is answerable by the stakeholder in one conversation.

---

*Document generated: 2026-06-17*
