<template>
  <div id="app">
    <!-- Login Screen -->
    <div v-if="!token" class="login-container">
      <div class="login-card">
        <div class="login-header">
          <div class="badge-icon">🛡️</div>
          <h1>Weapon Detection System</h1>
          <p class="subtitle">Law Enforcement Portal</p>
        </div>

        <div class="login-form">
          <div class="input-group">
            <label>Username / Badge Number</label>
            <input
              v-model="loginData.username"
              placeholder="Enter your username"
              class="input-field"
              @input="clearError"
              @keyup.enter="login"
            />
          </div>

          <div class="input-group">
            <label>Password</label>
            <input
              type="password"
              v-model="loginData.password"
              placeholder="Enter your password"
              class="input-field"
              @input="clearError"
              @keyup.enter="login"
            />
          </div>
          
          <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
          
          <button @click="login" class="login-btn" :disabled="isLoading">
            {{ isLoading ? 'Signing in...' : 'Sign In' }}
          </button>
        </div>

        <div class="login-footer">
          <p>Authorized Personnel Only</p>
        </div>
      </div>
    </div>

    <!-- Main App (Logged In) -->
    <MainApp 
      v-if="token"
      :key="appKey"
      :token="token"
      :user-data="userData"
      @logout="handleLogout"
    />

    <!-- NEW: Incident Alert Banner with Close Button -->
    <div v-if="token && currentAlert" class="incident-alert-banner">
      <button @click="dismissAlert" class="alert-close-btn" title="Dismiss alert">✕</button>
      <div class="alert-content">
        <div class="alert-icon">🚨</div>
        <div class="alert-info">
          <strong>NEW INCIDENT!</strong>
          <div class="alert-details">
            <div class="alert-detail-row">
              <span class="alert-label">📍 Location:</span>
              <span class="alert-value">{{ currentAlert.incident.camera_location }}</span>
            </div>
            <div class="alert-detail-row">
              <span class="alert-label">⚔️ Weapon:</span>
              <span :class="['weapon-badge', currentAlert.incident.weapon_type]">
                {{ formatWeaponName(currentAlert.incident.weapon_type) }}
              </span>
            </div>
            <div class="alert-detail-row">
              <span class="alert-label">🕐 Time:</span>
              <span class="alert-value">{{ formatDateTime(currentAlert.incident.detected_at) }}</span>
            </div>
          </div>
        </div>
        <div class="alert-incident-number">{{ currentAlert.incident.incident_number }}</div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import MainApp from './components/MainApp.vue'
import detectionService from './services/detectionService.js'

const token = ref('')
const userData = ref({
  username: '',
  fullName: '',
  role: '',
  userId: null
})

const appKey = ref(0)

const isLoading = ref(false)
const errorMessage = ref('')

const loginData = ref({
  username: '',
  password: ''
})

const currentAlert = ref(null)

let unsubscribeDetection = null

onMounted(() => {
  const savedToken = localStorage.getItem('authToken')
  if (savedToken) {
    token.value = savedToken
    userData.value = {
      username: localStorage.getItem('currentUsername') || '',
      fullName: localStorage.getItem('userFullName') || '',
      role: localStorage.getItem('userRole') || '',
      userId: parseInt(localStorage.getItem('userId')) || null
    }
    
    appKey.value++
    
    startDetectionService()
  }

  if ('Notification' in window && Notification.permission === 'default') {
    Notification.requestPermission()
  }
})

onUnmounted(() => {
  stopDetectionService()
})

function startDetectionService() {
  unsubscribeDetection = detectionService.subscribe((state) => {
    currentAlert.value = state.currentAlert
  })
  
  detectionService.startPolling(token.value)
  
  console.log('✅ Global detection service started')
}

function stopDetectionService() {
  if (unsubscribeDetection) {
    unsubscribeDetection()
    unsubscribeDetection = null
  }
  
  detectionService.stopPolling()
  
  console.log('🛑 Global detection service stopped')
}

