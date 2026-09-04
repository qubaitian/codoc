# codoc - code + doc

Codoc generates source files from marked Markdown code blocks.

## Marker

Put metadata on the opening fence:

````markdown
[Purpose in one sentence. Split the block if that fails.](src/test.py#L3-L10)
```python
# Compute the factorial.
def factorial(n):
    # Stop the recursion.
    if n <= 1:
        return 1

    # Compute the recursive product.
    return n * factorial(n - 1)
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
