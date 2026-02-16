# Technical Documentation: Documentation Agent Orchestrator

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [How It Works](#how-it-works)
4. [Core Components](#core-components)
5. [Command Details](#command-details)
6. [Governance System](#governance-system)
7. [Prompt Generation Logic](#prompt-generation-logic)
8. [Workflow Details](#workflow-details)
9. [Extension API](#extension-api)

---

## Overview

**Documentation Agent Orchestrator** is a VS Code extension that acts as a governance layer between raw content and AI-generated documentation. It enforces strict rules to prevent AI hallucination while enabling AI to create structured documentation from rough notes, code comments, or incomplete specifications.

### Key Principles

- **Correctness over fluency** - Accurate documentation is prioritized over well-written documentation
- **Preserve ambiguity** - Vague source content should produce vague but accurate documentation
- **No invention** - AI cannot add features, steps, or details not present in source material
- **Explicit uncertainty** - Ambiguous elements are explicitly documented in "Preserved Ambiguities" sections
- **Question-driven clarification** - AI asks questions only when documentation generation is blocked

---

## Architecture

### High-Level Design

```
┌─────────────────────┐
│   User Content      │ (Rough notes, code comments, specs)
│   (VS Code Editor)  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Extension Layer    │ (extension.ts)
│  - Command handlers │
│  - State management │
│  - UI interactions  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   Engine Layer      │
│  - promptGenerator  │ (Core logic)
│  - governance       │ (Rules)
│  - types            │ (Type system)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Generated Prompt   │ (Markdown document)
│  + Clipboard copy   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  External AI        │ (Claude, ChatGPT, etc.)
│  (User pastes)      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  AI Response        │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Diff Preview       │ (VS Code diff viewer)
│  Original vs New    │
└─────────────────────┘
```

### Project Structure

```
doc-agent-orchestrator/
├── src/
│   ├── extension.ts           # Entry point, command registration
│   └── engine/
│       ├── governance.ts      # Governance rules definition
│       ├── promptGenerator.ts # Core prompt generation logic
│       └── types.ts          # TypeScript type definitions
├── out/                       # Compiled JavaScript (build output)
├── DEMO/                      # Demo examples and guides
├── TESTING/                   # Test cases (25 scenarios)
├── package.json              # Extension manifest
├── tsconfig.json             # TypeScript configuration
└── README.md                 # User-facing documentation
```

---

## How It Works

### End-to-End Flow

#### Phase 1: Prompt Generation

1. **User selects content** in VS Code (or uses entire file)
2. **User triggers command**: "Generate Documentation Prompt"
3. **Extension captures**:
   - Selected text or full document content
   - Document language ID
   - Original text for later diff comparison
4. **User selects documentation type** from quick pick menu (7 options)
5. **User provides intent** via input box (e.g., "Document login process")
6. **Engine generates governed prompt**:
   - Applies governance rules
   - Injects task-specific output structure
   - Includes source content
   - Adds missing information policy
7. **Extension displays prompt**:
   - Opens in new editor (split view)
   - Automatically copies to clipboard
   - Shows success notification

#### Phase 2: AI Interaction (External)

1. **User pastes prompt** into AI assistant (Claude, ChatGPT, etc.)
2. **AI generates response** following governance rules
3. **User receives**:
    - Structured documentation, OR
    - Clarifying questions if generation is blocked

#### Phase 3: Clarification Loop (Optional)

4. **If AI asks questions**:

    - User runs "Provide Clarifications and Regenerate Prompt"
    - User enters answers to AI's questions
    - Extension regenerates prompt with clarifications section
    - Prompt includes: "Clarifications have been provided - treat as authoritative facts"
    - User pastes updated prompt into AI
    - AI generates documentation using clarifications

#### Phase 4: Diff Preview

1. **User triggers**: "Preview Documentation Rewrite Diff"
2. **Extension retrieves AI response** from:
    - Current editor content (if available), OR
    - Clipboard content
3. **Extension opens diff view**:
    - Left panel: Original content
    - Right panel: AI-generated content
    - Side-by-side comparison with inline changes highlighted
4. **User reviews**:
    - Check "Preserved Ambiguities" section
    - Verify no invented features
    - Validate terminology consistency
    - Review "Governance Notes" if present (indicates violations)

---

## Core Components

### 1. Extension Entry Point (`src/extension.ts`)

**Responsibilities:**

- Register three commands with VS Code
- Manage state between commands (context persistence)
- Handle user input and validation
- Coordinate with prompt generation engine
- Manage clipboard operations
- Open diff views

**Key State Variables:**

```typescript
// Stores original content for diff comparison
let lastRewriteContext: {
  originalText: string;
  languageId: string;
} | null = null;

// Stores prompt context for clarification regeneration
let lastPromptContext: {
  taskType: TaskType;
  userIntent: string;
  context: string;
} | null = null;
```

**Activation:**

- Extension activates when VS Code loads (no specific activation events)
- Commands are immediately available in command palette

### 2. Governance Rules (`src/engine/governance.ts`)

**Purpose:** Define the immutable rules that govern all AI behavior

**Rules:**

```typescript
export const GOVERNANCE_RULES = `
GOVERNANCE RULES:
- Do not invent features, permissions, or behavior that are not stated or implied
- Do not change established terminology from the source
- Do not add steps that require inventing information
- Preserve ambiguity when the source is vague - document it as-is
- Do not rewrite for style or elegance
`;
```

**Why these rules:**

- **No invention**: Prevents AI from adding features that don't exist
- **No terminology changes**: Preserves domain-specific language
- **No invented steps**: Ensures procedures reflect actual behavior
- **Preserve ambiguity**: Documents uncertainty instead of guessing
- **No style rewriting**: Focuses on correctness, not elegance

### 3. Prompt Generator (`src/engine/promptGenerator.ts`)

**Purpose:** Core engine that constructs governance-driven prompts

**Key Function:**

```typescript
export function generatePrompt(input: PromptInput): string
```

**Input Interface:**

```typescript
export interface PromptInput {
  taskType: TaskType;        // Documentation type
  userIntent: string;         // User's description
  context: string;            // Source content
  clarifications?: string;    // Optional answers to AI questions
}
```

**Prompt Structure:**

Every generated prompt contains:

1. **System Instructions**
   - Role definition: "You are a Technical Documentation Agent"
   - Governance rules (from governance.ts)
   - Rewrite policy

2. **Missing Information Policy** (conditional)
   - **Without clarifications**: Instructions on when to ask questions vs. preserve ambiguity
   - **With clarifications**: Instructions to treat clarifications as authoritative

3. **User Intent**
   - The description user provided (e.g., "Document the login process")

4. **Source Content**
   - The actual text selected/opened by user
   - Marked as "authoritative"

5. **Clarifications Section** (optional)
   - Present only when regenerating after AI asked questions
   - Marked as "authoritative - treat as facts"

6. **Task-Specific Output Structure**
   - Defined by documentation type
   - Includes mandatory headings
   - Always includes "Preserved Ambiguities" and "Governance Notes" sections

**Missing Information Handling Logic:**

The extension uses sophisticated logic to determine when AI should ask questions:

**Ask questions when:**

- Documentation requires inventing specific steps or features not mentioned
- There are undefined references (e.g., "standard workflow", "usual process")
- Logical gaps make the task impossible to complete factually

**DO NOT ask questions about:**

- Vague but valid descriptions (e.g., "extra options", "show error")
- Missing context that doesn't block the core task
- Details that can be documented with preserved ambiguity

This creates a "smart ambiguity tolerance" - AI can proceed with vague inputs as long as it documents the vagueness explicitly.

### 4. Type System (`src/engine/types.ts`)

**Supported Documentation Types:**

```typescript
export type TaskType =
  | "procedure"           // Step-by-step instructions
  | "concept"            // Technical explanations
  | "troubleshooting"    // Problem-solving guides
  | "reference"          // API/feature reference docs
  | "tutorial"           // Learning-focused guides
  | "release-notes"      // Version change documentation
  | "api-documentation"; // API endpoint documentation
```

---

## Command Details

### Command 1: `docAgent.generateDocumentation`

**Display Name:** "Generate Documentation Prompt"

**Purpose:** Generate a governance-driven AI prompt from selected content

**Trigger:** Command Palette → "Generate Documentation Prompt"

**Workflow:**

1. Check if editor is active
2. Get selected text (or full document if no selection)
3. Validate content exists
4. Store original content for later diff
5. Show quick pick menu with 7 documentation types
6. Show input box for user intent
7. Generate prompt using engine
8. Store prompt context for clarifications
9. Copy prompt to clipboard
10. Open prompt in new editor (split view)
11. Show success notification

**Error Handling:**

- No active editor → Error: "No active editor found"
- Empty content → Error: "No content found"
- User cancels type selection → Silent exit
- User cancels intent input → Silent exit
- Generation fails → Error with exception message

**User Experience:**

- Prompt opens beside current editor
- Clipboard is pre-populated for easy pasting
- Original editor remains open for reference

### Command 2: `docAgent.previewRewriteDiff`

**Display Name:** "Preview Documentation Rewrite Diff"

**Purpose:** Compare original content with AI-generated documentation

**Trigger:** Command Palette → "Preview Documentation Rewrite Diff"

**Prerequisites:**

- User must have previously run "Generate Documentation Prompt"
- AI response must be available (in editor or clipboard)

**Workflow:**

1. Check if rewrite context exists
2. Attempt to get AI response from multiple sources (priority order):
   - Current active editor (saved file with content)
   - Current active editor (untitled document with content)
   - Clipboard content
3. Validate AI response exists
4. Create temporary document with original content
5. Create temporary document with AI response
6. Open VS Code diff view (side-by-side)
7. Title diff: "Documentation Rewrite Preview"

**Error Handling:**

- No prior prompt generation → Error: "No rewrite context found. Please run 'Generate Documentation Prompt' first."
- Empty clipboard and no editor content → Error: "Clipboard is empty. Please copy the AI response first."
- Diff command fails → Error with exception message

**User Experience:**

- Original content on left (marked as read-only)
- AI response on right (marked as read-only)
- Changes highlighted with VS Code's diff colors
- Can scroll both panels synchronously
- User decides whether to accept, reject, or modify

### Command 3: `docAgent.provideClarifications`

**Display Name:** "Provide Clarifications and Regenerate Prompt"

**Purpose:** Answer AI questions and regenerate prompt with clarifications

**Trigger:** Command Palette → "Provide Clarifications and Regenerate Prompt"

**Prerequisites:**

- User must have previously run "Generate Documentation Prompt"
- AI must have asked clarifying questions

**Workflow:**

1. Check if prompt context exists
2. Show input box for clarifications
3. Validate clarifications are provided
4. Regenerate prompt with same context + clarifications
5. Copy new prompt to clipboard
6. Open new prompt in editor (split view)
7. Show success notification

**Clarifications Integration:**

The regenerated prompt includes:
```
CLARIFICATIONS (authoritative - treat as facts):
[User's answers to AI questions]
```

And modified missing information policy:
```
MISSING INFORMATION HANDLING:
- Clarifications have been provided below that resolve missing information
- Use the clarifications as authoritative facts
- Only ask about information NOT covered by the clarifications
- If clarifications resolve all gaps, proceed with generation
```

**Error Handling:**

- No prior prompt generation → Error: "No prompt context found"
- Empty clarifications → Validation error in input box
- Regeneration fails → Error with exception message

---

## Governance System

### Philosophy

The governance system is the core differentiator of this extension. It transforms AI from a "helpful assistant that often lies" into a "constrained agent that preserves truth."

### Enforcement Mechanism

Governance is enforced through **prompt engineering**, not through technical controls. The extension:

1. Injects governance rules into every prompt
2. Instructs AI to continue even if it violates rules
3. Requires AI to document violations in "Governance Notes" section

This creates **observable governance** - violations are visible, not prevented.

### Rule Breakdown

#### Rule 1: No Feature Invention

**What it prevents:**

- Adding features that don't exist
- Implying capabilities not mentioned
- Extrapolating from similar products

**Example:**

- ❌ Bad: "User clicks login, enters credentials, or uses SSO"
- ✅ Good: "User clicks login and enters credentials" (if SSO not mentioned)

#### Rule 2: No Terminology Changes

**What it prevents:**

- Replacing domain-specific terms with "clearer" alternatives
- Normalizing inconsistent terminology
- Using industry-standard terms when source uses different terms

**Example:**

- Source says: "refresh token rotation"
- ❌ Bad: AI changes to "token refresh mechanism"
- ✅ Good: AI keeps "refresh token rotation"

#### Rule 3: No Invented Steps

**What it prevents:**

- Adding steps that "should" exist
- Filling logical gaps with assumptions
- Borrowing steps from similar processes

**Example:**

- Source: "User clicks submit. Data is validated."
- ❌ Bad: "User fills form, clicks submit, data is validated"
- ✅ Good: Document as-is (or ask: "What happens before submit?")

#### Rule 4: Preserve Ambiguity

**What it prevents:**

- Resolving vague language with guesses
- Converting "some", "various", "extra" into specifics
- Making absolute statements from relative ones

**Example:**

- Source: "Admin users get extra features"
- ❌ Bad: "Admin users can manage users and view reports"
- ✅ Good: "Admin users get extra features" + note in Preserved Ambiguities

#### Rule 5: No Style Rewriting

**What it prevents:**

- Polishing rough language
- Improving sentence structure when meaning is clear
- Making documentation "sound better"

**Example:**

- Source: "Thing happens. Then other thing happens."
- ❌ Bad: "When the first event occurs, it triggers a subsequent event."
- ✅ Good: Reorder for clarity if needed, but keep informal tone

### Preserved Ambiguities Section

Every output includes a **Preserved Ambiguities** section that lists:

- Vague terms documented as-is
- Undefined references
- Implicit assumptions
- Missing details

**Purpose:**

- Makes uncertainty visible to reviewers
- Helps identify gaps in source material
- Distinguishes "incomplete docs" from "complete docs of incomplete specs"

**Example:**

```markdown
## Preserved Ambiguities
- "Extra features" for admin users - not specified in source
- "Standard workflow" - undefined reference
- Error handling behavior - not mentioned in source
```

### Governance Notes Section

Only included when AI violates governance rules.

**Purpose:**

- Documents where AI couldn't follow rules
- Helps improve source content or prompts
- Maintains transparency about AI limitations

**Example:**

```markdown
## Governance Notes
- Added "typically" when describing behavior to avoid stating certainty
- Reordered steps for logical flow (source order was unclear)
```

---

## Prompt Generation Logic

### Task-Specific Output Structures

Each documentation type has a predefined structure:

#### 1. Procedure

```markdown
Prerequisites
Procedure
Notes
Preserved Ambiguities (only if applicable)
Governance Notes (only if applicable)
```

**Use case:** Step-by-step user instructions, deployment guides, setup procedures

#### 2. Concept

```markdown
Overview
Key Components
Process Flow
Important Considerations
Preserved Ambiguities (only if applicable)
Governance Notes (only if applicable)
```

**Use case:** Architectural explanations, system designs, technical concepts

#### 3. Troubleshooting

```markdown
Symptoms
Possible Causes
Verification Steps
Resolution
Prevention
Preserved Ambiguities (only if applicable)
Governance Notes (only if applicable)
```

**Use case:** Problem-solving guides, error resolution, diagnostic procedures

#### 4. Reference

```markdown
Description
Parameters / Options
Examples
Related Information
Preserved Ambiguities (only if applicable)
Governance Notes (only if applicable)
```

**Use case:** Configuration references, command references, feature catalogs

#### 5. Tutorial

```markdown
Objective
Prerequisites
Steps
Verification
Next Steps
Preserved Ambiguities (only if applicable)
Governance Notes (only if applicable)
```

**Use case:** Learning-focused guides, getting started guides, hands-on tutorials

#### 6. Release Notes

```markdown
New Features
Improvements
Bug Fixes
Breaking Changes (if applicable)
Known Issues (if applicable)
Preserved Ambiguities (only if applicable)
Governance Notes (only if applicable)
```

**Use case:** Version releases, changelog entries, update announcements

#### 7. API Documentation

```markdown
Endpoint / Function
Description
Request / Parameters
Response / Return Value
Examples
Error Handling
Preserved Ambiguities (only if applicable)
Governance Notes (only if applicable)
```

**Use case:** REST API docs, function documentation, SDK references

### Dynamic Sections

Some sections are **conditional**:

- **Preserved Ambiguities**: Only if source content has vague elements
- **Governance Notes**: Only if AI violated rules
- **Breaking Changes**: Only for release notes with breaking changes
- **Known Issues**: Only for release notes with known issues

This prevents empty sections in output.

---

## Workflow Details

### Workflow 1: Simple Documentation Generation

**Scenario:** User has complete, clear source material

1. Select content in VS Code
2. Run "Generate Documentation Prompt"
3. Choose documentation type (e.g., "Procedure")
4. Describe intent: "Document the deployment process"
5. Paste prompt into Claude/ChatGPT
6. Receive structured documentation
7. Run "Preview Documentation Rewrite Diff"
8. Review changes and preserved ambiguities
9. Accept or modify as needed

**Timeline:** 2-3 minutes

### Workflow 2: Documentation with Clarifications

**Scenario:** Source material has critical gaps

1. Select incomplete content
2. Run "Generate Documentation Prompt"
3. Choose documentation type
4. Describe intent
5. Paste prompt into AI
6. **AI responds with questions** (e.g., "What happens after validation fails?")
7. Run "Provide Clarifications and Regenerate Prompt"
8. Enter answers to AI questions
9. Paste regenerated prompt into AI
10. Receive complete documentation
11. Preview diff and review

**Timeline:** 5-7 minutes

### Workflow 3: Iterative Refinement

**Scenario:** Multiple rounds of clarification needed

1. Generate initial prompt → AI asks questions (batch 1)
2. Provide clarifications → AI asks more questions (batch 2)
3. Provide more clarifications → AI generates documentation
4. Preview and review

**Timeline:** 10-15 minutes

**Note:** Extension stores prompt context throughout, so each regeneration preserves:

- Task type
- User intent
- Original source content
- All previous clarifications (cumulative)

---

## Extension API

### Exported Activation Function

```typescript
export function activate(context: vscode.ExtensionContext): void
```

**Registers:**

- `docAgent.generateDocumentation`
- `docAgent.previewRewriteDiff`
- `docAgent.provideClarifications`

### Exported Deactivation Function

```typescript
export function deactivate(): void
```

**Purpose:** Cleanup (currently no-op)

### Dependencies

**Runtime Dependencies:**

```json
{
  "vscode": "^1.104.0"  // VS Code Extension API
}
```

**Development Dependencies:**

```json
{
  "@types/vscode": "^1.104.0",
  "@types/node": "^22.10.2",
  "typescript": "^5.7.2",
  "esbuild": "^0.24.2"
}
```

### Build System

- **Compiler:** TypeScript 5.7.2
- **Bundler:** esbuild 0.24.2
- **Target:** ES2022
- **Module:** CommonJS
- **Output:** `out/extension.js` (single file bundle)

**Build Commands:**

```bash
npm run compile    # Compile TypeScript
npm run watch      # Watch mode for development
npm run package    # Create .vsix package
npm run publish    # Publish to marketplace
```

### Configuration

**Extension Manifest (package.json):**

- **Name:** `doc-agent-orchestrator`
- **Display Name:** "Documentation Agent Orchestrator"
- **Version:** 1.0.0
- **Publisher:** TharunSebastian
- **License:** MIT
- **Activation:** On VS Code startup (no specific events)

**No User Settings:**

- Extension has no configurable settings
- All behavior is hardcoded for consistency
- Governance rules are immutable

---

## Advanced Topics

### State Management

The extension uses **module-level variables** for state persistence:

```typescript
let lastRewriteContext: {
  originalText: string;
  languageId: string;
} | null = null;
```

**Lifecycle:**

- Set when "Generate Documentation Prompt" runs
- Read when "Preview Documentation Rewrite Diff" runs
- Persists until VS Code closes
- Lost on window reload

```typescript
let lastPromptContext: {
  taskType: TaskType;
  userIntent: string;
  context: string;
} | null = null;
```

**Lifecycle:**

- Set when "Generate Documentation Prompt" runs
- Read when "Provide Clarifications and Regenerate Prompt" runs
- Updated with each regeneration
- Lost on window reload

**Design Decision:** No persistent storage (workspace state, global state) because:

- State is short-lived (minutes, not hours)
- Re-running generation is fast if context is lost
- Simpler implementation
- No cleanup needed

### Clipboard Integration

The extension uses VS Code's clipboard API:

```typescript
await vscode.env.clipboard.writeText(prompt);  // Write
const text = await vscode.env.clipboard.readText();  // Read
```

**Write Strategy:**

- Automatic copy on prompt generation
- User doesn't need to manually copy
- Can still review in editor before pasting

**Read Strategy:**

- Fallback for diff preview
- Used when AI response not in editor
- No parsing or validation (trusts clipboard content)

### Diff View

Uses VS Code's built-in diff command:

```typescript
await vscode.commands.executeCommand(
  "vscode.diff",
  originalDoc.uri,
  rewrittenDoc.uri,
  "Documentation Rewrite Preview"
);
```

**Parameters:**

- `originalDoc.uri`: Left side (original content)
- `rewrittenDoc.uri`: Right side (AI response)
- `"Documentation Rewrite Preview"`: Tab title

**Features Inherited from VS Code:**

- Syntax highlighting
- Inline diff markers
- Synchronized scrolling
- Jump to next/previous change
- No save capability (read-only preview)

### Error Handling Philosophy

**User Errors (Expected):**

- Shown via `vscode.window.showErrorMessage()`
- Clear, actionable messages
- No stack traces

**System Errors (Unexpected):**

- Shown via `vscode.window.showErrorMessage()` with exception message
- Logged to console: `console.error()`
- Includes full error object for debugging

**Silent Failures:**

- User cancels operation → No notification
- Optional inputs skipped → No warning

### Extension Security

**No Network Calls:**

- Extension doesn't communicate with external services
- AI interaction is manual (copy/paste)
- No API keys or credentials needed

**No File System Writes:**

- Except when user explicitly saves documents
- No automatic file creation or modification
- Generated prompts are temporary documents

**No Telemetry:**

- No usage tracking
- No error reporting to external services
- No analytics

**Content Privacy:**

- Source content stays local
- Only leaves VS Code when user manually pastes into AI
- No logging of sensitive content

---

## Performance Characteristics

### Prompt Generation

- **Time:** < 50ms (instant)
- **Memory:** Minimal (single string concatenation)
- **CPU:** Negligible

### Diff Preview

- **Time:** < 200ms (dominated by VS Code rendering)
- **Memory:** Holds two copies of content in memory
- **CPU:** VS Code's diff algorithm (optimized)

### Large Documents

- **Tested up to:** 10,000 lines
- **Performance:** Remains instant for generation
- **Limitation:** AI context window, not extension

---

## Testing

### Test Suite Location

`TESTING/` directory contains:

- 25 test input files
- 15+ test response files
- Test report with findings

### Test Categories

1. **Complete input** - All necessary information provided
2. **Incomplete input** - Missing critical details
3. **Ambiguous input** - Vague but documentable content
4. **Edge cases** - Unusual content types

### Manual Testing Workflow

1. Open test input file
2. Run "Generate Documentation Prompt"
3. Paste into AI (Claude recommended)
4. Compare AI response with expected behavior:
   - Did AI invent features? (Should not)
   - Did AI preserve ambiguities? (Should)
   - Did AI ask questions when needed? (Should)
   - Did AI follow output structure? (Should)

### No Automated Tests

- Extension testing would require VS Code test environment
- Core logic (prompt generation) is deterministic
- AI behavior testing requires external services
- Manual testing is sufficient for current scope

---

## Limitations and Known Issues

### Current Limitations

1. **Context lost on reload**
   - Rewrite context doesn't persist across VS Code restarts
   - User must regenerate prompt after reload

2. **No multi-document support**
   - Can only generate from one document at a time
   - No aggregation of multiple files

3. **No prompt templates**
   - Output structures are fixed
   - Cannot customize governance rules per user

4. **Manual AI interaction**
   - No direct AI integration
   - Requires copy/paste workflow

5. **No response validation**
   - Extension doesn't parse AI responses
   - Cannot verify AI followed governance rules
   - User must manually review

### Design Limitations (Intentional)

1. **No AI API integration**
   - Keeps extension AI-agnostic
   - Works with any AI assistant
   - No API costs or rate limits

2. **No automatic acceptance**
   - Forces human review
   - Maintains accountability

3. **No style improvements**
   - Deliberately excluded
   - Focuses on correctness only

### Future Considerations

**Possible Enhancements:**

- Persistent context storage
- Custom governance rule templates
- Multi-file aggregation
- AI response parsing and validation
- Direct AI integration (optional)
- Workspace-level configuration

**Not Planned:**

- Style/grammar improvements
- Automatic documentation
- AI model training
- Content generation without source

---

## Troubleshooting

### Problem: "No active editor found"

**Cause:** No editor window is open or focused
**Solution:** Open a file before running the command

### Problem: "No rewrite context found"

**Cause:** Haven't run "Generate Documentation Prompt" yet
**Solution:** Generate prompt first, then preview diff

### Problem: Clipboard is empty for diff preview

**Cause:** AI response not copied or lost
**Solution:**

1. Paste AI response into a new VS Code file
2. Run diff preview (will use editor content instead)

### Problem: Prompt regeneration fails

**Cause:** Context was lost (VS Code reload)
**Solution:** Re-run "Generate Documentation Prompt" from beginning

### Problem: AI doesn't follow governance rules

**Cause:** AI model doesn't respect instructions
**Solution:**

1. Try different AI model (Claude > GPT-4 > GPT-3.5)
2. Regenerate and paste prompt again
3. Report in "Governance Notes" section if AI documents violations

### Problem: Extension commands not visible

**Cause:** Extension not activated or installed
**Solution:**

1. Check Extensions panel: "Documentation Agent Orchestrator" installed
2. Reload VS Code
3. Run command from Command Palette (Ctrl+Shift+P)

---

## Contributing and Development

### Development Setup

1. **Clone repository:**

   ```bash
   git clone https://github.com/Tharun135/doc-agent-orchestrator
   cd doc-agent-orchestrator
   ```

2. **Install dependencies:**

   ```bash
   npm install
   ```

3. **Compile TypeScript:**

   ```bash
   npm run compile
   ```

4. **Run in Extension Development Host:**
   - Press F5 in VS Code
   - New VS Code window opens with extension loaded

5. **Make changes:**
   - Edit files in `src/`
   - Run `npm run watch` for auto-compilation
   - Reload extension host (Ctrl+R in dev window)

### Code Style

- **Language:** TypeScript 5.7
- **Formatting:** None enforced (use default TypeScript style)
- **Linting:** ESLint configured (see `eslint.config.mjs`)

### Adding New Documentation Types

1. **Add type to `src/engine/types.ts`:**

   ```typescript
   export type TaskType =
     | "procedure"
     | "concept"
     // ... existing types
     | "your-new-type";
   ```

2. **Add to quick pick in `src/extension.ts`:**

   ```typescript
   { label: "Your New Type", value: "your-new-type" }
   ```

3. **Create output spec function in `src/engine/promptGenerator.ts`:**

   ```typescript
   function yourNewTypeOutputSpec(): string {
     return `
   TASK:
   Rewrite the source content into [your type].
   
   OUTPUT STRUCTURE (use exactly these headings):
   Heading 1
   Heading 2
   ...
   `;
   }
   ```

4. **Add case to switch statement:**

   ```typescript
   case "your-new-type":
     return base + yourNewTypeOutputSpec();
   ```

### Publishing

```bash
npm run package  # Creates .vsix file
npm run publish  # Publishes to marketplace (requires PAT)
```

**Requirements:**

- VS Code Extension Manager (vsce) installed
- Publisher account on VS Code Marketplace
- Personal Access Token (PAT) from Azure DevOps

---

## Appendix

### Full Prompt Example

**Input:**

- Task type: Procedure
- User intent: "Document the login process"
- Source content: "User enters username and password, clicks submit. System validates credentials. Redirect to dashboard on success."

**Generated Prompt:**

```markdown
SYSTEM:
You are a Technical Documentation Agent.

GOVERNANCE RULES:
- Do not invent features, permissions, or behavior that are not stated or implied
- Do not change established terminology from the source
- Do not add steps that require inventing information
- Preserve ambiguity when the source is vague - document it as-is
- Do not rewrite for style or elegance

REWRITE POLICY:
- Rewrite the source content into structured documentation
- Preserve original meaning and intent
- Reorder or clarify only when necessary for correctness

MISSING INFORMATION HANDLING:
- Ask questions ONLY when documentation cannot be generated without inventing behavior
- Preserve ambiguity from the source content - document vague details as-is
- Stop and ask questions if:
  • The task requires inventing specific steps or features not mentioned
  • There are undefined references (e.g., "standard workflow", "usual process")
  • Logical gaps make the task impossible to complete factually
- Do NOT ask questions about:
  • Vague but valid descriptions (e.g., "extra options", "show error")
  • Missing context that doesn't block the core task
  • Details that can be documented with preserved ambiguity

GOVERNANCE ENFORCEMENT:
- If any governance rule is violated:
  - Continue generation
  - Add a "Governance Notes" section listing the issues

PRESERVED AMBIGUITIES:
- When source content is vague but documentable, explicitly list what was preserved
- Add a "Preserved Ambiguities" section noting any vague terms documented as-is
- This helps reviewers understand intentional vagueness vs. missing research

USER INTENT:
Document the login process

SOURCE CONTENT (authoritative):
User enters username and password, clicks submit. System validates credentials. Redirect to dashboard on success.

TASK:
Rewrite the source content into a user-facing procedure.

OUTPUT STRUCTURE (use exactly these headings):
Prerequisites
Procedure
Notes
Preserved Ambiguities (only if applicable)
Governance Notes (only if applicable)
```

### Example AI Response

```markdown
# User Login

## Prerequisites
- Valid username and password credentials

## Procedure
1. Enter your username in the username field
2. Enter your password in the password field
3. Click the submit button
4. System validates your credentials
5. If credentials are valid, you are redirected to the dashboard

## Notes
- Invalid credentials are not described in the source material

## Preserved Ambiguities
- "Dashboard" destination - no details provided about dashboard content or URL
- Error handling for invalid credentials - behavior not specified in source
- Form field labels - assumed "username" and "password" based on source terminology
```

---

## Summary

**Documentation Agent Orchestrator** is a VS Code extension that bridges the gap between AI capability and documentation trustworthiness. It works by:

1. **Injecting governance rules** into AI prompts
2. **Structuring output** with task-specific templates
3. **Preserving ambiguity** instead of inventing details
4. **Enabling clarification loops** when needed
5. **Providing diff previews** for validation

The extension doesn't try to make AI smarter - it makes AI behavior **constrained, observable, and defensible**.

**Core insight:** AI is fluent but unreliable. By making AI preserve ambiguity and document uncertainty, the extension transforms it into a tool for creating accurate documentation from incomplete specifications.
