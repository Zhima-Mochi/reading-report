# Chapter 3 — Naming

## TL;DR (high-signal recall)
- Names shape how users understand your API; they are part of the contract.
- “Good” names are typically expressive, simple, and predictable.
- Changing public API names is hard (like changing a phone number); plan names as long-lived.
- Pick a consistent language, grammar, and syntax; consistency often beats cleverness.
- Prepositions in names are often “API smells” that hint at a deeper modeling problem.
- Include units in primitive field names and use richer data types when primitives become unclear.

## Why names matter (beyond style)
- Names in compiled code can disappear after compilation/minification; API names do not.
- API names are the surface area every user sees, copies, and bakes into client code.
- Renaming is expensive because you cannot update all consumers (especially private codebases).
- APIs are learned through repetition:
  - Users build mental models from names (resources, fields, methods).
  - Inconsistent naming breaks those models and increases cognitive load.
- Names are coupled to:
  - Documentation and examples
  - Client SDKs and generated code
  - Query languages, filters, field masks
  - Long-term compatibility (renames are breaking changes)

## What makes a name “good”
Think of these as constraints you balance, not a single scoring system.

### Expressive
- The name communicates intent without extra explanation.
- A user can guess what it does and what data it represents.
- Avoid ambiguous terms when your domain has collisions (e.g., “topic” can mean messaging topics or topic modeling).
- When collisions are likely, add a clarifying qualifier (e.g., `messagingTopic` vs `modelTopic`) to prevent misinterpretation.
- Heuristics:
  - Prefer concrete domain terms over internal implementation terms.
  - Prefer terms users already use (ubiquitous language) over “engineering slang.”

### Simple
- Minimize verbosity while preserving clarity.
- Avoid encoding multiple concepts into one name.
- Extra words should pay rent: every added token should reduce confusion in a realistic way.
- Heuristics:
  - If a name needs multiple qualifiers, you may be mixing concerns.
  - If you need a “kitchen sink” field name, consider splitting the concept.

### Predictable
- Users can infer naming patterns and apply them elsewhere.
- Heuristics:
  - Use the same term for the same concept everywhere.
  - Use consistent suffixes/prefixes (e.g., `...Id`, `...Count`, `is...`).
  - Keep consistent case and separator rules.
- Predictability is largely about **consistent behavior + consistent naming**: similar things should look similar and behave similarly.

## Language, grammar, and syntax
This chapter emphasizes a key practice: choose conventions and stick to them.

### Language
- Pick one natural language for identifiers (typically English).
- Avoid mixing languages (even if your team is multilingual).
- Prefer common, widely understood terms over rare synonyms.

### Grammar
- **Resources**: typically nouns (e.g., `Message`, `User`, `ChatRoom`).
- **Fields**: typically noun phrases or adjectives (e.g., `createTime`, `displayName`, `isActive`).
- **Methods**: verbs (standard ones are usually implied; custom ones are explicit).
- Prefer names that describe what the thing *is*, not how it is stored or computed.

### Syntax
- Choose casing (camelCase vs snake_case) and apply consistently.
- Be consistent with pluralization (collections vs single resources).
- Use consistent casing per artifact type when your IDL encourages it (e.g., messages vs fields).
- Avoid ambiguous abbreviations:
  - If abbreviations are necessary, standardize them and document the glossary.

### Case styles (common options)
- **camelCase**: `firstName`
- **snake_case**: `first_name`
- **kebab-case**: `first-name`
- The exact choice matters less than consistency; inconsistency causes misclassification (e.g., a message type that looks like a field).

### Reserved keywords
- Avoid using restricted keywords as names in your API definition language or target languages.
- If a word is likely to be reserved (`to`, `from`, `string`, etc.), pick a more specific domain term (e.g., `sender`/`recipient`, `payer`/`payee`).
- Keyword avoidance is most important for multi-language APIs; it prevents awkward client SDK ergonomics.

