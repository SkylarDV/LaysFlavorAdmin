<script setup>
import { ref, onMounted, computed } from 'vue'

const activeTab = ref('bags')
const bags = ref([])
const votes = ref([])
const users = ref({})
const allUsers = ref([])
const expandedBags = ref(new Set())
const loading = ref(true)
const error = ref(null)

const voteCounts = computed(() => {
  const counts = {}
  votes.value.forEach(vote => {
    const bagId = vote.bagId
    counts[bagId] = (counts[bagId] || 0) + 1
  })
  return counts
})

const getVoteCount = (bagId) => {
  return voteCounts.value[bagId] || 0
}

const sortedBags = computed(() => {
  return [...bags.value].sort((a, b) => {
    return getVoteCount(b._id) - getVoteCount(a._id)
  })
})

const isExpanded = (bagId) => {
  return expandedBags.value.has(bagId)
}

const fetchUserDetails = async (userId) => {
  if (!userId || users.value[userId]) return
  try {
    const res = await fetch(`https://laysflavorapi.onrender.com/api/user/${userId}`)
    const data = await res.json()
    if (data.status === 'success') {
      users.value[userId] = data.data.user
    }
  } catch (err) {
    console.error('Failed to fetch user', err)
  }
}

const toggleDetails = async (bag) => {
  if (expandedBags.value.has(bag._id)) {
    expandedBags.value.delete(bag._id)
  } else {
    expandedBags.value.add(bag._id)
    console.log('Bag data:', bag)
    const userId = bag.userId || bag.createdBy || bag.creator || bag.user
    if (userId) {
      await fetchUserDetails(userId)
    }
  }
}

const deleteBag = async (bagId) => {
  if (!confirm('Are you sure you want to remove this submission? This action cannot be undone.')) {
    return
  }
  
  try {
    const res = await fetch(`https://laysflavorapi.onrender.com/api/bag/${bagId}`, {
      method: 'DELETE'
    })
    
    if (res.ok) {
      bags.value = bags.value.filter(b => b._id !== bagId)
      expandedBags.value.delete(bagId)
    } else {
      alert('Failed to delete bag')
    }
  } catch (err) {
    alert('Error deleting bag: ' + err.message)
  }
}

const deleteUser = async (userId) => {
  if (!confirm('Are you sure you want to ban this user? This will delete their account permanently.')) {
    return
  }
  
  try {
    const authData = JSON.parse(localStorage.getItem('laysAdminAuth') || '{}')
    const headers = {
      'Content-Type': 'application/json'
    }
    if (authData.token) {
      headers['Authorization'] = `Bearer ${authData.token}`
    }
    
    const res = await fetch(`https://laysflavorapi.onrender.com/api/user/${userId}`, {
      method: 'DELETE',
      headers
    })
    
    const data = await res.json().catch(() => null)
    
    if (res.ok) {
      allUsers.value = allUsers.value.filter(u => u._id !== userId)
    } else {
      const message = data?.message || data?.error || 'Failed to ban user'
      alert(message)
    }
  } catch (err) {
    alert('Error banning user: ' + err.message)
  }
}

onMounted(async () => {
  try {
    const [bagsResponse, votesResponse, usersResponse] = await Promise.all([
      fetch('https://laysflavorapi.onrender.com/api/bag'),
      fetch('https://laysflavorapi.onrender.com/api/vote'),
      fetch('https://laysflavorapi.onrender.com/api/user')
    ])
    
    const bagsData = await bagsResponse.json()
    const votesData = await votesResponse.json()
    const usersData = await usersResponse.json()
    
    if (bagsData.status === 'success') {
      bags.value = bagsData.data.bags
    } else {
      error.value = 'Failed to load bags'
    }
    
    if (votesData.status === 'success') {
      votes.value = votesData.data.votes || []
    }
    
    if (usersData.status === 'success') {
      allUsers.value = usersData.data.users || []
    }
  } catch (err) {
    error.value = 'Error fetching data: ' + err.message
  } finally {
    loading.value = false
  }
})
</script>

