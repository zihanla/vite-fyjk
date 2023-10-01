<template>
  <div class="container mx-auto px-4 py-6">
    <div class="flex items-center justify-between">
      <textarea
        v-model="phoneNumbers"
        class="border-2 p-2 rounded-md w-full resize-none"
        rows="5"
        placeholder="请输入手机号，每行一个"
      ></textarea>
      <button
        @click="fetchData"
        :disabled="!(phoneNumbers && phoneNumbers.trim().length)"
        class="btn btn-primary ml-4"
      >
        查询
      </button>
    </div>

    <div v-if="responseData" class="mt-6 flex items-center justify-start">
      <div v-if="responseData.qrCodeAsPng">
        <img :src="`data:image/png;base64,${responseData.qrCodeAsPng}`" class="w-52 h-52 mr-2" />
      </div>

      <div>
        <div class="text-lg font-bold mb-2">总金额: {{ totalAmount }}</div>
        <div class="text-sm text-gray-500">{{ formatDateTime(responseData.dateTime) }}</div>
      </div>
    </div>

    <table v-if="responseData && responseData.data.length > 0" class="min-w-full table mt-6">
      <thead>
        <tr>
          <th class="border px-4 py-2">创建日期</th>
          <th class="border px-4 py-2">用户ID</th>
          <th class="border px-4 py-2">数量</th>
          <th class="border px-4 py-2">金额</th>
          <th class="border px-4 py-2">状态</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="record in responseData.data" :key="record.user_id">
          <td class="border px-4 py-2">{{ formatDate(record.created) }}</td>
          <td class="border px-4 py-2">{{ record.user_id }}</td>
          <td class="border px-4 py-2">{{ record.number }}</td>
          <td class="border px-4 py-2">{{ (record.number * conversionFactor).toFixed(2) }}</td>
          <td class="border px-4 py-2">{{ record.status === 1 ? '已支付' : '未支付' }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';
import { format } from 'date-fns';

// Constants
const API_URL = "https://agent.datamonitor.shop/api/v1/ZhangDan";
const HEADERS = {
  "accept": "text/plain",
  "agent_id": "158633",
  "content-type": "application/json",
  "Referer": "https://agent.datamonitor.shop/?agent_id=158633",
  "Referrer-Policy": "strict-origin-when-cross-origin"
};

let phoneNumbers = ref("");
let responseData = ref(null);
let conversionFactor = ref(0.012);

const totalAmount = computed(() => {
  if (!responseData.value || !responseData.value.data) return '0.00';
  const unpaidAmount = responseData.value.data
    .filter(record => record.status === 0)
    .reduce((sum, record) => sum + record.number, 0);
  return (unpaidAmount * conversionFactor.value).toFixed(2);
});

async function fetchData() {
  // 验证手机号码格式
  const phoneList = phoneNumbers.value.split('\n');
  const phoneRegex = /^1[3-9]\d{9}$/;
  const invalidPhones = [];
  for (let i = 0; i < phoneList.length; i++) {
    if (!phoneRegex.test(phoneList[i])) {
      invalidPhones.push(phoneList[i]);
    }
  }

  if (invalidPhones.length) {
    console.warn("以下手机号码格式不正确：", invalidPhones.join(', '));
    return;
  }

  try {
    const response = await fetch(API_URL, {
      headers: HEADERS,
      body: JSON.stringify({
        phone_list: phoneList
      }),
      method: "POST"
    });

    if (!response.ok) {
      const errorData = await response.json();
      console.error("Error from server:", errorData);
      // 你可以在此处展示友好的错误消息
      throw new Error(`Server responded with status: ${response.status}`);
    }

    responseData.value = await response.json();
    phoneNumbers.value = '';  // 清除电话号码以获取数据

  } catch (error) {
    console.error("Failed to fetch data:", error);
    // 展示友好的错误消息
  }
}

function formatDate(dateStr) {
  return format(new Date(dateStr), 'MM月dd日');
}

function formatDateTime(dateTimeStr) {
  return format(new Date(dateTimeStr), 'MM月dd日 HH:mm');
}
</script>

<style scoped>
.container {
  max-width: 1200px;
}

table {
  width: 100%;
  border-collapse: collapse;
}

th, td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

th {
  background-color: #f2f2f2;
}

tr:hover {
  background-color: #f5f5f5;
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}
</style>
