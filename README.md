import React, { useState, useMemo, useCallback, useEffect } from 'react';
import { PlusCircle, Edit, Trash2, ChevronUp, ChevronDown, CheckCircle, XCircle, ArrowRightCircle, Users, XOctagon } from 'lucide-react';

// IMPORTANT: For custom Tailwind colors to work in a simple HTML setup
// (like in the Canvas preview without a full build process),
// you need to configure them directly in your HTML file where Tailwind is loaded.
// Add this script BEFORE your main app script:
/*
<script>
  tailwind.config = {
    theme: {
      extend: {
        colors: {
          clickBlue: '#3f51b5', // Navy-like blue from your logo
          'clickBlue-dark': '#303f9f', // A darker shade for hover/focus
          'clickBlue-light': '#6679cc', // A lighter shade for subtle effects
          clickPink: '#e91e63', // Pink from your logo
          'clickPink-dark': '#c2185b', // A darker shade for hover/focus
          'clickPink-light': '#f06292', // A lighter shade for subtle effects
        }
      }
    }
  }
</script>
<script src="https://cdn.tailwindcss.com"></script>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800;900&display=swap" rel="stylesheet">
*/

// Helper function to normalize hours to a 1-5 scale for priority calculation
// You can adjust these ranges based on your preference for "long" vs. "short" tasks.
const normalizeHoursToScale = (hours) => {
  if (hours <= 2) return 1; // Very short
  if (hours <= 5) return 2; // Short
  if (hours <= 10) return 3; // Medium
  if (hours <= 20) return 4; // Long
  return 5; // Very long (20+ hours)
};

// Custom Modal Component for alerts and confirmations
function Modal({ isOpen, title, message, onConfirm, onCancel, type = 'alert' }) {
  if (!isOpen) return null;

  const buttonClasses = "px-4 py-2 rounded-md font-medium transition duration-150 ease-in-out shadow-sm";
  const primaryButtonClasses = "text-white bg-clickBlue hover:bg-clickBlue-dark focus:ring-2 focus:ring-offset-2 focus:ring-clickBlue";
  const secondaryButtonClasses = "text-gray-700 bg-gray-200 hover:bg-gray-300 focus:ring-2 focus:ring-offset-2 focus:ring-gray-300";

  return (
    <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center p-4 z-50">
      <div className="bg-white rounded-lg shadow-xl p-6 w-full max-w-sm">
        <h3 className="text-xl font-semibold text-gray-900 mb-4">{title}</h3>
        <p className="text-base text-gray-700 mb-6">{message}</p>
        <div className="flex justify-end space-x-3">
          {type === 'confirmation' && (
            <button
              onClick={onCancel}
              className={`${buttonClasses} ${secondaryButtonClasses}`}
            >
              Cancel
            </button>
          )}
          <button
            onClick={onConfirm || onCancel}
            className={`${buttonClasses} ${primaryButtonClasses}`}
          >
            {type === 'confirmation' ? 'Confirm' : 'OK'}
          </button>
        </div>
      </div>
    </div>
  );
}

// Reusable Rating Toggle Component
function RatingToggle({ label, value, onChange }) {
  const options = [1, 2, 3, 4, 5];

  return (
    <div>
      <label className="block text-sm font-medium text-clickBlue mb-1">{label}</label> {/* Label text is navy */}
      <div className="flex space-x-1 p-0.5 bg-gray-100 rounded-md">
        {options.map((option) => (
          <button
            key={option}
            type="button"
            onClick={() => onChange(option)}
            className={`flex-1 px-3 py-1.5 text-sm font-medium rounded-md transition-all duration-200 ease-in-out
              ${value === option
                ? 'bg-clickPink text-white shadow-md' // Using clickPink for selected state
                : 'text-gray-700 hover:bg-gray-200'
              }`}
          >
            {option}
          </button>
        ))}
      </div>
    </div>
  );
}


