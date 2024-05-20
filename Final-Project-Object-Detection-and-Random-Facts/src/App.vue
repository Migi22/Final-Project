<!-- App.vue -->
<template>
  <div id="app">
    <ObjectDetection />
  </div>
</template>

<script>
//import ImageUploader from "./components/ImageUploader.vue";
import ObjectDetection from "./components/ObjectDetection.vue";
export default {
  components: {
    ObjectDetection,
  },
};
</script>
