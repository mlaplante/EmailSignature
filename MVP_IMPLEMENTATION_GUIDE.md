# MVP Implementation Guide

This guide provides step-by-step instructions for implementing the **Minimum Viable Product** (MVP) version of the subscription feature, which allows for quick market validation before building the full automated system.

## MVP Approach: Manual Premium Access

**Goal**: Launch in 2-3 weeks with manual premium upgrades to validate demand before investing in full automation.

**Key Differences from Full Implementation**:
- ✅ No payment processing initially
- ✅ Manual tier assignment (via Firebase Console or admin script)
- ✅ Users request premium via email/form
- ✅ Quick validation of willingness to pay
- ✅ Lower technical complexity
- ✅ Can add Stripe later once validated

## Week 1: Setup & Authentication

### Day 1-2: Firebase Setup

#### Step 1: Create Firebase Project
1. Go to [Firebase Console](https://console.firebase.google.com/)
2. Click "Add Project"
3. Name: "email-signature-generator"
4. Disable Google Analytics (or enable if you want metrics)
5. Create project

#### Step 2: Enable Authentication
1. In Firebase Console, go to Authentication
2. Click "Get Started"
3. Enable "Google" sign-in method
4. Add authorized domain: `mlaplante.github.io`
5. Note down Web API Key

#### Step 3: Create Firestore Database
1. In Firebase Console, go to Firestore Database
2. Click "Create Database"
3. Start in **Production Mode** (we'll add rules later)
4. Choose a location (e.g., `us-central1`)
5. Click "Enable"

#### Step 4: Set Up Security Rules
In Firestore, go to Rules tab and add:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Users can read their own document
    match /users/{userId} {
      allow read: if request.auth != null && request.auth.uid == userId;
      allow write: if false; // Only admins can write (via Admin SDK)
    }
    
    // Users can read/write their own signatures
    match /signatures/{signatureId} {
      allow read, write: if request.auth != null && 
                          resource.data.userId == request.auth.uid;
      allow create: if request.auth != null;
    }
  }
}
```

#### Step 5: Get Firebase Config
1. Go to Project Settings (gear icon) > General
2. Scroll to "Your apps" section
3. Click web icon (</>)
4. Register app: "Email Signature Generator"
5. Copy the firebaseConfig object

### Day 3-4: Add Firebase to Application

#### Step 1: Update index.html CSP
Find the CSP meta tag and update it:

```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self'; 
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; 
  font-src 'self' https://fonts.gstatic.com; 
  script-src 'self' 'unsafe-inline' https://www.gstatic.com https://apis.google.com; 
  img-src 'self' data: https:; 
  connect-src 'self' https://*.googleapis.com https://*.firebaseio.com;
  frame-src https://accounts.google.com;
">
```

#### Step 2: Add Firebase SDKs
Add before the closing `</body>` tag in index.html:

```html
<!-- Firebase App (the core Firebase SDK) -->
<script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-firestore-compat.js"></script>

<script>
// Firebase configuration
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "your-project.firebaseapp.com",
  projectId: "your-project-id",
  storageBucket: "your-project.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abcdef123456"
};

// Initialize Firebase
firebase.initializeApp(firebaseConfig);
const auth = firebase.auth();
const db = firebase.firestore();
</script>
```

#### Step 3: Add Authentication UI
Add this HTML after the theme toggle button (around line 700):

```html
<!-- User Authentication -->
<div class="auth-container" id="authContainer">
    <button class="auth-button" id="signInBtn">
        <svg width="18" height="18" viewBox="0 0 18 18" fill="currentColor">
            <path d="M9 0C4.03 0 0 4.03 0 9s4.03 9 9 9 9-4.03 9-9-4.03-9-9-9zm0 16.2c-3.97 0-7.2-3.23-7.2-7.2S5.03 1.8 9 1.8s7.2 3.23 7.2 7.2-3.23 7.2-7.2 7.2zm0-12.6c-1.99 0-3.6 1.61-3.6 3.6s1.61 3.6 3.6 3.6 3.6-1.61 3.6-3.6-1.61-3.6-3.6-3.6z"/>
        </svg>
        Sign In
    </button>
    
    <div class="user-menu" id="userMenu" style="display: none;">
        <img class="user-avatar" id="userAvatar" src="" alt="User">
        <div class="user-info">
            <div class="user-name" id="userName"></div>
            <div class="user-tier" id="userTier">Free Tier</div>
        </div>
        <div class="user-dropdown" id="userDropdown">
            <div class="user-dropdown-item" id="upgradeBtn">
                ⭐ Upgrade to Premium
            </div>
            <div class="user-dropdown-item" id="accountBtn">
                ⚙️ Account Settings
            </div>
            <div class="user-dropdown-item" id="signOutBtn">
                🚪 Sign Out
            </div>
        </div>
    </div>
