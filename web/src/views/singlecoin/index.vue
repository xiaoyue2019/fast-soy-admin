<script setup lang="ts">
import { ref, onMounted } from 'vue';
import * as echarts from 'echarts';
import 'ant-design-vue/dist/antd.css';
import { useRoute } from 'vue-router'; // 引入 useRoute
import axios from 'axios';
import dayjs from 'dayjs';

// 初始化状态
const startDate = ref(null); // 开始日期
const endDate = ref(null);   // 结束日期
const chartRef = ref(null);  // 历史走势图表 DOM 引用
const yearHighChartRef = ref(null); // 年度最高点热力图 DOM 引用
const chartRef_rank = ref(null);  // 市值排名图表 DOM 引用
const holderStats = ref(null); // 顶部统计数据
const mainContracts = ref([]); // 主要合约
const topHolders = ref([]); // 大户列表数据
const twitterNews = ref([]); // 资讯
// 社交媒体数据
const socialData = ref(null);
const holderTrendData = ref([]);

// 获取路由参数传入的币种名称
const route = useRoute();
const coinName = ref(route.params.coin || 'ripple'); // 默认值为 'ripple'，如果没有传递则使用该默认值

// 请求社交媒体数据和持币地址变化趋势数据 资讯 请求
const fetchSocialData = async () => {
  try {
    const response = await axios.get(`https://dncapi.flink1.com/api/v3/coin/hotsocial?coincode=${coinName.value}&webp=1`);
    if (response.data?.data) {
      socialData.value = response.data.data.social;
      holderTrendData.value = response.data.data.holdcoin.list;
      renderSocialChart();
      renderHolderTrendChart();
    } else {
      alert('社交媒体数据获取失败');
    }
  } catch (error) {
    console.error('请求失败:', error);
    alert('网络错误，请稍后再试！');
  }
};

// 渲染社交媒体统计图表 资讯
const renderSocialChart = () => {
  const chartDom = document.getElementById('socialChart');
  const chart = echarts.init(chartDom);

  const option = {
    title: {
      text: '社交媒体关注人数',
      left: 'center',
    },
    tooltip: {
      trigger: 'item',
    },
    series: [
      {
        name: '关注人数',
        type: 'pie',
        radius: '50%',
        data: [
          { value: socialData.value?.redditcount, name: 'Reddit' },
          { value: socialData.value?.twittercount, name: 'Twitter' },
          { value: socialData.value?.facebookcount, name: 'Facebook' },
        ],
        emphasis: {
          itemStyle: {
            shadowBlur: 10,
            shadowOffsetX: 0,
            shadowColor: 'rgba(0, 0, 0, 0.5)',
          },
        },
      },
    ],
  };

  chart.setOption(option);
};

// 资讯
const renderHolderTrendChart = () => {
  const chartDom = document.getElementById('holderTrendChart');
  const chart = echarts.init(chartDom);

  const dates = holderTrendData.value.map(item => item.updatedate);
  const addressCounts = holderTrendData.value.map(item => item.addrcount);

  const option = {
    title: {
      text: '持币地址变化趋势',
      left: 'center',
    },
    tooltip: {
      trigger: 'axis',
    },
    xAxis: {
      type: 'category',
      data: dates,
    },
    yAxis: {
      type: 'value',
      name: '地址数量',
    },
    series: [
      {
        data: addressCounts,
        type: 'line',
        smooth: true,
        itemStyle: { color: '#73C0DE' },
        lineStyle: { width: 2 },
      },
    ],
  };

  chart.setOption(option);
};

// 获取 Twitter 资讯
const fetchTwitterNews = async () => {
  try {
    const response = await axios.get(
      'https://dncapi.flink1.com/api/v5/News/cointwitter',
      { params: { code: coinName.value, per_Page: 10, page: 1, webp: 1 } }
    );
    if (response.data?.data?.list) {
      twitterNews.value = response.data.data.list.map((item) => ({
        id: item.articleid,
        author: item.authorname,
        avatar: item.authoravatar,
        content: item.content,
        time: dayjs.unix(item.publishtime).format('YYYY-MM-DD HH:mm'),
      }));
    } else {
      alert('无法获取 Twitter 数据');
    }
  } catch (error) {
    console.error('Twitter 数据获取失败:', error);
    alert('网络错误，请稍后再试！');
  }
};

// 自动刷新 Twitter 数据
const startTwitterRefresh = () => {
  fetchTwitterNews();
  setInterval(fetchTwitterNews, 3600 * 1000); // 每小时刷新
};

