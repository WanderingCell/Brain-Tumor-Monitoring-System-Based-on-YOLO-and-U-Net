<template>
  <div class="home-container layout-pd">

    <el-row :gutter="15" class="home-card-two mb15">
      <el-col :xs="24" :sm="10" :md="10" :lg="10" :xl="8">
        <div class="home-card-item">
          <div style="height: 100%" ref="homePieRef"></div>
        </div>
      </el-col>
      <el-col :xs="24" :sm="14" :md="14" :lg="14" :xl="16">
        <div class="home-card-item">
          <div style="height: 100%" ref="homeBarRef"></div>
        </div>
      </el-col>
    </el-row>

    <!-- 第二行：折线图-->
    <el-row :gutter="15" class="home-card-three">
      <el-col :xs="24" :sm="24" :md="24" :lg="24" :xl="24">
        <div class="home-card-item">
          <div style="height: 100%" ref="homeLineRef"></div>
        </div>
      </el-col>
    </el-row>

    <!-- 第三行（底部）：实时预测信息 + 雷达图 -->
    <el-row :gutter="15" class="home-card-three" style="margin-top: 15px;">
      <el-col :xs="24" :sm="14" :md="14" :lg="16" :xl="16">
        <div class="home-card-item prediction-card">
          <div class="home-card-item-title">实时预测信息</div>
          <el-table :data="state.data" border stripe highlight-current-row style="width: 100%;" class="prediction-table" height="400px">
            <el-table-column prop="username" label="用户名" align="center"/>
            <el-table-column prop="weight" label="识别权重" align="center">
              <template #default="{ row }">
                <el-tag :type="row.weight > 0.8 ? 'success' : row.weight > 0.5 ? 'warning' : 'danger'" effect="dark">
                  {{ row.weight }}
                </el-tag>
              </template>
            </el-table-column>
            <el-table-column prop="conf" label="最小阈值" align="center">
              <template #default="{ row }">
                <el-progress :percentage="Math.round(row.conf * 100)" status="success" :stroke-width="10"/>
              </template>
            </el-table-column>
            <el-table-column prop="ai" label="AI助手" align="center">
              <template #default="{ row }">
                <el-tag type="info">{{ row.ai }}</el-tag>
              </template>
            </el-table-column>
            <el-table-column prop="startTime" label="时间" align="center"/>
          </el-table>
        </div>
      </el-col>

      <el-col :xs="24" :sm="10" :md="10" :lg="8" :xl="8">
        <div class="home-card-item">
          <div style="height: 100%" ref="homeradarRef"></div>
        </div>
      </el-col>
    </el-row>
  </div>
</template>


<script setup lang="ts" name="home">
import {reactive, onMounted, ref, watch, nextTick, onActivated, markRaw} from 'vue';
import * as echarts from 'echarts';
import {storeToRefs} from 'pinia';
import {useThemeConfig} from '/@/stores/themeConfig';
import {useTagsViewRoutes} from '/@/stores/tagsViewRoutes';
import request from '/@/utils/request';

// 定义变量内容
const homeLineRef = ref();
const homePieRef = ref();
const homeBarRef = ref();
const homeradarRef = ref();
const storesTagsViewRoutes = useTagsViewRoutes();
const storesThemeConfig = useThemeConfig();
const {themeConfig} = storeToRefs(storesThemeConfig);
const {isTagsViewCurrenFull} = storeToRefs(storesTagsViewRoutes);
const state = reactive({
  data: [] as any,
  global: {
    homeChartOne: null,
    homeChartTwo: null,
    homeCharThree: null,
    homeCharFour: null,
    dispose: [null, '', undefined],
  } as any,
  myCharts: [] as EmptyArrayType,
  charts: {
    theme: '',
    bgColor: '',
    color: '#303133',
  },
});

