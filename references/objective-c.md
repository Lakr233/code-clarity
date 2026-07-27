# Objective-C Clarity

Use these rules when reviewing or changing Objective-C, Objective-C++, or mixed Objective-C/C implementation files.

## Preserve the ownership model

ARC manages Objective-C object references; it does not manage every resource in an Objective-C file.

- Give each Core Foundation object, file descriptor, Mach port, mapping, allocation, lock, and spawned process one owner and one cleanup path.
- Pair successful acquisition with release on every later exit. Prefer one cleanup section when several resources share a lifetime.
- Use `__bridge`, `__bridge_retained`, and `__bridge_transfer` only to express a real ownership transfer.
- Capture objects and mutable state in blocks deliberately. Avoid adding `__block` merely to silence a structural problem.
- Check sizes, offsets, counts, and additions before allocation, copying, mapping, or pointer arithmetic.

Do not hide ownership behind a helper whose name or return type makes the caller believe cleanup is automatic.

## Keep APIs smaller than implementations

- Put implementation-only declarations in a class extension in the `.m` file.
- Add a property only when the object must retain state across operations. Keep a temporary value local when it belongs to one call.
- Use a category for a cohesive capability on an existing class, not to scatter one lifecycle across unrelated files.
- Prefer a concrete collaborator unless multiple implementations actually exist.
- Use `copy` for blocks and externally supplied immutable value objects when mutation or stack lifetime would otherwise leak through the boundary.
- Apply nullability at public and subsystem boundaries. Do not let nested `_Nullable` pointer syntax force a readable declaration into arbitrary one-token lines.

When Objective-C wraps a C subsystem, keep policy in the Objective-C owner and low-level mechanism in the C layer. Do not duplicate the same state machine in both.

## Group errors by recovery

- Define an error domain or code only when a caller branches on the distinction or the code is a stable subsystem boundary.
- Reuse one construction path for failures with the same caller response.
- Preserve underlying detail in `NSUnderlyingErrorKey` or diagnostics instead of creating control states for every low-level status.
- Convert `errno`, `kern_return_t`, launch status, and library-specific codes once at the owning boundary.
- Keep the successful path flat with early returns; avoid nested `if` ladders that only decorate an eventual `NSError`.

## Format statements, not tokens

Objective-C selectors are descriptive and can be long. Readability comes from seeing the operation and its meaningful groups, not from forcing every selector piece or argument onto a separate physical line.

Keep an ordinary statement on one line when it remains locally readable:

```objc
RLXKernelAccess *access = [[RLXKernelAccess alloc] initWithKernelcachePath:path];
return [super initWithStage:RLXEngineStageKernelAccessAcquisition context:context];
rlx_engine_log(RLX_ENGINE_LOG_INFO, "RLXEngine", message.UTF8String);
```

Avoid mechanical fragmentation:

```objc
RLXKernelAccess *access =
    [[RLXKernelAccess alloc]
        initWithKernelcachePath:
            path];
```

For longer messages, break at semantic groups. Keep the receiver and first selector together, and do not leave `=`, `return`, `[NSError`, `[NSString`, a cast, or a selector colon stranded on a line by itself.

```objc
return [NSError errorWithDomain:RLXEngineErrorDomain
                           code:RLXEngineErrorCodeInvalidAction
                       userInfo:@{
                           NSLocalizedDescriptionKey: @"Kernel access is unavailable.",
                           NSUnderlyingErrorKey: error,
                       }];
```

Preserve vertical layouts when each line is independently auditable:

- validation predicates with one invariant per line;
- whitelists, mappings, tables, and ordered pipeline stages;
- long low-level calls where argument position carries safety meaning;
- adjacent string literals arranged as a protocol, command, or diagnostic template;
- macro definitions and conditional-compilation regions.

Line count is not a target. Do not collapse a file, function, dictionary, condition table, or macro into one line merely because the language permits it.

## Treat formatting as a semantic-risk boundary

- Follow an existing committed Objective-C formatter configuration when it produces readable local results.
- If formatter output repeatedly fragments selectors or messages, fix the Objective-C-specific configuration before reformatting source.
- Do not apply a new whole-tree style while changing behavior.
- Keep includes/imports, preprocessor directives, comments, macro bodies, and adjacent string contents stable unless the task explicitly covers them.
- For a formatting-only change, inspect a token-aware diff and compile every affected target and conditional path available in the repository.

`clang-format` is a parser-backed formatter, not proof of semantic equivalence. Pay special attention to macros, blocks, Objective-C literals, mixed C declarations, and preprocessor continuations.

## Review checklist

1. Which object or function owns each piece of state and each non-object resource?
2. Can any property, flag, phase, or cached value be derived?
3. Do distinct error codes cause distinct recovery?
4. Is private behavior leaking into a public header?
5. Does a category preserve one capability or split one lifecycle?
6. Are blocks and queues capturing only the state they need?
7. Are arithmetic and buffer boundaries checked before the operation?
8. Does formatting reveal complete statements and semantic groups?
9. Are vertical lists intentionally auditable rather than mechanically wrapped?
10. Does the diff preserve tokens and compile across affected configurations?
