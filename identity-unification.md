# Identity Unification System — Design Document

## Problem

Authors contact BookLeaf from multiple platforms using different identifiers:

| Platform | Identifier |
|---|---|
| Email | sara.johnson@xyz.com |
| WhatsApp | +91 9876543210 |
| Dashboard | Sara J. |
| Instagram | @sarapoetry23 |

Without unification, each platform creates a separate support thread with no shared context — leading to duplicate work, missed history, and poor author experience.

---

## Goal

Link all platform identities to a single internal `author_id` with a confidence score, and flag low-confidence matches for manual verification.

---

## System Flow

```
Incoming contact signal
(email / phone / name / social handle)
            ↓
┌─────────────────────────────────────┐
│  STEP 1: Exact Email Match          │
│  confidence → 1.0                   │
│  action    → AUTO LINK ✅           │
└──────────────┬──────────────────────┘
               │ no match
               ↓
┌─────────────────────────────────────┐
│  STEP 2: Exact Phone Match          │
│  confidence → 0.95                  │
│  action    → AUTO LINK ✅           │
└──────────────┬──────────────────────┘
               │ no match
               ↓
┌─────────────────────────────────────┐
│  STEP 3: Fuzzy Name Match           │
│  (Levenshtein distance ≤ 2)         │
│  confidence → 0.75 – 0.85           │
│  action    → SUGGEST VERIFY ⚠️      │
└──────────────┬──────────────────────┘
               │ no match
               ↓
┌─────────────────────────────────────┐
│  STEP 4: LLM Semantic Match         │
│  (social handle / name pattern)     │
│  confidence → 0.50 – 0.74           │
│  action    → SUGGEST VERIFY ⚠️      │
└──────────────┬──────────────────────┘
               │ no match
               ↓
┌─────────────────────────────────────┐
│  NO MATCH FOUND                     │
│  confidence → 0.0                   │
│  action    → VERIFY MANUALLY 🔴     │
└─────────────────────────────────────┘

FINAL DECISION:
  confidence >= 0.85  →  Auto-link to author profile
  confidence 0.60–0.84 → Flag as "Likely match — please verify"
  confidence < 0.60   →  "Cannot determine — verify manually"
```

---

## Pseudocode

```javascript
async function unifyIdentity(input) {
  const { email, phone, name, handle } = input;
  const candidates = [];

  // ─── STEP 1: Exact email match ─────────────────────────────────────────
  if (email) {
    const match = await db.authors.findOne({ email: email.toLowerCase() });
    if (match) {
      return {
        author_id: match.id,
        confidence: 1.0,
        method: 'email_exact',
        action: 'auto_link'
      };
    }
  }

  // ─── STEP 2: Exact phone match ─────────────────────────────────────────
  if (phone) {
    const normalised = phone.replace(/\D/g, ''); // strip non-digits
    const match = await db.authors.findOne({ phone: normalised });
    if (match) {
      return {
        author_id: match.id,
        confidence: 0.95,
        method: 'phone_exact',
        action: 'auto_link'
      };
    }
  }

  // ─── STEP 3: Fuzzy name match (Levenshtein) ────────────────────────────
  if (name) {
    const allAuthors = await db.authors.findAll();
    const fuzzyMatches = allAuthors
      .map(author => ({
        author,
        distance: levenshtein(
          author.name.toLowerCase(),
          name.toLowerCase()
        )
      }))
      .filter(m => m.distance <= 2)
      .sort((a, b) => a.distance - b.distance);

    if (fuzzyMatches.length > 0) {
      const best = fuzzyMatches[0];
      const confidence = 0.85 - (best.distance * 0.05); // 0.85, 0.80, or 0.75
      candidates.push({
        author_id: best.author.id,
        confidence,
        method: 'fuzzy_name'
      });
    }
  }

  // ─── STEP 4: LLM semantic match on social handle ───────────────────────
  if (handle && candidates.length === 0) {
    const allAuthors = await db.authors.findAll();
    const authorNames = allAuthors.map(a => ({ id: a.id, name: a.name }));

    const llmResult = await callLLM(`
      An author contacted us via Instagram with the handle: "${handle}"
      Here is a list of registered authors: ${JSON.stringify(authorNames)}
      
      Does this handle likely belong to one of these authors?
      Consider: handle may contain their real name, initials, or a nickname.
      
      Return ONLY JSON:
      {
        "author_id": "<id or null>",
        "confidence": <0.0 to 0.74>,
        "reasoning": "<brief explanation>"
      }
    `);

    if (llmResult.author_id) {
      candidates.push({
        author_id: llmResult.author_id,
        confidence: llmResult.confidence,
        method: 'llm_semantic',
        reasoning: llmResult.reasoning
      });
    }
  }

  // ─── FINAL DECISION ────────────────────────────────────────────────────
  if (candidates.length === 0) {
    return {
      author_id: null,
      confidence: 0.0,
      method: 'no_match',
      action: 'verify_manually'
    };
  }

  const best = candidates.sort((a, b) => b.confidence - a.confidence)[0];

  if (best.confidence >= 0.85) {
    return { ...best, action: 'auto_link' };
  } else if (best.confidence >= 0.60) {
    return { ...best, action: 'suggest_verify' };
  } else {
    return { ...best, action: 'verify_manually' };
  }
}
```

---

## Confidence Score Reference

| Score | Meaning | Action |
|---|---|---|
| 1.0 | Exact email match | Auto-link ✅ |
| 0.95 | Exact phone match | Auto-link ✅ |
| 0.85 | Fuzzy name, distance 0 | Auto-link ✅ |
| 0.80 | Fuzzy name, distance 1 | Suggest verify ⚠️ |
| 0.75 | Fuzzy name, distance 2 | Suggest verify ⚠️ |
| 0.50–0.74 | LLM semantic handle match | Suggest verify ⚠️ |
| 0.0 | No signal found | Verify manually 🔴 |

---

## Example: Sara Johnson

```
Input signals:
  email:  sara.johnson@xyz.com
  phone:  +91 9876543210
  name:   Sara J.
  handle: @sarapoetry23

Step 1: email match → author found (confidence 1.0)
Result: AUTO LINK ✅ — no further steps needed
```

```
Input signals (WhatsApp only):
  phone: +91 9876543210

Step 1: email — not provided, skip
Step 2: phone match → author found (confidence 0.95)
Result: AUTO LINK ✅
```

```
Input signals (Instagram DM only):
  handle: @sarapoetry23

Step 1: email — skip
Step 2: phone — skip
Step 3: name — skip
Step 4: LLM compares "@sarapoetry23" against all author names
        → likely "Sara Johnson", confidence 0.68
Result: SUGGEST VERIFY ⚠️ — "Likely Sara Johnson, please confirm"
```

---

## What I Would Add With More Time

- **Cross-signal scoring** — combine multiple weak signals (handle + name + location) into a single weighted confidence score instead of stopping at first match
- **Platform identity table** in Supabase to persist verified links permanently
- **Admin review UI** — a simple dashboard showing all `suggest_verify` and `verify_manually` cases for the support team to resolve with one click
- **Feedback loop** — when a human verifies a match, store the result and use it to train better signal weights over time
