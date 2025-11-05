# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

rev.ng is an open-source binary analysis framework and decompiler based on LLVM and QEMU. It lifts binary code to LLVM IR, performs extensive analyses, and generates valid, recompilable C code. The project uses a pipeline-based architecture where analyses and transformations are orchestrated through a YAML-configurable pipeline system.

**Key Technologies**: C++20, LLVM, MLIR, QEMU, CMake, Python bindings

**License**: Individual files are MIT-licensed, but the project as a whole is GPLv2 due to QEMU dependencies.

## Build Commands

```bash
# Build from source (using orchestra - recommended)
python3 -m pip install --user --force-reinstall https://github.com/revng/revng-orchestra/archive/master.zip
export PATH="$HOME/.local/bin:$PATH"
git clone https://github.com/revng/orchestra
cd orchestra
orc install --test revng

# Manual CMake build (requires pre-installed dependencies)
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build . --parallel $(nproc)

# Build documentation
cmake --build build --target mkdocs
# Output: share/doc/revng/html/index.html
```

## Testing

```bash
# Run all tests
cd build && ctest --parallel $(nproc)
ctest --output-on-failure --parallel $(nproc)  # verbose

# Run tests by label
ctest -L unit        # Unit tests only
ctest -L pipeline    # Pipeline tests
ctest -L abi         # ABI tests

# Run specific test
ctest -R test_model -VV
ctest -R test_lazysmallbitvector -VV

# Find available tests
ctest --print-label-summary
```

Test suites:
- **Unit tests** (`tests/unit/`): Boost unit tests + LLVM/MLIR LIT tests
- **Pipeline tests** (`tests/pipeline/`): Integration tests with YAML pipelines
- **Python tests** (`tests/pypeline/`): pytest for Python/C++ integration
- **ABI tests** (`tests/abi/`): ABI analysis tests

## Code Conventions

### Style Enforcement

```bash
# Check and format code (uses clang-format, cmake-format, black, isort, prettier)
./libexec/revng/check-conventions --force-format <files>

# Without modification (dry run)
./libexec/revng/check-conventions <files>
```

Configuration: `share/revng/rcc-config.yml`, `.clang-tidy`

### Naming Conventions

- **Classes, types, variables, members**: `CamelCase`
- **Functions**: `camelBack`
- **Enums**: `CamelCase`

### Required Patterns

- **Main.cpp files must call `InitRevng`** to initialize LLVM facilities (bypass with `// rcc-ignore: initrevng`)
- **Bash scripts must use `set -euo pipefail`** (bypass with `# rcc-ignore: bash-set-flags`)
- **All files require MIT license headers** (except .txt, .md, .rst, .dot, and other exceptions in rcc-config.yml)
- **Use `revng::IRBuilder`** instead of plain `IRBuilder` to ensure proper wrappers
- **Use `revng_{assert,check,abort,unreachable}`** instead of standard assert/abort/llvm_unreachable

### Code Style Rules

- **Line length**: 80 characters maximum
- **No RTTI, no exceptions**: `-fno-rtti -fno-exceptions`
- **Standard**: C++20 with `-Wall -Wextra -Werror`
- **Includes**: Use `<...>` only for C++ standard library, use `"..."` for project headers
- **Headers**: No `static` declarations in headers

### Forbidden Patterns

The following patterns are flagged by `revng-check-conventions`:
- `;;` (double semicolons)
- `Twine` passed by value (must be passed by reference)
- `#pragma clang optimize off`
- Direct use of `verifyModule()`, `verifyFunction()`
- `clang-format off/on` comments
- Starting lines with operators (`==`, `!=`, `>`, `>=`, `<=`, `*>`, `/`, etc.)
- Ending lines with `::`, `} else`, template brackets in certain contexts

## High-Level Architecture

### Pipeline Framework (Core Orchestration)

The **Pipeline** system is the heart of rev.ng, managing all analysis and transformation steps:

- **Containers**: In-memory data structures (Model, LLVM modules, C code)
- **Pipes**: Processing units that consume and produce containers
- **Targets**: Desired outputs specified in YAML
- **Invalidation**: Smart dependency tracking and caching

Pipeline execution tools:
- `revng pipeline` - Run complete pipeline
- `revng pipe` - Execute single pipe
- `revng analyze` - Run specific analysis
- `revng artifact` - Manage output artifacts
- `revng invalidate` - Clear caches

Pipeline definition: `share/revng/pipelines/revng-pipelines.yml`

### Five-Layer Architecture

1. **IR Lifting Layer** (`lib/Lift/`)
   - Converts QEMU IR → LLVM IR
   - Architecture-independent representation
   - Generates early-linked modules per architecture

2. **Analysis Passes** (`lib/*Analysis*/`)
   - Function boundary detection
   - Data structure inference (Data Layout Analysis)
   - Register usage tracking
   - ABI analysis
   - Type inference

3. **Transformation Passes** (`lib/*Transform*/`, `lib/Canonicalize/`, `lib/RestructureCFG/`)
   - IR canonicalization
   - Control flow graph restructuring
   - Type shrinking
   - Removal of lifting artifacts
   - Function isolation

4. **C Code Generation** (`lib/Clift*/`, `lib/Clifter/`)
   - LLVM IR → C code
   - Valid, recompilable output
   - C-level transformations
   - Header generation

5. **Model System** (`lib/Model/`)
   - YAML-based binary representation
   - Stores: functions, types, stack frames, ABI info
   - Generated from binary analysis and user annotations
   - Used by all downstream components

