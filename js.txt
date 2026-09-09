let tasks = [];
let activeTaskId = null;
let timerId = null;
let continuousSecondsWorking = 0;
let breaksCount = 0;
let hasAlertedHighRisk = false;
let history = {}; // { "YYYY-MM-DD": { timeSeconds, tasksCompleted, breaks } }

const STORAGE_KEYS = {
    tasks: 'zenwork_tasks',
    breaks: 'zenwork_breaks',
    history: 'zenwork_history'
};

// ---------- Persistence ----------

function saveState() {
    localStorage.setItem(STORAGE_KEYS.tasks, JSON.stringify(tasks));
    localStorage.setItem(STORAGE_KEYS.breaks, String(breaksCount));
    localStorage.setItem(STORAGE_KEYS.history, JSON.stringify(history));
}

function loadState() {
    const savedTasks = localStorage.getItem(STORAGE_KEYS.tasks);
    const savedBreaks = localStorage.getItem(STORAGE_KEYS.breaks);
    const savedHistory = localStorage.getItem(STORAGE_KEYS.history);

    if (savedTasks) tasks = JSON.parse(savedTasks);
    if (savedBreaks) breaksCount = parseInt(savedBreaks, 10) || 0;
    if (savedHistory) history = JSON.parse(savedHistory);

    document.getElementById('break-count').innerText = breaksCount;
    renderTasks();
    updateDashboard();
    renderHistory();
}

// ---------- Helpers ----------

function formatTime(totalSeconds) {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const seconds = totalSeconds % 60;
    if (hours > 0) {
        return `${hours}h ${minutes}m ${seconds}s`;
    }
    return `${minutes}m ${seconds}s`;
}

function todayKey() {
    return new Date().toISOString().slice(0, 10); // YYYY-MM-DD
}

function getTodayRecord() {
    const key = todayKey();
    if (!history[key]) {
        history[key] = { timeSeconds: 0, tasksCompleted: 0, breaks: 0 };
    }
    return history[key];
}

// ---------- Tasks ----------

function addTask() {
    const input = document.getElementById('taskInput');
    const estimateInput = document.getElementById('taskEstimate');
    const name = input.value.trim();
    if (name === "") return;

    const estimateMinutes = parseInt(estimateInput.value, 10);

    const newTask = {
        id: Date.now(),
        name: name,
        timeSpent: 0,
        completed: false,
        estimatedMinutes: (!isNaN(estimateMinutes) && estimateMinutes > 0) ? estimateMinutes : null
    };

    tasks.push(newTask);
    input.value = "";
    estimateInput.value = "";
    input.focus();
    renderTasks();
    updateDashboard();
    saveState();
}

function toggleStartTask(id) {
    if (activeTaskId === id) {
        pauseTask();
    } else {
        pauseTask();
        activeTaskId = id;
        timerId = setInterval(() => {
            const task = tasks.find(t => t.id === activeTaskId);
            if (task) {
                task.timeSpent++;
                continuousSecondsWorking++;
                getTodayRecord().timeSeconds++;
                renderTasks();
                updateDashboard();
                monitorBurnoutRisk();
                renderHistory();
                saveState();
            }
        }, 1000);
    }
    renderTasks();
}

function pauseTask() {
    if (timerId !== null) {
        clearInterval(timerId);
        timerId = null;
    }
    activeTaskId = null;
    renderTasks();
}

function toggleComplete(id) {
    const task = tasks.find(t => t.id === id);
    if (task) {
        task.completed = !task.completed;
        if (task.completed && activeTaskId === id) {
            pauseTask();
        }
        if (task.completed) {
            getTodayRecord().tasksCompleted++;
        } else {
            getTodayRecord().tasksCompleted = Math.max(0, getTodayRecord().tasksCompleted - 1);
        }
        renderTasks();
        updateDashboard();
        renderHistory();
        saveState();
    }
}

function deleteTask(id) {
    if (activeTaskId === id) {
        pauseTask();
    }
    tasks = tasks.filter(t => t.id !== id);
    renderTasks();
    updateDashboard();
    saveState();
}

// ---------- Breaks & Burnout ----------

function takeBreak() {
    pauseTask();
    continuousSecondsWorking = 0;
    hasAlertedHighRisk = false;
    breaksCount++;
    getTodayRecord().breaks++;
    document.getElementById('break-count').innerText = breaksCount;

    const banner = document.getElementById('burnout-banner');
    const status = document.getElementById('burnout-status');
    const msg = document.getElementById('burnout-msg');

    banner.style.borderLeftColor = 'var(--success)';
    banner.style.background = '#EBF5FB';
    status.className = 'status-low';
    status.innerText = 'Resting';
    msg.innerText = 'Break active! Step away, stretch, and give your mind a rest.';
    updateDashboard();
    renderHistory();
    saveState();
}

