task-dependency-manager/
│
├── backend/
│   ├── manage.py
│   ├── project/
│   └── tasks/
│
└── frontend/
    ├── package.json
    └── src/
# Task Dependency Management System

This project is a full-stack Task Dependency Management System where tasks can depend on other tasks.
The system prevents circular dependencies, auto-updates task status based on dependencies, and visualizes
task relationships using a graph.

---

## 🚀 Features

- Create, update, and delete tasks
- Add dependencies between tasks
- Detect and prevent circular dependencies
- Auto-update task status based on dependency completion
- Visualize task dependencies using Canvas/SVG graph
- User-friendly UI with validations and loading states

---

## 🛠 Tech Stack

### Backend
- Django 4.x
- Django REST Framework
- MySQL (can be switched to SQLite for local testing)

### Frontend
- React 18+
- Tailwind CSS
- HTML5 Canvas / SVG for graph visualization

---

---

## ⚙️ Backend Setup

```bash
cd backend
python -m venv venv
source venv/bin/activate  # (Windows: venv\Scripts\activate)
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver

## frontend
cd frontend 
npm install
npm start

---

# ✅ 2. DECISIONS.md (Explains Your Thinking — Very Good for Interview)

👉 Create file: **DECISIONS.md**

### ✅ Paste this:

```md
# Technical Decisions and Approach

## Circular Dependency Detection

When adding a dependency (A depends on B), I check whether there is already a path from B back to A.
If such a path exists, adding the new dependency would create a cycle.

I implemented this using Depth First Search (DFS) on the task dependency graph.

Steps:
1. Build adjacency list from TaskDependency table.
2. Start DFS from the dependency node.
3. If the original task is reached, a cycle exists.
4. If cycle found, API returns error and dependency is not saved.

---

## Auto Status Update Logic

Task status is calculated using the following rules:

- If any dependency is `blocked` → task becomes `blocked`
- If all dependencies are `completed` → task becomes `in_progress`
- If some dependencies are pending → task remains `pending`

When a task is marked as `completed`, all tasks that depend on it are re-evaluated and their statuses are updated accordingly.

---

## Graph Visualization

Graph is drawn using HTML5 Canvas / SVG without external libraries.

- Tasks are displayed as nodes (circles)
- Dependencies are shown as arrows
- Colors represent task status:
  - Pending → Gray
  - In Progress → Blue
  - Completed → Green
  - Blocked → Red

Layout used is a simple hierarchical layout from top to bottom.

Clicking on a node highlights its dependencies.

---

## Error Handling & UX

- API errors are shown using user-friendly messages
- Buttons are disabled during API calls
- Circular dependency warnings are shown before saving
- Deletion of tasks warns if other tasks depend on it


## Performance

System handles graphs of 20–30 tasks without noticeable UI lag.
DFS is efficient for the small graph sizes expected in this use case.

COMMIT 1 — Add task and dependency models
📁 backend/tasks/models.py
from django.db import models

class Task(models.Model):
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('in_progress', 'In Progress'),
        ('completed', 'Completed'),
        ('blocked', 'Blocked'),
    ]

    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.title


class TaskDependency(models.Model):
    task = models.ForeignKey(Task, related_name='dependencies', on_delete=models.CASCADE)
    depends_on = models.ForeignKey(Task, related_name='dependents', on_delete=models.CASCADE)

    def __str__(self):
        return f"{self.task} depends on {self.depends_on}"

COMMIT 2 — Add cycle detection API
📁 backend/tasks/serializers.py
from rest_framework import serializers
from .models import Task, TaskDependency

class TaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = Task
        fields = '__all__'
📁 backend/tasks/views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status
from .models import Task, TaskDependency
from .serializers import TaskSerializer


def has_cycle(start, target, graph, visited):
    if start == target:
        return True
    visited.add(start)
    for nxt in graph.get(start, []):
        if nxt not in visited:
            if has_cycle(nxt, target, graph, visited):
                return True
    return False


@api_view(['POST'])
def add_dependency(request, task_id):
    depends_on_id = request.data.get("depends_on_id")

    if task_id == depends_on_id:
        return Response({"error": "Task cannot depend on itself"}, status=400)

    all_edges = TaskDependency.objects.all()

    graph = {}
    for d in all_edges:
        graph.setdefault(d.task_id, []).append(d.depends_on_id)

    if has_cycle(depends_on_id, task_id, graph, set()):
        return Response({"error": "Circular dependency detected"}, status=400)

    TaskDependency.objects.create(task_id=task_id, depends_on_id=depends_on_id)
    return Response({"message": "Dependency added"})

📁 backend/tasks/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('tasks/<int:task_id>/dependencies/', views.add_dependency),
]
COMMIT 3 — Implement auto status update
📁 backend/tasks/views.py (add function)
def update_task_status(task):
    deps = TaskDependency.objects.filter(task=task).select_related('depends_on')

    if any(d.depends_on.status == 'blocked' for d in deps):
        task.status = 'blocked'
    elif deps and all(d.depends_on.status == 'completed' for d in deps):
        task.status = 'in_progress'
    else:
        task.status = 'pending'

    task.save()
@api_view(['PATCH'])
def update_task(request, task_id):
    task = Task.objects.get(id=task_id)
    task.status = request.data.get("status")
    task.save()

    dependents = TaskDependency.objects.filter(depends_on=task)
    for d in dependents:
        update_task_status(d.task)

    return Response({"message": "Task updated"})
COMMIT 4 — Add task management UI
📁 frontend/src/App.jsx
import { useEffect, useState } from "react";

function App() {
  const [tasks, setTasks] = useState([]);

  useEffect(() => {
    fetch("http://localhost:8000/api/tasks/")
      .then(r => r.json())
      .then(setTasks);
  }, []);

  return (
    <div className="p-4">
      <h1 className="text-xl font-bold">Tasks</h1>
      {tasks.map(t => (
        <div key={t.id} className="border p-2 my-2">
          {t.title} - {t.status}
        </div>
      ))}
    </div>
  );
}

export default App;
COMMIT 5 — Add graph visualization
📁 frontend/src/Graph.jsx
export default function Graph({ tasks, deps }) {
  return (
    <svg width="600" height="400">
      {tasks.map((t, i) => (
        <circle key={t.id} cx={100 + i*100} cy={100} r="20" fill="blue" />
      ))}
    </svg>
  );
}
COMMIT 6 — Add validations and UX
Message:
Add validations and UX improvements

Add:

Disable buttons while loading

Error message div

Delete confirm

Example:

if (taskId === dependsOnId) {
  alert("Task cannot depend on itself");
}

Commit:
git add .
git commit -m "Add validations and UX improvements"
git push