// Main App Component
function App() {
  const [tasks, setTasks] = useState([]);
  const [nextId, setNextId] = useState(1);
  const [sortConfig, setSortConfig] = useState({ key: 'priority', direction: 'descending' });
  const [editingTask, setEditingTask] = useState(null);
  const [modalState, setModalState] = useState({ isOpen: false, title: '', message: '', onConfirm: null, onCancel: null, type: 'alert' });

  // Function to add a new task
  const addTask = useCallback((taskName, impact, effort, hoursRequired) => {
    if (!taskName.trim()) {
      setModalState({ isOpen: true, title: 'Input Error', message: 'Task name cannot be empty.', onConfirm: () => setModalState({ ...modalState, isOpen: false }) });
      return;
    }
    if (hoursRequired < 0) {
      setModalState({ isOpen: true, title: 'Input Error', message: 'Hours required cannot be negative.', onConfirm: () => setModalState({ ...modalState, isOpen: false }) });
      return;
    }

    const timeRequiredScaled = normalizeHoursToScale(hoursRequired);

    // New priority calculation with weights: Impact (50%), Effort (25%), Time (25%)
    // Multipliers (5, 2.5, 2.5) are used to scale the 1-5 values for a wider priority range.
    const priority = (parseInt(impact, 10) * 5) - (parseInt(effort, 10) * 2.5) - (timeRequiredScaled * 2.5);

    const newTask = {
      id: nextId,
      name: taskName,
      impact: parseInt(impact, 10),
      effort: parseInt(effort, 10),
      hoursRequired: parseInt(hoursRequired, 10),
      timeRequiredScaled: timeRequiredScaled,
      priority: priority,
      lane: 'focus', // Default lane for new tasks
    };
    setTasks((prevTasks) => [...prevTasks, newTask]);
    setNextId((prevId) => prevId + 1);
  }, [nextId, modalState]);

  // Function to update an existing task
  const updateTask = useCallback((id, updatedFields) => {
    setTasks((prevTasks) =>
      prevTasks.map((task) => {
        if (task.id === id) {
          const updatedTask = { ...task, ...updatedFields };
          // Recalculate priority if relevant fields are updated
          if (updatedFields.impact !== undefined || updatedFields.effort !== undefined || updatedFields.hoursRequired !== undefined) {
            updatedTask.timeRequiredScaled = normalizeHoursToScale(updatedTask.hoursRequired);
            updatedTask.priority = (updatedTask.impact * 5) - (updatedTask.effort * 2.5) - (updatedTask.timeRequiredScaled * 2.5);
          }
          return updatedTask;
        }
        return task;
      })
    );
    setEditingTask(null);
  }, []);

  // Function to delete a task with custom confirmation modal
  const deleteTask = useCallback((id) => {
    setModalState({
      isOpen: true,
      title: 'Confirm Deletion',
      message: 'Are you sure you want to delete this task?',
      type: 'confirmation',
      onConfirm: () => {
        setTasks((prevTasks) => prevTasks.filter((task) => task.id !== id));
        setModalState({ ...modalState, isOpen: false });
      },
      onCancel: () => setModalState({ ...modalState, isOpen: false }),
    });
  }, [modalState]);

  // Handle drag and drop to change task lanes
  const moveTaskToLane = useCallback((taskId, newLane) => {
    setTasks((prevTasks) =>
      prevTasks.map((task) =>
        task.id === taskId ? { ...task, lane: newLane } : task
      )
    );
  }, []);

  // Memoized sorted tasks based on sortConfig
  const sortedTasks = useMemo(() => {
    let sortableItems = [...tasks];
    if (sortConfig.key !== null) {
      sortableItems.sort((a, b) => {
        const aValue = a[sortConfig.key];
        const bValue = b[sortConfig.key];

        if (aValue < bValue) {
          return sortConfig.direction === 'ascending' ? -1 : 1;
        }
        if (aValue > bValue) {
          return sortConfig.direction === 'ascending' ? 1 : -1;
        }
        return 0;
      });
    }
    return sortableItems;
  }, [tasks, sortConfig]);

  // Request sort
  const requestSort = useCallback((key) => {
    let direction = 'ascending';
    if (sortConfig.key === key && sortConfig.direction === 'ascending') {
      direction = 'descending';
    } else if (sortConfig.key === key && sortConfig.direction === 'descending') {
      direction = 'ascending';
    }
    setSortConfig({ key, direction });
  }, [sortConfig]);

  return (
    // The background gradient now uses clickBlue and clickPink for branding
    <div className="min-h-screen bg-gradient-to-br from-clickBlue-light to-clickPink-light p-4 font-inter text-clickBlue antialiased">
      <div className="container mx-auto max-w-6xl bg-white shadow-xl rounded-xl p-6 md:p-8">
        <div className="flex items-center justify-center mb-8">
          <img src="http://googleusercontent.com/file_content/1" alt="Click Learning Logo" className="h-10 md:h-12 mr-3" />
          <h1 className="text-4xl md:text-5xl font-extrabold text-clickBlue drop-shadow-md">
            Task Prioritizer
          </h1>
        </div>

        {/* Task Form */}
        <TaskForm addTask={addTask} setModalState={setModalState} />

        {/* Dashboard Lanes */}
        <Dashboard
          tasks={tasks} // Pass unsorted tasks to Dashboard for lane-specific sorting
          updateTask={updateTask}
          deleteTask={deleteTask}
          editingTask={editingTask}
          setEditingTask={setEditingTask}
          setModalState={setModalState}
          moveTaskToLane={moveTaskToLane}
        />
      </div>
      <Modal {...modalState} />
    </div>
  );
}

