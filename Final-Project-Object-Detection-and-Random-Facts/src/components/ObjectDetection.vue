<template>
  <div style="display: flex; flex-direction: column; align-items: center;">
    <div style="display: flex;">
      <div style="position: relative;">
        <video ref="videoElement" width="400" height="300" autoplay></video>
        <canvas ref="videoCanvas" width="400" height="300" style="position: absolute; top: 0; left: 0;"></canvas>
      </div>
      <div style="margin-left: 20px;">
        <canvas ref="capturedCanvas" width="400" height="300"></canvas>
      </div>
    </div>
    <div>
      <button @click="startVideo">Start Video</button>
      <button @click="stopVideo">Stop Video</button>
      <button @click="captureImage">Capture Image</button>
    </div>
    <div v-if="loading">Loading...</div>
    <div v-else-if="result.length > 0">
      <ul>
        <li v-for="(item, index) in result" :key="index" v-html="item"></li>
      </ul>
    </div>    
    <div v-else>No result yet.</div>
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
      colors: ['red', 'green', 'blue', 'yellow', 'purple', 'orange'], // Add more colors if needed
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
      }
    },
    async captureFrame() {
      const videoCanvas = this.$refs.videoCanvas;
      const videoContext = videoCanvas.getContext("2d");
      videoContext.drawImage(this.$refs.videoElement, 0, 0, videoCanvas.width, videoCanvas.height);

      const imageData = videoCanvas.toDataURL("image/jpeg");
      const blob = await fetch(imageData).then((res) => res.blob());
      return blob;
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
      const videoCanvas = this.$refs.videoCanvas;
      const context = videoCanvas.getContext("2d");

      context.clearRect(0, 0, videoCanvas.width, videoCanvas.height);

      this.boundingBoxes.forEach((detection, index) => {
        const { xmin, ymin, xmax, ymax } = detection.box;
        const color = this.colors[index % this.colors.length];

        context.strokeStyle = color;
        context.lineWidth = 2;
        context.strokeRect(xmin, ymin, xmax - xmin, ymax - ymin);
      });

      const capturedCanvas = this.$refs.capturedCanvas;
      const capturedContext = capturedCanvas.getContext("2d");
      capturedContext.drawImage(videoCanvas, 0, 0);

      this.boundingBoxes.forEach((detection, index) => {
        const { xmin, ymin, xmax, ymax } = detection.box;
        const color = this.colors[index % this.colors.length];

        capturedContext.strokeStyle = color;
        capturedContext.lineWidth = 2;
        capturedContext.strokeRect(xmin, ymin, xmax - xmin, ymax - ymin);
      });
    },
    async captureImage() {
      const imageBlob = await this.captureFrame();
      this.processImage(imageBlob);

      const redrawVideoFrame = () => {
        const videoCanvas = this.$refs.videoCanvas;
        const videoContext = videoCanvas.getContext("2d");
        videoContext.drawImage(this.$refs.videoElement, 0, 0, videoCanvas.width, videoCanvas.height);
        requestAnimationFrame(redrawVideoFrame);
      };
      redrawVideoFrame();

      const capturedCanvas = this.$refs.capturedCanvas;
      const context = capturedCanvas.getContext("2d");
      const capturedImage = new Image();
      capturedImage.onload = () => {
        context.drawImage(capturedImage, 0, 0, capturedCanvas.width, capturedCanvas.height);
      };
      capturedImage.src = URL.createObjectURL(imageBlob);
    },
    async generateFacts(label) {
      try {
        const data = { inputs: `Did you know that, a ${label}` };
        const response = await fetch(
          "https://api-inference.huggingface.co/models/google/gemma-7b",
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
          return result[0].generated_text.replace(/\n/g, ' '); // Replace new lines with <br> tags
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
/* Your styles here */
</style>


