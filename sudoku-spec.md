# Sudoku: Intent-Level IR for Neural Software Synthesis

Status: draft specification. The canonical language name is `Sudoku`.

## 1. Purpose

Sudoku is a concise, human-authored, intent-level intermediate representation for software. An LLM consumes Sudoku
and proposes a conventional implementation. Sudoku describes required structure and behavior while leaving incidental
implementation details open.

Sudoku supports two transformations:

1. **Code to Sudoku:** analyze a source project and produce a folder of Sudoku specifications.
2. **Sudoku to code:** consume a Sudoku folder and produce a project in one selected target language.

Initial code-generation targets are Python and Rust. Each generation run produces one target language, not both. The
format must remain open to future targets and must not encode Python or Rust as its semantic model.

## 2. Implementation model

Sudoku is implemented as specifications and agent instructions, not as conventional language machinery. It has no
required parser, AST, interpreter, VM, deterministic compiler, or runtime. LLMs read, write, resolve, and transform it.

Conformance depends on semantic interpretation, generated verification, and human clarification. An agent must not
silently resolve a material ambiguity when multiple interpretations remain plausible.

The keywords **must**, **should**, and **may** express required, preferred, and permitted behavior respectively.

## 3. Project and directory mapping

A Sudoku project is a folder of text files beneath a specification root. A generation invocation supplies or implies a
source root. Relative directories map one-to-one between those roots.

Example:

```text
spec/foo/bar/content.sudoku  ->  src/foo/bar/<generated modules>
```

The filename does not determine a generated filename. One Sudoku file may declare one or many modules, so files map
one-to-many to target files. A file containing one module may happen to produce one target file.

`.sudoku` is the conventional extension, not a recognition rule. `.md`, `.txt`, or another text file in the project
must be accepted when its contents follow this specification. Agents identify Sudoku from content and project context.

## 4. Declaration structure

Markdown-style headings declare program constructs. Heading depth expresses semantic nesting. A heading's scope ends
at the next declaration heading of equal or lesser depth.

A level-one declaration must be a module. A file may contain multiple level-one modules. Core declaration forms are:

```sudoku
# module tasks:
## class Task:
### def complete:
## def create_task:
## module serialization:
### def encode:
```

The trailing-colon form is canonical and must be emitted by Sudoku-producing agents. Readers must also accept a colon
immediately after the construct keyword:

```sudoku
# module: tasks
## class: Task
## def: create_task
```

This dual form applies to every declaration kind. Generic document headings such as `Overview` must not be used.
Explanatory content belongs to a declaration and is introduced by a semantic clause such as `intent:` or `requires:`.

Other clearly named program constructs may be used when needed, but agents should prefer the small core vocabulary and
must ask when a construct has multiple plausible meanings.

## 5. Module-to-target mapping

Each declared module maps to the selected target language's native module or package representation. Module names and
nesting are required public structure; target-specific filenames are consequences of that structure.

For Python, a leaf module normally maps to a `.py` file. A module containing nested modules becomes a package. Its own
content maps to `__init__.py`, and each nested module maps beneath that package.

```sudoku
# module parent:
## def root_function:
## module child:
### def child_function:
```

```text
parent/
  __init__.py       # root_function
  child.py          # child_function
```

Other targets must use their closest native equivalent. The mapping must preserve semantic module identity even when
the target's filesystem conventions differ from Python's.

## 6. Required and open interface structure

Explicit `module`, `class`, and `def` declarations require corresponding public target-language symbols with the same
identity and nesting. The generated public interface must be a superset of the declared interface:

```text
declared public interface ⊆ generated public interface
```

The generator may add public symbols and may freely add private helpers, classes, functions, adapters, files, and
internal modules. A Sudoku declaration list is therefore a lower bound, not an exhaustive implementation inventory.

When a capability matters but its interface shape does not, describe it in the enclosing module without declaring a
class or function. The generator may then choose any interface that provides the required capability.

## 7. Declaration bodies

A declaration body uses colon-introduced clauses and indentation for scope. It may mix concise English, structured
constraints, Python-like behavioral pseudocode, state machines, and verification obligations.