### Key Data Structures

**Model (TupleTree-based)**:
- Central representation of binary analysis results
- YAML serialization with auto-generated C++/Python bindings
- Schema-driven (see `include/revng/Model/`)
- Manipulated via `revng analyze` commands

**MetaAddress**:
- Unique addressing for binary locations
- Supports code, data, and special regions
- Used consistently across all analyses
- See `lib/Support/MetaAddress.cpp`

**TupleTree**:
- Schema-based code generation system
- Generates: C++ classes, Python bindings, TypeScript definitions, JSON schemas
- Used for Model, Pipeline definitions, ABI specs
- Scripts: `scripts/tuple_tree_generator/`

## Directory Structure

### Core Libraries (`lib/`)

**Infrastructure**:
- `Support/` - Utilities, assertions, helpers, MetaAddress
- `Model/` - Binary model (functions, types, ABI)
- `Pipeline/`, `Pipes/` - Pipeline execution framework
- `Storage/` - Data persistence

**Analysis** (40+ analysis libraries):
- `ABI/` - ABI inference and definitions
- `DataLayoutAnalysis/` - Data structure recovery
- `EarlyFunctionAnalysis/` - Function boundary detection
- `BasicAnalyses/` - Core LLVM analyses
- `RegisterUsageAnalyses/` - Register tracking

**Transformation**:
- `Lift/` - QEMU to LLVM IR lifting
- `Canonicalize/` - IR normalization
- `RestructureCFG/` - Control flow restructuring
- `RemoveLiftingArtifacts/` - Cleanup pass
- `FunctionIsolation/` - Function extraction

**Code Generation**:
- `Clift/` - C lifting core
- `CliftEmitC/` - C code emission
- `Clifter/` - C lifting orchestrator
- `Recompile/` - IR recompilation

### Tools (`tools/`)

- `pipeline/` - Main CLI: `revng pipeline`, `revng pipe`, `revng analyze`, `revng artifact`
- `model/` - Model manipulation utilities
- `trace/` - Binary tracing
- `clift-opt/` - C lifting optimizer

### Configuration & Data (`share/revng/`)

- `pipelines/` - YAML pipeline definitions
- `early-linked.c`, `support.c` - Architecture-specific helper code
- `rcc-config.yml` - Convention checker configuration
- `cmake/` - CMake modules
- `well-known-models/` - Pre-analyzed test binaries

## Common Development Workflows

### Working with the Model

```bash
# Initialize model from binary
revng analyze import-binary -o model.yml binary

# Validate model
revng analyze validate-model model.yml

# Run specific analysis
revng analyze <analysis-name> model.yml binary

# Export to C
revng artifact export-c model.yml binary > output.c
```

### Running the Pipeline

```bash
# Complete analysis pipeline
revng pipeline revng-default --verbose model.yml binary

# Run with specific target
revng pipeline --target some-artifact model.yml binary
```

### Adding New Analysis Passes

New analyses are typically added as:
1. Library in `lib/YourAnalysis/` with CMakeLists.txt
2. Headers in `include/revng/YourAnalysis/`
3. Registration in pipeline YAML
4. Unit tests in `tests/unit/`

Analyses must:
- Inherit from appropriate LLVM pass base classes
- Use `revng::` utilities for IR manipulation
- Follow TupleTree patterns for model updates
- Include MIT license headers

### Python Bindings

Python bindings are auto-generated from TupleTree schemas:
- Source: `python/revng/`
- Generated: Build outputs Python modules
- Usage: `import revng.model`, `import revng.pipeline`

## Important Implementation Notes

### MetaAddress Usage

Always use `MetaAddress` for binary locations, never raw addresses. It provides:
- Uniform addressing across code/data
- Support for special regions
- Serialization/deserialization
- See `include/revng/Support/MetaAddress.h`

### Model Modifications

When modifying the Model:
1. Use TupleTree accessors
2. Maintain invariants (documented in model schema)
3. Trigger proper invalidation in pipelines
4. Validate after modifications

### LLVM IR Patterns

- Use `revng::IRBuilder` wrapper (not plain `llvm::IRBuilder`)
- Use `revng::FunctionTags` for function metadata
- Use `revng_assert` for invariants
- Follow LLVM coding standards with project-specific conventions

### Multi-Architecture Support

The framework supports: x86_64, i386, aarch64, arm, mips, mipsel, s390x

Architecture-specific code:
- Early-linked modules per architecture
- ABI definitions in `lib/ABI/`
- Register state deductions
- Generated at build time from `share/revng/early-linked.c`

## Debugging and Development

### Enable Debug Logging

```bash
# Set debug log topics
export REVNG_OPTIONS="--debug-log=verify,pipeline,model"
```

Available topics defined in respective components.

### Common Issues

- **Build failures**: Ensure LLVM version matches (check CMakeLists.txt for required version)
- **Test failures**: Run with `ctest -VV` for verbose output
- **Convention check failures**: Run `./libexec/revng/check-conventions --force-format` to auto-fix
- **Pipeline failures**: Check model validity with `revng analyze validate-model`

## Resources

- **Documentation**: `share/doc/revng/` (build with `make mkdocs`)
- **Pipeline definitions**: `share/revng/pipelines/revng-pipelines.yml`
- **Convention config**: `share/revng/rcc-config.yml`
- **Clang-tidy rules**: `.clang-tidy`
- **Online docs**: https://docs.rev.ng
- **Community**: Discord (https://discord.gg/wEQtgKJxcX), Discourse (https://discuss.rev.ng/)