// 大户请求
const fetchHoldersData = async () => {
  try {
    const response = await axios.get(
      'https://dncapi.flink1.com/api/v3/coin/holders',
      { params: { code: coinName.value, webp: 1 } }
    );

    if (response.data?.data) {
      const { top, maincoins, toplist } = response.data.data;
      holderStats.value = top;
      mainContracts.value = maincoins;
      topHolders.value = toplist;
    } else {
      alert('数据获取失败！');
    }
  } catch (error) {
    console.error('请求失败:', error);
    alert('网络错误，请稍后再试！');
  }
};

// 大户环形图
const initPieChart = () => {
  const chartDom = document.getElementById('top10Chart');
  if (!chartDom) return;

  const chart = echarts.init(chartDom);

  const option = {
    title: {
      text: 'Top 地址持币比例',
      left: 'center',
    },
    tooltip: {
      trigger: 'item',
    },
    series: [
      {
        name: '比例',
        type: 'pie',
        radius: ['40%', '70%'],
        data: [
          { value: holderStats.value?.top10rate, name: 'Top 10' },
          { value: holderStats.value?.top20rate - holderStats.value?.top10rate, name: 'Top 20' },
          { value: holderStats.value?.top50rate - holderStats.value?.top20rate, name: 'Top 50' },
          { value: holderStats.value?.top100rate - holderStats.value?.top50rate, name: 'Top 100' },
        ],
      },
    ],
  };

  chart.setOption(option);
};

// 请求数据
const fetchData = async () => {
  if (!startDate.value || !endDate.value) {
    alert('请选择起始和结束时间！');
    return;
  }

  const formattedStartDate = dayjs(startDate.value).format('YYYYMMDD');
  const formattedEndDate = dayjs(endDate.value).format('YYYYMMDD');

  console.log('开始时间:', formattedStartDate);
  console.log('结束时间:', formattedEndDate);

  try {
    const response = await axios.get(
      `https://dncapi.flink1.com/api/v3/coin/history`,
      {
        params: {
          coincode: coinName.value,
          begintime: formattedStartDate,
          endtime: formattedEndDate,
          page: 1,
          per_page: 1000,
          webp: 1,
        },
      }
    );

    if (response.data.code === 200) {
      const data = response.data.data;
      updateHistoryChart(data);
      updateYearHighHeatmap(data.highpricekline);
    } else {
      alert('数据获取失败：' + response.data.msg);
    }
  } catch (error) {
    console.error('数据获取失败:', error);
    alert('请求失败，请检查网络连接。');
  }
};

// 请求数据rank
const fetchData_rank = async () => {
  try {
    const response = await axios.get(
      `https://dncapi.flink1.com/api/coin/coinhisrank?code=${coinName.value}&webp=1`
    );
    if (response.data.data) {
      const data = response.data.data;
      updateRankChart(data);
    } else {
      alert('数据获取失败');
    }
  } catch (error) {
    console.error('请求数据失败:', error);
    alert('请求失败，请检查网络连接。');
  }
};

// 更新历史走势图表
const updateHistoryChart = (data) => {
  if (!chartRef.value) return;

  const chart = echarts.init(chartRef.value);

  // 对数据进行排序（按时间正序）
  const sortedData = data.list.sort((a, b) => new Date(a.tickertime) - new Date(b.tickertime));

  // 提取所需数据
  const dates = sortedData.map((item) => item.tickertime);
  const openPrices = sortedData.map((item) => item.openprice);
  const closePrices = sortedData.map((item) => item.closeprice);
  const highPrices = sortedData.map((item) => item.high);
  const lowPrices = sortedData.map((item) => item.low);
  const marketCaps = sortedData.map((item) => item.marketcap);

  const option = {
    title: {
      text: '历史走势',
      left: 'center',
    },
    tooltip: {
      trigger: 'axis',
    },
    legend: {
      top: '10%',
      left: 'center',
      itemGap: 20,
      itemWidth: 20,
      itemHeight: 10,
      orient: 'horizontal',
      data: ['开盘价', '收盘价', '最高价', '最低价', '市值'],
    },
    grid: {
      top: '25%',
      left: '10%',
      right: '10%',
      bottom: '15%',
    },
    xAxis: {
      type: 'category',
      data: dates,
      axisLabel: {
        rotate: 45,
        formatter: (value) => value.split('T')[0], // 日期格式化
      },
    },
    yAxis: [
      {
        type: 'value',
        name: '价格',
        position: 'left',
      },
      {
        type: 'value',
        name: '市值',
        position: 'right',
      },
    ],
    series: [
      {
        name: '开盘价',
        type: 'line',
        data: openPrices,
        smooth: true,
        lineStyle: { width: 2 },
        itemStyle: { color: '#5470C6' },
      },
      {
        name: '收盘价',
        type: 'line',
        data: closePrices,
        smooth: true,
        lineStyle: { width: 2 },
        itemStyle: { color: '#91CC75' },
      },
      {
        name: '最高价',
        type: 'line',
        data: highPrices,
        smooth: true,
        lineStyle: { width: 2 },
        itemStyle: { color: '#EE6666' },
      },
      {
        name: '最低价',
        type: 'line',
        data: lowPrices,
        smooth: true,
        lineStyle: { width: 2 },
        itemStyle: { color: '#73C0DE' },
      },
      {
        name: '市值',
        type: 'line',
        data: marketCaps,
        smooth: true,
        yAxisIndex: 1, // 使用第二个 Y 轴
        lineStyle: { width: 2 },
        itemStyle: { color: '#FAC858' },
      },
    ],
  };

  chart.setOption(option);
};

