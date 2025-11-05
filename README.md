<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>ミャクミャク英単語マスター 完全版＋スコア履歴＋正答率グラフ</title>
<style>
body {
  font-family: "Segoe UI", sans-serif;
  background: linear-gradient(120deg, #ffe0f0, #e0f7ff);
  text-align: center;
  margin: 0;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
}
button {
  padding: 18px 40px;
  margin: 8px;
  border: none;
  border-radius: 20px;
  background: linear-gradient(135deg, #ffd0a0, #ffd0ff);
  font-size: 18px;
  cursor: pointer;
}
button:hover { transform: scale(1.05); transition: 0.2s; }
.screen { display: none; width: 100%; }
.active { display: flex !important; flex-direction: column; align-items: center; justify-content: center; }

/* ホーム画面 */
#home-screen h1 {
  font-size: 52px;
  line-height: 1.3;
  margin-bottom: 30px;
}

/* テスト画面 */
#game-screen { text-align: center; }
#word-card {
  background: linear-gradient(135deg, #ffd0d0, #d0f0ff);
  border-radius: 25px;
  padding: 25px;
  font-size: 34px;
  width: 70%;
  margin: 20px auto;
}
input[type=text] {
  padding: 12px;
  font-size: 20px;
  width: 80%;
  border: 2px solid #ccc;
  border-radius: 15px;
}
#feedback { font-size: 26px; font-weight: bold; height: 40px; margin: 10px; }

/* 結果画面 */
#result-screen table {
  width: 80%;
  margin-top: 10px;
  border-collapse: collapse;
}
#result-screen th, #result-screen td {
  border: 1px solid #999;
  padding: 8px;
  font-size: 16px;
}
#result-screen th {
  background-color: #ffd0a0;
}
#history {
  margin-top: 30px;
  width: 80%;
}
#history h3 { margin-bottom: 5px; }
#history table {
  width: 100%;
  border-collapse: collapse;
}
#history th, #history td {
  border: 1px solid #999;
  padding: 6px;
  font-size: 14px;
}
#history th { background-color: #ffd0ff; }

/* グラフ */
#chart-container {
  margin-top: 30px;
  width: 80%;
}
canvas {
  background: #fff;
  border-radius: 10px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}
</style>
</head>
<body>

<!-- 🏠 ホーム画面 -->
<div id="home-screen" class="screen active">
  <h1>ミャクミャク<br>英単語マスター</h1>
  <button id="mode-ja-en">日本語 → 英語</button>
  <button id="mode-en-ja">英語 → 日本語</button>
  <button id="review-btn">復習モード</button>
</div>

<!-- 🧠 テスト画面 -->
<div id="game-screen" class="screen">
  <h2 id="mode-title"></h2>
  <p>スコア: <span id="score">0</span>　残り: <span id="remaining">10</span></p>
  <div id="word-card">Ready?</div>
  <input type="text" id="answer" placeholder="答えを入力…" autocomplete="off">
  <div id="feedback"></div>
  <div>
    <button id="submit-btn">採点</button>
    <button id="next-btn">次へ</button>
    <button id="back-btn">前へ</button>
    <button id="skip-btn">スキップ</button>
  </div>
  <button id="home-btn">ホームに戻る</button>
</div>

<!-- 🏁 結果画面 -->
<div id="result-screen" class="screen">
  <h2>テスト結果</h2>
  <p>スコア: <span id="final-score">0</span></p>
  <p>正答数: <span id="correct-count">0</span> / <span id="total-count">0</span></p>
  <p>間違えた単語:</p>
  <table>
    <thead><tr><th>日本語</th><th>英語</th></tr></thead>
    <tbody id="wrong-list"></tbody>
  </table>

  <!-- 🕒 スコア履歴 -->
  <div id="history">
    <h3>過去のスコア履歴（最大10件）</h3>
    <table>
      <thead><tr><th>日付</th><th>モード</th><th>スコア</th><th>正答数</th></tr></thead>
      <tbody id="history-list"></tbody>
    </table>
  </div>

  <!-- 📊 正答率グラフ -->
  <div id="chart-container">
    <h3>正答率の推移（％）</h3>
    <canvas id="accuracyChart" width="500" height="250"></canvas>
  </div>

  <button id="retry-btn">もう一度挑戦</button>
  <button id="home-btn2">ホームに戻る</button>
</div>