</div>
```

#### Step 4: Add Authentication Logic
Add this JavaScript after Firebase initialization:

```javascript
// Authentication Manager
class AuthManager {
    constructor() {
        this.user = null;
        this.subscriptionTier = 'free';
        this.userData = null;
        this.init();
    }

    init() {
        // Listen for auth state changes
        auth.onAuthStateChanged(async (user) => {
            this.user = user;
            if (user) {
                await this.loadUserData(user.uid);
                this.updateAuthUI(true);
                featureGatingController.applyFeatureGating(this.subscriptionTier);
            } else {
                this.subscriptionTier = 'free';
                this.updateAuthUI(false);
                featureGatingController.applyFeatureGating('free');
            }
        });

        // Set up UI listeners
        document.getElementById('signInBtn')?.addEventListener('click', () => this.signIn());
        document.getElementById('signOutBtn')?.addEventListener('click', () => this.signOut());
        document.getElementById('upgradeBtn')?.addEventListener('click', () => this.requestUpgrade());
        document.getElementById('userAvatar')?.addEventListener('click', () => this.toggleUserMenu());
        
        // Close dropdown when clicking outside
        document.addEventListener('click', (e) => {
            if (!e.target.closest('.user-menu')) {
                document.getElementById('userDropdown').classList.remove('show');
            }
        });
    }

    async signIn() {
        const provider = new firebase.auth.GoogleAuthProvider();
        try {
            const result = await auth.signInWithPopup(provider);
            showToast('Signed in successfully!', 'success');
            
            // Create user document if it doesn't exist
            const userRef = db.collection('users').doc(result.user.uid);
            const userDoc = await userRef.get();
            
            if (!userDoc.exists) {
                await userRef.set({
                    email: result.user.email,
                    displayName: result.user.displayName,
                    photoURL: result.user.photoURL,
                    subscriptionTier: 'free',
                    createdAt: firebase.firestore.FieldValue.serverTimestamp(),
                    updatedAt: firebase.firestore.FieldValue.serverTimestamp()
                });
            }
        } catch (error) {
            console.error('Sign in error:', error);
            showToast('Failed to sign in. Please try again.', 'error');
        }
    }

    async signOut() {
        try {
            await auth.signOut();
            showToast('Signed out successfully', 'success');
        } catch (error) {
            console.error('Sign out error:', error);
            showToast('Failed to sign out', 'error');
        }
    }

    async loadUserData(uid) {
        try {
            const userDoc = await db.collection('users').doc(uid).get();
            if (userDoc.exists) {
                this.userData = userDoc.data();
                this.subscriptionTier = this.userData.subscriptionTier || 'free';
            } else {
                this.subscriptionTier = 'free';
            }
        } catch (error) {
            console.error('Error loading user data:', error);
            this.subscriptionTier = 'free';
        }
    }

