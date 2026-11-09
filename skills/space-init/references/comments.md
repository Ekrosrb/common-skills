# Comments

Applies before writing or changing any comment.

## Project style comes first

New code should read like the code around it, so check how this project comments before
adding anything. Look at the file you are changing and a few comparable files nearby (same
module, same kind of class), not just one.

- If the project has almost no comments, write none. That holds for class and method level
  too, even where the defaults below would allow one.
- If the project comments only in certain places (for example, Javadoc on public
  interfaces but not on implementations), comment only in those places.
- Where the project has a clear style, follow it: the same mechanism, placement, length
  and tone. Where it conflicts with the defaults below, the project wins.

## Defaults

Use these where the project comments but its style does not settle the point.

- Never write line-level comments.
- Comment only at class or method level, using the language's documented mechanism
  (Javadoc, docstrings, and equivalents).
- Describe what the class or method does, not how the current implementation works.
- No change notes, implementation notes, or TODOs in comments.
- Keep them short and precise.
- Add none for simple or self-explanatory code. A comment earns its place only when it
  carries meaning the signature does not.