<script>
document.addEventListener("DOMContentLoaded", () => {
  const words = [
    { en:"apple", ja:"りんご" },
    { en:"school", ja:"学校" },
    { en:"book", ja:"本" },
    { en:"friend", ja:"友達" },
    { en:"music", ja:"音楽" },
    { en:"teacher", ja:"先生" },
    { en:"family", ja:"家族" },
    { en:"future", ja:"未来" },
    { en:"dream", ja:"夢" },
    { en:"mountain", ja:"山" }
  ];

  const screens = {
    home: document.getElementById("home-screen"),
    game: document.getElementById("game-screen"),
    result: document.getElementById("result-screen")
  };
  function show(screen) {
    Object.values(screens).forEach(s => s.classList.remove("active"));
    screens[screen].classList.add("active");
  }

  let currentMode = "ja-en";
  let used = [];
  let mistakes = [];
  let score = 0;
  let correctCount = 0;
  let currentWord;

  const scoreEl = document.getElementById("score");
  const remainingEl = document.getElementById("remaining");
  const wordEl = document.getElementById("word-card");
  const answerEl = document.getElementById("answer");
  const feedbackEl = document.getElementById("feedback");
  const modeTitle = document.getElementById("mode-title");

  function startGame(mode, review=false) {
    currentMode = mode;
    const savedMistakes = JSON.parse(localStorage.getItem("mistakes") || "[]");
    mistakes = review ? savedMistakes : [];
    if (review && mistakes.length === 0) {
      alert("復習する単語がありません。");
      return;
    }
    used = [];
    score = 0;
    correctCount = 0;
    modeTitle.textContent = mode === "ja-en" ? "日本語 → 英語" : mode === "en-ja" ? "英語 → 日本語" : "復習モード";
    show("game");
    nextWord();
  }

  function nextWord() {
    const pool = (currentMode === "review" ? mistakes : words).filter(w => !used.includes(w));
    if (pool.length === 0) return showResult();
    currentWord = pool[Math.floor(Math.random() * pool.length)];
    used.push(currentWord);
    wordEl.textContent = currentMode === "ja-en" ? currentWord.ja : currentWord.en;
    answerEl.value = "";
    feedbackEl.textContent = "";
    scoreEl.textContent = score;
    remainingEl.textContent = 10 - used.length;
  }

  function prevWord() {
    if (used.length > 1) {
      used.pop();
      currentWord = used[used.length - 1];
      wordEl.textContent = currentMode === "ja-en" ? currentWord.ja : currentWord.en;
      feedbackEl.textContent = "";
      answerEl.value = "";
    }
  }

  function checkAnswer() {
    const ans = answerEl.value.trim().toLowerCase();
    const correct = currentMode === "ja-en" ? currentWord.en.toLowerCase() : currentWord.ja.toLowerCase();
    if (ans === correct) {
      feedbackEl.textContent = "✅ 正解！";
      feedbackEl.style.color = "green";
      score += 10; correctCount++;
    } else {
      feedbackEl.textContent = "❌ 不正解！ 正答: " + correct;
      feedbackEl.style.color = "red";
      mistakes.push(currentWord);
    }
  }

  function showResult() {
    localStorage.setItem("mistakes", JSON.stringify(mistakes));
    document.getElementById("final-score").textContent = score;
    document.getElementById("correct-count").textContent = correctCount;
    document.getElementById("total-count").textContent = used.length;

    const tbody = document.getElementById("wrong-list");
    tbody.innerHTML = "";
    mistakes.forEach(w => {
      const tr = document.createElement("tr");
      tr.innerHTML = `<td>${w.ja}</td><td>${w.en}</td>`;
      tbody.appendChild(tr);
    });

    saveHistory();
    loadHistory();
    drawChart();
    show("result");
  }

  // 🧠 履歴保存
  function saveHistory() {
    const history = JSON.parse(localStorage.getItem("scoreHistory") || "[]");
    const total = used.length || 1;
    const accuracy = Math.round((correctCount / total) * 100);
    const entry = {
      date: new Date().toLocaleString(),
      mode: modeTitle.textContent,
      score: score,
      correct: correctCount,
      accuracy: accuracy
    };
    history.push(entry);
    if (history.length > 10) history.shift();
    localStorage.setItem("scoreHistory", JSON.stringify(history));
  }

  // 📋 履歴読み込み
  function loadHistory() {
    const history = JSON.parse(localStorage.getItem("scoreHistory") || "[]");
    const tbody = document.getElementById("history-list");
    tbody.innerHTML = "";
    history.slice().reverse().forEach(h => {
      const tr = document.createElement("tr");
      tr.innerHTML = `<td>${h.date}</td><td>${h.mode}</td><td>${h.score}</td><td>${h.correct}</td>`;
      tbody.appendChild(tr);
    });
  }

  // 📊 グラフ描画
  function drawChart() {
    const history = JSON.parse(localStorage.getItem("scoreHistory") || "[]");
    const ctx = document.getElementById("accuracyChart").getContext("2d");
    ctx.clearRect(0, 0, 500, 250);
    const accuracies = history.map(h => h.accuracy);
    const barWidth = 40;
    const gap = 20;
    const startX = 40;
    const baseY = 230;
    ctx.font = "12px sans-serif";
    ctx.fillStyle = "#333";
    ctx.fillText("0%", 5, baseY);
    ctx.fillText("100%", 0, 30);
    accuracies.forEach((acc, i) => {
      const x = startX + i * (barWidth + gap);
      const height = (acc / 100) * 200;
      ctx.fillStyle = "#66aaff";
      ctx.fillRect(x, baseY - height, barWidth, height);
      ctx.fillStyle = "#333";
      ctx.fillText(`${acc}%`, x, baseY - height - 5);
    });
  }

  // イベント
  document.getElementById("mode-ja-en").onclick = () => startGame("ja-en");
  document.getElementById("mode-en-ja").onclick = () => startGame("en-ja");
  document.getElementById("review-btn").onclick = () => startGame("review", true);
  document.getElementById("submit-btn").onclick = checkAnswer;
  document.getElementById("next-btn").onclick = nextWord;
  document.getElementById("back-btn").onclick = prevWord;
  document.getElementById("skip-btn").onclick = nextWord;
  document.getElementById("home-btn").onclick = () => show("home");
  document.getElementById("home-btn2").onclick = () => show("home");
  document.getElementById("retry-btn").onclick = () => startGame(currentMode);
});
</script>
</body>
</html>
