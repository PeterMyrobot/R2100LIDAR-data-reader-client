<template>
  <div class="rootDiv">
    <h1>Li-dar showing</h1>
    <div class="websoketDiv">
      <p>第一步： 打開python執行檔，並連接websocket</p>
      <div class="buttonContain">
        <button @click="connectSocket" :disabled="websocketConnect">Connect to Sokect</button>
        <div :class="{ socketStatus: true, connected: websocketConnect }">
          websocket連接狀態：{{ websocketConnect ? "Connect" : "Not connnect" }}
        </div>
      </div>
    </div>
    <div class="serialRoot">
      <div class="serial">
        <p>第二步： 偵測已連接上的port，並選擇port連接</p>
        <button @click="requestPorts">Detecing port</button>
        <div class="grow">
          <div class="selectDiv">
            <div class="params">
              Port:
              <select v-model="portName">
                <option v-for="port in ports" :value="port" :key="port">{{ port }}</option>
              </select>
            </div>
            <div class="params">
              buadrate:
              <select v-model="baudrate">
                <option v-for="baudrate in baudrates" :value="baudrate" :key="baudrate">{{
                  baudrate
                }}</option>
              </select>
            </div>
          </div>
          <button @click="connectSerail">connect</button>
        </div>
        <div :class="{ socketStatus: true, connected: serialConnect }">
          Serial 連接狀態：{{ serialConnect ? "Connect" : "Not connnect" }}
        </div>
      </div>
      <div class="command">
        <p>第三步： 輸入指令並按下發送測試回傳資料</p>
        <!-- <button @click="sendMsgToSocket">Send</button>
        <input type="text" v-model="msg" /> -->
        <div class="hex">
          Command: <input type="text" v-model="hexMsg" class="hexCmd" />
          <button @click="testhex">Send</button>
        </div>
        <div class="dataDiv">
          感測器回傳：<span class="data">{{ parseHex }}</span>
        </div>
      </div>
    </div>
    <div class="canvas">
      <canvas class="myCanvas" id="myCanvas" ref="myCanvas"></canvas>
    </div>
    <button @click="parseDistanceData">clear</button>
    <button @click="drawLine">draw</button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      ws: null,
      url: 'ws://localhost:5000',
      websocketConnect: false,
      serialConnect: false,
      ports: [],
      baudrates: [57600, 115200],
      hexMsg: '',
      portName: '',
      baudrate: '57600',
      cmds: {
        getPortsList: 'getPortsList',
        connectPort: 'connectPort',
        pulling: 'pulling',
      },
      res: {
        portsList: 'portsList',
        connectedPort: 'connectedPort',
        distData: 'distData',
      },
      distData: '',
      scale: 10,
      curindex: 0,
      fakeData: [
        '01 de 32 11 34 0a 4a 11 b7 09 9e 12 f7 08 57 13 5c 08 e0 14 18 08 4f 16 f1 07 5f 16 f6 07 2b 17 ca 06 93 15 6b 04 b9 17 6e 03 08 1a cf 02 11 1c 1c a3',
        '01 de 32 11 26 0a 5b 11 c6 09 9b 12 f4 08 66 13 5a 08 d5 14 13 08 63 16 f5 07 55 16 fa 07 32 17 ca 06 8d 15 69 04 c9 17 6d 03 1d 1a ce 02 0b 1c 1c 88',
        '01 de 32 11 21 0a 8c 11 c7 09 6e 12 ed 08 5a 13 5e 08 f4 14 19 08 52 16 f3 07 4f 16 fc 07 37 17 b0 06 bc 15 6e 04 c0 17 74 03 02 1a cb 02 06 1c 1c c3',
        '01 de 32 11 23 0a 4e 11 cb 09 9e 12 f3 08 8f 13 64 08 db 14 19 08 52 16 e8 07 58 16 f7 07 27 17 c7 06 93 15 69 04 c0 17 6f 03 fe 19 d5 02 02 1c 1d 96',
        '01 de 32 11 21 0a 77 11 be 09 a1 12 f5 08 4a 13 64 08 e0 14 22 08 48 16 ee 07 54 16 f8 07 32 17 b3 06 a5 15 71 04 99 17 78 03 fa 19 d3 02 09 1c 1c 32',
        '01 de 32 11 27 0a 3a 11 ca 09 a3 12 ee 08 6a 13 58 08 f8 14 1a 08 5b 16 eb 07 52 16 02 08 34 17 aa 06 b1 15 6b 04 b6 17 76 03 06 1a cd 02 10 1c 1c 05',
        '01 de 32 11 32 0a 50 11 c3 09 90 12 f4 08 61 13 5e 08 de 14 15 08 52 16 eb 07 4d 16 f7 07 37 17 b0 06 b6 15 6b 04 b2 17 72 03 11 1a d1 02 09 1c 1c 9e',
        '01 de 32 11 2c 0a 58 11 d4 09 83 12 f9 08 66 13 5d 08 f5 14 15 08 4b 16 f1 07 42 16 07 08 26 17 b9 06 a7 15 70 04 bc 17 77 03 fd 19 d3 02 06 1c 1c a6',
        '01 de 32 11 28 0a a3 11 ce 09 82 12 f7 08 52 13 63 08 e1 14 15 08 5d 16 f1 07 44 16 f5 07 25 17 bd 06 ba 15 71 04 bf 17 76 03 08 1a cd 02 0c 1c 1c 44',
        '01 de 32 11 2a 0a 7d 11 c5 09 72 12 fc 08 4b 13 5f 08 e4 14 15 08 5e 16 ec 07 40 16 fe 07 2a 17 c8 06 8f 15 6a 04 cf 17 74 03 17 1a cc 02 0b 1c 1c 66',
        '01 de 32 11 31 0a 82 11 c5 09 7d 12 f4 08 4b 13 67 08 eb 14 1a 08 55 16 ec 07 4a 16 0b 08 1c 17 cc 06 9d 15 72 04 b9 17 6a 03 12 1a ca 02 0f 1c 1c 11',
        '01 de 32 11 1c 03 a0 16 d9 01 7b 1d c9 03 47 18 35 08 0e 15 05 08 4c 16 f1 07 31 16 f5 07 24 17 b3 06 a3 15 68 04 af 17 71 03 15 1a d0 02 0e 1c 12 67',
        '01 de 32 11 e7 04 89 16 5b 03 a0 18 9d 01 5e 1d 84 01 70 22 52 03 29 1b c8 07 69 16 b0 07 5e 17 b4 06 87 15 64 04 a6 17 72 03 fe 19 cd 02 02 1c 0f fc',
        '01 de 32 11 06 0a 31 11 85 09 57 12 c8 05 7e 16 42 03 f9 19 b6 01 dc 1f 72 01 82 23 9d 01 87 1f 6d 03 24 18 49 04 9b 17 72 03 eb 19 cb 02 f6 1b 0e 85',
        '01 de 32 11 0d 0a 23 11 c2 09 6b 12 e0 08 2c 13 39 08 f1 14 8f 07 a7 16 8b 04 63 19 9e 01 8f 1f 46 01 f6 20 5d 01 e0 23 c4 01 53 1b b4 02 22 1c 0d bd',
      ],
    };
  },
  mounted() {
    this.connectSocket();
    this.canvas.height = 600;
    this.canvas.width = 800;
  },
  computed: {
    parseHex() {
      if (this.distData !== '') {
        const str = this.distData.match(/.{1,2}/g);
        return str.join(' ');
      }
      return '';
    },
    canvas() {
      return this.$refs.myCanvas;
    },
    ctx() {
      return this.canvas.getContext('2d');
    },
  },
  methods: {
    requestPorts() {
      const requestPortsMsg = { cmd: 'getPortsList' };
      this.ws.send(JSON.stringify(requestPortsMsg));
    },
    connectSocket() {
      if (!this.ws) {
        this.ws = new WebSocket(this.url);
        console.log(this.ws);
        this.ws.addEventListener('open', () => {
          console.log('連結建立成功。');
          this.websocketConnect = true;
        });
        this.ws.addEventListener('message', (e) => {
          // const msg = JSON.parse(e.data);
          // console.log(e.data);
          this.parseResponse(e.data);
        });
        this.ws.addEventListener('close', () => {
          console.log('連結關閉，咱們下次見~');
          this.ws = null;
          this.websocketConnect = false;
        });
      }
    },
    parseResponse(data) {
      const command = JSON.parse(data);
      console.log(command);
      if (command.cmd === this.res.portsList) {
        this.ports = command.data;
        if (this.ports.length) {
          [this.portName] = this.ports;
        } else {
          this.portName = '';
        }
      } else if (command.cmd === this.res.distData) {
        this.distData = command.data;
      } else if (command.cmd === this.res.connectedPort) {
        this.serialConnect = true;
      }
    },
    sendMsgToSocket() {
      const msg = { cmd: 'sendMsg', msg: this.msg };
      this.ws.send(JSON.stringify(msg));
    },
    connectSerail() {
      const msg = { cmd: 'openSerial', portName: this.portName, baudrate: this.baudrate };
      this.ws.send(JSON.stringify(msg));
    },
    testhex() {
      const msg = { cmd: 'hexMsg', msg: this.hexMsg };
      this.ws.send(JSON.stringify(msg));
    },

    clear() {
      this.ctx.fillStyle = '#ffffff';
      // this.ctx.translate(-this.canvas.width / 2, -this.canvas.height / 2);
      this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    },
    parseDistanceData() {
      console.log(this.curindex);
      const distance = this.fakeData[this.curindex].split(' ');
      // console.log(distance.slice(4));
      const lights = [];
      for (let i = 1; i < 12; i += 1) {
        const dis = distance[i * 4 + 1] + distance[i * 4];
        const echo = distance[i * 4 + 3] + distance[i * 4 + 2];
        lights.push({ dis, echo });
      }
      const lightValue = lights.map((el) => ({
        dis: parseInt(el.dis, 16),
        echo: parseInt(el.echo, 16),
      }));
      // console.log(lightValue);
      this.clear();
      this.drawTriangles(lightValue);
      this.curindex = (this.curindex + 1) % this.fakeData.length;
    },
    drawLine(dis) {
      console.log('draw line');
      this.clear();
      this.ctx.strokeStyle = '#ff0000';
      // this.ctx.translate(400, 300);
      this.ctx.beginPath();
      for (let i = 0; i < dis.length; i += 1) {
        const ang = this.toRadians(50 + i * 8);
        const x = 400 + (dis[i].dis / this.scale) * Math.cos(ang);
        const y = 600 - (dis[i].dis / this.scale) * Math.sin(ang);
        this.ctx.moveTo(400, 600);
        this.ctx.lineTo(x, y);
      }
      this.ctx.lineWidth = 5;
      this.ctx.stroke();
    },
    drawTriangles(dis) {
      console.log(dis);
      for (let i = 0; i < dis.length; i += 1) {
        const ang = 50 + i * 8;
        this.drawTriangle(dis[i].dis, ang);
      }
    },
    drawTriangle(dis, ang) {
      console.log('draw line', dis, ang);
      // this.clear();
      this.ctx.fillStyle = '#fe092a';
      this.ctx.beginPath();
      this.ctx.moveTo(400, 600);
      const ang1 = this.toRadians(ang + 4);
      const ang2 = this.toRadians(ang - 4);
      const firstPoint = [
        400 + (dis / this.scale) * Math.cos(ang1),
        600 - (dis / this.scale) * Math.sin(ang1),
      ];
      const secondPoint = [
        400 + (dis / this.scale) * Math.cos(ang2),
        600 - (dis / this.scale) * Math.sin(ang2),
      ];
      this.ctx.lineTo(firstPoint[0], firstPoint[1]);
      this.ctx.lineTo(secondPoint[0], secondPoint[1]);
      this.ctx.closePath();
      this.ctx.fill();
    },
    toRadians(angle) {
      return angle * (Math.PI / 180);
    },
  },
};
</script>

