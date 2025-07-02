<template>
  <div class="auth-page">
    <div class="auth-container">
      <h1>Reset Password</h1>
      <p class="subtitle">Enter your email address and we'll send you a link to reset your password.</p>
      
      <form @submit.prevent="resetPassword" v-if="!emailSent">
        <div class="form-group">
          <label for="email">Email Address</label>
          <input 
            type="email" 
            id="email" 
            v-model="email" 
            required
            placeholder="Enter your email address"
          >
        </div>
        <button type="submit" class="btn btn-primary" :disabled="loading">
          {{ loading ? 'Sending...' : 'Send Reset Link' }}
        </button>
      </form>

      <!-- Success message -->
      <div v-if="emailSent" class="success-message">
        <div class="success-icon">✓</div>
        <h3>Check your email</h3>
        <p>We've sent a password reset link to <strong>{{ email }}</strong></p>
        <p class="note">Didn't receive the email? Check your spam folder or try again.</p>
        <button @click="resetForm" class="btn btn-secondary">
          Try Again
        </button>
      </div>

      <!-- Error message -->
      <div v-if="errorMessage" class="error-message">
        {{ errorMessage }}
      </div>

      <div class="auth-footer">
        <router-link to="/login">← Back to Login</router-link>
      </div>
    </div>
  </div>
</template>

<script>
import { ref } from 'vue'
import { auth } from '../../services/firebase'
import { sendPasswordResetEmail } from 'firebase/auth'

export default {
  name: 'ForgotPassword',
  setup() {
    const email = ref('')
    const loading = ref(false)
    const emailSent = ref(false)
    const errorMessage = ref('')

    const resetPassword = async () => {
      loading.value = true
      errorMessage.value = ''
      
      try {
        await sendPasswordResetEmail(auth, email.value)
        emailSent.value = true
      } catch (error) {
        console.error('Password reset error:', error)
        
        // Handle specific Firebase errors
        switch (error.code) {
          case 'auth/user-not-found':
            errorMessage.value = 'No account found with this email address.'
            break
          case 'auth/invalid-email':
            errorMessage.value = 'Please enter a valid email address.'
            break
          case 'auth/too-many-requests':
            errorMessage.value = 'Too many requests. Please try again later.'
            break
          default:
            errorMessage.value = 'An error occurred. Please try again.'
        }
      } finally {
        loading.value = false
      }
    }

    const resetForm = () => {
      emailSent.value = false
      errorMessage.value = ''
      email.value = ''
    }

    return {
      email,
      loading,
      emailSent,
      errorMessage,
      resetPassword,
      resetForm
    }
  }
}
</script>

<style scoped>
.auth-page {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background-color: #f5f5f5;
}

.auth-container {
  background: white;
  padding: 30px;
  border-radius: 10px;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.1);
  width: 100%;
  max-width: 400px;
}

.auth-container h1 {
  text-align: center;
  margin-bottom: 10px;
  color: #2c3e50;
}

.subtitle {
  text-align: center;
  color: #7f8c8d;
  margin-bottom: 25px;
  font-size: 14px;
  line-height: 1.4;
}

.form-group {
  margin-bottom: 20px;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: 600;
  color: #2c3e50;
}

.form-group input {
  width: 100%;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 5px;
  font-size: 16px;
  transition: border-color 0.3s;
}

.form-group input:focus {
  border-color: #42b983;
  outline: none;
}

.btn {
  width: 100%;
  padding: 12px;
  border: none;
  border-radius: 5px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.3s;
}

.btn-primary {
  background-color: #42b983;
  color: white;
}

.btn-primary:hover {
  background-color: #369f6e;
}

.btn-primary:disabled {
  background-color: #95a5a6;
  cursor: not-allowed;
}

.btn-secondary {
  background-color: #ecf0f1;
  color: #2c3e50;
  margin-top: 15px;
}

.btn-secondary:hover {
  background-color: #d5dbdb;
}

.success-message {
  text-align: center;
  padding: 20px 0;
}

.success-icon {
  width: 60px;
  height: 60px;
  background-color: #27ae60;
  color: white;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  font-weight: bold;
  margin: 0 auto 20px;
}

.success-message h3 {
  color: #2c3e50;
  margin-bottom: 15px;
}

.success-message p {
  color: #7f8c8d;
  margin-bottom: 10px;
  line-height: 1.4;
}

.success-message .note {
  font-size: 14px;
  margin-bottom: 20px;
}

.error-message {
  background-color: #fee;
  color: #c0392b;
  padding: 12px;
  border-radius: 5px;
  margin-top: 15px;
  text-align: center;
  font-size: 14px;
  border: 1px solid #f8d7da;
}

.auth-footer {
  text-align: center;
  margin-top: 25px;
}

.auth-footer a {
  color: #42b983;
  text-decoration: none;
  font-weight: 600;
  font-size: 14px;
}

.auth-footer a:hover {
  text-decoration: underline;
}

@media (max-width: 480px) {
  .auth-container {
    padding: 20px;
    margin: 0 15px;
  }
  
  .subtitle {
    font-size: 13px;
  }
}
</style>