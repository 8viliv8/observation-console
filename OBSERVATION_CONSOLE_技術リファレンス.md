# OBSERVATION CONSOLE — 技術リファレンス

ブラウザ単体(HTML1ファイル、外部サーバー不要)で動くWeb Audio API楽器の設計メモ。
このドキュメントは「D AN D AN D !!!!」プロジェクト用に作った OBSERVATION CONSOLE の
構造を、**他の楽器・ツールに流用できる形**で抽出したもの。コード片はそのままコピーして
別プロジェクトの土台にできる。

---

## 1. 全体構成

```
[静的アセット]                [Web Audio グラフ]              [UI]
音源(mp3)を base64 化して   →  AudioContext 生成      →  <input type=range> = ノブ/フェーダー
JSファイルに直接埋め込み        ↓                         <div class="step"> = ステップシーケンサー
(サーバー不要、1ファイル完結)  複数の GainNode/バス       CSS Gridで16トラック×32ステップ
                              ↓
                            エフェクトチェーン
                              ↓
                            destination + 録音用タップ
```

**設計方針**:
- 依存ライブラリなし。Web Audio API + Canvas + 素のDOM操作のみ
- 音源は全部 base64 埋め込み → HTMLファイル1個で完結、配布・保存が簡単
- 見た目(スキン)と信号処理(エンジン)を分離 → 配色やCSSを変えるだけで別の楽器に見せられる

---

## 2. ドリフトしないスケジューラー(最重要)

### 問題
BPMやレートを**再生中に**変更すると、音がズレたり、映像がフリーズしたり、
「追いつき処理」が一気に走って暴走する。

### 原因
「次の発音時刻」を `前回時刻 + 間隔` の**足し算の積み重ね**で計算していると、
間隔(BPMやレート)を変えた瞬間、蓄積してきた時間軸と新しい間隔が食い違う。

### 解決:複数系統を独立した「起点(origin)」で管理し、値が変わるたびに再基準化する

```javascript
// 系統ごとに別々の起点変数を持つ(音のステップ、素材のリトリガー、映像フリッカーなど)
let stepOrigin = 0, retrigOrigin = 0, visualOrigin = 0;
let stepCounter = 0, retrigCounter = 0, visualCounter = 0;

// スケジューラー本体:起点 + カウンタ×間隔 で「次の発音時刻」を毎回計算し直す
function scheduler() {
  const secPerStep = (60.0/bpm)/8; // 例:32分音符
  while (stepOrigin + stepCounter*secPerStep < audioCtx.currentTime + 0.1) {
    const when = stepOrigin + stepCounter*secPerStep;
    // ここで発音処理
    stepCounter++;
  }
  setTimeout(scheduler, 25); // 25ms間隔で先読みスケジューリング
}

// BPMやレートを変更した時は、"今の位置"を保ったまま起点だけ動かす
function rebaseStepTiming() {
  const secPerStep = (60.0/bpm)/8;
  stepOrigin = audioCtx.currentTime - stepCounter*secPerStep;
  // → 次のループ位置(stepCounter%STEP_COUNT)は変えず、そこから先のテンポだけ変わる
}
// BPMスライダーのイベントリスナーで呼ぶ:
bpmSlider.addEventListener('input', () => { bpm = ...; rebaseStepTiming(); });
```

**ポイント**: 系統が複数ある場合(音のステップ・素材のリトリガー・映像フリッカーなど)、
**それぞれ別の origin/counter ペアを持たせる**。1つの起点を共有すると、片方を再基準化した
瞬間にもう片方がズレる。

---

## 3. グレイン(粒)化 — 素材からリズムを抽出する

一つの音素材(アンビエントテクスチャ等)を、内部のクリック・アタックの位置で
自動的に細切れにして、任意のステップに再配置できる「粒」の集合に変える手法。

### Python側(前処理、素材ごとに1回だけ実行)