// 折线图
const initLineChart = () => {
  if (!state.global.dispose.some((b: any) => b === state.global.homeChartOne)) state.global.homeChartOne.dispose();
  state.global.homeChartOne = markRaw(echarts.init(homeLineRef.value, state.charts.theme));
  // 1. 统计每天的预测数量
  const counts: Record<string, number> = state.data.reduce((acc, prediction) => {
    // 提取日期部分，例如 "2025-02-11 13:01:33" 提取 "2025-02-11"
    const date = prediction.startTime.split(' ')[0];
    acc[date] = (acc[date] || 0) + 1;
    return acc;
  }, {} as Record<string, number>);

  // 2. 对所有日期按降序排序（最新日期在前）
  const sortedDatesDesc = Object.keys(counts).sort((a, b) => b.localeCompare(a));

  // 3. 取最新的前 10 天数据
  const latest7DatesDesc = sortedDatesDesc.slice(0, 10);

  // 4. 为了便于展示，将这 10 个日期按升序排序（即从最早到最新）
  const latest7Dates = latest7DatesDesc.sort((a, b) => a.localeCompare(b));

  // 5. 构造返回格式
  const result = {
    dateData: latest7Dates,              // 例如：["2025-02-05", "2025-02-06", ..., "2025-02-11"]
    valueData: latest7Dates.map(date => counts[date])
  };

  const colors = [
  new echarts.graphic.LinearGradient(0, 1, 0, 0, [  // 纵向渐变，底部(1)透明，顶部(0)不透明
    { offset: 0, color: 'rgba(255, 127, 80, 0)' },  // 底部透明
    { offset: 1, color: 'rgba(255, 127, 80, 1)' },  // 顶部不透明橙色
  ]),
  new echarts.graphic.LinearGradient(0, 1, 0, 0, [
    { offset: 0, color: 'rgba(135, 206, 250, 0)' },
    { offset: 1, color: 'rgba(0, 191, 255, 1)' },
  ]),
  new echarts.graphic.LinearGradient(0, 1, 0, 0, [
    { offset: 0, color: 'rgba(218, 112, 214, 0)' },
    { offset: 1, color: 'rgba(186, 85, 211, 1)' },
  ]),
  new echarts.graphic.LinearGradient(0, 1, 0, 0, [
    { offset: 0, color: 'rgba(50, 205, 50, 0)' },
    { offset: 1, color: 'rgba(34, 139, 34, 1)' },
  ]),
  new echarts.graphic.LinearGradient(0, 1, 0, 0, [
    { offset: 0, color: 'rgba(255, 165, 0, 0)' },
    { offset: 1, color: 'rgba(255, 140, 0, 1)' },
  ]),
];


const option = {
  backgroundColor: state.charts.bgColor,
  title: {
    text: '近十日预测数量',
    x: 'left',
    textStyle: { fontSize: 15, color: state.charts.color },
  },
  grid: { top: 70, right: 20, bottom: 50, left: 40 },
  tooltip: { trigger: 'axis' },
  xAxis: {
    type: 'category',
    data: result.dateData,
    axisLabel: { color: '#000' },
    axisTick: { alignWithLabel: true },
  },
  yAxis: {
    type: 'value',
    name: '数量',
    splitLine: { show: true, lineStyle: { type: 'dashed', color: '#f5f5f5' } },
    axisLabel: { color: '#000' },
  },
  series: [
    {
      type: 'custom',
      renderItem: function (params, api) {
        const x = api.value(0);
        const y = api.value(1);
        const coord = api.coord([x, y]);
        const zeroCoord = api.coord([x, 0]);
        const barWidth = api.size([1, 0])[0] * 0.6; // 柱子宽度，60%缩小
        const halfWidth = barWidth / 2;

        // 三角形3个顶点坐标
        const points = [
          [coord[0], coord[1]],          // 顶点(上)
          [coord[0] - halfWidth, zeroCoord[1]],  // 左下
          [coord[0] + halfWidth, zeroCoord[1]],  // 右下
        ];

        return {
          type: 'polygon',
          shape: {
            points: points,
          },
          style: api.style({
            fill: colors[params.dataIndex % colors.length],
          }),
        };
      },
      data: result.valueData.map((v, i) => [i, v]),
      // tooltip里展示用
      encode: {
        x: 0,
        y: 1,
      },
    },
    {
      name: '预测数量',
      type: 'line',
      smooth: true,
      symbol: 'circle',
      symbolSize: 6,
      data: result.valueData,
      lineStyle: { color: '#9E87FF' },
      itemStyle: { color: '#9E87FF', borderColor: '#9E87FF' },
      areaStyle: {
        color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
          { offset: 0, color: '#9E87FFb3' },
          { offset: 1, color: '#9E87FF03' },
        ]),
      },
      emphasis: {
        itemStyle: {
          color: {
            type: 'radial',
            x: 0.5,
            y: 0.5,
            r: 0.5,
            colorStops: [
              { offset: 0, color: '#9E87FF' },
              { offset: 0.4, color: '#9E87FF' },
              { offset: 0.5, color: '#fff' },
              { offset: 0.7, color: '#fff' },
              { offset: 0.8, color: '#fff' },
              { offset: 1, color: '#fff' },
            ],
          },
          borderColor: '#9E87FF',
          borderWidth: 2,
        },
      },
    },
  ],
};



  state.global.homeChartOne.setOption(option);
  state.myCharts.push(state.global.homeChartOne);
};

