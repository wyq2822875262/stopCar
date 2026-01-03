<template>
  <div class="app-wrapper">
    <div class="background-layer"></div>
    <div class="stop-car-container">
      <h1>优惠券二维码生成器</h1>
    
    <div class="form-section">
      <div class="form-group">
        <label for="coupon">选择优惠券：</label>
        <select id="coupon" v-model="selectedCoupon" @change="generateLink">
          <option value="">请选择优惠券</option>
          <option 
            v-for="coupon in coupons" 
            :key="coupon.id" 
            :value="coupon.id"
            :disabled="coupon.couponBalance <= 0"
          >
            {{ coupon.couponName }} (ID: {{ coupon.id }})
            <span v-if="coupon.couponBalance <= 0" class="out-of-stock"> - 已售罄</span>
            <span v-else class="stock-info">(库存: {{ coupon.couponBalance }})</span>
          </option>
        </select>
      </div>

      <div class="form-group">
        <label for="validity">选择有效期：</label>
        <select id="validity" v-model="selectedValidity" @change="generateLink">
          <option value="10">10分钟</option>
          <option value="30">30分钟</option>
          <option value="60">1小时</option>
          <option value="120">2小时</option>
          <option value="240">4小时</option>
          <option value="480">8小时</option>
          <option value="1440">24小时</option>
        </select>
      </div>

      <div class="form-group">
        <label>自定义过期时间：</label>
        <input 
          type="datetime-local" 
          v-model="customTime" 
          @change="generateLink"
          :min="minDateTime"
        />
      </div>
    </div>

    <div class="result-section" v-if="generatedLink">
      <div class="link-section">
        <h3>生成的链接：</h3>
        <div class="link-container">
          <input 
            type="text" 
            :value="generatedLink" 
            readonly 
            ref="linkInput"
            class="link-input"
          />
          <button @click="copyLink" class="copy-btn">复制</button>
          <button @click="openLink" class="visit-btn" :disabled="!generatedLink">访问</button>
        </div>
      </div>

      <div class="qr-section">
        <h3>二维码：</h3>
        <div class="qr-container">
          <QRCode 
            :value="generatedLink" 
            :size="200" 
            level="H"
            render-as="svg"
          />
        </div>
        <div class="qr-actions">
          <button @click="downloadQR" class="download-btn">下载二维码</button>
        </div>
      </div>
    </div>

    <div class="info-section" v-if="selectedCouponData">
      <div class="info-header">
        <h3>优惠券详细信息：</h3>
        <button @click="updateCouponStock" class="update-btn" :disabled="isUpdating">
          <span v-if="!isUpdating">更新库存</span>
          <span v-else class="loading">
            <span class="spinner"></span>
            更新中...
          </span>
        </button>
      </div>
      <div class="coupon-info-grid">
        <div class="info-item">
          <span class="info-label">优惠券ID：</span>
          <span class="info-value">{{ selectedCouponData.id }}</span>
        </div>
        <div class="info-item">
          <span class="info-label">优惠券名称：</span>
          <span class="info-value">{{ selectedCouponData.couponName }}</span>
        </div>
        <div class="info-item">
          <span class="info-label">当前库存：</span>
          <span class="info-value" :class="{ 'low-stock': selectedCouponData.couponBalance < 1000, 'updated': stockUpdated }">
            {{ selectedCouponData.couponBalance }}
            <span v-if="stockUpdated" class="update-indicator">✓ 已更新</span>
          </span>
        </div>
        <div class="info-item">
          <span class="info-label">授权停车场：</span>
          <span class="info-value">{{ selectedCouponData.authGarage }}</span>
        </div>
        <div class="info-item">
          <span class="info-label">有效期：</span>
          <span class="info-value">{{ selectedCouponData.validityStart }} 至 {{ selectedCouponData.validityEnd }}</span>
        </div>
        <div class="info-item">
          <span class="info-label">单次费用：</span>
          <span class="info-value">{{ selectedCouponData.onceFee === '0' ? '免费' : '¥' + selectedCouponData.onceFee }}</span>
        </div>
        <div class="info-item">
          <span class="info-label">二维码过期时间：</span>
          <span class="info-value">{{ formatTime(expirationTime) }}</span>
        </div>
        <div class="info-item">
          <span class="info-label">剩余有效时间：</span>
          <span class="info-value" :class="{ 'expired': isExpired }">
            {{ getRemainingTime() }}
          </span>
        </div>
        <div class="info-item" v-if="lastUpdateTime">
          <span class="info-label">上次更新时间：</span>
          <span class="info-value">{{ formatTime(lastUpdateTime) }}</span>
        </div>
      </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import QRCode from 'qrcode.vue'
