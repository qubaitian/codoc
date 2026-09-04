# Keep target files inside the project root

Codoc resolves target paths from the project root.
It rejects absolute paths and traversal outside that root.
Symlink targets must also remain inside the project root.
We choose this boundary to prevent accidental writes elsewhere.
