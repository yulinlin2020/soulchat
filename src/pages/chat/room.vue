<template>
  <view class="chat-room">
    <!-- 聊天头部 -->
    <view class="chat-header">
      <button class="back-btn" @click="exitChat">
        <text class="back-icon">←</text>
      </button>
      <view class="chat-info">
        <text class="partner-name">{{ partnerInfo.nickname || '神秘聊友' }}</text>
        <text class="online-status">{{ connectionStatus }}</text>
      </view>
      <button class="menu-btn" @click="showExitConfirm">
        <text class="menu-icon">⋯</text>
      </button>
    </view>

    <!-- 消息列表 -->
    <scroll-view 
      class="message-list" 
      scroll-y 
      :scroll-top="scrollTop"
      :scroll-with-animation="true"
    >
      <view class="message-container">
        <!-- 系统消息 -->
        <view class="system-message">
          <text class="system-text">🎉 匹配成功！开始愉快的聊天吧</text>
        </view>

        <!-- 聊天消息 -->
        <view 
          v-for="message in messages" 
          :key="message.id"
          class="message-wrapper"
          :class="[message.senderId === currentUserId ? 'my-message' : 'other-message']"
        >
          <view class="message-bubble">
            <text class="message-text">{{ message.content }}</text>
            <view class="message-meta">
              <text class="message-time">{{ formatMessageTime(message.sendTime) }}</text>
              <text 
                v-if="message.senderId === currentUserId" 
                class="message-status"
                :class="[`status-${message.status}`]"
              >
                {{ getStatusText(message.status) }}
              </text>
            </view>
          </view>
        </view>

        <!-- 对方离开提示 -->
        <view v-if="roomClosed" class="system-message">
          <text class="system-text">😔 对方已离开聊天</text>
        </view>
      </view>
    </scroll-view>

    <!-- 输入区域 -->
    <view class="input-area" v-if="!roomClosed">
      <view class="input-container">
        <input 
          v-model="inputText"
          class="message-input"
          type="text"
          placeholder="输入消息..."
          :disabled="sendingMessage"
          @confirm="sendMessage"
          confirm-type="send"
          maxlength="500"
        />
        <button 
          class="send-btn" 
          :class="[canSend ? 'can-send' : 'cannot-send']"
          :disabled="!canSend || sendingMessage"
          @click="sendMessage"
        >
          <text class="send-text">{{ sendingMessage ? '发送中' : '发送' }}</text>
        </button>
      </view>
    </view>

    <!-- 确认退出对话框 -->
    <view v-if="showExitDialog" class="exit-dialog-overlay" @click="hideExitConfirm">
      <view class="exit-dialog" @click.stop>
        <text class="dialog-title">确认退出聊天？</text>
        <text class="dialog-desc">退出后将无法恢复当前聊天记录</text>
        <view class="dialog-actions">
          <button class="dialog-btn cancel-btn" @click="hideExitConfirm">
            <text>取消</text>
          </button>
          <button class="dialog-btn confirm-btn" @click="confirmExit">
            <text>退出</text>
          </button>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, nextTick } from 'vue'
import { navigateTo, showToast, formatDate, parseQueryString } from '@/utils/index'
import { auth, app } from '@/utils/cloudbase'

// 页面参数
let roomId = ''
const currentUserId = ref('')
const partnerInfo = ref<any>({})

// 聊天状态
const roomClosed = ref(false)
const connectionStatus = ref('连接中...')
const messages = ref<any[]>([])
const inputText = ref('')
const sendingMessage = ref(false)
const scrollTop = ref(0)

// 对话框状态
const showExitDialog = ref(false)

// 计算属性
const canSend = computed(() => {
  return inputText.value.trim().length > 0 && !roomClosed.value
})

// 获取页面参数
const getPageParams = () => {
  // 获取页面参数
  const pages = getCurrentPages()
  const currentPage = pages[pages.length - 1] as any
  const options = currentPage.options || {}
  
  roomId = options.roomId || ''
  console.log('房间ID:', roomId)
}

// 初始化聊天室
const initChatRoom = async () => {
  try {
    // 获取当前用户信息
    const loginState = await auth.getLoginState()
    if (loginState) {
      currentUserId.value = loginState.user?.uid || ''
    } else {
      showToast('请先登录', 'error')
      navigateTo('/pages/login/phone-login', 'redirectTo')
      return
    }

    // 获取聊天伙伴信息
    await getPartnerInfo()

    connectionStatus.value = '在线'

    // 模拟加载历史消息
    await loadMessages()

    // TODO: 这里将来会监听实时消息
    console.log('开始监听聊天室消息...')
    
  } catch (error) {
    console.error('初始化聊天室失败:', error)
    showToast('加载聊天室失败', 'error')
  }
}

