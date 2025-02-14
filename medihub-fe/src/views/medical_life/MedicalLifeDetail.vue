<script setup>
import {ref, onMounted, computed} from 'vue';
import {useRoute, useRouter} from 'vue-router';

import axios from 'axios';
import LocalDateTimeFormat from '@/components/common/LocalDateTimeFormat.vue';
import beforeLike from '@/assets/images/like/before-like.png';
import afterLike from '@/assets/images/like/after-like.png';
import beforeBookmark from "@/assets/images/bookmark/before-bookmark.png";
import afterBookmark from "@/assets/images/bookmark/after-bookmark.png";
import LineDivider from "@/components/medicallife/MedicalLifeLineDivider.vue";
import Pagination from '@/components/common/Pagination.vue';
import MedicalLifeContent from '@/components/medicallife/MedicalLifeContent.vue'

const boardDetail = ref({});
const comment = ref([]);
const commentCount = ref(0);
const currentCommentPage = ref(1);
const commentItemCount = ref(10);
const totalComment = ref(0);
const newCommentContent = ref('');
const characterCount = ref(0);
const route = useRoute();
const router = useRouter(); // 라우터 추가
const token = localStorage.getItem('accessToken');
const decodedToken = token ? JSON.parse(atob(token.split('.')[1])) : null;
const loggedInUserSeq = decodedToken ? decodedToken.userSeq : null;
const loggedInUserRole = decodedToken ? decodedToken.auth : null;

const isAuthorizedToModify = (commentUserSeq) => {

  return loggedInUserSeq === commentUserSeq || loggedInUserRole === 'ADMIN';
};

const fetchBoardDetail = async () => {
  const medicalLifeSeq = route.params.id;

  try {
    const response = await axios.get(`/medical-life/detail/${medicalLifeSeq}`, {
      withCredentials: true,
    });
    boardDetail.value = response.data.data || {};
    boardDetail.value.isLiked = false; // 좋아요 초기값
    boardDetail.value.isBookmark = false; // 북마크 초기값
    boardDetail.value.isAuthor = response.data.data.isAuthor; // 작성자 여부 추가

    await Promise.all([
      checkLikeStatus(medicalLifeSeq),
      checkBookmarkStatus(medicalLifeSeq),
    ]);

    await fetchComment(medicalLifeSeq);
  } catch (error) {
    console.error('Error fetching board detail:', error);
  }
};

const fetchComment = async (medicalLifeSeq) => {
  try {
    const response = await axios.get(`/medical-life/${medicalLifeSeq}/comments`);
    comment.value = response.data.data.map((item) => ({
      commentSeq: item.commentSeq,
      userSeq: item.userSeq,
      userName: item.userName,
      part: item.part,
      rankingName: item.rankingName,
      commentContent: item.commentContent,
      createdAt: item.createdAt,
    }));
    commentCount.value = comment.value.length;
    totalComment.value = commentCount.value;
  } catch (error) {
    console.error('Error fetching comments:', error);
  }
};

// 게시글 내용 파싱
const parsedContent = computed(() => {
  try {
    return JSON.parse(boardDetail.value.medicalLifeContent || "{}");
  } catch (error) {
    console.error("Error parsing board content:", error);
    return null;
  }
});


// 좋아요 토글
const toggleLike = async () => {
  const medicalLifeSeq = route.params.id;

  try {
    const response = await axios.patch(`/medical-life/${medicalLifeSeq}/prefer`);
    boardDetail.value.isLiked = response.data.data;
    alert(boardDetail.value.isLiked ? '좋아요가 등록되었습니다.' : '좋아요가 취소되었습니다.');
  } catch (error) {
    console.error('좋아요 처리 중 오류 발생:', error);
    alert('좋아요 처리에 실패했습니다.');
  }
};

// 북마크 토글
const toggleBookmark = async () => {
  const medicalLifeSeq = route.params.id;

  try {
    const response = await axios.patch(`/medical-life/${medicalLifeSeq}/bookmark`);
    boardDetail.value.isBookmark = response.data.data;
    alert(boardDetail.value.isBookmark ? '북마크가 등록되었습니다.' : '북마크가 해제되었습니다.');
  } catch (error) {
    console.error('북마크 처리 중 오류 발생:', error);
    alert('북마크 처리에 실패했습니다.');
  }
};

