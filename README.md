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

---

## Performance

System handles graphs of 20–30 tasks without noticeable UI lag.
DFS is efficient for the small graph sizes expected in this use case.


