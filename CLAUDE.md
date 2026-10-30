# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A Python MCP server (`FastMCP("docs")`) that exposes document-processing tools (e.g. converting DOCX/PDF to markdown via `markitdown`) to AI assistants.

## Commands

Use `uv` for everything. `uv run` uses `.venv` and syncs it to `uv.lock` on its own, so you don't need to activate the venv.

```bash
uv venv                      # create virtual env (first time only)
uv pip install -e .          # install package in editable mode
uv run main.py               # start the MCP server (stdio)
uv run mcp dev main.py       # run with the MCP Inspector (mcp[cli] is a dependency)
uv run pytest                # run all tests
uv run pytest tests/test_document.py::TestBinaryDocumentToMarkdown::test_binary_document_to_markdown_with_pdf  # single test
```

No linter or formatter is configured.

## Architecture

- `main.py` builds the `FastMCP` instance and registers tools. Registering a tool means importing its plain function from `tools/` and calling `mcp.tool()(fn)`. Tool modules do not import or depend on `mcp`.
- `tools/` holds plain Python functions, one module per domain (`math.py`, `document.py`). Because they are ordinary functions, tests call them directly with no MCP server involved.
- `tools/document.py`: `binary_document_to_markdown(binary_data, file_type)` wraps the bytes in a `BytesIO` and passes `StreamInfo(extension=file_type)` to `MarkItDown.convert`. `file_type` is an extension without the dot (`"docx"`, `"pdf"`). This function is **not yet registered** as an MCP tool, and it has no `Field` descriptions or detailed docstring yet.
- `tests/fixtures/` holds sample `mcp_docs.docx` and `mcp_docs.pdf` files used by the tests.

## Conventions

- Always give every function argument an appropriate type annotation, in tools, helpers and tests alike.

## Defining MCP tools

Write each tool as a plain function in `tools/`, then register it in `main.py`:

```python
from tools.math import add
mcp.tool()(add)
```

FastMCP builds the tool's schema from the function signature and docstring, so both must be thorough. `tools/math.py::add` is the reference example to copy.

**Parameters:** describe every parameter with pydantic `Field`, and give the return type:

```python
from pydantic import Field

def my_tool(
    param1: str = Field(description="Detailed description of this parameter"),
    param2: int = Field(description="Explain what this parameter does"),
) -> ReturnType:
    """Comprehensive docstring here"""
```

**Docstring (it becomes the tool description the model sees):**
1. Begin with a one-line summary.
2. Give a detailed explanation of what the tool does.
3. Explain when to use the tool, and when not to.
4. Include usage examples with expected input/output (doctest-style `>>>` as in `add`).
