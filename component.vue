<template>
  <div class='app'>
    <div class="firstScreen">
      <div class="silos">
        <div class="silos_box" v-for="(item, index) in silosList" :key="item.id || index">
          <div class="silos_box_title">
            <div class="real-time-clock">
              {{ currentTime }}
            </div>
          </div>
          <div class="silos_box_center">
            <div class="silos_box_left">
              <div class="left_text">
                <span>材料规格：</span>
                <div class="content">{{ item.materialSpec || '中粗砂（0-4.75mm）' }}</div>
              </div>
              <div class="left_text">
                <span>检验日期：</span>
                <div class="content" style="width: 100px;">{{ formatDate(item.inspectionDate) || '2025-07-06' }}</div>
              </div>
              <div class="left_text">
                <span>检验状态：</span>
                <div class="content" style="width: 70px;">{{ getStatusText(item.status) || '合格' }}</div>
              </div>
              <div class="left_text">
                <span>报告编号：</span>
                <div class="content" title="">{{ item.number || 'BG-HT-A2-2025-XJL-0062' }}</div>
              </div>
              <div class="left_text">
                <span>库存量(t)：</span>
                <div class="content">{{ item.sku || '100' }}</div>
              </div>
              <div class="left_text">
                <span>关键参数：</span>
                <div class="content" title="">
                  <div>
                    <div class="scroll-container" style="width: 327px; overflow: hidden;" @mouseenter="pauseScroll" @mouseleave="resumeScroll">
                      <div class="scroll-wrapper" :class="{ 'paused': isScrollPaused }">
                        <div class="scroll-item"><span>泥粉含量：{{ formatPercent(item.contentMudPowder) }}，泥块含量：{{ formatPercent(item.contentMud) }}，含水率：{{ formatPercent(item.contentWater) }}；</span></div>
                        <div class="scroll-item"><span>泥粉含量：{{ formatPercent(item.contentMudPowder) }}，泥块含量：{{ formatPercent(item.contentMud) }}，含水率：{{ formatPercent(item.contentWater) }}；</span></div>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
            <div class="silos_box_right">
              <div>
                <div class="right-bigbox">{{ item.code || 'H' + (index + 1) }}</div>
              </div>
            </div>
          </div>
          <div class="silos_box_bottom">
            <img class="silos_img" src="@/assets/logo1.png" alt="">中国建筑第七工程局有限公司
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import dayjs from "dayjs";