## Naming “smells” to watch for
- **Prepositions** inside names (e.g., “messagesForUser”, “policiesOfRoom”):
  - Often indicates a relationship modeling issue:
    - Maybe you need a new resource or a better hierarchy.
    - Maybe you need a better standard method on a collection.
- “And” names (e.g., `createAndPublish`):
  - Often indicates a method doing too many things or hiding side effects.
- Overloaded terms:
  - Same word used for different concepts (e.g., “id” means different identifier types).
  - Different words used for the same concept (e.g., `createdAt` vs `createTime` vs `creation_date`).

## Context (names don’t exist in isolation)
- A name’s meaning is affected by where it appears:
  - Field names inside a resource (context provided by the resource type)
  - Method names (context provided by the target resource/collection)
  - Nested objects (context provided by containing field names)
- Risk:
  - A name that is clear in one context becomes misleading in another.
  - Example pattern: `name` can mean “human-friendly display name” or “resource identifier.”
- Context can add helpful meaning (e.g., “record” inside an audio recording API) but it can also mislead when a term carries strong meaning from elsewhere.

## Data types and units (naming + semantics)
- For primitives, include units when they matter:
  - `timeoutSeconds`, `sizeBytes`, `priceCents`, `latDegrees`
- A field name like `size` is ambiguous across contexts (bytes, seconds, pixels); adding units makes intent clear (`sizeBytes`, `sizeMegapixels`).
- Prefer rich types when possible:
  - Instead of `startTimeMs` consider a timestamp type (and document timezone/format).
  - Instead of `moneyAmount` consider structured money representation.
- When a primitive would require extra formatting rules, a richer type often clarifies both structure and meaning (e.g., `Dimensions { lengthPixels, widthPixels }` instead of `"1024x768"`).
- Why this matters:
  - Unit confusion is a common source of client bugs and production incidents.
  - Defaults and bounds often depend on units.

## Case study: choosing bad names (how it fails)
Use this as a checklist for spotting and fixing naming problems:
- Names that are too generic (`data`, `value`, `info`) force users to read docs constantly.
- Names that expose implementation (`dbId`, `mongoKey`) reduce flexibility.
- Inconsistent patterns force clients to special-case code and break predictability.

### Subtle meaning: “max” vs “exact”
- When a field is a maximum (upper bound), naming it like an exact value misleads clients.
- The “max” framing prevents clients from assuming “if I asked for 10, I must get 10,” which is often false in distributed systems (latency/consistency/availability constraints).
- A misnamed “pageSize” that is actually a maximum encourages clients to stop early when they receive fewer results than requested.

### Units mismatch: why names must expose assumptions
- The Mars Climate Orbiter failure illustrates how hidden unit assumptions can cause catastrophic integration errors.
- Including units in names makes mismatches obvious at the boundary (e.g., `impulsePoundForceSeconds` vs `impulseNewtonSeconds`) and blocks unsafe “pipe output into input” assumptions.

## Practical “naming checklist” (use when designing)
- Names reflect user/domain language (not internal architecture).
- Similar resources and fields use consistent terminology across the API.
- Collections are plural and single resources are singular, consistently.
- Boolean names use a positive true value form (e.g., `isEnabled`).
- Identifier fields use consistent suffixes (e.g., `...Id`) and stable types.
- Primitive units are explicit and defaults are documented.
- A new user can infer intent and behavior from names alone.

## Quick recall (answers)
- Expressive names make the represented concept unambiguous to someone without insider context.
- Simple names avoid unnecessary length while preserving clarity; extra qualifiers exist only when they reduce real confusion.
- Predictable names follow consistent patterns so users can guess new names and behaviors from familiar ones.
- Prepositions in names often signal that the API is encoding a relationship/query in the name instead of modeling it as resources, hierarchy, or standard list/filter behavior.
- Unit-bearing names prevent implicit assumptions from leaking across team/system boundaries and reduce integration risk.
