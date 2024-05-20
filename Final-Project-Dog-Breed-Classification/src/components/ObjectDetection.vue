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
              <li v-for="(item, index) in result" :key="index">
                  {{ `${index + 1}. ${item}` }}
              </li>
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
        boundingBox: null,
        detecting: false,
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
            this.result = []; // Initialize result as an empty array

            // Process each detection and extract required data
            for (const detection of result) {
                const { score, label } = detection;
                const facts = await this.generateFacts(label);
                const formattedResult = `Detected: ${label}, Confidence level: ${Math.round(score * 100)}%, ${facts}`;
                this.result.push(formattedResult); // Push formatted result to result array
            }

            this.loading = false;

            // Extract bounding box from the first result if available
            if (result.length > 0) {
                this.boundingBox = result[0].box; // Assuming the first result contains the bounding box
                this.drawBoundingBox(); // Call method to draw the bounding box
            } else {
                this.boundingBox = null; // Reset bounding box if no object detected
            }
        } catch (error) {
            console.error("Error processing image:", error);
            this.loading = false;
        }
      },


      drawBoundingBox() {
        const videoCanvas = this.$refs.videoCanvas;
        const context = videoCanvas.getContext("2d");
        const { xmin, ymin, xmax, ymax } = this.boundingBox;
  
        // Clear previous drawings
        context.clearRect(0, 0, videoCanvas.width, videoCanvas.height);
  
        // Draw the bounding box
        context.strokeStyle = "red"; // Set color to red
        context.lineWidth = 2;
        context.strokeRect(xmin, ymin, xmax - xmin, ymax - ymin);
  
        // Copy the captured frame to the second canvas
        const capturedCanvas = this.$refs.capturedCanvas;
        const capturedContext = capturedCanvas.getContext("2d");
        capturedContext.drawImage(videoCanvas, 0, 0);
  
        // Draw bounding box on the captured image canvas
        capturedContext.strokeStyle = "red"; // Set color to red
        capturedContext.lineWidth = 2;
        capturedContext.strokeRect(xmin, ymin, xmax - xmin, ymax - ymin);
      },
      async captureImage() {
        const imageBlob = await this.captureFrame();
        this.processImage(imageBlob);

        // Redraw the video frame on the videoCanvas continuously
        const redrawVideoFrame = () => {
          const videoCanvas = this.$refs.videoCanvas;
          const videoContext = videoCanvas.getContext("2d");
          videoContext.drawImage(this.$refs.videoElement, 0, 0, videoCanvas.width, videoCanvas.height);
          requestAnimationFrame(redrawVideoFrame); // Request next frame
        };
        redrawVideoFrame(); // Start continuous redraw

        // Draw the captured image on the capturedCanvas
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
            const data = { "inputs": `Can you give us a 5 facts about a ${label}` }; // Dummy data for debugging
            const response = await fetch(
                "https://api-inference.huggingface.co/models/meta-llama/Meta-Llama-3-8B-Instruct",
                {
                    headers: {
                        Authorization: "Bearer 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",
                    },
                    method: "POST",
                    //body: JSON.stringify({ "inputs": `Can you please let us know more details about a ${label}` }),
                    body: JSON.stringify(data),
                }
            );

            // Check if response is successful
            if (!response.ok) {
                const errorMessage = `Error ${response.status}: ${response.statusText}`;
                throw new Error(errorMessage);
            }

            const result = await response.json();
            return result.generated_text;
        } catch (error) {
            console.error("Error generating facts:", error);
            return "Could not generate facts";
        }
      },



    },
    beforeDestroy() {
      this.stopVideo();
    },
  };
  </script>
  
  <style>
  /* Your styles here */
  </style>
  