// 更新年度最高点热力图
const updateYearHighHeatmap = (highPriceKline) => {
  if (!yearHighChartRef.value) return;

  const chart = echarts.init(yearHighChartRef.value);

  // 生成热力图数据
  const heatmapData = [];
  const years = [];
  highPriceKline.forEach((item) => {
    const year = item.year;
    years.push(year); // 年份
    item.list.forEach((value, monthIndex) => {
      heatmapData.push([monthIndex, years.indexOf(year), value]); // [月份索引, 年份索引, 值]
    });
  });

  const option = {
    title: {
      text: '年度最高点热力图',
      left: 'center',
    },
    tooltip: {
      position: 'top',
      formatter: (params) => {
        return `${params.data[1] + 2021}年 ${params.data[0] + 1}月<br/>最高价: ${params.data[2]}`;
      },
    },
    grid: {
      top: '16%', // 图表顶部距离
      left: '10%',
      right: '10%',
      bottom: '12%', // 图表底部距离，不用过大，保持图表大小
    },
    xAxis: {
      type: 'category',
      data: ['1月', '2月', '3月', '4月', '5月', '6月', '7月', '8月', '9月', '10月', '11月', '12月'],
      splitArea: {
        show: true,
      },
    },
    yAxis: {
      type: 'category',
      data: years.map((year) => `${year}年`),
      splitArea: {
        show: true,
      },
    },
    visualMap: {
      min: 0,
      max: Math.max(...heatmapData.map((d) => d[2])),
      calculable: true,
      orient: 'horizontal', // 横向显示图例
      left: 'center', // 图例居中
      bottom: '-2%', // 让图例单独下移，而不影响图表大小
    },
    series: [
      {
        name: '月度最高点',
        type: 'heatmap',
        data: heatmapData,
        label: {
          show: true,
        },
        emphasis: {
          itemStyle: {
            shadowBlur: 10,
            shadowColor: 'rgba(0, 0, 0, 0.5)',
          },
        },
      },
    ],
  };

  chart.setOption(option);
};

// 更新市值排名图表
const updateRankChart = (data) => {
  if (!chartRef_rank.value) return;

  const chart = echarts.init(chartRef_rank.value);

  // 处理数据
  const dates = data.map(item => dayjs(item.ticker_time).format('YYYY-MM-DD'));
  const ranks = data.map(item => item.rank_no);

  const option = {
    title: {
      text: '历史市值排行榜',
      left: 'center',
    },
    tooltip: {
      trigger: 'axis',
      formatter: (params) => {
        // 格式化提示框内容，展示日期和排名
        const date = params[0].name;
        const rank = params[0].value;
        return `日期: ${date}<br/>排名: ${rank}`;
      },
    },
    xAxis: {
      type: 'category',
      data: dates.map(date => {
        // 将日期格式化为 "YYMMDD" 的形式
        const [year, month, day] = date.split('-');
        return `${year.slice(2)}${month}${day}`;
      }),
      axisLabel: {
        rotate: 45, // 日期旋转
        fontSize: 12, // 日期字体大小
      },
    },
    yAxis: {
      type: 'value',
      name: '排名',
      inverse: false, // 排名越小越高，所以需要反转 Y 轴
      min: Math.min(...ranks), // 设置 Y 轴最小值为数据的最小值
      max: Math.max(...ranks), // 设置 Y 轴最大值为数据的最大值
    },
    series: [
      {
        name: '市值排名',
        type: 'line',
        data: ranks,
        smooth: 1, // 调整为具体数值，增加曲线弯曲程度
        lineStyle: {
          width: 2, // 增加线条宽度
          color: '#73C0DE', // 设置曲线颜色
        },
        itemStyle: {
          color: '#73C0DE',
        },
        showSymbol: false, // 隐藏拐点
        animationDuration: 2000, // 动画时长
        animationEasing: 'cubicOut', // 动画效果
      },
    ],
    dataZoom: [
      {
        type: 'inside', // 启用鼠标滚轮缩放
        start: 0,
        end: 100,
      },
      {
        type: 'slider', // 启用滑动条
        show: false,
        start: 0,
        end: 100,
      },
    ],
  };

  chart.setOption(option);
};