<style scoped>
.rootDiv {
  width: 100%;
  max-width: 1200px;
}
.websoketDiv {
  padding: 20px 0px;
  border-bottom: 1px dashed;
}
.socketStatus {
  color: red;
  font-size: 1.5rem;
  font-weight: bold;
}
.connected {
  color: green;
}
.serialRoot {
  display: flex;
  flex-direction: row;
  padding: 2rem;
}
.serial {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  min-width: 300px;
  padding: 0px 20px;
}
.grow {
  display: flex;
  margin-top: 10px;
  align-items: center;
  width: 100%;
  justify-content: space-between;
}
.selectDiv {
  display: flex;
  flex-grow: 1;
  margin-right: 10px;
  flex-direction: column;
}
.command {
  flex-grow: 1;
  padding: 0 20px;
}
.params {
  flex-grow: 1;
  display: flex;
  padding-bottom: 10px;
}
.hex {
  width: 100%;
  display: flex;
  justify-content: space-between;
  font-size: 1.5rem;
}
.hexCmd {
  flex-grow: 1;
  margin: 0px 10px;
}
.dataDiv {
  width: 100%;
  margin-top: 20px;
  display: flex;
  border-top: 1px solid;
  padding-top: 10px;
}
.data {
  flex-grow: 1;
}
.myCanvas {
  border: 1px dashed;
}
button {
  border: none;
  background-color: #0888af;
  color: white;
  padding: 5px 10px;
  height: 30px;
  border-radius: 3px;
  cursor: pointer;
  opacity: 0.9;
}
button:hover {
  opacity: 1;
}
button:disabled {
  cursor: not-allowed;
  opacity: 0.3;
}
select {
  border: none;
  border-bottom: 1px solid;
  margin-left: 10px;
  flex-grow: 1;
  text-align-last: center;
}
select:focus {
  outline: none;
}
</style>
