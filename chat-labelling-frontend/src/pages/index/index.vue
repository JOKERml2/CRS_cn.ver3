<template>
  <div>
<!--    <button @click="openPopup">打开弹窗</button>-->
    <Modal v-model="showInstructions" draggable scrollable title="操作说明">
      <div v-html="instructions_cus" v-if="role === 'cus'"></div>
      <div v-html="instructions_sys" v-if="role === 'sys'"></div>
      <div slot="footer">
        <Button type="primary" @click="showInstructions=false">关闭</Button>
      </div>
    </Modal>
    <profile v-show="needProfile" @completed="profileCompleted"/>
    <Card :bordered="false" dis-hover v-show="!finished&&!needProfile">
      <p slot="title" style="text-align:center;overflow:hidden">
        <strong slot="title">
          <Icon type="ios-chatbubbles-outline" :size="20" />
          当前用户是{{username}}，正在与用户{{partner}}聊天。
        </strong>
      </p>
      <div slot="extra">
        <Button type="text" @click="showInstructions=true">
          <Icon type="ios-information-circle-outline" />
          <span style="font-weight: bold; color: red;">操作说明</span>
        </Button>
      </div>
      <Spin size="large" v-if="loading" fix>
        <Icon type="ios-loading" size="50" class="spin-icon-load"></Icon>
        <div>{{message}}, please don't leave...</div>
      </Spin>
      <div>
        <Row :gutter="8">
          <Col :span="10">
            <chatUI ref="chatUI" :style="{height:(height+35)+'px',overflow:'auto'}" />
          </Col>
          <Col :span="14">
            <Alert type="warning" v-if="!finished">
              <strong>注意:</strong>
              <ul style="padding:0 10px;line-height:150%;font-size:80%">
                <li>
                  刷新页面可能会<strong>终止</strong>对话！
                </li>
                <li v-if="disabled">
                  请等待您聊天伙伴的消息。
                  <Button
                    style="margin-left:10px"
                    type="primary"
                    size="small"
                    @click="hurryup"
                    :disabled="hurryupLeftSeconds>0"
                  >催促您的对话对象{{hurryupLeftSeconds>0? ('('+hurryupLeftSeconds+'s)'):''}}</Button>
                </li>
              </ul>
            </Alert>
            <Popup ref="popup" :touser="touser" :touuid="touuid" @stop-timer="stopTimer"></Popup>
            <labelling
              :height="(height-20-(role=='cus'? 60:0))+'px'"
              ref="labelling"
              :role="role"
              :currentState="currentState"
              :searchData="searchData"
              :sendData="sendData"
              :sendDisabled="disabled"
              :actions="actions"
              :conversationId="conversationId"
              :finish="finish"
              :searchResultConfig="searchResultConfig"
              :searching="searching"
              :searchPanelLoading="searchPanelLoading"
              @on-filters="searchWithFilters"
              @states-backup="saveStatesBackup"
              @changeBackupForParent="saveChangeBackup"
              @getRecommendInfo="getRecommendInfo"
              @changeisrecommend="changeisrecommend"
              @passselected="passselected"
              :bus="bus"
            />
          </Col>
        </Row>
      </div>
    </Card>
    <rating v-show="finished&&!needProfile" :recommend_info="recommend_info" :finished="finish" :need-profile="needProfile" :isrecommend="isrecommend" :role="role" :userId="username" :conversationId="conversationId"  @completed="ratingCompleted"/>
  </div>
</template>
<script>