```python
import numpy as np
from scipy.io import wavfile
from scipy.signal import spectrogram, find_peaks

def extract_grains(path, out_prefix, max_grain_dur=0.18):
    sr, x = wavfile.read(path)
    x = x.astype('float32') / 32768.0  # 16bit想定。24bit/32bitは別途対応要
    if x.ndim > 1: x = x.mean(axis=1)

    # スペクトログラムの変化量(spectral flux)でオンセット検出
    f, t, Sxx = spectrogram(x, sr, nperseg=1024, noverlap=768)
    flux = np.sum(np.maximum(np.diff(10*np.log10(Sxx+1e-10), axis=1), 0), axis=0)
    flux = np.concatenate([[0], flux])
    peaks, _ = find_peaks(flux, height=np.percentile(flux, 80),
                           distance=int(0.05*sr/(1024-768)))
    onset_samples = (t[peaks] * sr).astype(int)

    # 各オンセットから次のオンセットまで(最大 max_grain_dur 秒)を1粒として切り出し
    for i, start in enumerate(onset_samples):
        end = min(start + int(max_grain_dur*sr),
                   onset_samples[i+1] if i+1 < len(onset_samples) else len(x))
        grain = x[start:end]
        fade = min(int(0.005*sr), len(grain)//4)
        if fade > 0: grain[-fade:] *= np.linspace(1, 0, fade)  # クリック防止フェード
        wavfile.write(f'{out_prefix}{i:03d}.wav', sr, (grain*32767).astype('int16'))
```

出力された粒(数十〜百数十個)を mp3 に一括変換 → base64化 → JSに埋め込み。

### JS側(再生)

```javascript
function playGrain(track, time) {
  const buf = grainBuffers[grainKeys[track.grainIdx]]; // 事前に decodeAudioData 済み
  const src = audioCtx.createBufferSource();
  src.buffer = buf;
  src.connect(track.gainNode);
  track.panNode.pan.setValueAtTime(Math.random()*1.2-0.6, time); // 毎回ランダムパン
  src.start(time);
}
```

**応用**: 声素材(単語)も同じ手法でチョップすれば、音節単位の断片が作れる。

---

## 4. エフェクトチェーンの組み方

### 4-1. Crush(誤正規化オーバードライブ)
「24bit音声を16bit用の係数で正規化してしまい、想定より大きい値が出て
強制クリップする」という**実際に起きたバグ**を、意図的に再現できる形にしたもの。
`WaveShaperNode` でゲイン→クリップの関数を作るだけで実装できる。

```javascript
function makeCrushCurve(amount) { // amount: 0(無加工)〜1(激しく歪む)
  const n = 1024;
  const curve = new Float32Array(n);
  const gain = Math.pow(2, amount * 16); // 0〜16bit分のゲイン誤差を再現
  for (let i = 0; i < n; i++) {
    const x = (i/(n-1))*2 - 1;
    curve[i] = Math.max(-1, Math.min(1, x * gain));
  }
  return curve;
}
const shaper = audioCtx.createWaveShaper();
shaper.curve = makeCrushCurve(0.5);
```

### 4-2. リングモジュレーター(1ノブで「ゲート→音色変化→エイリアシング」を連続的に)

```javascript
const ringOsc = audioCtx.createOscillator();
const ringDepthGain = audioCtx.createGain(); // 変調の深さ
const ringGainNode = audioCtx.createGain();  // 実際に信号を通すノード

ringOsc.connect(ringDepthGain);
ringDepthGain.connect(ringGainNode.gain); // オシレーターの出力をgainパラメータに直結

// ノブの値 v (0〜1) を、周波数(指数カーブ)と深さ(線形)の両方に同時適用
function setRingMod(v) {
  const freq = 0.5 * Math.pow(4000/0.5, v); // 0.5Hz(ゲート)〜4000Hz(音色/エイリアシング)
  ringOsc.frequency.setValueAtTime(freq, audioCtx.currentTime);
  ringDepthGain.gain.setValueAtTime(v, audioCtx.currentTime);
  ringGainNode.gain.setValueAtTime(1-v, audioCtx.currentTime); // v=0で完全ドライ
}
```

低い周波数域では「音が周期的に途切れるゲート」、高い周波数域では「金属的な音色変化」、
さらに上げると「不規則なエイリアシングのブチブチ感」に自然に繋がる。

### 4-3. OTT風マルチステージ・コンプレッション

```javascript
const comp1 = audioCtx.createDynamicsCompressor(); // 緩め、アップワード寄りの効き
const comp2 = audioCtx.createDynamicsCompressor(); // 仕上げ用リミッター的役割
const makeup = audioCtx.createGain();

function setOTT(v) {
  comp1.threshold.setValueAtTime(-v*35, audioCtx.currentTime);
  comp1.ratio.setValueAtTime(1+v*15, audioCtx.currentTime);
  comp2.threshold.setValueAtTime(-6-v*10, audioCtx.currentTime);
  makeup.gain.setValueAtTime(1+v*2.5, audioCtx.currentTime);
}
// 接続順: crush → comp1 → comp2 → makeup → (dry / delay / reverb センドへ分岐)
```

