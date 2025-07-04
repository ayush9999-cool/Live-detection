# Live-detection
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>YOLOv8 ONNX Runtime Web Demo</title>
  <style>
    body, html { margin:0; padding:0; background:#222; color:#eee; font-family: Arial,sans-serif; }
    #container { position: relative; width: 100vw; height: 100vh; overflow: hidden; }
    video, canvas { position: absolute; top: 0; left: 0; width: 100%; height: 100%; object-fit: cover; }
    #info { position: absolute; bottom: 10px; left: 10px; background: rgba(0,0,0,0.7); padding: 8px; border-radius: 6px; max-width: 95vw; }
  </style>
</head>
<body>
<div id="container">
  <video id="video" autoplay muted playsinline></video>
  <canvas id="canvas"></canvas>
  <div id="info">Loading model...</div>
</div>

<script src="https://cdn.jsdelivr.net/npm/onnxruntime-web/dist/ort.min.js"></script>

<script>
(async () => {
  const video = document.getElementById('video');
  const canvas = document.getElementById('canvas');
  const ctx = canvas.getContext('2d');
  const info = document.getElementById('info');

  // Set up back camera video
  async function setupCamera() {
    const stream = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: 'environment', width: 640, height: 480 },
      audio: false
    });
    video.srcObject = stream;
    return new Promise(resolve => {
      video.onloadedmetadata = () => {
        resolve(video);
      };
    });
  }

  await setupCamera();

  canvas.width = video.videoWidth;
  canvas.height = video.videoHeight;

  // Load ONNX YOLOv8 model (COCO pretrained weights converted to ONNX & hosted)
  // Note: Replace with a valid publicly hosted model URL for better availability
  const modelURL = 'https://onnxruntimewebdemo.blob.core.windows.net/models/yolov8n.onnx';

  info.innerText = 'Loading YOLOv8 model...';
  const session = await ort.InferenceSession.create(modelURL);
  info.innerText = 'Model loaded. Starting detection...';

  // Postprocessing helper functions:

  // YOLOv8 COCO class labels (80 classes)
  const classes = [
    "person", "bicycle", "car", "motorcycle", "airplane", "bus", "train", "truck", "boat",
    "traffic light", "fire hydrant", "stop sign", "parking meter", "bench", "bird", "cat",
    "dog", "horse", "sheep", "cow", "elephant", "bear", "zebra", "giraffe", "backpack",
    "umbrella", "handbag", "tie", "suitcase", "frisbee", "skis", "snowboard", "sports ball",
    "kite", "baseball bat", "baseball glove", "skateboard", "surfboard", "tennis racket",
    "bottle", "wine glass", "cup", "fork", "knife", "spoon", "bowl", "banana", "apple",
    "sandwich", "orange", "broccoli", "carrot", "hot dog", "pizza", "donut", "cake",
    "chair", "couch", "potted plant", "bed", "dining table", "toilet", "tv", "laptop",
    "mouse", "remote", "keyboard", "cell phone", "microwave", "oven", "toaster",
    "sink", "refrigerator", "book", "clock", "vase", "scissors", "teddy bear", "hair drier",
    "toothbrush"
  ];

  // Helper: preprocess frame to model input
  function preprocess(image) {
    // Resize to 640x640 with letterboxing, normalize to [0,1], transpose HWC->CHW
    const inputSize = 640;
    const canvas = document.createElement('canvas');
    canvas.width = inputSize;
    canvas.height = inputSize;
    const ctx = canvas.getContext('2d');

    // Letterbox scaling
    const scale = Math.min(inputSize / image.width, inputSize / image.height);
    const scaledWidth = image.width * scale;
    const scaledHeight = image.height * scale;
    const dx = (inputSize - scaledWidth) / 2;
    const dy = (inputSize - scaledHeight) / 2;

    ctx.fillStyle = 'black';
    ctx.fillRect(0, 0, inputSize, inputSize);
    ctx.drawImage(image, dx, dy, scaledWidth, scaledHeight);

    // Get image data and convert to Float32 tensor
    const imageData = ctx.getImageData(0, 0, inputSize, inputSize);
    const data = imageData.data;

    const float32Data = new Float32Array(3 * inputSize * inputSize);
    // Normalize and transpose
    for (let i = 0; i < inputSize * inputSize; i++) {
      float32Data[i] = data[i * 4] / 255.0;           // R
      float32Data[i + inputSize * inputSize] = data[i * 4 + 1] / 255.0; // G
      float32Data[i + 2 * inputSize * inputSize] = data[i * 4 + 2] / 255.0; // B
    }

    return new ort.Tensor('float32', float32Data, [1, 3, inputSize, inputSize]);
  }

  // Helper: parse YOLOv8 outputs and do NMS
  function postprocess(output, threshold=0.3, iouThreshold=0.45) {
    // output is [1, 8400, 85] tensor: [x_center, y_center, w, h, conf, class_probs...]

    const data = output.data;
    const numBoxes = output.dims[1];
    const boxes = [];
    for (let i = 0; i < numBoxes; i++) {
      const offset = i * 85;
      const scores = data.subarray(offset + 5, offset + 85);
      const classId = scores.indexOf(Math.max(...scores));
      const classScore = scores[classId];
      const conf = data[offset + 4];
      const finalScore = conf * classScore;
      if (finalScore > threshold) {
        // decode box
        const xCenter = data[offset];
        const yCenter = data[offset + 1];
        const width = data[offset + 2];
        const height = data[offset + 3];
        const xMin = xCenter - width / 2;
        const yMin = yCenter - height / 2;
        boxes.push({
          classId,
          label: classes[classId],
          score: finalScore,
          bbox: [xMin, yMin, width, height]
        });
      }
    }
    // Non-Maximum Suppression (NMS)
    return nms(boxes, iouThreshold);
  }

  // Simple NMS implementation
  function nms(boxes, iouThreshold) {
    boxes.sort((a,b) => b.score - a.score);
    const result = [];
    while (boxes.length > 0) {
      const current = boxes.shift();
      result.push(current);
      boxes = boxes.filter(box => iou(current.bbox, box.bbox) < iouThreshold);
    }
    return result;
  }

  // IOU calculation between two boxes [x,y,w,h]
  function iou(box1, box2) {
    const [x1, y1, w1, h1] = box1;
    const [x2, y2, w2, h2] = box2;
    const area1 = w1 * h1;
    const area2 = w2 * h2;
    const xi1 = Math.max(x1, x2);
    const yi1 = Math.max(y1, y2);
    const xi2 = Math.min(x1 + w1, x2 + w2);
    const yi2 = Math.min(y1 + h1, y2 + h2);
    const interWidth = Math.max(0, xi2 - xi1);
    const interHeight = Math.max(0, yi2 - yi1);
    const interArea = interWidth * interHeight;
    return interArea / (area1 + area2 - interArea);
  }

  // Distance estimation - same heuristic as before
  function estimateDistance(bboxHeight, canvasHeight) {
    const normalized = bboxHeight / canvasHeight;
    if (normalized <= 0) return null;
    const dist = Math.min(4, Math.max(0.3, 1.5 / normalized));
    return dist;
  }

  // Convert meters to steps (roughly 0.75m per step)
  function metersToSteps(m) {
    if (m === null) return null;
    return Math.round(m / 0.75);
  }

  // Speech throttling and priority (biggest box first)
  let lastSpeakTime = 0;
  const speakCooldown = 4000; // 4 sec cooldown

  function speak(text) {
    const now = Date.now();
    if (now - lastSpeakTime > speakCooldown) {
      const utterance = new SpeechSynthesisUtterance(text);
      speechSynthesis.speak(utterance);
      lastSpeakTime = now;
    }
  }

  // Main detection loop
  async function detectFrame() {
    ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

    const inputTensor = preprocess(video);
    const feeds = { images: inputTensor };

    try {
      const results = await session.run(feeds);
      const output = results.output;

      const boxes = postprocess(output, 0.4, 0.5);

      // Clear canvas overlay
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

      if (boxes.length > 0) {
        // Sort by box area descending
        boxes.sort((a, b) => (b.bbox[2] * b.bbox[3]) - (a.bbox[2] * a.bbox[3]));

        // Filter boxes with distance <=4m approx
        const filteredBoxes = boxes.filter(b => {
          const dist = estimateDistance(b.bbox[3] * canvas.height, canvas.height);
          return dist !== null && dist <= 4;
        });

        // Draw boxes and labels
        for (const box of filteredBoxes) {
          const [x, y, w, h] = box.bbox;
          const xPx = x * canvas.width;
          const yPx = y * canvas.height;
          const wPx = w * canvas.width;
          const hPx = h * canvas.height;
          ctx.strokeStyle = '#00FF00';
          ctx.lineWidth = 3;
          ctx.strokeRect(xPx, yPx, wPx, hPx);

          const dist = estimateDistance(hPx, canvas.height);
          const distSteps = metersToSteps(dist);
          const labelText = `${box.label} ${(dist).toFixed(2)}m (${distSteps} steps)`;
          ctx.fillStyle = '#00FF00';
          ctx.font = '18px Arial';
          ctx.fillText(labelText, xPx + 5, yPx + 20);
        }

        // Speak the biggest detected box label + distance
        if (filteredBoxes.length > 0) {
          const biggest = filteredBoxes[0];
          const dist = estimateDistance(biggest.bbox[3] * canvas.height, canvas.height);
          speak(`${biggest.label} at approximately ${dist.toFixed(1)} meters`);
        }
      }

    } catch (e) {
      console.error(e);
      info.innerText = 'Error in detection.';
    }

    requestAnimationFrame(detectFrame);
  }

  detectFrame();

})();
</script>
</body>
</html>