Common clauses include:

- `intent:` — the purpose and user-visible meaning of the construct.
- `requirements:` — behavior or capabilities that must be provided.
- `requires:` — preconditions.
- `ensures:` — postconditions and results.
- `invariant:` — conditions that must remain true.
- `effects:` — required state changes or externally visible effects.
- `errors:` — required failure conditions and observable error behavior.
- `forbid:` — behavior or implementation properties that are disallowed.
- `allow:` — explicit implementation freedoms.
- `state_machine:` — states, transitions, guards, and terminal conditions.
- `verify:` — English-language conditions that must become executable checks.
- `suggest:` — non-binding implementation guidance.

The vocabulary is extensible. An agent may interpret another concise label when its meaning is unambiguous. It must ask
for clarification when the label affects semantics and admits multiple plausible interpretations.

Normative clauses must be jointly satisfied. If two normative statements conflict, the agent must report the conflict
and ask for clarification rather than choosing one silently. A suggestion loses to every normative statement.

Example:

```sudoku
# module tasks:
    intent:
        Provide persistent task tracking.

    requirements:
        Tasks survive process restarts.

    allow:
        Choose any storage engine appropriate for the selected target.

## def complete_task:
    requires:
        The referenced task exists.

    ensures:
        The returned task is completed.

    invariant:
        Its identity and creation time do not change.

    errors:
        A missing task produces a not-found error without creating a task.
```

## 8. Behavioral pseudocode

Sudoku permits Python-like expressions and control flow because they concisely disambiguate behavior. Colons and
indentation express scope. Familiar constructs such as `if`, `elif`, `else`, `for`, `while`, and `match` may be used.

Behavioral pseudocode is normative about its described behavior, not its textual implementation. The target may use a
different algorithm or control-flow construct if all observable behavior and constraints are preserved.

```sudoku
## def complete_task:
    behavior:
        if task.status == completed:
            return task
        else:
            mark task completed
            return task
```

This does not require an inline `if`, those variable names, or that implementation structure in generated code.

Python-like notation is a precise authoring convention, not a requirement that Sudoku or its output execute as Python.

## 9. State machines

State-machine bodies may be used whenever behavior depends on history or allowed transitions. English-like notation is
valid when states, transitions, guards, effects, and forbidden transitions remain clear.

```sudoku
## class Task:
    state_machine:
        initial: open

        open:
            complete -> completed
            delete -> deleted

        completed:
            reopen -> open
            delete -> deleted

        deleted:
            terminal
```

Generated code may represent the state machine in any suitable way but must preserve its permitted behavior.

## 10. Suggestions and code blocks

Implementation guidance may be general or target-specific:

```sudoku
suggest:
    Keep storage behind a repository boundary.

suggest for Python:
    Prefer a dataclass for immutable value objects.

suggest for Rust:
    Prefer an enum when all variants are closed and known.
```

A code block may illustrate an algorithm, API, architecture, or target-language idiom. A code block under `suggest:` is
advisory and need not appear in the output. Code-like content under a normative behavioral clause specifies semantics,
not literal source text. Agents must infer force from the containing clause, not from the presence of a code fence.

Target-specific advice applies only when that target is selected and must not change target-neutral required behavior.

## 11. Verification

`verify:` is primarily English-language, checkable acceptance criteria. During Sudoku-to-code generation, the agent
must operationalize each verification item as one or more of:

- an in-code assertion;
- a unit, property, integration, or end-to-end test;
- both assertions and tests when appropriate.

The agent chooses the mechanism case by case based on scope, cost, and the selected target. It must preserve a clear
relationship between each verification item and the generated check.

```sudoku
verify:
    Completing an already completed task succeeds without emitting a second completion event.
    A deleted task is never returned by the ordinary list operation.
```

Verification describes what must be checked; it does not prescribe the testing framework or assertion placement.

## 12. Specification-level control flow

Python-like control flow may contain Sudoku declarations. In this role it is metaprogramming over the specification,
not required runtime control flow. It expresses families of declarations without listing each one.