    updateAuthUI(isSignedIn) {
        const authContainer = document.getElementById('authContainer');
        const signInBtn = document.getElementById('signInBtn');
        const userMenu = document.getElementById('userMenu');

        if (isSignedIn && this.user) {
            signInBtn.style.display = 'none';
            userMenu.style.display = 'flex';
            
            document.getElementById('userAvatar').src = this.user.photoURL || 'data:image/svg+xml,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="gray"><circle cx="12" cy="12" r="10"/></svg>';
            document.getElementById('userName').textContent = this.user.displayName || 'User';
            
            const tierText = this.subscriptionTier === 'premium' ? '⭐ Premium' : 'Free Tier';
            document.getElementById('userTier').textContent = tierText;
            document.getElementById('userTier').className = `user-tier tier-${this.subscriptionTier}`;
            
            // Hide upgrade button if already premium
            document.getElementById('upgradeBtn').style.display = 
                this.subscriptionTier === 'premium' ? 'none' : 'block';
        } else {
            signInBtn.style.display = 'flex';
            userMenu.style.display = 'none';
        }
    }

    toggleUserMenu() {
        document.getElementById('userDropdown').classList.toggle('show');
    }

    requestUpgrade() {
        featureGatingController.showUpgradeModal('Premium Access');
    }

    hasFeature(featureName) {
        if (this.subscriptionTier === 'premium' || this.subscriptionTier === 'enterprise') {
            return true;
        }
        
        // Free tier features
        const freeTierFeatures = ['name', 'title', 'email', 'phone'];
        return freeTierFeatures.includes(featureName);
    }
}

// Initialize auth manager
const authManager = new AuthManager();
```

### Day 5: Add Feature Gating

Add this JavaScript to implement feature locking:

```javascript
// Feature Gating Controller
class FeatureGatingController {
    constructor() {
        this.premiumFields = [
            { id: 'company', name: 'Company Name', label: 'Company Name' },
            { id: 'address', name: 'Office Address', label: 'Office Address' },
            { id: 'bookMe', name: 'Book Me Link', label: 'Book Me Link' },
            { id: 'disclaimer', name: 'Legal Disclaimer', label: 'Legal Disclaimer' }
        ];
        
        this.premiumFeatures = {
            logo: 'Custom Logo',
            layouts: 'Horizontal Layout',
            fontSize: 'Font Size Control',
            icons: 'Icon Toggle',
            themes: 'Advanced Brand Themes'
        };
    }

    applyFeatureGating(tier) {
        if (tier === 'free') {
            this.lockPremiumFields();
            this.lockPremiumFeatures();
        } else {
            this.unlockAllFeatures();
        }
    }

    lockPremiumFields() {
        this.premiumFields.forEach(field => {
            const element = document.getElementById(field.id);
            if (!element) return;
            
            const inputGroup = element.closest('.input-group');
            if (!inputGroup) return;
            
            const label = inputGroup.querySelector('label');
            
            // Disable input
            element.disabled = true;
            element.placeholder = 'Premium Feature - Sign in and upgrade to unlock';
            element.style.cursor = 'not-allowed';
            element.style.opacity = '0.6';
            
            // Add premium badge
            if (!label.querySelector('.premium-badge')) {
                const badge = document.createElement('span');
                badge.className = 'premium-badge';
                badge.textContent = '⭐ Premium';
                label.appendChild(badge);
            }
            
            // Add click handler
            const clearBtn = inputGroup.querySelector('.clear-btn');
            if (clearBtn) {
                clearBtn.style.display = 'none';
            }
            
            element.addEventListener('click', (e) => {
                e.preventDefault();
                this.showUpgradeModal(field.label);
            });
        });
    }

