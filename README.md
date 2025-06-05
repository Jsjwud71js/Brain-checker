<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Philippines Subject Quiz</title>
<style>
  body { font-family: Arial, sans-serif; padding: 20px; max-width: 600px; margin: auto; background: #f9f9f9; }
  button { margin: 5px; padding: 10px 15px; font-size: 16px; cursor: pointer; border-radius: 5px; border: 1px solid #ccc; transition: background-color 0.3s ease; }
  button:disabled { cursor: not-allowed; opacity: 0.6; }
  #answers button { width: 45%; margin: 5px; }
  #timer { font-weight: bold; font-size: 20px; margin-top: 10px; }
  #scoreboard { margin-top: 15px; font-size: 18px; font-weight: bold; }
  #quizContainer { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 0 12px rgba(0,0,0,0.1); }
</style>
</head>
<body>

<h2>Philippines Subject Quiz</h2>

<label>Subject: </label>
<select id="subject">
  <option value="English">English</option>
  <option value="MAPEH">MAPEH</option>
  <option value="Filipino">Filipino</option>
  <option value="Math">Math</option>
  <option value="Science">Science</option>
  <option value="Araling Panlipunan">Araling Panlipunan</option>
  <option value="EPP">EPP</option>
  <option value="ESP">ESP</option>
</select>

<label> Grade Level: </label>
<select id="grade">
  <option value="Grade 1">Grade 1</option>
  <option value="Grade 2">Grade 2</option>
  <option value="Grade 3">Grade 3</option>
  <option value="Grade 4">Grade 4</option>
  <option value="Grade 5">Grade 5</option>
  <option value="Grade 6">Grade 6</option>
  <option value="Grade 7">Grade 7</option>
  <option value="Grade 8">Grade 8</option>
  <option value="Grade 9">Grade 9</option>
  <option value="Grade 10">Grade 10</option>
  <option value="Grade 11">Grade 11</option>
  <option value="Grade 12">Grade 12</option>
  <option value="1st Year College">1st Year College</option>
  <option value="2nd Year College">2nd Year College</option>
  <option value="3rd Year College">3rd Year College</option>
  <option value="4th Year College">4th Year College</option>
  <option value="5th Year College">5th Year College</option>
</select>

<br/><br/>

<label>Seconds per Question (1000 = until answered): </label>
<input type="number" id="timerInput" min="0" max="60" value="10" />

<br/><br/>

<label>Wrong Answers Allowed Before Next Question: </label>
<input type="number" id="maxWrong" min="1" max="10" value="3" />

<br/><br/>

<button id="startBtn">Start Quiz</button>
<button id="stopBtn" disabled>Stop Quiz</button>

<hr/>

<div id="quizContainer" style="display:none;">
  <div id="questionText" style="font-size:18px; margin-bottom:10px;"></div>
  <div id="answers">
    <button id="aBtn" data-choice="a">a.</button>
    <button id="bBtn" data-choice="b">b.</button>
    <button id="cBtn" data-choice="c">c.</button>
    <button id="dBtn" data-choice="d">d.</button>
  </div>
  <br/>
  <button id="showAnswerBtn" disabled>Show Answer</button>
  <div id="timer">Timer: 0</div>
  <div id="scoreboard">Score: 0 | Wrong: 0</div>
</div>

<script>
  const API_KEY = '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'; // Replace with your real key!
  const API_URL = 'https://openrouter.ai/api/v1/chat/completions';

  const startBtn = document.getElementById('startBtn');
  const stopBtn = document.getElementById('stopBtn');
  const subjectSelect = document.getElementById('subject');
  const gradeSelect = document.getElementById('grade');
  const timerInput = document.getElementById('timerInput');
  const maxWrongInput = document.getElementById('maxWrong');
  const quizContainer = document.getElementById('quizContainer');
  const questionText = document.getElementById('questionText');
  const answerButtons = Array.from(document.querySelectorAll('#answers button'));
  const showAnswerBtn = document.getElementById('showAnswerBtn');
  const timerDisplay = document.getElementById('timer');
  const scoreboard = document.getElementById('scoreboard');

  let currentQuestion = null;
  let score = 0;
  let wrongCount = 0;
  let maxWrongAllowed = 3;
  let timePerQuestion = 10;
  let countdown = 0;
  let countdownInterval = null;
  let quizRunning = false;
  let answered = false;

  async function fetchQuestion(subject, grade) {
    const prompt = `Generate ONE multiple-choice question about "${subject}" for "${grade}" level.
Provide question text, 4 options labeled a., b., c., d., each on its own line.
At the end, specify the correct answer as "ANSWER: x" where x is the correct letter.`;

    const res = await fetch(API_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${API_KEY}`
      },
      body: JSON.stringify({
        model: "gpt-4o-mini",
        messages: [{ role: "user", content: prompt }],
        max_tokens: 150,
        temperature: 0.7
      })
    });

    if (!res.ok) throw new Error(`API request failed: ${res.status} ${res.statusText}`);

    const data = await res.json();
    const reply = data.choices?.[0]?.message?.content?.trim();
    if (!reply) throw new Error("No response from API.");

    const lines = reply.split('\n').map(l => l.trim()).filter(l => l.length > 0);

    let questionLines = [];
    const options = {};
    let answer = '';

    for (const line of lines) {
      if (/^a\./i.test(line)) options.a = line.substring(2).trim();
      else if (/^b\./i.test(line)) options.b = line.substring(2).trim();
      else if (/^c\./i.test(line)) options.c = line.substring(2).trim();
      else if (/^d\./i.test(line)) options.d = line.substring(2).trim();
      else if (/^answer:/i.test(line)) answer = line.split(':')[1].trim().toLowerCase();
      else questionLines.push(line);
    }

    const question = questionLines.join(' ');

    if (!question || Object.keys(options).length !== 4 || !answer || !options[answer]) {
      throw new Error("Invalid question format from API.");
    }

    return { question, options, answer };
  }

  function displayQuestion(q) {
    questionText.textContent = q.question;
    answerButtons.forEach(btn => {
      const choice = btn.getAttribute('data-choice');
      btn.textContent = `${choice}. ${q.options[choice]}`;
      btn.disabled = false;
      btn.style.backgroundColor = '';
    });
    showAnswerBtn.disabled = false;
    wrongCount = 0;
    answered = false;
    updateScoreboard();
    if (timePerQuestion > 0) {
      timerDisplay.textContent = `Timer: ${timePerQuestion} ⏳`;
    } else {
      timerDisplay.textContent = `Timer: ∞ (answer to proceed)`;
    }
  }

  function updateScoreboard() {
    scoreboard.textContent = `Score: ${score} | Wrong: ${wrongCount}/${maxWrongAllowed}`;
  }

  function startTimer() {
    clearInterval(countdownInterval);
    if (timePerQuestion <= 0) {
      timerDisplay.textContent = 'Timer: ∞ (answer to proceed) ⏳';
      return;
    }
    countdown = timePerQuestion;
    timerDisplay.textContent = `Timer: ${countdown} ⏳`;
    countdownInterval = setInterval(() => {
      countdown--;
      timerDisplay.textContent = `Timer: ${countdown} ⏳`;
      if (countdown <= 0) {
        clearInterval(countdownInterval);
        // When time runs out, count as a wrong answer
        if (!answered) {
          answered = true;
          handleWrongAnswer();
        }
      }
    }, 1000);
  }

  function handleWrongAnswer() {
    wrongCount++;
    updateScoreboard();
    enableAnswerButtons(false);
    showAnswerBtn.disabled = false;
    if (wrongCount >= maxWrongAllowed) {
      setTimeout(loadNextQuestion, 1500);
    } else {
      // Let user try again on the same question
      setTimeout(() => {
        enableAnswerButtons(true);
        answered = false;
        if (timePerQuestion > 0) {
          countdown = timePerQuestion;
          timerDisplay.textContent = `Timer: ${countdown} ⏳`;
          startTimer();
        }
      }, 1500);
    }
  }

  function enableAnswerButtons(enable) {
    answerButtons.forEach(btn => btn.disabled = !enable);
  }

  function loadNextQuestion() {
    clearInterval(countdownInterval);
    if (!quizRunning) return;
    fetchQuestion(subjectSelect.value, gradeSelect.value)
      .then(q => {
        currentQuestion = q;
        displayQuestion(q);
        if (timePerQuestion > 0) startTimer();
      })
      .catch(err => {
        questionText.textContent = `Error loading question: ${err.message}`;
        enableAnswerButtons(false);
        showAnswerBtn.disabled = true;
      });
  }

  answerButtons.forEach(btn => {
    btn.addEventListener('click', () => {
      if (answered) return;
      answered = true;
      clearInterval(countdownInterval);
      const choice = btn.getAttribute('data-choice');
      if (choice === currentQuestion.answer) {
        score++;
        btn.style.backgroundColor = '#4caf50'; // green for correct
        wrongCount = 0;
      } else {
        wrongCount++;
        btn.style.backgroundColor = '#f44336'; // red for wrong
      }
      updateScoreboard();
      enableAnswerButtons(false);
      showAnswerBtn.disabled = false;

      if (choice === currentQuestion.answer) {
        setTimeout(loadNextQuestion, 1500);
      } else {
        if (wrongCount >= maxWrongAllowed) {
          setTimeout(loadNextQuestion, 1500);
        } else {
          setTimeout(() => {
            enableAnswerButtons(true);
            answered = false;
            if (timePerQuestion > 0) {
              countdown = timePerQuestion;
              timerDisplay.textContent = `Timer: ${countdown} ⏳`;
              startTimer();
            }
          }, 1500);
        }
      }
    });
  });

  showAnswerBtn.addEventListener('click', () => {
    alert(`Correct answer is: ${currentQuestion.answer.toUpperCase()}. ${currentQuestion.options[currentQuestion.answer]}`);
    showAnswerBtn.disabled = true;
  });

  startBtn.addEventListener('click', () => {
    score = 0;
    wrongCount = 0;
    maxWrongAllowed = parseInt(maxWrongInput.value) || 3;
    timePerQuestion = parseInt(timerInput.value) || 10;
    quizRunning = true;
    startBtn.disabled = true;
    stopBtn.disabled = false;
    quizContainer.style.display = 'block';
    loadNextQuestion();
  });

  stopBtn.addEventListener('click', () => {
    quizRunning = false;
    clearInterval(countdownInterval);
    startBtn.disabled = false;
    stopBtn.disabled = true;
    quizContainer.style.display = 'none';
    questionText.textContent = '';
    enableAnswerButtons(false);
    showAnswerBtn.disabled = true;
    timerDisplay.textContent = '';
    scoreboard.textContent = '';
  });
</script>

</body>
</html>
