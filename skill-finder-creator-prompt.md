# Claude Code Input: Create the `skill-finder` Skill

Paste into Claude Code in an empty project directory. Prerequisites: `skill-creator` skill available, and `skill-registry` MCP server already running and connected with `search_skills` and `get_skill_manifest` tools available.

---

## Prompt for Claude Code

Use the `skill-creator` skill to create a new skill called `skill-finder`. Read `skill-creator`'s SKILL.md first and follow its guidance for structure, evaluation, and iterative refinement.

### Purpose

`skill-finder` is a discovery-only skill. Given a user's description of what they want to accomplish, it searches the connected skill registry to determine whether a suitable skill already exists. It returns a structured result so a future `skill-builder` skill can compose it with `skill-creator`.

### Composability Requirements

1. **Discovery only.** Never invokes `skill-creator`, never installs, never modifies the registry, never suggests creating a skill.
2. **Three explicit outcomes:** `match_found`, `partial_match`, or `no_match`. No ambiguous states.
3. **Structured output** so an orchestrating skill can parse the result deterministically.

### Available MCP Tools

- `search_skills(query, limit)` — returns ranked candidates with `slug`, `display_name`, `summary`, `latest_version`, `tags`, `updated_at`
- `get_skill_manifest(slug, version)` — returns full SKILL.md content and parsed metadata for a specific version

### Procedural Core

**Step 0: Verify the skill-registry MCP server is available.**
Before doing anything else, confirm the `skill-registry` MCP server is connected and its required tools (`search_skills` and `get_skill_manifest`) are accessible. The simplest reliable check is to attempt a minimal call — for example, `search_skills(query: "ping", limit: 1)` — and observe whether it succeeds, fails with an authentication error, or fails because the tool doesn't exist.

If the MCP server is not connected or the required tools are missing, halt with a clear message and remediation instructions:

> "The skill-registry MCP server is required for skill discovery but is not available. [Specific issue: not connected / authentication failed / tools missing]
>
> Please ensure the skill-registry MCP server is configured and running:
>
> ```
> # Verify the MCP server is registered with your client (Claude Code, Claude Desktop, etc.)
> # Check your MCP client configuration for an entry named 'skill-registry'
>
> # Verify required environment variables are set (example for Artifactory backend):
> echo $ARTIFACTORY_PLATFORM_URL
> echo $ARTIFACTORY_REPOSITORY
> # ARTIFACTORY_ACCESS_TOKEN should be set but not echoed
>
> # Restart your MCP client after configuration changes so the server is loaded.
> ```
>
> After resolving the connection issue, re-run your request."

Do not proceed to any subsequent step if the MCP server is unavailable — without registry access, this skill has no source of truth and cannot return a meaningful result. Returning `no_match` in this state would falsely signal that no matching skill exists when in reality the registry was never queried.

**Step 1: Decompose the request.**
Extract: the capability (action), the domain (what it acts on), constraints (file types, tools, format).

**Step 2: Issue parallel searches.**
Call `search_skills` with up to 5 query variations: literal description, capability alone, domain alone, capability + domain, obvious synonyms. Deduplicate results by `slug`.

**Step 3: Shortlist.**
Pick up to 5 most relevant candidates based on `display_name`, `summary`, `tags`. If none look plausible, go to Step 5 with `no_match`.

**Step 4: Inspect.**
Call `get_skill_manifest` for each shortlisted candidate. Read the full SKILL.md. Classify each as: `match` (fully covers), `partial` (overlaps but needs extension/parameterization), or `reject` (doesn't actually fit).

**Step 5: Report.**
End with one of the structured blocks below.

### Required Output Formats

Every response must end with exactly one structured block:

**Match found (one or more):**
```
=== SKILL_FINDER_RESULT ===
outcome: match_found
matched_skills:
  - slug: <slug>
    version: <latest_version>
    display_name: <display_name>
    explanation: |
      <2-4 sentences explaining why this skill covers the user's need,
      referencing specific parts of its SKILL.md>
  - slug: <slug>
    version: <latest_version>
    display_name: <display_name>
    explanation: |
      <2-4 sentences explaining why this skill also covers the need>
recommendation: |
  <1-2 sentences: if multiple matches, briefly note which is preferred and why,
  or state that they are equivalent. Omit if single match.>
=== END ===
```

**Partial match:**
```
=== SKILL_FINDER_RESULT ===
outcome: partial_match
candidates:
  - slug: <slug>
    version: <latest_version>
    display_name: <display_name>
    coverage: <"extends" | "parameterizes" | "adjacent">
    gap: |
      <1-2 sentences on what the candidate doesn't cover>
explanation: |
  <2-4 sentences summarizing the partial fit>
=== END ===
```

**No match:**
```
=== SKILL_FINDER_RESULT ===
outcome: no_match
searched_queries:
  - <query 1>
  - <query 2>
candidates_inspected: <count>
explanation: |
  <2-3 sentences on what was searched and why nothing fit>
=== END ===
```

Above the structured block, include a brief human-readable summary (1-3 sentences) so direct users get a readable response.

### What the Skill Must NOT Do

- Do not skip the MCP server precondition check — verify connectivity before any search
- Do not return `no_match` when the MCP server is unavailable — surface the connectivity issue instead
- Do not invoke `skill-creator` or suggest creating a skill
- Do not install or download skills
- Do not return `match_found` if the SKILL.md doesn't clearly cover the request — prefer `partial_match` when uncertain
- Do not return `no_match` without inspecting top candidates if search returned plausible results

### Edge Cases

- **MCP server unavailable at start (precondition fails):** Halt with install/configuration instructions per Step 0. Do not proceed to search.
- **MCP server fails mid-flow:** If a search or manifest fetch fails after the precondition passed, surface the error explicitly and do not return `no_match`.
- **Empty registry:** Return `no_match` with `candidates_inspected: 0`.
- **Vague user request:** Ask one clarifying question before searching.

### Evaluation Set

Build evals covering:

1. **MCP server unavailable → halt with instructions:** server not connected; assert `skill-finder` halts before searching and surfaces configuration instructions
2. **MCP server connected → proceed:** server is connected; assert `skill-finder` proceeds to decomposition and search
3. **Clear match** — assert correct slug returned
4. **Clear no-match** — niche request with nothing in registry
5. **Partial match** — overlap but not full coverage
6. **Vague request** — assert clarifying question
7. **Unusual phrasing** — decomposition surfaces the right matches
8. **MCP error mid-flow** — server fails on a search call after precondition passed; assert error surfaced, not silent `no_match`
9. **Format compliance** — structured block well-formed

Iterate against the eval set until consistently passing.

### Skill Description (for SKILL.md frontmatter)

Tune during iteration. Starting point:

> Use this skill when the user wants to find or discover an existing skill that performs a specific capability, before creating a new one. Triggers include "is there a skill that...", "do we have a skill for...", "find me a skill that...", or any request to check the registry for matching skills. Searches the registry, inspects candidates, returns a structured result indicating match, partial match, or no match. Does not create or install skills.

### Composition Note

Include in SKILL.md and README: `skill-finder` is designed to be composed inside a future `skill-builder` skill that calls `skill-finder` first and `skill-creator` second when no match is found. `skill-finder` itself stays out of that decision.

### Deliverables

- `SKILL.md` with procedural core, output format spec, clear triggers
- Eval set covering all cases above
- Brief README with installation steps

Begin by reading `skill-creator`'s SKILL.md, then propose your implementation plan before writing the skill. Wait for confirmation before implementing.

---

**End of prompt.**