    lockPremiumFeatures() {
        // Lock logo upload
        const logoBtn = document.getElementById('uploadBtn');
        const logoGroup = logoBtn.closest('.input-group');
        const logoLabel = logoGroup.querySelector('label');
        
        logoBtn.disabled = true;
        logoBtn.style.opacity = '0.6';
        logoBtn.style.cursor = 'not-allowed';
        
        if (!logoLabel.querySelector('.premium-badge')) {
            const badge = document.createElement('span');
            badge.className = 'premium-badge';
            badge.textContent = '⭐ Premium';
            logoLabel.appendChild(badge);
        }
        
        logoBtn.addEventListener('click', (e) => {
            e.preventDefault();
            this.showUpgradeModal('Custom Logo');
        });
        
        // Lock horizontal layout
        const horizontalRadio = document.getElementById('layoutHorizontal');
        const horizontalLabel = document.querySelector('label[for="layoutHorizontal"]');
        
        horizontalRadio.disabled = true;
        horizontalLabel.style.opacity = '0.6';
        horizontalLabel.style.cursor = 'not-allowed';
        
        // Lock font size
        const fontSizeInput = document.getElementById('fontSize');
        fontSizeInput.disabled = true;
        fontSizeInput.style.opacity = '0.6';
        
        // Lock icon toggle
        const showIconsCheckbox = document.getElementById('showIcons');
        showIconsCheckbox.disabled = true;
        showIconsCheckbox.parentElement.style.opacity = '0.6';
        
        // Lock premium brand themes
        const brandSelect = document.getElementById('brand');
        const premiumBrands = ['creative-purple', 'eco-green', 'energetic-orange', 
                               'modern-teal', 'elegant-burgundy', 'tech-cyan', 'minimalist-gray'];
        
        Array.from(brandSelect.options).forEach(option => {
            if (premiumBrands.includes(option.value)) {
                option.disabled = true;
                option.textContent = option.textContent + ' 🔒';
            }
        });
    }

    unlockAllFeatures() {
        // Remove premium badges
        document.querySelectorAll('.premium-badge').forEach(el => el.remove());
        
        // Enable all fields
        this.premiumFields.forEach(field => {
            const element = document.getElementById(field.id);
            if (element) {
                element.disabled = false;
                element.style.opacity = '1';
                element.style.cursor = 'text';
                
                const clearBtn = element.closest('.input-group')?.querySelector('.clear-btn');
                if (clearBtn) clearBtn.style.display = '';
            }
        });
        
        // Enable logo
        const logoBtn = document.getElementById('uploadBtn');
        logoBtn.disabled = false;
        logoBtn.style.opacity = '1';
        logoBtn.style.cursor = 'pointer';
        
        // Enable layout
        const horizontalRadio = document.getElementById('layoutHorizontal');
        const horizontalLabel = document.querySelector('label[for="layoutHorizontal"]');
        horizontalRadio.disabled = false;
        horizontalLabel.style.opacity = '1';
        horizontalLabel.style.cursor = 'pointer';
        
        // Enable controls
        document.getElementById('fontSize').disabled = false;
        document.getElementById('fontSize').style.opacity = '1';
        document.getElementById('showIcons').disabled = false;
        document.getElementById('showIcons').parentElement.style.opacity = '1';
        
        // Enable all themes
        const brandSelect = document.getElementById('brand');
        Array.from(brandSelect.options).forEach(option => {
            option.disabled = false;
            option.textContent = option.textContent.replace(' 🔒', '');
        });
    }

    showUpgradeModal(featureName) {
        // Check if user is signed in
        if (!authManager.user) {
            showToast('Please sign in to upgrade to Premium', 'info');
            document.getElementById('signInBtn')?.click();
            return;
        }
        
        // Show upgrade modal
        const existingModal = document.getElementById('upgradeModal');
        if (existingModal) {
            existingModal.remove();
        }
        
        const modal = this.createUpgradeModal(featureName);
        document.body.appendChild(modal);
        modal.style.display = 'flex';
    }