export default {
  data() {
    return {
      currentTime: '', // 存储当前时间
      silosList: [], // 存储料仓列表数据
      code: '', // 存储传入的料仓编号
      loading: false,
      isScrollPaused: false // 控制滚动暂停状态
    }
  },
  mounted() {
    // 获取URL参数中的code
    this.code = this.$route.query.code || this.$route.params.code;

    // 初始化时间
    this.updateTime();
    // 每秒更新一次时间
    this.timer = setInterval(() => {
      this.updateTime();
    }, 1000);

    // 获取料仓数据
    this.getSilosData();

    // 定时刷新数据（每30秒刷新一次）
    this.dataTimer = setInterval(() => {
      this.getSilosData();
    }, 30000);
  },
  beforeDestroy() {
    // 组件销毁前清除定时器
    clearInterval(this.timer);
    if (this.dataTimer) {
      clearInterval(this.dataTimer);
    }
  },
  watch: {
    // 监听路由变化，重新获取数据
    '$route'(to, from) {
      this.code = to.query.code || to.params.code;
      this.getSilosData();
    }
  },
  methods: {
    updateTime() {
      const now = new Date();
      // 格式化时间为 YYYY-MM-DD HH:mm:ss 格式
      this.currentTime = now.toLocaleString('zh-CN', {
        year: 'numeric',
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit',
        second: '2-digit',
        hour12: false
      }).replace(/\//g, '-');
    },

    // 获取料仓数据
    getSilosData() {
      // 如果有传入code，则获取单个料仓详情，否则获取所有料仓列表
      if (this.code) {
        this.getSingleSiloData();
      } else {
        this.getAllSilosData();
      }
    },

    // 获取单个料仓详情
    getSingleSiloData() {
      this.loading = true;
      this.$http({
        url: '/storageSilo/Equipment/page',
        method: 'get',
        params: {
          current: 1,
          size: 1,
          code: this.code // 通过code查询
        },
      }).then(({data}) => {
        this.loading = false;
        if (data.code === 0 && data.data.records && data.data.records.length > 0) {
          this.silosList = [data.data.records[0]]; // 只显示一个料仓
        } else {
          this.$message.error('未找到料仓信息');
          this.silosList = [{}];
        }
      }).catch(error => {
        this.loading = false;
        console.error('获取料仓详情失败:', error);
        this.$message.error('网络请求失败');
        this.silosList = [{}];
      });
    },

    // 获取所有料仓列表
    getAllSilosData() {
      this.loading = true;
      this.$http({
        url: '/storageSilo/Equipment/page',
        method: 'get',
        params: {
          current: 1,
          size: 100, // 获取更多数据用于展示
        },
      }).then(({data}) => {
        this.loading = false;
        if (data.code === 0) {
          this.silosList = data.data.records || [];
          // 如果没有数据，可以设置一个默认的空对象用于显示默认值
          if (this.silosList.length === 0) {
            this.silosList = [{}]; // 至少显示一个panel
          }
        } else {
          this.$message.error('获取料仓列表失败');
          this.silosList = [{}];
        }
      }).catch(error => {
        this.loading = false;
        console.error('获取料仓数据失败:', error);
        this.$message.error('网络请求失败');
        // 如果接口失败，设置一个默认对象
        this.silosList = [{}];
      });
    },

    // 格式化日期
    formatDate(date) {
      return date ? dayjs(date).format('YYYY-MM-DD') : '';
    },

    // 获取状态文本
    getStatusText(status) {
      switch(status) {
        case '0':
          return '合格';
        case '1':
          return '待检';
        default:
          return '';
      }
    },

    // 格式化百分比
    formatPercent(value) {
      if (!value && value !== 0) {
        return '/';
      }
      // 如果值已经包含%，直接返回
      if (typeof value === 'string' && value.includes('%')) {
        return value;
      }
      // 如果是数字，添加%符号
      if (typeof value === 'number' || !isNaN(parseFloat(value))) {
        return parseFloat(value).toFixed(1) + '%';
      }
      return value || '/';
    },

    // 刷新数据
    refreshData() {
      this.getSilosData();
    },

    // 返回列表页（如果需要）
    goBack() {
      this.$router.go(-1);
    },

    // 暂停滚动
    pauseScroll() {
      this.isScrollPaused = true;
    },

    // 恢复滚动
    resumeScroll() {
      this.isScrollPaused = false;
    }
  }
}
</script>

<style scoped>
.app {
  font-family: Avenir,Helvetica,Arial,sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  overflow: hidden;
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
}

.firstScreen{
  width: 100%;
  height: 100%;
  padding: 20px;
  box-sizing: border-box;
}

.silos{
  width: 100%;
  height: 100%;
  display: -webkit-box;
  display: -ms-flexbox;
  display: flex;
  -ms-flex-wrap: wrap;
  flex-wrap: wrap;
  gap: 20px;
  justify-content: center;
  align-items: flex-start;
  overflow-y: auto;
}

.silos_box{
  min-width: 420px;
  min-height: 210px;
  flex: 1;
  max-width: 500px;
  height: auto;
  min-height: 250px;
  padding: 20px 10px 10px 10px;
  background: url('~@/assets/bg3.6abf27d0.png') no-repeat 50%/100% 100%;
  background-size: cover;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0,0,0,0.1);
}

.silos_box_title{
  width: 94%;
  margin: 0 auto;
  height: 30px;
  background-color: #293a49;
  border: 1px solid #71afce;
  font-size: 14px;
  color: #fff;
  font-weight: 600;
  text-align: center;
  line-height: 30px;
  border-radius: 4px;
}

