<script lang="ts">
    import { onMount } from 'svelte';

    interface Task {
        id: number;
        title: string;
        description: string;
        priority: 'low' | 'medium' | 'high';
        dueDate: string;
    }

    interface TaskList {
        [key: string]: Task[];
    }

    interface EditingTask extends Task {
        sourceList?: string;
    }

    interface FormData {
        title: string;
        description: string;
        priority: 'low' | 'medium' | 'high';
        dueDate: string;
    }

    let tasks: TaskList = {};
    let editingTask: EditingTask | null = null;
    let showEditModal = false;
    let showColumnModal = false;
    let newColumnName = '';
    
    // Form data
    let formData: FormData = {
        title: '',
        description: '',
        priority: 'medium',
        dueDate: ''
    };

    // Update form data when editing task changes
    $: if (editingTask) {
        formData = {
            title: editingTask.title,
            description: editingTask.description,
            priority: editingTask.priority,
            dueDate: editingTask.dueDate
        };
    }

    // Load tasks from localStorage on component mount
    onMount(() => {
        const savedTasks = localStorage.getItem('kanbanTasks');
        if (savedTasks) {
            tasks = JSON.parse(savedTasks);
        }
    });

    // Save tasks to localStorage whenever they change
    $: {
        if (typeof window !== 'undefined') {
            localStorage.setItem('kanbanTasks', JSON.stringify(tasks));
        }
    }

    function handleDragStart(event: DragEvent, task: Task, sourceList: string) {
        if (event.dataTransfer) {
            event.dataTransfer.setData('text/plain', JSON.stringify({ task, sourceList }));
        }
    }

    function handleDrop(event: DragEvent, targetList: string) {
        event.preventDefault();
        const data = event.dataTransfer?.getData('text/plain');
        if (!data) return;
        
        const { task, sourceList } = JSON.parse(data) as { task: Task; sourceList: string };
        
        if (sourceList !== targetList) {
            // Moving between columns
            tasks[sourceList] = tasks[sourceList].filter(t => t.id !== task.id);
            tasks[targetList] = [...tasks[targetList], task];
        } else {
            // Reordering within the same column
            const dropIndex = tasks[targetList].findIndex(t => 
                event.target instanceof Element && 
                t.id.toString() === event.target.closest('[data-task-id]')?.getAttribute('data-task-id')
            );
            if (dropIndex !== -1) {
                const sourceIndex = tasks[sourceList].findIndex(t => t.id === task.id);
                const taskList = [...tasks[targetList]];
                taskList.splice(sourceIndex, 1);
                taskList.splice(dropIndex, 0, task);
                tasks[targetList] = taskList;
            }
        }
        tasks = tasks;
    }

    function handleDragOver(event: DragEvent) {
        event.preventDefault();
    }

    function addTask() {
        if (Object.keys(tasks).length === 0) {
            alert('Please create at least one column first');
            return;
        }
        editingTask = {
            id: Math.max(0, ...Object.values(tasks).flat().map(t => t.id)) + 1,
            title: '',
            description: '',
            priority: 'medium',
            dueDate: new Date().toISOString().split('T')[0]
        };
        formData = {
            title: '',
            description: '',
            priority: 'medium',
            dueDate: new Date().toISOString().split('T')[0]
        };
        showEditModal = true;
    }

    function editTask(task: Task, sourceList: string) {
        editingTask = { ...task, sourceList };
        showEditModal = true;
    }

    function deleteTask(taskId: number, sourceList: string) {
        tasks[sourceList] = tasks[sourceList].filter(t => t.id !== taskId);
        tasks = tasks;
    }

    function deleteColumn(columnName: string) {
        const confirmed = confirm(`Are you sure you want to delete the "${columnName}" column and all its tasks?`);
        if (confirmed) {
            const { [columnName]: deletedColumn, ...remainingTasks } = tasks;
            tasks = remainingTasks;
        }
    }

    function addColumn() {
        if (!newColumnName.trim()) {
            alert('Please enter a column name');
            return;
        }
        if (tasks[newColumnName]) {
            alert('A column with this name already exists');
            return;
        }
        tasks = {
            ...tasks,
            [newColumnName]: []
        };
        newColumnName = '';
        showColumnModal = false;
    }

    function saveTask() {
        if (!editingTask) return;

        const selectedColumn = editingTask.sourceList || Object.keys(tasks)[0];
        const updatedTask = {
            id: editingTask.id,
            title: formData.title,
            description: formData.description,
            priority: formData.priority,
            dueDate: formData.dueDate
        };

        if (editingTask.sourceList) {
            // Editing existing task
            tasks[selectedColumn] = tasks[selectedColumn].map(t => 
                t.id === updatedTask.id ? updatedTask : t
            );
        } else {
            // Adding new task
            tasks[selectedColumn] = [...tasks[selectedColumn], updatedTask];
        }
        tasks = tasks;
        closeModal();
    }

    function closeModal() {
        showEditModal = false;
        showColumnModal = false;
        editingTask = null;
    }

    function getPriorityColor(priority: Task['priority']): string {
        switch (priority) {
            case 'high': return 'bg-red-100 text-red-800';
            case 'medium': return 'bg-yellow-100 text-yellow-800';
            case 'low': return 'bg-green-100 text-green-800';
            default: return 'bg-gray-100 text-gray-800';
        }
    }

    function isOverdue(dueDate: string): boolean {
        return new Date(dueDate).getTime() < new Date().setHours(0, 0, 0, 0);
    }