// 좋아요 상태 확인
const checkLikeStatus = async (medicalLifeSeq) => {
  try {
    const response = await axios.get(`/medical-life/${medicalLifeSeq}/prefer`);
    boardDetail.value.isLiked = response.data.data; // 서버에서 반환된 좋아요 상태
  } catch (error) {
    console.error('좋아요 상태 확인 중 오류 발생:', error);
    boardDetail.value.isLiked = false; // 오류 시 기본값 설정
  }
};

// 북마크 상태 확인
const checkBookmarkStatus = async (medicalLifeSeq) => {
  try {
    const response = await axios.get(`/medical-life/${medicalLifeSeq}/bookmark`);
    boardDetail.value.isBookmark = response.data.data; // 서버에서 반환된 북마크 상태
  } catch (error) {
    console.error('북마크 상태 확인 중 오류 발생:', error);
    boardDetail.value.isBookmark = false; // 오류 시 기본값 설정
  }
};

// 게시글 수정
const handleEdit = () => {
  router.push(`/medicalLife/edit/${route.params.id}`);
};

// 게시글 삭제
const handleDelete = async () => {
  if (confirm('정말로 이 게시글을 삭제하시겠습니까?')) {
    try {
      await axios.delete(`/medical-life/${route.params.id}`);
      alert('게시글이 삭제되었습니다.');
      router.push('/medicalLife');
    } catch (error) {
      console.error('게시글 삭제 실패:', error);
      alert('게시글 삭제에 실패했습니다.');
    }
  }
};

const contentBlocks = computed(() => {
  try {
    const medicalLifeContent = JSON.parse(boardDetail.value.medicalLifeContent);

    return medicalLifeContent.blocks.map((block) => {
      if (block.type === "paragraph") {
        return {type: "text", content: block.data.text};
      } else if (block.type === "image") {
        return {type: "image", content: block.data.file.url};
      } else {
        return null;
      }
    }).filter(Boolean);
  } catch (error) {
    console.error("Error parsing board content:", error);
    return [];
  }
});

// 댓글 페이지네이션
const paginatedComment = computed(() => {
  const start = (currentCommentPage.value - 1) * commentItemCount.value;
  return comment.value.slice(start, start + commentItemCount.value);
});

// 댓글 추가
const currentEditingCommentSeq = ref(null); // 수정 중인 댓글 ID

// 댓글 작성 또는 수정 요청
const saveComment = async () => {
  if (!newCommentContent.value.trim()) {
    alert('댓글 내용을 입력해 주세요.');
    return;
  }

  const medicalLifeSeq = route.params.id;

  try {
    if (currentEditingCommentSeq.value) {
      // 댓글 수정 로직
      await axios.put(
          `/medical-life/${medicalLifeSeq}/comment/${currentEditingCommentSeq.value}`,
          { commentContent: newCommentContent.value },
          { headers: { 'Content-Type': 'application/json' } }
      );

      alert('댓글이 수정되었습니다.');
    } else {
      // 댓글 작성 로직
      await axios.post(
          `/medical-life/${medicalLifeSeq}/comments`,
          { commentContent: newCommentContent.value },
          { headers: { 'Content-Type': 'application/json' } }
      );

      alert('댓글이 등록되었습니다.');
    }

    // 상태 초기화 및 댓글 갱신
    newCommentContent.value = '';
    currentEditingCommentSeq.value = null; // 수정 모드 해제
    fetchComment(medicalLifeSeq); // 댓글 목록 갱신
  } catch (error) {
    console.error('댓글 저장 중 오류 발생:', error);
    alert('댓글 저장에 실패했습니다.');
  }
};

// 댓글 수정 시작
const editComment = (commentItem) => {
  newCommentContent.value = commentItem.commentContent; // 입력 필드에 기존 내용 설정
  currentEditingCommentSeq.value = commentItem.commentSeq; // 수정 중인 댓글 ID 저장
};

