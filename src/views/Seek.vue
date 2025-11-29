<template>
  <Chat
    :onMessageSend="sendMessage"
    :chats="chats"
    :roleConfig="roleInfo"
    ref="chatRef"
    style="margin: 0 auto"
  ></Chat>
</template>
<script setup>
import { ref, useTemplateRef ,nextTick} from 'vue';
import { Chat } from '@kousum/semi-ui-vue';

import logo from '@/assets/deepseek.png'
const chatRef = useTemplateRef('chatRef');

let chats = ref([]);
const roleInfo = {
  user: {
    name: 'Happyfe',
    avatar:
      'https://happyfe.com/logo.png',
  },
  assistant: {
    name: 'DeepSeek',
    avatar: logo,
  },
  system: {
    name: 'System',
    avatar:
      'https://lf3-static.bytednsdoc.com/obj/eden-cn/ptlz_zlp/ljhwZthlaukjlkulzlp/other/logo.png',
  },
};
const defaultMessage = [
  {
    role: 'system',
    id: '1',
    createAt: 1715676751919,
    content: "Hello, I'm your AI assistant.",
  },
  {
    role: 'user',
    id: '2',
    createAt: 1715676751919,
    content: '给一个 Semi Design 的 Button 组件的使用示例',
  },
  {
    role: 'assistant',
    id: '3',
    createAt: 1715676751919,
    content:
      "以下是一个 Semi 代码的使用示例：\n\`\`\`jsx \nimport React from 'react';\nimport { Button } from '@kousum/semi-ui-vue';\n\nconst MyComponent = () => {\n  return (\n    <Button>Click me</Button>\n );\n};\nexport default MyComponent;\n\`\`\`\n",
  },
];
let count = 0;
const sendMessage = (message) => {
  chats.value.push({
    role: 'user',
    id: count++,
    createAt: new Date().getTime(),
    content: message,
  });
  console.log(message);
  chats.value.push({
    role: 'assistant',
    id: count++,
    createAt: new Date().getTime(),
    content: '',
    status: 'loading',
  });
  const eventSource = new EventSource(`/api/stream?message=${message}`);
  eventSource.onmessage = (event) => {
    let response = event.data;
    console.log(response);
    if (response == 'end') {
      chats.value[chats.value.length - 1].status = 'complete';
      close();
      return;
    }

    response = JSON.parse(response).content;
    chats.value[chats.value.length - 1].status = 'incomplete';
    chats.value[chats.value.length - 1].content += response;
    console.log(chatRef.value.scrollToBottom);
    // nextTick(() => {
    //   // var element = document.getElementsByClassName('semi-chat')[0];
    //   // element.scrollTop = element.scrollHeight;
    //   // chatRef.value.scrollToBottom(true);
    //   // chatRef.value.scrollTop = chatRef.value.scrollHeight;
    // })
    chatRef.value.scrollToBottom(true);
  };

  eventSource.onerror = (error) => {
    chats.value[chats.value.length - 1].status = 'error';
    close();
  };

  const close = () => {
    eventSource.close();
  };
};
</script>
<style lang="scss" scoped></style>
