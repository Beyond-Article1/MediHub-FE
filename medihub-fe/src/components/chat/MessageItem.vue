<script setup>
import { ref, computed } from 'vue';
import axios from "axios";

const props = defineProps({
  message: Object,
  isDateDivider: Boolean,
  currentUserSeq: Number,
})

const isMyMessage = computed(() => props.message.senderUserSeq === props.currentUserSeq);

const openFile = (url) => {
  window.open(url, "_blank");
};

const getFileIcon = (filename) => {
  const extension = filename.split(".").pop().toLowerCase();
  switch(extension) {
    case "pdf":
      return "/chat/pdf-icon.png";
    case "xls":
    case "xlsx":
      return "/chat/excel-icon.png"
    case "doc":
    case "docx":
      return "/chat/word-icon.png"
    default:
      return "/chat/download.png"
  }
};

// 메시지 삭제하기
const hovered = ref(false); // 마우스 오버 상태 관리
const deleteMessage = async (messageSeq) => {
  try {
    //console.log("messageSeq: ", messageSeq);
    await axios.delete(`/chat/${messageSeq}`);
    console.log('메시지 삭제 요청 전송 성공');
  } catch (error) {
    console.error('메시지 삭제 요청 실패: ', error);
    alert('메시지 삭제 중 오류가 발생했습니다. 다시 시도해주세요.');
  }
}

// 날짜 포맷 함수
const formatDate = (timestamp) => {
  if (!timestamp) return "";
  const date = new Date(timestamp);
  const month = (date.getMonth() + 1).toString().padStart(2, "0");
  const day = date.getDate().toString().padStart(2, "0");
  return `${month}월 ${day}일`;
};

// 시간 포맷 함수
const formatTime = (timestamp) => {
  if (!timestamp) return "";
  const date = new Date(timestamp);
  const hours = date.getHours();
  const minutes = date.getMinutes().toString().padStart(2, "0");
  return `${hours}:${minutes}`;
};
</script>

<template>
  <div>
    <!-- 날짜 구분선 -->
    <div v-if="isDateDivider" class="date-divider">
      <span class="line"></span>
      <span class="date-text">{{ formatDate(props.message.createdAt) }}</span>
      <span class="line"></span>
    </div>

    <!-- notify 타입 -->
    <template v-if="props.message.type === 'notify'">
      <div class="notify-message">{{ props.message.message }}</div>
    </template>

    <!-- 내가 보낸 메시지 -->
    <template v-else-if="isMyMessage">
      <div class="my-message">
        <div class="my-message-content" @mouseover="hovered = true" @mouseleave="hovered = false">
          <p class="message-time">{{ formatTime(props.message.createdAt) }}
            <!-- 삭제 버튼 -->
            <span v-if="hovered" @click="deleteMessage(props.message.messageSeq)" class="delete-button">❌</span>
          </p>
          <div class="my-message-content-row">
            <p v-if="props.message.type === 'text'" class="message-text">{{ props.message.message }}</p>
            <img v-else-if="props.message.type === 'image'" class="message-image"
                 :src="props.message.attachment.url" alt="이미지"
                 @click="openFile(props.message.attachment.url)"
            />
            <div v-else-if="props.message.type === 'file'" class="file-message">
              <img :src="getFileIcon(props.message.attachment.originName)" alt="파일 아이콘" />
              <div class="file-message-row">
                <span>{{ props.message.attachment.originName }}</span>
                <span class="open-link" @click="openFile(props.message.attachment.url)">열기</span>              </div>
            </div>
          </div>
        </div>
      </div>
    </template>

    <!-- 다른 사람이 보낸 메시지 -->
    <template v-else>
      <div class="other-message">
        <img class="profile-img" :src="props.message.senderUserProfileUrl || '/default-profile.png'" alt="프로필 이미지" />
        <div class="other-message-content">
          <p class="sender-name">{{ props.message.senderUserName }}</p>
          <div class="other-message-content-row">
            <p v-if="props.message.type === 'text'" class="message-text">{{ props.message.message }}</p>
            <img v-else-if="props.message.type === 'image'" class="message-image"
                 :src="props.message.attachment.url" alt="이미지"
                 @click="openFile(props.message.attachment.url)"
            />
            <div v-else-if="props.message.type === 'file'" class="file-message">
              <img :src="getFileIcon(props.message.attachment.originName)" alt="파일 아이콘" />
              <div class="file-message-row">
                <span>{{ props.message.attachment.originName }}</span>
                <span class="open-link" @click="openFile(props.message.attachment.url)">열기</span>              </div>
            </div>
            <p class="message-time">{{ formatTime(props.message.createdAt) }}</p>
          </div>
        </div>
      </div>
    </template>
  </div>
