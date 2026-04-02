# Chrome Preferences Font Override × AI Agent: An Architecturally Complete Attack Chain and Its Engineering Constraints

## Origin

While debugging a font rendering issue on Claude.ai, I identified a counterintuitive state distinction in Chrome's `Preferences` JSON file: a field being absent (never modified — Chrome uses its internal fallback logic and respects `@font-face` declarations normally) versus a field being present with its default value (explicitly set by the user, even if set back to default). The latter alters the priority behavior of Chrome's font resolution pipeline. The UI layer can only modify field values; there is no interface to delete a field itself. `chrome://settings/reset` does not cover font preference fields. The only recovery paths are manually editing the `Preferences` file to remove the relevant fields, or uninstalling and reinstalling Chrome.

This observation raised a question: can this persistence path constitute an attack vector?

## Attack Chain Architecture

The attack proceeds in four steps, each using a distinct mechanism:

**Step 1: Preferences persistence.** Write a `webkit.webprefs.fonts` entry into Chrome's `%LOCALAPPDATA%\Google\Chrome\User Data\Default\Preferences`, pointing a target font family (e.g., serif) to an attacker-controlled system font name. This operation is behaviorally indistinguishable from a user legitimately changing their font preferences — the `Preferences` file is always written by the `chrome.exe` process itself (triggered via the Settings UI). EDR policies monitoring "non-Chrome process writes" are ineffective here. Clearing cache, clearing cookies, and Chrome reset all leave this field untouched.

**Step 2: Malicious font installation.** Install a glyph-remapped OpenType font as a system font, using the same name written in Step 1. This font exploits the GSUB (Glyph Substitution) table's Contextual Chaining Substitution feature to perform substitutions on specific character sequences — the visual shapes output by the rendering layer do not correspond to the underlying Unicode codepoints.

**Step 3: Visual deception takes effect.** Text rendered in Chrome displays shapes processed by the malicious font; what the user sees on screen diverges from the underlying data. For example, the underlying text reads "mass approve without human review," but the screen renders "flag and hold for human review."

**Step 4: Decision hijacking.** The user acts on what they see, executing instructions that were preset by the attacker.

## How the AI Agent Scenario Changes the Permission Calculus

In traditional attack models, Step 1 requires local file write access and Step 2 requires system font installation privileges (typically administrator-level). An attacker who already holds both has far more efficient attack paths available — keylogging, credential theft, process injection — making the font route a poor ROI choice.

The AI agent scenario changes this calculation. Users proactively grant agents file read/write and system command execution capabilities as prerequisites for normal operation. The attack entry point shifts from "breaching a permission boundary" to "prompt injection hijacking existing permissions" — once the agent receives injected malicious instructions, it executes the four steps above using its own legitimate permissions. The action of "modifying a font field in the Preferences JSON" is behaviorally indistinguishable from the agent's normal configuration operations.

## Two Unresolved Engineering Constraints

### Constraint 1: False trigger problem in GSUB word-level substitution

OpenType GSUB Contextual Chaining Substitution supports multi-character sequence matching and is theoretically capable of word-level substitution. However, there is a meaningful engineering gap between ligature-level substitution (fi → ﬁ, a 2–3 glyph deterministic sequence) and word-level semantic substitution (reject → accept).

Ligatures are reliable precisely because sequences like fi/fl/ffi almost always represent valid trigger points in natural language text, providing enormous error tolerance. Word-level substitution must handle morphological variants (reject/rejected/rejection/rejecting) and substring collisions (target substrings appearing inside proper nouns or other words). GSUB's backtrack/lookahead mechanism allows limited word boundary detection (matching surrounding spaces or punctuation), which mitigates but does not eliminate substring collisions — compound words and hyphenated constructions remain failure cases. A single unexpected substitution appearing on screen immediately exposes the attack.

Rule maintenance cost grows multiplicatively as the target vocabulary expands (number of morphological variants per target word × number of context disambiguation rules per variant). When the rule set must cover a sufficient number of semantic substitution pairs, its complexity and testing cost become practical bottlenecks.

### Constraint 2: Pure visual deception does not penetrate any non-human-eye verification

GSUB modifies glyph rendering; it does not alter underlying codepoints. This means:

- **Ctrl+F search**: Searching for the rendered visible text (e.g., searching "flag") will fail, because the underlying text is "mass approve."
- **Copy-paste**: The pasted output is the true text.
- **DevTools inspect**: Displays the true text.
- **Screen readers**: Read out the true text.
- **Multi-surface exposure**: Agent output is typically delivered simultaneously to a browser page, a Slack notification, an email digest, a mobile app, and other surfaces. Chrome Preferences font override only affects the rendering pipeline of that specific Chrome instance; every other surface displays the true text. The attack requires that the target user reads exclusively through the compromised Chrome instance, never encounters the same content on any other surface, and makes a high-privilege decision based solely on that reading.

The viable attack window is constrained to: "user reads purely by eye, performs no mechanical verification, and makes a high-privilege decision based on that visual reading" — which is precisely the window that security-critical operational workflows tend to make narrowest.

## A Medium-Difficulty Detection Point

"AI agent installs a system font" is an operationally usable flag. Legitimate scenarios in which an agent needs to install system fonts are rare (design-oriented agents may have such needs; general-purpose work agents almost never do). If an agent safety layer uses an operation-type whitelist rather than behavioral intent classification, this step can directly trigger review. However, mainstream agent frameworks have not yet established a unified operation-level permission model. There remains a gap between "this should be flaggable" and "this is currently flagged."

## Conclusion

The attack chain is architecturally complete: Preferences persistence provides a covert write point; the system font name bridges the Preferences file to the malicious font binary; GSUB Contextual Chaining provides targeted glyph substitution without requiring any additional CSS injection; and the agent's legitimate permissions eliminate the permission acquisition bottleneck present in traditional attack models.

There are two unresolved hard engineering constraints: GSUB word-level rule false triggers in natural language limit attack reliability, and pure visual deception does not penetrate any non-human-eye verification method, limiting blast radius.

The value of this chain lies not in whether it is operationally viable on its own, but in what it concretely illustrates about a more general agent security problem: when each individual permission granted to an agent is legitimate in isolation, a malicious combination can be executed dispersed across time (modify a font preference field today, install a font file next week), and each individual step's behavioral signature is indistinguishable from normal operation — existing safety mechanisms based on single-step behavioral intent classification have a structural blind spot. This "malicious temporal combination of legitimate operations" problem is open in the agent security field, with no mature solution.
