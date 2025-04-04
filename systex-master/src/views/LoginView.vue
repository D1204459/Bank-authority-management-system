<template>
  <div class="login-container">
    <div class="login-box">
      <div class="login-left">
        <h3>請使用系統帳號/密碼登入平台</h3>
        <img src="@/img/systexlogo.png" alt="Systex Logo" class="login-image"/>
        <p>欲變更密碼，請聯絡系統管理員</p>
      </div>
      <div class="login-right">
        <h2>系統登入</h2>
        <div class="form-group">
          <label>帳號:</label>
          <input
              type="text"
              v-model="username"
              class="form-input"
          />
        </div>
        <div class="form-group">
          <label>密碼:</label>
          <div class="password-input-wrapper">
            <input
                :type="showPassword ? 'text' : 'password'"
                v-model="password"
                class="form-input"
                @keydown.enter="handleLogin"
            />
            <span class="password-toggle" @click="showPassword = !showPassword">
              <i :class="['fas', showPassword ? 'fa-eye-slash' : 'fa-eye']"></i>
            </span>
          </div>
        </div>
        <button
            class="login-btn"
            @click="handleLogin"
            :disabled="isLoading"
            :class="{ 'admin-mode': isAdmin }"
        >
          {{ isLoading ? '登入中...' : 'Login' }}
        </button>
      </div>
    </div>
  </div>
  <ChangePasswordDialog
      v-model:visible="showPasswordDialog"
      :account="currentAccount"
      :token="currentToken"
      @password-changed="handlePasswordChanged"
      @update-token="handleTokenUpdate"
  />
  <!-- 添加驗證碼對話框 -->
  <verification-dialog
      v-model:visible="showVerificationDialog"
      :account="username"
      @unlock-success="handleUnlockSuccess"
  />

  <change-password-dialog
      v-model:visible="showPasswordDialog"
      :account="currentAccount"
      :token="currentToken"
      @password-changed="handlePasswordChanged"
      @update-token="handleTokenUpdate"
  />
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRouter } from 'vue-router'
import axios from 'axios'
import { ElMessage } from 'element-plus'
import { getTokenInfo } from '@/router/guards'
import ChangePasswordDialog from '@/components/shared/ChangePasswordDialog.vue'
import VerificationDialog from '@/components/shared/VerificationDialog.vue'

const router = useRouter()

// 響應式狀態
const username = ref('')
const password = ref('')
const showPassword = ref(false)
const isLoading = ref(false)
const showPasswordDialog = ref(false)
const currentToken = ref('')
const currentAccount = ref('')
// 添加驗證碼對話框的狀態
const showVerificationDialog = ref(false)

// 根據帳號判斷是否為管理員
const isAdmin = computed(() => {
  return username.value.toLowerCase().startsWith('admin')
})

// 路由處理函數
const handleRouting = async (tokenData) => {
  try {
    let targetPath = '/user/profile' // 預設路徑

    if (tokenData.role === 'admin') {
      targetPath = '/admin'
    } else {
      switch(tokenData.department) {
        case 'BD':
          targetPath = '/business/dashboard'
          break
        case 'FD':
          targetPath = '/finance/dashboard'
          break
        case 'LD':
          targetPath = '/loan/dashboard'
          break
      }
    }

    await router.replace(targetPath)
  } catch (error) {
    console.error('Navigation error:', error)
    ElMessage.error('導航錯誤，請重試')
  }
}

// 處理錯誤響應
const handleErrorResponse = (status, data, isAdmin) => {
  switch (status) {
    case 403:
      // 當帳號被鎖定且支援驗證碼解鎖時
      if (data.isLocked && data.requireVerification) {
        showVerificationDialog.value = true
        return
      }

      if (data.lockUntil) {
        const message = isAdmin
            ? `帳號已被鎖定，請 ${Math.ceil((data.lockUntil - Date.now()) / 1000 / 60)} 分鐘後再試`
            : '帳號已被鎖定，請聯繫系統管理員解鎖';

        ElMessage({
          message: message,
          type: 'error',
          duration: 5000,
          showClose: true
        });
      } else {
        ElMessage.error(data.msg || '帳號已被鎖定');
      }
      break;
    case 400:
      if (data.attemptsLeft !== undefined) {
        const warning = isAdmin
            ? `密碼錯誤，還剩 ${data.attemptsLeft} 次嘗試機會，超過將鎖定15分鐘`
            : `密碼錯誤，還剩 ${data.attemptsLeft} 次嘗試機會，超過將鎖定且需聯繫管理員解鎖`;

        ElMessage({
          message: warning,
          type: 'warning',
          duration: 3000,
          showClose: true
        });
      } else {
        ElMessage.error(data.msg);
      }
      break;
    default:
      ElMessage.error(data.msg || '登入失敗，請稍後再試');
  }
};