</template>

<style scoped>
.date-divider {
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 10px 0;
  color: gray;
  font-size: 0.9rem;
}

.date-divider .line {
  flex: 1;
  height: 1px;
  background-color: #ccc;
}

.date-divider .date-text {
  margin: 0 10px;
}

.notify-message {
  text-align: center;
  font-style: italic;
  font-size: 0.9rem;
  color: #666;
  padding: 10px;
  border-radius: 8px;
}

.message-image {
  max-width: 100%;
  max-height: 300px;
  object-fit: contain; /* 비율 유지하며 축소 */
  border-radius: 8px;
}

.my-message {
  display: flex;
  justify-content: flex-end;
}

.my-message-content-row {
  display: flex;
  flex-direction: row;
  align-items: flex-end;
  border-radius: 10px;
}

.delete-button {
  cursor: pointer;
  margin-left: 10px;
}

.my-message-content {
  display: flex;
  color: black;
  max-width: 70%;
  word-wrap: break-word;
  align-items: flex-end;
}

.my-message-content .message-text {
  background-color: #ffe3a9;
  padding: 10px;
  border-radius: 10px;
  color: black;
  word-wrap: break-word;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.my-message-content .file-message {
  background-color: #ffe3a9;
}

.my-message-content .file-message-row {
  background-color: #ffe3a9;
}

.my-message .message-time {
  font-size: 0.8rem;
  color: #777;
  margin-top: 5px;
  margin-right: 5px;
  text-align: left;
}

.other-message {
  display: flex;
  justify-content: flex-start;
  margin-bottom: 5px;
}

.other-message-content {
  display: flex;
  flex-direction: column;
  max-width: 70%;
}

.other-message-content-row {
  display: flex;
  flex-direction: row;
  align-items: flex-end;
}

.profile-img {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  margin-right: 10px;
}

.sender-name {
  font-weight: bold;
  color: #333;
  font-size: 0.9rem;
  margin-bottom: 5px;
}

.other-message-content .message-text {
  background-color: white;
  padding: 10px;
  border-radius: 10px;
  color: black;
  word-wrap: break-word;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.other-message .message-time {
  font-size: 0.8rem;
  color: #777;
  text-align: left;
  margin-top: 5px;
  margin-left: 5px;
}

/* 파일 메시지 */
.file-message {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
  margin-top: 10px;
  padding: 10px;
  background-color: #f9f9f9;
  border-radius: 8px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.file-message-row {
  display: flex;
  flex-direction: column;
  background-color: #f9f9f9;
  border-radius: 8px;
}

.file-message img {
  width: 50px;
  height: 50px;
  margin-right: 10px;
}

.file-message span {
  margin-right: 10px;
}

.file-message .open-link {
  font-size: 14px;
  color: #007bff;
  cursor: pointer;
  text-decoration: none;
  margin-left: auto;
}

.file-message .open-link:hover {
  text-decoration: underline;
}

.file-message button {
  background: none;
  border: none;
  padding: 5px 5px;
  font-size: 0.8rem;
  color: silver;
  cursor: pointer;
  text-decoration: underline;
}
</style>