A selector may be concise English enclosed in brackets. It may range over symbols and properties in the Sudoku project.
No rigid query grammar is required.

```sudoku
# module bindings:
    for x in [every public function in `source_api`]:
        ## def $(x)_bind:
            requirements:
                Provide the Python binding for `$(x)`.
```

The loop conceptually adds one required declaration for each selected function. The implementation agent need not emit
or store an expanded Sudoku file.

Metavariable substitution uses `$(name)`. Parentheses are required, especially when composing identifiers:

```text
$(x)_bind       # value of x followed by the literal suffix _bind
```

`$x_bind` is not valid substitution because it is ambiguous between variables named `x` and `x_bind`.

English-like conditional metaprogramming is also allowed when its selection rule and resulting declarations are clear.
An agent must ask if it cannot determine the selected symbol set with confidence.

## 13. Symbol references

Backticks denote references to symbols declared in Sudoku, locally or elsewhere in the project:

```text
`Task`
`complete_task`
`users.create`
```

Explicit imports are not required. Agents resolve references across the entire Sudoku project.

An unqualified reference may be used when it resolves unambiguously. When names collide, the author must qualify the
reference, for example `users.create` rather than `create`. The agent must request qualification if context does not
resolve an ambiguity confidently; it must not guess silently.

Backticks identify semantic symbols, not arbitrary source fragments or filesystem paths.

## 14. Code-to-Sudoku transformation

A code-to-Sudoku agent must:

1. inspect the complete relevant source project and its tests, configuration, and interfaces;
2. create a specification tree whose relative directories mirror the source tree;
3. represent required public modules, classes, functions, behavior, invariants, and externally visible errors;
4. express existing acceptance behavior as `verify:` items where useful;
5. retain target-specific implementation facts only when they are requirements or useful labeled suggestions;
6. omit incidental helpers when a higher-level requirement fully explains them;
7. qualify cross-project symbol references when short names are ambiguous; and
8. ask rather than invent when code and context do not reveal the intended semantics.

The result is a semantic specification, not a line-by-line transliteration or a promise of byte-identical regeneration.

## 15. Sudoku-to-code transformation

A Sudoku-to-code agent must:

1. read every relevant text file in the Sudoku project regardless of conventional extension;
2. resolve declarations and project-wide symbol references, asking about material ambiguities;
3. conceptually expand specification-level control flow and substitutions;
4. select exactly one requested target language;
5. map directories one-to-one and modules to native target structure;
6. generate at least the declared public interface while adding helpers as needed;
7. implement all normative behavior without treating pseudocode as literal source;
8. apply compatible suggestions, including target-specific recommendations, when useful;
9. generate assertions or tests for every `verify:` item; and
10. report unresolved requirements, unverified obligations, and material implementation assumptions.

The generator may make ordinary implementation choices left open by the specification. It must not turn missing product
behavior into an unreported assumption when that choice could materially change the result.

## 16. Conformance

A generated project conforms to a Sudoku project when all of the following hold:

- every declared module, class, and function exists in the required public nesting;
- all normative requirements and behavioral descriptions are satisfied;
- all state-machine and invariant constraints are preserved;
- every `verify:` item has a corresponding generated executable check;
- extra interfaces and implementation helpers do not contradict the specification;
- target-specific suggestions are treated as advice, not as stronger requirements; and
- material ambiguities and unsatisfied obligations are disclosed rather than hidden.

Conformance does not require textual similarity to behavioral pseudocode, preservation of source-code architecture,
identical helper structure across targets, or a deterministic output project.

## 17. Design summary

Sudoku is a target-neutral lower bound on required software structure plus a behavioral contract. Its Pythonic surface
improves precision for humans and LLMs without making Python its execution model. Its declarations constrain public
shape; its clauses constrain semantics; its metaprogramming describes declaration families; its verification clauses
create executable checks; and its suggestions guide, but do not bind, the synthesizer.

The LLM is an untrusted semantic transformer with broad implementation freedom. The Sudoku project remains the compact
source of intent against which generated structure, behavior, tests, and disclosed assumptions are evaluated.