function dismissAlert() {
  detectionService.dismissAlert()
}

function clearError() {
  errorMessage.value = ''
}

async function login() {
  clearError()
  
  if (!loginData.value.username || !loginData.value.password) {
    errorMessage.value = 'Please enter username and password'
    return
  }
  
  isLoading.value = true
  
  try {
    const res = await fetch('/api/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(loginData.value)
    })
    
    const data = await res.json()
    
    if (res.ok && data.access_token) {
      token.value = data.access_token
      userData.value = {
        username: data.username,
        fullName: data.full_name || data.username,
        role: data.role || 'officer',
        userId: data.user_id
      }
      
      localStorage.setItem('authToken', data.access_token)
      localStorage.setItem('currentUsername', data.username)
      localStorage.setItem('userFullName', data.full_name || data.username)
      localStorage.setItem('userRole', data.role || 'officer')
      localStorage.setItem('userId', data.user_id)
      
      loginData.value = { username: '', password: '' }
      
      appKey.value++
      console.log('🔄 App key incremented to:', appKey.value)
      
      startDetectionService()
    } else {
      errorMessage.value = data.error || 'Invalid credentials'
    }
  } catch (error) {
    errorMessage.value = 'Network error. Please try again.'
    console.error('Login error:', error)
  }
  
  isLoading.value = false
}

function handleLogout() {
  console.log('🔓 LOGOUT - Starting cleanup...')
  
  stopDetectionService()
  detectionService.reset()
  
  token.value = ''
  userData.value = { username: '', fullName: '', role: '', userId: null }
  currentAlert.value = null
  
  localStorage.removeItem('authToken')
  localStorage.removeItem('currentUsername')
  localStorage.removeItem('userFullName')
  localStorage.removeItem('userRole')
  localStorage.removeItem('userId')
  
  appKey.value++
  
  console.log('✅ LOGOUT complete - all state cleared, app key:', appKey.value)
  
  setTimeout(() => {
    console.log('🔄 Reloading page to ensure clean state...')
    window.location.reload()
  }, 100)
}

function formatWeaponName(weaponType) {
  const names = {
    'gun': 'Pistol',
    'heavy-weapon': 'Heavy Weapon',
    'heavy_weapon': 'Heavy Weapon',
    'knife': 'Knife',
    'pistol': 'Pistol'
  }
  return names[weaponType] || weaponType.replace('-', ' ').replace('_', ' ')
}

function formatDateTime(timestamp) {
  if (!timestamp) return ''
  try {
    const date = new Date(timestamp)
    return `${date.toLocaleDateString()} ${date.toLocaleTimeString()}`
  } catch {
    return ''
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  overflow: hidden;
  background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
}

#app {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
}

.login-container {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  padding: 20px;
}

.login-card {
  background: white;
  border-radius: 16px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  width: 100%;
  max-width: 420px;
  overflow: hidden;
}

.login-header {
  background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
  color: white;
  padding: 40px 30px;
  text-align: center;
}

.badge-icon {
  font-size: 4rem;
  margin-bottom: 15px;
}

.login-header h1 {
  font-size: 1.8rem;
  margin-bottom: 8px;
  font-weight: 700;
}

.subtitle {
  font-size: 1rem;
  opacity: 0.9;
  font-weight: 400;
}

.login-form {
  padding: 35px 30px;
}

.input-group {
  margin-bottom: 22px;
}

.input-group label {
  display: block;
  margin-bottom: 8px;
  color: #2c3e50;
  font-weight: 600;
  font-size: 0.9rem;
}

.input-field {
  width: 100%;
  padding: 14px 16px;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  font-size: 1rem;
  transition: all 0.3s ease;
}

.input-field:focus {
  border-color: #2a5298;
  outline: none;
  box-shadow: 0 0 0 3px rgba(42, 82, 152, 0.1);
}

