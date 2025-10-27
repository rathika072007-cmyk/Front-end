#node-js-projectv
{  
  "name": "todo-backend",  
  "version": "1.0.0",  
  "main": "index.js",  
  "scripts": {  
    "start": "node index.js",  
    "dev": "nodemon index.js"  
  },  
  "dependencies": {  
    "cors": "^2.8.5",  
    "express": "^4.18.2",  
    "better-sqlite3": "^8.4.0",  
    "body-parser": "^1.20.2"  
  },  
  "devDependencies": {  
    "nodemon": "^2.0.22"  
  }  
}

Backend: backend/db.js

const Database = require('better-sqlite3');  
const db = new Database('./database.sqlite');  
  
// Initialize table  
const init = () => {  
  db.prepare(  
    `CREATE TABLE IF NOT EXISTS todos (  
      id INTEGER PRIMARY KEY AUTOINCREMENT,  
      title TEXT NOT NULL,  
      completed INTEGER DEFAULT 0,  
      created_at TEXT DEFAULT (datetime('now'))  
    )`  
  ).run();  
};  
  
init();  
  
module.exports = db;

Backend: backend/index.js

const express = require('express');  
const bodyParser = require('body-parser');  
const cors = require('cors');  
const db = require('./db');  
  
const app = express();  
app.use(cors());  
app.use(bodyParser.json());  
  
// Get all todos  
app.get('/api/todos', (req, res) => {  
  const rows = db.prepare('SELECT id, title, completed, created_at FROM todos ORDER BY id DESC').all();  
  const todos = rows.map(r => ({ ...r, completed: !!r.completed }));  
  res.json(todos);  
});  
  
// Create todo  
app.post('/api/todos', (req, res) => {  
  const { title } = req.body;  
  if (!title || !title.trim()) return res.status(400).json({ error: 'Title required' });  
  const info = db.prepare('INSERT INTO todos (title, completed) VALUES (?, 0)').run(title.trim());  
  const todo = db.prepare('SELECT id, title, completed, created_at FROM todos WHERE id = ?').get(info.lastInsertRowid);  
  todo.completed = !!todo.completed;  
  res.status(201).json(todo);  
});  
  
// Update todo  
app.put('/api/todos/:id', (req, res) => {  
  const { id } = req.params;  
  const { title, completed } = req.body;  
  const existing = db.prepare('SELECT * FROM todos WHERE id = ?').get(id);  
  if (!existing) return res.status(404).json({ error: 'Not found' });  
  const newTitle = typeof title === 'string' ? title.trim() : existing.title;  
  const newCompleted = typeof completed === 'boolean' ? (completed ? 1 : 0) : existing.completed;  
  db.prepare('UPDATE todos SET title = ?, completed = ? WHERE id = ?').run(newTitle, newCompleted, id);  
  const updated = db.prepare('SELECT id, title, completed, created_at FROM todos WHERE id = ?').get(id);  
  updated.completed = !!updated.completed;  
  res.json(updated);  
});  
  
// Delete  
app.delete('/api/todos/:id', (req, res) => {  
  const { id } = req.params;  
  db.prepare('DELETE FROM todos WHERE id = ?').run(id);  
  res.json({ success: true });  
});  
  
const PORT = process.env.PORT || 4000;  
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));


---

Frontend (Vite + React): frontend/package.json

{  
  "name": "todo-frontend",  
  "version": "1.0.0",  
  "private": true,  
  "scripts": {  
    "dev": "vite",  
    "build": "vite build",  
    "preview": "vite preview"  
  },  
  "dependencies": {  
    "axios": "^1.4.0",  
    "react": "^18.2.0",  
    "react-dom": "^18.2.0"  
  },  
  "devDependencies": {  
    "vite": "^5.0.0"  
  }  
}

Frontend: frontend/src/main.jsx

import React from 'react'  
import { createRoot } from 'react-dom/client'  
import App from './App'  
import './index.css'  
  
createRoot(document.getElementById('root')).render(<App />)

Frontend: frontend/src/App.jsx

import React, { useEffect, useState } from 'react'  
import axios from 'axios'  
import TodoForm from './components/TodoForm'  
import TodoList from './components/TodoList'  
  
const API = import.meta.env.VITE_API_URL || 'http://localhost:4000/api'  
  
