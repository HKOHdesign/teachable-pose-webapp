<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>AI 마스크 착용 판별기</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Roboto:wght@400;700&display=swap');

    body {
      font-family: 'Roboto', sans-serif;
      background: linear-gradient(135deg, #eef2f3, #dfe6e9);
      color: #333;
      text-align: center;
      padding: 30px;
      margin: 0;
    }

    .container {
      max-width: 600px;
      margin: 0 auto;
      padding: 20px;
    }

    .title {
      font-size: 2.5rem;
      font-weight: 700;
      margin-bottom: 10px;
      color: #2d3436;
    }

    .description {
      font-size: 1rem;
      margin-bottom: 30px;
      color: #636e72;
      line-height: 1.6;
    }

    .start-button {
      background: #6c5ce7;
      color: #fff;
      border: none;
      padding: 14px 28px;
      font-size: 1.05rem;
      border-radius: 50px;
      cursor: pointer;
      transition: background 0.3s ease;
      box-shadow: 0 4px 12px rgba(0,0,0,0.1);
    }

    .start-button:hover {
      background: #4a34c0;
    }

    .canvas-container {
      margin: 30px auto 20px;
      width: 260px;
      height: 260px;
      background: #fff;
      border-radius: 20px;
      padding: 15px;
      box-shadow: 0 8px 24px rgba(0,0,0,0.08);
    }

    #canvas {
      border-radius: 12px;
      width: 230px;
      height: 230px;
    }

    #label-container {
      margin-top: 25px;
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 10px;
    }

    #label-container div {
      background: #fff;
      padding: 10px 18px;
      border-radius: 50px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.05);
      font-size: 0.95rem;
      color: #2d3436;
      width: fit-content;
    }

    .good {
      color: #00b894;
      font-weight: 700;
    }

    .warn {
      color: #fd7e14;
      font-weight: 700;
    }

    #message img {
      width: 32px;
      height: 32px;
      vertical-align: middle;
      margin-left: 8px;
    }

    footer {
      margin-top: 50px;
      font-size: 0.85rem;
      color: #b2bec3;
    }
  </style>
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@1.3.1/dist/tf.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@teachablemachine/pose@0.8/dist/teachablemachine-pose.min.js"></script>
</head>
<body>
  <div class="container">
    <div class="title">😷 마스크 착용 판별기</div>
    <p class="description">
      이 인공지능은 <strong>마스크 착용과 마스크 미착용</strong>을 식별합니다.<br>
      📌 <strong>Start</strong> 버튼을 눌러 웹캠을 실행하고, 카메라 앞에서 마스크를 착용하거나 벗은 상태로 서보세요!
    </p>

    <button type="button" class="start-button" onclick="init()">Start</button>

    <div class="canvas-container">
      <canvas id="canvas"></canvas>
    </div>

    <div id="label-container"></div>
    <div id="message" style="margin-top:20px; font-size:1.3rem;"></div>

    <footer>
      Made with 💜 using Teachable Machine & Tensorflow.js
    </footer>
  </div>

  <!-- 소리 -->
  <audio id="sound-good" src="https://cdn.pixabay.com/download/audio/2022/03/15/audio_185ed1c69e.mp3?filename=success-1-6297.mp3"></audio>
  <audio id="sound-warn" src="https://cdn.pixabay.com/download/audio/2022/03/15/audio_8d5fba935d.mp3?filename=negative_beeps-6000.mp3"></audio>

  <script>
    const URL = "https://teachablemachine.withgoogle.com/models/Kim9WXS0Q/";
    let model, webcam, ctx, labelContainer, maxPredictions;
    let lastMessage = "";

    async function init() {
      const modelURL = URL + "model.json";
      const metadataURL = URL + "metadata.json";

      model = await tmPose.load(modelURL, metadataURL);
      maxPredictions = model.getTotalClasses();

      const size = 230;
      const flip = true;
      webcam = new tmPose.Webcam(size, size, flip);
      await webcam.setup();
      await webcam.play();
      window.requestAnimationFrame(loop);

      const canvas = document.getElementById("canvas");
      canvas.width = size;
      canvas.height = size;
      ctx = canvas.getContext("2d");
      labelContainer = document.getElementById("label-container");
      labelContainer.innerHTML = "";
      for (let i = 0; i < maxPredictions; i++) {
        labelContainer.appendChild(document.createElement("div"));
      }
    }

    async function loop() {
      webcam.update();
      await predict();
      window.requestAnimationFrame(loop);
    }

    async function predict() {
      const { pose, posenetOutput } = await model.estimatePose(webcam.canvas);
      const prediction = await model.predict(posenetOutput);

      let maskProb = 0, noMaskProb = 0;

      for (let i = 0; i < maxPredictions; i++) {
        const classPrediction = prediction[i].className + ": " + prediction[i].probability.toFixed(2);
        labelContainer.childNodes[i].innerHTML = classPrediction;

        if (prediction[i].className.includes("마스크 착용")) {
          maskProb = prediction[i].probability;
        }
        if (prediction[i].className.includes("마스크 미착용")) {
          noMaskProb = prediction[i].probability;
        }
      }

      const messageDiv = document.getElementById("message");
      const goodSound = document.getElementById("sound-good");
      const warnSound = document.getElementById("sound-warn");

      if (maskProb >= 0.85 && lastMessage !== "good") {
        messageDiv.innerHTML = `<span class="good">좋아요 👍</span> <img src="https://img.icons8.com/color/48/000000/face-mask.png"/>`;
        goodSound.play();
        lastMessage = "good";
      } else if (noMaskProb >= 0.4 && lastMessage !== "warn") {
        messageDiv.innerHTML = `<span class="warn">아쉬워요 😷</span> <img src="https://img.icons8.com/color/48/000000/no-mask.png"/>`;
        warnSound.play();
        lastMessage = "warn";
      } else if (maskProb < 0.85 && noMaskProb < 0.4 && lastMessage !== "") {
        messageDiv.innerHTML = "";
        lastMessage = "";
      }

      drawPose(pose);
    }

    function drawPose(pose) {
      if (webcam.canvas) {
        ctx.drawImage(webcam.canvas, 0, 0);
        if (pose) {
          const minPartConfidence = 0.5;
          tmPose.drawKeypoints(pose.keypoints, minPartConfidence, ctx);
          tmPose.drawSkeleton(pose.keypoints, minPartConfidence, ctx);
        }
      }
    }
  </script>
</body>
</html>