// 登入處理函數
const handleLogin = async () => {
  if (!username.value || !password.value) {
    ElMessage.warning('請輸入帳號和密碼')
    return
  }

  if (isLoading.value) return
  isLoading.value = true

  try {
    // 清除舊數據
    localStorage.removeItem('token')
    sessionStorage.removeItem('userInfo')

    const response = await axios.post(
        `http://localhost:5000/api/${isAdmin.value ? 'admin' : 'user'}/login`,
        {
          account: username.value,
          password: password.value,
          isAdminMode: isAdmin.value,
          userAgent: navigator.userAgent
        },
        {
          headers: {
            'Cache-Control': 'no-cache',
            'Pragma': 'no-cache',
            'X-Client-Platform': navigator.platform,
            'X-Client-Browser': navigator.vendor
          }
        }
    )

    if (response.data.msg === '登入成功' && response.data.token) {
      try {
        localStorage.setItem('token', response.data.token)
        const tokenInfo = getTokenInfo(response.data.token)

        if (!tokenInfo) {
          throw new Error('Token parsing failed')
        }

        if (isAdmin.value && tokenInfo.role !== 'admin') {
          throw new Error('Invalid admin token')
        }

        if (!isAdmin.value && response.data.user.isFirstLogin) {
          showPasswordDialog.value = true
          currentToken.value = response.data.token
          currentAccount.value = username.value
          return
        }

        ElMessage({
          message: `${isAdmin.value ? '管理員' : '使用者'}登入成功`,
          type: 'success',
          duration: 2000
        })

        await new Promise(resolve => setTimeout(resolve, 300))
        await handleRouting(tokenInfo)

      } catch (tokenError) {
        console.error('Token processing error:', tokenError)
        localStorage.removeItem('token')
        sessionStorage.removeItem('userInfo')
        ElMessage.error('登入過程發生錯誤，請重試')
      }
    } else {
      ElMessage.error(response.data.msg || '登入失敗')
    }
  } catch (error) {
    console.error('Login error:', error)
    if (error.response) {
      const { status, data } = error.response;
      handleErrorResponse(status, data, isAdmin.value);
    } else {
      ElMessage.error('網路錯誤，請檢查連線');
    }
  } finally {
    setTimeout(() => {
      isLoading.value = false
    }, 300)
  }
}

// 密碼更改處理
const handlePasswordChanged = async (newToken) => {
  try {
    localStorage.setItem('token', newToken)
    currentToken.value = newToken

    const tokenInfo = getTokenInfo(newToken)
    if (!tokenInfo) {
      throw new Error('Invalid token after password change')
    }

    showPasswordDialog.value = false

    ElMessage({
      message: '密碼更改成功，即將進入系統',
      type: 'success',
      duration: 2000
    })

    await new Promise(resolve => setTimeout(resolve, 300))
    await handleRouting(tokenInfo)
  } catch (error) {
    console.error('Password change handling error:', error)
    ElMessage.error('處理密碼更改時發生錯誤')

    localStorage.removeItem('token')
    sessionStorage.removeItem('userInfo')
    currentToken.value = ''
  }
}

const handleTokenUpdate = (newToken) => {
  localStorage.setItem('token', newToken)
  currentToken.value = newToken
}

// 處理解鎖成功
const handleUnlockSuccess = async () => {
  ElMessage.success('帳號已解鎖，請重新登入')
  password.value = '' // 清除密碼輸入
};

// 組件掛載時檢查登入狀態
onMounted(async () => {
  await checkLoginStatus()
})

// 檢查登入狀態
const checkLoginStatus = async () => {
  const token = localStorage.getItem('token')
  if (!token) return false

  try {
    const tokenInfo = getTokenInfo(token)
    if (!tokenInfo) {
      localStorage.removeItem('token')
      sessionStorage.removeItem('userInfo')
      return false
    }

    await handleRouting(tokenInfo)
    return true
  } catch (error) {
    console.error('Token check error:', error)
    localStorage.removeItem('token')
    sessionStorage.removeItem('userInfo')
    return false
  }
}
</script>

<style scoped>
.login-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: calc(100vh - 120px);
  background-color: #f5f5f5;
}

.login-box {
  display: flex;
  background: #ffffff;
  border-radius: 10px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  overflow: hidden;
  width: 800px;
}

.login-left {
  flex: 1;
  padding: 2rem;
  background-color: #fff;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  text-align: center;
}

.login-right {
  flex: 1;
  padding: 2rem;
  background-color: #faf9f9;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.login-btn {
  width: 100%;
  padding: 0.8rem;
  background-color: #002b9f;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.login-btn:hover {
  background-color: #001f75;
}

.login-btn.admin-mode {
  background-color: #ff4444;
}

.login-btn.admin-mode:hover {
  background-color: #cc0000;
}

h2 {
  margin-bottom: 2rem;
  font-size: 1.8rem;
  color: #333;
  text-align: center;
}

h3 {
  margin-bottom: 1rem;
  font-size: 1.2rem;
  color: #666;
}

.form-group {
  margin-bottom: 1.5rem;
}

label {
  display: block;
  margin-bottom: 0.5rem;
  color: #666;
}

.form-input {
  width: 100%;
  padding: 0.8rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
  transition: all 0.3s ease;
}

.form-input:focus {
  outline: none;
  border-color: #4a90e2;
}

.password-input-wrapper {
  position: relative;
  width: 100%;
}

.password-toggle {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  cursor: pointer;
  color: #666;
  padding: 5px;
}

.password-toggle:hover {
  color: #333;
}

.password-input-wrapper .form-input {
  padding-right: 40px;
}

p {
  color: #666;
  line-height: 1.6;
}

.login-image {
  width: 80%;
  max-width: 300px;
  margin: 2rem 0;
}

@media (max-width: 768px) {
  .login-box {
    flex-direction: column;
    width: 90%;
    margin: 1rem;
  }

  .login-left,
  .login-right {
    padding: 1.5rem;
  }
}
</style>