// 获取聊天伙伴信息
const getPartnerInfo = async () => {
  try {
    // 通过房间ID获取房间信息，找到聊天伙伴
    const db = app.database()
    const roomResult = await db.collection('chat_rooms')
      .doc(roomId)
      .get()
    
    if (roomResult.data.length > 0) {
      const room = roomResult.data[0]
      const participants = room.participants || []
      
      // 找到对方的userId（不是当前用户的）
      const partnerUserId = participants.find((id: string) => id !== currentUserId.value)
      
      if (partnerUserId) {
        // 获取对方的用户信息
        const userResult = await db.collection('users')
          .where({
            userId: partnerUserId
          })
          .get()
        
        if (userResult.data.length > 0) {
          const partner = userResult.data[0]
          partnerInfo.value = {
            userId: partner.userId,
            nickname: partner.nickname || '神秘聊友',
            avatar: partner.avatar || '👤'
          }
          console.log('获取聊天伙伴信息成功:', partnerInfo.value)
        } else {
          console.error('未找到聊天伙伴用户信息')
          partnerInfo.value = {
            nickname: '神秘聊友',
            avatar: '👤'
          }
        }
      } else {
        console.error('未找到聊天伙伴ID')
        partnerInfo.value = {
          nickname: '神秘聊友', 
          avatar: '👤'
        }
      }
    } else {
      console.error('未找到聊天房间信息')
      partnerInfo.value = {
        nickname: '神秘聊友',
        avatar: '👤'
      }
    }
  } catch (error: any) {
    console.error('获取聊天伙伴信息失败:', error)
    partnerInfo.value = {
      nickname: '神秘聊友',
      avatar: '👤'
    }
  }
}

// 加载消息
const loadMessages = async () => {
  // 模拟一些历史消息
  messages.value = []
  
  // 滚动到底部
  scrollToBottom()
}

// 发送消息
const sendMessage = async () => {
  if (!canSend.value || sendingMessage.value) return

  const messageText = inputText.value.trim()
  if (!messageText) return

  try {
    sendingMessage.value = true

    // 创建消息对象
    const newMessage = {
      id: Date.now().toString(),
      roomId: roomId,
      senderId: currentUserId.value,
      content: messageText,
      type: 'text',
      sendTime: new Date(),
      status: 'sending'
    }

    // 添加到消息列表
    messages.value.push(newMessage)
    inputText.value = ''

    // 滚动到底部
    scrollToBottom()

    // TODO: 这里将来会调用云函数发送消息
    console.log('发送消息:', newMessage)

    // 模拟发送成功
    setTimeout(() => {
      newMessage.status = 'sent'
      sendingMessage.value = false

      // 模拟接收对方回复（50%概率）
      if (Math.random() > 0.5) {
        setTimeout(() => {
          receiveMessage()
        }, 1000 + Math.random() * 3000)
      }
    }, 1000)

  } catch (error) {
    console.error('发送消息失败:', error)
    showToast('发送失败', 'error')
    sendingMessage.value = false

    // 更新消息状态为失败
    const lastMessage = messages.value[messages.value.length - 1]
    if (lastMessage) {
      lastMessage.status = 'failed'
    }
  }
}

// 模拟接收消息
const receiveMessage = () => {
  const replies = [
    '你好呀！很高兴遇见你',
    '今天天气不错呢',
    '你平时都喜欢做什么？',
    '哈哈哈，有意思',
    '说得对',
    '我也这样觉得',
    '真的吗？',
    '太棒了！',
    '嗯嗯，没错'
  ]

  const replyMessage = {
    id: Date.now().toString(),
    roomId: roomId,
    senderId: 'partner_id',
    content: replies[Math.floor(Math.random() * replies.length)],
    type: 'text',
    sendTime: new Date(),
    status: 'sent'
  }

  messages.value.push(replyMessage)
  scrollToBottom()
}

// 滚动到底部
const scrollToBottom = async () => {
  await nextTick()
  scrollTop.value = 9999999
}

// 格式化消息时间
const formatMessageTime = (time: Date) => {
  const now = new Date()
  const msgTime = new Date(time)
  const diffMinutes = Math.floor((now.getTime() - msgTime.getTime()) / (1000 * 60))

  if (diffMinutes < 1) return '刚刚'
  if (diffMinutes < 60) return `${diffMinutes}分钟前`
  
  const diffHours = Math.floor(diffMinutes / 60)
  if (diffHours < 24) return `${diffHours}小时前`
  
  return formatDate(msgTime, 'MM-DD HH:mm')
}

// 获取消息状态文本
const getStatusText = (status: string) => {
  switch (status) {
    case 'sending': return '发送中'
    case 'sent': return '已发送'
    case 'failed': return '发送失败'
    default: return ''
  }
}

// 显示退出确认
const showExitConfirm = () => {
  showExitDialog.value = true
}

// 隐藏退出确认
const hideExitConfirm = () => {
  showExitDialog.value = false
}

// 确认退出
const confirmExit = () => {
  hideExitConfirm()
  exitChat()
}