// 댓글 삭제
const deleteComment = async (commentSeq) => {
  const medicalLifeSeq = route.params.id;

  if (confirm('정말로 이 댓글을 삭제하시겠습니까?')) {
    try {
      await axios.delete(`/medical-life/${medicalLifeSeq}/comment/${commentSeq}`);
      alert('댓글이 성공적으로 삭제되었습니다.');
      fetchComment(medicalLifeSeq); // 댓글 목록 갱신
    } catch (error) {
      console.error('댓글 삭제 중 오류 발생:', error);
      alert('댓글 삭제에 실패했습니다.');
    }
  }
};


// 문자 수 업데이트
const updateCharacterCount = () => {
  characterCount.value = newCommentContent.value.length;
};

// 페이지 변경
const changePage = (page) => {
  currentCommentPage.value = page;
};

onMounted(() => {
  fetchBoardDetail();
});
</script>

<template>
  <div class="board-detail" v-if="boardDetail.userName">
    <!-- 제목과 기본 정보 -->
    <h1 class="board-title">{{ boardDetail.medicalLifeTitle }}</h1>
    <div class="board-info">
      <img
          class="profile-pic"
          :src="boardDetail.profileImage || '@/assets/images/anonymousBoard/empty-profile.png'"
          alt="Profile Picture"
      />
      <p class="author">{{ boardDetail.userName }} ({{ boardDetail.rankingName }})</p>
      <p class="date">
        <LocalDateTimeFormat :data="boardDetail.createdAt"/>
      </p>
      <p class="view-count">조회수: {{ boardDetail.medicalLifeViewCount }}</p>

      <!-- 좋아요, 북마크 버튼 -->
      <div class="actions">
        <div class="like-btn" @click="toggleLike">
          <img :src="boardDetail.isLiked ? afterLike : beforeLike" alt="좋아요"/>
        </div>
        <div class="bookmark-btn" @click="toggleBookmark">
          <img :src="boardDetail.isBookmark ? afterBookmark : beforeBookmark" alt="북마크"/>
        </div>

        <!-- 수정/삭제 버튼 -->
        <button v-if="isAuthorizedToModify(boardDetail.userSeq)" @click="handleEdit">수정</button>
        <button v-if="isAuthorizedToModify(boardDetail.userSeq)" @click="handleDelete">삭제</button>
      </div>
    </div>

    <LineDivider/>

    <!-- 키워드 -->
    <div v-if="boardDetail.keywords && boardDetail.keywords.length">
      <h2 class="keyword-title">키워드</h2>
      <div class="keyword-list">
        <span v-for="(keyword, index) in boardDetail.keywords" :key="index" class="keyword-item">
          # {{ keyword.keywordName }}
        </span>
      </div>
    </div>

    <LineDivider/>

    <!-- 콘텐츠 표시 -->
    <div class="content">
      <MedicalLifeContent v-if="parsedContent" :content="parsedContent" />
    </div>

    <LineDivider/>

    <!-- 댓글 섹션 -->
    <div class="comment-section">
      <h2>댓글 ({{ commentCount }})</h2>
      <!-- 댓글 목록 -->
      <div v-for="(commentItem, index) in paginatedComment" :key="index" class="comment">
        <p class="comment-author">
          {{ commentItem.userName }} ({{ commentItem.part || '파트 정보 없음' }}, {{ commentItem.rankingName || '직급 정보 없음' }})
        </p>
        <p class="comment-content">{{ commentItem.commentContent }}</p>
        <p class="comment-date">
          <LocalDateTimeFormat :data="commentItem.createdAt" />
        </p>
        <div class="comment-actions" v-if="commentItem.userSeq === loggedInUserSeq">
          <!-- 수정 버튼 -->
          <button @click="editComment(commentItem)" class="action-btn">수정</button>
          <!-- 삭제 버튼 -->
          <button @click="deleteComment(commentItem.commentSeq)" class="action-btn">삭제</button>
        </div>
      </div>

      <!-- 댓글 목록 페이지네이션 -->
      <Pagination
          :totalData="totalComment"
          :limitPage="commentItemCount"
          :page="currentCommentPage"
          @updatePage="changePage"
      />
    </div>

    <!-- 댓글 입력 -->
    <div class="comment-input">
      <span class="char-count">{{ newCommentContent.length }} / 1000</span>
      <textarea
          v-model="newCommentContent"
          placeholder="댓글을 입력하세요."
          maxlength="1000"
          @input="updateCharacterCount"
      ></textarea>
      <button @click="saveComment">
        {{ currentEditingCommentSeq ? '수정 완료' : '댓글 달기' }}
      </button>
    </div>
  </div>