</script>

<div class="min-h-screen bg-blue-900 p-4 md:p-8 kanban-scroll">
    <div class="max-w-full mx-auto">
        <div class="flex justify-between items-center mb-8 px-4">
            <h1 class="text-2xl md:text-3xl font-bold text-white">Kanban Board</h1>
            <div class="flex gap-3">
                <button
                    on:click={() => showColumnModal = true}
                    class="bg-green-500 hover:bg-green-400 text-white px-4 py-2 rounded-lg shadow-sm transition-colors"
                >
                    Add Column
                </button>
                <button
                    on:click={addTask}
                    class="bg-blue-500 hover:bg-blue-400 text-white px-4 py-2 rounded-lg shadow-sm transition-colors"
                >
                    Add Task
                </button>
            </div>
        </div>

        <div class="overflow-x-auto kanban-scroll">
            <div class="inline-flex gap-4 p-4 min-w-full">
                {#each Object.entries(tasks) as [columnName, taskList] (columnName)}
                    <div
                        role="region"
                        aria-label={columnName}
                        class="bg-blue-800 rounded-lg shadow-lg p-4 border border-blue-700 w-80 flex-shrink-0"
                        on:drop={(e) => handleDrop(e, columnName)}
                        on:dragover={handleDragOver}
                    >
                        <div class="flex justify-between items-center mb-4">
                            <h2 class="text-lg font-semibold text-white">
                                {columnName}
                            </h2>
                            <div class="flex items-center gap-2">
                                <span class="bg-blue-700 px-2.5 py-0.5 rounded-full text-sm text-blue-100 font-medium">
                                    {taskList.length}
                                </span>
                                <button
                                    on:click={() => deleteColumn(columnName)}
                                    class="text-red-300 hover:text-red-100 p-1"
                                    title="Delete column"
                                >
                                    🗑️
                                </button>
                            </div>
                        </div>
                        <div class="space-y-3 max-h-[calc(100vh-12rem)] overflow-y-auto kanban-scroll">
                            {#each taskList as task (task.id)}
                                <div
                                    role="article"
                                    data-task-id={task.id}
                                    aria-label={`Task: ${task.title}`}
                                    class="bg-blue-700 hover:bg-blue-600 p-4 rounded-lg shadow-md cursor-move group hover:shadow-lg transition-all duration-200 border border-blue-600"
                                    draggable={true}
                                    on:dragstart={(e) => handleDragStart(e, task, columnName)}
                                >
                                    <div class="flex justify-between items-start mb-2">
                                        <h3 class="font-medium text-white">{task.title}</h3>
                                        <div class="flex space-x-2 opacity-0 group-hover:opacity-100 transition-opacity">
                                            <button
                                                aria-label={`Edit task: ${task.title}`}
                                                on:click={() => editTask(task, columnName)}
                                                class="text-blue-200 hover:text-white"
                                            >
                                                ✏️
                                            </button>
                                            <button
                                                aria-label={`Delete task: ${task.title}`}
                                                on:click={() => deleteTask(task.id, columnName)}
                                                class="text-red-300 hover:text-red-100"
                                            >
                                                🗑️
                                            </button>
                                        </div>
                                    </div>
                                    <p class="text-sm text-blue-100 mb-3">{task.description}</p>
                                    <div class="flex flex-wrap gap-2">
                                        <span class={`px-2.5 py-1 rounded-full text-xs font-medium ${getPriorityColor(task.priority)}`}>
                                            {task.priority}
                                        </span>
                                        <span class={`px-2.5 py-1 rounded-full text-xs font-medium ${isOverdue(task.dueDate) ? 'bg-red-100 text-red-800' : 'bg-blue-100 text-blue-800'}`}>
                                            {task.dueDate}
                                        </span>
                                    </div>
                                </div>
                            {/each}
                        </div>
                    </div>
                {/each}
            </div>
        </div>
    </div>
</div>

{#if showEditModal}
    <div
        role="dialog"
        aria-label={editingTask?.sourceList ? 'Edit Task' : 'New Task'}
        class="fixed inset-0 bg-black bg-opacity-70 flex items-center justify-center"
    >
        <div class="bg-blue-800 p-6 rounded-lg w-full max-w-md">
            <h2 class="text-xl font-bold mb-4 text-white">
                {editingTask?.sourceList ? 'Edit Task' : 'New Task'}
            </h2>
            <form on:submit|preventDefault={saveTask} class="space-y-4">
                <div>
                    <label for="task-title" class="block text-sm font-medium text-blue-100">Title</label>
                    <input
                        id="task-title"
                        type="text"
                        bind:value={formData.title}
                        class="mt-1 block w-full rounded-md bg-blue-700 border-blue-600 text-white placeholder-blue-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
                        required
                    />
                </div>
                <div>
                    <label for="task-description" class="block text-sm font-medium text-blue-100">Description</label>
                    <textarea
                        id="task-description"
                        bind:value={formData.description}
                        class="mt-1 block w-full rounded-md bg-blue-700 border-blue-600 text-white placeholder-blue-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
                        rows="3"
                        required
                    ></textarea>
                </div>
                <div>
                    <label for="task-priority" class="block text-sm font-medium text-blue-100">Priority</label>
                    <select
                        id="task-priority"
                        bind:value={formData.priority}
                        class="mt-1 block w-full rounded-md bg-blue-700 border-blue-600 text-white shadow-sm focus:border-blue-500 focus:ring-blue-500"
                        required
                    >
                        <option value="low">Low</option>
                        <option value="medium">Medium</option>
                        <option value="high">High</option>
                    </select>
                </div>
                <div>
                    <label for="task-due-date" class="block text-sm font-medium text-blue-100">Due Date</label>
                    <input
                        id="task-due-date"
                        type="date"
                        bind:value={formData.dueDate}
                        class="mt-1 block w-full rounded-md bg-blue-700 border-blue-600 text-white shadow-sm focus:border-blue-500 focus:ring-blue-500"
                        required
                    />
                </div>
                <div class="mt-6 flex justify-end space-x-3">
                    <button
                        type="button"
                        on:click={closeModal}
                        class="px-4 py-2 border border-blue-500 rounded-md text-blue-100 hover:bg-blue-700"
                    >
                        Cancel
                    </button>
                    <button
                        type="submit"
                        class="px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-400"
                    >
                        Save
                    </button>
                </div>
            </form>
        </div>
    </div>
{/if}

{#if showColumnModal}
    <div
        role="dialog"
        aria-label="Add New Column"
        class="fixed inset-0 bg-black bg-opacity-70 flex items-center justify-center"
    >
        <div class="bg-blue-800 p-6 rounded-lg w-full max-w-md">
            <h2 class="text-xl font-bold mb-4 text-white">Add New Column</h2>
            <form on:submit|preventDefault={addColumn} class="space-y-4">
                <div>
                    <label for="column-name" class="block text-sm font-medium text-blue-100">Column Name</label>
                    <input
                        id="column-name"
                        type="text"
                        bind:value={newColumnName}
                        class="mt-1 block w-full rounded-md bg-blue-700 border-blue-600 text-white placeholder-blue-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
                        placeholder="Enter column name"
                        required
                    />
                </div>
                <div class="mt-6 flex justify-end space-x-3">
                    <button
                        type="button"
                        on:click={closeModal}
                        class="px-4 py-2 border border-blue-500 rounded-md text-blue-100 hover:bg-blue-700"
                    >
                        Cancel
                    </button>
                    <button
                        type="submit"
                        class="px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-400"
                    >
                        Add Column
                    </button>
                </div>
            </form>
        </div>
    </div>
{/if}
