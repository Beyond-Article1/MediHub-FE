<script setup>
import { ref, nextTick, watch } from "vue";

// 플러그인 임포트
import EditorJS from "@editorjs/editorjs";
import Header from "@editorjs/header";
import List from "@editorjs/list";
import Checklist from "@editorjs/checklist";
import Table from "@editorjs/table";
import CodeTool from "@editorjs/code";
import Marker from "@editorjs/marker";
import Quote from "@editorjs/quote";
import Embed from "@editorjs/embed";
import InlineCode from "@editorjs/inline-code";
import Delimiter from "@editorjs/delimiter";
import Paragraph from "@editorjs/paragraph";
import ImageTool from "@editorjs/image";
import Underline from "@editorjs/underline";
import ColorPlugin from "editorjs-text-color-plugin";

const props = defineProps({

  initialData: {

    type: Object,
    // 초기 데이터
    default: () => ({ blocks: [] }) }
});

let editorInstance = null;
let checkOnce = true;

// 이미지 파일 저장
const images = ref([]);
const imageInput = ref(null);

const initializeEditor = async (data) => {

  if(editorInstance && typeof editorInstance.destroy === "function") {

    // 안전하게 destroy 호출
    editorInstance.destroy();

    editorInstance = null;
  }

  const editorData = data && data.blocks && data.blocks.length > 0 ? data : { blocks: [] };

  await nextTick();

  editorInstance = new EditorJS({

    holder: "editor",
    // 데이터가 있으면 placeholder 제거
    placeholder: editorData.blocks.length > 0 ? "" : "내용을 입력하세요.",
    data: data || { blocks: [] },
    tools: {

      header: { class: Header, config: { levels: [2, 3, 4], defaultLevel: 2 } },
      paragraph: { class: Paragraph, inlineToolbar: true },
      list: { class: List, inlineToolbar: true },
      checklist: { class: Checklist, inlineToolbar: true },
      table: Table,
      code: CodeTool,
      quote: Quote,
      embed: Embed,
      inlineCode: InlineCode,
      delimiter: Delimiter,
      underline: Underline,
      marker: Marker,
      textColor: {

        class: ColorPlugin,
        config: {

          colorCollections: ["#FF5733", "#33FF57", "#3357FF", "#000000"],
          defaultColor: "#FF5733",
          customPicker: true
        },
      },
      image: {

        class: ImageTool,
        config: {

          uploader: {

            uploadByFile: async (file) => {

              const blobUrl = URL.createObjectURL(file);

              images.value.push(file);

              monitorImageLoad(blobUrl);

              return { success: 1, file: { url: blobUrl } };
            },
          },
        },
      },
    },
  });

  checkOnce = false;
};

watch(
    () => props.initialData,
    (newData) => { if (newData) initializeEditor(newData); },
    { immediate: true,  deep: true }
);

// 툴바 기능 메서드
const insertHeader = (level) => editorInstance.blocks.insert("header", { level });
const insertParagraph = () => editorInstance.blocks.insert("paragraph", {});
const insertQuote = () => editorInstance.blocks.insert("quote", {});
const insertWarning = () => editorInstance.blocks.insert(
    "warning",
    { anonymousBoardTitle: "주의", message: "경고 내용" }
);
const insertDelimiter = () => editorInstance.blocks.insert("delimiter");
const insertUnorderedList = () => editorInstance.blocks.insert("list", { style: "unordered" });
const insertOrderedList = () => editorInstance.blocks.insert("list", { style: "ordered" });
const insertChecklist = () => editorInstance.blocks.insert("checklist", {});
const insertTable = () => editorInstance.blocks.insert("table", { anonymousBoardContent: [[""]] });
const insertCode = () => editorInstance.blocks.insert("code", { code: "" });
const insertInlineCode = () => editorInstance.inlineToolbar.activate("inlineCode");
const insertEmbed = () => editorInstance.blocks.insert("embed", {});
// 이미지 업로드 버튼 클릭
const triggerImageUpload = () => imageInput.value.click();

const handleImageUpload = (event) => {

  const file = event.target.files[0];

  if(!file) return;

  const reader = new FileReader();

  reader.onload = (e) => insertImage(e.target.result);
  reader.readAsDataURL(file);
};

const insertImage = (url) => editorInstance.blocks.insert("image", { file: { url } });
// 텍스트 정렬 및 색상
const setTextAlignment = (alignment) => editorInstance.blocks.update(
    editorInstance.blocks.getCurrentBlockIndex(),
    { alignment }
);
const changeTextColor = () => editorInstance.inlineToolbar.activate("textColor");
// 인라인 서식 툴
const applyInlineTool = (tool) => editorInstance.inlineToolbar.activate(tool);

const monitorImageLoad = (url) => {

  const img = new Image();

  img.src = url;

  // 이미지 로드 성공
  img.onload = () => { console.log("이미지 로드 성공:", url); };
  // 이미지 로드 실패
  img.onerror = () => {

    console.error("이미지 로드 실패:", url);

    alert("이미지 로드에 실패하였습니다. 해당 블록이 삭제됩니다.");

    // 실패한 이미지 블록 삭제
    removeImageBlock(url);
  };
  // DOM에 추가하지 않고 로드 상태만 확인
};

const removeImageBlock = (url) => {

  const blocksCount = editorInstance.blocks.getBlocksCount();

  for(let i = 0; i < blocksCount; i++) {

    const block = editorInstance.blocks.getBlockByIndex(i);

    if(block && block.type === "image" && block.data.file.url === url) {

      // 이미지 블록 삭제
      editorInstance.blocks.delete(i);

      break;
    }
  }
};

