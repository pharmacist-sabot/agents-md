# Update `README.md` to Reflect Production Standards

## 1. Overview

The `README.md` must be updated to reflect the significant architectural and functional improvements made to the `pdf-splitter` utility (CLI Argument Parsing, Modularity, Optimization). A world-class `README` is the primary documentation for an open-source project, and it must be accurate, clear, and professional.

## 2. Technical Specifications

The developer must completely rewrite the `README.md` file with the following structure and content:

### 2.1. New `README.md` Content

```markdown
# pdf-splitter

A **fast, robust, and production-ready** command-line utility written in Rust to split multi-page PDF documents into individual pages. Optimized for performance using multi-threading (`rayon`).

## Features

- 📄 Splits a multi-page PDF into separate single-page PDF files.
- 🚀 **High Performance:** Utilizes multi-threading (`rayon`) for parallel page processing.
- 🛡️ **Secure:** Implements secure path handling to prevent Path Traversal vulnerabilities.
- ⚙️ **Optimized Output:** Automatically prunes unused objects for smaller, cleaner output files.
- 📁 Organizes output with zero-padded filenames (e.g., `page_001.pdf`).

## Installation

### From Source

Ensure you have [Rust](https://www.rust-lang.org/tools/install) installed (Rust 2024 Edition or newer), then:

```bash
git clone https://github.com/suradet-ps/pdf-splitter.git
cd pdf-splitter
cargo build --release
```

The binary will be available at `target/release/pdf-splitter`.

## Usage

The `pdf-splitter` tool requires two main arguments: the input file path and the output directory path.

### Basic Command

Split a PDF file and save the output to a specified directory.

```bash
./target/release/pdf-splitter --input /path/to/your/document.pdf --output ./split_pages
```

### Argument Details

| Option | Short | Description | Example |
| :--- | :--- | :--- | :--- |
| `--input <FILE>` | `-i` | **Required.** Path to the multi-page PDF file to be split. | `-i report.pdf` |
| `--output <DIR>` | `-o` | **Optional.** Path to the directory where split pages will be saved. **Defaults to `output_pages`**. | `-o /tmp/output` |

### Example Output

If `document.pdf` has 10 pages, the output will look like this:

```
Starting to split: /path/to/your/document.pdf
✅ Successfully split 10 pages into 'split_pages'
```

The output directory structure:

```
split_pages/
├── page_001.pdf
├── page_002.pdf
├── page_003.pdf
└── ...
```

## Dependencies

| Crate | Version | Description |
| :--- | :--- | :--- |
| [`lopdf`](https://crates.io/crates/lopdf) | 0.39.0 | Core PDF document manipulation library. |
| [`clap`](https://crates.io/crates/clap) | 4.5 | Command-line argument parser. |
| [`thiserror`](https://crates.io/crates/thiserror) | 1.0 | Declarative error handling for robust error types. |
| [`rayon`](https://crates.io/crates/rayon) | 1.10 | Data parallelism library for high-performance processing. |

## Contributing

We welcome contributions to make `pdf-splitter` even better! Please follow these guidelines:

1.  **Fork** the repository and create your feature branch (`git checkout -b feature/AmazingFeature`).
2.  Ensure your code adheres to the project's **Coding Standards** (run `cargo fmt` and `cargo clippy`).
3.  Write **Unit Tests** for all new or changed functionality.
4.  Ensure all existing tests pass (`cargo test`).
5.  Commit your changes (`git commit -m 'Add some AmazingFeature'`).
6.  Push to the branch (`git push origin feature/AmazingFeature`).
7.  Open a **Pull Request** against the `main` branch.

## License

This project is licensed under the **MIT License**. See the `LICENSE` file in the repository for full details.
```

### 2.2. Verification

The implementation is considered complete when the following conditions are met:

1.  **File Overwrite**: The existing `README.md` is completely overwritten with the new content.
2.  **Accuracy**: The `Usage` section accurately reflects the new CLI arguments (`--input`, `--output`) and the removal of the hardcoded `input/sample.pdf` requirement.
3.  **Completeness**: The `Contributing` and `License` sections are present and professionally structured.

## 3. Architectural Rationale

A well-maintained `README.md` is essential for the project's long-term success and adoption. By updating it, we ensure that the external-facing documentation matches the internal, production-grade quality of the code. This is a key step in marketing the project as a reliable, secure, and high-performance utility.