.silos_box_center{
  width: 94%;
  margin: 0 auto;
  margin-top: 15px;
  display: -webkit-box;
  display: -ms-flexbox;
  display: flex;
  min-height: 140px;
}

.silos_box_left{
  width: 88%;
  min-height: 140px;
  background-color: rgba(13,34,50,.4);
  border: 1px solid #71afce;
  display: -webkit-box;
  display: -ms-flexbox;
  display: flex;
  -ms-flex-wrap: wrap;
  flex-wrap: wrap;
  -webkit-box-align: center;
  -ms-flex-align: center;
  align-items: center;
  -webkit-box-pack: justify;
  -ms-flex-pack: justify;
  justify-content: space-between;
  color: #fff;
  padding: 10px 5px;
  border-radius: 4px;
}

.silos_box_right{
  height: 30px;
  margin: 0 0 0 5px;
  color: #fff;
}

.right-bigbox{
  width: 80px;
  height: 140px;
  background-color: rgba(13,34,50,.4);
  border: 1px solid #71afce;
  color: #ffd142;
  font-size: 48px;
  display: -webkit-box;
  display: -ms-flexbox;
  display: flex;
  -webkit-box-align: center;
  -ms-flex-align: center;
  align-items: center;
  -webkit-box-pack: center;
  -ms-flex-pack: center;
  justify-content: center;
  font-weight: 600;
  border-radius: 4px;
}

.silos_box_bottom{
  font-size: 14px;
  color: #fff;
  font-weight: 600;
  display: -webkit-box;
  display: -ms-flexbox;
  display: flex;
  -webkit-box-align: center;
  -ms-flex-align: center;
  align-items: center;
  margin: 12px 0 0 20px;
}

.silos_img {
  width: 18px;
  height: 20px;
  margin-right: 8px;
}

.left_text{
  display: -webkit-box;
  display: -ms-flexbox;
  display: flex;
  -webkit-box-align: center;
  -ms-flex-align: center;
  align-items: center;
  margin: 3px 0;
  width: 100%;
}

.left_text span {
  font-size: 13px;
  min-width: 80px;
  margin-right: 5px;
}

.content{
  font-size: 13px;
  flex: 1;
  white-space: nowrap;
  text-align: center;
  overflow: hidden;
  line-height: 22px;
  background-color: #4e6b79;
  border-radius: 5px;
  padding: 2px 4px;
  min-height: 22px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.scroll-container {
  width: 100%;
  overflow: hidden;
}

.scroll-wrapper {
  display: flex;
  width: 200%;
  animation: scroll 15s linear infinite;
}

.scroll-wrapper.paused {
  animation-play-state: paused;
}

.scroll-item {
  width: 50%;
  flex-shrink: 0;
  display: flex;
  justify-content: center;
  white-space: nowrap;
}

@keyframes scroll {
  0% {
    transform: translateX(0%);
  }
  100% {
    transform: translateX(-50%);
  }
}

/* 响应式设计 */
@media (max-width: 1200px) {
  .silos_box {
    min-width: 380px;
    max-width: 450px;
  }
}

@media (max-width: 768px) {
  .silos_box {
    min-width: 100%;
    max-width: 100%;
    margin: 10px 0;
  }
  
  .silos_box_center {
    flex-direction: column;
  }
  
  .silos_box_right {
    margin: 10px 0 0 0;
  }
  
  .right-bigbox {
    width: 100%;
    height: 60px;
    font-size: 36px;
  }
}

@media (max-width: 480px) {
  .firstScreen {
    padding: 10px;
  }
  
  .silos {
    gap: 10px;
  }
  
  .silos_box {
    padding: 15px 8px 8px 8px;
  }
  
  .silos_box_title {
    font-size: 12px;
    height: 25px;
    line-height: 25px;
  }
  
  .left_text span {
    font-size: 12px;
    min-width: 70px;
  }
  
  .content {
    font-size: 12px;
  }
}
</style>