const getEditorData = async () => {

  if(!editorInstance) return { anonymousBoardContent: { blocks: [] }, images: [] };

  try {

    const savedData = await editorInstance.save();

    return { anonymousBoardContent: savedData, images: images.value };
  } catch(error) {

    return { anonymousBoardContent: { blocks: [] }, images: [] };
  }
};

defineExpose({

  // 부모 컴포넌트에서 호출할 수 있도록 노출
  getEditorData,
  initializeEditor
});
</script>

<template>
  <div class="board-editor">
    <div class="toolbar">
      <button @click="insertLink" title="링크"><i class="bi bi-link-45deg"></i></button>
      <button @click="insertParagraph" title="본문"><i class="bi bi-fonts"></i></button>
      <button @click="insertHeader(2)" title="제목 2"><i class="bi bi-type-h2"></i></button>
      <button @click="insertHeader(3)" title="제목 3"><i class="bi bi-type-h3"></i></button>
      <button @click="insertHeader(4)" title="제목 4"><i class="bi bi-type-h4"></i></button>
      <button @click="insertQuote" title="인용"><i class="bi bi-quote"></i></button>
      <button @click="insertWarning" title="경고"><i class="bi bi-exclamation-circle"></i></button>
      <button @click="insertDelimiter" title="구분선"><i class="bi bi-dash-lg"></i></button>
      <button @click="insertUnorderedList" title="순서 없는 리스트"><i class="bi bi-list-ul"></i></button>
      <button @click="insertOrderedList" title="순서 있는 리스트"><i class="bi bi-list-ol"></i></button>
      <button @click="insertChecklist" title="체크리스트"><i class="bi bi-check2-square"></i></button>
      <button @click="insertTable" title="표"><i class="bi bi-table"></i></button>
      <button @click="insertCode" title="코드"><i class="bi bi-code-slash"></i></button>
      <button @click="insertInlineCode" title="인라인 코드"><i class="bi bi-code"></i></button>
      <button @click="triggerImageUpload" title="이미지"><i class="bi bi-image"></i></button>
      <button @click="insertEmbed" title="Embed"><i class="bi bi-camera-video"></i></button>
      <button @click="applyInlineTool('bold')" title="굵게"><i class="bi bi-type-bold"></i></button>
      <button @click="applyInlineTool('italic')" title="기울임"><i class="bi bi-type-italic"></i></button>
      <button @click="applyInlineTool('underline')" title="밑줄"><i class="bi bi-type-underline"></i></button>
      <button @click="applyInlineTool('marker')" title="마커"><i class="bi bi-highlighter"></i></button>
      <button @click="changeTextColor" title="텍스트 색상"><i class="bi bi-palette"></i></button>
      <button @click="setTextAlignment('left')" title="왼쪽 정렬"><i class="bi bi-text-left"></i></button>
      <button @click="setTextAlignment('center')" title="가운데 정렬"><i class="bi bi-text-center"></i></button>
      <button @click="setTextAlignment('right')" title="오른쪽 정렬"><i class="bi bi-text-right"></i></button>
    </div>

    <!-- Editor.js 에디터 -->
    <div id="editor"></div>
    <input type="file" ref="imageInput" @change="handleImageUpload" style="display: none" />
  </div>
</template>

<style scoped>
@import "bootstrap-icons/font/bootstrap-icons.css";
:deep(.ce-toolbar__actions) {
  left: 0 !important; /* 왼쪽에 고정 */
  right: auto !important; /* 오른쪽 위치 초기화 */
  transform: none !important; /* 이동 보정 */
  position: absolute !important; /* 위치 제어 */
  margin-left: -70px; /* 좌측 여백 보정 */
}

:deep(.ce-toolbar__plus) {
  left: -70px !important; /* + 버튼 왼쪽으로 이동 */
  right: auto !important;
  transform: none !important;
}

:deep(.ce-block__content) {
  max-width: 1000px !important; /* 최대 폭을 1000px로 고정 */
  margin: 0 auto; /* 내용 중앙 정렬 */
}

:deep(.ce-toolbar__content){
  max-width: 1000px !important; /* 최대 폭을 1000px로 고정 */
}

:deep(.codex-editor__redactor) {
  padding: 0 !important; /* redactor 요소 스타일 제거 */
  margin: 0 !important; /* redactor 요소 스타일 제거 */
}

.board-editor {
  width: 100%; /* 전체 폭을 차지하도록 설정 */
  max-width: 1400px; /* 원하는 최대 너비 설정 */
  margin: 0 auto; /* 가운데 정렬 */
  padding: 20px;
  box-sizing: border-box;
  background-color: #fff;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

#editor {
  width: 100%;
  height: 650px; /* 최대 높이 설정 */
  overflow-y: auto; /* 세로 스크롤 추가 */
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 10px;
  box-sizing: border-box;
}

.toolbar {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  padding: 10px;
  background-color: #f9f9f9;
  border-bottom: 1px solid #ddd;
}

.toolbar button {
  background-color: #fff;
  border: 1px solid #ccc;
  padding: 6px 10px;
  border-radius: 4px;
  cursor: pointer;
}

.toolbar button:hover {
  background-color: #f0f0f0;
}

.toolbar button i {
  font-size: 14px;
}
</style>