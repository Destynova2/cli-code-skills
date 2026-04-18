# L6 — CFG Topology Watermark

> **When to read:** Phase 2, applying layer 6.

---

## Principle (Venkatesan et al., 2001 — still valid)

The watermark is encoded in the **topology of the control flow graph** (CFG), not in the code. Each basic block is a node, each branch is an edge. The watermark is embedded by adding or rearranging edges in a way that encodes a bitstream.

This is the most language-agnostic code-level technique: Rust → Go → Python, the CFG of an algorithm stays structurally similar.

## Technique

### 1. Document your CFG privately

Before watermarking, extract the CFG of your key functions:

```bash
# Rust: use cargo-callgraph or manual analysis
# Python: use pyan3
# Go: use go-callvis

# Manual: for each key function, draw the basic blocks and branches
```

### 2. Encode in branching order

The order of `match` arms, `if/else` chains, and `switch` cases is arbitrary when all branches are independent. Choose the order according to your bitstream.

```rust
// Natural ordering: alphabetical or by frequency
match model {
    "claude" => route_claude(),
    "gpt-4" => route_gpt4(),
    "llama" => route_llama(),
}

// Watermarked ordering: derived from L6_KEY
// bit pattern 101 → claude, llama, gpt-4
match model {
    "claude" => route_claude(),  // bit 1
    "llama" => route_llama(),    // bit 0
    "gpt-4" => route_gpt4(),    // bit 1
}
```

### 3. Encode in error handling topology

Where you check errors (early return vs. nested if vs. result chain) is a topological choice:

```rust
// Topology A: early return chain (flat CFG)
fn process(req: Request) -> Result<Response> {
    let model = validate_model(&req)?;
    let route = find_route(model)?;
    let result = execute(route)?;
    Ok(result)
}

// Topology B: nested (deep CFG) — same behavior
fn process(req: Request) -> Result<Response> {
    match validate_model(&req) {
        Ok(model) => match find_route(model) {
            Ok(route) => execute(route),
            Err(e) => Err(e),
        },
        Err(e) => Err(e),
    }
}
```

Both are correct. The CFG topology is different. Document which you chose and why.

## Detection

To compare two implementations' CFGs:

1. Extract the CFG from both (manually or with tools)
2. Compute the **graph edit distance** between them
3. If the distance is below a threshold → the topologies match → evidence of copying

## Codebook entry

```json
{
  "layer": "L6",
  "cfg_markers": [
    {
      "function": "process_request",
      "file": "src/router/mod.rs",
      "topology": "early_return_chain",
      "branch_order": [2, 0, 1],
      "derived_from": "L6_KEY bits 0-2"
    }
  ]
}
```

## Resilience

- **Rename**: ✅ topology is structural
- **Restructure**: ⚠️ if the function is split or merged
- **Language change**: ✅ the CFG shape transfers
- **2 rewrites**: ✅ an AI preserves the algorithm's branching structure
- **Clean code**: ⚠️ a linter might reorder match arms alphabetically
