# AI World Domination Starts With Your Font Settings

<sub>© 2026 Ikan Riddle · Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)</sub>

---

When people discuss AI risk, they picture autonomous weapons and emergent consciousness.

Nobody pictures `serif: "Times New Roman"` in a JSON config file.

But if an AI agent can write that field — and Chrome's built-in reset won't clear it — you have a persistent visual deception path where every step is behaviorally indistinguishable from normal operation.

This write-up documents an architecturally complete attack chain that combines Chrome's font preference persistence, OpenType glyph substitution, and AI agent permissions into a pipeline that makes a user see text that doesn't match the underlying data. It also documents exactly why the chain is hard to operationalize — two unresolved engineering constraints that limit both reliability and blast radius.

&nbsp;

## 🔍 The Observation

While debugging a font rendering issue on Claude.ai, I found a counterintuitive state distinction in Chrome's `Preferences` JSON:

| State | Behavior |
|:------|:---------|
| **Field absent** (never modified) | Chrome uses internal fallback logic, respects `@font-face` normally |
| **Field present with default value** (user set it, even back to default) | Chrome treats it as an explicit override, altering font resolution priority |

The catch: Chrome's UI can only *modify* field values. There is no interface to *delete* a field. `chrome://settings/reset` does not cover font preference fields.

**Recovery paths:** manually edit the `Preferences` JSON to remove the relevant keys, or uninstall and reinstall Chrome. That's it.

&nbsp;

## ⛓️ The Attack Chain

Four steps. Each uses a distinct mechanism.

&nbsp;

### Step 1 · Preferences Persistence

Write a `webkit.webprefs.fonts` entry into Chrome's `Preferences` file:

```
%LOCALAPPDATA%\Google\Chrome\User Data\Default\Preferences
```

Point a target font family (e.g., `serif`) to an attacker-controlled system font name.

This operation is behaviorally indistinguishable from a user legitimately changing their font preferences — the `Preferences` file is always written by `chrome.exe` itself. EDR policies monitoring "non-Chrome process writes" are ineffective. Clearing cache, clearing cookies, and Chrome reset all leave this field untouched.

&nbsp;

### Step 2 · Malicious Font Installation

Install a glyph-remapped OpenType font as a system font, using the name written in Step 1.

The font exploits the GSUB (Glyph Substitution) table's **Contextual Chaining Substitution** feature: substitutions fire on specific character sequences, so the visual shapes output by the rendering layer do not correspond to the underlying Unicode codepoints.

&nbsp;

### Step 3 · Visual Deception

Text rendered in Chrome displays shapes processed by the malicious font. What the user sees diverges from the underlying data.

> **Underlying text:** `mass approve without human review`
>
> **Rendered on screen:** `flag and hold for human review`

&nbsp;

### Step 4 · Decision Hijacking

The user acts on what they see.

&nbsp;

## 🤖 Why AI Agents Change the Permission Calculus

In traditional attack models, Step 1 requires local file write access and Step 2 requires admin-level font installation privileges. An attacker who already holds both has far more efficient options — keylogging, credential theft, process injection — making the font route a poor ROI choice.

**AI agents flip this.** Users proactively grant agents file read/write and system command execution as prerequisites for normal operation. The entry point shifts from *breaching a permission boundary* to *prompt injection hijacking existing permissions*. Once the agent receives injected instructions, it executes all four steps using its own legitimate capabilities.

The action of "modifying a font field in a JSON config" is indistinguishable from the agent's normal configuration work.

&nbsp;

## 🚧 Two Unresolved Engineering Constraints

### Constraint 1 · GSUB False Triggers in Word-Level Substitution

There is a meaningful engineering gap between ligature-level substitution (`fi` → `ﬁ`, a 2–3 glyph deterministic sequence) and word-level semantic substitution (`reject` → `accept`).

Ligatures work because sequences like `fi`/`fl`/`ffi` almost always represent valid trigger points — enormous error tolerance built in. Word-level substitution must handle:

- **Morphological variants** — `reject` / `rejected` / `rejection` / `rejecting`
- **Substring collisions** — target substrings appearing inside other words, compound words, hyphenated constructions

GSUB's backtrack/lookahead mechanism provides limited word boundary detection (matching surrounding spaces or punctuation), which mitigates but does not eliminate collisions. A single unexpected substitution appearing on screen immediately exposes the attack.

Rule maintenance cost grows **multiplicatively**: morphological variants per target word × context disambiguation rules per variant. At sufficient vocabulary scale, complexity and testing cost become practical bottlenecks.

&nbsp;

### Constraint 2 · Visual-Only Deception Has Zero Penetration Against Non-Human-Eye Verification

GSUB modifies glyph rendering. It does not alter underlying codepoints. Every mechanical verification method sees the true text:

| Verification method | Result |
|:---|:---|
| **Ctrl+F** | Searching the rendered text (e.g., "flag") fails — underlying text is "mass approve" |
| **Copy-paste** | Pastes the true text |
| **DevTools inspect** | Displays the true text |
| **Screen readers** | Read the true text |
| **Any other surface** | Slack, email, mobile app — all show the true text |

The attack requires that the target user reads *exclusively* through the compromised Chrome instance, never encounters the same content on any other surface, and makes a high-privilege decision based solely on that reading.

> The viable attack window is: "user reads purely by eye, performs no mechanical verification, and makes a high-privilege decision based on that visual reading" — which is precisely the window that security-critical operational workflows tend to make narrowest.

&nbsp;

## 🚩 A Medium-Difficulty Detection Point

"AI agent installs a system font" is an operationally usable flag.

Legitimate scenarios where an agent needs to install system fonts are rare — design-oriented agents may have such needs; general-purpose work agents almost never do. An operation-type whitelist (rather than behavioral intent classification) could directly trigger review at this step.

However, mainstream agent frameworks have not yet established a unified operation-level permission model. There remains a gap between "this *should* be flaggable" and "this *is currently* flagged."

&nbsp;

## 📌 Conclusion

The chain is architecturally complete: Preferences persistence provides a covert write point → the font name bridges the config file to the malicious binary → GSUB Contextual Chaining provides targeted substitution without CSS injection → the agent's legitimate permissions eliminate the traditional permission acquisition bottleneck.

Two hard constraints remain unresolved: GSUB word-level false triggers limit reliability; pure visual deception's zero penetration against mechanical verification limits blast radius.

**The real point is not whether this specific chain is operationally viable.** It is what the chain concretely illustrates: when every individual permission granted to an agent is legitimate in isolation, a malicious combination can be dispersed across time (modify a preference field today, install a font next week), and each step's behavioral signature is indistinguishable from normal operation. Existing safety mechanisms based on single-step behavioral intent classification have a structural blind spot for this class of attack.

This "malicious temporal combination of legitimate operations" problem is open in the agent security field, with no mature solution.