import chatUI from './chatUI'
import profile from './profile'
import labelling from './labelling'
import rating from './rating'
import util from '../../utils/util'
import Vue from 'vue'
import axios from 'axios'
import md5 from 'crypto-js/md5'
import Popup from './Popup.vue'
const maxHurryupLeftSeconds = 60
export default {
  components: {
    chatUI,
    labelling,
    rating,
    profile,
    Popup
  },
  mounted () {
    this.init()
    this.startTimer()
    window.addEventListener('beforeunload', e => this.beforeunloadFn(e))
  },
  destroyed () {
    window.removeEventListener('beforeunload', e => this.beforeunloadFn(e))
    this.$socket.close()
  },
  data () {
    return {
      raw_info: [],
      add_info: [],
      dialogVisible: false,
      touuid: '', // 你的用户数据
      torole: '',
      touser: '',
      searching: false,
      searchPanelLoading: false,
      recommend_info: [],
      height: window.screen.availHeight - 230,
      loading: false,
      message: '',
      username: '',
      partner: '',
      conversationId: '',
      disabled: false,
      role: '',
      hurryupLeftSeconds: 0,
      actions: [],
      currentState: [],
      finished: false,
      needProfile: false,
      background: '',
      instructions: '',
      showInstructions: false,
      searchResultConfig: {},
      statesBackupList: [],
      bus: new Vue(),
      searchResultsBackup: {},
      selectedResultsBackup: [],
      isrecommend: false,
      instructions_sys: '感谢您同意参加此研究，您将扮演普通用户或系统角色进行对话。对话内容包括寻找一款产品（普通用户角色）或向用户推荐产品（系统角色）。<br>请仔细观看以下操作介绍视频：<a href="https://www.bilibili.com/video/BV1xN4y1Q7PY/" target="_blank">chat-labelling平台系统用户操作说明(1分56秒)。</a>如视频不清晰，请在视频观看页面调节至高清版本观看。 <br><br>要完成此研究，请按照以下步骤操作。<br>1. 我们想了解您的一些一般信息，因此，在对话开始前，请您填写关于个人信息的调查问卷，其中的问题包括性别、年龄和专业等。<br>2. 接下来，您将与用户进行对话。请等待用户发送第一条消息。在您收到用户消息后，您需要首先选择一个“意图”。在选择“意图”时，您可以将鼠标悬停在“意图”图标旁边的“？”上以查看每个“意图”的解释。然后，在“回复”中编写一条消息（通过单击“发送”按钮）。如果您想要在一轮中发送多条消息，请确保在发送消息之前选中“发送另一条消息”。<br>3. 在每轮对话中，您可以在“查询”框内输入内容以搜索与产品相关的结果。在必要时，您还可以单击“显示过滤器选项”以查看筛选条件，并基于筛选条件搜索更相关的产品。请注意，搜索引擎可能会有些慢，请在提交查询后耐心等待结果。<br>4. 当您认为有产品满足用户的需求时，您可以在“意图”中选择“推荐产品”，并从搜索结果面板中选择相应的产品（通过选中产品的搜索结果旁边的复选框）以推荐给用户。在“回复”文本框中写下一条消息（例如：“以下选择如何？”），您选择的推荐产品将自动添加到您的文本消息中以发送给用户。<br>5. 当您认为需要了解更多关于用户需求的信息时，您可以在“意图”中选择“问用户一个需求问题”，并从结果面板中选择相应的“明晰问题提问词”（通过选中“明晰问题提问词”旁边的复选框），并在“回复”文本框中提出有关“明晰问题提问词”的问题（例如，您更喜欢哪个操作系统？“操作系统”是您要提问的明晰问题提问词）。<br>6. 在用户完成对话并单击“结束对话”按钮后，您需要回答一些调查问卷以评价对话。',
      instructions_cus: '感谢您同意参加此研究，您将扮演普通用户或系统角色进行对话。对话内容包括寻找一款产品（普通用户角色）或向用户推荐产品（系统角色）。<br>请仔细观看以下操作介绍视频：<a href="https://www.bilibili.com/video/BV12j411C77L/" target="_blank">chat-labelling平台普通用户操作说明（1分09秒）</a>如视频不清晰，请在视频观看页面调节至高清版本观看。<br><br>要完成此研究，请按照以下步骤操作。<br>1. 我们想了解您的一些一般信息，因此，在对话开始前，请您填写关于个人信息的调查问卷，其中的问题包括性别、年龄和专业等。<br>2. 接下来，您将与系统进行对话。对话将由您发起。您需要首先选择一个“意图”，然后在“回复”中编写一条消息以发送给系统（通过单击“发送”按钮）。在选择“意图”时，您可以将鼠标悬停在“意图”图标旁边的“？”上以查看每个“意图”的解释。如果您想要在一轮中发送多条消息，请确保在发送消息之前选中“发送另一条消息”。<br>3. 在您发送消息后，请等待系统回复或向您推荐产品。在收到系统消息后，您可以重复第2步以继续对话。<br>4. 当您认为对话已经完成时，您可以单击“结束对话”以结束对话。<br>5. 在单击“结束对话”后，您需要<strong>评价对话中系统推荐的产品的与您目标产品的相关性</strong>。因此，请确保您了解推荐产品与您的目标产品的相关性。'
    }
  },
  methods: {
    changeisrecommend () {
      this.isrecommend = true
      // this.getRecommendInfo()
      console.log(this.recommend_info)
    },
    openPopup () {
      this.$refs.popup.openPopup()
      // this.do()
    },
    passselected (selected) {
      // this.raw_info = selected
      console.log('ffffffffffffffffffffffffffffffffffffff')
      console.log(selected)
      this.raw_info = selected.map(item => item.title)
      // for (let i = 0; i < selected.length; i++) {
      //   console.log('item')
      //   console.log(selected[i])
      //   console.log(selected[i].title)
      //   if (!this.add_row.includes(selected[i].title)) {
      //     // 如果不在，将其添加到 this.add_row 中
      //     this.add_row.push(selected[i].title)
      //   }
      // }
      for (let i = 0; i < this.raw_info.length; i++) {
        if (!this.add_info.includes(this.raw_info[i])) {
          this.add_info.push(this.raw_info[i])
        }
      }
      console.log('raw')
      console.log(this.raw_info)
      console.log('add')
      console.log(this.add_info)
      this.recommend_info = this.add_info
    },
    async do () {
      // 在这里执行你的操作
      console.log('定时触发的操作')
      console.log('this role:' + this.role)
      // eslint-disable-next-line no-unused-vars
      const response = await this.$http.get('/api/checkpal', {
        params: {
          role: this.role
        }
      })
      console.log('role is :' + response.data.role)
      if (response.data.name === 'none') {
        return
      }
      this.touser = response.data.name
      // eslint-disable-next-line no-unused-vars
      let usrURL = 'http://localhost:9191/login?name=' + response.data.name + '&password=' + response.data.name + '&role=' + response.data.role + '&action=' + 'login'
      axios.post(
        usrURL
      ).then((json) => {
        this.$http.post('/checkLogin',
          'username=' + response.data.name + '&password=' + response.data.name,
          {headers: {'Content-Type': 'application/x-www-form-urlencoded'}}
        ).then((response) => {
          if (response.data.code === 200) {
            // window.location = response.data.profile ? '/index?_uuid=' + response.data.uuid + '&_user=' + this.formInline.user : '/profile?_uuid=' + response.data.uuid + '&_user=' + this.formInline.user
            // window.location = '/index?_uuid=' + response.data.uuid + '&_user=' + this.formInline.user
            this.touuid = response.data.uuid
            this.$refs.popup.openPopup()
            // console.log(response.data)
            // const queryParams = new URLSearchParams(data).toString()
            // console.log(this.formInline.role)
          } else {
            this.$Message.error({
              content: response.data.msg,
              duration: 10,
              closable: true
            })
            this.submitting = false
          }
        }).catch((e) => {
          console.log(e)
          this.submitting = false
          this.$Message.error('Something wrong when sending data!')
        })
      }).catch((error) => {
        // debugger
        console.log(error)
        this.$Message.error('Something wrong when sending data!')
      })
    },
    startTimer () {
      // 当 partner 不为空时，执行定时器相关的代码
      this.timer = setInterval(() => {
        console.log('partner:' + this.partner)
        if (this.partner !== '') {
          this.do() // 调用定时触发的函数
        }
      }, 50000) // 30秒 = 30000毫秒
    },
    stopTimer () {
      // 停止定时器
      clearInterval(this.timer)
    },
    beforeDestroy () {
      // 在组件销毁前停止定时器，以防止内存泄漏
      this.stopTimer()
    },
    async getRecommendInfo () {
      try {
        // 调用后端API来获取搜索结果消息
        const response = await this.$http.get('/api/getSearchedMessage', {
          params: {
            conversationId: this.conversationId
          }
        })
        if (response.data !== null) {
          this.searchedMessage = response.data // 将搜索结果赋值给数据对象
          const regex = /href="[^"]*qid=(\d+&sr=\d+-\d+)/g // 提取qid和后续内容的正则表达式
          const matches = [...this.searchedMessage.matchAll(regex)] // 匹配所有链接中的qid和后续内容
          const productNames = this.searchedMessage ? this.searchedMessage.match(/>([^<]+)(?=<\/a>)/g).map(item => item.slice(1)) : [] // 提取产品名称
          // console.log(matches)
          console.log(productNames)
          if (productNames.length === 0) {
            this.recommend_info = []
            return
          }
          const recommendInfoMap = {} // 辅助对象用于去重
          this.recommend_info = matches.map((match, index) => {
            const name = productNames[index] // 获取产品名称
            // print(name)
            const id = md5(name).toString() // 提取qid
            if (!recommendInfoMap[name]) {
              recommendInfoMap[name] = true
              return { id, name }
            }
            return null // 已经添加过的id，返回null
          }).filter(item => item !== null)
        }
        // console.log(this.recommend_info)
      } catch (error) {
        console.error('Error fetching searched message:', error)
      }
    },
    beforeunloadFn (e) {
      if (this.finished) return
      // 这个事件只有在鼠标真正和浏览器有了交互，再刷新或者关闭时才会触发
      e = e || window.event
      if (e) {
        e.returnValue = '关闭提示'
      }
      return '关闭提示'
    },
    async init () {
      this.loadUser()
      this.loading = true
      this.message = 'Connecting to server'
      this.initWebSocket()
      // this.message = 'Checking profile'
      await this.checkProfile()
      // this.message = 'Loading instructions'
      await this.loadInstructions()
      // this.message = 'Loading search result config'
      await this.loadSearchResultConfig()
      // this.message = 'Loading actions'
      await this.loadAction()
      // this.message = 'Waiting for commands from server'
    },
    profileCompleted () {
      this.needProfile = false
    },
    ratingCompleted () {
      window.location = '/endchoice.html'
    },
    loadInstructions () {
      return this.$http.get('/api/instructions').then((response) => {
        this.instructions = response.data
      })
    },
    loadSearchResultConfig () {
      return this.$http.get('/api/loadSearchResultConfig').then((response) => {
        this.searchResultConfig = response.data
      })
    },
    checkProfile () {
      return this.$http.get('/api/checkProfile').then((response) => {
        this.needProfile = response.data
      })
    },
    initWebSocket () {
      let loc = window.location
      this.$connect('ws://' + loc.hostname + ':' + loc.port + '/websocket/' + this.username)
      this.$socket.onopen = this.websocketonopen
      this.$socket.onerror = this.websocketonerror
      this.$socket.onmessage = this.websocketonmessage
      this.$socket.onclose = this.websocketclose
    },
    finish (data, callback) {
      this.loading = true
      this.message = '对话结束！'
      this.websocketsend({type: 'FINISH'})
      callback()
    },
    reset () {
      this.loading = true
      this.message = ''
      this.partner = ''
      this.disabled = false
      this.role = ''
      this.finished = false
      this.$refs.chatUI.reset()
      this.$refs.labelling.reset()
    },
    websocketonopen (e) {
      // console.log('WebSocket连接成功', e)
    },

    websocketonerror (e) {
      // 错误
      this.$Notice.error(
        {
          title: '错误',
          desc: '服务器错误!'
        })
    },
    websocketonmessage (e) {
      const dataList = JSON.parse(e.data)
      dataList.map(async data => {
        // 数据接收，同步的执行方式，load数据完之后，才能往下执行
        if (data.messageCommand === 'START' && !this.finished) {
          this.$Notice.info(
            {
              title: '通知',
              desc: '您的对话对象已上线!'
            }
          )
          this.reset()
          let tempData = data.data
          this.partner = tempData.partner
          this.role = tempData.role
          this.conversationId = tempData.conversationId
          // this.userId = tempData.userId
          this.loading = true
          this.message = '正在读取'
          await this.loadHistoryAndBackground()
          this.loading = false
        }
        if (data.messageCommand === 'WAIT4PARTNER' && !this.finished) {
          this.message = '正在匹配'
          this.partner = ''
          this.role = ''
          this.loading = true
        }
        if (data.messageCommand === 'SENDMESSAGE' && !this.finished) {
          console.log('hhhhhhhhhhhhhhhhhhhhhhhhhhhhhh')
          console.log(data)
          console.log('hhhhhhhhhhhhhhhhhhhhhhhhhhhhhh')
          console.log(dataList)
          if (this.role === 'cus') {
            this.raw_info = data.content.split('iuiuiuiuiuiui')
            data.content = this.raw_info[0]
            console.log(this.raw_info)
            let tmp = this.raw_info[1].toString()
            this.raw_info = tmp.split('|*&*|')
            this.raw_info.pop()
            console.log(this.raw_info)
            for (let i = 0; i < this.raw_info.length; i++) {
              let tmp2 = this.raw_info[i].split('imidimid')
              let tmp3 = {name: tmp2[0], id: tmp2[1]}
              if (!this.add_info.some(item => item.name === tmp3.name && item.id === tmp3.id)) {
                this.add_info.push(tmp3)
              }
            }
            console.log('raw')
            console.log(this.raw_info)
            console.log('add')
            console.log(this.add_info)
            this.recommend_info = this.add_info
          }
          if (data.content) {
            this.$refs.chatUI.addMessage({message: data.content, time: new Date()}, 'sys')
          }
          console.log(this.recommend_info)
          if (this.role === 'sys') {
            this.loading = true
            this.disabled = true
            this.message = 'Loading states'
            await this.loadCurrentState()
          }
          this.loading = false
          this.disabled = false
        }
        if (data.messageCommand === 'WAIT4MESSAGE' && !this.finished) {
          if (data.content) {
            this.$refs.chatUI.addMessage({message: data.content, time: new Date()}, 'sys')
          }
          this.loading = false
          this.disabled = true
        }
        if (data.messageCommand === 'ERROR' && !this.finished) {
          if (data.conversationId >= 0) {
            this.$Notice.error(
              {
                title: '错误',
                desc: data.content
              }
            )
          } else {
            this.loading = true
            this.$Modal.warning({
              title: '提示',
              content: data.content
            })
          }
        }
        if (data.messageCommand === 'STOP' && !this.finished) {
          this.$Notice.error(
            {
              title: '错误',
              desc: '您的对话对象已经下线!'
            }
          )
          this.message = '正在等待您的对话对象重新连接'
          this.loading = true
        }
        if (data.messageCommand === 'HURRYUP' && !this.finished) {
          this.$Notice.warning(
            {
              title: '提示',
              desc: '请加快速度!'
            }
          )
        }
        if (data.messageCommand === 'FINISH') {
          this.finished = true
          this.$Notice.warning(
            {
              title: '提示',
              desc: '对话已结束，请在离开之前评价。'
            }
          )
          this.loading = false
          this.disabled = true
        }
      })
    },
    websocketsend (data) {
      console.log('saddddddddddddddddddddddddddddd')
      console.log(data)
      this.$socket.send(JSON.stringify({
        from: this.username,
        to: this.partner,
        type: this.role === 'sys' ? 'SYS2CUS' : 'CUS2SYS',
        product: 'wow',
        ...data
      }))
    },
    hurryup () {
      this.$socket.send(JSON.stringify({
        content: '请加快速度!',
        from: this.username,
        to: this.partner,
        type: 'HURRYUP'
      }))
      this.hurryupLeftSeconds = maxHurryupLeftSeconds
      let interval = setInterval(() => {
        this.hurryupLeftSeconds--
        if (this.hurryupLeftSeconds === 0) {
          clearInterval(interval)
        }
      }, 1000)
    },
    websocketclose (e) {
      // 关闭链接时触发
      var code = e.code//  状态码表 https://developer.mozilla.org/zh-CN/docs/Web/API/CloseEvent
      var reason = e.reason
      var wasClean = e.wasClean
      console.log(code, reason, wasClean)
    },
    loadUser () {
      this.username = util.parseUrl(location.href)._user
    },
    loadHistoryAndBackground () {
      const params = {
        sysName: this.role === 'sys' ? this.username : this.partner,
        cusName: this.role === 'cus' ? this.username : this.partner,
        conversationId: this.conversationId
      }
      return this.$http.get('/api/loadHistoryAndBackground', {params}).then((response) => {
        // todo 处理history
        if (response.data.history) {
          let history = []
          response.data.history.map(value => {
            history.unshift({
              message: value.content,
              time: value.updateTime,
              type: (value.type === 'CUS2SYS' && this.role === 'cus') || (value.type === 'SYS2CUS' && this.role === 'sys') ? 'user' : 'sys'
            })
          })
          this.$refs.chatUI.addHistoy(history)
        }
        if (response.data.background) {
          this.$refs.chatUI.addMessage({
            message: '<strong>您的角色: </strong>' + (this.role === 'sys' ? '系统. ' : '普通用户. ') + '<br/>' + (this.role === 'cus' ? '<strong>Background: </strong>' + response.data.background + '<br/>' : '') + '<strong>对话开始!</strong>' + (this.role === 'cus' ? '<br/>' + '注意: 由于系统角色需要在向您发送回应之前进行多个步骤，所以系统回复可能会有点慢，请耐心等待。' : ''),
            time: new Date()
          }, 'other')
          if (this.role === 'cus') {
            this.background = response.data.background
          } else {
            this.background = '请尝试帮助您的伙伴获取他/她的目标信息。'
          }
        }
      })
    },
    loadCurrentState () {
      return this.$http.get('/api/loadCurrentState').then((response) => {
        // this.currentState = response.data
        this.currentState = ''
      })
    },
    saveStatesBackup (newStates) {
      this.statesBackupList = newStates
      console.log('NEW STATES SAVED: ' + newStates)
    },
    searchWithFilters (newFilters) {
      console.log('Searching with filters: ' + newFilters)
      console.log('Searching for the query:  ' + this.statesBackupList)
      if (this.statesBackupList.length === 0 || newFilters.length === 0) {
        return
      }
      // debugger
      let that = this
      let url = 'http://localhost:9191?query=' + encodeURIComponent(this.statesBackupList.join(' ')) + '&refinements=' + encodeURIComponent(newFilters.join(','))
      axios.get(
        url, { timeout: 50000 }
      ).then((json) => {
        if (json.data.ref === 'error') {
          this.$Notice.warning({
            title: '警告',
            desc: '您选择了太多限制，请重新选择。'
          })
          return
        }
        // debugger
        this.$http.post('/api/saveSearchResults', {
          query: this.statesBackupList.join(' '),
          filter: newFilters.join(' '),
          conversationId: this.conversationId,
          data: json.data
        })
        this.$Notice.success({
          title: '成功!',
          desc: '搜索成功.'
        })
        this.searchResultsBackup = json.data
        that.bus.$emit('loadMore', json.data)
      }).catch((error) => {
        console.log(error)
        this.$Notice.error({
          title: '错误',
          desc: '请刷新页面或更改查询以重新搜索!'
        })
        let emptyData = {
          Answer: [],
          Suggest: [],
          Filters: [],
          Aspects: []
        }
        this.searchResultsBackup = emptyData
        that.bus.$emit('loadMore', emptyData)
      })
    },
    searchData (states, callback) {
      if (states.length === 0) {
        return
      }
      console.log('Searching for ' + states)
      this.$Notice.info({
        title: '搜索中',
        desc: '搜索中，请稍候'
      })
      this.searching = true
      this.searchPanelLoading = false
      let url = 'http://localhost:9191?query=' + encodeURIComponent(states.join(' '))
      axios.get(
        url, { timeout: 50000 }
      ).then((json) => {
        if (json.data.key === 'error') {
          this.$Notice.warning({
            title: '警告',
            desc: '查询范围过大，请选择一个更精确的查询。'
          })
          let emptyData = {
            Answer: [],
            Suggest: [],
            Filters: [],
            Aspects: []
          }
          this.searchResultsBackup = emptyData
          callback(emptyData)
          return
        }
        this.$http.post('/api/saveSearchResults', {
          query: states.join(' '),
          filter: '',
          conversationId: this.conversationId,
          data: json.data
        })
        console.log('Search successful.')
        this.$Notice.success({
          title: '成功!',
          desc: '搜索成功.'
        })
        this.searchResultsBackup = json.data
        callback(json.data)
      }).catch((error) => {
        console.log(error)
        this.$Notice.error({
          title: '错误',
          desc: '请刷新页面或更改查询以重新搜索!'
        })
        let emptyData = {
          Answer: [],
          Suggest: [],
          Filters: [],
          Aspects: []
        }
        this.searchResultsBackup = emptyData
        callback(emptyData)
      })
    },
    loadAction () {
      return this.$http.get('/api/loadActions').then((response) => {
        this.actions = response.data
      })
    },
    sendData (data, callback) {
      console.log(data.sendAnother)
      this.loading = true
      this.message = 'Sending message'
      let msg = data.response
      console.log('response')
      console.log(data.response)
      console.log(this.selectedResultsBackup)
      for (let i = 0; i < this.selectedResultsBackup.length; i++) {
        let item = this.selectedResultsBackup[i]
        if (item.hasOwnProperty('link')) {
          msg += `<div style="display: flex; align-items: center;"><img src="${item.image}" alt="${item.title}" height="50px" width="50px" style="margin-right: 10px;"> <a style="color: yellowgreen" target="_blank" href="${item.link}">${item.title}</a></div>`
        }
        if (i < this.selectedResultsBackup.length - 1) {
          msg += '<br>' // Add an empty line between results
        }
      }
      let msg2 = msg
      if (this.role === 'sys') {
        msg += 'iuiuiuiuiuiui'
        for (let i = 0; i < this.recommend_info.length; i++) {
          msg += this.recommend_info[i]
          const id = md5(name).toString()
          msg += 'imidimid' + id
          msg += '|*&*|'
        }
      }
      data.response = msg
      if (this.role === 'sys') {
        msg = msg2
      }
      console.log('data:')
      console.log(data)
      this.websocketsend(data)
      this.$refs.chatUI.addMessage({message: msg2, time: new Date()}, 'user')
      callback()
    },
    saveChangeBackup (checked) {
      this.selectedResultsBackup = checked
    }
  }
}
</script>
