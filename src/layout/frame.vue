<script setup lang="ts">
import { useI18n } from "vue-i18n";
import { useRoute } from "vue-router";
import { ref, unref, watch, onMounted, nextTick } from "vue";

defineOptions({
  name: "LayFrame"
});

const props = defineProps<{
  frameInfo?: {
    frameSrc?: string;
    fullPath?: string;
  };
}>();

const { t } = useI18n();
const loading = ref(true);
const currentRoute = useRoute();
const frameSrc = ref<string>("");
const frameRef = ref<HTMLElement | null>(null);
if (unref(currentRoute.meta)?.frameSrc) {
  frameSrc.value = unref(currentRoute.meta)?.frameSrc as string;
}
unref(currentRoute.meta)?.frameLoading === false && hideLoading();

function hideLoading() {
  loading.value = false;
}

function init() {
  nextTick(() => {
    const iframe = unref(frameRef);
    if (!iframe) return;
    const _frame = iframe as any;
    if (_frame.attachEvent) {
      _frame.attachEvent("onload", () => {
        hideLoading();
      });
    } else {
      iframe.onload = () => {
        hideLoading();
      };
    }
  });
}

let isRedirect = false;
watch(
  () => currentRoute.fullPath,
  path => {
    if (
      currentRoute.name === "Redirect" &&
      path.includes(props.frameInfo?.fullPath)
    ) {
      // 不采用更改iframe的src的方式，其会导致两次src的改变，当iframe同源且注册了beforeunload事件时，会导致beforeunload事件被触发两次
      // frameSrc.value = path; // redirect时，置换成任意值，待重定向后 重新赋值
      isRedirect = true;
      loading.value = true;
    }
    // 重新赋值
    if (props.frameInfo?.fullPath === path) {
      frameSrc.value = props.frameInfo?.frameSrc;
      if (isRedirect) {
        let joinChar = new URL(props.frameInfo.frameSrc)?.search ? "&" : "?";
        frameSrc.value =
          props.frameInfo.frameSrc + `${joinChar}t=` + Date.now();
        // 一旦点击“重新加载”，就会触发“加载中”，此处需隐藏加载中
        // 因无法得知用户是点击了“确认离开页面”还是“不离开页面”，为了保证不离开情况下不显示加载中，此处统一隐藏
        hideLoading();
      }
      isRedirect = false;
    }
  }
);

onMounted(() => {
  init();
});
</script>

<template>
  <div
    v-loading="loading"
    class="frame"
    :element-loading-text="t('status.pureLoad')"
  >
    <iframe ref="frameRef" :src="frameSrc" class="frame-iframe" />
  </div>
</template>

<style lang="scss" scoped>
.frame {
  position: absolute;
  inset: 0;

  .frame-iframe {
    box-sizing: border-box;
    width: 100%;
    height: 100%;
    overflow: hidden;
    border: 0;
  }
}

.main-content {
  margin: 2px 0 0 !important;
}
</style>
