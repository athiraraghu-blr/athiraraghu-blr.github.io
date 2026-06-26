Title: Python Best Practices for Clean and Maintainable Code
Date: 2026-06-26
Category: Article
Tags: Python, coding
Slug: best-practice-for-clean-and-maintanable-code


Writing Python code that works is one thing. Writing code that your future self — or a teammate — can actually understand, extend, and trust six months from now is another skill entirely. This article covers the practices that separate good Python from great Python.


# **Follow PEP 8 — But Don't Obsess Over It**

PEP 8 is Python's official style guide, and most of its rules exist for good reason. The big ones worth internalizing:

1. Use 4 spaces for indentation (never tabs)
2. Keep lines under 79 characters (or 88–99 if your team prefers Black's defaults)
3. Two blank lines between top-level functions and classes; one blank line between methods
4. Use snake_case for variables and functions, PascalCase for classes, UPPER_CASE for constants

That said, consistency within a codebase matters more than strict PEP 8 compliance. If you join an existing project with a different convention, match it. Use a formatter like **Black or Ruff** to automate this entirely so you never argue about style again.

python

    # Not this
    def calculateTotalPrice(item_list,tax_rate):
        total=0
        for i in item_list:total+=i['price']
        return total*(1+tax_rate)

    # This
    def calculate_total_price(items: list[dict], tax_rate: float) -> float:
        total = sum(item["price"] for item in items)
        return total * (1 + tax_rate)


# **Write Meaningful Names**

The single highest-leverage habit in readable code is naming things well. A good name makes a comment unnecessary.

python

    # Bad
    def proc(d, n):
        return [x for x in d if x > n]

    # Good
    def filter_above_threshold(data: list[float], threshold: float) -> list[float]:
        return [value for value in data if value > threshold]

Rules of thumb:

1. Variables and functions should describe what they are or what they do
2. Avoid single-letter names except in short loops (i, j) or math contexts (x, y)
3. Boolean variables should read like questions: is_active, has_permission, was_updated
4. Don't abbreviate unless the abbreviation is universally understood (url, id, cfg)


# **Use Type Hints**

Type hints, introduced in Python 3.5 and steadily improved since, are one of the best tools for making code self-documenting and catching bugs before runtime.

python

    from collections.abc import Sequence

    def get_user_emails(user_ids: Sequence[int], active_only: bool = True) -> list[str]:
        ...

Type hints don't enforce anything at runtime, but they enable static analysis tools like **mypy and Pyright**, and they make function signatures immediately understandable without reading the implementation. Treat them as documentation that can be verified.

For complex types, the typing module (and Python 3.10+'s built-in syntax) gives you Optional, Union, Literal, TypedDict, and more:

python

    # Python 3.10+
    def process(value: int | str | None) -> str:
        ...


# **Embrace Python's Built-ins and Standard Library**

Experienced Python developers reach for built-ins and standard library tools instinctively. Knowing these well makes your code shorter, faster, and easier to read.

**Comprehensions over manual loops:**

python

    # Fine
    results = []
    for user in users:
        if user.is_active:
            results.append(user.email)

    # Better
    results = [user.email for user in users if user.is_active]

**enumerate** instead of manual indexing:

python

    for i, item in enumerate(items):
        print(f"{i}: {item}")

**zip** for parallel iteration:

python

    for name, score in zip(names, scores):
        print(f"{name}: {score}")

**collections.defaultdict and Counter:**

python

    from collections import Counter

    word_counts = Counter(text.split())
    most_common = word_counts.most_common(10)

**pathlib instead of os.path:**

python

    from pathlib import Path

    config_path = Path.home() / ".config" / "myapp" / "settings.json"
    if config_path.exists():
        data = config_path.read_text()


# **Handle Exceptions Properly**

Swallowing exceptions silently is one of the most common sources of bugs that are hard to diagnose.

python

    # Never do this
    try:
        result = process(data)
    except Exception:
        pass  # bugs disappear here

    # Don't do this either — too broad
    try:
        result = process(data)
    except Exception as e:
        print(f"Something went wrong: {e}")

    # Do this — catch specific exceptions
    try:
        result = int(user_input)
    except ValueError:
        raise ValueError(f"Expected an integer, got: {user_input!r}")

A few principles:

1. Catch the most specific exception type possible
2. Re-raise with context using raise ... from ... when wrapping exceptions
3. Use finally for cleanup, or better yet, context managers (with statements)
4. Don't use exceptions for flow control when a simple conditional works


# **Write Functions That Do One Thing**

Functions should be short, focused, and named after what they do. If you find yourself writing a function that has "and" in its description, it's doing too much.

python

    # Doing too much
    def process_order(order):
        # validate
        if not order.get("items"):
            raise ValueError("Order has no items")
        # calculate
        total = sum(item["price"] * item["qty"] for item in order["items"])
        # apply discount
        if order.get("coupon") == "SAVE10":
            total *= 0.9
        # save to DB
        db.save({"order": order, "total": total})
        # send email
        email.send(order["customer_email"], f"Your total is ${total:.2f}")

    # Better — each function does one thing
    def validate_order(order: dict) -> None:
        if not order.get("items"):
            raise ValueError("Order has no items")

    def calculate_total(order: dict) -> float:
        subtotal = sum(item["price"] * item["qty"] for item in order["items"])
        discount = 0.1 if order.get("coupon") == "SAVE10" else 0.0
        return subtotal * (1 - discount)

    def save_order(order: dict, total: float) -> None:
        db.save({"order": order, "total": total})

    def notify_customer(email_address: str, total: float) -> None:
        email.send(email_address, f"Your total is ${total:.2f}")

# **Use Context Managers for Resource Management**

Whenever you open a file, acquire a lock, or manage any resource that needs cleanup, use a with statement.

python

    # Risky — file may not close if an exception occurs
    f = open("data.txt")
    data = f.read()
    f.close()

    # Correct
    with open("data.txt", encoding="utf-8") as f:
        data = f.read()

You can also write your own context managers using contextlib.contextmanager:

python

    from contextlib import contextmanager

    @contextmanager
    def temporary_directory():
        path = Path(tempfile.mkdtemp())
        try:
            yield path
        finally:
            shutil.rmtree(path)

# **Write Tests**

Untested code is code you can't safely change. Python's ecosystem makes testing straightforward — **pytest** is the de facto standard.

python

    # my_math.py
    def divide(a: float, b: float) -> float:
        if b == 0:
            raise ZeroDivisionError("Cannot divide by zero")
        return a / b


    # test_my_math.py
    import pytest
    from my_math import divide

    def test_divide_normal():
        assert divide(10, 2) == 5.0

    def test_divide_by_zero():
        with pytest.raises(ZeroDivisionError):
            divide(10, 0)

    def test_divide_floats():
        assert divide(1, 3) == pytest.approx(0.333, rel=1e-3)

Start with tests for your most critical logic. As a rule: if you're afraid to change a piece of code without breaking something, that's a sign it needs tests.


# **Use Dataclasses and Named Structures**

Avoid using plain tuples or dicts to pass related data around. Structured types make code far easier to read and maintain.

python

    # Avoid
    def create_user(name, email, age, is_admin=False):
        return (name, email, age, is_admin)  # what does index 2 mean again?

    # Use dataclasses
    from dataclasses import dataclass, field

    @dataclass
    class User:
        name: str
        email: str
        age: int
        is_admin: bool = False
        permissions: list[str] = field(default_factory=list)

For immutable data, @dataclass(frozen=True) works well. For validated data with complex constraints, consider **Pydantic.**


# **Keep Dependencies Explicit and Environments Isolated**

Use virtual environments — always. Never install packages into your system Python.

python

    python -m venv .venv
    source .venv/bin/activate   # or .venv\Scripts\activate on Windows
    pip install -r requirements.txt

Pin your dependencies in a requirements.txt (or better, use pyproject.toml with a tool like Poetry or uv). Unpinned dependencies lead to "it works on my machine" problems.

Document what each dependency is for. If you add a package, note in your PR or commit why it's needed.


# **Document Thoughtfully**

Good code should be mostly self-documenting through clear naming and structure. Comments and docstrings fill the gap for why, not what.

python

    # Bad comment — just restates the code
    x = x + 1  # increment x by 1

    # Good comment — explains the reason
    x = x + 1  # offset by 1 because the API uses 1-based indexing

For public functions and classes, write docstrings:

python

    def retry(func, max_attempts: int = 3, delay: float = 1.0):
        """
        Call `func` repeatedly until it succeeds or max_attempts is reached.

        Args:
            func: A callable that may raise an exception.
            max_attempts: Maximum number of tries before re-raising the last exception.
            delay: Seconds to wait between attempts.

        Returns:
            The return value of `func` on success.

        Raises:
            The last exception raised by `func` if all attempts fail.
        """
        ...

# **Putting It Together**

Clean Python code is not about following every rule perfectly — it's about making considered choices that reduce cognitive load for the next person who reads your code (often yourself). Start with the habits that give you the most immediate value: meaningful names, type hints, small focused functions, and tests. The rest follows naturally.

The best Python developers aren't the ones who know the most obscure features — they're the ones who make their intent obvious and their code easy to change.