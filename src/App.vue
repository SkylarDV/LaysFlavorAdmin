<script setup>
import { computed, onMounted, ref } from 'vue'
import BagsList from './components/BagsList.vue'

const username = ref('')
const password = ref('')
const loading = ref(false)
const error = ref('')
const user = ref(null)

const isAuthenticated = computed(() => Boolean(user.value?.admin))

const redirectAway = () => {
  window.location.href = 'https://lays-flavor.vercel.app'
}

const persistUser = (value) => {
  localStorage.setItem('laysAdminAuth', JSON.stringify(value))
}

const loadPersistedUser = () => {
  try {
    const saved = JSON.parse(localStorage.getItem('laysAdminAuth') || 'null')
    if (saved?.admin) {
      user.value = saved
    } else if (saved && saved.admin === false) {
      redirectAway()
    }
  } catch (err) {
    console.error('Failed to read saved auth', err)
    localStorage.removeItem('laysAdminAuth')
  }
}

const logout = () => {
  user.value = null
  localStorage.removeItem('laysAdminAuth')
}

const decodeIdFromToken = (jwt) => {
  if (!jwt) return null
  try {
    const parts = jwt.split('.')
    if (parts.length !== 3) return null
    const payload = JSON.parse(atob(parts[1]))
    return payload?._id || payload?.id || null
  } catch (err) {
    console.log('JWT decode error', err)
    return null
  }
}

const verifyAdmin = async (userId, token) => {
  if (!userId) return { admin: false, userData: null }
  try {
    const res = await fetch(`https://laysflavorapi.onrender.com/api/user/${userId}`, {
      headers: token ? { Authorization: `Bearer ${token}` } : {},
    })
    const raw = await res.json().catch(() => null)
    const admin = raw?.data?.user?.admin ?? raw?.user?.admin ?? raw?.admin ?? false
    const userData = raw?.data?.user || raw?.user || null
    return { admin: Boolean(admin), userData }
  } catch (err) {
    return { admin: false, userData: null }
  }
}

const login = async () => {
  const userInput = username.value.trim()
  const passInput = password.value.trim()

  if (!userInput || !passInput) {
    error.value = 'Enter username/email and password'
    return
  }

  loading.value = true
  error.value = ''

  try {
    // Send both username and email to satisfy backend validation while supporting either input
    const loginPayload = {
      username: userInput,
      email: userInput,
      password: passInput,
    }

    const res = await fetch('https://laysflavorapi.onrender.com/api/user/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(loginPayload),
    })

    const data = await res.json().catch(() => null)

    if (!res.ok) {
      const message = data?.message || data?.error || 'Login failed'
      throw new Error(message)
    }

    const token = data?.data?.token || data?.token || null
    let userId = data?.data?.user?._id || data?.user?._id || data?.data?.user?.id || data?.user?.id || null
    if (!userId) {
      userId = decodeIdFromToken(token)
    }

    const name =
      data?.data?.user?.username ||
      data?.user?.username ||
      data?.data?.user?.email ||
      data?.user?.email ||
      userInput

    const { admin: adminFlag, userData } = await verifyAdmin(userId, token)

    if (!adminFlag) {
      redirectAway()
      return
    }

    const authedUser = {
      username: name,
      token,
      admin: true,
      id: userId,
      email: userData?.email || null,
    }
    user.value = authedUser
    persistUser(authedUser)
  } catch (err) {
    error.value = err?.message || 'Login failed'
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  loadPersistedUser()
})
</script>

<template>
  <div class="app-shell">
    <div v-if="isAuthenticated" class="content">
      <header class="topbar">
        <div class="brand">Lays Admin</div>
        <div class="user-row">
          <span class="user-name">{{ user?.username }}</span>
          <button class="ghost" @click="logout">Log out</button>
        </div>
      </header>
      <BagsList />
    </div>

    <div v-else class="auth-wrapper">
      <div class="login-card">
        <h1>Admin Login</h1>
        <p class="hint">Only admins can view this page.</p>
        <form @submit.prevent="login" class="form">
          <label class="field">
            <span>Username</span>
            <input
              v-model="username"
              type="text"
              name="username"
              autocomplete="username"
              placeholder="Enter your username or email"
              required
            />
          </label>
          <label class="field">
            <span>Password</span>
            <input
              v-model="password"
              type="password"
              name="password"
              autocomplete="current-password"
              placeholder="Enter password"
              required
            />
          </label>
          <button type="submit" :disabled="loading">
            {{ loading ? 'Signing in…' : 'Sign in' }}
          </button>
          <p v-if="error" class="error">{{ error }}</p>
        </form>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app-shell {
  min-height: 100vh;
  background: var(--bg);
  color: var(--text);
}

.content {
  max-width: 1200px;
  margin: 0 auto;
}

.topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 20px 24px 0 24px;
}

.brand {
  font-weight: 800;
  letter-spacing: -0.3px;
}

.user-row {
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 14px;
  color: var(--muted);
}

.user-name {
  font-weight: 700;
  color: var(--text);
}

.ghost {
  background: #fff;
  border: 1px solid rgba(0, 0, 0, 0.08);
  padding: 8px 12px;
  border-radius: 10px;
  font-weight: 700;
  cursor: pointer;
}

.ghost:hover {
  background: #fff7d9;
}

.auth-wrapper {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
}

.login-card {
  background: var(--card);
  border-radius: 16px;
  padding: 24px;
  width: 100%;
  max-width: 420px;
  box-shadow: 0 18px 48px rgba(0, 0, 0, 0.12);
}

.login-card h1 {
  margin: 0 0 8px 0;
}

.hint {
  margin: 0 0 16px 0;
  color: var(--muted);
  font-size: 14px;
}

.form {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-size: 13px;
  color: var(--muted);
}

.field input {
  padding: 10px 12px;
  border: 1px solid rgba(0, 0, 0, 0.12);
  border-radius: 10px;
  font-size: 14px;
  background: #fff;
}

.field input:focus {
  outline: 2px solid #ffd54f;
  border-color: #ffce33;
}

.form button {
  width: 100%;
}

.form button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.error {
  margin: 4px 0 0 0;
  color: #b00020;
  font-size: 13px;
}

@media (max-width: 640px) {
  .login-card {
    padding: 20px;
  }

  .topbar {
    padding: 16px;
  }
}
</style>