import couponsData from '../assets/data.json'
import axios from 'axios'

const selectedCoupon = ref('')
const selectedValidity = ref('30')
const customTime = ref('')
const generatedLink = ref('')
const linkInput = ref(null)
const currentTime = ref(Date.now())
const coupons = ref(couponsData)
const isUpdating = ref(false)
const stockUpdated = ref(false)
const lastUpdateTime = ref(null)

const selectedCouponData = computed(() => {
  return coupons.value.find(coupon => coupon.id === parseInt(selectedCoupon.value))
})

const isExpired = computed(() => {
  return expirationTime.value <= Date.now()
})

const minDateTime = computed(() => {
  const now = new Date()
  now.setMinutes(now.getMinutes() + 1)
  return now.toISOString().slice(0, 16)
})

const expirationTime = computed(() => {
  if (customTime.value) {
    return new Date(customTime.value).getTime()
  } else {
    return currentTime.value + (parseInt(selectedValidity.value) * 60 * 1000)
  }
})

const generateLink = () => {
  if (!selectedCoupon.value) {
    generatedLink.value = ''
    return
  }
  
  const times = expirationTime.value
  generatedLink.value = `http://a.redic.cn/apark/?type=couponQr&state=1&couponId=${selectedCoupon.value}&times=${times}`
}

const copyLink = async () => {
  try {
    await navigator.clipboard.writeText(generatedLink.value)
    alert('链接已复制到剪贴板')
  } catch (err) {
    // 降级方案
    linkInput.value.select()
    document.execCommand('copy')
    alert('链接已复制到剪贴板')
  }
}

const openLink = () => {
  if (!generatedLink.value) return
  
  // 在新标签页中打开链接
  const newWindow = window.open(generatedLink.value, '_blank')
  
  // 检查弹窗是否被阻止
  if (!newWindow || newWindow.closed || typeof newWindow.closed === 'undefined') {
    // 如果弹窗被阻止，尝试在当前标签页打开
    alert('已在新标签页中打开链接，如果无法打开，请检查浏览器弹窗设置')
    window.location.href = generatedLink.value
  }
}

