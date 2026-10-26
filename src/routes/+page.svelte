<!--
  ============================================================
  src/routes/+page.svelte
  バックグラウンド録音テストページ（otorec ステップ0）

  目的：
    Android の Chrome で、ページを裏に回したり画面を消したりしても
    録音が続くかどうかを「データで」確かめる。

  しくみ：
    - マイクの音を MediaRecorder で録音する（1秒ごとに小分けにして受け取る）
    - 同時に「記録係」が1秒ごとに 時刻・音の大きさ・画面が表か裏か をメモする
    - 裏に回った／表に戻った／マイクが止められた などの出来事も記録する
    - 停止したら録音を解析して、
        ・実際に録音できた長さ（壁時計の経過時間と比べる）
        ・1秒ごとの音の大きさ（裏にいた時間帯に音が入っているか）
      を表示する

  判断の考え方：
    裏ではタイマー（setInterval）が間引かれることがあるので、
    「記録係のメモ」よりも「録音そのものの長さと中身」を一番の証拠にする。
  ============================================================
-->
<script>
  import { onDestroy } from "svelte";

  // ============================================================
  // 画面に表示する状態（$state で変わると画面も変わる）
  // ============================================================

  /** テストの状態：'idle'(未開始) / 'recording'(録音中) / 'processing'(集計中) / 'stopped'(終了) */
  let status = $state("idle");

  /** エラーが起きたときの説明文 */
  let errorMessage = $state("");

  /** 今の音の大きさ（dB、メーター表示用） */
  let currentLevel = $state(-100);

  /** 今ページが表に見えているかどうか */
  let isVisible = $state(true);

  /** 録音開始からの経過秒数 */
  let elapsedSec = $state(0);

  /** 受け取った録音データの合計サイズ（バイト） */
  let totalBytes = $state(0);

  /** 受け取った録音データの個数（1秒ごとに1個のはず） */
  let chunkCount = $state(0);

  /** 記録係の1秒ごとのメモ { at:ミリ秒, level:dB, visible:真偽, ctxState:文字列 } */
  let tickLogs = $state([]);

  /** 出来事の記録 { at:ミリ秒, type:種類, text:説明 } */
  let events = $state([]);

  /** 裏にいた時間帯の一覧 { start:ミリ秒, end:ミリ秒 } */
  let hiddenPeriods = $state([]);

  /** 録音を再生するための URL */
  let audioUrl = $state("");

  /** 録音を解析して分かった「1秒ごとの音の大きさ」 { sec:秒, db:dB, hidden:真偽 } */
  let perSecondLevels = $state([]);

  /** 停止後の集計結果 */
  let summary = $state(null);

  /** 「結果をコピー」したときのお知らせ */
  let copyMessage = $state("");

  // ============================================================
  // 画面に出さない作業用の変数（$state にしない）
  // ============================================================

  /** マイクの音の流れ */
  let stream = null;
  /** 音の大きさを測るための Web Audio の部品 */
  let audioContext = null;
  let analyser = null;
  let sourceNode = null;
  /** 音の大きさを測るときに使う入れ物 */
  let levelBuffer = null;
  /** 録音係 */
  let mediaRecorder = null;
  /** 小分けの録音データ */
  let chunks = [];
  /** 録音データが届いた時刻の記録 { at, bytes, visible } */
  let chunkArrivals = [];
  /** 記録係のタイマー */
  let tickTimer = null;
  /** メーター更新のアニメーション番号 */
  let meterFrame = null;
  /** 録音を開始した時刻（Date.now() の値） */
  let startTime = 0;
  /** 今の「裏にいる時間帯」が始まった時刻（表にいるときは null） */
  let hiddenStart = null;
  /** 停止した時刻（開始からのミリ秒） */
  let stopAt = 0;

  // ============================================================
  // 小さな道具の関数
  // ============================================================

  /**
   * nowRel：録音開始から今までのミリ秒を返す
   * @returns {number}
   */
  function nowRel() {
    return Date.now() - startTime;
  }

  /**
   * formatTime：ミリ秒を「分:秒」の文字にする（例：65000 → "1:05"）
   * @param {number} ms
   * @returns {string}
   */
  function formatTime(ms) {
    const totalSec = Math.floor(ms / 1000);
    const m = Math.floor(totalSec / 60);
    const s = totalSec % 60;
    return `${m}:${String(s).padStart(2, "0")}`;
  }

  /**
   * formatBytes：バイト数を読みやすい文字にする（例：20480 → "20.0 KB"）
   * @param {number} bytes
   * @returns {string}
   */
  function formatBytes(bytes) {
    if (bytes < 1024) return `${bytes} B`;
    if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
    return `${(bytes / 1024 / 1024).toFixed(2)} MB`;
  }

  /**
   * rmsToDb：音の波の「平均的な強さ(RMS)」を dB に変える
   * 完全な無音（0）は -100dB として扱う
   * @param {number} rms
   * @returns {number}
   */
  function rmsToDb(rms) {
    if (rms <= 0) return -100;
    const db = 20 * Math.log10(rms);
    return Math.max(-100, db);
  }

  /**
   * dbToPercent：dB をメーターや棒グラフの高さ（0〜100%）に変える
   * -80dB 以下は 0%、0dB で 100%
   * @param {number} db
   * @returns {number}
   */
  function dbToPercent(db) {
    const p = ((db + 80) / 80) * 100;
    return Math.min(100, Math.max(0, p));
  }

  /**
   * measureLevel：マイクの今の音の大きさを測って dB で返す
   * @returns {number}
   */
  function measureLevel() {
    if (!analyser || !levelBuffer) return -100;
    // 今の音の波を入れ物に写す
    analyser.getFloatTimeDomainData(levelBuffer);
    // 2乗の平均の平方根（RMS）を求める
    let sum = 0;
    for (let i = 0; i < levelBuffer.length; i++) {
      sum += levelBuffer[i] * levelBuffer[i];
    }
    const rms = Math.sqrt(sum / levelBuffer.length);
    return rmsToDb(rms);
  }

  /**
   * addEvent：出来事を記録に追加する
   * @param {string} type 種類（hidden / visible / mute / unmute / ended / ctx / freeze / resume / error / info）
   * @param {string} text 説明
   */
  function addEvent(type, text) {
    events.push({ at: status === "idle" ? 0 : nowRel(), type, text });
  }

  // ============================================================
  // ページの表・裏の切り替わりなどを見張る関数
  // ============================================================

  /**
   * handleVisibilityChange：ページが表に見える／裏に回ったときに呼ばれる
   * 裏にいた時間帯を hiddenPeriods に記録する
   */
  function handleVisibilityChange() {
    const visible = document.visibilityState === "visible";
    isVisible = visible;
    if (!visible) {
      // 裏に回った：時間帯の始まりを覚えておく
      hiddenStart = nowRel();
      addEvent("hidden", "裏に回った（ホーム画面・別アプリ・画面オフなど）");
    } else {
      // 表に戻った：時間帯を閉じて記録する
      if (hiddenStart !== null) {
        hiddenPeriods.push({ start: hiddenStart, end: nowRel() });
        hiddenStart = null;
      }
      addEvent("visible", "表に戻った");
    }
  }

  /**
   * handleFreeze：Chrome がページを「凍結（一時停止）」したときに呼ばれる
   * 凍結されると JavaScript が止まるので、録音に影響が出る可能性がある
   */
  function handleFreeze() {
    addEvent("freeze", "ページが凍結された（Chrome がページを一時停止）");
  }

  /**
   * handleResume：凍結から戻ったときに呼ばれる
   */
  function handleResume() {
    addEvent("resume", "ページの凍結が解けた");
  }

  /**
   * addPageListeners：ページの表・裏や凍結の見張りを始める
   */
  function addPageListeners() {
    document.addEventListener("visibilitychange", handleVisibilityChange);
    document.addEventListener("freeze", handleFreeze);
    document.addEventListener("resume", handleResume);
  }

  /**
   * removePageListeners：ページの表・裏や凍結の見張りをやめる
   */
  function removePageListeners() {
    document.removeEventListener("visibilitychange", handleVisibilityChange);
    document.removeEventListener("freeze", handleFreeze);
    document.removeEventListener("resume", handleResume);
  }

  // ============================================================
  // 記録係とメーター
  // ============================================================

  /**
   * tick：記録係。1秒ごとに今の様子をメモする
   * （裏ではブラウザがタイマーを間引くので、メモの間隔が空くこともある。
   *   それ自体も「裏でどう扱われたか」の手がかりになる）
   */
  function tick() {
    const at = nowRel();
    elapsedSec = Math.floor(at / 1000);
    tickLogs.push({
      at,
      level: measureLevel(),
      visible: document.visibilityState === "visible",
      ctxState: audioContext ? audioContext.state : "-",
    });
  }

  /**
   * meterLoop：画面のメーターをなめらかに動かすための繰り返し処理
   * （裏では止まるが、メーターは表にいるときしか見ないので問題ない）
   */
  function meterLoop() {
    currentLevel = measureLevel();
    meterFrame = requestAnimationFrame(meterLoop);
  }

  // ============================================================
  // 開始・停止
  // ============================================================

  /**
   * resetAll：前回のテスト結果をすべて消して、まっさらな状態にする
   */
  function resetAll() {
    errorMessage = "";
    currentLevel = -100;
    isVisible = document.visibilityState === "visible";
    elapsedSec = 0;
    totalBytes = 0;
    chunkCount = 0;
    tickLogs = [];
    events = [];
    hiddenPeriods = [];
    perSecondLevels = [];
    summary = null;
    copyMessage = "";
    if (audioUrl) URL.revokeObjectURL(audioUrl);
    audioUrl = "";
    chunks = [];
    chunkArrivals = [];
    hiddenStart = null;
    stopAt = 0;
  }

  /**
   * pickMimeType：このブラウザで使える録音形式を選ぶ
   * Android Chrome なら 'audio/webm;codecs=opus' になるはず
   * @returns {string} 使える形式（見つからなければ空文字＝ブラウザ任せ）
   */
  function pickMimeType() {
    const candidates = ["audio/webm;codecs=opus", "audio/webm", "audio/mp4"];
    for (const type of candidates) {
      if (MediaRecorder.isTypeSupported(type)) return type;
    }
    return "";
  }

  /**
   * startTest：テストを開始する
   * マイクの許可 → 音の大きさを測る準備 → 録音開始 → 記録係スタート
   */
  async function startTest() {
    resetAll();

    // ---- マイクの使用を許可してもらう ----
    try {
      stream = await navigator.mediaDevices.getUserMedia({
        audio: {
          // 本体アプリでは楽器や生活音も録りたいので、
          // 声向けの自動調整（エコー除去・雑音除去・音量自動調整）は切っておく
          echoCancellation: false,
          noiseSuppression: false,
          autoGainControl: false,
        },
      });
    } catch (err) {
      errorMessage = `マイクを使えませんでした：${err.name}（${err.message}）`;
      return;
    }

    // ---- 音の大きさを測る準備 ----
    audioContext = new AudioContext();
    sourceNode = audioContext.createMediaStreamSource(stream);
    analyser = audioContext.createAnalyser();
    analyser.fftSize = 2048;
    levelBuffer = new Float32Array(analyser.fftSize);
    sourceNode.connect(analyser);
    // AudioContext が止められた（suspended）などの変化も記録する
    audioContext.onstatechange = () => {
      addEvent("ctx", `音の解析部品の状態が「${audioContext.state}」になった`);
    };

    // ---- マイクが止められた・戻った・終わったを見張る ----
    const track = stream.getAudioTracks()[0];
    track.onmute = () => addEvent("mute", "マイクが止められた（無音になった）");
    track.onunmute = () => addEvent("unmute", "マイクが戻った");
    track.onended = () => addEvent("ended", "マイクの使用が終了させられた");

    // ---- 録音係の準備 ----
    const mimeType = pickMimeType();
    try {
      mediaRecorder = mimeType ? new MediaRecorder(stream, { mimeType }) : new MediaRecorder(stream);
    } catch (err) {
      errorMessage = `録音を始められませんでした：${err.name}（${err.message}）`;
      stopStream();
      return;
    }

    // 録音データが届くたびに溜めて、届いた時刻も記録する
    mediaRecorder.ondataavailable = (e) => {
      if (e.data && e.data.size > 0) {
        chunks.push(e.data);
        chunkArrivals.push({
          at: nowRel(),
          bytes: e.data.size,
          visible: document.visibilityState === "visible",
        });
        totalBytes += e.data.size;
        chunkCount += 1;
      }
    };
    mediaRecorder.onerror = (e) => {
      addEvent("error", `録音中にエラー：${e.error ? e.error.name : "不明"}`);
    };
    // 停止が完了したら集計に進む
    mediaRecorder.onstop = () => {
      finishResult();
    };

    // ---- いっせいにスタート ----
    startTime = Date.now();
    status = "recording";
    addEvent("info", `録音開始（形式：${mediaRecorder.mimeType || "不明"}）`);
    // 1000ミリ秒ごとに録音データを小分けにして受け取る
    mediaRecorder.start(1000);
    tickTimer = setInterval(tick, 1000);
    meterLoop();
    addPageListeners();
  }

  /**
   * stopTest：テストを停止する（停止ボタンから呼ばれる）
   * 録音係を止めると onstop → finishResult に進む
   */
  function stopTest() {
    if (status !== "recording") return;
    status = "processing";
    stopAt = nowRel();

    // 記録係とメーターを止める
    clearInterval(tickTimer);
    tickTimer = null;
    cancelAnimationFrame(meterFrame);
    meterFrame = null;

    // 裏にいる途中で止めた場合は、その時間帯を閉じる
    if (hiddenStart !== null) {
      hiddenPeriods.push({ start: hiddenStart, end: stopAt });
      hiddenStart = null;
    }

    addEvent("info", "停止ボタンが押された");

    if (mediaRecorder && mediaRecorder.state !== "inactive") {
      mediaRecorder.stop();
    } else {
      finishResult();
    }
  }

  /**
   * stopStream：マイクの使用を終わらせる（スマホの「マイク使用中」表示も消える）
   */
  function stopStream() {
    if (stream) {
      stream.getTracks().forEach((t) => {
        // 終了時のイベントで余計な記録が出ないように見張りを外してから止める
        t.onended = null;
        t.onmute = null;
        t.onunmute = null;
        t.stop();
      });
      stream = null;
    }
  }

  /**
   * finishResult：録音を解析して、結果を集計・表示する
   */
  async function finishResult() {
    removePageListeners();
    stopStream();

    // ---- 小分けの録音データを1つにまとめる ----
    const blobType = (mediaRecorder && mediaRecorder.mimeType) || "audio/webm";
    const blob = new Blob(chunks, { type: blobType });
    audioUrl = URL.createObjectURL(blob);

    // ---- 録音を解析する ----
    // Chrome の WebM は「長さ」の情報が入っていないことが多いので、
    // いったん音のデータに戻して（デコードして）本当の長さと中身を調べる
    let recordedSec = null;
    const levels = [];
    try {
      const arrayBuffer = await blob.arrayBuffer();
      const decoded = await audioContext.decodeAudioData(arrayBuffer);
      recordedSec = decoded.duration;

      // 1秒ずつ区切って、その1秒の音の大きさ(RMS→dB)を求める
      const data = decoded.getChannelData(0);
      const rate = decoded.sampleRate;
      const totalSeconds = Math.ceil(decoded.duration);
      for (let s = 0; s < totalSeconds; s++) {
        const from = s * rate;
        const to = Math.min(data.length, (s + 1) * rate);
        let sum = 0;
        for (let i = from; i < to; i++) sum += data[i] * data[i];
        const rms = to > from ? Math.sqrt(sum / (to - from)) : 0;
        levels.push({
          sec: s,
          db: rmsToDb(rms),
          hidden: isSecondHidden(s),
        });
      }
    } catch (err) {
      addEvent("error", `録音の解析に失敗：${err.name}（${err.message}）`);
    }
    perSecondLevels = levels;

    // ---- 解析が終わったので音の部品を片付ける ----
    if (audioContext) {
      audioContext.onstatechange = null;
      await audioContext.close();
      audioContext = null;
    }
    analyser = null;
    sourceNode = null;

    summary = buildSummary(recordedSec, levels);
    status = "stopped";
  }

  /**
   * isSecondHidden：録音の s 秒目〜s+1 秒目が「裏にいた時間帯」と重なるかを調べる
   * @param {number} s 秒
   * @returns {boolean}
   */
  function isSecondHidden(s) {
    const from = s * 1000;
    const to = (s + 1) * 1000;
    return hiddenPeriods.some((p) => p.start < to && p.end > from);
  }

  /**
   * buildSummary：集計結果を作る
   * @param {number|null} recordedSec 実際に録音できた長さ（秒）。解析失敗なら null
   * @param {Array} levels 1秒ごとの音の大きさ
   * @returns {object} 集計結果
   */
  function buildSummary(recordedSec, levels) {
    const wallSec = stopAt / 1000;

    // 裏にいた時間帯ごとの詳しい様子
    const periods = hiddenPeriods.map((p) => {
      const durSec = (p.end - p.start) / 1000;
      // その時間帯に記録係が何回メモできたか（本来は約 durSec 回）
      const ticks = tickLogs.filter((t) => t.at >= p.start && t.at < p.end).length;
      // その時間帯に録音データが何個届いたか
      const arrived = chunkArrivals.filter((c) => c.at >= p.start && c.at < p.end).length;
      // その時間帯の録音の中で、一番大きかった音（声を出したなら大きくなるはず）
      const inside = levels.filter((l) => l.hidden && l.sec * 1000 < p.end && (l.sec + 1) * 1000 > p.start);
      const maxDb = inside.length > 0 ? Math.max(...inside.map((l) => l.db)) : null;
      return {
        startText: formatTime(p.start),
        endText: formatTime(p.end),
        durSec,
        ticks,
        arrived,
        maxDb,
      };
    });

    const hiddenTotalSec = periods.reduce((sum, p) => sum + p.durSec, 0);
    const muteCount = events.filter((e) => e.type === "mute").length;
    const freezeCount = events.filter((e) => e.type === "freeze").length;
    // 壁時計の経過時間に比べて、録音が何秒足りないか
    const missingSec = recordedSec === null ? null : Math.max(0, wallSec - recordedSec);

    // 判定：録音の長さが壁時計とほぼ同じ（差2秒以内）なら「途切れずに続いた」
    let continuity = "unknown";
    if (missingSec !== null) continuity = missingSec <= 2 ? "ok" : "ng";

    // 裏の時間帯の録音が「完全な無音（-100dB 付近）」なら、マイクが止められていた疑い
    const silentInHidden = periods.some((p) => p.maxDb !== null && p.maxDb <= -99);

    return {
      wallSec,
      recordedSec,
      missingSec,
      hiddenTotalSec,
      periods,
      muteCount,
      freezeCount,
      continuity,
      silentInHidden,
      mimeType: (mediaRecorder && mediaRecorder.mimeType) || "不明",
      userAgent: navigator.userAgent,
    };
  }

  // ============================================================
  // 結果のコピー
  // ============================================================

  /**
   * buildResultText：結果を文字にまとめる（チャットに貼り付けて相談する用）
   * @returns {string}
   */
  function buildResultText() {
    if (!summary) return "";
    const lines = [];
    lines.push("【バックグラウンド録音テスト結果】");
    lines.push(`端末: ${summary.userAgent}`);
    lines.push(`形式: ${summary.mimeType}`);
    lines.push(`経過時間(壁時計): ${summary.wallSec.toFixed(1)}秒`);
    lines.push(`録音できた長さ: ${summary.recordedSec === null ? "解析失敗" : summary.recordedSec.toFixed(1) + "秒"}`);
    lines.push(`不足: ${summary.missingSec === null ? "-" : summary.missingSec.toFixed(1) + "秒"}`);
    lines.push(`裏にいた合計: ${summary.hiddenTotalSec.toFixed(1)}秒（${summary.periods.length}回）`);
    lines.push(`マイク停止イベント: ${summary.muteCount}回 / 凍結イベント: ${summary.freezeCount}回`);
    summary.periods.forEach((p, i) => {
      lines.push(`  裏${i + 1}: ${p.startText}〜${p.endText}（${p.durSec.toFixed(1)}秒）メモ${p.ticks}回・データ到着${p.arrived}個・最大音量${p.maxDb === null ? "-" : p.maxDb.toFixed(1) + "dB"}`);
    });
    lines.push("出来事:");
    events.forEach((e) => lines.push(`  ${formatTime(e.at)} ${e.text}`));
    lines.push("1秒ごとの音量(dB, *は裏):");
    lines.push("  " + perSecondLevels.map((l) => `${l.sec}:${Math.round(l.db)}${l.hidden ? "*" : ""}`).join(" "));
    return lines.join("\n");
  }

  /**
   * copyResult：結果の文字をクリップボードにコピーする
   */
  async function copyResult() {
    try {
      await navigator.clipboard.writeText(buildResultText());
      copyMessage = "コピーしました";
    } catch (err) {
      copyMessage = `コピーできませんでした：${err.name}`;
    }
  }

  // ============================================================
  // ページを離れるときの後片付け
  // ============================================================
  onDestroy(() => {
    if (typeof document === "undefined") return; // サーバー側では何もしない
    clearInterval(tickTimer);
    cancelAnimationFrame(meterFrame);
    removePageListeners();
    if (mediaRecorder && mediaRecorder.state !== "inactive") {
      mediaRecorder.onstop = null;
      mediaRecorder.stop();
    }
    stopStream();
    if (audioContext) audioContext.close();
    if (audioUrl) URL.revokeObjectURL(audioUrl);
  });