// 页面加载时的默认请求
onMounted(() => {
  // 计算最近半年的时间范围
  const now = dayjs();
  endDate.value = now.toDate(); // 当前日期
  startDate.value = now.subtract(24, 'month').toDate(); // 半年前

  // 自动请求数据
  fetchData();
  fetchData_rank();
  fetchHoldersData().then(() => {
    initPieChart();
  });
  startTwitterRefresh();
  fetchSocialData();
});

</script>

<template>
  <!-- 历史走势 年度最高点热力图 市值排名图表 -->
  <br/>
  <div>
    <!-- 历史走势 -->
    <div style="margin-bottom: 40px;">
      <div ref="chartRef" style="width: 100%; height: 400px;"></div>
    </div>

    <!-- 年度最高点热力图 -->
    <div>
      <div ref="yearHighChartRef" style="width: 100%; height: 500px;"></div>
    </div>
    
    <br/>
    <br/>
    <!-- 市值排名图表 -->
    <div>
      <div ref="chartRef_rank" style="width: 100%; height: 400px;"></div>
    </div>

  </div>

  <!-- 大户地址详情 -->
  <div>
    <!-- 模块标题 -->
    <h2 style="text-align: center; margin-bottom: 20px;">大户地址详情</h2>

    <div style="display: flex; gap: 20px; margin-bottom: 20px;">
    <!-- 顶部统计卡片 -->
    <div style="flex: 1; display: grid; grid-template-columns: repeat(auto-fill, minmax(120px, 1fr)); gap: 10px; align-content: stretch; min-height: 400px; border: 1px solid #ddd; border-radius: 10px; background: #f5f5f5; padding: 10px;">
  <div v-if="holderStats" v-for="(value, key) in holderStats" :key="key" style="text-align: center; padding: 20px; border: 1px solid #ccc; border-radius: 10px; background: #fff; transition: transform 0.3s ease; cursor: pointer;" 
       @mouseover="hover = true" @mouseleave="hover = false" :style="{ transform: hover ? 'scale(1.05)' : 'scale(1)' }">
    <h3 style="font-size: 16px; margin: 0; color: #333;">
      <span v-if="key === 'updatedate'">更新时间</span>
      <span v-if="key === 'addrcount'">持币地址总数</span>
      <span v-if="key === 'top10rate'">TOP10持币占比</span>
      <span v-if="key === 'top20rate'">TOP20持币占比</span>
      <span v-if="key === 'top50rate'">TOP50持币占比</span>
      <span v-if="key === 'top100rate'">TOP100持币占比</span>
    </h3>
    <p style="margin: 5px 0 0; color: #555;">
      {{ value }}
    </p>
  </div>
</div>

  <!-- 持币比例环形图 -->
  <div id="top10Chart" style="flex: 1; min-height: 400px; border: 1px solid #ddd; border-radius: 10px; background: #fff;"></div>