const saveToJsonFile = async (data) => {
  try {
    // 在浏览器环境中，我们只能模拟保存到JSON文件
    // 实际的文件写入需要后端支持
    console.log('模拟保存数据到JSON文件:', data)
    
    // 创建下载链接供用户保存
    const jsonString = JSON.stringify(data, null, 2)
    const blob = new Blob([jsonString], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    
    const link = document.createElement('a')
    link.href = url
    link.download = `data_${Date.now()}.json`
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
    URL.revokeObjectURL(url)
    
    return true
  } catch (error) {
    console.error('保存JSON文件失败:', error)
    return false
  }
}

const updateCouponStock = async () => {
  if (!selectedCoupon.value || isUpdating.value) return
  
  isUpdating.value = true
  stockUpdated.value = false
  
  try {
    // 方法1: 使用 axios
    const config = {
      method: 'GET',
      url: `http://a.redic.cn/v3/park/parkPay/getCoupon?couponId=${selectedCoupon.value}`,
      headers: {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/132.0.0.0 Safari/537.36',
        'Accept': 'application/json, text/plain, */*',
        'Accept-Encoding': 'gzip, deflate',
        'Accept-Language': 'zh-CN,zh;q=0.9',
        'Cache-Control': 'no-cache',
        'Referer': `http://a.redic.cn/apark/?type=couponQr&state=1&couponId=${selectedCoupon.value}&times=${Date.now()}`
      },
      timeout: 10000,
      responseType: 'json'
    }
    
    console.log('正在发送请求:', config.url)
    const response = await axios.request(config)
    console.log('收到响应:', response.data)
    
    if (response.data && response.data.code === 200 && response.data.data) {
      const updatedCoupon = response.data.data
      
      // 更新本地数据
      const couponIndex = coupons.value.findIndex(c => c.id === updatedCoupon.id)
      if (couponIndex !== -1) {
        coupons.value[couponIndex] = updatedCoupon
        stockUpdated.value = true
        lastUpdateTime.value = Date.now()
        
        // 保存到JSON文件
        const saveSuccess = await saveToJsonFile(coupons.value)
        
        // 3秒后隐藏更新指示器
        setTimeout(() => {
          stockUpdated.value = false
        }, 3000)
        
        if (saveSuccess) {
          alert('库存更新成功！数据已保存到JSON文件。')
        } else {
          alert('库存更新成功！但保存到JSON文件失败，请手动备份数据。')
        }
        return
      } else {
        throw new Error('本地数据更新失败')
      }
    } else {
      throw new Error(response.data?.msg || '服务器返回数据格式错误')
    }
  } catch (error) {
    console.error('方法1失败:', error)
    
    // 降级方案2: 使用 fetch
    try {
      console.log('尝试使用 fetch 请求...')
      const fetchResponse = await fetch(`http://a.redic.cn/v3/park/parkPay/getCoupon?couponId=${selectedCoupon.value}`, {
        method: 'GET',
        headers: {
          'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/132.0.0.0 Safari/537.36',
          'Accept': 'application/json',
          'Referer': `http://a.redic.cn/apark/?type=couponQr&state=1&couponId=${selectedCoupon.value}&times=${Date.now()}`
        }
      })
      
      if (!fetchResponse.ok) {
        throw new Error(`HTTP ${fetchResponse.status}`)
      }
      
      const data = await fetchResponse.json()
      console.log('fetch 响应:', data)
      
      if (data.code === 200 && data.data) {
        const updatedCoupon = data.data
        const couponIndex = coupons.value.findIndex(c => c.id === updatedCoupon.id)
        if (couponIndex !== -1) {
          coupons.value[couponIndex] = updatedCoupon
          stockUpdated.value = true
          lastUpdateTime.value = Date.now()
          
          // 保存到JSON文件
          const saveSuccess = await saveToJsonFile(coupons.value)
          
          setTimeout(() => {
            stockUpdated.value = false
          }, 3000)
          
          if (saveSuccess) {
            alert('库存更新成功！数据已保存到JSON文件。')
          } else {
            alert('库存更新成功！但保存到JSON文件失败，请手动备份数据。')
          }
          return
        }
      }
      throw new Error(data.msg || '数据格式错误')
    } catch (fetchError) {
      console.error('方法2也失败:', fetchError)
      
      // 最终错误处理
      let errorMessage = '网络错误，请重试'
      
      if (error.code === 'ERR_EMPTY_RESPONSE' || fetchError.message?.includes('Empty')) {
        errorMessage = '服务器返回空响应，可能是网络问题或服务器暂时不可用'
      } else if (error.code === 'ECONNREFUSED' || fetchError.message?.includes('Failed to fetch')) {
        errorMessage = '无法连接到服务器，请检查网络连接'
      } else if (error.code === 'ETIMEDOUT' || fetchError.name === 'AbortError') {
        errorMessage = '请求超时，请稍后重试'
      } else if (error.response?.status === 404 || fetchError.message?.includes('404')) {
        errorMessage = '优惠券不存在或已下架'
      } else if (error.response?.status === 500 || fetchError.message?.includes('500')) {
        errorMessage = '服务器内部错误，请稍后重试'
      } else if (error.response?.status === 403 || fetchError.message?.includes('403')) {
        errorMessage = '访问被拒绝，可能是权限问题'
      } else if (error.message) {
        errorMessage = error.message
      }
      
      alert(`更新失败: ${errorMessage}\n建议: 请检查网络连接或联系管理员`)
    }
  } finally {
    isUpdating.value = false
  }
}

const downloadQR = () => {
  const svgElement = document.querySelector('.qr-container svg')
  if (!svgElement) return
  
  const svgData = new XMLSerializer().serializeToString(svgElement)
  const canvas = document.createElement('canvas')
  const ctx = canvas.getContext('2d')
  const img = new Image()
  
  img.onload = function() {
    canvas.width = 300
    canvas.height = 300
    ctx.fillStyle = 'white'
    ctx.fillRect(0, 0, canvas.width, canvas.height)
    ctx.drawImage(img, 50, 50, 200, 200)
    
    const link = document.createElement('a')
    link.download = `coupon-${selectedCoupon.value}-qrcode.png`
    link.href = canvas.toDataURL()
    link.click()
  }
  
  img.src = 'data:image/svg+xml;base64,' + btoa(unescape(encodeURIComponent(svgData)))
}

const formatTime = (timestamp) => {
  return new Date(timestamp).toLocaleString('zh-CN')
}

const getRemainingTime = () => {
  if (!selectedCoupon.value) return ''
  
  const remaining = expirationTime.value - Date.now()
  if (remaining <= 0) return '已过期'
  
  const hours = Math.floor(remaining / (1000 * 60 * 60))
  const minutes = Math.floor((remaining % (1000 * 60 * 60)) / (1000 * 60))
  const seconds = Math.floor((remaining % (1000 * 60)) / 1000)
  
  if (hours > 0) {
    return `${hours}小时${minutes}分钟${seconds}秒`
  } else if (minutes > 0) {
    return `${minutes}分钟${seconds}秒`
  } else {
    return `${seconds}秒`
  }
}

let timer = null

onMounted(() => {
  timer = setInterval(() => {
    currentTime.value = Date.now()
  }, 1000)
})

onUnmounted(() => {
  if (timer) {
    clearInterval(timer)
  }
})
</script>

<style scoped>
.app-wrapper {
  position: relative;
  min-height: 100vh;
  width: 100%;
  overflow-x: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
}

.background-layer {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  z-index: -1;
}

.stop-car-container {
  max-width: 800px;
  width: 100%;
  margin: 0 auto;
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background: transparent;
  box-sizing: border-box;
  position: relative;
  z-index: 1;
}

h1 {
  text-align: center;
  color: white;
  margin-bottom: 30px;
  font-size: 2.5rem;
  font-weight: 700;
  text-shadow: 0 2px 4px rgba(0,0,0,0.3);
  padding-top: 20px;
}

.form-section {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  padding: 30px;
  border-radius: 20px;
  margin-bottom: 25px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.1);
  border: 1px solid rgba(255,255,255,0.2);
  animation: slideUp 0.6s ease-out;
}

