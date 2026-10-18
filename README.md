# Source as Spec (SaS)

**Source as Spec** is a method for developing software with coding agents, where
**work requests live in source code**. Developers express intent through
declarations, documentation, tests and explicit agent work markers, supplemented
by specification comments when useful. Developers can implement code themselves
or delegate selected parts to an agent, review the changes and continue
iterating.

The method is **language-independent**. Rust will serve as the initial example.




## A note on this approach

I'm a regular software engineer exploring a more direct way to work with coding
agents. **Source as Spec** grew out of problems I encounter in everyday
development.

I've seen vibe-coded applications become harder to maintain as they grow. When I
delegate too much through chat, I start losing track of how the code works. I
also worry about losing coding skills when I'm no longer actively digging into
code myself, which makes reviewing generated code harder. This may sound
familiar.

With SaS, you define types, structure and expectations directly in source,
instead of describing them in prose and correcting generated code through
repeated prompts. You write and edit code yourself, delegating selected
implementation work to the agent.

You may already be thinking along similar lines. The idea feels like it's in the
air. I wanted to write it down and share one possible convention for working
this way.

SaS isn't a new invention. It adapts techniques we've used for decades, such as
TDD, specifications and TODO comments, to today’s software development. It
doesn't promise perfect software. Use it as it is, adapt it or take only the
parts that help. The approach matters more than adopting SaS itself.

SaS does not rule out chat. Chat can help explore ideas, scaffold a project, set
up an environment or generate a quick script. SaS is meant to **support**
**normal programming, not replace it**.




## The method

It consists of three parts:

1. **Writing the source** — How to prepare source code and tests.
2. **Spec language** — Instruction markers describing the requested action,
   expected code behavior and restrictions on the agent.
3. **Working with agents** — How source instructions are assembled into a
   request and executed.




## The source

Write ordinary source code. Document what it does and express expected behavior
in tests.

### 1. Write and document the code

Define your functions, types and interfaces as usual. Use **normal documentation
comments** to explain their purpose and use to other developers. Write the same
documentation you would without an agent. Leave unfinished bodies in a form that
the language accepts so tests can run. For example:

```rust
/// Decodes frames prefixed by a four-byte big-endian length.
struct FrameDecoder {
    buffer: Vec<u8>,
    max_frame_len: usize,
}

impl FrameDecoder {
    /// Accepts bytes and returns complete frames.
    fn push(&mut self, chunk: &[u8]) -> Result<Vec<Vec<u8>>, DecodeError> {
        // The implement! macro is explained in the Action section below.
        implement!()
    }

    /// Checks that the stream ended at a frame boundary.
    fn finish(&self) -> Result<(), DecodeError> {
        implement!()
    }
}

#[derive(Debug)]
enum DecodeError {
    FrameTooLarge,
    TruncatedFrame,
}
```

### 2. Cover the behavior with tests

Write the expected results **before the implementation**, including relevant
edge cases. For example:

```rust
#[test]
fn decodes_a_frame_across_chunks() {
    let mut decoder = FrameDecoder::new(1024);
    let frame = encode_frame(b"hello");

    assert!(decoder.push(&frame[..2]).unwrap().is_empty());
    assert!(decoder.push(&frame[2..6]).unwrap().is_empty());
    assert_eq!(
        decoder.push(&frame[6..]).unwrap(),
        vec![b"hello".to_vec()]
    );
    assert!(decoder.finish().is_ok());
}
```

Run the tests before implementation to confirm they fail. After implementation,
rerun them and review the code.

The same approach can cover a **complete module or feature**: define its types
and interfaces, sketch the structure and write tests showing how the parts
should work together. The agent can then implement the selected parts in one
execution.




## The spec

Declarations, documentation and tests already express much of the intent. A
small **specification language in source comments** adds a concise way to
describe behavior, request changes or reviews, and set implementation
permissions without writing every detail as code.

This section defines the features and marker meanings an in-source specification
should provide. A notation inside comments in Rust source files is proposed as
*one example* of those features.

### Instruction markers

The spec uses **markers** to identify the kind of instruction. A marker may be
followed by instructions in plain language or pseudocode, on one or more lines.
It can also stand alone when no additional text is needed.

The spec contains three kinds of instructions for the agent:

1. **Action** — What the agent should do.
2. **Behavior** — What the code should do.
3. **Freedom** — What the agent is allowed to do.