</div>

    <!-- 大户列表 -->
    <table style="width: 100%; border-collapse: collapse; margin-bottom: 20px;">
      <thead>
        <tr>
          <th>地址</th>
          <th>持仓数量</th>
          <th>占比</th>
          <th>平台</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="holder in topHolders" :key="holder.address">
          <!-- <td><a :href="`https://www.oklink.com/zh-hans/bsc/address/${holder.address}/assets`" target="_blank">{{ holder.address }}</a></td> -->
          <td><a :href="`https://www.oklink.com/zh-hans/all-chain`" target="_blank">{{ holder.address }}</a></td>
          
          <td>{{ holder.quantity }}</td>
          <td>{{ holder.percentage }}%</td>
          <td>
            {{ holder.platform_name }}
          </td>
        </tr>
      </tbody>
    </table>
  </div>

  <!-- 新闻动态 -->
  <div style="padding: 20px;">
    <!-- 使用 Flexbox 将两个模块并排 -->
    <div style="display: flex; gap: 20px; justify-content: space-between; flex-wrap: wrap;">

      <!-- Twitter资讯 -->
      <div style="flex: 1; min-width: 300px; display: flex; flex-direction: column; justify-content: flex-start; gap: 15px; height: 100%;">
        <h2 style="text-align: center; margin-top: 5px; font-size: 18px;">Twitter 最新资讯</h2>
        <div style="display: flex; flex-wrap: wrap; gap: 10px; justify-content: space-between; flex: 1; margin-top: -10px;">
          <!-- 限制显示最多8条新闻 -->
          <div
            v-for="(news, index) in twitterNews.slice(0, 10)"
            :key="news.id"
            style="width: 48%; background: #fff; border-radius: 8px; box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1); padding: 12px; transition: transform 0.3s; height: 180px; overflow: hidden;"
            @mouseover="event.currentTarget.style.transform = 'scale(1.05)'"
            @mouseout="event.currentTarget.style.transform = 'scale(1)'"
          >
            <div style="display: flex; align-items: center; margin-bottom: 8px; font-size: 12px;">
              <img :src="news.avatar" alt="avatar" style="width: 32px; height: 32px; border-radius: 50%; margin-right: 10px;" />
              <span style="font-weight: bold; font-size: 12px;">{{ news.author }}</span>
            </div>
            <!-- 允许文本换行，但不超出框 -->
            <p style="font-size: 12px; color: #555; margin-bottom: 8px; line-height: 1.4; max-height: 60px; overflow: hidden; text-overflow: ellipsis;" :title="news.content">{{ news.content }}</p>
            <span style="font-size: 10px; color: #888;">发布时间: {{ news.time }}</span>
          </div>
        </div>
      </div>

      <!-- 社交媒体统计 -->
      <div style="flex: 1; min-width: 300px; display: flex; flex-direction: column; justify-content: space-between; height: 100%; padding: 20px; background: #f9f9f9; border-radius: 8px; box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);">
        <!-- 社交媒体关注人数 -->
        <div style="text-align: center; font-size: 16px; color: #333;">
          <h3>社交媒体关注人数</h3>
          <div id="socialChart" style="width: 100%; height: 300px;"></div>
        </div>

        <!-- 关注人数卡片 -->
        <div style="display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 20px; margin-top: 20px;">
          <div v-for="(count, platform) in {
              Reddit: socialData?.redditcount || 0,
              Twitter: socialData?.twittercount || 0,
              Facebook: socialData?.facebookcount || 0
            }" :key="platform" class="social-card">
            <div class="social-card-content">
              <h4 style="font-size: 14px;">{{ platform }}</h4>
              <p class="count" style="font-size: 14px;">{{ count.toLocaleString() }}</p>
              <p class="growth" style="font-size: 12px; color: #888;">
                {{ 
                  platform === 'Reddit' ? socialData?.redditcountup : 
                  platform === 'Twitter' ? socialData?.twittercountup : 
                  socialData?.facebookcountup 
                }} ↑
              </p>
            </div>
          </div>
        </div>

        <!-- 持币地址变化趋势 -->
        <div style="margin-top: 20px;">
          <h3 style="font-size: 16px;">持币地址变化趋势</h3>
          <div id="holderTrendChart" style="width: 100%; height: 400px;"></div>
        </div>
      </div>

    </div>
  </div>

</template>

<style scoped>
.social-card {
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  padding: 20px;
  transition: transform 0.3s;
  height: 100%;
}

.social-card:hover {
  transform: scale(1.05);
}

.social-card-content {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  height: 100%;
}

.count {
  font-size: 16px;
  font-weight: bold;
}

.growth {
  font-size: 14px;
  color: #888;
}
.social-card {
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  padding: 20px;
  transition: transform 0.3s;
  height: 100%;
}

.social-card:hover {
  transform: scale(1.05);
}

.social-card-content {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  height: 100%;
}

.count {
  font-size: 16px;
  font-weight: bold;
}

.growth {
  font-size: 14px;
  color: #888;
}
button:hover {
  background-color: #40a9ff;
}
table {
  border: 1px solid #ddd;
}

th, td {
  padding: 10px;
  text-align: center;
  border: 1px solid #ddd;
}
.social-card {
  background-color: #fff;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s ease;
  cursor: pointer;
}

.social-card:hover {
  transform: scale(1.05);
}

.social-card-content h4 {
  font-size: 16px;
  color: #333;
  margin-bottom: 10px;
}

.social-card-content .count {
  font-size: 24px;
  font-weight: bold;
  color: #1e90ff;
}

.social-card-content .growth {
  color: #2ecc71;
  font-size: 14px;
}
</style>