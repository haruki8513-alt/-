<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>口コミAI監視デモ</title>

  <style>
    * { box-sizing: border-box; }
    body {
      margin: 0;
      padding: 30px 15px;
      background: #f3f6fa;
      color: #1f2937;
      font-family: Arial, "Noto Sans JP", sans-serif;
    }
    .container { max-width: 800px; margin: auto; }
    h1 { margin-bottom: 8px; }
    .description { color: #6b7280; margin-bottom: 24px; }
    .card {
      background: white;
      padding: 24px;
      margin-bottom: 20px;
      border-radius: 14px;
      box-shadow: 0 4px 14px rgba(0,0,0,0.08);
    }
    label { display: block; margin-bottom: 8px; font-weight: bold; }
    textarea {
      width: 100%;
      min-height: 150px;
      padding: 14px;
      border: 1px solid #d1d5db;
      border-radius: 8px;
      font-size: 16px;
      resize: vertical;
    }
    button {
      border: 0;
      border-radius: 8px;
      padding: 12px 18px;
      margin: 12px 8px 0 0;
      color: white;
      font-size: 15px;
      font-weight: bold;
      cursor: pointer;
    }
    button:hover { opacity: 0.85; }
    .analyze-button { background: #2563eb; }
    .sample-button { background: #6b7280; }
    .notify-button { background: #059669; }
    .result {
      display: none;
      padding: 18px;
      border-radius: 10px;
      line-height: 1.8;
    }
    .high { background: #fee2e2; border-left: 6px solid #dc2626; }
    .medium { background: #fef3c7; border-left: 6px solid #d97706; }
    .low { background: #dcfce7; border-left: 6px solid #16a34a; }
    .status {
      display: inline-block;
      padding: 4px 10px;
      border-radius: 999px;
      color: white;
      font-size: 13px;
      font-weight: bold;
    }
    .status.high { background: #dc2626; }
    .status.medium { background: #d97706; }
    .status.low { background: #16a34a; }
    .log {
      padding: 14px;
      min-height: 100px;
      color: #d1d5db;
      background: #111827;
      border-radius: 8px;
      white-space: pre-wrap;
      font-family: monospace;
      font-size: 13px;
    }
    .notice {
      margin-top: 15px;
      padding: 12px;
      background: #ecfdf5;
      border: 1px solid #a7f3d0;
      border-radius: 8px;
      color: #047857;
      display: none;
    }
  </style>
</head>

<body>
  <main class="container">
    <h1>飲食店口コミAI監視デモ</h1>
    <p class="description">口コミを入力すると、問題の有無をAIで判定します。</p>

    <section class="card">
      <label for="review">口コミ本文</label>
      <textarea id="review" placeholder="口コミを入力してください"></textarea>

      <button class="sample-button" onclick="insertSample()">サンプルを入力</button>
      <button class="analyze-button" onclick="analyzeReview()">AIで分析</button>
      <button class="notify-button" onclick="sendNotification()">店主へ通知</button>
    </section>

    <section class="card">
      <h2>分析結果</h2>
      <div id="result" class="result"></div>
      <div id="notice" class="notice"></div>
    </section>

    <section class="card">
      <h2>実行ログ</h2>
      <div id="log" class="log">システムを起動しました。</div>
    </section>
  </main>

  <script>
    let latestResult = null;

    function writeLog(message) {
      const log = document.getElementById("log");
      const time = new Date().toLocaleTimeString("ja-JP");
      log.textContent += `\n[${time}] ${message}`;
      log.scrollTop = log.scrollHeight;
    }

    function insertSample() {
      const sample = "料理の中に髪の毛が入っていました。店員に伝えましたが、謝罪もなく大変残念でした。衛生管理を改善してください。";
      document.getElementById("review").value = sample;
      writeLog("サンプル口コミを入力しました。");
    }

    async function analyzeReview() {
      const review = document.getElementById("review").value.trim();
      if (!review) {
        alert("口コミ本文を入力してください。");
        return;
      }

      writeLog("AIに口コミを送信しています...");

      try {
        const response = await fetch("http://127.0.0.1:5000/analyze", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ review })
        });

        if (!response.ok) {
          throw new Error("AIサーバーでエラーが発生しました");
        }

        const result = await response.json();
        latestResult = result;

        displayResult(result);

        if (result.isProblem) {
          writeLog(`問題を検出しました。重要度: ${result.severity}`);
        } else {
          writeLog("問題は検出されませんでした。");
        }

      } catch (error) {
        console.error(error);
        writeLog("AIへの接続に失敗しました。");
        alert("AIに接続できませんでした。OllamaとPythonサーバーを確認してください。");
      }
    }

    function displayResult(result) {
      const resultBox = document.getElementById("result");
      const severity = result.severity || "low";
      const statusText = result.isProblem ? "問題あり" : "問題なし";

      resultBox.style.display = "block";
      resultBox.className = `result ${severity}`;

      resultBox.innerHTML = `
        <div>
          <span class="status ${severity}">${statusText}</span>
        </div>
        <p><strong>重要度:</strong> ${severity}</p>
        <p><strong>カテゴリ:</strong> ${result.category}</p>
        <p><strong>概要:</strong> ${result.summary}</p>
        <p><strong>推奨対応:</strong> ${result.action}</p>
      `;
    }

    function sendNotification() {
      if (!latestResult) {
        alert("先に口コミを分析してください。");
        return;
      }

      if (!latestResult.isProblem) {
        alert("問題が検出されなかったため、通知しません。");
        writeLog("問題がないため通知をキャンセルしました。");
        return;
      }

      const notice = document.getElementById("notice");
      notice.style.display = "block";
      notice.textContent = "通知テスト完了：店主へ通知した想定です。";

      writeLog(`店主へ通知しました。カテゴリ: ${latestResult.category}`);
    }
  </script>
</body>
</html>
