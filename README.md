<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>LOCKWORD</title>

<style>
body {
  font-family: system-ui, sans-serif;
  background: #121212;
  color: #fff;
  text-align: center;
  padding: 20px;
}

#board {
  display: grid;
  grid-template-columns: repeat(5, 60px);
  gap: 8px;
  justify-content: center;
  margin: 20px auto;
}

.cell {
  width: 60px;
  height: 60px;
  border-radius: 8px;
  font-size: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #2a2a2a;
}

.lock { background: #2ecc71; }
.keyed { background: #f1c40f; color: #000; }
.boom { background: #555; }

#keyboard {
  display: grid;
  grid-template-columns: repeat(10, 1fr);
  gap: 6px;
  max-width: 500px;
  margin: 20px auto;
}

.key {
  padding: 12px;
  background: #333;
  border-radius: 6px;
  cursor: pointer;
  user-select: none;
}

.key.lock { background: #2ecc71; }
.key.keyed { background: #f1c40f; color: #000; }
.key.boom { background: #555; }

#stats {
  margin-top: 30px;
  border-top: 1px solid #333;
  padding-top: 20px;
  max-width: 400px;
  margin-left: auto;
  margin-right: auto;
}
</style>
</head>

<body>

<h1>🔐 LOCKWORD</h1>
<p>Daily Puzzle</p>

<div id="board"></div>
<div id="keyboard"></div>
<div id="message"></div>

<div id="stats"></div>

<script>
/* ------------------ WORD DATA ------------------ */
const ANSWERS = [
  "GLYPH","FJORD","PLUMB","MYTHS","CRISP",
  "NERVE","WHISK","CROWN","BLINK","SHARD"
];

const MAX_TURNS = 6;
const WORD_LENGTH = 5;

/* ------------------ DAILY WORD ------------------ */
function getDailyWord() {
  const start = new Date("2024-01-01");
  const today = new Date();
  const diff = Math.floor((today - start) / 86400000);
  return ANSWERS[diff % ANSWERS.length];
}

const todayKey = new Date().toDateString();
let secret = getDailyWord();

/* ------------------ GAME STATE ------------------ */
let turn = 0;
let locked = Array(5).fill(null);
let banned = new Set();
let finished = false;

const board = document.getElementById("board");
const message = document.getElementById("message");

/* ------------------ KEYBOARD ------------------ */
const keyboardLayout = [
  ..."QWERTYUIOP",
  ..."ASDFGHJKL",
  "ENTER", ..."ZXCVBNM", "⌫"
];

const keyboard = document.getElementById("keyboard");
const keyStates = {};

keyboardLayout.forEach(k => {
  const btn = document.createElement("div");
  btn.textContent = k;
  btn.className = "key";
  btn.onclick = () => handleKey(k);
  keyboard.appendChild(btn);
});

let currentGuess = "";

function handleKey(key) {
  if (finished) return;

  if (key === "ENTER") submitGuess();
  else if (key === "⌫") currentGuess = currentGuess.slice(0, -1);
  else if (currentGuess.length < 5) currentGuess += key;
}

/* ------------------ EVALUATION ------------------ */
function evaluateGuess(secret, guess) {
  let result = Array(5).fill("");
  let remaining = secret.split("");

  for (let i = 0; i < 5; i++) {
    if (guess[i] === secret[i]) {
      result[i] = "lock";
      remaining[i] = null;
    }
  }

  for (let i = 0; i < 5; i++) {
    if (!result[i]) {
      const idx = remaining.indexOf(guess[i]);
      if (idx !== -1) {
        result[i] = "keyed";
        remaining[idx] = null;
      } else {
        result[i] = "boom";
      }
    }
  }
  return result;
}

function updateKeyboard(letter, state) {
  [...keyboard.children].forEach(k => {
    if (k.textContent === letter && !k.classList.contains("lock")) {
      k.className = `key ${state}`;
    }
  });
}

/* ------------------ SUBMIT GUESS ------------------ */
function submitGuess() {
  if (currentGuess.length !== 5) return;

  for (let i = 0; i < 5; i++) {
    if (locked[i] && currentGuess[i] !== locked[i]) {
      message.textContent = `Position ${i+1} locked`;
      return;
    }
  }

  if ([...currentGuess].some(l => banned.has(l))) {
    turn++;
    currentGuess = "";
    return;
  }

  const feedback = evaluateGuess(secret, currentGuess);

  feedback.forEach((type, i) => {
    const cell = document.createElement("div");
    cell.className = `cell ${type}`;
    cell.textContent = currentGuess[i];
    board.appendChild(cell);

    if (type === "lock") locked[i] = currentGuess[i];
    if (type === "boom") banned.add(currentGuess[i]);

    updateKeyboard(currentGuess[i], type);
  });

  turn++;

  if (currentGuess === secret) {
    endGame(true);
  } else if (turn >= MAX_TURNS) {
    endGame(false);
  }

  currentGuess = "";
}

/* ------------------ ANALYTICS ------------------ */
const defaultStats = {
  played: 0,
  wins: 0,
  streak: 0,
  maxStreak: 0,
  guesses: [0,0,0,0,0,0],
  lastPlayed: null
};

let stats = JSON.parse(localStorage.getItem("lockwordStats")) || defaultStats;

function endGame(win) {
  finished = true;

  if (stats.lastPlayed !== todayKey) {
    stats.played++;

    if (win) {
      stats.wins++;
      stats.streak++;
      stats.maxStreak = Math.max(stats.streak, stats.maxStreak);
      stats.guesses[turn - 1]++;
      message.textContent = "🎉 You cracked the lock!";
    } else {
      stats.streak = 0;
      message.textContent = `🔓 Word was ${secret}`;
    }

    stats.lastPlayed = todayKey;
    localStorage.setItem("lockwordStats", JSON.stringify(stats));
  }

  renderStats();
}

/* ------------------ SCOREBOARD ------------------ */
function renderStats() {
  const winPct = stats.played
    ? Math.round((stats.wins / stats.played) * 100)
    : 0;

  document.getElementById("stats").innerHTML = `
    <h2>📊 Stats</h2>
    <p>Played: ${stats.played}</p>
    <p>Wins: ${stats.wins}</p>
    <p>Win %: ${winPct}</p>
    <p>Streak: ${stats.streak}</p>
    <p>Max Streak: ${stats.maxStreak}</p>
    <h3>Guess Distribution</h3>
    ${stats.guesses.map((g,i) =>
      `<div>${i+1}: ${"█".repeat(g)}</div>`
    ).join("")}
  `;
}

renderStats();
</script>

</body>
</html>
