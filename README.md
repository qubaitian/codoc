# codoc - code + doc

Codoc generates source files from marked Markdown code blocks.

## Marker

Put metadata on the opening fence:

````markdown
```(:lang commonlisp :path src/package.lisp :start 1 :end 24)
(defpackage #:example)
```
````

Codoc fills missing lines with blank lines.

## Command

Run `codoc` from the project directory.

It reads `./codoc.md` by default.

Pass one Markdown path to use another source document.

```sh
./codoc
./codoc path/to/document.md
```
