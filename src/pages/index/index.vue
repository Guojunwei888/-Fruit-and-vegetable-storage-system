<template>
 <div class="wrapper">
    <div class="header-wrapper">
      <div class="header-title">
        <span>管理状态-{{ airText }}</span>
        <span>{{ area }}-{{ city }}</span>
      </div>
      <div class="header-text">
        <span>{{ airValue }}</span>
        <span>{{ weather }}</span>
      </div>
      <div class="weather-advice">{{ weatherAdvice }}</div>
    </div>
    <div class="body-wrapper">
      <div class="body">
        <div class="data-wrapper">
          <div class="data">
            <img class="data-logo" src="/static/images/wendu.png" />
            <div class="data-text">
              <div class="data-title">温度</div>
              <div class="data-value">{{ Temp }}℃</div>
            </div>
          </div>
          <div class="data">
            <img class="data-logo" src="/static/images/shidu.png" />
            <div class="data-text">
              <div class="data-title">湿度</div>
              <div class="data-value">{{ Hum }}%RH</div>
            </div>
          </div>
        </div>
        <div class="data-wrapper">
          <div class="data">
            <img class="data-logo" src="/static/images/yanwu.png" />
            <div class="data-text">
              <div class="data-title">烟雾浓度</div>
              <div class="data-value">{{ Air }}%PPM</div>
            </div>
          </div>
          <div class="data">
            <img class="data-logo" src="/static/images/led.png" />
            <div class="data-text">
              <div class="data-title">报警灯</div>
              <div class="data-value">
                <switch @change="onLedChange" :checked="Led" color="#3d7ef9" />
              </div>
            </div>
          </div>
        </div>
        <div class="data-wrapper">
          <div class="data">
            <img class="data-logo" src="/static/images/beep.png" />
            <div class="data-text">
              <div class="data-title">蜂鸣报警器</div>
              <div class="data-value">
                <switch
                  @change="onBeepChange"
                  :checked="Beep"
                  color="#3d7ef9"
                />
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { connect } from 'mqtt/dist/mqtt.js'; 
import { deprecate } from '../../../dist/wx/common/vendor';

/******************* 可能需要你修改的部分 ******************/
const mqttHost = "broker.emqx.io"; //mqtt 服务器域名/IP
const mqttPort = 8084; //mqtt 服务器域名/IP

const deviceSubTopic = "/mysmartMemory/sub"; //  设备订阅topic（小程序发布命令topic）
const devicePubTopic = "/mysmartMemory/pub"; //  设备发布topic（小程序订阅数据topic）

/********************* 一般不用动这些 ********************/

const mpSubTopic = devicePubTopic;
const mpPubTopic = deviceSubTopic;

const mqttUrl = `wxs://${mqttHost}:${mqttPort}/mqtt`; //  mqtt连接路径

export default {
  data () {
    return {
      client: {},
      Temp: 0,
      Hum: 0,
      Air: 0,
      Led: false,
      Beep: false,
      area: "黑龙江", //城区
      city: "哈尔滨", //城市
      airText: "正常", //仓库状态
      airValue: 22, //空气指数
      weather: "黑龙江科技大学", //学校
      weatherAdvice: "仓库运行一切正常", //仓库建议
    }
  },

  components: {
   
  },

  methods: { 
    onLedChange(event) {
      var that = this;
      console.log(event.mp.detail);
      let sw = event.mp.detail.value;
      that.Led = sw;
      if (sw) {
        that.client.publish(
          mpPubTopic,
          '{"target":"LED","value":0}',
          function (err) {
            if (!err) {
              console.log("成功下发命令——开灯");
            }
          }
        );
      } else {
        that.client.publish(
          mpPubTopic,
          '{"target":"LED","value":1}',
          function (err) {
            if (!err) {
              console.log("成功下发命令——关灯");
            }
          }
        );
      }
    },
    onBeepChange(event) {
      var that = this;
      console.log(event.mp.detail);
      let sw = event.mp.detail.value;
      that.Beep = sw;
      if (sw) {
        that.client.publish(
          mpPubTopic,
          '{"target":"BEEP","value":1}',
          function (err) {
            if (!err) {
              console.log("成功下发命令——打开报警器");
            }
          }
        );
      } else {
        that.client.publish(
          mpPubTopic,
          '{"target":"BEEP","value":0}',
          function (err) {
            if (!err) {
              console.log("成功下发命令——关闭报警器");
            }
          }
        );
      }
    },
  },

    // created () {
    // let app = getApp()
    // },
    onShow(){
      var that = this;
    wx.showToast({
      title: "连接服务器....",
      icon: "loading",
      duration: 10000,
      mask: true,
    });
    let second = 10;
    var toastTimer = setInterval(() => {
      second--;
      if (second) {
        wx.showToast({
          title: `连接服务器...${second}`,
          icon: "loading",
          duration: 1000,
          mask: true,
        });
      } else {
        clearInterval(toastTimer);
        wx.showToast({
          title: "连接失败",
          icon: "error",
          mask: true,
        });
      }
    }, 1000);

    that.client = connect(mqttUrl);
    that.client.on("connect", function () {
      console.log("成功连接mqtt服务器！");
      clearInterval(toastTimer);
      wx.showToast({
        title: "连接成功",
        icon: "success",
        mask: true,
      });
      // 一秒后订阅主题
      setTimeout(() => {
        that.client.subscribe(mpSubTopic, function (err) {
          if (!err) {
            console.log("成功订阅设备上行数据Topic!");
            wx.showToast({
              title: "订阅成功",
              icon: "success",
              mask: true,
            });
          }
        });
      }, 1000);
    });
    that.client.on("message", function (topic, message) {
      console.log(topic)
      let dataFromDev = {};
      dataFromDev = JSON.parse(message);
      console.log(dataFromDev);
      that.Temp = dataFromDev.Temp;
      that.Hum = dataFromDev.Hum;
      that.Air = dataFromDev.Air;
      that.Led = dataFromDev.Led;
      that.Beep = dataFromDev.Beep;

      
    });
  }
} 
</script>

<style lang="scss"scoped>
.wrapper {
  padding: 15px;
  .header-wrapper {
    background-color: #238E23;
    border-radius: 20px;
    color: #fff;
    box-shadow: #d6d6d6 0px 0px 5px;
    padding: 15px 30px;
    .header-title {
      display: flex;
      justify-content: space-between;
    }
    .header-text {
      font-size: 32px;
      font-weight: 400;
      display: flex;
      justify-content: space-between;
    }
    .weather-advice {
      margin-top: 20px;
      font-size: 12px;
    }
  }
    .data-wrapper {
    margin-top: 20px;
    display: flex;
    justify-content: space-between;
    .data {
      background-color: #fff;
      width: 150px;
      height: 80px;
      border-radius: 20px;
      display: flex;
      justify-content: space-around;
      padding: 0 8px;
      box-shadow: #d6d6d6 0px 0px 5px;
      .data-logo {
        height: 36px;
        width: 36px;
        margin-top: 15px;
      }
      .data-text {
        margin-top: 15px;
        color: #7f7f7f;
        .data-title {
          text-align: right;
        }
        .data-value {
          font-size: 26px;
        }
      }
    }
  }
}
</style>
