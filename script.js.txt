const STORAGE_KEY = "schoolQuestData";

const practiceTools = [
  {
    name: "IXL Math",
    subject: "Math",
    link: "https://www.ixl.com/math"
  },
  {
    name: "IXL English",
    subject: "English",
    link: "https://www.ixl.com/ela"
  },
  {
    name: "Je Lis Libre",
    subject: "French Immersion Reading",
    link: "https://jelislibre.leslibraires.ca"
  },
  {
    name: "Duolingo",
    subject: "French Language Practice",
    link: "https://www.duolingo.com"
  },
  {
    name: "Math-Aids",
    subject: "Math Practice Sheets",
    link: "https://www.math-aids.com"
  }
];

const state = loadState();

const pointsTotal = document.getElementById("pointsTotal");
const streakTotal = document.getElementById("streakTotal");
const taskForm = document.getElementById("taskForm");
const taskList = document.getElementById("taskList");
const taskTemplate = document.getElementById("taskTemplate");
const practiceList = document.getElementById("practiceList");
const rewardForm = document.getElementById("rewardForm");
const rewardList = document.getElementById("rewardList");

init();

function init() {
  renderPracticeTools();
  renderTasks();
  renderRewards();
  updateHeader();

  taskForm.addEventListener("submit", (event) => {
    event.preventDefault();

    const task = {
      id: crypto.randomUUID(),
      subject: document.getElementById("subject").value,
      title: document.getElementById("title").value.trim(),
      dueDate: document.getElementById("dueDate").value,
      progress: document.getElementById("progress").value
    };

    state.tasks.unshift(task);
    addPoints(5, "Added a quest");
    markUsage();
    persistAndRender();
    taskForm.reset();
  });

  rewardForm.addEventListener("submit", (event) => {
    event.preventDefault();
    const name = document.getElementById("rewardName").value.trim();
    const cost = Number(document.getElementById("rewardCost").value);

    if (!name || cost <= 0) {
      return;
    }

    state.rewards.push({ id: crypto.randomUUID(), name, cost });
    persistAndRender();
    rewardForm.reset();
  });
}

function renderTasks() {
  taskList.innerHTML = "";

  if (!state.tasks.length) {
    taskList.innerHTML = `<p class="hint">No quests yet. Add one to get started!</p>`;
    return;
  }

  state.tasks.forEach((task) => {
    const node = taskTemplate.content.cloneNode(true);
    node.querySelector(".task-title").textContent = task.title;
    node.querySelector(".meta").textContent = `${task.subject} • ${task.progress}`;
    node.querySelector(".due").textContent = `Due: ${formatDate(task.dueDate)}`;

    const progressSelect = node.querySelector(".task-progress");
    progressSelect.value = task.progress;
    progressSelect.addEventListener("change", (event) => {
      const newValue = event.target.value;
      const oldValue = task.progress;
      task.progress = newValue;

      if (newValue === "Done" && oldValue !== "Done") {
        addPoints(10, "Completed a quest");
      }

      markUsage();
      persistAndRender();
    });

    const deleteBtn = node.querySelector(".delete");
    deleteBtn.addEventListener("click", () => {
      state.tasks = state.tasks.filter((item) => item.id !== task.id);
      persistAndRender();
    });

    taskList.appendChild(node);
  });
}

function renderPracticeTools() {
  practiceList.innerHTML = "";
  practiceTools.forEach((tool) => {
    const li = document.createElement("li");
    li.innerHTML = `<strong>${tool.name}</strong><br>${tool.subject}<br><a href="${tool.link}" target="_blank" rel="noopener noreferrer">Open practice tool</a>`;
    practiceList.appendChild(li);
  });
}

function renderRewards() {
  rewardList.innerHTML = "";

  if (!state.rewards.length) {
    rewardList.innerHTML = '<p class="hint">Add your first reward for motivation.</p>';
    return;
  }

  state.rewards.forEach((reward) => {
    const row = document.createElement("div");
    row.className = "reward-row";
    row.innerHTML = `<span>${reward.name} (${reward.cost} pts)</span>`;

    const button = document.createElement("button");
    button.textContent = "Redeem";
    button.disabled = state.points < reward.cost;
    button.addEventListener("click", () => {
      if (state.points >= reward.cost) {
        state.points -= reward.cost;
        persistAndRender();
      }
    });

    row.appendChild(button);
    rewardList.appendChild(row);
  });
}

function updateHeader() {
  pointsTotal.textContent = state.points;
  streakTotal.textContent = `${state.streak} day${state.streak === 1 ? "" : "s"}`;
}

function persistAndRender() {
  saveState(state);
  renderTasks();
  renderRewards();
  updateHeader();
}

function addPoints(amount) {
  state.points += amount;
}

function markUsage() {
  const today = new Date().toISOString().slice(0, 10);
  if (state.lastUsageDate === today) {
    return;
  }

  if (state.lastUsageDate) {
    const prev = new Date(state.lastUsageDate);
    const curr = new Date(today);
    const diff = Math.round((curr - prev) / (1000 * 60 * 60 * 24));
    state.streak = diff === 1 ? state.streak + 1 : 1;
  } else {
    state.streak = 1;
  }

  state.lastUsageDate = today;

  if (state.streak > 0 && state.streak % 5 === 0) {
    addPoints(15, "5-day streak bonus");
  }
}

function loadState() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) {
      return defaultState();
    }

    return { ...defaultState(), ...JSON.parse(raw) };
  } catch {
    return defaultState();
  }
}

function saveState(nextState) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(nextState));
}

function defaultState() {
  return {
    tasks: [],
    points: 0,
    streak: 0,
    lastUsageDate: null,
    rewards: [
      { id: crypto.randomUUID(), name: "Choose family movie night", cost: 50 },
      { id: crypto.randomUUID(), name: "Extra 30 mins game time", cost: 40 }
    ]
  };
}

function formatDate(value) {
  if (!value) {
    return "No due date";
  }

  return new Date(`${value}T00:00:00`).toLocaleDateString(undefined, {
    weekday: "short",
    month: "short",
    day: "numeric"
  });
}
