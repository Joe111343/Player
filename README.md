<!DOCTYPE html>
<html lang="zh-Hant">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>穩定定格播放器</title>
  <style>
    html, body {
      margin: 0; padding: 0; height: 100vh;
      background: url('背景1.png') no-repeat center center fixed;
      background-size: cover;
      font-family: sans-serif;
      color: white;
      overflow: hidden;
    }
    #container {
      position: fixed;
      bottom: 125px;
      right: 20px;
      width: 300px;
      height: 150px;
      transform: rotate(-45deg);
      transform-origin: bottom right;
      clip-path: polygon(10% 0%, 90% 0%, 100% 50%, 90% 100%, 10% 100%, 0% 50%);
      background: transparent;
      z-index: 999;
    }
    #canvas {
      width: 100%; height: 100%; display: block;
      image-rendering: pixelated;
    }
    #topLeftText, #thanks {
      position: fixed;
      font-size: 14px;
      pointer-events: none;
      text-shadow: 0 0 3px black;
      z-index: 10;
      display: flex;
      align-items: center;
      gap: 8px;
      user-select: none;
    }
    #topLeftText {
      top: 10px; left: 10px;
      pointer-events: auto;
    }
    #thanks {
      bottom: 10px; left: 10px;
    }
    #fileInput {
      position: fixed;
      top: 10px; right: 10px;
      z-index: 100;
    }
    #spectrum {
      position: fixed;
      top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      width: 1000px; height: 120px;
      pointer-events: none;
      z-index: 20;
      background: transparent;
      border-radius: 8px;
    }
    #playBtn {
      position: fixed;
      top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      z-index: 200;
      font-size: 18px;
      padding: 10px 20px;
      border: none;
      background: lime;
      color: black;
      cursor: pointer;
      border-radius: 8px;
    }
    #limiterToggle label {
      cursor: pointer;
      display: flex;
      align-items: center;
      gap: 6px;
      user-select: none;
      font-size: 14px;
    }
    #limiterCheckbox {
      width: 16px;
      height: 16px;
      cursor: pointer;
    }

    /* 加入進度條 */
    #gifProgress {
      position: fixed;
      bottom: 0;
      left: 0;
      height: 24px;
      width: 100vw;
      background: url('GIF.gif') repeat-x;
      background-size: contain;
      background-repeat: repeat-x;
      background-position: left center;
      z-index: 9999;
      pointer-events: none;
      mask-image: linear-gradient(to right, black 0%, black 0%, transparent 100%);
      -webkit-mask-image: linear-gradient(to right, black 0%, black 0%, transparent 100%);
    }
  </style>
