<template>
  <div class="incidents-tab">
    <div class="incidents-header">
      <h2>🚨 Incident Management</h2>
      <div class="header-actions">
        <select v-model="filterWeapon" @change="loadIncidents" class="filter-select">
          <option :value="null">All Weapons</option>
          <option value="knife">Knife</option>
          <option value="pistol">Pistol</option>
          <option value="heavy_weapon">Heavy Weapon</option>
        </select>

        <select v-model="filterStatus" @change="loadIncidents" class="filter-select">
          <option value="">All Status</option>
          <option value="pending">🔴 Pending</option>
          <option value="responding">🟡 Responding</option>
          <option value="resolved">🟢 Resolved</option>
        </select>
        
        <select v-if="userData.role === 'admin'" v-model="filterOfficer" @change="loadIncidents" class="filter-select">
          <option value="">All Officers</option>
          <option v-for="officer in officers" :key="officer.id" :value="officer.id">
            {{ officer.first_name }} {{ officer.last_name }} ({{ officer.badge_number }})
          </option>
        </select>
        
        <select v-model="dateRangeType" @change="handleDateRangeChange" class="filter-select">
          <option value="preset">Quick Select</option>
          <option value="custom">Custom Range</option>
        </select>

        <select v-if="dateRangeType === 'preset'" v-model="filterDays" @change="loadIncidents" class="filter-select">
          <option :value="1">Today</option>
          <option :value="7">Last 7 days</option>
          <option :value="14">Last 2 weeks</option>
          <option :value="30">Last 30 days</option>
          <option :value="60">Last 2 months</option>
          <option :value="90">Last 3 months</option>
          <option :value="180">Last 6 months</option>
          <option :value="365">Last year</option>
        </select>

        <template v-if="dateRangeType === 'custom'">
          <input 
            type="date" 
            v-model="startDate" 
            @change="loadIncidents"
            :max="endDate"
            class="date-input"
          />
          <input 
            type="date" 
            v-model="endDate" 
            @change="loadIncidents"
            :min="startDate"
            :max="today"
            class="date-input"
          />
        </template>
        
        <!-- View Toggle -->
        <div class="view-toggle">
          <button 
            @click="viewMode = 'horizontal'" 
            :class="['toggle-btn', { active: viewMode === 'horizontal' }]"
            title="Card View"
          >
            ⊞ Cards
          </button>
          <button 
            @click="viewMode = 'vertical'" 
            :class="['toggle-btn', { active: viewMode === 'vertical' }]"
            title="List View"
          >
            ☰ List
          </button>
        </div>

        <button @click="loadIncidents" class="refresh-btn">🔄</button>
      </div>
    </div>

    <!-- Stats Summary -->
    <div class="stats-row">
      <div class="stat-card pending">
        <div class="stat-icon">🔴</div>
        <div class="stat-info">
          <div class="stat-value">{{ stats.pending }}</div>
          <div class="stat-label">Pending</div>
        </div>
      </div>
      <div class="stat-card responding">
        <div class="stat-icon">🟡</div>
        <div class="stat-info">
          <div class="stat-value">{{ stats.responding }}</div>
          <div class="stat-label">Responding</div>
        </div>
      </div>
      <div class="stat-card resolved">
        <div class="stat-icon">🟢</div>
        <div class="stat-info">
          <div class="stat-value">{{ stats.resolved }}</div>
          <div class="stat-label">Resolved</div>
        </div>
      </div>
      <div class="stat-card total">
        <div class="stat-icon">📊</div>
        <div class="stat-info">
          <div class="stat-value">{{ stats.total }}</div>
          <div class="stat-label">Total Incidents</div>
        </div>
      </div>
    </div>

    <!-- Incidents List -->
    <div class="incidents-list">
      <div v-if="isLoading" class="loading">Loading incidents...</div>
      <div v-else-if="sortedIncidents.length === 0" class="no-incidents">
        No incidents found for selected filters
      </div>

      <!-- List View - Table Style -->
      <div v-else-if="viewMode === 'vertical'" class="table-view">
        <div class="table-header">
          <div class="th-status" @click="sortBy('status')">
            Status
            <span v-if="sortColumn === 'status'" class="sort-icon">
              {{ sortDirection === 'asc' ? '▲' : '▼' }}
            </span>
          </div>
          <div class="th-weapon" @click="sortBy('weapon_type')">
            Weapon
            <span v-if="sortColumn === 'weapon_type'" class="sort-icon">
              {{ sortDirection === 'asc' ? '▲' : '▼' }}
            </span>
          </div>
          <div class="th-camera" @click="sortBy('camera_name')">
            Camera
            <span v-if="sortColumn === 'camera_name'" class="sort-icon">
              {{ sortDirection === 'asc' ? '▲' : '▼' }}
            </span>
          </div>
          <div class="th-location" @click="sortBy('camera_location')">
            Location
            <span v-if="sortColumn === 'camera_location'" class="sort-icon">
              {{ sortDirection === 'asc' ? '▲' : '▼' }}
            </span>
          </div>
          <div class="th-time" @click="sortBy('detected_at')">
            Detected At
            <span v-if="sortColumn === 'detected_at'" class="sort-icon">
              {{ sortDirection === 'asc' ? '▲' : '▼' }}
            </span>
          </div>
          <div class="th-officer" @click="sortBy('assigned_to_username')">
            Assigned To
            <span v-if="sortColumn === 'assigned_to_username'" class="sort-icon">
              {{ sortDirection === 'asc' ? '▲' : '▼' }}
            </span>
          </div>
        </div>

        <div 
          v-for="incident in sortedIncidents" 
          :key="incident.id" 
          :class="['table-row', incident.status]"
          @click="selectIncident(incident)"
        >
          <div class="td-status" data-label="Status">
            <span :class="['status-badge', incident.status]">
              {{ formatStatus(incident.status) }}
            </span>
          </div>
          <div class="td-weapon" data-label="Weapon">
            <span :class="['weapon-badge', incident.weapon_type]">
              {{ formatWeaponName(incident.weapon_type) }}
            </span>
          </div>
          <div class="td-camera" data-label="Camera">
            <strong>{{ incident.camera_name }}</strong>
          </div>
          <div class="td-location" data-label="Location">
            {{ incident.camera_location }}
          </div>
          <div class="td-time" data-label="Detected At">
            {{ formatDateTime(incident.detected_at) }}
          </div>
          <div class="td-officer" data-label="Assigned To">
            {{ incident.assigned_to_username || '-' }}
          </div>
        </div>
      </div>

      <!-- Card View -->
      <div v-else class="incident-cards">
        <div v-for="incident in sortedIncidents" :key="incident.id" 
             :class="['incident-card', incident.status]"
             @click="selectIncident(incident)">
          <div class="incident-header-row">
            <div :class="['status-badge', incident.status]">
              {{ formatStatus(incident.status) }}
            </div>
          </div>
          
          <div class="incident-info">
            <div class="info-row">
              <span class="label">Camera:</span>
              <span class="value">{{ incident.camera_name }}</span>
            </div>
            <div class="info-row">
              <span class="label">Location:</span>
              <span class="value">{{ incident.camera_location }}</span>
            </div>
            <div class="info-row">
              <span class="label">Weapon:</span>
              <span :class="['weapon-badge', incident.weapon_type]">
                {{ formatWeaponName(incident.weapon_type) }}
              </span>
            </div>
            <div class="info-row">
              <span class="label">Detected:</span>
              <span class="value">{{ formatDateTime(incident.detected_at) }}</span>
            </div>
            <div v-if="incident.assigned_to_username" class="info-row">
              <span class="label">Assigned:</span>
              <span class="value">{{ incident.assigned_to_username }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Modal (unchanged) -->
    <div v-if="selectedIncident" class="modal-overlay" @click="closeModal">
      <div class="modal-detail" @click.stop>
        <div class="modal-header">
          <h3>{{ selectedIncident.incident_number }}</h3>
          <button @click="closeModal" class="close-btn">✕</button>
        </div>
        
        <div class="modal-content">
          <div class="detail-section">
            <h4>Incident Details</h4>
            <div class="detail-grid">
              <div class="detail-item">
                <label>Incident ID:</label>
                <span class="incident-number-text">{{ selectedIncident.incident_number }}</span>
              </div>
              <div class="detail-item">
                <label>Status:</label>
                <span :class="['status-badge', selectedIncident.status]">
                  {{ formatStatus(selectedIncident.status) }}
                </span>
              </div>
              <div class="detail-item">
                <label>Camera:</label>
                <span>{{ selectedIncident.camera_name }}</span>
              </div>
              <div class="detail-item">
                <label>Location:</label>
                <span>{{ selectedIncident.camera_location }}</span>
              </div>
              <div class="detail-item">
                <label>Weapon Type:</label>
                <span :class="['weapon-badge', selectedIncident.weapon_type]">
                  {{ formatWeaponName(selectedIncident.weapon_type) }}
                </span>
              </div>
              <div class="detail-item">
                <label>Detected At:</label>
                <span>{{ formatDateTime(selectedIncident.detected_at) }}</span>
              </div>
              <div class="detail-item">
                <label>Created By:</label>
                <span>{{ selectedIncident.created_by_username }}</span>
              </div>
              <div v-if="selectedIncident.assigned_to_username" class="detail-item">
                <label>Assigned To:</label>
                <span>{{ selectedIncident.assigned_to_username }}</span>
              </div>
            </div>
          </div>

          <div v-if="selectedIncident.description" class="detail-section">
            <h4>Description</h4>
            <p class="description-text">{{ selectedIncident.description }}</p>
          </div>

          <div v-if="selectedIncident.response_notes" class="detail-section">
            <h4>Response Notes</h4>
            <p class="description-text">{{ selectedIncident.response_notes }}</p>
          </div>

          <div v-if="selectedIncident.resolution_notes" class="detail-section">
            <h4>Resolution Notes</h4>
            <p class="description-text">{{ selectedIncident.resolution_notes }}</p>
          </div>

          <div class="detail-section">
            <h4>Take Action</h4>
            
            <div v-if="selectedIncident.status === 'pending'" class="action-form">
              <div v-if="userData.role === 'admin'" class="form-group">
                <label>Assign to Officer:</label>
                <select v-model="actionData.assigned_to" class="input-field">
                  <option value="">Select Officer</option>
                  <option v-for="officer in officers" :key="officer.id" :value="officer.id">
                    {{ officer.first_name }} {{ officer.last_name }} ({{ officer.badge_number }})
                  </option>
                </select>
              </div>
              
              <div class="form-group">
                <label>Response Notes:</label>
                <textarea v-model="actionData.response_notes" class="textarea-field" 
                          placeholder="Enter response details..." rows="2"></textarea>
              </div>
              
              <button @click="updateStatus('responding')" class="action-btn responding">
                🟡 Start Responding
              </button>
            </div>
            
            <div v-else-if="selectedIncident.status === 'responding'" class="action-form">
              <div class="form-group">
                <label>Resolution Notes:</label>
                <textarea v-model="actionData.resolution_notes" class="textarea-field" 
                          placeholder="Enter resolution details..." rows="2"></textarea>
              </div>
              
              <button @click="updateStatus('resolved')" class="action-btn resolved">
                🟢 Mark as Resolved
              </button>
            </div>
            
            <div v-else class="resolved-message">
              ✓ This incident has been resolved
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

const props = defineProps({
  token: String,
  userData: Object
})

const incidents = ref([])
const officers = ref([])
const selectedIncident = ref(null)
const filterWeapon = ref(null)
const filterStatus = ref('')
const filterOfficer = ref('')
const dateRangeType = ref('preset')
const filterDays = ref(7)
const isLoading = ref(false)
const viewMode = ref('horizontal')

// Sorting
const sortColumn = ref('detected_at')
const sortDirection = ref('desc')

const today = new Date().toISOString().split('T')[0]
const startDate = ref(getDateDaysAgo(7))
const endDate = ref(today)

const actionData = ref({
  assigned_to: '',
  response_notes: '',
  resolution_notes: ''
})

function getDateDaysAgo(days) {
  const date = new Date()
  date.setDate(date.getDate() - days)
  return date.toISOString().split('T')[0]
}

function formatTime(dateTimeString) {
  if (!dateTimeString) return 'N/A'
  try {
    return new Date(dateTimeString).toLocaleTimeString([], { 
      hour: '2-digit', 
      minute: '2-digit' 
    })
  } catch {
    return 'N/A'
  }
}

const filteredIncidents = computed(() => {
  let filtered = [...incidents.value]
  
  // Filter by weapon type
  if (filterWeapon.value) {
    filtered = filtered.filter(incident => incident.weapon_type === filterWeapon.value)
  }
  
  // Filter by date range
  if (dateRangeType.value === 'custom') {
    const start = new Date(startDate.value)
    start.setHours(0, 0, 0, 0)
    const end = new Date(endDate.value)
    end.setHours(23, 59, 59, 999)
    
    filtered = filtered.filter(incident => {
      const incidentDate = new Date(incident.detected_at)
      return incidentDate >= start && incidentDate <= end
    })
  } else {
    const cutoffDate = new Date()
    cutoffDate.setDate(cutoffDate.getDate() - filterDays.value)
    cutoffDate.setHours(0, 0, 0, 0)
    
    filtered = filtered.filter(incident => {
      const incidentDate = new Date(incident.detected_at)
      return incidentDate >= cutoffDate
    })
  }
  
  return filtered
})

const sortedIncidents = computed(() => {
  return [...filteredIncidents.value].sort((a, b) => {
    let aVal = a[sortColumn.value]
    let bVal = b[sortColumn.value]
    
    // Handle date sorting
    if (sortColumn.value === 'detected_at') {
      aVal = new Date(aVal).getTime()
      bVal = new Date(bVal).getTime()
    }
    
    // Handle string sorting (case-insensitive)
    if (typeof aVal === 'string' && typeof bVal === 'string') {
      aVal = aVal.toLowerCase()
      bVal = bVal.toLowerCase()
    }
    
    // Handle null values
    if (aVal === null || aVal === undefined) aVal = ''
    if (bVal === null || bVal === undefined) bVal = ''
    
    if (sortDirection.value === 'asc') {
      return aVal > bVal ? 1 : aVal < bVal ? -1 : 0
    } else {
      return aVal < bVal ? 1 : aVal > bVal ? -1 : 0
    }
  })
})

const stats = computed(() => {
  return {
    pending: filteredIncidents.value.filter(i => i.status === 'pending').length,
    responding: filteredIncidents.value.filter(i => i.status === 'responding').length,
    resolved: filteredIncidents.value.filter(i => i.status === 'resolved').length,
    total: filteredIncidents.value.length
  }
})

onMounted(async () => {
  await loadOfficers()
  await loadIncidents()
  setInterval(loadIncidents, 30000)
})

function sortBy(column) {
  if (sortColumn.value === column) {
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc'
  } else {
    sortColumn.value = column
    sortDirection.value = column === 'detected_at' ? 'desc' : 'asc'
  }
}

function handleDateRangeChange() {
  if (dateRangeType.value === 'preset') {
    filterDays.value = 7
  } else {
    startDate.value = getDateDaysAgo(7)
    endDate.value = today
  }
}

async function loadOfficers() {
  if (props.userData.role !== 'admin') return
  
  try {
    const res = await fetch('/api/officers', {
      headers: { 'Authorization': `Bearer ${props.token}` }
    })
    
    if (res.ok) {
      const data = await res.json()
      officers.value = data.officers
    }
  } catch (error) {
    console.error('Could not load officers:', error)
  }
}

async function loadIncidents() {
  isLoading.value = true
  
  try {
    let url = '/api/incidents?limit=1000'
    if (filterStatus.value) url += `&status=${filterStatus.value}`
    if (filterOfficer.value) url += `&assigned_to=${filterOfficer.value}`
    
    const res = await fetch(url, {
      headers: { 'Authorization': `Bearer ${props.token}` }
    })
    
    if (res.ok) {
      const data = await res.json()
      incidents.value = data.incidents
    }
  } catch (error) {
    console.error('Could not load incidents:', error)
  }
  
  isLoading.value = false
}

function selectIncident(incident) {
  selectedIncident.value = incident
  actionData.value = {
    assigned_to: incident.assigned_to || '',
    response_notes: '',
    resolution_notes: ''
  }
}

function closeModal() {
  selectedIncident.value = null
  actionData.value = {
    assigned_to: '',
    response_notes: '',
    resolution_notes: ''
  }
}

async function updateStatus(newStatus) {
  if (!selectedIncident.value) return
  
  const updates = { status: newStatus }
  
  if (newStatus === 'responding') {
    if (actionData.value.assigned_to) {
      updates.assigned_to = actionData.value.assigned_to
    }
    if (actionData.value.response_notes) {
      updates.response_notes = actionData.value.response_notes
    }
  } else if (newStatus === 'resolved') {
    if (!actionData.value.resolution_notes) {
      alert('Please enter resolution notes')
      return
    }
    updates.resolution_notes = actionData.value.resolution_notes
  }
  
  try {
    const res = await fetch(`/api/incidents/${selectedIncident.value.id}`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${props.token}`
      },
      body: JSON.stringify(updates)
    })
    
    if (res.ok) {
      await loadIncidents()
      closeModal()
    } else {
      alert('Failed to update incident')
    }
  } catch (error) {
    console.error('Could not update incident:', error)
    alert('Network error. Please try again.')
  }
}

function formatStatus(status) {
  const map = {
    'pending': 'Pending',
    'responding': 'Responding',
    'resolved': 'Resolved'
  }
  return map[status] || status
}

function formatWeaponName(weaponType) {
  const names = {
    'knife': 'Knife',
    'pistol': 'Pistol',
    'heavy_weapon': 'Heavy Weapon'
  }
  return names[weaponType] || weaponType
}

function formatDateTime(dateTimeString) {
  if (!dateTimeString) return 'N/A'
  try {
    const date = new Date(dateTimeString)
    return `${date.toLocaleDateString()} ${date.toLocaleTimeString()}`
  } catch {
    return dateTimeString
  }
}
</script>

<style scoped>
.incidents-tab {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.incidents-header {
  background: white;
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 15px;
}

.incidents-header h2 {
  color: #2c3e50;
  margin: 0;
}

.header-actions {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  align-items: center;
}

.view-toggle {
  display: flex;
  border: 2px solid #e0e0e0;
  border-radius: 6px;
  overflow: hidden;
}

.toggle-btn {
  padding: 8px 16px;
  background: white;
  border: none;
  cursor: pointer;
  font-size: 0.95rem;
  color: #2c3e50;
  transition: all 0.3s ease;
  border-right: 1px solid #e0e0e0;
}

.toggle-btn:last-child {
  border-right: none;
}

.toggle-btn.active {
  background: #4a90e2;
  color: white;
  font-weight: 600;
}

.toggle-btn:hover:not(.active) {
  background: #f8f9fa;
}

.filter-select,
.date-input {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  background: white;
  font-size: 0.95rem;
  cursor: pointer;
}

.date-input {
  min-width: 140px;
}

.refresh-btn {
  padding: 8px 16px;
  background: #4a90e2;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.95rem;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.refresh-btn:hover {
  background: #357ab7;
}

.stats-row {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 15px;
}

.stat-card {
  background: white;
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  display: flex;
  align-items: center;
  gap: 15px;
  border-left: 4px solid #ddd;
}

.stat-card.pending { border-left-color: #e74c3c; }
.stat-card.responding { border-left-color: #f39c12; }
.stat-card.resolved { border-left-color: #27ae60; }
.stat-card.total { border-left-color: #3498db; }

.stat-icon {
  font-size: 2.5rem;
}

.stat-value {
  font-size: 2rem;
  font-weight: 700;
  color: #2c3e50;
}

.stat-label {
  font-size: 0.9rem;
  color: #7f8c8d;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.incidents-list {
  background: white;
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  min-height: 400px;
}

.loading,
.no-incidents {
  text-align: center;
  padding: 60px 20px;
  color: #7f8c8d;
  font-style: italic;
}

/* ========== TABLE VIEW (List Mode) ========== */
.table-view {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.table-header {
  display: grid;
  grid-template-columns: 110px 120px 150px 1fr 180px 140px;
  gap: 12px;
  padding: 12px 16px;
  background: #f8f9fa;
  border-radius: 8px;
  font-weight: 600;
  color: #2c3e50;
  font-size: 0.9rem;
}

.table-header > div {
  cursor: pointer;
  user-select: none;
  transition: color 0.2s ease;
}

.table-header > div:hover {
  color: #4a90e2;
}

.sort-icon {
  margin-left: 4px;
  color: #4a90e2;
  font-size: 0.8rem;
}

.table-row {
  display: grid;
  grid-template-columns: 110px 120px 150px 1fr 180px 140px;
  gap: 12px;
  padding: 16px;
  background: #f8f9fa;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  border-left: 4px solid transparent;
  align-items: center;
}

.table-row:hover {
  background: #e9ecef;
  transform: translateX(4px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.table-row.pending { border-left-color: #e74c3c; }
.table-row.responding { border-left-color: #f39c12; }
.table-row.resolved { border-left-color: #27ae60; }

/* ========== CARD VIEW ========== */
.incident-cards {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 15px;
}

.incident-card {
  background: #f8f9fa;
  padding: 18px;
  border-radius: 10px;
  border-left: 4px solid #e74c3c;
  cursor: pointer;
  transition: all 0.3s ease;
  position: relative;
  overflow: hidden;
}

.incident-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.incident-card.pending { 
  border-left-color: #e74c3c;
  box-shadow: 0 0 0 2px rgba(231, 76, 60, 0.2);
}

.incident-card.responding { 
  border-left-color: #f39c12; 
}

.incident-card.resolved { 
  border-left-color: #27ae60; 
}

.incident-header-row {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  margin-bottom: 12px;
}

.incident-info {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 12px;
}

.info-row {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.9rem;
}

.label {
  color: #7f8c8d;
  min-width: 60px;
}

.value {
  color: #2c3e50;
  font-weight: 500;
}

/* Status badges for table */
.status-badge {
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 0.8rem;
  font-weight: 600;
  text-transform: uppercase;
  display: inline-block;
}

.status-badge.pending {
  background: #fee;
  color: #e74c3c;
}

.status-badge.responding {
  background: #fff3cd;
  color: #f39c12;
}

.status-badge.resolved {
  background: #d4edda;
  color: #27ae60;
}

.weapon-badge {
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 0.85rem;
  font-weight: 600;
  display: inline-block;
}

.weapon-badge.knife {
  background: #ffebf4;
  color: #e73c8c;
}

.weapon-badge.pistol {
  background: #e0e2ff;
  color: #3638ca;
}

.weapon-badge.heavy_weapon {
  background: #f3e5f5;
  color: #9b59b6;
}

/* ========== MODAL ========== */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 20px;
}

.modal-detail {
  background: white;
  border-radius: 12px;
  width: 100%;
  max-width: 700px;
  max-height: 85vh;
  display: flex;
  flex-direction: column;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 25px;
  border-bottom: 1px solid #e0e0e0;
  background: #f8f9fa;
  border-radius: 12px 12px 0 0;
}

.modal-header h3 {
  color: #2c3e50;
  margin: 0;
}

.close-btn {
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  color: #7f8c8d;
  transition: color 0.3s ease;
}

.close-btn:hover {
  color: #2c3e50;
}

.modal-content {
  padding: 20px 25px;
  overflow-y: auto;
  flex: 1;
}

.detail-section {
  margin-bottom: 20px;
}

.detail-section h4 {
  color: #2c3e50;
  margin-bottom: 12px;
  border-bottom: 2px solid #e0e0e0;
  padding-bottom: 6px;
  font-size: 0.95rem;
}

.detail-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 12px;
}

.detail-item {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.detail-item label {
  font-weight: 600;
  color: #7f8c8d;
  font-size: 0.85rem;
  text-transform: uppercase;
}

.incident-number-text {
  font-family: 'Courier New', monospace;
  font-weight: 600;
  color: #4a90e2;
}

.description-text {
  color: #2c3e50;
  line-height: 1.5;
}

.action-form {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.form-group label {
  font-weight: 600;
  color: #2c3e50;
}

.input-field,
.textarea-field {
  padding: 8px 10px;
  border: 2px solid #e0e0e0;
  border-radius: 6px;
  transition: border-color 0.3s ease;
}

.input-field:focus,
.textarea-field:focus {
  border-color: #4a90e2;
  outline: none;
}

.textarea-field {
  resize: vertical;
  font-family: inherit;
}

.action-btn {
  padding: 12px 20px;
  border: none;
  border-radius: 8px;
  color: white;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
}

.action-btn.responding {
  background: #f39c12;
}

.action-btn.responding:hover {
  background: #e67e22;
  transform: translateY(-2px);
}

.action-btn.resolved {
  background: #27ae60;
}

.action-btn.resolved:hover {
  background: #229954;
  transform: translateY(-2px);
}

.resolved-message {
  text-align: center;
  padding: 20px;
  background: #d4edda;
  color: #27ae60;
  border-radius: 8px;
  font-weight: 600;
}

/* ========== RESPONSIVE ========== */
@media (max-width: 1200px) {
  .table-header,
  .table-row {
    grid-template-columns: 100px 110px 140px 1fr 160px 120px;
    font-size: 0.85rem;
  }
}

@media (max-width: 768px) {
  .incidents-header {
    flex-direction: column;
    align-items: flex-start;
  }
  
  .header-actions {
    width: 100%;
    flex-direction: column;
  }
  
  .view-toggle,
  .filter-select,
  .date-input,
  .refresh-btn {
    width: 100%;
  }
  
  .incident-cards {
    grid-template-columns: 1fr;
  }
  
  .table-header {
    display: none;
  }
  
  .table-row {
    display: flex;
    flex-direction: column;
    gap: 8px;
    padding: 16px;
  }
  
  .table-row > div {
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  
  .table-row > div::before {
    content: attr(data-label);
    font-weight: 600;
    color: #7f8c8d;
    font-size: 0.85rem;
  }
}
</style>