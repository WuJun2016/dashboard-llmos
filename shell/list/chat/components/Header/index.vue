<script setup>
import { ref } from "vue";
const num = ref(1000);
console.log("----num.value", num.value);
</script>

<template>
  <div class="flex justify-between items-center">
    <div class="flex items-center gap-x-5">
      <!-- <h1 class="mb-0">
        {{ t("chat.type.chat") }}
      </h1> -->

      <!-- <img
        v-if="isChatType"
        class="size-[30px]"
        :src="modelLogo"
      > -->
    </div>
    <h2>header content {{ num }}</h2>
    <p>h-p {{ num }}</p>
    <!-- <div class="flex items-center gap-x-3">
      <slot name="extra" />
      <ButtonGroup v-model:value="chatType" :options="options" />
      <a-button type="link" @click="toggleDrawer">
        <i class="icon icon-lg icon-gear" />
      </a-button>
    </div> -->
  </div>
</template>