.form-group {
  margin-bottom: 25px;
}

.form-group label {
  display: block;
  margin-bottom: 10px;
  font-weight: 600;
  color: #2d3748;
  font-size: 1rem;
  letter-spacing: -0.025em;
}

.form-group select,
.form-group input {
  width: 100%;
  padding: 16px 20px;
  border: 2px solid #e2e8f0;
  border-radius: 12px;
  font-size: 16px;
  box-sizing: border-box;
  background: white;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  font-family: inherit;
}

.form-group select:focus,
.form-group input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
  transform: translateY(-2px);
}

.form-group select:hover,
.form-group input:hover {
  border-color: #cbd5e0;
}

.form-group select {
  cursor: pointer;
  appearance: none;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12'%3E%3Cpath fill='%23333' d='M6 9L1 4h10z'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 16px center;
  padding-right: 40px;
}

.result-section {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255,255,255,0.2);
  border-radius: 20px;
  padding: 30px;
  margin-bottom: 25px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.1);
  animation: slideUp 0.8s ease-out;
}

.link-section h3,
.qr-section h3 {
  margin-top: 0;
  color: #2d3748;
  margin-bottom: 20px;
  font-size: 1.3rem;
  font-weight: 600;
}

.link-container {
  display: flex;
  gap: 12px;
  align-items: center;
  flex-wrap: wrap;
}

