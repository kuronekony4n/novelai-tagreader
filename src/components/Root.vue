<template>
  <div class="text-center p-6 max-w-1024px mx-auto">
    <h1 class="font-bold text-3xl">NovelAI 法术解析</h1>
    <p class="text-gray-500 my-2 text-sm">
      从 NovelAI 生成的图片读取内嵌的 prompt
    </p>
    <el-upload
      class="upload-demo"
      accept="image/*"
      drag
      multiple
      :before-upload="handleUpload"
    >
      <el-icon class="el-icon--upload"><upload-filled /></el-icon>
      <div class="el-upload__text">拖动文件到这里或者点击上传</div>
    </el-upload>
    <div v-if="fileRef" class="my-6">
      <div
        class="bg-white max-w-720px mx-auto border border-gray-300 p-2"
        v-if="imageRef"
      >
        <img
          v-if="imageRef"
          v-bind="imageRef"
          alt=""
          style="display: block; width: 100%; height: auto"
        />
      </div>

      <div v-if="fileInfoRef" class="mt-4 text-left max-w-740px mx-auto">
        <h1 class="font-bold text-2xl mb-4">图片信息</h1>
        <div
          :class="[index === 0 && 'border-t border-t-gray-300']"
          class="bg-white border-b border-l border-r px-4 border-b-gray-300 border-l-gray-300 border-r-gray-300 py-2"
          v-for="(item, index) in fileInfoRef"
          :key="item.key"
        >
          <h1 class="font-semibold text-sm text-gray-800">{{ item.key }}</h1>
          <p class="text-wrap break-all text-sm mt-1 text-gray-600">
            {{ item.value }}
          </p>
          <json-viewer
            :value="jsonData"
            v-if="item.key == 'Comment'"
          ></json-viewer>
        </div>
      </div>

      <div v-if="exifRef" class="mt-4 text-left max-w-740px mx-auto">
        <h1 class="font-bold text-2xl mb-4">EXIF</h1>
        <div
          :class="[index === 0 && 'border-t border-t-gray-300']"
          class="bg-white border-b border-l border-r px-4 border-b-gray-300 border-l-gray-300 border-r-gray-300 py-2"
          v-for="(item, index) in exifRef"
          :key="item.key"
        >
          <h1 class="font-semibold text-sm text-gray-800">{{ item.key }}</h1>
          <p class="text-wrap break-all text-sm mt-1 text-gray-600">
            {{ item.value.description }}
          </p>
        </div>
      </div>
    </div>
    <p class="text-gray-500 my-2 text-sm">
      *运算完全在你的电脑上运行不会上传图片到云端！
    </p>
    <div class="my-4 pt-4">
      <a
        class="inline-block text-sm text-gray-500"
        href="https://github.com/Akegarasu/novelai-tagreader"
        >View on GitHub</a
      >
      <br />
      <span class="inline-block mt-2 text-sm text-gray-500">
        Made with ❤️ by
        <a class="text-gray-500" href="https://github.com/Akegarasu"
          >@Akegarasu</a
        >
      </span>
    </div>
  </div>
</template>

<script setup>
import ExifReader from "exifreader";
import { computed, ref, watch } from "vue";
import prettyBytes from "pretty-bytes";
import extractChunks from "png-chunks-extract";
import text from "png-chunk-text";
import jsonViewer from "vue-json-viewer";
import { UploadFilled } from "@element-plus/icons-vue";

const fileRef = ref(null);
const imageRef = ref(null);
const exifRef = ref(null);
const fileInfoRef = ref(null);
const jsonData = ref(null);
const imageMaxSizeRef = ref(0);

watch(fileRef, () => {
  if (!fileRef.value) return;
  readImageBase64();
  readExif();
  readFileInfo();
});

async function handleUpload(file) {
  console.log(file);
  fileRef.value = file;
  return false;
}

async function readNovelaiTag(file) {
  const buf = await file.arrayBuffer();
  const chunks = extractChunks(new Uint8Array(buf));
  const textChunks = chunks
    .filter(function (chunk) {
      return chunk.name === "tEXt" || chunk.name === "iTXt";
    })
    .map(function (chunk) {
      if (chunk.name === "iTXt") {
        let data = chunk.data.filter((x) => x != 0x0);
        let txt = new TextDecoder().decode(data);
        return {
          keyword: "信息",
          text: txt.slice(11),
        };
      }
      return text.decode(chunk.data);
    });
  console.log(textChunks);
  return textChunks;
}

async function readFileInfo() {
  const file = fileRef.value;
  let nai = await readNovelaiTag(file);
  if (nai.length == 1) {
    nai = await handleWebUiTag(nai[0]);
  }
  fileInfoRef.value = [
    { key: "文件名", value: file.name },
    { key: "文件大小", value: prettyBytes(file.size) },
    ...nai.map((v, k) => {
      if (v.keyword == "Comment") {
        jsonData.value = JSON.parse(v.text);
      }
      return {
        key: v.keyword,
        value: v.text,
      };
    }),
  ];
  if (nai.length == 0) {
    fileInfoRef.value.push({
      key: "提示",
      value: "这可能不是一张NovelAI生成的图或者不是原图, 经过了压缩",
    });
  }
}

async function handleWebUiTag(data) {
  let promptSplit = data.text.split("Negative prompt: ");
  let otherSplit = promptSplit[1].split("Steps: ");
  return [
    {
      keyword: "提示词",
      text: promptSplit[0],
    },
    {
      keyword: "负面提示词",
      text: otherSplit[0],
    },
    {
      keyword: "其他参数",
      text: "Steps: " + otherSplit[1],
    },
  ];
}

function readImageBase64() {
  imageRef.value = null;
  const reader = new FileReader();
  reader.onload = () => {
    const image = new Image();
    image.onload = () => {
      const { width, height } = image;
      imageRef.value = {
        width,
        height,
        src: reader.result,
      };

      imageMaxSizeRef.value = width;
    };
    image.src = reader.result;
  };
  reader.readAsDataURL(fileRef.value);
}

async function readExif() {
  const file = fileRef.value;
  const data = await ExifReader.load(file);
  const entries = Object.entries(data);
  exifRef.value = entries.map(([key, value]) => ({ key, value }));
}
</script>
