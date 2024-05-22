<template>
  <div class="container">
    <div class="video-container">
      <div class="video-wrapper">
        <video ref="videoElement" class="video-element" width="400" height="300" autoplay></video>
      </div>
      <div class="captured-canvas-wrapper">
        <canvas ref="capturedCanvas" class="captured-canvas" width="400" height="300"></canvas>
      </div>
    </div>
    <div class="buttons-container">
      <button @click="startVideo" class="action-button">Start Video</button>
      <button @click="stopVideo" class="action-button">Stop Video</button>
      <button @click="captureImage" class="action-button">Capture Image</button>
    </div>
    <div v-if="loading" class="loading-text">Capturing Object, Please don't move...</div>
    <div v-else-if="result.length > 0" class="result-list">
      <ul>
        <li v-for="(item, index) in result" :key="index" v-html="item"></li>
      </ul>
    </div>
    <div v-else class="no-result-text">No result yet.</div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      loading: false,
      result: [],
      videoStream: null,
      boundingBoxes: [],
      colors: ['red', 'green', 'blue', 'yellow', 'purple', 'orange'],
      prompts: [
        "Amazing fact about a",
        "Interesting information about a",
        "Did you know about a",
        "Fun fact about a",
        "Here's something cool about a",
        "Learn something new about a"
      ],
    };
  },
  methods: {
    async startVideo() {
      this.videoStream = await navigator.mediaDevices.getUserMedia({ video: true });
      this.$refs.videoElement.srcObject = this.videoStream;
    },
    stopVideo() {
      if (this.videoStream) {
        this.videoStream.getTracks().forEach((track) => track.stop());
        this.$refs.videoElement.srcObject = null;
        this.videoStream = null;
      }
    },
    async captureFrame() {
      const videoElement = this.$refs.videoElement;
      const canvas = document.createElement("canvas");
      canvas.width = videoElement.videoWidth;
      canvas.height = videoElement.videoHeight;
      const context = canvas.getContext("2d");
      context.drawImage(videoElement, 0, 0, canvas.width, canvas.height);

      const imageData = canvas.toDataURL("image/jpeg");
      const blob = await fetch(imageData).then((res) => res.blob());
      return blob;
    },
    getRandomPrompt() {
      const randomIndex = Math.floor(Math.random() * this.prompts.length);
      return this.prompts[randomIndex];
    },
    async processImage(imageBlob) {
      try {
        this.loading = true;
        const response = await fetch(
          "https://api-inference.huggingface.co/models/facebook/detr-resnet-50",
          {
            headers: {
              Authorization: "Bearer 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",
            },
            method: "POST",
            body: imageBlob,
          }
        );
        const result = await response.json();
        this.result = [];
        this.boundingBoxes = [];

        for (const detection of result) {
          const { score, label, box } = detection;
          const facts = await this.generateFacts(label);
          const color = this.colors[this.result.length % this.colors.length]; // Get color based on result index
          const formattedResult = `
            Detected: ${label}<br>
            Color: ${color}<br>
            Confidence level: ${Math.round(score * 100)}%<br>
            Random Facts: ${facts}
          `;
          this.result.push(formattedResult);
          this.boundingBoxes.push({ box, label, color });
        }

        this.loading = false;
        this.drawBoundingBoxes();
      } catch (error) {
        console.error("Error processing image:", error);
        this.loading = false;
      }
    },
    drawBoundingBoxes() {
      const videoElement = this.$refs.videoElement;
      const capturedCanvas = this.$refs.capturedCanvas;
      const context = capturedCanvas.getContext("2d");

      // Clear the canvas
      context.clearRect(0, 0, capturedCanvas.width, capturedCanvas.height);

      // Draw the current video frame on the canvas
      context.drawImage(videoElement, 0, 0, capturedCanvas.width, capturedCanvas.height);

      // Draw each bounding box
      this.boundingBoxes.forEach((detection, index) => {
        const { xmin, ymin, xmax, ymax } = detection.box;
        const color = this.colors[index % this.colors.length];

        // Scale the coordinates
        const scaleX = capturedCanvas.width / videoElement.videoWidth;
        const scaleY = capturedCanvas.height / videoElement.videoHeight;

        const scaledXMin = xmin * scaleX;
        const scaledYMin = ymin * scaleY;
        const scaledXMax = xmax * scaleX;
        const scaledYMax = ymax * scaleY;

        context.strokeStyle = color;
        context.lineWidth = 2;
        context.strokeRect(scaledXMin, scaledYMin, scaledXMax - scaledXMin, scaledYMax - scaledYMin);
      });
    },
    async captureImage() {
      const imageBlob = await this.captureFrame();
      this.processImage(imageBlob);
    },
    async generateFacts(label) {
      try {
        const prompt = this.getRandomPrompt();
        const data = { inputs: `${prompt} ${label}` };
        const response = await fetch(
          "https://api-inference.huggingface.co/models/meta-llama/Meta-Llama-3-8B-Instruct",
          {
            headers: {
              Authorization: "Bearer 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",
              "Content-Type": "application/json",
            },
            method: "POST",
            body: JSON.stringify(data),
          }
        );

        if (!response.ok) {
          const errorMessage = `Error ${response.status}: ${response.statusText}`;
          throw new Error(errorMessage);
        }

        const result = await response.json();
        console.log(result);

        if (result.length > 0 && result[0].generated_text) {
          return result[0].generated_text.replace(/\n/g, ' '); // Replace new lines with spaces
        } else {
          return "No facts generated";
        }
      } catch (error) {
        console.error("Error generating facts:", error);
        return "Could not generate facts";
      }
    }
  },
  beforeDestroy() {
    this.stopVideo();
  },
};
</script>

<style>
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.video-container {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-bottom: 20px;
}

.video-wrapper {
  position: relative;
}

.video-element, .captured-canvas {
  border: 1px solid #ccc;
}

.captured-canvas-wrapper {
  margin-left: 20px;
}

.buttons-container {
  display: flex;
  justify-content: center;
  margin-bottom: 20px;
}

.action-button {
  margin: 0 10px;
  padding: 10px 20px;
  background-color: #007bff;
  color: #fff;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.loading-text {
  margin-bottom: 10px;
}

.result-list {
  margin-bottom: 10px;
}

.no-result-text {
  font-style: italic;
}
</style>