.link-input {
  flex: 1;
  min-width: 200px;
  padding: 14px 18px;
  border: 2px solid #e2e8f0;
  border-radius: 12px;
  font-family: 'SF Mono', Monaco, 'Courier New', monospace;
  font-size: 14px;
  background: #f7fafc;
  color: #4a5568;
  transition: all 0.3s ease;
}

.link-input:focus {
  outline: none;
  border-color: #667eea;
  background: white;
}

.copy-btn,
.download-btn,
.visit-btn {
  padding: 14px 24px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  font-size: 15px;
  font-weight: 600;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  box-shadow: 0 4px 15px rgba(102, 126, 234, 0.3);
  white-space: nowrap;
}

.copy-btn:hover,
.download-btn:hover,
.visit-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.4);
}

.copy-btn:active,
.download-btn:active,
.visit-btn:active {
  transform: translateY(0);
}

.qr-container {
  display: flex;
  justify-content: center;
  margin: 25px 0;
  padding: 30px;
  background: white;
  border: 2px solid #e2e8f0;
  border-radius: 20px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.08);
  position: relative;
  overflow: hidden;
}

.qr-container::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(45deg, #f0f4ff 25%, transparent 25%, transparent 75%, #f0f4ff 75%, #f0f4ff),
              linear-gradient(45deg, #f0f4ff 25%, transparent 25%, transparent 75%, #f0f4ff 75%, #f0f4ff);
  background-size: 20px 20px;
  background-position: 0 0, 10px 10px;
  opacity: 0.3;
  z-index: 0;
}

.qr-container :deep(svg) {
  position: relative;
  z-index: 1;
  border-radius: 8px;
  overflow: hidden;
}

.qr-actions {
  text-align: center;
  margin-top: 20px;
}

.info-section {
  background: linear-gradient(135deg, #48bb78 0%, #38a169 100%);
  border: none;
  border-radius: 20px;
  padding: 25px;
  color: white;
  box-shadow: 0 10px 30px rgba(72, 187, 120, 0.3);
  animation: slideUp 1s ease-out;
}

.info-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  flex-wrap: wrap;
  gap: 15px;
}

.info-section h3 {
  margin: 0;
  color: white;
  font-size: 1.2rem;
  font-weight: 600;
}

.coupon-info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 15px;
}

.info-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  transition: all 0.3s ease;
}

.info-item:hover {
  background: rgba(255, 255, 255, 0.15);
  transform: translateY(-1px);
}

.info-label {
  font-weight: 600;
  font-size: 0.9rem;
  color: rgba(255, 255, 255, 0.9);
}

.info-value {
  font-weight: 500;
  font-size: 0.95rem;
  color: white;
}

.info-value.low-stock {
  color: #feb2b2;
  font-weight: 700;
}

.info-value.expired {
  color: #feb2b2;
  font-weight: 700;
  animation: pulse 2s infinite;
}

.out-of-stock {
  color: #e53e3e;
  font-weight: 600;
}

.stock-info {
  color: #38a169;
  font-size: 0.85rem;
}

