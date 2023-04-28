<template>
  <div>
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