For Rust source files, a common notation is proposed for all markers. Add a
colon after the marker name when instructions follow it. Use `//@` for a single
line, and consecutive `//@` comments or `/*@ ... */` for multiple lines:

```rust
//@ <marker>
//@ <marker>: <instructions>

//@ <marker>: <first line of instructions>
//@   <next line of instructions>

/*@ <marker>:
<first line of instructions>
<next line of instructions>
*/
```

In consecutive line comments, **extra indentation** after `//@` continues the
preceding instruction. A marker at the **normal indentation** starts a new
instruction. Continuation lines do not need to align with the first line's text.
Both multiline forms have the same meaning.

Place each marked comment before its target in the source. Several markers can
apply to the same target. The examples below use this proposed Rust notation.

Only an **Action** requests work. **Behavior** and **Freedom** instructions do
not request work on their own.

After successful execution, the agent should remove the processed markers except
**Protected**.

If any request conflicts with its editing permissions or protection, the agent
should report the conflict and **stop the entire execution without modifying the
source**. The agent should proceed only after the conflict is resolved.

#### Action

An action requests work on a selected part of the source. Actions form two
groups.

**Modifiers** change code:

- **Implement** — Fill an unfinished part with working code.
- **Change** — Modify existing code to produce the requested behavior.
- **Refactor** — Improve the implementation while preserving behavior.

**Reporters** produce reports:

- **Review** — Inspect the code and report problems, suggested improvements or
  alternative designs.
- **Explain** — Describe how the code works, its design decisions and its
  relationships with other code.
- **Investigate** — Diagnose an observed problem by reproducing it, testing
  possible causes and reporting findings supported by evidence.

Each target can have **at most one Modifier** and **multiple Reporters**.
Combine related changes in one multiline Modifier.

Actions on the same target run in **source order**. In this example, Review runs
before Change and Explain runs after it:

```rust
//@ review: identify weaknesses in cancellation handling
//@ change: propagate cancellation to all worker tasks
//@ explain: describe the resulting cancellation flow
impl WorkerPool {
    // Existing implementation...
}
```

This ordering applies only to **Actions**. **Behavior** and **Freedom** apply
regardless of marker order.

Nested targets can have their own Actions. Run them **from the inside out**:
complete each nested target's Actions before running those on its enclosing
target. Assembly should report conflicts between their instructions.

The following are separate action marker examples:

```rust
//@ implement
//@ change: resume interrupted downloads without duplicating data
//@ refactor: separate parsing from filesystem access
//@ review: suggest improvements to cancellation and compare alternative designs
//@ explain: trace cancellation across the worker tasks
//@ investigate: reproduce the crash after reconnecting and identify its cause
```

```rust
//@ change: enforce the frame size limit before buffering payloads
impl FrameDecoder {
    //@ change: validate the frame length as soon as its prefix is complete
    fn push(&mut self, input: &[u8]) -> Result<Vec<Frame>, DecodeError> {
        // Existing implementation...
    }
    // Other methods...
}
```

##### Metaprogramming for Modifiers

**Modifiers** can also be expressed through metaprogramming. Here is a proposed
**Implement** macro in Rust:

```rust
macro_rules! implement {
    () => { todo!("Implementation requested") };
    ($text:literal) => {{
        let text: &str = $text;
        todo!("{}", text)
    }};
}
```

This macro keeps unfinished bodies compilable, so tests can run and fail when
they reach them. A call such as `implement!("Reject incomplete frames")` is
assembled as an ordinary **Implement** action targeting the call's source range.
The optional string literal becomes the action's `text`. A call without
arguments uses `text: ""`.

#### Behavior

A behavior marker describes what the resulting code should do:

- **Expect** — An expected result or general rule about observable behavior,
  including errors and state changes.
- **Case** — One concrete scenario with an expected outcome. A stateful case can
  have several steps. Each independent case needs its own marker.

All **Behavior** markers on a target belong to its **Modifier**, regardless of
comment order. **Reporters** do not accept Behavior. Their criteria belong in
the action text.

The agent should translate **Behavior** instructions into **native tests**. Also
don't forget that you can write tests directly when setup or assertions are
easier to express in code.

For example:

```rust
//@ expect: emit frames in input order
//@ expect: parse("text.json") => Error
//@ case: finish with an incomplete frame => TruncatedFrame
```

