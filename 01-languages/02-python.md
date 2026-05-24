# 1.2 · Python

## Overview

Python is used in backend contexts for scripting, tooling, data pipelines, and AI/ML workloads. Knowing Python alongside Go broadens the scope of solvable problems.

---

## Core Concepts

- Dynamic typing, duck typing, type hints (`typing` module, `mypy`)
- List/dict/set comprehensions, generators, iterators (`yield`, `__iter__`)
- Decorators (`@property`, `@staticmethod`, `@classmethod`, custom)
- Context managers (`with`, `__enter__`, `__exit__`)
- `*args` and `**kwargs`
- Dunder methods (`__str__`, `__repr__`, `__eq__`, `__hash__`)
- GIL, `threading`, `multiprocessing`, `asyncio`
- `dataclasses`, `pydantic` for structured data

---

## Key Commands / Code Snippets

```python
# Decorator with arguments
def retry(times=3, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == times - 1:
                        raise
        return wrapper
    return decorator

# Context manager
class ManagedResource:
    def __enter__(self):
        self.acquire()
        return self
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.release()
        return False  # don't suppress exceptions

# Async HTTP
import asyncio, httpx

async def fetch_all(urls: list[str]) -> list[str]:
    async with httpx.AsyncClient() as client:
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)
        return [r.text for r in responses]

# Generator for large files
def read_chunks(path: str, size: int = 1024):
    with open(path, "rb") as f:
        while chunk := f.read(size):
            yield chunk
```

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pytest -v --tb=short
mypy src/
ruff check . && black --check .
```

---

## Common Interview Questions

**Q: What is the GIL and when does it matter?**
> The GIL allows only one thread to execute Python bytecode at a time. For CPU-bound work use `multiprocessing`; for I/O-bound use `threading` or `asyncio`.

**Q: How do generators differ from lists?**
> Generators are lazy — they produce values on demand, keeping only one value in memory at a time. Ideal for large datasets or infinite sequences.

**Q: What is `asyncio` and when would you use it?**
> `asyncio` enables cooperative multitasking using an event loop. Use for I/O-bound workloads (HTTP calls, DB queries) where you'd otherwise block waiting. Not useful for CPU-bound tasks.

---

## Gotchas & Best Practices

- Mutable default arguments: use `None`, initialize inside the function body
- `is` vs `==`: `is` checks identity, `==` checks equality — only use `is` for `None`
- `asyncio.run()` cannot be nested inside a running event loop (use `nest_asyncio` in notebooks)
- List comprehensions are faster than equivalent `for` + `append` loops
- Prefer `pathlib.Path` over `os.path` for file operations
- Use `__slots__` on data-heavy classes to reduce memory usage

---

## Resources

- [ ] [Python Docs](https://docs.python.org/3/)
- [ ] [Fluent Python (2nd ed.)](https://www.oreilly.com/library/view/fluent-python-2nd/9781492056348/)
- [ ] [Real Python](https://realpython.com/)
- [ ] [FastAPI Docs](https://fastapi.tiangolo.com/)

---

## My Notes

<!-- ADD YOUR PERSONAL NOTES HERE -->

