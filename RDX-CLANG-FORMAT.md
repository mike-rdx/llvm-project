# rdx clang-format

This fork of [llvm/llvm-project](https://github.com/llvm/llvm-project) carries a modified
clang-format for the rdx code base. It adds two style options that the rdx coding rules need and
that upstream clang-format does not have, plus one backported upstream change, which the fork
extends with a `WithoutName` sub-option and lambda return types.

Everything else is unchanged LLVM 21.1.8. With the new options left at their defaults, the
modified clang-format produces exactly the same output as upstream clang-format 21.1.8.

## Branches

| Branch | Content |
|---|---|
| `backport-pr-169160` | **The working branch.** Tag `llvmorg-21.1.8` plus the commits listed below. Build clang-format from this branch. |
| `pr-169160` | The three original commits of upstream PR [#169160](https://github.com/llvm/llvm-project/pull/169160) on LLVM `main`; the source of the backport. |

Commits on `backport-pr-169160` on top of `llvmorg-21.1.8`:

| Commit | Change |
|---|---|
| `8533d4b5` | Backport: `[clang-format][NFC] Upgrade PointerAlignment option to a struct` (Daan De Meyer) |
| `dd0aee0a` | Backport: `[clang-format] Allow custom pointer/ref alignment in return types` (Daan De Meyer) |
| `a0331763` | `[clang-format] Add BreakAfterAssignment style option` |
| `83b37e58` | `[clang-format] Guard forced assignment breaks` |
| `35b84c76` | `[clang-format][NFC] Fix Defaulf typo in FormatTest` (the backported test did not compile) |
| `c88ad7eb` | `[clang-format] Add LambdaHeaderOnStatementLine style option` |
| `fc5ad20d` | `Add rdx clang-format documentation` (this file) |
| `ccf6c2ff` | `[clang-format] Measure only the header of a stored lambda in BreakAfterAssignment` |
| `28e3a734` | `[clang-format] Find stored lambda headers ending in noexcept and a trailing return type` |
| `0923ff11` | `[clang-format] Align a stored lambda's body with its header's line` |
| `845e286c` | `[clang-format] Allow a trailing comment after the header in LambdaHeaderOnStatementLine` |
| `aa87b167` | `[clang-format] Treat lambda trailing return types as return types in ReturnType alignment` |
| `1b1329d9` | `[clang-format] Add WithoutName to PointerAlignment and ReferenceAlignment` |

## New options

### BreakAfterAssignment

```yaml
BreakAfterAssignment: Never        # default: unchanged behaviour
BreakAfterAssignment: IfOverLimit
```

With `IfOverLimit`, a line break is forced right after an assignment operator when the right-hand
side does not fit in the column limit, so the statement wraps as a "ladder":

```cpp
// Never
const ResultTable result_table = TableBuilder::buildFromParameters(
	InputFormat::Full, source_parameters.first_readable_location, source_parameters.second_readable_location);

// IfOverLimit
const ResultTable result_table =
	TableBuilder::buildFromParameters(
		InputFormat::Full, source_parameters.first_readable_location, source_parameters.second_readable_location);
```

Why a new option: upstream clang-format cannot express this rule.
`BreakBeforeBinaryOperators: NonAssignment` only decides on which side of the `=` a break goes,
and `PenaltyBreakAssignment` only makes a break after `=` cheaper or more expensive. clang-format
optimizes the layout of the whole statement and often prefers breaking inside the right-hand-side
call instead of after the `=`. `IfOverLimit` therefore makes the break mandatory instead of tuning
penalties.

The break is forced only when all of the following hold:

- it is the first assignment of a top-level statement (not inside parentheses, a ternary branch, a
  default argument, or later in a chain like `a = b = c`);
- the statement does not declare several variables (`int a = 1, b = 2;`);
- it is not `= 0`, `= default` or `= delete` after a function declaration;
- the right-hand side does not start with `{`;
- a line break is allowed at that point;
- the right-hand side, measured from after the `=` to the end of the expression, does not fit.
  Trailing comments, lambda bodies and the final `;` are not counted.

For a lambda stored in a variable, only the lambda's header is measured: its captures, parameters,
specifiers and trailing return type, without the body. If the header doesn't fit after the `=`, it
moves to the next line ("staircase"). If it doesn't fit there either, it breaks after `](`, with
the parameters one level deeper; a break inside the captures is allowed only if the capture list
itself doesn't fit on the line. With `BraceWrapping.BeforeLambdaBody`, the body's `{`, the body and
the closing `};` are then aligned with the line the header starts on, as hand-written rdx code does
(not with `LambdaBodyIndentation: OuterScope`, which keeps them at the statement's indentation).

```cpp
// before
auto updateSelectedRow = [this, &selection_model,
						  &visible_row_indexes](const QModelIndex &index, const SelectionMode mode) -> bool
{
	return apply(index, mode);
};
auto writeRowToReport = [this, &report_stream, &column_widths, first_visible_column_index](
							const ReportRow &row, const QStringView row_prefix, const QStringView row_suffix)
{
	write(row);
};

// IfOverLimit
auto updateSelectedRow =
	[this, &selection_model, &visible_row_indexes](const QModelIndex &index, const SelectionMode mode) -> bool
	{
		return apply(index, mode);
	};
auto writeRowToReport =
	[this, &report_stream, &column_widths, first_visible_column_index](
		const ReportRow &row, const QStringView row_prefix, const QStringView row_suffix)
	{
		write(row);
	};
```

### LambdaHeaderOnStatementLine

```yaml
LambdaHeaderOnStatementLine: Never               # default: unchanged behaviour
LambdaHeaderOnStatementLine: IfFitsOnAssignment
LambdaHeaderOnStatementLine: IfFitsAlways
```

This implements the rdx style-guide exception for lambdas: if a statement ends with a call whose
last argument is a lambda with a multi-line body, and everything from the start of the statement
through the lambda header fits on one line, that line stays unbroken and the lambda body is placed
at the indentation of the statement:

```cpp
// Never
const auto item_iter = std::ranges::find_if(
	child_items,
	[&expected_title, this](const ItemPath &item_path)
	{
		return (getItemTitle(item_path) == expected_title);
	});

// IfFitsAlways / IfFitsOnAssignment
const auto item_iter = std::ranges::find_if(child_items, [&expected_title, this](const ItemPath &item_path)
{
	return (getItemTitle(item_path) == expected_title);
});
```

- `IfFitsOnAssignment` applies only to statements with a top-level assignment operator, such as
  `x = f(...);` or `auto x = f(...);` (the assignments `BreakAfterAssignment` recognizes). Direct
  initialization `T x(f(...));` is not included.
- `IfFitsAlways` applies to all statements: assignments, plain calls such as
  `QObject::connect(...)`, macro calls, and `return`.

A line keeps the regular formatting if any of these is true:

- the whole statement fits on one line;
- the lambda is not the last argument (e.g. `std::visit([...] {...}, values)`), or its body is
  empty;
- the statement is a declaration: a function parameter list (e.g. a default argument), `using`,
  `typedef`, `static_assert` or `template`; or it starts with `if`, `for`, `while` or `switch`;
- the line is in a macro definition;
- the header contains a comment (other than a `//` comment that ends the header's line, see below),
  another block, a multi-line token (e.g. a raw string) or a forced line break;
- the header does not fit in the column limit.

A `//` comment that ends the header's line stays on that line and counts toward the column limit.
Nothing can follow it there, so such a lambda is never kept on one line, and the statement gets the
layout even if it would fit on one line without the comment:

```cpp
Util::callWithExceptionSuppression([this]() // no one catches exceptions from here
{
	updateView();
});
```

Short lambdas: if `AllowShortLambdasOnASingleLine` would merge the lambda into one line and it fits
on a continuation line, it stays on one line. Line breaks inside it are penalized, so it is neither
folded into `{ ...; }` on the next line nor split inside its capture list.

The option requires `BraceWrapping.BeforeLambdaBody: true`.

### Pointer and reference alignment: return types (backport of PR #169160) and `WithoutName`

`PointerAlignment` and `ReferenceAlignment` accept a struct with a `ReturnType` sub-option that
overrides the alignment in function return types. The two overrides are independent of each
other. rdx uses the struct form for both: pointers and references right-aligned, except in
function return types, where they are left-aligned.

```yaml
PointerAlignment:
  Default: Right
  ReturnType: Left    # Default | Left | Right | Middle
ReferenceAlignment:
  Default: Pointer
  ReturnType: Left
```

```cpp
int* function(int *argument);
int *variable;
auto trailing(int *argument) -> int*;
int& referenceFunction(int &argument);
const std::vector<int>&& takeValues();
```

- The override applies to leading and trailing return types, member functions, templates,
  qualifiers, multiple indirection, `&` and `&&`. Variables and parameters keep the `Default`
  alignment.
- rdx addition: the trailing return types of lambdas count as return types too
  (`[](int *a) -> int* { ... }`); upstream PR #169160 covers only functions.
- `ReturnType: Default` keeps the LLVM 21 behaviour.
- The old one-word form, e.g. `PointerAlignment: Right`, is still accepted, including the old
  boolean aliases.

#### rdx addition: `WithoutName`

A third sub-option, with the same values as `ReturnType`, sets the alignment of a pointer or
reference that no name follows: in casts, template arguments, function types, `sizeof`, unnamed
parameters (also with a default value or a commented-out name) and `catch` clauses. It applies when
the next token after the `*`/`&` (skipping further `*`/`&`, `const`/`volatile` and `...`) is `>`,
`)`, `,` or `=`. `ReturnType` takes precedence.

```yaml
PointerAlignment:
  Default: Right
  ReturnType: Left
  WithoutName: Left
ReferenceAlignment:
  Default: Pointer
  ReturnType: Left
  WithoutName: Left
```

```cpp
auto *p_derived = dynamic_cast<Derived*>(p_base);
std::map<Key*, std::vector<Value*>> values;
std::function<void(const QString&)> callback;
char *p_text = (char*)p_data;
void addItem(Item*, const QString &title, int* = nullptr);
catch (const std::bad_alloc&)
```

Named declarations, structured bindings (`const auto &[a, b]`), ref-qualifiers
(`Foo &operator=(const Foo&) & = delete;`) and expressions (`a * b`, `&value`) are not affected.
`WithoutName: Default` keeps the `Default` alignment.

## Building (Windows, Visual Studio 2022)

```bat
cmake -S llvm -B build-codex -G "Visual Studio 17 2022" -A x64 -DLLVM_ENABLE_PROJECTS=clang -DLLVM_TARGETS_TO_BUILD=X86
cmake --build build-codex --config Release --target clang-format FormatTests
```

- clang-format: `build-codex\Release\bin\clang-format.exe`
- unit tests: `build-codex\tools\clang\unittests\Format\Release\FormatTests.exe`

## Tests

All 1,225 `FormatTests` pass. The new options are covered by these tests in
`clang/unittests/Format/FormatTest.cpp`:

| Test | Covers |
|---|---|
| `BreakAfterAssignment`, `BreakAfterAssignmentUsesRightHandSideLength`, `BreakAfterAssignmentSkipsFunctionSpecifiersAndDeclarators` | the forced break, the right-hand-side measurement, the exclusions |
| `BreakAfterAssignmentMeasuresStoredLambdaHeader` | stored lambdas: header on the statement line, staircase, break after `](`, captures too long for a line, a lambda inside a call, Allman braces, `noexcept -> T`, `Never` |
| `BreakAfterAssignmentAlignsStoredLambdaBody` | the body's braces follow the header's line: header on the statement line, staircase, `noexcept -> bool`, a nested lambda, `OuterScope`, `Never` |
| `LambdaHeaderOnStatementLine` | `Never` vs. `IfFitsAlways`, trailing return types, `connect()`, nested statements, short bodies, `-> double &`, parentheses around the lambda |
| `LambdaHeaderOnStatementLineOnAssignment` | assignments get the layout, plain calls do not |
| `LambdaHeaderOnStatementLineSkipsNonFittingHeaders` | header of exactly the column limit vs. one more, lambda not the last argument, comment in the header |
| `LambdaHeaderOnStatementLineKeepsTrailingComment` | a `//` comment after the header: long and short bodies, `-> bool`, `[]` without parameters, the comment counted in the column limit, other comments (on a line of their own, block comment, inside the captures), `IfFitsOnAssignment` |
| `LambdaHeaderOnStatementLineSkipsNonStatements` | default argument, type alias, `static_assert`, multi-line raw string, empty lambda |
| `ReturnTypeAlignment` | the backported `ReturnType` tests, plus lambda trailing return types |
| `WithoutNameAlignment` | casts, template arguments, function types, `sizeof`, unnamed parameters, `catch`; named declarations, structured bindings, ref-qualifiers and expressions unchanged; `ReturnType` precedence |

`ConfigParseTest.cpp` checks that all option values parse.

Not covered by unit tests yet: the guard against layouts with no valid solution, deeper nesting
such as `f(g(x, [] {...}))`, `BraceWrapping.IndentBraces`, and lambdas in macro definitions.

## Use in rdx

The rdx `.clang-format` enables the options of this fork:

```yaml
BreakAfterAssignment: IfOverLimit
LambdaHeaderOnStatementLine: IfFitsAlways
PointerAlignment:
  Default: Right
  ReturnType: Left
ReferenceAlignment:
  Default: Pointer
  ReturnType: Left
```

The examples in this document also depend on these upstream options of the rdx configuration.
After the forced break after `=`, they decide how the call itself is wrapped; with other values the
results look different:

```yaml
BreakBeforeBraces: Allman     # also sets BraceWrapping.BeforeLambdaBody
ColumnLimit: 120
AlignAfterOpenBracket: AlwaysBreak
BinPackArguments: true
BinPackParameters: BinPack
PackConstructorInitializers: Never
BreakBeforeBinaryOperators: NonAssignment
AllowShortLambdasOnASingleLine: Inline
```

A stock clang-format stops with "unknown key" on this file, so every user and build agent needs
this build. The rdx scripts find it through an environment variable, then a local build path, then
`clang-format` on `PATH`, and refuse to run with a binary that cannot read the configuration.

- A style-check script checks enrolled modules (check only).
- A formatting script formats code written by an AI agent: new files completely, existing files
  only on the changed lines, then runs a coding-rules lint. Claude Code hooks and a skill use it.

### Validation on rdx code

Measured on the C++ files of the main rdx solution with the rdx configuration:

- With the new options off, the output is byte-identical to upstream clang-format 21.1.8.
- `BreakAfterAssignment: IfOverLimit` changed 94 places in a 150-file sample. All are breaks after
  a top-level assignment; no new lines exceed 120 columns.
- `LambdaHeaderOnStatementLine` on the 554 files that contain lambdas:

  | | Before | `IfFitsOnAssignment` | `IfFitsAlways` |
  |---|---|---|---|
  | Files changed | – | 20 | 179 |
  | Statements changed | – | 31 | 386 |
  | Lambda in a call, header fits (40): exception layout | 0 | 30 | 30 |
  | One-line lambdas | 585 | 586 | 601 |
  | Folded `{ ...; }` bodies | 160 | 154 | 92 |

- Stored lambdas (`BreakAfterAssignment` measuring only the lambda header): 39 headers that the
  previous build broke inside the captures or parameters now use the staircase. Of these, 9 fit on
  the line after `=`; the others break after `](`, except two whose capture lists don't fit on any
  line. With the body's braces aligned with the header's line, all 142 staircase lambdas get the
  `{` under the `[`; hand-written rdx code does the same in 134 of 156 cases. Compared with the
  previous build, only stored-lambda statements change (110 of 381 files containing such lambdas,
  none of 300 other files).
- No crashes, no non-whitespace changes, and no new files that need a second formatting pass.

## Known limitations

- A lambda capture list that doesn't fit on a line by itself wraps inside the brackets.
- Some files need two formatting passes because of comment re-wrapping. This is upstream
  behaviour, not caused by these changes.

## Updating to a newer LLVM

Newer LLVM releases add finer controls for breaking after opening brackets, but none of them
provides `BreakAfterAssignment` or `LambdaHeaderOnStatementLine`. A newer release therefore does not
replace this fork: the rdx commits have to be carried forward.

1. `git fetch origin --tags`
2. Create a branch from the new release tag, e.g. `git switch -c rdx-22 llvmorg-22.1.0`.
3. Cherry-pick the rdx commits: `git cherry-pick llvmorg-21.1.8..backport-pr-169160`. Drop the two
   backport commits if the new release already contains PR #169160.
4. Build, run `FormatTests`, and compare the output on rdx code before switching the team to the
   new binary.

## Repository setup

- `origin`: `llvm/llvm-project`, fetch only. Its push URL is set to `no_push_to_upstream`, so a
  push to upstream fails locally.
- `fork`: `mike-rdx/llvm-project`. Both branches track it, so a plain `git push` updates the fork.