    createUpgradeModal(featureName) {
        const modal = document.createElement('div');
        modal.id = 'upgradeModal';
        modal.className = 'modal';
        modal.innerHTML = `
            <div class="modal-content">
                <button class="modal-close" onclick="document.getElementById('upgradeModal').remove()">&times;</button>
                <div class="modal-header">
                    <h2>🌟 Unlock Premium Features</h2>
                    <p class="modal-subtitle">Get access to <strong>${featureName}</strong> and more!</p>
                </div>
                <div class="pricing-card">
                    <h3>Premium Plan</h3>
                    <div class="price">$9.99<span>/month</span></div>
                    <div class="price-annual">or $99/year (save 17%)</div>
                    <ul class="features-list">
                        <li>✓ Custom Logo Upload</li>
                        <li>✓ Company Name, Address & Legal Disclaimer</li>
                        <li>✓ Book Me Link Integration</li>
                        <li>✓ All 10 Brand Themes</li>
                        <li>✓ Both Layout Options</li>
                        <li>✓ Font Size Control</li>
                        <li>✓ Icon Toggle</li>
                        <li>✓ Priority Support</li>
                    </ul>
                    <button class="btn-upgrade" onclick="requestPremiumAccess()">Request Premium Access</button>
                    <p class="upgrade-note">
                        📧 For now, premium upgrades are handled manually. 
                        Click above to send us an email request, and we'll activate your premium access within 24 hours!
                    </p>
                </div>
            </div>
        `;
        
        // Close on background click
        modal.addEventListener('click', (e) => {
            if (e.target === modal) {
                modal.remove();
            }
        });
        
        return modal;
    }
}

// Initialize feature gating
const featureGatingController = new FeatureGatingController();

// Request premium access function
function requestPremiumAccess() {
    const subject = encodeURIComponent('Premium Access Request - Email Signature Generator');
    const body = encodeURIComponent(`Hi,

I would like to upgrade to Premium for the Email Signature Generator.

My account email: ${authManager.user?.email || 'Not signed in'}
User ID: ${authManager.user?.uid || 'N/A'}

Please activate premium access for my account.

Thank you!`);
    
    window.open(`mailto:your-email@example.com?subject=${subject}&body=${body}`, '_blank');
    
    showToast('Email client opened. Send the email to request premium access!', 'info');
    document.getElementById('upgradeModal')?.remove();
}
```

## Week 2: Polish & Testing

### Day 6-7: Add Required CSS

Add this CSS to the `<style>` section in index.html:

```css
/* Authentication UI */
.auth-container {
    position: fixed;
    top: 2rem;
    right: 6rem;
    z-index: 1000;
}

.auth-button {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.75rem 1.5rem;
    background: var(--accent);
    color: white;
    border: none;
    border-radius: 24px;
    font-size: 0.95rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.3s ease;
    box-shadow: 0 2px 8px rgba(217, 119, 87, 0.3);
}

.auth-button:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(217, 119, 87, 0.4);
}

.user-menu {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    padding: 0.5rem 1rem;
    background: var(--bg-secondary);
    border: 1px solid var(--border);
    border-radius: 24px;
    cursor: pointer;
    transition: all 0.2s ease;
    position: relative;
    box-shadow: 0 2px 8px var(--shadow);
}

.user-menu:hover {
    box-shadow: 0 4px 12px var(--shadow);
}

.user-avatar {
    width: 36px;
    height: 36px;
    border-radius: 50%;
    object-fit: cover;
}

.user-info {
    display: flex;
    flex-direction: column;
    gap: 0.125rem;
}

.user-name {
    font-size: 0.9rem;
    font-weight: 500;
    color: var(--text-primary);
}

.user-tier {
    font-size: 0.75rem;
    color: var(--text-muted);
}

.user-tier.tier-premium {
    color: #667eea;
    font-weight: 600;
}

.user-dropdown {
    position: absolute;
    top: calc(100% + 0.5rem);
    right: 0;
    background: var(--bg-secondary);
    border: 1px solid var(--border);
    border-radius: 12px;
    box-shadow: 0 8px 24px var(--shadow);
    min-width: 220px;
    display: none;
    overflow: hidden;
    z-index: 1001;
}

.user-dropdown.show {
    display: block;
    animation: slideDown 0.2s ease;
}