// TaskForm Component
function TaskForm({ addTask, setModalState }) {
  const [taskName, setTaskName] = useState('');
  const [impact, setImpact] = useState(3);
  const [effort, setEffort] = useState(3);
  const [hoursRequired, setHoursRequired] = useState(5);

  const handleSubmit = (e) => {
    e.preventDefault();
    if (hoursRequired < 0) {
      setModalState({ isOpen: true, title: 'Input Error', message: 'Hours required cannot be negative.', onConfirm: () => setModalState(prev => ({ ...prev, isOpen: false })) });
      return;
    }
    addTask(taskName, impact, effort, hoursRequired);
    setTaskName('');
    setImpact(3);
    setEffort(3);
    setHoursRequired(5);
  };

  return (
    <form onSubmit={handleSubmit} className="bg-blue-50 p-6 rounded-lg shadow-inner mb-8 space-y-4">
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
        {/* Task Name */}
        <div>
          <label htmlFor="taskName" className="block text-sm font-medium text-clickBlue mb-1">
            Task Name
          </label>
          <input
            type="text"
            id="taskName"
            className="w-full px-4 py-2 border border-gray-300 rounded-md focus:ring-clickBlue focus:border-clickBlue transition duration-150 ease-in-out text-gray-800"
            value={taskName}
            onChange={(e) => setTaskName(e.target.value)}
            placeholder="e.g., Plan team meeting"
            required
          />
        </div>

        {/* Impact Toggle */}
        <RatingToggle
          label="Impact (1-5, 5=High)"
          value={impact}
          onChange={setImpact}
        />

        {/* Effort Toggle */}
        <RatingToggle
          label="Effort (1-5, 5=High)"
          value={effort}
          onChange={setEffort}
        />

        {/* Hours Required Input */}
        <div>
          <label htmlFor="hoursRequired" className="block text-sm font-medium text-clickBlue mb-1">
            Hours Required
          </label>
          <input
            type="number"
            id="hoursRequired"
            className="w-full px-4 py-2 border border-gray-300 rounded-md focus:ring-clickBlue focus:border-clickBlue transition duration-150 ease-in-out text-gray-800"
            value={hoursRequired}
            onChange={(e) => setHoursRequired(e.target.value)}
            min="0"
            required
          />
        </div>
      </div>

      <button
        type="submit"
        className="w-full flex items-center justify-center px-6 py-3 border border-transparent text-base font-medium rounded-md text-white bg-clickPink hover:bg-clickPink-dark focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-clickPink shadow-md transition duration-150 ease-in-out transform hover:-translate-y-0.5"
      >
        <PlusCircle className="mr-2 h-5 w-5" /> Add Task
      </button>
    </form>
  );
}

