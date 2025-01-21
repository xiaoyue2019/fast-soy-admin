<script setup>
import { ref, onMounted } from "vue";

// 数据相关
const coins = ref([]);
const currentPage = ref(1);
const totalPages = ref(0);
const loading = ref(false);
const sortKey = ref("rank");
const sortOrder = ref("asc"); // asc: 升序, desc: 降序

// 加载数据的函数
const fetchData = async (page = 1) => {
  loading.value = true;
  try {
    const res = await fetch(
      `https://dncapi.flink1.com/api/coin/web-coinrank?page=${page}&type=-1&pagesize=100&webp=1`
    );
    const data = await res.json();
    coins.value = data.data;
    totalPages.value = data.maxpage;
    currentPage.value = data.currpage;
  } catch (error) {
    console.error("数据加载失败", error);
  } finally {
    loading.value = false;
  }
};

// 排序函数
const sortData = (key) => {
  if (sortKey.value === key) {
    sortOrder.value = sortOrder.value === "asc" ? "desc" : "asc";
  } else {
    sortKey.value = key;
    sortOrder.value = "asc";
  }

  coins.value.sort((a, b) => {
    if (sortOrder.value === "asc") {
      return a[key] > b[key] ? 1 : -1;
    } else {
      return a[key] < b[key] ? 1 : -1;
    }
  });
};

// 分页切换
const changePage = (page) => {
  if (page > 0 && page <= totalPages.value) {
    fetchData(page);
  }
};

// 初始加载
onMounted(() => {
  fetchData(currentPage.value);
});
</script>

<template>
  <div class="container">
    <h1>加密货币排行榜</h1>
    <div v-if="loading" class="loading">加载中...</div>
    <table v-else class="coin-table">
      <thead>
        <tr>
          <th @click="sortData('rank')">排名</th>
          <th>图标</th>
          <th @click="sortData('name')">名称</th>
          <th @click="sortData('current_price')">价格 (¥)</th>
          <th @click="sortData('current_price_usd')">价格 ($)</th>
          <th @click="sortData('change_percent')">涨跌幅 (%)</th>
          <th @click="sortData('market_value')">市值 (¥)</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="coin in coins" :key="coin.code">
          <td>{{ coin.rank }}</td>
          <td><img :src="coin.logo" :alt="coin.name" class="coin-logo" /></td>
          <td>{{ coin.fullname }} ({{ coin.name }})</td>
          <td>{{ coin.current_price.toLocaleString() }}</td>
          <td>{{ coin.current_price_usd.toLocaleString() }}</td>
          <td :class="{ negative: coin.change_percent < 0, positive: coin.change_percent > 0 }">
            {{ coin.change_percent }}%
          </td>
          <td>{{ coin.market_value.toLocaleString() }}</td>
        </tr>
      </tbody>
    </table>

    <!-- 分页控件 -->
    <div class="pagination">
      <button :disabled="currentPage === 1" @click="changePage(currentPage - 1)">上一页</button>
      <span>第 {{ currentPage }} 页 / 共 {{ totalPages }} 页</span>
      <button :disabled="currentPage === totalPages" @click="changePage(currentPage + 1)">下一页</button>
    </div>
  </div>
</template>

<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
}

h1 {
  text-align: center;
  margin-bottom: 20px;
}

.loading {
  text-align: center;
  font-size: 18px;
  color: #555;
}

.coin-table {
  width: 100%;
  border-collapse: collapse;
  margin-bottom: 20px;
}

.coin-table th,
.coin-table td {
  padding: 10px;
  text-align: left;
  border: 1px solid #ddd;
}

.coin-table th {
  background-color: #f4f4f4;
  cursor: pointer;
}

.coin-table th:hover {
  background-color: #eaeaea;
}

.coin-table .coin-logo {
  width: 24px;
  height: 24px;
}

.coin-table .positive {
  color: green;
}

.coin-table .negative {
  color: red;
}

.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
}

.pagination button {
  margin: 0 10px;
  padding: 5px 15px;
  border: none;
  background-color: #007bff;
  color: white;
  cursor: pointer;
}

.pagination button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.pagination span {
  font-size: 16px;
}
</style>