// 饼图
const initPieChart = () => {
  if (!state.global.dispose.some((b: any) => b === state.global.homeChartTwo)) state.global.homeChartTwo.dispose();
  state.global.homeChartTwo = markRaw(echarts.init(homePieRef.value, state.charts.theme));
  // 1. 统计每个 username 的预测数量
  const usernameCounts = state.data.reduce((acc, prediction) => {
    const username = prediction.username;
    acc[username] = (acc[username] || 0) + 1;
    return acc;
  }, {});

  // 2. 将所有 username 按数量从大到小排序
  const sortedUsernames = Object.keys(usernameCounts).sort((a, b) => usernameCounts[b] - usernameCounts[a]);

  // 3. 只取数量最大的前 6 个数据
  const top5Usernames = sortedUsernames.slice(0, 6);
  const top5Values = top5Usernames.map(u => usernameCounts[u]);

  // 构造返回格式
  const result = {
    usernameData: top5Usernames, // 排序后的前五个用户名数组
    valueData: top5Values        // 对应的预测数量数组
  };

  // 将统计数据转换为饼图数据
  const pieData = result.usernameData.map((username, i) => ({
    name: username,
    value: result.valueData[i]
  }));

  // 配置颜色渐变列表和颜色线条数组
  const colorList = [
  {
    type: 'linear',
    x: 0, y: 0, x2: 1, y2: 1,
    colorStops: [
      { offset: 0, color: '#A18CD1' },
      { offset: 1, color: '#FBC2EB' }
    ],
    globalCoord: false
  },
  {
    type: 'linear',
    x: 0, y: 1, x2: 1, y2: 0,
    colorStops: [
      { offset: 0, color: '#FAD961' },
      { offset: 1, color: '#F76B1C' }
    ],
    globalCoord: false
  },
  {
    type: 'linear',
    x: 1, y: 0, x2: 0, y2: 1,
    colorStops: [
      { offset: 0, color: '#84FAB0' },
      { offset: 1, color: '#8FD3F4' }
    ],
    globalCoord: false
  },
  {
    type: 'linear',
    x: 1, y: 1, x2: 0, y2: 0,
    colorStops: [
      { offset: 0, color: '#FCCF31' },
      { offset: 1, color: '#F55555' }
    ],
    globalCoord: false
  },
  {
    type: 'linear',
    x: 0, y: 1, x2: 1, y2: 0,
    colorStops: [
      { offset: 0, color: '#43E97B' },
      { offset: 1, color: '#38F9D7' }
    ],
    globalCoord: false
  },
  {
    type: 'linear',
    x: 0, y: 0, x2: 1, y2: 0,
    colorStops: [
      { offset: 0, color: '#667EEA' },
      { offset: 1, color: '#764BA2' }
    ],
    globalCoord: false
  }
];

  const colorLine = ['#33C0CD', '#73ACFF', '#9E87FF', '#FE6969', '#FDB36A', '#FECE43'];

  // 为饼图中的富文本标签生成样式
  function getRich() {
    let rich = {};
    colorLine.forEach((v, i) => {
      rich[`hr${i}`] = {
        backgroundColor: colorLine[i],
        borderRadius: 3,
        width: 3,
        height: 3,
        padding: [3, 3, 0, -12]
      };
      rich[`a${i}`] = {
        padding: [-11, 6, -20, 6],
        color: colorLine[i],
        backgroundColor: 'transparent',
        fontSize: 12
      };
    });
    return rich;
  }

  // 给每个饼图数据设置 labelLine 的颜色
  pieData.forEach((v: any, i) => {
    v.labelLine = {
      lineStyle: {
        width: 1,
        color: colorLine[i]
      }
    };
  });

const option = {
  backgroundColor: state.charts.bgColor,
  title: {
    text: '不同用户的预测个数',
    left: 'center',
    top: 'top',
    textStyle: {
      fontSize: 16,
      color: state.charts.color,
      fontWeight: 'bold'
    }
  },
  legend: {
    orient: 'horizontal',
    bottom: '0%',
    itemWidth: 12,
    itemHeight: 12,
    textStyle: {
      color: state.charts.color,
      fontSize: 12
    }
  },
  tooltip: {
    trigger: 'item',
    formatter: '{b}: {c} ({d}%)'
  },
  series: [
    {
      name: '用户预测分布',
      type: 'pie',
      radius: ['30%', '70%'], // 中空圆环效果
      center: ['50%', '50%'],
      roseType: 'radius',
      minAngle: 10,
      itemStyle: {
        borderRadius: 10,  // 圆角
        borderWidth: 2,
        borderColor: state.charts.bgColor,
        color: function (params) {
          return colorList[params.dataIndex % colorList.length];
        }
      },
      label: {
        show: true,
        position: 'outside',
        formatter: function (params) {
          const name = params.name;
          const percent = params.percent + '%';
          const index = params.dataIndex;
          return [`{a${index}|${name}：${percent}}`, `{hr${index}|}`].join('\n');
        },
        rich: getRich()
      },
      labelLine: {
        length: 16,
        length2: 12,
        lineStyle: {
          width: 1
        }
      },
      data: pieData,
      animationType: 'scale',
      animationEasing: 'elasticOut',
      animationDelay: function (idx) {
        return Math.random() * 200;
      }
    }
  ]
};


  state.global.homeChartTwo.setOption(option);
  state.myCharts.push(state.global.homeChartTwo);
};
//雷达图
const initradarChart = () => {
  if (!state.global.dispose.some((b: any) => b === state.global.homeCharFour)) state.global.homeCharFour.dispose();
  state.global.homeCharFour = markRaw(echarts.init(homeradarRef.value, state.charts.theme));
  // 1. 按 username 分组，累计置信度的平均值和计数
  const confStatsByUser = state.data.reduce((acc, prediction) => {
    const {username, confidence} = prediction;
    const confidences = JSON.parse(confidence).map(percentStr => {
      const numValue = parseFloat(percentStr.replace('%', '')) / 100;
      return Number(numValue.toFixed(4));
    });
    const predictionAvg = confidences.reduce((sum, val) => sum + val, 0) / confidences.length;
    if (!acc[username]) {
      acc[username] = {total: predictionAvg, count: 1};
    } else {
      acc[username].total += predictionAvg;
      acc[username].count += 1;
    }
    return acc;
  }, {});

  // 2. 计算每个用户的最终平均置信度（保持原始顺序）
  const avgConfData = Object.keys(confStatsByUser).map(username => ({
    username,
    avgConf: confStatsByUser[username].total / confStatsByUser[username].count,
  }));

  // 3. 取前7个用户（不排序）
  const top7AvgConfData = avgConfData.slice(0, 7);

  // 4. 格式化为百分比数值（保留两位小数）
  const result = {
    usernameData: top7AvgConfData.map(item => item.username),
    valueData: top7AvgConfData.map(item => Number((item.avgConf * 100).toFixed(2))),
  };

  // 构造雷达图所需数据
  const data = top7AvgConfData.map(item => Number((item.avgConf * 100).toFixed(2)));
  const indicatorNames = top7AvgConfData.map(item => item.username);
  const maxData = Array(data.length).fill(100);

  // 构造 indicator 数组
  const indicator = indicatorNames.map((name, idx) => ({name, max: maxData[idx]}));

  // 辅助函数：生成内层数据
  const innerData = i => Array(data.length).fill(100 - 20 * i);

  // 构造系列数据
  const getData = data => {
    const series = [
      {
        type: 'radar',
        symbolSize: 10,
        symbol: 'circle',
        areaStyle: {
          color: 'rgba(108,80,243,0.5)',
          opacity: 0.3,
        },
        lineStyle: {
          color: new echarts.graphic.LinearGradient(
              0, 0, 0, 1,
              [
                {offset: 0, color: 'rgba(108,80,243,0)'},
                {offset: 1, color: 'rgba(108,80,243,0.3)'},
              ],
              false
          ),
          width: 3,
        },
        itemStyle: {
          color: '#fff',
          borderColor: new echarts.graphic.LinearGradient(
              0, 0, 0, 1,
              [
                {offset: 0, color: 'rgba(108,80,243,0)'},
                {offset: 1, color: 'rgba(108,80,243,0.3)'},
              ],
              false
          ),
          borderWidth: 4,
          opacity: 1,
        },
        label: {show: false},
        data: [{value: data}],
        z: 100,
      },
    ] as any;

    // 添加每个指标对应的内层系列
    data.forEach((_, i) => {
      series.push({
        type: 'radar',
        data: [{value: innerData(i)}],
        symbol: 'none',
        lineStyle: {width: 0},
        itemStyle: {color: '#fff'},
        areaStyle: {
          color: '#fff',
          shadowColor: 'rgba(14,122,191,0.15)',
          shadowBlur: 30,
          shadowOffsetY: 20,
        },
      });
    });
    return {series};
  };

  const optionData = getData(data);

  const option = {
  backgroundColor: state.charts.bgColor,
  title: {
    text: '不同用户间的平均置信度',
    left: 'left',
    textStyle: {
      fontSize: 16,
      color: state.charts.color,
      fontWeight: '600',
    },
  },
  tooltip: {
    backgroundColor: 'rgba(50,50,50,0.7)',
    textStyle: { color: '#fff' },
    formatter: () =>
      indicatorNames
        .map((name, i) => `${name} : ${data[i]}%`)
        .join('<br>'),
  },
  radar: {
    radius: '70%',
    center: ['50%', '50%'],
    indicator: indicator,
    splitArea: {
      show: true,
      areaStyle: {
        color: ['#f0f4ff', '#d9e2ff', '#f0f4ff', '#d9e2ff'], // 交替浅蓝色
      },
    },
    splitLine: {
      show: true,
      lineStyle: {
        color: 'rgba(108,80,243,0.3)', // 轻柔的紫色线条
        width: 1,
        type: 'dashed',
      },
    },
    axisLine: {
      lineStyle: {
        color: 'rgba(108,80,243,0.4)', // 柔和的轴线颜色
      },
    },
    axisLabel: { show: false },
    name: {
      formatter: params => {
        const idx = indicatorNames.indexOf(params);
        const percent = data[idx];
        return `{percentStyle|${percent}%}\n{nameStyle|${params}}`;
      },
      rich: {
        percentStyle: {
          fontSize: 14,
          fontWeight: '700',
          color: '#6c50f3',
          padding: [0, 0, 4, 0],
        },
        nameStyle: {
          fontSize: 12,
          color: '#333',
          fontWeight: '500',
        },
      },
    },
  },
  series: [{
    type: 'radar',
    data: optionData.series[0].data,
    symbol: 'circle',
    symbolSize: 6,
    lineStyle: {
      width: 3,
      color: 'rgba(108,80,243,0.9)',
      shadowColor: 'rgba(108,80,243,0.5)',
      shadowBlur: 10,
      shadowOffsetY: 4,
    },
    areaStyle: {
      color: new echarts.graphic.RadialGradient(0.5, 0.5, 0.7, [
        { offset: 0, color: 'rgba(108,80,243,0.4)' },
        { offset: 1, color: 'rgba(108,80,243,0.1)' },
      ]),
      shadowColor: 'rgba(108,80,243,0.5)',
      shadowBlur: 15,
    },
    // 添加呼吸动画效果: 通过动画更新透明度
    animation: true,
    animationDuration: 2000,
    animationEasing: 'cubicInOut',
    animationLoop: true,
    emphasis: {
      lineStyle: {
        width: 4,
      },
      areaStyle: {
        color: 'rgba(108,80,243,0.6)',
      },
    },
  }]

};

  state.global.homeCharFour.setOption(option);
  state.myCharts.push(state.global.homeCharFour);
};
// 柱状图
const initBarChart = () => {
  if (!state.global.dispose.some((b: any) => b === state.global.homeCharThree)) state.global.homeCharThree.dispose();
  state.global.homeCharThree = markRaw(echarts.init(homeBarRef.value, state.charts.theme));
  // 固定的五个类别
  const categories = ['正常', '胶质瘤', '脑膜瘤', '垂体', '占位性病变'];
  /**
   * 统计数据中每个 label（已 JSON 字符串存储）中对应类别的出现次数
   * @param data 数据数组
   * @returns 一个元组，第一个数组为类别数组，第二个数组为对应的次数（字符串格式）
   */
  function countLabelCategories(data: any[]): [string[], string[]] {
    // 初始化计数器
    const counts: Record<string, number> = {};
    categories.forEach(cat => {
      counts[cat] = 0;
    });

    data.forEach(item => {
      try {
        // 解析 label 字段（假设为 JSON 字符串格式，如 '["正常"]'）
        const labels: string[] = JSON.parse(item.label);
        labels.forEach(label => {
          if (categories.includes(label)) {
            counts[label]++;
          }
        });
      } catch (error) {
        console.error('解析 label 失败：', item, error);
      }
    });

    // 将计数转为字符串数组
    const countStrings = categories.map(cat => counts[cat].toString());
    return [categories, countStrings];
  }

  // 调用函数并输出结果
  const [catArr, countsArr] = countLabelCategories(state.data);
  const gradientColors = [
  ['#8E44AD', '#D2B4DE'],
  ['#2980B9', '#85C1E9'],
  ['#27AE60', '#82E0AA'],
  ['#F1C40F', '#F9E79F'],
  ['#E67E22', '#F5CBA7'],
  ['#C0392B', '#F1948A'],
  ['#7D3C98', '#BB8FCE'],
  ['#138D75', '#76D7C4'],
  ['#2471A3', '#AED6F1'],
  ['#D68910', '#F7DC6F'],
];

const option = {
  backgroundColor: state.charts.bgColor,
  title: {
    text: '不同结果的检测个数',
    x: 'left',
    textStyle: { fontSize: '15', color: state.charts.color },
  },
  tooltip: { trigger: 'axis' },
  grid: { top: 70, right: 80, bottom: 30, left: 80 },
  xAxis: [
    {
      type: 'category',
      data: catArr,
      boundaryGap: true,
      axisTick: { show: false },
      axisLabel: {
        color: '#000',
      },
    },
  ],
  yAxis: [
    {
      type: 'value',
      name: '个数',
      splitLine: { show: true, lineStyle: { type: 'dashed', color: '#f5f5f5' } },
      axisLabel: {
        color: '#000',
      },
    },
  ],
  series: [
    {
      name: '预测个数',
      type: 'bar',
      barWidth: 30,
      data: countsArr.map((val, i) => ({
        value: val,
        itemStyle: {
          color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
            { offset: 0, color: gradientColors[i % gradientColors.length][0] },
            { offset: 1, color: gradientColors[i % gradientColors.length][1] },
          ]),
          borderRadius: [30, 30, 0, 0],
          shadowColor: 'rgba(0, 0, 0, 0.2)',
          shadowBlur: 8,
          shadowOffsetX: 2,
          shadowOffsetY: 2,
        },
      })),
    },
  ],
};

  state.global.homeCharThree.setOption(option);
  state.myCharts.push(state.global.homeCharThree);
};
// 批量设置 echarts resize
const initEchartsResizeFun = () => {
  nextTick(() => {
    for (let i = 0; i < state.myCharts.length; i++) {
      setTimeout(() => {
        state.myCharts[i].resize();
      }, i * 1000);
    }
  });
};
// 批量设置 echarts resize
const initEchartsResize = () => {
  window.addEventListener('resize', initEchartsResizeFun);
};
// 页面加载时
onMounted(() => {
  request.get('/api/imgRecords/all').then((res) => {
    if (res.code == 0) {
      state.data = res.data.reverse();
    }
  });
  initEchartsResize();
});
// 由于页面缓存原因，keep-alive
onActivated(() => {
  initEchartsResizeFun();
});
// 监听 pinia 中的 tagsview 开启全屏变化，重新 resize 图表，防止不出现/大小不变等
watch(
    () => isTagsViewCurrenFull.value,
    () => {
      initEchartsResizeFun();
    }
);
// 监听 pinia 中是否开启深色主题
watch(
    () => themeConfig.value.isIsDark,
    (isIsDark) => {
      nextTick(() => {
        state.charts.theme = isIsDark ? 'dark' : '';
        state.charts.bgColor = isIsDark ? 'transparent' : '';
        state.charts.color = isIsDark ? '#dadada' : '#303133';
        setTimeout(() => {
          initLineChart();
          initradarChart();
          initPieChart();
          initBarChart();
        }, 500);
      });
    },
    {
      deep: true,
      immediate: true,
    }
);
</script>

