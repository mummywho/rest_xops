<template>
  <div :style="'width:' + width" class="container">
    <el-row :gutter="32">
      <el-col :span="10">
        <el-card class="box-card">
          <span>项目管理</span>
          <el-form ref="project" :model="project_data" :rules="rules" size="small" label-width="80px">
            <el-form-item label="服务器" prop="server_ids">
              <el-select v-model="project_data.server_ids" placeholder="请选择服务器">
                <el-option v-for="item in hosts" :key="item.key" :label="item.label" :value="item.key"/>
              </el-select>
            </el-form-item>
            <el-form-item>
              <div style="display: inline-block;margin: 0px 1px;">
                <el-button :loading="startloading" size="mini" type="success" icon="el-icon-share" @click="doStart">启动</el-button>
                <el-button :loading="stoploading" size="mini" type="primary" icon="el-icon-plus" @click="doStop">停止</el-button>
              </div>
            </el-form-item>
          </el-form>
        </el-card>
      </el-col>
      <el-col :span="14">
        <el-card class="box-card">
          <span>日志管理</span>
          <el-form ref="log" :model="log_data" :rules="rules" size="small" label-width="80px">
            <el-form-item label="服务器" prop="server_ids">
              <el-select v-model="log_data.server_ids" placeholder="请选择服务器">
                <el-option v-for="item in hosts" :key="item.key" :label="item.label" :value="item.key"/>
              </el-select>
            </el-form-item>
            <el-form-item>
              <div style="display: inline-block;margin: 0px 1px;">
                <el-button :loading="tailloading" size="mini" type="success" icon="el-icon-share" @click="doTailStart">启动</el-button>
                <el-button size="mini" type="primary" icon="el-icon-plus" @click="doTailStop">停止</el-button>
              </div>
            </el-form-item>
          </el-form>
        </el-card>
      </el-col>
    </el-row>
    <el-card class="box-card">
      <el-tooltip content="返回上一页" class="closepage item" effect="dark" placement="left">
        <el-button type="info" size="mini" circle @click="closeTag"><svg-icon icon-class="return"/></el-button>
      </el-tooltip>
      <el-tooltip :content="content" class="lock item" effect="dark" placement="left">
        <el-button type="info" size="mini" circle @click="doLock"><svg-icon :icon-class="ico"/></el-button>
      </el-tooltip>
      <div id="console" :style="'height:'+ height" class="console">
        <div v-for="item in data" :key="item.id">
          <span class="line-html" v-html="item"/>
        </div>
      </div>
    </el-card>
  </div>
</template>