</head>
<body>
  <div id="topLeftText">
    趔陽不會陽bilibili
    <div id="limiterToggle">
      <label for="limiterCheckbox">
        <input type="checkbox" id="limiterCheckbox" checked />
        限幅器ON
      </label>
    </div>
  </div>
  <div id="thanks">感謝 <span style="color:deepskyblue">@趔陽不會陽</span> 的BY player</div>
  <div id="container">
    <canvas id="canvas" width="600" height="300"></canvas>
  </div>
  <canvas id="spectrum"></canvas>
  <input type="file" id="fileInput" accept=".mp3,.wav,.m4a" />
  <button id="playBtn">▶ 播放音訊</button>
  <div id="gifProgress"></div>

  <script>
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    const width = canvas.width;
    const height = canvas.height;
    const image = new Image();
    image.src = 'R.png';
    let imageLoaded = false;

    const gifProgress = document.getElementById('gifProgress');

    const spectrumCanvas = document.getElementById('spectrum');
    const spectrumCtx = spectrumCanvas.getContext('2d');
    spectrumCanvas.width = 1000;
    spectrumCanvas.height = 120;

    let audioCtx, analyser, source, dataArray, freqArray, audioFile, audio;
    let sampleRate = 441000;
    let nyquist = sampleRate / 2;
    let readyToPlay = false;

    let limiterEnabled = true;
    let bassEQ, volumeGain, limiter;

    const frameBuffer = [];
    const maxFrames = 4;
    let lastScanTime = 0;
    const scanInterval = 27.5;
    let smoothedMax = 0;
    let smoothedOffset = 0;

    image.onload = () => {
      imageLoaded = true;
      draw();
    };

    function drawImageScaledCenter(ctx, img, canvasW, canvasH) {
      const scale = Math.min(canvasW / img.width, canvasH / img.height);
      const imgW = img.width * scale;
      const imgH = img.height * scale;
      const offsetX = (canvasW - imgW) / 2;
      const offsetY = (canvasH - imgH) / 2;
      ctx.drawImage(img, offsetX, offsetY, imgW, imgH);
    }

    function draw() {
      requestAnimationFrame(draw);
      if (!imageLoaded) return;

      if (!analyser) {
        ctx.clearRect(0, 0, width, height);
        drawImageScaledCenter(ctx, image, width, height);
        return;
      }

      const now = performance.now();
      if (now - lastScanTime >= scanInterval) {
        analyser.getByteTimeDomainData(dataArray);
        analyser.getByteFrequencyData(freqArray);

        const offCanvas = document.createElement('canvas');
        offCanvas.width = width;
        offCanvas.height = height;
        const offCtx = offCanvas.getContext('2d');
        drawImageScaledCenter(offCtx, image, width, height);
        const srcImageData = offCtx.getImageData(0, 0, width, height);
        const newFrame = offCtx.createImageData(width, height);
        const srcPixels = srcImageData.data;
        const dstPixels = newFrame.data;
        const halfHeight = Math.floor(height / 2);

        const lowpassCutoffHz = 100;
        const binSize = nyquist / freqArray.length;
        const lowpassBin = Math.floor(lowpassCutoffHz / binSize);

        let weightedSum = 0;
        let totalWeight = 0;
        for (let i = 0; i <= lowpassBin; i++) {
          const val = freqArray[i];
          const weight = 1 - (i / lowpassBin);
          weightedSum += val * weight;
          totalWeight += weight;
        }
        const lowFreqPower = weightedSum / totalWeight;
        smoothedMax = smoothedMax * 0.9 + lowFreqPower * 0.1;

        for (let y = 0; y < halfHeight; y++) {
          for (let x = 0; x < width; x++) {
            const i = (y * width + x) * 4;
            const index = Math.floor(x * dataArray.length / width);
            const rawOffset = ((dataArray[index] / 255) - 0.5);
            smoothedOffset = smoothedOffset * 0.9 + rawOffset * 0.1;
            // 震動幅度根據音量倍率動態計算
            let offset = smoothedOffset * (smoothedMax / 255) * volumeGain.gain.value * 100;
            // 不限制newY，讓震動可以超出上半部高度
            let newY = y + Math.floor(offset);
            newY = Math.max(0, Math.min(height - 1, newY));
            const j = (newY * width + x) * 4;
            dstPixels[j] = srcPixels[i];
            dstPixels[j + 1] = srcPixels[i + 1];
            dstPixels[j + 2] = srcPixels[i + 2];
            dstPixels[j + 3] = srcPixels[i + 3];
          }
        }

        for (let y = halfHeight; y < height; y++) {
          for (let x = 0; x < width; x++) {
            const i = (y * width + x) * 4;
            dstPixels[i] = srcPixels[i];
            dstPixels[i + 1] = srcPixels[i + 1];
            dstPixels[i + 2] = srcPixels[i + 2];
            dstPixels[i + 3] = srcPixels[i + 3];
          }
        }

        frameBuffer.push(newFrame);
        if (frameBuffer.length > maxFrames) frameBuffer.shift();
        lastScanTime = now;
      }

      ctx.clearRect(0, 0, width, height);
      for (let i = 0; i < frameBuffer.length; i++) {
        ctx.globalAlpha = (i + 1) / frameBuffer.length / 1.5;
        ctx.putImageData(frameBuffer[i], 0, 0);
      }
      ctx.globalAlpha = 1;

      spectrumCtx.clearRect(0, 0, spectrumCanvas.width, spectrumCanvas.height);
      const binSize = nyquist / freqArray.length;
      const startHz = 1;
      const endHz = 300;
      const startBin = Math.floor(startHz / binSize);
      const endBin = Math.min(freqArray.length - 1, Math.floor(endHz / binSize));
      const visibleBins = endBin - startBin;
      const barWidth = spectrumCanvas.width / visibleBins;

      for (let i = startBin; i <= endBin; i++) {
        const val = freqArray[i];
        const barHeight = (val / 255) * spectrumCanvas.height;
        spectrumCtx.fillStyle = 'lime';
        spectrumCtx.fillRect((i - startBin) * barWidth, spectrumCanvas.height - barHeight, barWidth - 1, barHeight);
      }

      // 移動進度條
      if (audio && audio.duration > 0) {
        const progress = audio.currentTime / audio.duration;
        gifProgress.style.maskImage = `linear-gradient(to right, black ${progress * 100}%, transparent ${progress * 100.1}%)`;
        gifProgress.style.webkitMaskImage = gifProgress.style.maskImage;
      }
    }

    document.getElementById('fileInput').addEventListener('change', e => {
      const file = e.target.files[0];
      if (!file) return;
      audioFile = file;
      readyToPlay = true;
      document.getElementById('playBtn').style.display = 'block';
    });

    document.getElementById('playBtn').addEventListener('click', () => {
      if (!readyToPlay || !audioFile) return;
      if (audioCtx) audioCtx.close();

      audio = new Audio();
      audio.src = URL.createObjectURL(audioFile);
      audio.crossOrigin = 'anonymous';

      audioCtx = new (window.AudioContext || window.webkitAudioContext)();
      sampleRate = audioCtx.sampleRate;
      nyquist = sampleRate / 2;
      source = audioCtx.createMediaElementSource(audio);

      bassEQ = audioCtx.createBiquadFilter();
      bassEQ.type = "lowshelf";
      bassEQ.frequency.value = 100;
      bassEQ.gain.value = 20;

      volumeGain = audioCtx.createGain();
      volumeGain.gain.value = 0.316;

      limiter = audioCtx.createDynamicsCompressor();
      limiter.threshold.value = 0;
      limiter.knee.value = 0;
      limiter.ratio.value = 20;
      limiter.attack.value = 0.001;
      limiter.release.value = 0.05;

      analyser = audioCtx.createAnalyser();
      analyser.fftSize = 8192;
      dataArray = new Uint8Array(analyser.fftSize);
      freqArray = new Uint8Array(analyser.frequencyBinCount);

      connectAudioNodes();
      audio.play();
      document.getElementById('playBtn').style.display = 'none';
    });

    function connectAudioNodes() {
      try { source.disconnect(); } catch(e){}
      try { bassEQ.disconnect(); } catch(e){}
      try { volumeGain.disconnect(); } catch(e){}
      try { limiter.disconnect(); } catch(e){}
      try { analyser.disconnect(); } catch(e){}

      if (limiterEnabled) {
        source.connect(bassEQ);
        bassEQ.connect(volumeGain);
        volumeGain.connect(limiter);
        limiter.connect(analyser);
        analyser.connect(audioCtx.destination);
      } else {
        source.connect(bassEQ);
        bassEQ.connect(volumeGain);
        volumeGain.connect(analyser);
        analyser.connect(audioCtx.destination);
      }
    }

    document.getElementById('limiterCheckbox').addEventListener('change', () => {
      limiterEnabled = document.getElementById('limiterCheckbox').checked;
      document.getElementById('limiterCheckbox').nextSibling.textContent = limiterEnabled ? ' 限幅器ON' : ' 限幅器OFF';
      if (!audioCtx || !source) return;
      connectAudioNodes();
    });
  </script>
</body>
</html>