.update-btn {
  padding: 10px 20px;
  background: linear-gradient(135deg, #3182ce 0%, #2c5282 100%);
  color: white;
  border: none;
  border-radius: 10px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 600;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  box-shadow: 0 3px 12px rgba(49, 130, 206, 0.3);
  display: flex;
  align-items: center;
  gap: 8px;
  white-space: nowrap;
}

.update-btn:hover:not(:disabled) {
  transform: translateY(-1px);
  box-shadow: 0 5px 16px rgba(49, 130, 206, 0.4);
  background: linear-gradient(135deg, #2c5282 0%, #2a4e7c 100%);
}

.update-btn:active {
  transform: translateY(0);
}

.update-btn:disabled {
  opacity: 0.7;
  cursor: not-allowed;
  transform: none;
}

.visit-btn:disabled {
  opacity: 0.7;
  cursor: not-allowed;
  transform: none;
}

.visit-btn:not(:disabled) {
  background: linear-gradient(135deg, #48bb78 0%, #38a169 100%);
  box-shadow: 0 4px 15px rgba(72, 187, 120, 0.3);
}

.visit-btn:not(:disabled):hover {
  box-shadow: 0 6px 20px rgba(72, 187, 120, 0.4);
}

.loading {
  display: flex;
  align-items: center;
  gap: 8px;
}

.spinner {
  width: 14px;
  height: 14px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top: 2px solid white;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

.update-indicator {
  margin-left: 8px;
  font-size: 0.85rem;
  color: #90ee90;
  font-weight: 600;
}

.info-value.updated {
  color: #90ee90;
  font-weight: 600;
  position: relative;
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes pulse {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.7;
  }
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

/* 移动端适配 */
@media (max-width: 768px) {
  .app-wrapper {
    align-items: flex-start;
    justify-content: center;
    padding-top: 20px;
  }
  
  .stop-car-container {
    padding: 15px;
  }
  
  h1 {
    font-size: 2rem;
    margin-bottom: 25px;
    padding-top: 15px;
  }
  
  .form-section,
  .result-section {
    padding: 25px;
    margin-bottom: 20px;
  }
  
  .link-container {
    flex-direction: column;
    align-items: stretch;
    gap: 12px;
  }
  
  .link-input {
    min-width: auto;
  }
  
  .copy-btn {
    margin-top: 0;
  }
}

@media (max-width: 480px) {
  .app-wrapper {
    padding-top: 15px;
  }
  
  .stop-car-container {
    padding: 12px;
  }
  
  h1 {
    font-size: 1.8rem;
    margin-bottom: 20px;
  }
  
  .form-section,
  .result-section {
    padding: 20px;
    margin-bottom: 15px;
    border-radius: 16px;
  }
  
  .form-group {
    margin-bottom: 20px;
  }
  
  .form-group label {
    font-size: 0.95rem;
  }
  
  .form-group select,
  .form-group input {
    padding: 14px 16px;
    font-size: 16px; /* 防止iOS自动缩放 */
  }
  
  .qr-container {
    padding: 20px;
    margin: 20px 0;
  }
  
  .info-section {
    padding: 20px;
    border-radius: 16px;
  }
  
  .copy-btn,
  .download-btn,
  .visit-btn {
    padding: 12px 20px;
    font-size: 14px;
  }
  
  .info-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 10px;
  }
  
  .update-btn {
    width: 100%;
    justify-content: center;
  }
}

/* 超小屏幕适配 */
@media (max-width: 320px) {
  .app-wrapper {
    padding-top: 10px;
  }
  
  .stop-car-container {
    padding: 10px;
  }
  
  h1 {
    font-size: 1.6rem;
  }
  
  .form-section,
  .result-section {
    padding: 15px;
  }
}

/* 暗色模式支持 */
@media (prefers-color-scheme: dark) {
  .stop-car-container {
    background: linear-gradient(135deg, #1a202c 0%, #2d3748 100%);
  }
  
  .form-section,
  .result-section {
    background: rgba(45, 55, 72, 0.95);
    border-color: rgba(255,255,255,0.1);
  }
  
  .form-group label {
    color: #e2e8f0;
  }
  
  .form-group select,
  .form-group input {
    background: #2d3748;
    border-color: #4a5568;
    color: #e2e8f0;
  }
  
  .form-group select:focus,
  .form-group input:focus {
    border-color: #667eea;
  }
  
  .link-section h3,
  .qr-section h3 {
    color: #e2e8f0;
  }
  
  .link-input {
    background: #2d3748;
    border-color: #4a5568;
    color: #e2e8f0;
  }
}

/* 平板适配 */
@media (min-width: 769px) and (max-width: 1024px) {
  .stop-car-container {
    padding: 25px;
  }
  
  .form-section,
  .result-section {
    padding: 35px;
  }
}

/* 大屏幕优化 */
@media (min-width: 1200px) {
  .stop-car-container {
    padding: 40px;
  }
  
  h1 {
    font-size: 3rem;
  }
}
</style>