### 4-4. ディレイ・リバーブ(センド方式)

```javascript
// ディレイ:フィードバックループを持つ DelayNode
const delayNode = audioCtx.createDelay(2.0);
const feedback = audioCtx.createGain();
delayNode.connect(feedback); feedback.connect(delayNode);
delayNode.connect(delayWet); // delayWet → masterへ

// リバーブ:ノイズを指数減衰させたバッファをインパルス応答として使う(サンプル不要)
function makeReverbIR(ctx, dur=2.5, decay=3.0) {
  const ir = ctx.createBuffer(2, ctx.sampleRate*dur, ctx.sampleRate);
  for (let ch=0; ch<2; ch++) {
    const d = ir.getChannelData(ch);
    for (let i=0; i<d.length; i++) d[i] = (Math.random()*2-1) * Math.pow(1-i/d.length, decay);
  }
  return ir;
}
const convolver = audioCtx.createConvolver();
convolver.buffer = makeReverbIR(audioCtx);
```

---

## 5. Solo/Mute の標準パターン

**「複数トラックを同時にソロできる」**(1つだけしかソロできない、ではない)実装。
グレイントラック・素材スロットなど、複数箇所で共通して使える。

```javascript
function shouldPlay(track, allTracks) {
  const anySolo = allTracks.some(t => t.solo);
  if (anySolo) return track.solo;      // 誰かがソロ中なら、ソロされてる奴だけ鳴る
  return !track.muted;                  // 誰もソロしてなければ、通常のミュート判定
}
```

---

## 6. UI部品

### 6-1. 本当に回転するノブ(角度ベース)

ドラッグの上下/左右移動量ではなく、**ノブの中心からポインタへの角度**を直接
値に変換する。270°スイープ(下部90°がデッドゾーン)が一般的な物理ノブの挙動に近い。

```javascript
function angleToValue(el, clientX, clientY) {
  const rect = el.getBoundingClientRect();
  const cx = rect.left + rect.width/2, cy = rect.top + rect.height/2;
  const atanDeg = Math.atan2(clientY-cy, clientX-cx) * 180/Math.PI;
  let deg = atanDeg + 90; // 「上向き=0度」の基準に変換
  if (deg > 180) deg -= 360;
  if (deg < -180) deg += 360;
  if (deg > 135) return 1;    // 上限デッドゾーン
  if (deg < -135) return 0;   // 下限デッドゾーン
  return (deg+135)/270;
}
// pointerdown/pointermoveの両方でこの関数を呼び、値を即座に更新する
// (pointerdownでも呼ぶことで「クリックした位置に即座に飛ぶ」動作になる)
```

**ハマりやすい罠**: ノブの針(インジケーター)を表すdiv要素は、デフォルトで
「下向き」に伸びる(`top:50%; height:14px` だけだと下方向に伸びる)。
上向きにするには `margin-top: -14px; transform-origin: 50% 100%;` が必要。
ここを間違えると、見た目の基準と実際の角度計算が180°ズレて「クリックした位置と
違うところに反応する」バグになる(実際にハマった)。

### 6-2. 折りたたみセクション+ヘッダーだけ常時表示のコントロール

```html
<div class="collapse-head" id="head">
  <span class="chev">▶</span>
  <span class="label">SECTION NAME</span>
  <!-- ヘッダー上のボタン/スライダーは折りたたみと連動させない -->
  <button id="muteBtn" onclick="event.stopPropagation()">M</button>
  <input type="range" onclick="event.stopPropagation()">
</div>
<div class="collapse-body" id="body"><!-- 中身 --></div>
```
```javascript
head.addEventListener('click', () => {
  const open = body.classList.toggle('open');
  head.classList.toggle('open', open);
});
```
`event.stopPropagation()` を子要素に入れておかないと、ボタンを押したつもりが
折りたたみもトグルされてしまう。

---

## 7. 背景写真とUIの座標合わせ

「実写の機材パネルに、動くUIを窓のように埋め込む」手法。

1. 背景にしたい写真の**窓・パネル部分の相対座標**を Python で確認する
```python
from PIL import Image
im = Image.open('photo.png')
w, h = im.size
box = (int(w*0.245), int(h*0.145), int(w*0.785), int(h*0.585)) # 左,上,右,下(比率)
im.crop(box).save('test_crop.png') # 目視で位置が合っているか確認してから本番へ
```
2. HTML側で、写真を`position:relative`のラッパーに`width:100%`で敷き、
   UI要素を`position:absolute`+パーセンテージ指定で重ねる