```rust
//@ <modifier>
//@ ...
//@ expect: retain incomplete input between calls
/*@ case:
receive half a frame => emit nothing
receive the remainder => emit that frame
*/
//@ case: finish with an incomplete frame => TruncatedFrame
impl FrameDecoder {
    // Existing implementations of push and finish...
}
```

#### Freedom

A freedom marker defines what the agent is allowed to change:

- **Restriction** — Editing permissions and additional limits for the *current
  execution*.
- **Protected** — Makes the attached item, or a specified part of it, read-only
  for the agent *across executions*.

**Restriction** supports two modes:

- **Strict** — Preserve supplied declarations and surrounding code. Implement or
  modify only the requested parts within that structure.
- **Flexible** — May reshape the selected code, including function signatures,
  and update affected callers.

A target can have **at most one Restriction marker**. Multiple **Restriction**
markers on the same target should produce an error.

An item without a **Restriction** inherits it from its enclosing scope. A local
marker replaces the inherited mode and instructions within that scope. At the
top level, the default is **Strict**. The agent should inspect the native scope
and module hierarchy to apply these rules.

Additional instructions can narrow the permitted changes. Both modes must
satisfy the requested behavior and respect protection.

A **Protected** marker *without instructions* applies to the entire attached
item, except temporary instruction comments. *With instructions*, it names the
part that must remain unchanged, such as a signature. **Protected** markers
themselves must remain unchanged. Ordinary comments can explain why an item is
protected.

The following are separate marker examples:

```rust
//@ restriction: strict
//@ restriction: flexible; use existing dependencies
//@ protected
//@ protected: signature
```

```rust
//@ protected
fn encode_frame(payload: &[u8]) -> Vec<u8> {
    // Existing implementation...
}

//@ change: distinguish malformed input from an incomplete frame
//@ restriction: flexible; use existing dependencies
fn decode_frame(input: &[u8]) -> Option<Frame> {
    // Existing implementation...
}
```




## Working with agents

This section explains how to work with an agent. It covers:

- **Workflow** — The development cycle.
- **Code Synthesis** — How source instructions are assembled and executed.
- **Verification** — How results are checked and review effort can be reduced.
- **Tools** — The tools used for assembly and execution.

Both semantic analysis and execution require the method rules. Supply them
through a **skill** or equivalent instructions supported by the harness.


### Workflow

Keep the source instructions and the resulting changes as separate commits:

1. **Commit the instructions.** Commit the source describing the work.
2. **Generate the result.** Build a prompt from that source and supply it to the
   agent.
3. **Commit the result.** Commit the generated changes after execution and
   checks succeed.

Both commits can use the same description, with a prefix identifying the stage.
For example:

```text
spec: Reject oversized frames before buffering
impl: Reject oversized frames before buffering
```

If you edit the source after assembly but before execution, rebuild the request.
A tool or script can automate this check and the cycle, including both commits.

Review the **implementation commit** and the agent's reports. For manual
corrections, stage the changes and use `git commit --amend` to update that
commit. To generate a fix instead, write new source instructions and repeat the
cycle.

This workflow keeps the instructions in Git history and shows them alongside the
implementation changes in diffs.


### Code synthesis

Code synthesis has two stages:

1. **Assembly** — Collect the source instructions into a structured prompt and
   check them.
2. **Execution** — Supply that prompt to the agent to perform the work.

#### Assembly

This stage is performed by a dedicated tool called an **assembler**.

Assembly has three steps, performed in order:

1. **Collection** — Build a structured prompt (**request**) from source
   instructions.
2. **Programmatic analysis** — Check marker syntax, targets and request
   structure.
3. **Semantic analysis** — Have an agent check the meaning of the instructions
   before the requested work begins.

The assembler should not generate records for inherited restrictions or defaults.

The assembler should produce the request in a structured format such as YAML or
JSON. Assembly should **leave the source unchanged**. It does not compile the
code or verify its correctness.

Both analyses should explain any issues found and identify the relevant
instructions.

##### The request format

```yaml
version: 1

# Lines and columns are 1-based.
# Columns count Unicode scalar values. A tab counts as one.

targets: # work targets and scopes with Freedom markers
  - file: <source path>
    start: [<line>, <column>]
    end: [<line>, <column>] # position after the target's last character

    actions: # optional, in source order, with at most one Modifier
      - kind: <implement | change | refactor | review | explain | investigate>
        text: <original instruction text> # keep line breaks
        at: [<line>, <column>] # omit when equal to the target's start
        behavior: # modifiers only
          - kind: <expect | case>
            text: <original instruction text>
            at: [<line>, <column>]

    restriction: # optional
      mode: <strict | flexible>
      text: <additional instructions> # optional
      at: [<line>, <column>]

    protected:
      - at: [<line>, <column>]
        text: <protected part> # optional — omit to protect the whole item
```

