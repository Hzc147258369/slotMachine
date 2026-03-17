<template>
  <div @click="toFruitMachine">跳转到水果机</div>
</template>
<script setup>
const toFruitMachine = () => {
  uni.navigateTo({
    url: "/pages/fruitMachine/index",
  });
};
</script>
<style scoped></style>
