<script setup>
import { ref, onMounted } from 'vue'

const bags = ref([])
const loading = ref(true)
const error = ref(null)

onMounted(async () => {
  try {
    const response = await fetch('https://laysflavorapi.onrender.com/api/bag')
    const data = await response.json()
    
    if (data.status === 'success') {
      bags.value = data.data.bags
    } else {
      error.value = 'Failed to load bags'
    }
  } catch (err) {
    error.value = 'Error fetching bags: ' + err.message
  } finally {
    loading.value = false
  }
})
</script>

<template>
  <div class="bags-container">
    <h1>Lays Flavors</h1>
    
    <div v-if="loading" class="loading">Loading bags...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    
    <div v-else class="bags-grid">
      <div v-for="bag in bags" :key="bag._id" class="bag-card">
        <img :src="bag.bagImage" :alt="bag.name" class="bag-image" />
        
        <div class="colour-square" :style="{ backgroundColor: bag.colour }"></div>
        
        <div class="bag-info">
          <h3>{{ bag.name }}</h3>
          <p>{{ bag.flavor }}</p>
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
  margin: 0 0 2rem 0;
  font-size: 22px;
  letter-spacing: -0.3px;
  color: var(--text);
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
  align-items: center;
  gap: 1rem;
  padding: 1.5rem;
  border: none;
  border-radius: 14px;
  background-color: var(--card);
  width: 100%;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.08);
  transition: transform 0.2s, box-shadow 0.2s;
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
