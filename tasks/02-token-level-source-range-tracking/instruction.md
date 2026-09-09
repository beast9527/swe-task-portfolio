# Instruction: Add Token-Level Source Range Tracking and Location-Aware Lexer Diagnostics

## Overview

This task extends the tokenizer and lexer subsystems of the csstree repository. The goal is to attach deterministic source locations to tokens produced by `tokenize()`, expose an iteration counter on all Lexer match results, include precise source locations in `SyntaxMatchError` objects, and add a public `getLocation(offset)` method to `TokenStream`. All changes must be backward compatible: consumers that ignore the new properties must continue to receive the same token stream shape and match result structure otherwise.

## Public Contracts

### 1. Token Source Range Tracking (`tokenize`)

- `tokenize(source)` returns an array of token objects. Each token object must have a `loc` property with numeric `offset`, `line`, and `column` fields.
- `offset` is the zero-based character index of the token's first character in `source`.
- `line` and `column` are 1-based.
- For a token starting at source index 0, `loc` is `{ offset: 0, line: 1, column: 1 }`.
- For a token starting after a newline, `line` increments by 1 and `column` resets to 1.
- For a token starting after a non-newline character, `column` increments by 1 and `line` remains unchanged.
- The `loc` property is added without changing the token type, value, or any other token fields. Consumers that ignore `loc` will continue to receive the same token stream shape otherwise.

### 2. Lexer Match Iteration Counter (`Lexer`)

- Every result object returned by `Lexer.match`, `Lexer.matchProperty`, `Lexer.matchType`, `Lexer.matchAtrulePrelude`, `Lexer.matchAtruleDescriptor`, and `Lexer.matchDeclaration` must have an `iterations` property that is a non-negative integer.
- The value is the number of iterations performed by the matching algorithm for that call.
- When a match succeeds, the result object has `matched` set to the matched AST fragment, `error` set to `null`, and `iterations` set to the iteration count.
- When a match fails, `matched` is `null`, `error` is a `SyntaxMatchError` or other error object, and `iterations` is still present and numeric.
- The result object may contain additional properties.

### 3. Lexer Match Location Diagnostics (`SyntaxMatchError`)

- When a Lexer match fails and produces a `SyntaxMatchError`, the error object must have a `loc` property with `start` and `end` objects.
- Each of `start` and `end` has numeric `offset`, `line`, and `column` fields.
- The `start` and `end` locations correspond to the mismatched token range derived from the token `loc` data.
- The error object also has top-level `offset`, `line`, and `column` properties equal to the `start` location's `offset`, `line`, and `column` respectively.
- The `error.css` property contains the concatenated token values used for matching.
- The `error.mismatchOffset` and `error.mismatchLength` properties are numeric and describe the mismatch within `error.css`.

### 4. TokenStream Location Query (`TokenStream`)

- `TokenStream.getLocation(offset)` returns an object with numeric `line` and `column` fields for the given character offset.
- The `offset` is zero-based.
- The returned `line` and `column` are 1-based and correspond to the position of the character at that offset in the source string.
- If `offset` is less than 0, `getLocation` returns `{ line: 1, column: 1 }`.
- If `offset` is greater than or equal to the source length, `getLocation` returns the location of the end of the source, which is the position after the last character.

## Implementation Constraints

These constraints are mandatory for the implementation but are not part of the public API contract. They must be followed to ensure correctness and performance.

- The tokenizer must compute `loc` incrementally during token emission using the source string and the current token start offset. It must not perform a separate full-source scan after tokenization.
- The `iterations` value must be obtained from the internal match result object returned by `matchAsTree` and passed through `buildMatchResult` without modification.
- The `locateMismatch` function must use the token `loc` objects from the match result tokens when available. If a token lacks a `loc` object, the fallback behavior using `node.loc` and `defaultLoc` must be preserved.
- `getLocation` must use the precomputed location data from the token stream's `OffsetToLocation` instance. It must not rescan the source string.

## Acceptance Criteria

Your implementation must satisfy all public contracts above. The behavior must be deterministic and backward compatible for consumers that do not use the new properties.