```html
<div class="photo-wrap">
  <img class="bg-photo" src="...">
  <div class="window-frame" style="left:24.5%; top:14.5%; width:54.0%; height:44.0%;">
    <!-- ここに動くUI -->
  </div>
</div>
```
3. **写真を差し替えた時は必ず座標を測り直す**(トリミングや構図が変わると数値がズレる)

### スタンプ/装飾だけを別画像として合成したい場合
テンプレートマッチングで元画像内の位置を自動検出できる:
```python
import numpy as np
orig = np.array(Image.open('full_page.png').convert('L'))
crop = np.array(Image.open('stamp_crop.png').convert('L'))
# oh,ow / ch,cw で全探索し、差分最小の位置を採用(4pxステップ間引きで十分速い)
```

---

## 8. 自分の音源ファイルを読み込ませる(ローカルファイル、サーバー不要)

```javascript
const fileInput = document.createElement('input');
fileInput.type = 'file'; fileInput.accept = 'audio/*'; fileInput.style.display = 'none';
fileInput.addEventListener('change', async (e) => {
  const file = e.target.files[0];
  const arrBuf = await file.arrayBuffer();
  const buf = await audioCtx.decodeAudioData(arrBuf); // これでそのままAudioBufferとして使える
  track.buffer = buf;
});
loadBtn.addEventListener('click', () => fileInput.click());
```

---

## 9. 録音してWAVで書き出す

`MediaRecorder`はブラウザ標準だと`webm/opus`圧縮でしか録れないため、
録音停止後に**一度デコードしてから手動でWAVヘッダーを組み立てて再書き出し**する。
MP3は標準エンコーダが無くライブラリが要るため、実装コストで見るとWAVが手軽。

```javascript
const dest = audioCtx.createMediaStreamDestination();
masterGain.connect(dest);
const recorder = new MediaRecorder(dest.stream);
const chunks = [];
recorder.ondataavailable = e => chunks.push(e.data);
recorder.onstop = async () => {
  const webmBlob = new Blob(chunks, {type:'audio/webm'});
  const audioBuf = await new AudioContext().decodeAudioData(await webmBlob.arrayBuffer());
  const wavBlob = audioBufferToWav(audioBuf); // 下記
  const a = document.createElement('a');
  a.href = URL.createObjectURL(wavBlob);
  a.download = 'output.wav';
  a.click();
};

function audioBufferToWav(buffer) {
  const nCh = buffer.numberOfChannels, sr = buffer.sampleRate, len = buffer.length;
  const blockAlign = nCh*2, dataSize = len*blockAlign;
  const buf = new ArrayBuffer(44+dataSize), view = new DataView(buf);
  const ws = (o,s) => { for(let i=0;i<s.length;i++) view.setUint8(o+i, s.charCodeAt(i)); };
  ws(0,'RIFF'); view.setUint32(4,36+dataSize,true); ws(8,'WAVE'); ws(12,'fmt ');
  view.setUint32(16,16,true); view.setUint16(20,1,true); view.setUint16(22,nCh,true);
  view.setUint32(24,sr,true); view.setUint32(28,sr*blockAlign,true);
  view.setUint16(32,blockAlign,true); view.setUint16(34,16,true);
  ws(36,'data'); view.setUint32(40,dataSize,true);
  const chData = []; for(let c=0;c<nCh;c++) chData.push(buffer.getChannelData(c));
  let off=44;
  for(let i=0;i<len;i++) for(let c=0;c<nCh;c++){
    let s = Math.max(-1,Math.min(1,chData[c][i]));
    view.setInt16(off, s<0?s*0x8000:s*0x7FFF, true); off+=2;
  }
  return new Blob([buf], {type:'audio/wav'});
}
```

---

## 10. パフォーマンス的な演出

### 10-1. ホールド式の瞬間効果(「異常」ボタンのようなもの)
```javascript
let interval = null;
btn.addEventListener('pointerdown', () => {
  let on = true;
  interval = setInterval(() => {
    on = !on;
    gateNode.gain.setValueAtTime(on?1:0, audioCtx.currentTime);
  }, gateIntervalMs); // BPM同期させるなら (60000/bpm/N)/3 のように拍の分数から算出
});
btn.addEventListener('pointerup', () => { clearInterval(interval); gateNode.gain.value = 1; });
```
拍に同期した3連符などの間隔にすると、ランダムなノイズではなく「音楽的な」効果になる。