.error-message {
  background: #fee;
  border: 1px solid #fcc;
  color: #c33;
  padding: 12px;
  border-radius: 6px;
  margin-bottom: 20px;
  font-size: 0.9rem;
  text-align: center;
}

.login-btn {
  width: 100%;
  padding: 14px 20px;
  background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1.05rem;
  font-weight: 600;
  transition: all 0.3s ease;
  box-shadow: 0 4px 12px rgba(30, 60, 114, 0.3);
}

.login-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(30, 60, 114, 0.4);
}

.login-btn:disabled {
  background: #bdc3c7;
  cursor: not-allowed;
  box-shadow: none;
}

.login-footer {
  background: #f8f9fa;
  padding: 20px;
  text-align: center;
  border-top: 1px solid #e0e0e0;
}

.login-footer p {
  color: #7f8c8d;
  font-size: 0.85rem;
  font-weight: 600;
}

/* NEW: Incident Alert Banner Styles */
.incident-alert-banner {
  position: fixed;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 9999;
  min-width: 500px;
  max-width: 90vw;
  background: linear-gradient(135deg, #e74c3c 0%, #c0392b 100%);
  color: white;
  padding: 20px 25px;
  border-radius: 12px;
  box-shadow: 0 8px 24px rgba(231, 76, 60, 0.4);
  animation: slideUp 0.3s ease-out, pulse 2s ease-in-out infinite;
  position: relative;
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateX(-50%) translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
}

@keyframes pulse {
  0%, 100% { 
    box-shadow: 0 8px 24px rgba(231, 76, 60, 0.4);
  }
  50% { 
    box-shadow: 0 8px 32px rgba(231, 76, 60, 0.6);
  }
}

.alert-close-btn {
  position: absolute;
  top: 10px;
  right: 10px;
  background: rgba(255, 255, 255, 0.2);
  color: white;
  border: none;
  border-radius: 50%;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 1.2rem;
  font-weight: bold;
  transition: all 0.3s ease;
  z-index: 10;
}

.alert-close-btn:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: scale(1.1);
}

.alert-content {
  display: flex;
  align-items: flex-start;
  gap: 15px;
  padding-right: 30px;
}

.alert-icon {
  font-size: 2rem;
  animation: shake 0.5s ease-in-out infinite;
  flex-shrink: 0;
}

@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-3px) rotate(-5deg); }
  75% { transform: translateX(3px) rotate(5deg); }
}

.alert-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.alert-info strong {
  font-size: 1.2rem;
  letter-spacing: 0.5px;
  display: block;
  margin-bottom: 5px;
}

.alert-details {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.alert-detail-row {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.95rem;
}

.alert-label {
  opacity: 0.9;
  font-weight: 500;
  min-width: 90px;
}

.alert-value {
  opacity: 0.95;
  font-weight: 600;
}

.weapon-badge {
  padding: 3px 10px;
  border-radius: 10px;
  font-size: 0.85rem;
  font-weight: 600;
  display: inline-block;
}

.weapon-badge.knife {
  background: rgba(255, 255, 255, 0.3);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.4);
}

.weapon-badge.pistol {
  background: rgba(255, 255, 255, 0.3);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.4);
}

.weapon-badge.heavy_weapon {
  background: rgba(255, 255, 255, 0.3);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.4);
}

.alert-incident-number {
  font-size: 0.85rem;
  opacity: 0.9;
  font-weight: 600;
  white-space: nowrap;
  align-self: flex-start;
}

@media (max-width: 768px) {
  .login-card {
    max-width: 100%;
  }
  
  .login-header h1 {
    font-size: 1.5rem;
  }
  
  .badge-icon {
    font-size: 3rem;
  }
  
  .incident-alert-banner {
    min-width: 90vw;
    bottom: 10px;
  }
  
  .alert-content {
    flex-direction: column;
  }
  
  .alert-incident-number {
    align-self: flex-end;
  }
}
</style>