# Fix for Context.Tag.Identifier Dependency Extraction

## Problem

The architecture analyzer was unable to extract dependencies from services that used the `Context.Tag.Identifier<typeof X.Y.tag>` pattern in their layer type annotations.

**Example:**
```typescript
export const Default: Layer.Layer<
  SynthesisService,
  never,
  Context.Tag.Identifier<typeof EvaluationSession.EvaluationSession.tag> | LanguageModel.LanguageModel
> = Layer.effect(
  SynthesisService,
  Effect.gen(function* () {
    const session = yield* EvaluationSession
    const model = yield* LanguageModel
    return { synthesize: () => Effect.succeed("result") }
  })
)
```

**Expected:** Extract dependencies: `EvaluationSession`, `LanguageModel`
**Actual:** Extract nothing, showing `<never, never>` in the dependency graph

## Root Cause

The `extractDepsFromType()` function in `.claude/scripts/analyze-architecture.ts` could extract simple type names but failed on `Context.Tag.Identifier<typeof X.Y.tag>` patterns because:

1. When recursively processing type arguments, it would encounter a type with symbol name "Identifier"
2. It would add "Identifier" as a dependency (filtered out as infrastructure)
3. It never unwrapped the `typeof X.Y.tag` expression inside to extract the actual service name

## Solution

Added special handling for the "Identifier" symbol name before general type processing:

```typescript
if (symbolName === "Identifier" && isTypeReference) {
  const typeArgs = checker.getTypeArguments(typeRef)
  if (typeArgs && typeArgs.length > 0) {
    const typeArg = typeArgs[0]
    const typeArgString = checker.typeToString(typeArg)

    if (typeArgString.startsWith("typeof ")) {
      const typeofContent = typeArgString.slice(7)
      const parts = typeofContent.split(".")
      const serviceName = parts[0]

      if (
        !isEffectInfrastructure(serviceName) &&
        !isExcludedFromGraph(serviceName) &&
        !deps.includes(serviceName)
      ) {
        deps.push(serviceName)
      }
    }
  }
  return
}
```

## How It Works

1. **Detect the pattern:** Check if the symbol name is "Identifier" (from `Context.Tag.Identifier` or `Tag.Identifier`)
2. **Extract type argument:** Get the first type argument (the `typeof X.Y.tag` part)
3. **Convert to string:** Use `checker.typeToString()` to get the string representation
4. **Parse typeof expression:**
   - Strip the `"typeof "` prefix
   - Split by `.` to handle nested property access like `X.Y.tag`
   - Take the first part as the service name
5. **Add dependency:** Add the extracted service name to the dependencies list

## Test Coverage

- Updated both `/claude/scripts/analyze-architecture.ts` and `/claude/scripts/__tests__/analyze-architecture.test.ts`
- All existing tests pass
- Added (skipped) test case for `Context.Tag.Identifier` pattern (requires real Effect library to fully test)
- Verified analyzer still works correctly on the demo codebase

## Examples of Patterns Handled

```typescript
Context.Tag.Identifier<typeof EvaluationSession.EvaluationSession.tag>
→ Extracts: "EvaluationSession"

Context.Tag.Identifier<typeof LanguageModel>
→ Extracts: "LanguageModel"

Tag.Identifier<typeof MyService.tag>
→ Extracts: "MyService"
```

## Files Modified

1. `.claude/scripts/analyze-architecture.ts` - Lines 182-203
2. `.claude/scripts/__tests__/analyze-architecture.test.ts` - Lines 121-170

Both files had the same `extractDepsFromType` function that needed updating.
