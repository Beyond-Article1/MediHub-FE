<script setup>
import {defineEmits, defineProps} from "vue";
import afterBookmark from "@/assets/images/bookmark/after-bookmark.png";
import beforeBookmark from "@/assets/images/bookmark/before-bookmark.png";

// 부모 컴포넌트로부터 전달받는 props
const props = defineProps({
  currentIsBookmark: Boolean, // 현재 북마크 유무 true, false
});

// 이벤트 전송 emit
const emit = defineEmits(["updateBookmark"]);

// 북마크 버튼 클릭 이벤트
function onClickBookmark(event) {
  event.stopPropagation(); // 부모 이벤트 전파 방지
  // 현재 북마크 상태를 반대로 만들어 상위 컴포넌트에 전달
  emit("updateBookmark", !props.currentIsBookmark);
}
</script>

<template>
  <div class="bookmark-btn align-mid" @click="onClickBookmark">
    <img :src="currentIsBookmark ? afterBookmark : beforeBookmark"
         :alt="currentIsBookmark ? 'After' : 'Before'"/>
  </div>
</template>

<style scoped>
.bookmark-btn {
  cursor: pointer;
}

.bookmark-btn img {
  width: 25px;
}
</style>