// 退出聊天
const exitChat = async () => {
  try {
    // TODO: 调用云函数关闭房间
    console.log('退出聊天房间:', roomId)
    
    showToast('已退出聊天', 'none')
    
    // 返回聊天主页
    navigateTo('/pages/chat/home', 'redirectTo')
    
  } catch (error) {
    console.error('退出聊天失败:', error)
    showToast('退出失败', 'error')
  }
}

// 处理对方离开
const handlePartnerLeft = () => {
  roomClosed.value = true
  connectionStatus.value = '已离线'
  showToast('对方已离开聊天', 'none')
}

onMounted(() => {
  getPageParams()
  initChatRoom()
})

onUnmounted(() => {
  // TODO: 清理监听器
  console.log('清理聊天室监听器')
})
</script>

<style scoped>
.chat-room {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: #f5f5f5;
}

/* 聊天头部 */
.chat-header {
  background: #6c5ce7;
  padding: 20rpx 30rpx;
  display: flex;
  align-items: center;
  color: white;
  position: relative;
  z-index: 10;
}

.back-btn {
  width: 60rpx;
  height: 60rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: none;
  margin-right: 20rpx;
}

.back-btn::after {
  border: none;
}

.back-icon {
  color: white;
  font-size: 40rpx;
  font-weight: bold;
}

.chat-info {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.partner-name {
  font-size: 32rpx;
  font-weight: 600;
  color: white;
  margin-bottom: 4rpx;
}

.online-status {
  font-size: 24rpx;
  color: rgba(255,255,255,0.8);
}

.menu-btn {
  width: 60rpx;
  height: 60rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: none;
}

.menu-btn::after {
  border: none;
}

.menu-icon {
  color: white;
  font-size: 36rpx;
  font-weight: bold;
}

/* 消息列表 */
.message-list {
  flex: 1;
  padding: 20rpx;
  background: #f5f5f5;
}

.message-container {
  padding-bottom: 40rpx;
}

.system-message {
  text-align: center;
  margin: 30rpx 0;
}

.system-text {
  background: rgba(0,0,0,0.1);
  color: #666;
  font-size: 24rpx;
  padding: 12rpx 20rpx;
  border-radius: 16rpx;
  display: inline-block;
}

.message-wrapper {
  margin-bottom: 24rpx;
  display: flex;
}

.my-message {
  justify-content: flex-end;
}

.other-message {
  justify-content: flex-start;
}

.message-bubble {
  max-width: 70%;
  padding: 20rpx;
  border-radius: 20rpx;
  position: relative;
}

.my-message .message-bubble {
  background: #6c5ce7;
  color: white;
}

.other-message .message-bubble {
  background: white;
  color: #333;
  border: 1rpx solid #eee;
}

.message-text {
  font-size: 28rpx;
  line-height: 1.4;
  word-wrap: break-word;
}

.message-meta {
  margin-top: 12rpx;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 20rpx;
}

.my-message .message-meta {
  color: rgba(255,255,255,0.8);
}

.other-message .message-meta {
  color: #999;
}

.message-time {
  font-size: 20rpx;
}

.message-status {
  font-size: 20rpx;
}

.status-sending {
  color: rgba(255,255,255,0.6);
}

.status-sent {
  color: rgba(255,255,255,0.8);
}

.status-failed {
  color: #ff4757;
}

/* 输入区域 */
.input-area {
  background: white;
  padding: 20rpx;
  border-top: 1rpx solid #eee;
}

.input-container {
  display: flex;
  align-items: center;
  background: #f8f9fa;
  border-radius: 25rpx;
  padding: 8rpx 8rpx 8rpx 20rpx;
}

.message-input {
  flex: 1;
  background: transparent;
  border: none;
  font-size: 28rpx;
  color: #333;
  min-height: 60rpx;
  line-height: 60rpx;
}

.send-btn {
  background: #6c5ce7;
  color: white;
  border: none;
  border-radius: 20rpx;
  padding: 12rpx 24rpx;
  margin-left: 12rpx;
  min-width: 120rpx;
}

.send-btn.cannot-send {
  background: #ccc;
}

.send-btn::after {
  border: none;
}

.send-text {
  font-size: 26rpx;
  font-weight: 500;
}

/* 退出对话框 */
.exit-dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.exit-dialog {
  background: white;
  border-radius: 20rpx;
  padding: 40rpx;
  margin: 40rpx;
  max-width: 500rpx;
}

.dialog-title {
  font-size: 32rpx;
  font-weight: 600;
  color: #333;
  display: block;
  margin-bottom: 20rpx;
  text-align: center;
}

.dialog-desc {
  font-size: 26rpx;
  color: #666;
  line-height: 1.4;
  text-align: center;
  margin-bottom: 40rpx;
  display: block;
}

.dialog-actions {
  display: flex;
  gap: 20rpx;
}

.dialog-btn {
  flex: 1;
  padding: 20rpx;
  border-radius: 12rpx;
  font-size: 28rpx;
  border: none;
}

.dialog-btn::after {
  border: none;
}

.cancel-btn {
  background: #f8f9fa;
  color: #666;
}

.confirm-btn {
  background: #ff4757;
  color: white;
}
</style> 