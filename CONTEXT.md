# Codoc

Codoc connects documented code blocks to locations in source files.  

## Language

**Purpose link**:  
A link above a code block whose text states the purpose of that code.  

**Target range**:  
The inclusive, one-based line interval identified by a purpose link.  

**Mapped block**:  
A fenced code block associated with a purpose link containing a target range.  

**Target file**:  
The local code file identified by a mapped block's purpose link.  
Its entire content is defined by the document's mapped blocks and the blank lines between their target ranges.  
