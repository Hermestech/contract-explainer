<template>
  <div class="image-preview">
    <img :src="src" alt="Cargando imagen..." />
  </div>
</template>

<script>
export default {
  props: {
    src: {
      type: String,
      required: true,
    },
  },
};
</script>

<style scoped>
.image-preview {
  width: 320px;
  height: 180px;
  overflow: hidden;
  position: relative;
}

.image-preview img {
  object-fit: cover;
  width: 100%;
  height: 100%;
}
</style>

