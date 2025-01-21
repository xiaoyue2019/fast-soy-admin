<script setup>
import { ref, onMounted } from "vue";
import { useRouter } from "vue-router";
import axios from "axios";
import dayjs from "dayjs";

// 搜索相关状态
const searchTerm = ref(""); // 用户输入的搜索关键词
const coins = ref([]); // 币种列表
const news = ref([]); // 新闻列表
const loading = ref(false); // 是否正在加载数据
const activeSection = ref("coins"); // 当前展示的部分，默认为 "coins"
const hoveredRow = ref(null); // 当前悬停的行

const router = useRouter();

// 搜索函数
const search = async () => {
  if (!searchTerm.value.trim()) {
    alert("请输入搜索关键词！");
    return;
  }

  loading.value = true;

  try {
    // 并行请求两个接口
    const [coinsResponse, newsResponse] = await Promise.all([
      axios.get(
        `https://dncapi.flink1.com/api/search/websearch?page=1&exchange_page=1&wallet_page=1&pagesize=25&code=${searchTerm.value}&token=&webp=1`
      ),
      axios.get(
        `https://dncapi.flink1.com/api/search/news?filter=${searchTerm.value}&per_page=50&channelid=1&page=1&webp=1`
      ),
    ]);

    // 处理币种数据
    coins.value = coinsResponse.data.coinlist || [];

    // 处理新闻数据
    news.value = newsResponse.data.data.list || [];
  } catch (error) {
    console.error("搜索失败：", error);
    alert("搜索失败，请稍后重试！");
  } finally {
    loading.value = false;
  }
};

// 跳转到币种详情页
const goToCoinPage = (coin) => {
  router.push(`/singlecoin/${coin}`); // 跳转到 '/singlecoin/:coinName' 路由，传递 coin.coincode 参数
};

// 初始化页面
onMounted(() => {
  // 初次加载页面时，可以展示热门数据或留空
  searchTerm.value = "trump"; // 示例默认关键词
  search();
});
</script>

<template>
  <div class="search-page">
    <!-- 搜索框 -->
    <div class="search-header">
      <input
        v-model="searchTerm"
        type="text"
        class="search-input"
        placeholder="请输入关键词搜索币种或新闻..."
      />
      <button @click="search" class="search-button">搜索</button>
    </div>

    <!-- 加载中提示 -->
    <div v-if="loading" class="loading">正在加载数据...</div>

    <!-- 搜索结果 -->
    <div v-else>
      <div class="results-header">
        <h2>搜索结果</h2>
        <div class="toggle-buttons">
          <button
            :class="{ active: activeSection === 'coins' }"
            @click="activeSection = 'coins'"
          >
            币种
          </button>
          <button
            :class="{ active: activeSection === 'news' }"
            @click="activeSection = 'news'"
          >
            新闻
          </button>
        </div>
      </div>

      <!-- 币种部分 -->
      <div v-show="activeSection === 'coins'" class="coins-section">
        <h3>币种结果</h3>
        <table class="coins-table">
          <thead>
            <tr>
              <th>排名</th>
              <th>图标</th>
              <th>名称</th>
              <th>价格 (¥)</th>
              <th>涨跌幅 (%)</th>
              <th>市值 (¥)</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="coin in coins"
              :key="coin.coincode"
              @mouseover="hoveredRow = coin.coincode"
              @mouseleave="hoveredRow = null"
              :class="{ hovered: hoveredRow === coin.coincode }"
              @click="goToCoinPage(coin.coincode)"
            >
              <td>{{ coin.rank_no }}</td>
              <td><img :src="coin.coinlogo" :alt="coin.coinname" /></td>
              <td>{{ coin.coinname }}</td>
              <td>{{ coin.price.toLocaleString() }}</td>
              <td
                :class="{ positive: coin.change_percent > 0, negative: coin.change_percent < 0 }"
              >
                {{ coin.change_percent }}%
              </td>
              <td>{{ coin.market_value.toLocaleString() }}</td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- 新闻部分 -->
      <div v-show="activeSection === 'news'" class="news-section">
        <h3>新闻结果</h3>
        <ul class="news-list">
          <li v-for="item in news" :key="item.id">
            <h4 v-html="item.title"></h4>
            <p class="summary">{{ item.summary }}</p>
            <span class="author">作者: {{ item.username || "未知" }}</span>
            <span class="time">
              发布时间: {{ dayjs.unix(item.issuetime).format("YYYY-MM-DD HH:mm") }}
            </span>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* 页面整体样式 */
.search-page {
  max-width: 100%;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
  width: 90%; /* 页面宽度自适应，最大 90% */
  box-sizing: border-box;
}

/* 搜索框部分 */
.search-header {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 20px;
}
.search-input {
  width: 100%;
  max-width: 500px; /* 限制最大宽度 */
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  margin-right: 10px;
}
.search-button {
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
.search-button:hover {
  background-color: #0056b3;
}

/* 结果标题部分 */
.results-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 20px;
}
.toggle-buttons button {
  padding: 10px 20px;
  margin-left: 10px;
  border: 1px solid #ccc;
  background-color: #f4f4f4;
  cursor: pointer;
  transition: background-color 0.3s ease;
}
.toggle-buttons button.active {
  background-color: #007bff;
  color: white;
}
.toggle-buttons button:hover {
  background-color: #e0e0e0;
}

/* 币种部分样式 */
.coins-section {
  margin-bottom: 20px;
}
.coins-table {
  width: 100%;
  border-collapse: collapse;
  margin-bottom: 20px;
}
.coins-table th,
.coins-table td {
  padding: 10px;
  border: 1px solid #ddd;
}
.coins-table th {
  background-color: #f4f4f4;
}
.coins-table td img {
  width: 36px;
  height: 36px;
}

/* 涨跌幅颜色 */
.positive {
  color: green;
}
.negative {
  color: red;
}

/* 鼠标悬停行高亮 */
.coins-table tr.hovered {
  background-color: #e0f7fa;
  cursor: pointer;
}

/* 新闻部分样式 */
.news-section {
  margin-bottom: 20px;
}
.news-list {
  list-style: none;
  padding: 0;
}
.news-list li {
  margin-bottom: 20px;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background-color: #f9f9f9;
}
.news-list li h4 {
  margin: 0 0 10px;
  color: #007bff;
}
.news-list li .summary {
  margin: 0 0 10px;
  color: #555;
}
.news-list li .author,
.news-list li .time {
  font-size: 12px;
  color: #888;
  display: block;
}

/* 自适应布局 */
@media (max-width: 768px) {
  .search-page {
    width: 100%;
    padding: 15px;
  }
  .search-input {
    width: 100%;
    max-width: 100%;
  }
  .toggle-buttons {
    flex-direction: column;
    align-items: flex-start;
  }
  .toggle-buttons button {
    margin-bottom: 10px;
  }
}
</style>