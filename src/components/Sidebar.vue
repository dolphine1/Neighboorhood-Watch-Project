<template>
  <aside class="sidebar">
    <div class="sidebar-header professional">
      <h2>N-Watch</h2>
      <p>Community Safety System</p>
    </div>
    
    <nav class="sidebar-nav">
      <template v-if="isAdmin">
        <h3>Admin</h3>
        <router-link to="/admin/dashboard" class="nav-link">
          <i class="fas fa-tachometer-alt"></i> Dashboard
        </router-link>
        <router-link to="/admin/manage-incidents" class="nav-link">
          <i class="fas fa-tasks"></i> Manage Incidents
        </router-link>
        <router-link to="/admin/broadcast-alerts" class="nav-link">
          <i class="fas fa-bullhorn"></i> Broadcast Alerts
        </router-link>
        <router-link to="/admin/emergency-alerts" class="nav-link">
          <i class="fas fa-exclamation-triangle"></i> Emergency Alerts
          <span v-if="alertCount > 0" class="alert-count">{{ alertCount }}</span>
        </router-link>
        <router-link to="/admin/neighborhood-map" class="nav-link">
          <i class="fas fa-map"></i> Neighborhood Map
        </router-link>
        <router-link to="/admin/user-management" class="nav-link">
          <i class="fas fa-users"></i> User Management
        </router-link>
        <router-link to="/admin/system-settings" class="nav-link">
          <i class="fas fa-cog"></i> System Settings
        </router-link>
      </template>

      <template v-else-if="isPolice">
        <h3>Police</h3>
        <router-link to="/police/dashboard" class="nav-link">
          <i class="fas fa-tachometer-alt"></i> Dashboard
        </router-link>
        <router-link to="/police/assigned-incidents" class="nav-link">
          <i class="fas fa-clipboard-list"></i> Assigned Incidents
        </router-link>
        <router-link to="/police/emergency-alerts" class="nav-link">
          <i class="fas fa-exclamation-triangle"></i> Emergency Alerts
          <span v-if="alertCount > 0" class="alert-count">{{ alertCount }}</span>
        </router-link>
        <router-link to="/police/neighborhood-map" class="nav-link">
          <i class="fas fa-map"></i> Neighborhood Map
        </router-link>
      </template>
      
      <template v-else>
        <h3>Resident</h3>
        <router-link to="/dashboard" class="nav-link">
          <i class="fas fa-tachometer-alt"></i> Dashboard
        </router-link>
        <router-link to="/report-incident" class="nav-link">
          <i class="fas fa-edit"></i> Report Incident
        </router-link>
        <router-link to="/incident-feed" class="nav-link">
          <i class="fas fa-list"></i> Incident Feed
        </router-link>
        <router-link to="/neighborhood-map" class="nav-link">
          <i class="fas fa-map"></i> Neighborhood Map
        </router-link>
        <router-link to="/broadcast-alerts" class="nav-link">
          <i class="fas fa-bell"></i> Broadcast Alerts
          <span v-if="alertCount > 0" class="alert-count">{{ alertCount }}</span>
        </router-link>
      </template>
    </nav>
  </aside>
</template>

<script>
import { ref, onMounted, onUnmounted } from 'vue'
import { auth } from '../services/firebase'
import { doc, getDoc, collection, query, where, onSnapshot } from 'firebase/firestore'
import { db } from '../services/firebase'

export default {
  name: 'Sidebar',
  setup() {
    const isAdmin = ref(false)
    const isPolice = ref(false)
    const alertCount = ref(0)
    let alertsUnsubscribe = null
    let userUnsubscribe = null

    const checkUserRole = async (user) => {
      try {
        // Get user document from Firestore
        const userDocRef = doc(db, 'users', user.uid)
        const userDoc = await getDoc(userDocRef)
        
        if (userDoc.exists()) {
          const userData = userDoc.data()
          // Check user role and approval status
          isAdmin.value = userData.role === 'admin' && userData.approved
          isPolice.value = userData.role === 'police' && userData.approved
        } else {
          // User document doesn't exist, default to resident
          isAdmin.value = false
          isPolice.value = false
        }
      } catch (error) {
        console.error('Error checking user role:', error)
        isAdmin.value = false
        isPolice.value = false
      }
    }

    const checkForUnviewedAlerts = async (user) => {
      try {
        // Get user's viewed alerts
        const userDocRef = doc(db, 'users', user.uid)
        
        // Listen to user document changes for viewed alerts
        userUnsubscribe = onSnapshot(userDocRef, async (userDoc) => {
          const viewedAlerts = userDoc.exists() ? (userDoc.data().viewedAlerts || []) : []
          
          // Query for active alerts
          let alertsQuery = collection(db, 'emergencies')
          
          // If not admin, only check active alerts
          if (!isAdmin.value) {
            alertsQuery = query(
              alertsQuery,
              where('status', '==', 'active')
            )
          }
          
          // Listen to alerts collection changes
          if (alertsUnsubscribe) {
            alertsUnsubscribe()
          }
          
          alertsUnsubscribe = onSnapshot(alertsQuery, (querySnapshot) => {
            // Count unviewed alerts
            const unviewedCount = querySnapshot.docs.filter(doc => {
              const alertId = doc.id
              return !viewedAlerts.includes(alertId)
            }).length
            
            alertCount.value = unviewedCount
          })
        })
      } catch (error) {
        console.error('Error checking for unviewed alerts:', error)
        alertCount.value = 0
      }
    }

    onMounted(() => {
      // Listen for auth state changes
      auth.onAuthStateChanged(async (user) => {
        if (user) {
          await checkUserRole(user)
          await checkForUnviewedAlerts(user)
        } else {
          isAdmin.value = false
          isPolice.value = false
          alertCount.value = 0
          
          // Clean up listeners
          if (alertsUnsubscribe) {
            alertsUnsubscribe()
            alertsUnsubscribe = null
          }
          if (userUnsubscribe) {
            userUnsubscribe()
            userUnsubscribe = null
          }
        }
      })
    })

    onUnmounted(() => {
      // Clean up listeners when component is unmounted
      if (alertsUnsubscribe) {
        alertsUnsubscribe()
      }
      if (userUnsubscribe) {
        userUnsubscribe()
      }
    })

    return { 
      isAdmin, 
      isPolice,
      alertCount
    }
  }
}
</script>