// Dashboard Component to display tasks in lanes
function Dashboard({ tasks, updateTask, deleteTask, editingTask, setEditingTask, setModalState, moveTaskToLane }) {
  const lanes = [
    { id: 'focus', title: 'Focus (High Priority)', icon: <ArrowRightCircle className="h-5 w-5 text-clickBlue" /> },
    { id: 'delegate', title: 'Delegate', icon: <Users className="h-5 w-5 text-green-600" /> }, // Green for delegate for clear distinction
    { id: 'kill', title: 'Kill (Don\'t Do)', icon: <XOctagon className="h-5 w-5 text-clickPink" /> }, // Pink for kill
  ];

  // Group tasks by lane and sort them by priority within each lane
  const tasksByLane = useMemo(() => {
    const grouped = {};
    lanes.forEach(lane => {
      grouped[lane.id] = tasks
        .filter(task => task.lane === lane.id)
        .sort((a, b) => b.priority - a.priority); // Sort by priority descending within each lane
    });
    return grouped;
  }, [tasks, lanes]);

  // State to hold the ID of the task being dragged
  const [draggedTaskId, setDraggedTaskId] = useState(null);

  // Drag start handler
  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('text/plain', taskId); // Store the task ID
    setDraggedTaskId(taskId); // Keep track of the dragged task for styling
  };

  // Drag over handler for lanes
  const handleDragOver = (e) => {
    e.preventDefault(); // Essential to allow dropping
  };

  // Drop handler for lanes
  const handleDrop = (e, targetLaneId) => {
    e.preventDefault();
    const taskId = parseInt(e.dataTransfer.getData('text/plain'), 10); // Retrieve task ID
    moveTaskToLane(taskId, targetLaneId);
    setDraggedTaskId(null); // Reset dragged task ID
  };

  const getLaneBgColor = (laneId) => {
    switch (laneId) {
      case 'focus': return 'bg-blue-50'; // Keeping a light blue background for the focus lane
      case 'delegate': return 'bg-green-50';
      case 'kill': return 'bg-pink-50'; // Using a light pink background for the kill lane
      default: return 'bg-gray-50';
    }
  };

  return (
    <div className="grid grid-cols-1 md:grid-cols-3 gap-6 mt-8">
      {lanes.map((lane) => (
        <div
          key={lane.id}
          className={`flex flex-col p-4 rounded-lg shadow-md min-h-[300px] ${getLaneBgColor(lane.id)} transition-all duration-200 ease-in-out`}
          onDragOver={handleDragOver}
          onDrop={(e) => handleDrop(e, lane.id)}
        >
          <h3 className={`text-xl font-semibold mb-4 flex items-center gap-2
            ${lane.id === 'focus' ? 'text-clickBlue' : ''}
            ${lane.id === 'delegate' ? 'text-green-700' : ''}
            ${lane.id === 'kill' ? 'text-clickPink' : ''}
          `}>
            {lane.icon} {lane.title}
          </h3>
          <div className="space-y-3 flex-grow">
            {tasksByLane[lane.id].length === 0 ? (
              <p className="text-gray-500 text-center py-4">No tasks in this lane.</p>
            ) : (
              tasksByLane[lane.id].map((task) => (
                <TaskCard
                  key={task.id}
                  task={task}
                  updateTask={updateTask}
                  deleteTask={deleteTask}
                  isEditing={editingTask && editingTask.id === task.id}
                  setEditingTask={setEditingTask}
                  setModalState={setModalState}
                  onDragStart={handleDragStart}
                  isDragged={draggedTaskId === task.id}
                />
              ))
            )}
          </div>
        </div>
      ))}
    </div>
  );
}