<template>
  <div class="bags-container">
    <h1>Lays Admin</h1>
    
    <div class="tabs">
      <button 
        class="tab" 
        :class="{ active: activeTab === 'bags' }" 
        @click="activeTab = 'bags'"
      >
        Bags
      </button>
      <button 
        class="tab" 
        :class="{ active: activeTab === 'users' }" 
        @click="activeTab = 'users'"
      >
        Users
      </button>
    </div>
    
    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    
    <div v-else-if="activeTab === 'bags'" class="bags-grid">
      <div v-for="bag in sortedBags" :key="bag._id" class="bag-card" :class="{ expanded: isExpanded(bag._id) }">
        <div class="card-main">
          <div class="vote-badge">
            <span class="vote-icon">❤️</span>
            <span class="vote-count">{{ getVoteCount(bag._id) }}</span>
          </div>
          
          <img :src="bag.bagImage" :alt="bag.name" class="bag-image" />
          
          <div class="colour-square" :style="{ backgroundColor: bag.colour }"></div>
          
          <div class="bag-info">
            <h3>{{ bag.name }}</h3>
            <p>{{ bag.flavor }}</p>
          </div>
          
          <button class="details-btn" @click="toggleDetails(bag)">
            {{ isExpanded(bag._id) ? 'Hide' : 'Details' }}
          </button>
          
          <button class="remove-btn" @click="deleteBag(bag._id)">
            Remove Inappropriate Submission
          </button>
        </div>
        
        <div v-if="isExpanded(bag._id)" class="card-details">
          <h4>Submission Details</h4>
          <div v-if="(bag.userId || bag.createdBy || bag.creator || bag.user) && users[bag.userId || bag.createdBy || bag.creator || bag.user]" class="user-info">
            <div class="info-row">
              <span class="label">Username:</span>
              <span class="value">{{ users[bag.userId || bag.createdBy || bag.creator || bag.user].username }}</span>
            </div>
            <div class="info-row">
              <span class="label">Email:</span>
              <span class="value">{{ users[bag.userId || bag.createdBy || bag.creator || bag.user].email }}</span>
            </div>
          </div>
          <div v-else-if="bag.userId || bag.createdBy || bag.creator || bag.user" class="loading-user">Loading user details...</div>
          <div v-else class="no-user">No creator information available</div>
        </div>
      </div>
    </div>
    
    <div v-else-if="activeTab === 'users'" class="users-grid">
      <div v-for="user in allUsers" :key="user._id" class="user-card">
        <div class="user-main">
          <div class="user-info-main">
            <h3>{{ user.username }}</h3>
            <p>{{ user.email }}</p>
          </div>
          <div class="admin-badge" v-if="user.admin">
            Admin
          </div>
          <button v-else class="ban-btn" @click="deleteUser(user._id)">
            Ban User
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.bags-container {
  padding: 20px 24px 40px 24px;
  width: 100%;
}

h1 {
  margin: 0 0 1.5rem 0;
  font-size: 22px;
  letter-spacing: -0.3px;
  color: var(--text);
}

.tabs {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 2rem;
  border-bottom: 2px solid rgba(0, 0, 0, 0.06);
}

.tab {
  padding: 12px 24px;
  background: none;
  border: none;
  border-bottom: 3px solid transparent;
  font-weight: 700;
  font-size: 14px;
  color: var(--muted);
  cursor: pointer;
  transition: all 0.2s;
  margin-bottom: -2px;
}

.tab:hover {
  color: var(--text);
  background: rgba(0, 0, 0, 0.02);
}

.tab.active {
  color: var(--accent);
  border-bottom-color: var(--accent);
}

.loading,
.error {
  text-align: center;
  padding: 2rem;
  font-size: 1.1rem;
  color: var(--muted);
}

.error {
  color: #b00020;
}

.bags-grid {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  width: 100%;
}

.bag-card {
  display: flex;
  flex-direction: column;
  gap: 0;
  padding: 1.5rem;
  border: none;
  border-radius: 14px;
  background-color: var(--card);
  width: 100%;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.08);
  transition: transform 0.2s, box-shadow 0.2s;
}

.bag-card.expanded {
  box-shadow: 0 12px 32px rgba(0, 0, 0, 0.12);
}

.card-main {
  display: flex;
  align-items: center;
  gap: 1rem;
  width: 100%;
}

.bag-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 12px 32px rgba(0, 0, 0, 0.12);
}