<script>
import Vue from 'vue'
import { DeployExcu } from '@/api/deploy'
import { parseTime } from '@/utils/index'
import { getToken } from '@/utils/auth'
import { getDeviceList } from '@/api/device'
import { retrieve } from '@/api/project'
import VueNativeSock from 'vue-native-websocket'
export default {
  data() {
    return {
      ico: 'unlock', unlock: true, content: '锁定滚动条',
      height: document.documentElement.clientHeight - 94.5 + 'px;',
      width: document.documentElement.clientWidth - 185 + 'px;',
      startloading: false,
      stoploading: false,
      tailloading: false,
      servers: '',
      project_data: {
        server_ids: ''
      },
      hosts: [],
      log_data: {
        server_ids: ''
      },
      data: [],
      vm: null,
      rules: {
        server_ids: [
          { required: true, message: '请选择服务器', trigger: 'blur' }
        ]
      }
    }
  },
  // 监听控制滚动条
  watch: {
    data: {
      handler(val, oldVal) {
        this.$nextTick(() => {
          if (this.unlock) {
            var div = document.getElementById('console')
            div.scrollTop = div.scrollHeight
          }
        })
      }
    }
  },
  created() {
    this.init()
    setTimeout(() => {
      getDeviceList('Linux').then(res => {
        const _servers = this.servers.split(',')
        const _hosts = this.hosts
        const _project_data = this.project_data
        const _log_data = this.log_data
        res.forEach(function(data, index) {
          for (const i of _servers) {
            if (i === data.key) {
              _hosts.push(data)
            }
          }
        })
        _project_data.server_ids = _servers[0]
        _log_data.server_ids = _servers[0]
      })
    }, 200)
  },
  mounted: function() {
    this.initWebSocket()
    this.start()
  },
  beforeDestroy: function() {
    this.vm.$disconnect()
    console.log('---离开页面关闭Websocket---')
  },
  methods: {
    parseTime,
    initWebSocket() {
      const wsuri = 'ws://' + process.env.BASE_API.replace(/http:\/\//, '') + '/websocket/console?token=' + getToken()
      Vue.use(VueNativeSock, wsuri, {
        // format: 'json',
        connectManually: true,
        reconnection: true,
        reconnectionAttempts: 5,
        reconnectionDelay: 3000
      })
      this.vm = new Vue()
      this.vm.$connect()
      console.log('---连接Websocket----')
    },
    init() {
      retrieve(this.$route.query.id).then(res => {
        this.project_data.app_start = res.app_start
        this.project_data.app_stop = res.app_stop
        this.project_data.app_bin_path = res.app_bin_path
        this.project_data.server_ids = res.server_ids
        this.servers = res.server_ids
        this.log_data.app_log_file = res.app_log_file
        this.log_data.server_ids = res.server_ids
      })
    },
    start() {
      this.$socket.onopen = (data) => {
        if (data.target.readyState === 1) {
          console.log('--------正在读取数据!--------')
          this.$socket.onmessage = (data) => {
            const color_data = this.getColor(data.data)
            this.data.push(color_data)
          }
          // 开始读取
        }
      }
    },
    doStart() {
      this.$refs['project'].validate((valid) => {
        if (valid) {
          this.startloading = true
          const form = {
            excu: 'app_start',
            host: this.project_data.server_ids,
            app_start: this.project_data.app_start
          }
          DeployExcu(form).then(res => {
            this.startloading = false
          }).catch(err => {
            this.startloading = false
            console.log(err)
          })
        } else {
          return false
        }
      })
    },
    doStop() {
      this.$refs['project'].validate((valid) => {
        if (valid) {
          this.stoploading = true
          const form = {
            excu: 'app_stop',
            host: this.project_data.server_ids,
            app_stop: this.project_data.app_stop
          }
          DeployExcu(form).then(res => {
            this.stoploading = false
          }).catch(err => {
            this.stoploading = false
            console.log(err)
          })
        } else {
          return false
        }
      })
    },
    doTailStart() {
      this.$refs['log'].validate((valid) => {
        if (valid) {
          this.tailloading = true
          const form = {
            excu: 'tail_start',
            host: this.log_data.server_ids,
            app_log_file: this.log_data.app_log_file
          }
          DeployExcu(form).then(res => {
            this.tailloading = false
          }).catch(err => {
            this.tailloading = false
            console.log(err)
          })
        } else {
          return false
        }
      })
    },
    doTailStop() {
      this.$refs['log'].validate((valid) => {
        if (valid) {
          const form = {
            excu: 'tail_stop'
          }
          DeployExcu(form).then(res => {
            this.$message({
              showClose: true,
              type: 'success',
              message: '执行成功!请等待线程结束.',
              duration: 3500
            })
          }).catch(err => {
            this.tailloading = false
            console.log(err)
          })
        } else {
          return false
        }
      })
    },
    closeTag() {
      this.$confirm('是否关闭页面并返回上一页?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.vm.$disconnect()
        this.$router.go(-1)
        this.$store.dispatch('delView', this.$store.state.tagsView.visitedViews.slice(-1)[0])
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '取消操作'
        })
      })
    },
    getColor(text) {
      const info = text.match(/INFO/i)
      const warn = text.match(/WARN/i)
      const debug = text.match(/DEBUG/i)
      const host = text.match(/\[(.*?)@(.*?)\]/)
      const error = text.match(/ERROR/i)
      if (info) {
        return text.replace(/INFO/i, '<span style="color: #7FFF00;">' + info + '</span>')
      } else if (warn) {
        return text.replace(/WARN/i, '<span style="color: #FFFF00;">' + warn + '</span>')
      } else if (debug) {
        return text.replace(/DEBUG/i, '<span style="color: #0000ff;">' + debug + '</span>')
      } else if (host) {
        return text.replace(/\[(.*?)@(.*?)\]/, '<span style="color: #FFA54F;">' + host[0] + '</span>')
      } else if (error) {
        return text.replace(/ERROR/i, '<span style="color: #FF0000;">' + error + '</span>')
      } else {
        return text
      }
    },
    doLock() {
      if (this.unlock) {
        this.content = '解除锁定'
        this.ico = 'lock'
      } else {
        this.content = '锁定滚动条'
        this.ico = 'unlock'
      }
      this.unlock = !this.unlock
    }
  }
}
</script>

<style scoped>
button,
input,
textarea {
  outline: 0;
}
.line-html {
  line-height: 1.85
}
.container .buttons .closes,
.container .buttons .maximize,
.container .buttons .minimize {
  padding: 0;
  margin: 0;
  margin-right: 6px;
  width: 12px;
  height: 12px;
  border: 1px solid transparent;
  border-radius: 6px;
}
.container {
  width: 100%;
  margin: 5px;
}
.container .console {
  font-family: consolas;
  overflow-y: scroll;
  background: #494949;
  color: #f7f7f7;
  padding: 10px;
  font-size: 14px;
}
.lock {
  position: fixed;
  right: 45px;
  bottom: 6.8%;
  z-index: 100000;
}
.closepage {
  margin: 40px;
  position: fixed;
  right: 5px;
  bottom: 6.8%;
  z-index: 100000;
}
</style>