function notifyHighRisk() {
    if (typeof Notification === 'undefined') return;

    if (Notification.permission === 'granted') {
        new Notification('Time for a break!', {
            body: "You've been working 50+ minutes straight. Step away for a bit."
        });
    } else if (Notification.permission !== 'denied') {
        Notification.requestPermission().then(permission => {
            if (permission === 'granted') {
                new Notification('Time for a break!', {
                    body: "You've been working 50+ minutes straight. Step away for a bit."
                });
            }
        });
    }
}

function monitorBurnoutRisk() {
    const banner = document.getElementById('burnout-banner');
    const status = document.getElementById('burnout-status');
    const msg = document.getElementById('burnout-msg');

    const continuousMinutes = Math.floor(continuousSecondsWorking / 60);

    if (continuousMinutes >= 50) {
        banner.style.borderLeftColor = 'var(--danger)';
        banner.style.background = '#FDEDEC';
        status.className = 'status-high';
        status.innerText = 'HIGH BURNOUT RISK';
        msg.innerText = 'CRITICAL: You have been working for 50+ min straight. Take a break immediately!';

        if (!hasAlertedHighRisk) {
            hasAlertedHighRisk = true;
            notifyHighRisk();
        }
    } else if (continuousMinutes >= 30) {
        banner.style.borderLeftColor = 'var(--warning)';
        banner.style.background = '#FEF9E7';
        status.className = 'status-med';
        status.innerText = 'Moderate Risk';
        msg.innerText = 'Continuous work approaching 30+ min. Plan a break soon.';
    } else {
        banner.style.borderLeftColor = 'var(--success)';
        banner.style.background = '#EBF5FB';
        status.className = 'status-low';
        status.innerText = 'Low Risk';
        msg.innerText = "You're in a good rhythm. Keep going and rest when ready!";
    }
}

// ---------- Dashboard ----------

function updateDashboard() {
    const totalSeconds = tasks.reduce((acc, t) => acc + t.timeSpent, 0);
    document.getElementById('total-time').innerText = formatTime(totalSeconds);

    const totalTasks = tasks.length;
    const completedTasks = tasks.filter(t => t.completed).length;
    document.getElementById('completed-count').innerText = `${completedTasks}/${totalTasks}`;

    if (totalTasks === 0) {
        document.getElementById('efficiency-rate').innerText = '0%';
        return;
    }

    let completionPercentage = (completedTasks / totalTasks) * 100;

    let penalty = 0;
    if (continuousSecondsWorking / 60 > 50) penalty = 20;
    else if (continuousSecondsWorking / 60 > 35) penalty = 10;

    let efficiency = Math.max(0, Math.round(completionPercentage - penalty));
    document.getElementById('efficiency-rate').innerText = `${efficiency}%`;
}

// ---------- Rendering ----------

function renderTasks() {
    const list = document.getElementById('taskList');
    list.innerHTML = "";

    tasks.forEach(task => {
        const isActive = activeTaskId === task.id;
        const isOver = task.estimatedMinutes && task.timeSpent > task.estimatedMinutes * 60;

        const li = document.createElement('li');
        li.className = `task-item ${isActive ? 'active' : ''} ${isOver ? 'over-estimate' : ''}`;

        const estimateText = task.estimatedMinutes
            ? `<span class="estimate-label">/ ${task.estimatedMinutes}m est.</span>`
            : '';

        li.innerHTML = `
            <div class="task-header">
                <div class="task-title ${task.completed ? 'completed' : ''}">
                    <input type="checkbox" ${task.completed ? 'checked' : ''} onchange="toggleComplete(${task.id})">
                    <span>${task.name}</span>
                </div>
            </div>
            <div class="task-footer">
                <span class="time-spent ${isOver ? 'over' : ''}">
                    Time Spent: ${formatTime(task.timeSpent)} ${estimateText}
                </span>
                <div class="task-actions">
                    ${!task.completed ? `
                        <button class="${isActive ? 'btn-pause' : 'btn-start'}" onclick="toggleStartTask(${task.id})">
                            ${isActive ? 'Pause' : 'Start'}
                        </button>
                    ` : ''}
                    <button class="btn-delete" onclick="deleteTask(${task.id})">X</button>
                </div>
            </div>
        `;
        list.appendChild(li);
    });
}

function renderHistory() {
    const list = document.getElementById('historyList');
    list.innerHTML = "";

    const days = Object.keys(history).sort((a, b) => b.localeCompare(a)).slice(0, 7);

    if (days.length === 0) {
        list.innerHTML = '<li class="history-empty">No history yet — start a task to begin tracking.</li>';
        return;
    }

    days.forEach(day => {
        const record = history[day];
        const li = document.createElement('li');
        li.className = 'history-item';
        li.innerHTML = `
            <span class="history-date">${day}</span>
            <span class="history-stats">${formatTime(record.timeSeconds)} · ${record.tasksCompleted} done · ${record.breaks} breaks</span>
        `;
        list.appendChild(li);
    });
}

// ---------- Init ----------

document.getElementById('taskInput').addEventListener('keypress', e => {
    if (e.key === 'Enter') addTask();
});

document.getElementById('taskEstimate').addEventListener('keypress', e => {
    if (e.key === 'Enter') addTask();
});

window.addEventListener('load', loadState);