export default function App() {  
  const [todos, setTodos] = useState([])  
  const [filter, setFilter] = useState('all')  
  
  useEffect(() => { fetchTodos() }, [])  
  
  async function fetchTodos() {  
    const res = await axios.get(`${API}/todos`)  
    setTodos(res.data)  
  }  
  
  async function addTodo(title) {  
    const res = await axios.post(`${API}/todos`, { title })  
    setTodos(prev => [res.data, ...prev])  
  }  
  
  async function updateTodo(id, changes) {  
    const res = await axios.put(`${API}/todos/${id}`, changes)  
    setTodos(prev => prev.map(t => (t.id === id ? res.data : t)))  
  }  
  
  async function deleteTodo(id) {  
    await axios.delete(`${API}/todos/${id}`)  
    setTodos(prev => prev.filter(t => t.id !== id))  
  }  
  
  const filtered = todos.filter(t => {  
    if (filter === 'active') return !t.completed  
    if (filter === 'completed') return t.completed  
    return true  
  })  
  
  return (  
    <div className="min-h-screen flex items-start justify-center p-6 bg-slate-50">  
      <div className="w-full max-w-md bg-white rounded-2xl shadow p-6">  
        <h1 className="text-2xl font-bold mb-4">To-Do List</h1>  
        <TodoForm onAdd={addTodo} />  
        <div className="mt-4">  
          <TodoList todos={filtered} onToggle={t => updateTodo(t.id, { completed: !t.completed })} onEdit={updateTodo} onDelete={deleteTodo} />  
        </div>  
        <div className="mt-4 flex items-center justify-between text-sm">  
          <div>{todos.filter(t => !t.completed).length} items left</div>  
          <div className="space-x-2">  
            <button onClick={() => setFilter('all')}>All</button>  
            <button onClick={() => setFilter('active')}>Active</button>  
            <button onClick={() => setFilter('completed')}>Completed</button>  
          </div>  
        </div>  
      </div>  
    </div>  
  )  
}

Frontend: frontend/src/components/TodoForm.jsx

import React, { useState } from 'react'  
  
export default function TodoForm({ onAdd }) {  
  const [title, setTitle] = useState('')  
  const submit = e => {  
    e.preventDefault()  
    if (!title.trim()) return  
    onAdd(title.trim())  
    setTitle('')  
  }  
  return (  
    <form onSubmit={submit} className="flex gap-2">  
      <input value={title} onChange={e => setTitle(e.target.value)} placeholder="Add a new task" className="flex-1 p-2 border rounded" />  
      <button className="px-3 py-2 border rounded">Add</button>  
    </form>  
  )  
}

Frontend: frontend/src/components/TodoList.jsx

import React, { useState } from 'react'  
  
function TodoItem({ todo, onToggle, onEdit, onDelete }) {  
  const [editing, setEditing] = useState(false)  
  const [value, setValue] = useState(todo.title)  
  
  function save() {  
    if (!value.trim()) return  
    onEdit(todo.id, { title: value.trim() })  
    setEditing(false)  
  }  
  
  return (  
    <div className="flex items-center gap-2 py-2 border-b">  
      <input type="checkbox" checked={todo.completed} onChange={() => onToggle(todo)} />  
      {editing ? (  
        <input className="flex-1 p-1" value={value} onChange={e => setValue(e.target.value)} onBlur={save} onKeyDown={e => e.key === 'Enter' && save()} />  
      ) : (  
        <div className={`flex-1 ${todo.completed ? 'line-through text-slate-400' : ''}`}>{todo.title}</div>  
      )}  
      {editing ? (  
        <button onClick={save}>Save</button>  
      ) : (  
        <>  
          <button onClick={() => { setEditing(true); setValue(todo.title) }}>Edit</button>  
          <button onClick={() => onDelete(todo.id)}>Delete</button>  
        </>  
      )}  
    </div>  
  )  
}  
  
export default function TodoList({ todos, onToggle, onEdit, onDelete }) {  
  if (todos.length === 0) return <div className="p-4 text-center text-sm text-slate-500">No tasks yet</div>  
  return (  
    <div>  
      {todos.map(t => (  
        <TodoItem key={t.id} todo={t} onToggle={onToggle} onEdit={onEdit} onDelete={onDelete} />  
      ))}  
    </div>  
  )  
}

Frontend: frontend/index.html (root)

<!doctype html>  
<html>  
  <head>  
    <meta charset="utf-8" />  
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />  
    <title>To-Do App</title>  
  </head>  
  <body>  
    <div id="root"></div>  
    <script type="module" src="/src/main.jsx"></script>  
  </body>  
</html>
