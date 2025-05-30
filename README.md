# Cetus Compiler Transformation Test Harness

This repository contains a robust testing framework designed to evaluate the behavior and correctness of Cetus for loop parallelization. The harness automates the process of preprocessing, transforming, and comparing code, logging outcomes, and identifying different types of failures such as crashes, diff mismatches, or missed optimization opportunities.

---

## Table of Contents

* [Introduction](#introduction)
* [Core Components](#core-components)
    * `main.c` (Test Runner Logic)
    * `helper_tests.h` (Declarations & Configuration)
    * `check_syntax.sh` (Semantic Check Script)
* [How It Works](#how-it-works)
    * Preprocessing (`.c` to `.i`)
    * Semantic Check
    * Cetus Transformation (`.i` to transformed `.i`)
    * Ground Truth Handling
    * Comparison
    * Logging
* [Setup and Dependencies](#setup-and-dependencies)
    * Clang/LLVM
    * Cetus
    * GNU Diff
    * Directory Structure
* [Compilation](#compilation)
* [Running Tests](#running-tests)
    * Test Modes
    * Expected Behavior
    * Examples
* [Understanding Test Outcomes](#understanding-test-outcomes)
* [Log Files](#log-files)
* [Customization](#customization)
* [Journey & Insights](#journey--insights)

---

## Introduction

Evaluating source-to-source transformation tools, especially those involving complex code analysis and modifications like parallelization, can be challenging. Manual verification is tedious and error-prone. This test harness provides an automated solution to systematically test such tools.

It takes a C source file, preprocesses it, feeds it to the transformation tool (Cetus in this case), and then compares the tool's output against a predefined "ground truth" version of the code. This allows for automated detection of functional correctness, missed optimizations, or unintended transformations.

---

## Core Components

### `main.c` (Test Runner Logic)

This is the heart of the test harness. It contains the `main` function and the `run_test_case` function, which orchestrates the entire testing process for a single test file.

Key responsibilities include:

* **Argument Parsing**: Handles command-line arguments for test mode (`compare`/`generate`), test category, input file, and expected parallelization behavior.
* **Directory Management**: Creates necessary directories for intermediate preprocessed files (`cetus_intermediate_i_files`), transformed output (`cetus_transformed_output`), and ground truth files (`ground_truth`).
* **Preprocessing**: Uses `clang` to preprocess the input `.c` file into a `.i` file, which is a standard preprocessed C source.
* **Semantic Check**: Runs a shell script (`check_syntax.sh`) to perform a basic syntax check on the preprocessed code, ensuring it's valid C before passing it to Cetus.
* **Cetus Execution**: Invokes the Cetus compiler with specified flags (`-parallelize-loops=1 -ompGen=1`) to transform the preprocessed `.i` file.
* **Ground Truth Management**:
    * In `GENERATE_MODE`, it copies Cetus's actual output as the new ground truth.
    * In `COMPARE_MODE`, it locates the existing ground truth file.
* **Comparison**: Uses the `diff` utility to compare Cetus's output with the ground truth.
* **Outcome Logging**: Logs the results of each step and the final test outcome (Pass, Fail, Crash, Missed Opportunity, etc.) to various dedicated log files and the console.
* **Error Handling**: Catches and logs errors at each stage, providing detailed reasons for failures.

### `helper_tests.h` (Declarations & Configuration)

This header file defines constants, enumerations, and function declarations used throughout the test harness.

* **`MAX_PATH_LENGTH`**: Defines the maximum length for file paths.
* **`CLANG_PATH`**: **Crucially**, this defines the path to your Clang executable. You **must** adjust this if your Clang installation is in a different location.
* **`CETUS_PATH`**: **Crucially**, this defines the path to your Cetus executable. You **must** adjust this if your Cetus executable is not in the same directory as the test harness.
* **`GROUND_TRUTH_SUFFIX`**: Defines the naming convention for ground truth files (e.g., `_gt.c`).
* **`TestMode` Enum**: Specifies whether to `COMPARE_MODE` (run tests and compare output) or `GENERATE_MODE` (create/update ground truth files).
* **`ExpectedParallelization` Enum**: Specifies the expected behavior of Cetus for a given test case (`EXPECT_PARALLELIZATION`, `EXPECT_NO_PARALLELIZATION`, `EXPECT_FAILURE`). This is used for more nuanced failure classification.
* **`TestOutcome` Enum**: Provides a detailed classification of test results, including specific failure types (e.g., `TEST_FAILED_CRASH`, `TEST_FAILED_MISSED_OPPORTUNITY`).
* **Global Log File Pointers**: Declares external pointers to `FILE` objects used for logging.
* **Helper Function Declarations**: Declares utility functions like `get_current_time`, `file_size`, `execute_command`, and `log_test_outcome`.

### `check_syntax.sh` (Semantic Check Script)

This is a simple shell script used to perform a basic semantic check on the preprocessed `.i` files before they are passed to Cetus. This helps to quickly identify issues that might stem from invalid C syntax rather than Cetus's behavior.

A basic `check_syntax.sh` could look like this:

```bash
#!/bin/bash
# check_syntax.sh
# Checks the syntax of a C file using clang, discarding output.
# Exit code 0 for success, non-zero for failure.

INPUT_FILE="$1"

if [ -z "$INPUT_FILE" ]; then
    echo "Usage: $0 <input_file>"
    exit 1
fi

# Try to compile with clang to check for syntax errors.
# We don't care about the output, just the exit status.
/opt/homebrew/opt/llvm/bin/clang -fsyntax-only "$INPUT_FILE" > /dev/null 2>&1

exit $?