</script>

<svelte:head>
  <title>録音テスト | otorec</title>
  <meta name="description" content="スマホのブラウザで、裏に回しても録音が続くかを確かめるテストページです。" />
</svelte:head>

<main>
  <h1>バックグラウンド録音テスト</h1>

  <!-- ===== 手順の説明 ===== -->
  <section class="card">
    <h2>手順</h2>
    <ol>
      <li>「開始」を押して「いち」と言う</li>
      <li>ホーム画面に戻り、別のアプリを1分使いながら「に」と言う</li>
      <li>画面を消して1分待ち、「さん」と言う</li>
      <li>戻って「停止」を押し、結果と再生を確認する</li>
    </ol>
  </section>

  <!-- ===== ボタン ===== -->
  <div class="buttons">
    {#if status === "recording"}
      <button class="stop" onclick={stopTest}>停止</button>
    {:else}
      <button class="start" onclick={startTest} disabled={status === "processing"}>
        {status === "stopped" ? "もう一度テスト" : "開始"}
      </button>
    {/if}
  </div>

  {#if errorMessage}
    <p class="error">{errorMessage}</p>
  {/if}

  <!-- ===== 録音中の様子 ===== -->
  {#if status === "recording"}
    <section class="card">
      <h2>録音中</h2>
      <p class="big">{formatTime(elapsedSec * 1000)}</p>
      <!-- 音の大きさメーター -->
      <div class="meter">
        <div class="meter-fill" style="width: {dbToPercent(currentLevel)}%"></div>
      </div>
      <p class="small">音の大きさ：{currentLevel.toFixed(1)} dB</p>
      <p class="small">ページ：{isVisible ? "表" : "裏"}</p>
      <p class="small">録音データ：{chunkCount} 個（{formatBytes(totalBytes)}）</p>
    </section>
  {/if}

  {#if status === "processing"}
    <p>集計中…</p>
  {/if}

  <!-- ===== 結果 ===== -->
  {#if status === "stopped" && summary}
    <section class="card">
      <h2>結果</h2>

      <!-- 判定 -->
      {#if summary.continuity === "ok"}
        <p class="verdict ok">録音は途切れずに続きました</p>
      {:else if summary.continuity === "ng"}
        <p class="verdict ng">録音が約 {summary.missingSec.toFixed(1)} 秒欠けています</p>
      {:else}
        <p class="verdict unknown">録音の長さを解析できませんでした</p>
      {/if}

      {#if summary.silentInHidden}
        <p class="warn">裏にいた時間帯の録音が完全な無音です。マイクが止められていた可能性が高いです。</p>
      {/if}

      <table class="summary-table">
        <tbody>
          <tr><th>経過時間（壁時計）</th><td>{summary.wallSec.toFixed(1)} 秒</td></tr>
          <tr>
            <th>録音できた長さ</th>
            <td>{summary.recordedSec === null ? "解析失敗" : summary.recordedSec.toFixed(1) + " 秒"}</td>
          </tr>
          <tr>
            <th>裏にいた合計</th>
            <td>{summary.hiddenTotalSec.toFixed(1)} 秒（{summary.periods.length} 回）</td>
          </tr>
          <tr><th>マイク停止イベント</th><td>{summary.muteCount} 回</td></tr>
          <tr><th>凍結イベント</th><td>{summary.freezeCount} 回</td></tr>
          <tr><th>形式</th><td>{summary.mimeType}</td></tr>
        </tbody>
      </table>

      <!-- 裏にいた時間帯ごとの様子 -->
      {#if summary.periods.length > 0}
        <h3>裏にいた時間帯</h3>
        <ul class="periods">
          {#each summary.periods as p, i}
            <li>
              <strong>裏{i + 1}</strong>：{p.startText}〜{p.endText}（{p.durSec.toFixed(1)} 秒）<br />
              メモ {p.ticks} 回・データ到着 {p.arrived} 個・最大音量
              {p.maxDb === null ? "-" : p.maxDb.toFixed(1) + " dB"}
            </li>
          {/each}
        </ul>
      {/if}

      <!-- 再生 -->
      <h3>再生</h3>
      <audio controls src={audioUrl}></audio>

      <!-- 1秒ごとの音の大きさ（棒グラフ。オレンジ＝裏にいた時間） -->
      <h3>1秒ごとの音の大きさ</h3>
      <p class="small">青＝表、オレンジ＝裏。「に」「さん」の位置に山があれば、裏でも声が録れています。</p>
      <div class="bars">
        {#each perSecondLevels as l}
          <div class="bar {l.hidden ? 'hidden' : ''}" style="height: {Math.max(2, dbToPercent(l.db))}%" title="{l.sec}秒：{l.db.toFixed(1)}dB"></div>
        {/each}
      </div>

      <div class="buttons">
        <button onclick={copyResult}>結果をコピー</button>
      </div>
      {#if copyMessage}
        <p class="small">{copyMessage}</p>
      {/if}
    </section>
  {/if}

  <!-- ===== 出来事の記録（新しい順） ===== -->
  {#if events.length > 0}
    <section class="card">
      <h2>出来事</h2>
      <ul class="events">
        {#each [...events].reverse() as e}
          <li class="ev-{e.type}"><span class="time">{formatTime(e.at)}</span> {e.text}</li>
        {/each}
      </ul>
    </section>
  {/if}

  <!-- ===== 記録係のメモ（停止後に表示） ===== -->
  {#if status === "stopped" && tickLogs.length > 0}
    <section class="card">
      <h2>1秒ごとのメモ（{tickLogs.length} 件）</h2>
      <div class="log-box">
        <table class="log-table">
          <thead>
            <tr><th>時刻</th><th>音量</th><th>ページ</th><th>解析部品</th></tr>
          </thead>
          <tbody>
            {#each tickLogs as t}
              <tr class={t.visible ? "" : "row-hidden"}>
                <td>{formatTime(t.at)}</td>
                <td>{t.level.toFixed(1)}</td>
                <td>{t.visible ? "表" : "裏"}</td>
                <td>{t.ctxState}</td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
    </section>
  {/if}
</main>

<style>
  /* ===== 色の設定（明るい画面用） ===== */
  :global(:root) {
    --bg: #f6f7f9;
    --card: #ffffff;
    --text: #1d2330;
    --sub: #5d6575;
    --line: #dde1e8;
    --blue: #3b6fe0;
    --orange: #e8892b;
    --red: #d64545;
    --green: #2e9e5b;
  }
  /* ===== 色の設定（暗い画面用） ===== */
  @media (prefers-color-scheme: dark) {
    :global(:root) {
      --bg: #14171c;
      --card: #1e2229;
      --text: #e8ebf0;
      --sub: #9aa2b1;
      --line: #2f3540;
      --blue: #6b95f0;
      --orange: #f0a050;
      --red: #f07070;
      --green: #5cc78a;
    }
  }

  :global(body) {
    margin: 0;
    background: var(--bg);
    color: var(--text);
    font-family:
      system-ui,
      -apple-system,
      "Hiragino Sans",
      "Noto Sans JP",
      sans-serif;
  }

  main {
    max-width: 640px;
    margin: 0 auto;
    padding: 16px;
  }

  h1 {
    font-size: 1.3rem;
    margin: 8px 0 16px;
  }
  h2 {
    font-size: 1.05rem;
    margin: 0 0 8px;
  }
  h3 {
    font-size: 0.95rem;
    margin: 16px 0 6px;
  }

  /* 白い箱 */
  .card {
    background: var(--card);
    border: 1px solid var(--line);
    border-radius: 12px;
    padding: 14px;
    margin-bottom: 14px;
  }

  ol {
    margin: 0;
    padding-left: 1.3em;
    line-height: 1.7;
  }

  /* ボタン */
  .buttons {
    display: flex;
    gap: 10px;
    margin: 12px 0;
  }
  button {
    flex: 1;
    font-size: 1.05rem;
    padding: 14px;
    border-radius: 10px;
    border: 1px solid var(--line);
    background: var(--card);
    color: var(--text);
  }
  button.start {
    background: var(--blue);
    border-color: var(--blue);
    color: #fff;
  }
  button.stop {
    background: var(--red);
    border-color: var(--red);
    color: #fff;
  }
  button:disabled {
    opacity: 0.5;
  }

  .big {
    font-size: 2.2rem;
    font-variant-numeric: tabular-nums;
    margin: 4px 0 10px;
  }
  .small {
    font-size: 0.85rem;
    color: var(--sub);
    margin: 4px 0;
  }
  .error {
    color: var(--red);
  }
  .warn {
    color: var(--orange);
    font-size: 0.9rem;
  }

  /* 音の大きさメーター */
  .meter {
    height: 14px;
    background: var(--line);
    border-radius: 7px;
    overflow: hidden;
  }
  .meter-fill {
    height: 100%;
    background: var(--blue);
    transition: width 0.08s linear;
  }

  /* 判定 */
  .verdict {
    font-weight: bold;
    font-size: 1.05rem;
    margin: 0 0 10px;
  }
  .verdict.ok {
    color: var(--green);
  }
  .verdict.ng {
    color: var(--red);
  }
  .verdict.unknown {
    color: var(--sub);
  }

  /* 集計の表 */
  .summary-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.9rem;
  }
  .summary-table th,
  .summary-table td {
    text-align: left;
    padding: 5px 4px;
    border-bottom: 1px solid var(--line);
  }
  .summary-table th {
    color: var(--sub);
    font-weight: normal;
    width: 50%;
  }

  .periods {
    padding-left: 1.1em;
    font-size: 0.88rem;
    line-height: 1.6;
  }

  audio {
    width: 100%;
  }

  /* 1秒ごとの棒グラフ（横にスクロールできる） */
  .bars {
    display: flex;
    align-items: flex-end;
    gap: 1px;
    height: 90px;
    overflow-x: auto;
    border-bottom: 1px solid var(--line);
    padding-bottom: 2px;
  }
  .bar {
    flex: 0 0 4px;
    background: var(--blue);
    border-radius: 1px 1px 0 0;
  }
  .bar.hidden {
    background: var(--orange);
  }

  /* 出来事の記録 */
  .events {
    list-style: none;
    padding: 0;
    margin: 0;
    font-size: 0.88rem;
    line-height: 1.7;
  }
  .time {
    color: var(--sub);
    font-variant-numeric: tabular-nums;
    margin-right: 6px;
  }
  .ev-mute,
  .ev-ended,
  .ev-error,
  .ev-freeze {
    color: var(--red);
  }
  .ev-hidden {
    color: var(--orange);
  }

  /* 記録係のメモ */
  .log-box {
    max-height: 300px;
    overflow-y: auto;
  }
  .log-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.82rem;
    font-variant-numeric: tabular-nums;
  }
  .log-table th,
  .log-table td {
    padding: 3px 4px;
    border-bottom: 1px solid var(--line);
    text-align: left;
  }
  .row-hidden {
    color: var(--orange);
  }
</style>