.bag-image {
  width: 80px;
  height: 80px;
  object-fit: contain;
  flex-shrink: 0;
  border-radius: 10px;
  background: linear-gradient(135deg, #fdf7de, #fff);
  padding: 8px;
}

.colour-square {
  width: 40px;
  height: 40px;
  border-radius: 6px;
  flex-shrink: 0;
  border: 1px solid rgba(0, 0, 0, 0.08);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

.bag-info {
  flex: 1;
  min-width: 0;
}

.bag-info h3 {
  margin: 0 0 0.5rem 0;
  font-size: 16px;
  font-weight: 800;
  color: var(--text);
}

.bag-info p {
  margin: 0;
  font-size: 13px;
  color: var(--muted);
  line-height: 1.4;
}

.vote-badge {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 12px;
  background: rgba(255, 200, 0, 0.1);
  border: 1px solid rgba(255, 200, 0, 0.3);
  border-radius: 8px;
  flex-shrink: 0;
}

.vote-icon {
  font-size: 16px;
  line-height: 1;
}

.vote-count {
  font-size: 14px;
  font-weight: 700;
  color: var(--text);
  min-width: 20px;
  text-align: center;
}

.details-btn {
  padding: 8px 16px;
  background: #fff;
  border: 1px solid rgba(0, 0, 0, 0.08);
  border-radius: 8px;
  font-weight: 700;
  font-size: 13px;
  cursor: pointer;
  transition: all 0.2s;
  flex-shrink: 0;
}

.details-btn:hover {
  background: #f5f5f5;
  border-color: rgba(0, 0, 0, 0.12);
}

.remove-btn {
  padding: 8px 16px;
  background: #fff;
  border: 1px solid #ff4444;
  border-radius: 8px;
  font-weight: 700;
  font-size: 13px;
  color: #b00020;
  cursor: pointer;
  transition: all 0.2s;
  flex-shrink: 0;
}

.remove-btn:hover {
  background: #fff0f0;
  border-color: #cc0000;
  color: #cc0000;
}

.card-details {
  margin-top: 1.5rem;
  padding-top: 1.5rem;
  border-top: 1px solid rgba(0, 0, 0, 0.06);
}

.card-details h4 {
  margin: 0 0 1rem 0;
  font-size: 14px;
  font-weight: 700;
  color: var(--text);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.user-info {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.info-row {
  display: flex;
  gap: 1rem;
  font-size: 14px;
}

.info-row .label {
  font-weight: 700;
  color: var(--muted);
  min-width: 80px;
}

.info-row .value {
  color: var(--text);
}

.loading-user,
.no-user {
  font-size: 13px;
  color: var(--muted);
  font-style: italic;
}

.users-grid {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  width: 100%;
}

.user-card {
  padding: 1.5rem;
  border: none;
  border-radius: 14px;
  background-color: var(--card);
  width: 100%;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.08);
  transition: transform 0.2s, box-shadow 0.2s;
}

.user-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 12px 32px rgba(0, 0, 0, 0.12);
}

.user-main {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.user-info-main {
  flex: 1;
}

.user-info-main h3 {
  margin: 0 0 0.5rem 0;
  font-size: 16px;
  font-weight: 800;
  color: var(--text);
}

.user-info-main p {
  margin: 0;
  font-size: 13px;
  color: var(--muted);
}

.admin-badge {
  padding: 6px 12px;
  background: rgba(255, 200, 0, 0.15);
  border: 1px solid rgba(255, 200, 0, 0.4);
  border-radius: 6px;
  font-size: 12px;
  font-weight: 700;
  color: #997500;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.ban-btn {
  padding: 8px 16px;
  background: #fff;
  border: 1px solid #ff4444;
  border-radius: 8px;
  font-weight: 700;
  font-size: 13px;
  color: #b00020;
  cursor: pointer;
  transition: all 0.2s;
  flex-shrink: 0;
}

.ban-btn:hover {
  background: #fff0f0;
  border-color: #cc0000;
  color: #cc0000;
}

@media (max-width: 768px) {
  .bags-container {
    padding: 16px 20px 32px 20px;
  }

  .bag-card {
    gap: 12px;
    padding: 12px;
  }

  .bag-image {
    width: 60px;
    height: 60px;
  }

  .colour-square {
    width: 32px;
    height: 32px;
  }

  .bag-info h3 {
    font-size: 14px;
  }

  .bag-info p {
    font-size: 12px;
  }
}
</style>
