# Codoc Context

Codoc combines Markdown explanations with generated project files.
This context defines Codoc's domain language.

## Language

**Source document**:
The single Markdown document containing prose and marked code blocks.
_Avoid_: documentation source, source file

**Marked code block**:
A fenced block paired with a target reference.
It contributes text to one target file.
_Avoid_: snippet, example block

**Target file**:
A project file assembled from marked code blocks.
_Avoid_: output document, generated source

**Target range**:
An inclusive line interval inside a target file.
_Avoid_: line span, position

**Target reference**:
A Markdown link naming a target file and target range.
_Avoid_: marker, annotation

**Project root**:
The directory where Codoc runs and resolves target paths.
_Avoid_: document directory, output root

**Generated output**:
The target files produced from one source document.
_Avoid_: build documentation, derived source