@keyframes slideDown {
    from {
        opacity: 0;
        transform: translateY(-10px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.user-dropdown-item {
    padding: 0.875rem 1rem;
    cursor: pointer;
    transition: background 0.2s ease;
    font-size: 0.9rem;
    color: var(--text-primary);
}

.user-dropdown-item:hover {
    background: var(--bg-primary);
}

.user-dropdown-item:not(:last-child) {
    border-bottom: 1px solid var(--border);
}

/* Premium Badge */
.premium-badge {
    display: inline-block;
    margin-left: 0.5rem;
    padding: 0.125rem 0.5rem;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    border-radius: 8px;
    font-size: 0.7rem;
    font-weight: 600;
    vertical-align: middle;
}

/* Upgrade Modal */
.modal {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.6);
    display: none;
    align-items: center;
    justify-content: center;
    z-index: 10000;
    padding: 1rem;
}

.modal-content {
    background: var(--bg-secondary);
    border-radius: 16px;
    max-width: 600px;
    width: 100%;
    max-height: 90vh;
    overflow-y: auto;
    padding: 2.5rem;
    position: relative;
    box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
    animation: modalSlideIn 0.3s ease;
}

@keyframes modalSlideIn {
    from {
        opacity: 0;
        transform: translateY(30px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.modal-close {
    position: absolute;
    top: 1rem;
    right: 1rem;
    background: none;
    border: none;
    font-size: 2rem;
    color: var(--text-muted);
    cursor: pointer;
    width: 40px;
    height: 40px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    transition: all 0.2s ease;
    line-height: 1;
}

.modal-close:hover {
    background: var(--bg-primary);
    color: var(--text-primary);
}

.modal-header {
    text-align: center;
    margin-bottom: 2rem;
}

.modal-header h2 {
    font-family: 'Crimson Pro', serif;
    font-size: 2rem;
    margin-bottom: 0.5rem;
}

.modal-subtitle {
    color: var(--text-muted);
    font-size: 1.05rem;
}

.pricing-card {
    background: var(--bg-primary);
    border: 2px solid var(--border);
    border-radius: 12px;
    padding: 2rem;
    text-align: center;
}

.pricing-card h3 {
    font-size: 1.5rem;
    margin-bottom: 1rem;
    color: var(--text-primary);
}

.price {
    font-size: 3rem;
    font-weight: 700;
    color: var(--accent);
    margin-bottom: 0.25rem;
}

.price span {
    font-size: 1.25rem;
    color: var(--text-muted);
    font-weight: 400;
}

.price-annual {
    color: var(--text-muted);
    font-size: 0.95rem;
    margin-bottom: 1.5rem;
}

.features-list {
    list-style: none;
    text-align: left;
    margin: 1.5rem 0;
}

.features-list li {
    padding: 0.5rem 0;
    color: var(--text-primary);
    font-size: 0.95rem;
}

.btn-upgrade {
    width: 100%;
    padding: 1rem 2rem;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    border: none;
    border-radius: 10px;
    font-size: 1.1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
    margin-top: 1rem;
}

.btn-upgrade:hover {
    transform: translateY(-2px);
    box-shadow: 0 8px 24px rgba(102, 126, 234, 0.4);
}

.upgrade-note {
    margin-top: 1.5rem;
    padding: 1rem;
    background: var(--bg-secondary);
    border-radius: 8px;
    font-size: 0.875rem;
    color: var(--text-muted);
    line-height: 1.5;
}

/* Responsive adjustments */
@media (max-width: 768px) {
    .auth-container {
        top: 1rem;
        right: 1rem;
    }
    
    .user-info {
        display: none;
    }
    
    .modal-content {
        padding: 1.5rem;
    }
    
    .modal-header h2 {
        font-size: 1.5rem;
    }
    
    .price {
        font-size: 2.5rem;
    }
}
```

### Day 8: Testing

#### Manual Testing Checklist
- [ ] Click "Sign In" → Google sign-in opens
- [ ] After sign-in, user menu appears with name and avatar
- [ ] User tier shows "Free Tier"
- [ ] Premium fields are disabled and show lock badges
- [ ] Clicking locked field shows upgrade modal
- [ ] "Request Premium Access" opens email client
- [ ] Sign out works correctly
- [ ] After sign out, fields remain locked
- [ ] Page refresh maintains auth state
- [ ] Multiple tabs sync auth state

#### Manual Premium Upgrade Testing
1. Sign in with test account
2. Go to Firebase Console > Firestore
3. Find the user's document in `users/{userId}`
4. Change `subscriptionTier` from `'free'` to `'premium'`
5. Refresh the application
6. Verify all fields are unlocked
7. Verify "⭐ Premium" appears in user menu
8. Verify "Upgrade" button is hidden in dropdown

## Week 3: Launch & Documentation

### Day 9: Create Admin Script for Manual Upgrades

Create a Node.js script for easy premium upgrades:

```javascript
// scripts/upgrade-user.js
const admin = require('firebase-admin');
const serviceAccount = require('./serviceAccountKey.json');

admin.initializeApp({
  credential: admin.credential.cert(serviceAccount)
});

const db = admin.firestore();

async function upgradeUser(email, tier = 'premium') {
  try {
    // Find user by email
    const usersSnapshot = await db.collection('users')
      .where('email', '==', email)
      .get();
    
    if (usersSnapshot.empty) {
      console.log(`No user found with email: ${email}`);
      return;
    }
    
    // Update first matching user
    const userDoc = usersSnapshot.docs[0];
    await userDoc.ref.update({
      subscriptionTier: tier,
      updatedAt: admin.firestore.FieldValue.serverTimestamp()
    });
    
    console.log(`✅ Successfully upgraded ${email} to ${tier}`);
  } catch (error) {
    console.error('Error upgrading user:', error);
  }
}

// Get email from command line
const email = process.argv[2];
const tier = process.argv[3] || 'premium';

if (!email) {
  console.log('Usage: node upgrade-user.js <email> [tier]');
  console.log('Example: node upgrade-user.js user@example.com premium');
  process.exit(1);
}

upgradeUser(email, tier).then(() => {
  process.exit(0);
});
```

### Day 10: Update Documentation

Update README.md with subscription information:

```markdown
## Subscription Tiers

### Free Tier
- Basic signature fields (Name, Title, Email, Phone)
- 3 brand themes
- Vertical layout
- Copy to clipboard
- Export HTML

### Premium Tier ($9.99/month or $99/year)
- All free features plus:
- Custom logo upload
- Company name, address, and legal disclaimer fields
- Book Me link integration
- All 10 brand themes
- Both layout options (vertical & horizontal)
- Font size control
- Icon toggle
- Priority support

To upgrade to Premium:
1. Sign in with your Google account
2. Click "Upgrade to Premium" in your user menu
3. Follow the instructions to request access
4. We'll activate your premium access within 24 hours!
```

### Day 11: Deploy & Launch

1. Commit all changes to GitHub
2. Push to GitHub Pages
3. Test on live site
4. Announce on social media/Product Hunt
5. Monitor for issues

## Manual Upgrade Process

When users request premium access:

1. Check email for upgrade request
2. Verify payment received (PayPal, Venmo, etc.)
3. Note the user's email from request
4. Run admin script: `node scripts/upgrade-user.js user@example.com premium`
5. Confirm upgrade with user via email
6. User refreshes page and sees premium features

## Next Steps: Adding Stripe (Optional)

Once you've validated demand (e.g., 10+ paying users), implement automated payments:

1. Set up Stripe account
2. Create subscription products
3. Add Stripe Checkout integration
4. Implement webhooks
5. Automate tier updates
6. Migrate manual users to Stripe

Refer to the full `TECHNICAL_ARCHITECTURE.md` for complete Stripe implementation.

## Support

For questions or issues:
- Email: your-email@example.com
- GitHub Issues: https://github.com/mlaplante/EmailSignature/issues

## Summary

This MVP approach gets you to market quickly with:
- ✅ 2-3 week timeline
- ✅ Authentication & user management
- ✅ Feature gating based on tier
- ✅ Professional upgrade experience
- ✅ Validated willingness to pay
- ✅ Low technical complexity
- ✅ Foundation for full automation

Once validated, you can invest in full Stripe integration with confidence!
