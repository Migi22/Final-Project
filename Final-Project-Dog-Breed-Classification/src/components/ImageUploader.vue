<template>
  <div>
    <input type="file" @change="handleFileUpload" accept="image/*" />
    <div v-if="imageUrl">
      <img :src="imageUrl" alt="Uploaded Image" style="max-width: 300px" />
    </div>
    <button @click="query" :disabled="!imageUrl">Process Image</button>
    <div v-if="result">
      <h2>Result:</h2>
      <pre>{{ result }}</pre>
      <h2>Generated Text:</h2>
      <pre>{{ generatedText }}</pre>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      imageUrl: null,
      result: null,
      generatedText: null,
    };
  },

  methods: {
    handleFileUpload(event) {
      const file = event.target.files[0];
      if (file) {
        this.imageUrl = URL.createObjectURL(file);
      }
    },

    async query() {
      try {
        const data = await fetch(this.imageUrl).then((response) =>
          response.blob()
        );

        const response = await fetch(
          "https://api-inference.huggingface.co/models/google/vit-base-patch16-224",
          {
            headers: {
              Authorization: "Bearer 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",
            },
            method: "POST",
            body: data,
          }
        );

        const results = await response.json();
        const firstResult = results[0]; // Get the first result

        if (firstResult) {
          const label = firstResult.label.split(",")[0]; // Extract the label
          const scorePercentage = Math.round(firstResult.score * 100); // Round the score to a whole number

          this.result = `Detected: ${label}: ${scorePercentage}%`;
          console.log(this.result);

          // Generate additional text using the result label from the query
          await this.resultLabelQuery({
            "inputs": `Can you please give us facts about german sheppered`
          });
        } else {
          console.error("No results found.");
        }
      } catch (error) {
        console.error("Error:", error);
      }
    },

    async resultLabelQuery(data) {
      try {
        console.log(JSON.stringify(data))
        const response = await fetch(
          "https://api-inference.huggingface.co/models/microsoft/Phi-3-mini-4k-instruct",
          {
            headers: {
              Authorization: "Bearer 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",
            },
            method: "POST",
            body: JSON.stringify(data),
          }
        );
        const result = await response.json();
        //this.generatedText = result[0].generated_text; // Assuming the generated text is in the first item of the result array
        console.log(result);
      } catch (error) {
        console.error("Error:", error);
      }
    },
  },
};
</script>

<style scoped>
/* Your CSS styles here */
</style>
