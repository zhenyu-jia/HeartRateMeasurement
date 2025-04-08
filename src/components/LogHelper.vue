<script>
import { ref, provide } from 'vue';

// 创建一个全局日志方法
const log = ref(''); // 用于存储日志内容

// 全局日志函数，支持多个参数
function logMessage(...args) {
  const message = args.map(arg => (typeof arg === 'object' ? JSON.stringify(arg) : String(arg))).join(' ');
  log.value += message + '\n'; // 将格式化后的日志追加到 log 中
}

// 提供全局日志方法
export function useLog() {
  return logMessage;
}

export default {
  setup() {
    // 使用 Vue 的 provide/inject 机制，将 logMessage 提供给其他组件
    provide('logMessage', logMessage);

    return {
      log
    };
  }
};
</script>

<template>
  <div>
    <!-- 添加一个 textarea，用于显示日志 -->
    <textarea v-model="log" rows="10" cols="50" readonly></textarea>
  </div>
</template>

<style scoped>
textarea {
  width: 100%;
  resize: none;
}
</style>