For example, these source instructions:

```rust
//@ change: reject oversized frames before buffering payloads
//@ explain: describe the resulting frame size check
//@ expect: reject an oversized length prefix without waiting for its payload
//@ restriction: strict; use existing dependencies
//@ protected: method signatures
impl FrameDecoder {
    fn finish(&self) -> Result<(), DecodeError> {
        implement!("Reject incomplete frames")
    }
    // Other methods...
}

//@ protected
fn encode_frame(payload: &[u8]) -> Vec<u8> {
    // Existing implementation...
}
```

could produce this request:

```yaml
version: 1
targets:
  - file: src/frame.rs
    start: [6, 1]
    end: [11, 2]
    actions:
      - kind: change
        text: reject oversized frames before buffering payloads
        at: [1, 1]
        behavior:
          - kind: expect
            text: reject an oversized length prefix without waiting for its payload
            at: [3, 1]
      - kind: explain
        text: describe the resulting frame size check
        at: [2, 1]
    restriction:
      mode: strict
      text: use existing dependencies
      at: [4, 1]
    protected:
      - at: [5, 1]
        text: method signatures

  - file: src/frame.rs
    start: [8, 9]
    end: [8, 47]
    actions:
      - kind: implement
        text: Reject incomplete frames

  - file: src/frame.rs
    start: [14, 1]
    end: [16, 2]
    actions: []
    protected:
      - at: [13, 1]
```

The generated requests are build artifacts. You can track all or selected
request files in Git, or ignore them. The original source instructions are
already preserved in Git history.

##### Programmatic analysis

The assembler should check marker syntax, target attachment and the request
against its schema. More than one **Modifier** on a target is an error, as is
**Behavior** without one. Explicit conflicts, such as **Change** on an entirely
**Protected** item, should also be detected.

##### Semantic analysis

After programmatic analysis succeeds, the assembler runs an agent in **read-only
mode** to check the instructions for contradictions, unclear action combinations
and missing decisions that block the work. The agent can consult the source and
suggest improvements to the instructions.

The agent should not ask the developer to do the work assigned to it (for
example, asking which algorithm to use when that choice is left to the agent).

#### Execution

Execution begins only after assembly succeeds.

Run the coding **harness** directly with the request file as the prompt. An
illustrative command:

```sh
harness --prompt path/to/request.yaml
```

The harness runs the agent, which acts as the **executor**. The agent should
carry out the actions within their permissions, translate **Behavior**
instructions into native tests and run the relevant checks.

Modifiers produce **changes in source files**. Reporters produce **reports**.

After success, the agent should remove its temporary markers and preserve
**Protected** markers. It should report the work done, verification evidence and
unresolved issues. A permission or protection conflict should stop execution and
leave the source as it was before execution.


### Verification

Verification can become the bottleneck when agents generate code faster than
developers can review it.

Generated changes should include **verification evidence** showing which
requested behaviors and restrictions were checked and what remains unchecked.
Reports should identify the checks run, their outcomes and any assumptions
underlying proofs. An agent's confidence alone is insufficient.

Tool implementations should help **reduce review effort**. Possible techniques
include automated test generation, static analysis, programmatic restriction
checks, formal verification and focused review aids. The appropriate combination
depends on the language, project and requested work.

These techniques should be judged by whether they reduce review effort while
maintaining quality.


### Tools

An **assembler** builds and checks the request. A coding **harness** runs the
agent with the request. The assembler also uses the harness for semantic
analysis.

Existing harnesses such as
[Codex](https://learn.chatgpt.com/docs/non-interactive-mode), [Claude
Code](https://code.claude.com/docs/en/headless) and
[OpenCode](https://opencode.ai/docs/cli/#run) support non-interactive execution.

Assembly and execution can run separately or through one combined tool.

**`sas`** is a command-line tool **under development** to automate assembly and
execution across programming languages. It will let you run source instructions
with a configured coding harness in one command, or assemble a request to
inspect and execute later.

The tool is *optional*. Projects can use equivalent tooling.