<style scoped lang="scss">
$homeNavLengh: 8;

.home-container {
  overflow: hidden;

  .home-card-one,
  .home-card-two,
  .home-card-three {
    .home-card-item {
      width: 100%;
      height: 130px;
      border-radius: 4px;
      transition: all ease 0.3s;
      padding: 20px;
      overflow: hidden;
      background: var(--el-color-white);
      color: var(--el-text-color-primary);
      border: 1px solid var(--next-border-color-light);

      &:hover {
        box-shadow: 0 2px 12px var(--next-color-dark-hover);
        transition: all ease 0.3s;
      }

      &-icon {
        width: 70px;
        height: 70px;
        border-radius: 100%;
        flex-shrink: 1;

        i {
          color: var(--el-text-color-placeholder);
        }
      }

      &-title {
        font-size: 15px;
        font-weight: bold;
        height: 30px;
      }
    }
  }

  .home-card-one {
    @for $i from 0 through 3 {
      .home-one-animation#{$i} {
        opacity: 0;
        animation-name: error-num;
        animation-duration: 0.5s;
        animation-fill-mode: forwards;
        animation-delay: calc($i/4) + s;
      }
    }
  }

  .home-card-two,
  .home-card-three {
    .home-card-item {
      height: 400px;
      width: 100%;
      overflow: hidden;

      .home-monitor {
        height: 100%;

        .flex-warp-item {
          width: 25%;
          height: 111px;
          display: flex;

          .flex-warp-item-box {
            margin: auto;
            text-align: center;
            color: var(--el-text-color-primary);
            display: flex;
            border-radius: 5px;
            background: var(--next-bg-color);
            cursor: pointer;
            transition: all 0.3s ease;

            &:hover {
              background: var(--el-color-primary-light-9);
              transition: all 0.3s ease;
            }
          }

          @for $i from 0 through $homeNavLengh {
            .home-animation#{$i} {
              opacity: 0;
              animation-name: error-num;
              animation-duration: 0.5s;
              animation-fill-mode: forwards;
              animation-delay: calc($i/10) + s;
            }
          }
        }
      }
    }
  }
}
</style>
