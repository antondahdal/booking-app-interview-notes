# Python map — mid-level, from zero

Added 2026-10-26.
Anton has no Python. The target is a mid-level backend interview, the same kind of role as the Java work.
Monday through Thursday stay Java. This file does not move those slots.

## When

| Day | Clock | What |
|---|---|---|
| Friday | Half day, after the Java morning | One or two subjects. He types. |
| Saturday | Full day, about five hours with breaks | The next subjects, still one at a time, then a small program that uses them. |
| Sunday | Off | — |

First Python block: Friday 2026-10-30, after the Week 11 Java morning.
First full day: Saturday 2026-11-01.

Code does not go in `SpringBoot-bookingApp`. A separate practice repo starts on the first Saturday. This file is the checklist.

## How to learn

Learn here, and implement here. He types. One subject, then a check.

The free reference is the official Python tutorial: https://docs.python.org/3/tutorial/
Use it to look up a syntax fact. Do not watch it as a course.

No Udemy as the plan. A video course re-teaches programming he already has, and watching it does not produce answers he can say. If a single topic is stuck, one short official page, then back to typing.

## How a subject closes

Status starts at **Not started**.
**In progress** means the file is open.
**Done** means he wrote it, the check was right, and he can say the interview line without notes.

Wrong answer: correct, stay on that row. Do not open the next row.

## Out of this map

Django, data science, pandas, scrapers, and a rewrite of the booking app.
Those are a later choice. They are not required to talk as a mid-level Python backend.

## A — The language

Java already covers the ideas. These rows are the Python shape of them.

| # | Subject | Done when | Status |
|---|---|---|---|
| 1 | Install, `venv`, run one file | A file prints a line from a virtual env | Not started |
| 2 | Types and variables | `int`, `float`, `str`, `bool`, and `None` are named without guessing | Not started |
| 3 | Strings | Slice, `f` string, and `split` on a real line of text | Not started |
| 4 | Lists and tuples | A list changes. A tuple does not. He says which one a function should return | Not started |
| 5 | Dicts and sets | Lookup by key. A set drops duplicates | Not started |
| 6 | `if`, `for`, `while` | One loop over a list, with a condition inside | Not started |
| 7 | Functions | Parameters, a return value, a default argument | Not started |
| 8 | Comprehensions | A list built in one expression, and the same loop written long-form | Not started |
| 9 | Modules, packages, `pip` | One file imports another. One library is installed into the venv | Not started |
| 10 | Files and `with` | Read a text file. The file closes even when the read fails | Not started |
| 11 | Exceptions | `try` / `except` / `finally`. A bare `except` is refused | Not started |
| 12 | Classes | `self`, `__init__`, one subclass, one method overridden | Not started |
| 13 | Mutability, `is` and `==` | Two names can point at one list. `is` is identity. `==` is value | Not started |
| 14 | Iterators and generators | A generator yields one value at a time. He says why the whole list is not built | Not started |
| 15 | Decorators | A function wraps another function. One example, then stop | Not started |
| 16 | Type hints and dataclasses | A function signature shows the types. A dataclass replaces a hand-written constructor | Not started |
| 17 | `pytest` | One test fails, then passes, on code he wrote | Not started |

## B — A small backend

Start B only after row 17 is Done.

| # | Subject | Done when | Status |
|---|---|---|---|
| 18 | FastAPI, one resource | `GET` and `POST` on one path, run locally | Not started |
| 19 | Pydantic | A bad body is rejected before the function runs | Not started |
| 20 | Config | A secret comes from the environment, not from the file | Not started |
| 21 | One table | Insert and read one row. The SQL is visible | Not started |
| 22 | API tests | A test hits the path with a fake or a test database | Not started |
| 23 | Logging | One request writes one log line with a request id | Not started |
| 24 | GIL, threads, `asyncio` | He says threads do not speed CPU work, and `async` waits on network calls | Not started |

## C — Say it

| # | Subject | Done when | Status |
|---|---|---|---|
| 25 | Project layout | He walks folders: app, tests, requirements, README | Not started |
| 26 | One service, end to end | He explains request, check, save, response, and one failure status | Not started |

## First two sessions

Friday 2026-10-30, half day: rows 1–3.
Saturday 2026-11-01: rows 4–7, then a short program that uses a list and a function. Rows 8 and 9 only if 4–7 are Done.
