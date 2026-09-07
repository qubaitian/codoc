# codoc = code + doc

codoc is a markdown convention.  
codoc is also a tool written in Common Lisp.  

## Write mapped blocks

Write a purpose link above a fenced code block.  

[Compute the factorial of a nonnegative integer.](examples/factorial.lisp#L3-L9)
```commonlisp
;; Compute the factorial.
(defun get-factorial (n)
  ;; Stop the recursion.
  (if (<= n 1)
      1
      ;; Compute the recursive product.
      (* n (get-factorial (- n 1)))))
```

Write a purpose in one sentence.  
Split the code block if one sentence cannot state the purpose.  
codoc get writes the seven code lines above into lines 3 through 9 of `examples/factorial.lisp`.  
codoc validates all mapped blocks before deleting their existing target files.  
codoc recreates each target file using only the document's mapped blocks.  
Unmapped lines before or between target ranges become blank lines.  
Existing content after the last target range is removed.  
Generated files use UTF-8 with LF line endings and a final newline.  

The block below is not written to any file.  

```shell
./codoc get README.md
sbcl --noinform --load examples/factorial.lisp --eval '(format t "~D~%" (get-factorial 5))' --quit
```

This block has no purpose link and no line numbers.  

`codoc del` deletes every mapped target file.  
`codoc get` deletes those files, then recreates them from the document.  
Without a filename, both commands read `./CODOC.md`.  
If `./CODOC.md` is missing, `codoc get` writes `./CODOC.md` and a small greeting example.  
The generated document maps `get-greeting` to `examples/hello.lisp`.  
An existing greeting example is deleted and recreated.  
Pass one filename after the command to read a different Markdown document.  
An explicitly supplied missing document is an error.  

```shell
./codoc get
./codoc get README.md
./codoc del
./codoc del README.md
```

## Build and test

[CODOC.md](CODOC.md) contains the implementation, build scripts, and tests.  
Edit its mapped blocks to maintain the code.  
The existing executable regenerates all four Lisp files from that document.  
Rebuild the executable with SBCL, then use it to regenerate and test the code.  

```shell
./codoc del
./codoc get
sbcl --script src/build.lisp
sbcl --script tests/codoc-tests.lisp
./codoc del
```

## Release

```shell
gh release upload v0.1.0 './codoc#codoc-darwin-arm64' --clobber
```
