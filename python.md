from __future__ import annotations

import json
import re
from dataclasses import asdict, dataclass
from datetime import datetime, timezone
from pathlib import Path
from statistics import mean


DATA_DIR = Path("practice_data")
TASK_FILE = DATA_DIR / "tasks.json"


@dataclass(slots=True)
class Task:
    id: int
    title: str
    priority: int
    completed: bool = False
    tags: list[str] | None = None

    def __post_init__(self) -> None:
        if self.priority not in {1, 2, 3, 4, 5}:
            raise ValueError("priority must be between 1 and 5")

        if self.tags is None:
            self.tags = []

    def label(self) -> str:
        status = "done" if self.completed else "open"
        return f"#{self.id:03d} [{status}] P{self.priority} - {self.title}"


def ensure_data_dir() -> None:
    DATA_DIR.mkdir(parents=True, exist_ok=True)


def normalize_title(title: str) -> str:
    cleaned = re.sub(r"\s+", " ", title.strip())

    if len(cleaned) < 3:
        raise ValueError("title must contain at least 3 characters")

    return cleaned


def task_to_dict(task: Task) -> dict[str, object]:
    return asdict(task)


def task_from_dict(data: dict[str, object]) -> Task:
    return Task(
        id=int(data["id"]),
        title=str(data["title"]),
        priority=int(data["priority"]),
        completed=bool(data.get("completed", False)),
        tags=[str(tag) for tag in data.get("tags", [])],
    )


def save_tasks(tasks: list[Task], path: Path = TASK_FILE) -> None:
    ensure_data_dir()

    payload = {
        "updated_at": datetime.now(timezone.utc).isoformat(),
        "count": len(tasks),
        "tasks": [task_to_dict(task) for task in tasks],
    }

    path.write_text(
        json.dumps(payload, indent=2, ensure_ascii=False),
        encoding="utf-8",
    )


def load_tasks(path: Path = TASK_FILE) -> list[Task]:
    if not path.exists():
        return []

    try:
        payload = json.loads(path.read_text(encoding="utf-8"))
    except json.JSONDecodeError as exc:
        raise ValueError(f"Invalid JSON in {path}: {exc}") from exc

    raw_tasks = payload.get("tasks", [])

    if not isinstance(raw_tasks, list):
        raise TypeError("'tasks' must be a list")

    return [task_from_dict(item) for item in raw_tasks if isinstance(item, dict)]


def add_task(
    tasks: list[Task],
    title: str,
    priority: int,
    tags: list[str] | None = None,
) -> Task:
    next_id = max((task.id for task in tasks), default=0) + 1

    task = Task(
        id=next_id,
        title=normalize_title(title),
        priority=priority,
        tags=tags or [],
    )
    tasks.append(task)

    return task


def find_task(tasks: list[Task], task_id: int) -> Task:
    for task in tasks:
        if task.id == task_id:
            return task

    raise LookupError(f"Task {task_id} was not found")


def complete_task(tasks: list[Task], task_id: int) -> Task:
    task = find_task(tasks, task_id)
    task.completed = True
    return task


def filter_tasks(
    tasks: list[Task],
    *,
    completed: bool | None = None,
    minimum_priority: int = 1,
    tag: str | None = None,
) -> list[Task]:
    result = [
        task
        for task in tasks
        if task.priority >= minimum_priority
    ]

    if completed is not None:
        result = [
            task
            for task in result
            if task.completed is completed
        ]

    if tag is not None:
        result = [
            task
            for task in result
            if tag in task.tags
        ]

    return result


def sort_tasks(
    tasks: list[Task],
    *,
    descending: bool = True,
) -> list[Task]:
    return sorted(
        tasks,
        key=lambda task: (task.priority, task.id),
        reverse=descending,
    )


def summarize_tasks(tasks: list[Task]) -> dict[str, float | int]:
    total = len(tasks)
    completed = sum(task.completed for task in tasks)
    open_count = total - completed
    average_priority = mean(task.priority for task in tasks) if tasks else 0.0

    return {
        "total": total,
        "completed": completed,
        "open": open_count,
        "average_priority": round(average_priority, 2),
    }


def print_report(tasks: list[Task]) -> None:
    print("\n=== Task Report ===")

    for task in sort_tasks(tasks):
        tags = ", ".join(task.tags) if task.tags else "-"
        print(f"{task.label()} | tags={tags}")

    summary = summarize_tasks(tasks)

    print(
        "\nSummary:",
        f"total={summary['total']},",
        f"completed={summary['completed']},",
        f"open={summary['open']},",
        f"avg_priority={summary['average_priority']}",
    )


def build_sample_tasks() -> list[Task]:
    tasks: list[Task] = []

    add_task(
        tasks,
        "Practice brackets, quotes, and underscores",
        priority=5,
        tags=["typing", "symbols"],
    )
    add_task(
        tasks,
        "Review list and dictionary comprehensions",
        priority=4,
        tags=["python", "collections"],
    )
    add_task(
        tasks,
        "Write JSON files with pathlib",
        priority=3,
        tags=["json", "files"],
    )
    add_task(
        tasks,
        "Practice regular expressions",
        priority=2,
        tags=["regex", "strings"],
    )

    complete_task(tasks, 2)

    return tasks


def run_self_check() -> None:
    tasks = build_sample_tasks()

    assert len(tasks) == 4
    assert tasks[0].priority == 5
    assert tasks[1].completed is True

    high_priority = filter_tasks(
        tasks,
        completed=False,
        minimum_priority=4,
    )
    assert [task.id for task in high_priority] == [1]

    tagged = filter_tasks(
        tasks,
        tag="python",
    )
    assert len(tagged) == 1
    assert tagged[0].title == "Review list and dictionary comprehensions"

    save_tasks(tasks)

    loaded = load_tasks()
    assert len(loaded) == len(tasks)
    assert loaded[0].title == tasks[0].title
    assert loaded[1].completed is True

    print_report(loaded)
    print("\nSelf-check passed: all examples work correctly.")


if __name__ == "__main__":
    run_self_check()