### 10-2. 特定ステップで音と映像を同時に「切る」
```javascript
if (silenceLane[stepIdx]) {
  masterGain.gain.setValueAtTime(vol, when);
  masterGain.gain.setValueAtTime(0, when+0.002);         // 即座に無音
  masterGain.gain.setValueAtTime(0, when+stepDur-0.004);
  masterGain.gain.setValueAtTime(vol, when+stepDur-0.002); // 元に戻す
  setTimeout(() => img.style.opacity = '0', msUntil(when));
  setTimeout(() => img.style.opacity = '0.9', msUntil(when+stepDur));
}
```

---

## 11. パターンの保存/呼び出し(セッション内、簡易版)

localStorageは使わず(このHTML単体運用では環境によって使えないため)、
**メモリ上のオブジェクトに state をまるごと複製して退避**する方式で十分実用的。

```javascript
const presetSlots = {1:null, 2:null, 3:null, 4:null};
function captureState() {
  return { tracks: tracks.map(t => ({...t, steps: t.steps.slice()})), /* 他の状態も同様に */ };
}
function applyState(st) { /* 各状態をコピーし戻して再描画 */ }
```

---

## 12. アセット埋め込みの実務上のコツ

- 音源は **mp3・22050Hz・モノラル・低ビットレート(48〜56kbps)** で十分実用的な音質になり、
  数百個のグレインでも合計サイズを数百KB〜1MB台に抑えられる
- 画像は **JPEG・幅900px前後・quality 70〜80** が「1つのHTMLファイルとして持ち歩ける」
  サイズ感の目安(全体で1.5MB程度なら快適)
- JSへの埋め込みは `const NAME_B64 = { "key": "base64string", ... };` の形で機械的に生成:
```python
import base64, glob
files = sorted(glob.glob('grains/*.mp3'))
entries = []
for f in files:
    key = f.split('/')[-1].replace('.mp3','')
    with open(f,'rb') as fh:
        b64 = base64.b64encode(fh.read()).decode('ascii')
    entries.append(f'  "{key}": "{b64}"')
js = "const GRAIN_B64 = {\n" + ",\n".join(entries) + "\n};\n"
```
- 巨大なbase64文字列をコードエディタで直接編集しようとすると事故りやすいので、
  **JSファイル生成はPython側で行い、HTMLへの差し込みは文字列置換(プレースホルダー)で行う**
  (`__ASSETS__` のようなプレースホルダーを仕込んでおき、最後に実データで置換する)

---

## 13. デバッグ・検証の型

- コードを書き換えたら**必ず `node --check` でJS構文だけ先に検証**してからHTML化する
  (実行確認前に文法エラーを弾ける、コストが低い)
- 見た目の確認は Playwright でヘッドレスブラウザに実際に開かせてスクリーンショットを撮る
  ```python
  from playwright.sync_api import sync_playwright
  with sync_playwright() as p:
      browser = p.chromium.launch()
      page = browser.new_page(viewport={'width':1100,'height':900})
      page.goto('file:///path/to/file.html')
      page.click('#someCollapseHead') # 折りたたみなど、初期状態で隠れてる部分も開いて確認
      page.screenshot(path='check.png')
  ```
- `getElementById` の参照漏れ(IDの綴りミスなど)は、正規表現で全部抽出して突き合わせると
  一括検出できる:
  ```python
  import re
  refs = set(re.findall(r"getElementById\('([^']+)'\)", html))
  defs = set(re.findall(r'id="([^"]+)"', html))
  print("参照されてるが定義されてないID:", refs - defs)
  ```

---

## 14. このアーキテクチャを別の楽器に転用する時のチェックリスト

- [ ] 音源をグレイン化したいか?→ 2章のPython前処理を素材に合わせて実行
- [ ] BPM/レートを演奏中に変えるか?→ 2章のマルチ起点スケジューラーを必ず使う
- [ ] エフェクトはどれが要る?→ 4章から必要なものだけ接続順を組み替えて採用
- [ ] 複数トラックのソロ/ミュートが要るか?→ 5章のパターンをそのまま流用
- [ ] 実写背景に埋め込むか?→ 7章の座標合わせフローを踏む(写真差し替え時は必ず再計測)
- [ ] 書き出し機能が要るか?→ 9章のWAVエクスポートをそのまま移植可能
- [ ] 仕上げに Playwright で実際にレンダリングして確認する(13章)