<style scoped>
.sidebar {
  width: 250px;
  height: 100vh;
  position: fixed;
  left: 0;
  top: 0;
  background: linear-gradient(180deg, #1a252f 0%, #2c3e50 100%);
  color: white;
  padding: 20px 0;
  box-shadow: 2px 0 10px rgba(0, 0, 0, 0.2);
  z-index: 1000;
}

.sidebar-header {
  padding: 0 20px 25px;
  border-bottom: 2px solid rgba(66, 185, 131, 0.3);
  text-align: center;
  position: relative;
}

.sidebar-header::after {
  content: '';
  position: absolute;
  bottom: -2px;
  left: 50%;
  transform: translateX(-50%);
  width: 60px;
  height: 2px;
  background: linear-gradient(90deg, #42b983, #4ecdc4);
  border-radius: 2px;
}

.sidebar-header h2 {
  margin: 0;
  font-size: 1.8rem;
  font-weight: 700;
  background: linear-gradient(135deg, #42b983, #4ecdc4);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
  letter-spacing: 1px;
  position: relative;
}

.sidebar-header h2::before {
  content: '🏘️';
  position: absolute;
  left: -25px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 1.2rem;
}

.sidebar-header p {
  margin: 8px 0 0;
  font-size: 0.85rem;
  opacity: 0.9;
  font-weight: 500;
  color: #bdc3c7;
  text-transform: uppercase;
  letter-spacing: 0.8px;
  font-family: 'Arial', sans-serif;
}

/* Alternative styling option - more professional */
.sidebar-header.professional h2 {
  background: none;
  -webkit-text-fill-color: initial;
  color: #ecf0f1;
  font-family: 'Georgia', serif;
  font-size: 1.6rem;
  border-bottom: 1px solid rgba(66, 185, 131, 0.5);
  padding-bottom: 5px;
  margin-bottom: 8px;
}

.sidebar-header.professional h2::before {
  content: '👁️';
  margin-right: 8px;
  position: static;
  transform: none;
}

.sidebar-header.professional p {
  color: #42b983;
  font-size: 0.8rem;
  font-style: italic;
  text-transform: none;
  letter-spacing: 0.5px;
}

.sidebar-nav {
  padding: 20px 0;
  display: flex;
  flex-direction: column;
  height: calc(100vh - 140px);
}

.sidebar-nav h3 {
  padding: 0 20px;
  font-size: 0.9rem;
  text-transform: uppercase;
  margin: 15px 0 10px;
  opacity: 0.7;
  letter-spacing: 0.5px;
  color: #42b983;
  font-weight: 600;
}

.nav-link {
  display: flex;
  align-items: center;
  padding: 12px 20px;
  color: rgba(255, 255, 255, 0.9);
  text-decoration: none;
  transition: all 0.3s ease;
  font-size: 0.95rem;
  position: relative;
  border-left: 3px solid transparent;
}

.nav-link:hover {
  background: linear-gradient(90deg, rgba(66, 185, 131, 0.1), rgba(78, 205, 196, 0.05));
  color: white;
  border-left-color: rgba(66, 185, 131, 0.5);
}

.nav-link.router-link-exact-active {
  background: linear-gradient(90deg, rgba(66, 185, 131, 0.2), rgba(78, 205, 196, 0.1));
  color: #4ecdc4;
  border-left-color: #42b983;
  box-shadow: inset 0 0 10px rgba(66, 185, 131, 0.1);
}

.nav-link i {
  margin-right: 12px;
  width: 20px;
  text-align: center;
  font-size: 1rem;
}

.alert-count {
  position: absolute;
  top: 50%;
  right: 15px;
  transform: translateY(-50%);
  background: linear-gradient(135deg, #ff4757, #ff3742);
  color: white;
  font-size: 0.75rem;
  font-weight: bold;
  padding: 3px 7px;
  border-radius: 12px;
  min-width: 20px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 3px 8px rgba(255, 71, 87, 0.4);
  animation: pulse-count 2s infinite;
}

@keyframes pulse-count {
  0% {
    opacity: 1;
    transform: translateY(-50%) scale(1);
    box-shadow: 0 3px 8px rgba(255, 71, 87, 0.4);
  }
  50% {
    opacity: 0.8;
    transform: translateY(-50%) scale(1.05);
    box-shadow: 0 4px 12px rgba(255, 71, 87, 0.6);
  }
  100% {
    opacity: 1;
    transform: translateY(-50%) scale(1);
    box-shadow: 0 3px 8px rgba(255, 71, 87, 0.4);
  }
}

/* Responsive adjustments */
@media (max-width: 768px) {
  .sidebar {
    width: 220px;
    transform: translateX(-100%);
    transition: transform 0.3s ease;
  }
  
  .sidebar.active {
    transform: translateX(0);
  }
  
  .sidebar-header h2 {
    font-size: 1.5rem;
  }
  
  .sidebar-header h2::before {
    left: -20px;
    font-size: 1rem;
  }
  
  .alert-count {
    right: 12px;
  }
}
</style>