</template>

<style scoped>
.board-detail {
  margin: 40px;
  padding: 30px;
  background-color: #fff;
  border-radius: 8px;
  border: 1px solid black;
}

.board-title {
  font-size: 28px;
  font-weight: bold;
  margin-bottom: 10px;
  padding-bottom: 5px;
}

.board-info {
  display: flex;
  align-items: center;
  font-size: 14px;
  color: #666;
  margin-bottom: 20px;
}

.author-info {
  display: flex;
  align-items: center;
  margin-right: 20px;
}

.profile-pic {
  width: 50px;
  height: 50px;
  border-radius: 50%;
  margin-right: 100px;
}

.author,
.date,
.view-count {
  font-size: 18px
}

.author {
  font-weight: bold;
  margin-top: 15px;
  margin-right: 100px;
}

.date {
  margin-top: 15px;
  margin-right: 100px;
}

.view-count {
  margin-top: 15px;
}

.actions {
  margin-left: auto;
  display: flex;
  align-items: center;
}

.like-btn,
.bookmark-btn {
  cursor: pointer;
  margin-left: 10px;
}

.like-btn img,
.bookmark-btn img {
  width: 25px;
}

.keyword-detail {
  border-radius: 8px;
  border: 1px solid black;
  padding: 20px;
}

.keyword-detail {
  font-size: 25px;
  margin-top: 30px;
  margin-bottom: 30px;
  font-weight: bold;
}

.keyword-title,
.comment-title {
  font-size: 25px;
  font-weight: bold;
}

.comment-title {
  margin: 0;
}

.comment-count {
  font-weight: bold;
}

.keyword-list {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.keyword-item {
  background-color: #fff0d3;
  padding: 8px 12px;
  font-size: 20px;
}

.content {
  margin-top: 25px;
  margin-bottom: 20px;
  font-size: 20px
}

.picture-list img {
  max-width: 100%;
  height: auto;
  margin: 10px 0;
  font-size: 20px
}

.comment-section {
  border-radius: 8px;
  border: 1px solid black;
  padding: 20px;
}

.comment-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.comment {
  padding: 10px;
  border-radius: 4px;
  margin-bottom: 10px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.comment-content {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.comment-author,
.comment-text,
.comment-date {
  font-size: 18px;
}

.comment-author {
  font-weight: bold;
  margin-right: 10px;
}

.comment-text {
  font-weight: bold;
}

.comment-date {
  font-weight: bold;
  margin-left: 10px;
}

.comment-input {
  margin-top: 30px;
  position: relative;
  width: 100%;
}

.comment-input textarea {
  width: 100%;
  height: 200px;
  border: 1px solid black;
  border-radius: 4px;
  padding: 10px;
  resize: none;
}

.char-count {
  position: absolute;
  top: 10px;
  right: 23px;
  font-size: 15px;
  color: #666;
}

.comment-input button {
  position: absolute;
  bottom: 10px;
  right: 10px;
  background-color: white;
  color: #003366;
  border: 1px solid #003366;
  border-radius: 7px;
  cursor: pointer;
  font-weight: bold;
  width: 90px;
  height: 30px;
}

.comment-actions {
  display: flex;
  gap: 10px;
}

.action-btn {
  background: none;
  border: none;
  color: #007bff;
  cursor: pointer;
  font-size: 14px;
  text-decoration: underline;
}

.action-btn:hover {
  color: #0056b3;
}
</style>