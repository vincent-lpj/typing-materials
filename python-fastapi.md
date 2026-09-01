from enum import Enum
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, Path, Query, status
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field


app = FastAPI(
    title="Typing Practice API",
    version="1.0.0",
    description="A small Task API for practicing FastAPI and Python typing.",
)


class SortOrder(str, Enum):
    asc = "asc"
    desc = "desc"


class TaskCreate(BaseModel):
    title: str = Field(min_length=3, max_length=80)
    priority: int = Field(default=3, ge=1, le=5)
    tags: list[str] = Field(default_factory=list)


class TaskUpdate(BaseModel):
    title: str | None = Field(default=None, min_length=3, max_length=80)
    priority: int | None = Field(default=None, ge=1, le=5)
    tags: list[str] | None = None
    completed: bool | None = None


class Task(BaseModel):
    id: int
    title: str
    priority: int
    tags: list[str]
    completed: bool = False


tasks: dict[int, Task] = {
    1: Task(
        id=1,
        title="Practice FastAPI path parameters",
        priority=4,
        tags=["python", "fastapi"],
    ),
    2: Task(
        id=2,
        title="Review HTTP status codes",
        priority=3,
        tags=["api", "http"],
        completed=True,
    ),
    3: Task(
        id=3,
        title="Type query validation examples",
        priority=5,
        tags=["python", "typing"],
    ),
}


def get_task_or_404(
    task_id: Annotated[int, Path(ge=1, description="Positive task ID")],
) -> Task:
    task = tasks.get(task_id)

    if task is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Task {task_id} was not found.",
        )

    return task


def get_page(
    skip: Annotated[int, Query(ge=0)] = 0,
    limit: Annotated[int, Query(ge=1, le=50)] = 10,
) -> tuple[int, int]:
    return skip, limit


@app.get("/")
async def read_root() -> dict[str, str]:
    return {
        "message": "FastAPI typing practice is running.",
        "docs": "/docs",
    }


@app.get("/tasks", response_model=list[Task])
async def list_tasks(
    page: Annotated[tuple[int, int], Depends(get_page)],
    q: Annotated[
        str | None,
        Query(min_length=2, max_length=30, description="Search task titles"),
    ] = None,
    completed: bool | None = None,
    sort: SortOrder = SortOrder.asc,
) -> list[Task]:
    skip, limit = page
    result = list(tasks.values())

    if q is not None:
        keyword = q.casefold()
        result = [task for task in result if keyword in task.title.casefold()]

    if completed is not None:
        result = [task for task in result if task.completed is completed]

    result.sort(
        key=lambda task: (task.priority, task.id),
        reverse=sort is SortOrder.desc,
    )

    return result[skip : skip + limit]


@app.get("/tasks/{task_id}", response_model=Task)
async def read_task(
    task: Annotated[Task, Depends(get_task_or_404)],
) -> Task:
    return task


@app.post(
    "/tasks",
    response_model=Task,
    status_code=status.HTTP_201_CREATED,
)
async def create_task(payload: TaskCreate) -> Task:
    next_id = max(tasks, default=0) + 1

    task = Task(
        id=next_id,
        title=payload.title,
        priority=payload.priority,
        tags=payload.tags,
    )
    tasks[next_id] = task

    return task


@app.patch("/tasks/{task_id}", response_model=Task)
async def update_task(
    payload: TaskUpdate,
    task: Annotated[Task, Depends(get_task_or_404)],
) -> Task:
    changes = payload.model_dump(exclude_unset=True, exclude_none=True)
    updated_task = task.model_copy(update=changes)
    tasks[task.id] = updated_task

    return updated_task


@app.delete(
    "/tasks/{task_id}",
    status_code=status.HTTP_204_NO_CONTENT,
)
async def delete_task(
    task: Annotated[Task, Depends(get_task_or_404)],
) -> None:
    del tasks[task.id]


def run_self_check() -> None:
    client = TestClient(app)

    response = client.get("/")
    assert response.status_code == 200
    assert response.json()["docs"] == "/docs"

    response = client.get(
        "/tasks",
        params={"skip": 0, "limit": 2, "sort": "desc"},
    )
    assert response.status_code == 200
    assert len(response.json()) == 2

    response = client.post(
        "/tasks",
        json={
            "title": "Practice brackets and quotes",
            "priority": 5,
            "tags": ["typing", "symbols"],
        },
    )
    assert response.status_code == 201

    task_id = response.json()["id"]

    response = client.patch(
        f"/tasks/{task_id}",
        json={"completed": True, "priority": 4},
    )
    assert response.status_code == 200
    assert response.json()["completed"] is True

    response = client.get("/tasks/9999")
    assert response.status_code == 404
    assert response.json() == {"detail": "Task 9999 was not found."}

    response = client.delete(f"/tasks/{task_id}")
    assert response.status_code == 204

    print("Self-check passed: all FastAPI examples work correctly.")


if __name__ == "__main__":
    run_self_check()