// TaskCard Component (replaces TaskItem for dashboard view)
function TaskCard({ task, updateTask, deleteTask, isEditing, setEditingTask, setModalState, onDragStart, isDragged }) {
  const [editedName, setEditedName] = useState(task.name);
  const [editedImpact, setEditedImpact] = useState(task.impact);
  const [editedEffort, setEditedEffort] = useState(task.effort);
  const [editedHoursRequired, setEditedHoursRequired] = useState(task.hoursRequired);

  useEffect(() => {
    if (isEditing) {
      setEditedName(task.name);
      setEditedImpact(task.impact);
      setEditedEffort(task.effort);
      setEditedHoursRequired(task.hoursRequired);
    }
  }, [isEditing, task]);

  const handleSave = () => {
    if (!editedName.trim()) {
      setModalState({ isOpen: true, title: 'Input Error', message: 'Task name cannot be empty.', onConfirm: () => setModalState(prev => ({ ...prev, isOpen: false })) });
      return;
    }
    if (editedHoursRequired < 0) {
      setModalState({ isOpen: true, title: 'Input Error', message: 'Hours required cannot be negative.', onConfirm: () => setModalState(prev => ({ ...prev, isOpen: false })) });
      return;
    }

    updateTask(task.id, {
      name: editedName,
      impact: parseInt(editedImpact, 10),
      effort: parseInt(editedEffort, 10),
      hoursRequired: parseInt(editedHoursRequired, 10),
    });
    setEditingTask(null);
  };

  const handleCancel = () => {
    setEditingTask(null);
  };

  return (
    <div
      className={`bg-white rounded-lg shadow-sm p-4 border border-gray-200 cursor-grab transition-all duration-150 ease-in-out
        ${isDragged ? 'border-clickBlue ring-2 ring-clickBlue-light opacity-70' : 'hover:shadow-md'}
      `}
      draggable="true"
      onDragStart={(e) => onDragStart(e, task.id)}
    >
      {isEditing ? (
        <div className="space-y-2">
          <input
            type="text"
            className="w-full border rounded-md px-2 py-1 text-sm focus:ring-clickBlue focus:border-clickBlue text-gray-800"
            value={editedName}
            onChange={(e) => setEditedName(e.target.value)}
          />
          <RatingToggle
            label="Impact"
            value={editedImpact}
            onChange={setEditedImpact}
          />
          <RatingToggle
            label="Effort"
            value={editedEffort}
            onChange={setEditedEffort}
          />
          <div>
            <label className="block text-xs font-medium text-clickBlue mb-0.5">Hours</label> {/* Label text is navy */}
            <input
              type="number"
              className="w-full border rounded-md px-2 py-1 text-sm focus:ring-clickBlue focus:border-clickBlue text-gray-800"
              value={editedHoursRequired}
              onChange={(e) => setEditedHoursRequired(e.target.value)}
              min="0"
            />
          </div>
          <div className="flex justify-end space-x-2 mt-3">
            <button
              onClick={handleSave}
              className="inline-flex items-center px-2.5 py-1.5 border border-transparent text-xs font-medium rounded-md shadow-sm text-white bg-clickPink hover:bg-clickPink-dark focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-clickPink transition-colors"
            >
              <CheckCircle className="h-3.5 w-3.5 mr-1" /> Save
            </button>
            <button
              onClick={handleCancel}
              className="inline-flex items-center px-2.5 py-1.5 border border-gray-300 text-xs font-medium rounded-md shadow-sm text-gray-700 bg-white hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-300 transition-colors"
            >
              <XCircle className="h-3.5 w-3.5 mr-1" /> Cancel
            </button>
          </div>
        </div>
      ) : (
        <div>
          <h4 className="font-semibold text-clickBlue mb-1">{task.name}</h4> {/* Task name text is navy */}
          <p className="text-xs text-gray-600">Impact: {task.impact} | Effort: {task.effort} | Hours: {task.hoursRequired}</p>
          <p className="text-sm font-bold text-clickBlue mt-2">Priority: {task.priority.toFixed(1)}</p>
          <div className="flex justify-end space-x-2 mt-3">
            <button
              onClick={() => setEditingTask(task)}
              className="inline-flex items-center px-2.5 py-1.5 border border-transparent text-xs font-medium rounded-md shadow-sm text-white bg-clickBlue hover:bg-clickBlue-dark focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-clickBlue transition-colors"
            >
              <Edit className="h-3.5 w-3.5 mr-1" /> Edit
            </button>
            <button
              onClick={() => deleteTask(task.id)}
              className="inline-flex items-center px-2.5 py-1.5 border border-transparent text-xs font-medium rounded-md shadow-sm text-white bg-clickPink hover:bg-clickPink-dark focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-clickPink transition-colors"
            >
              <Trash2 className="h-3.5 w-3.5 mr-1" /> Delete
            </button>
          </div>
        </div>
      )}
    </div>
  );
}

// This needs to be exported as default for the Canvas environment
export default App;
