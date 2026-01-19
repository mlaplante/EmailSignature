# Technical Architecture for Subscription Feature

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         User's Browser                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │             Email Signature Generator (SPA)               │ │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐  │ │
│  │  │   UI Layer  │  │ Auth Manager │  │ Feature Gating  │  │ │
│  │  │  (HTML/CSS) │  │ (Firebase)   │  │   Controller    │  │ │
│  │  └─────────────┘  └──────────────┘  └─────────────────┘  │ │
│  │         │                 │                    │          │ │
│  │         └─────────────────┴────────────────────┘          │ │
│  │                           │                                │ │
│  │                  ┌────────▼─────────┐                     │ │
│  │                  │  LocalStorage    │                     │ │
│  │                  │  - Signature Data│                     │ │
│  │                  │  - User Prefs    │                     │ │
│  │                  └──────────────────┘                     │ │
│  └───────────────────────────────────────────────────────────┘ │
│                           │                                     │
│                           │ Firebase SDK / REST API             │
└───────────────────────────┼─────────────────────────────────────┘
                            │
        ┌───────────────────┴────────────────────┐
        │                                        │
        ▼                                        ▼
┌───────────────────┐                  ┌──────────────────┐
│  Firebase Services│                  │ Vercel Functions │
│  ┌──────────────┐ │                  │ ┌──────────────┐ │
│  │ Authentication│ │                  │ │  API Routes  │ │
│  │   - Google   │ │                  │ │              │ │
│  │   - Email/PW │ │                  │ │ /api/verify  │ │
│  └──────────────┘ │                  │ │ /api/webhook │ │
│  ┌──────────────┐ │                  │ │ /api/create  │ │
│  │  Firestore   │ │◄─────────────────┤ └──────────────┘ │
│  │  Database    │ │   Read/Write     └──────────────────┘
│  │              │ │                           │
│  │ - users/     │ │                           │
│  │ - subs/      │ │                           │
│  └──────────────┘ │                           │
└───────────────────┘                           │
                                                 │
                                                 │ Webhooks
                                                 │
                                        ┌────────▼─────────┐
                                        │  Stripe Services │
                                        │ ┌──────────────┐ │
                                        │ │   Checkout   │ │
                                        │ │   Sessions   │ │
                                        │ └──────────────┘ │
                                        │ ┌──────────────┐ │
                                        │ │ Subscriptions│ │
                                        │ │  Management  │ │
                                        │ └──────────────┘ │
                                        │ ┌──────────────┐ │
                                        │ │   Customer   │ │
                                        │ │    Portal    │ │
                                        │ └──────────────┘ │
                                        └──────────────────┘
```

## Component Details

### 1. Frontend Application (index.html)

#### Current State
- Single HTML file with inline CSS and JavaScript
- No build process
- No dependencies
- Hosted on GitHub Pages

#### Required Changes

**A. Add Firebase SDK**
```html
<!-- Add before closing </body> tag -->
<script src="https://www.gstatic.com/firebasejs/9.x.x/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.x.x/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.x.x/firebase-firestore-compat.js"></script>
```

**B. Update Content Security Policy**
```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'self'; 
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; 
  font-src 'self' https://fonts.gstatic.com; 
  script-src 'self' 'unsafe-inline' https://www.gstatic.com https://apis.google.com; 
  img-src 'self' data: https:; 
  connect-src 'self' https://*.googleapis.com https://*.firebaseio.com https://your-vercel-app.vercel.app;
  frame-src https://accounts.google.com https://checkout.stripe.com;
">
```

**C. New JavaScript Modules to Add**

**auth-manager.js** (inline in HTML)
```javascript
class AuthManager {
  constructor() {
    this.user = null;
    this.subscriptionTier = 'free';
    this.initFirebase();
  }

  initFirebase() {
    // Firebase configuration
    const firebaseConfig = {
      apiKey: "YOUR_API_KEY",
      authDomain: "YOUR_AUTH_DOMAIN",
      projectId: "YOUR_PROJECT_ID",
      // ... other config
    };
    
    firebase.initializeApp(firebaseConfig);
    this.auth = firebase.auth();
    this.db = firebase.firestore();
    
    // Listen for auth state changes
    this.auth.onAuthStateChanged((user) => {
      this.handleAuthStateChange(user);
    });
  }

  async handleAuthStateChange(user) {
    this.user = user;
    if (user) {
      await this.loadSubscriptionStatus(user.uid);
      this.updateUI();
    } else {
      this.subscriptionTier = 'free';
      this.updateUI();
    }
  }

  async loadSubscriptionStatus(uid) {
    const userDoc = await this.db.collection('users').doc(uid).get();
    if (userDoc.exists) {
      const data = userDoc.data();
      this.subscriptionTier = data.subscriptionTier || 'free';
    }
  }

  async signInWithGoogle() {
    const provider = new firebase.auth.GoogleAuthProvider();
    try {
      await this.auth.signInWithPopup(provider);
    } catch (error) {
      console.error('Sign in error:', error);
      showToast('Failed to sign in. Please try again.', 'error');
    }
  }

  async signOut() {
    try {
      await this.auth.signOut();
      showToast('Signed out successfully', 'success');
    } catch (error) {
      console.error('Sign out error:', error);
    }
  }

  isAuthenticated() {
    return this.user !== null;
  }

  hasFeature(featureName) {
    const featureMap = {
      free: ['name', 'title', 'email', 'phone', 'basicThemes'],
      premium: ['name', 'title', 'email', 'phone', 'basicThemes', 
                'company', 'logo', 'address', 'bookMe', 'disclaimer', 
                'allThemes', 'layouts', 'fontSize', 'icons', 'previews'],
      enterprise: ['all']
    };
    
    if (this.subscriptionTier === 'enterprise') return true;
    return featureMap[this.subscriptionTier]?.includes(featureName) || false;
  }

  updateUI() {
    // Update header with user info
    // Enable/disable features based on tier
    featureGatingController.applyFeatureGating(this.subscriptionTier);
  }
}

const authManager = new AuthManager();
```

**feature-gating-controller.js** (inline in HTML)
```javascript
class FeatureGatingController {
  constructor() {
    this.premiumFields = [
      { id: 'company', name: 'Company Name' },
      { id: 'address', name: 'Office Address' },
      { id: 'bookMe', name: 'Book Me Link' },
      { id: 'disclaimer', name: 'Legal Disclaimer' },
      { id: 'logo', name: 'Custom Logo', type: 'file' }
    ];
    
    this.premiumThemes = [
      'creative-purple', 'eco-green', 'energetic-orange',
      'modern-teal', 'elegant-burgundy', 'tech-cyan', 'minimalist-gray'
    ];
  }

  applyFeatureGating(tier) {
    if (tier === 'free') {
      this.lockPremiumFields();
      this.lockPremiumThemes();
      this.lockLayoutOptions();
      this.lockAdvancedControls();
    } else {
      this.unlockAllFeatures();
    }
  }

  lockPremiumFields() {
    this.premiumFields.forEach(field => {
      const element = document.getElementById(field.id);
      const inputGroup = element.closest('.input-group');
      const label = inputGroup.querySelector('label');
      
      // Disable input
      element.disabled = true;
      element.placeholder = `Premium Feature - Upgrade to Unlock`;
      
      // Add premium badge if not exists
      if (!label.querySelector('.premium-badge')) {
        const badge = this.createPremiumBadge();
        label.appendChild(badge);
      }
      
      // Add click handler to show upgrade modal
      inputGroup.style.cursor = 'pointer';
      inputGroup.style.position = 'relative';
      inputGroup.onclick = (e) => {
        e.preventDefault();
        this.showUpgradeModal(field.name);
      };
      
      // Add overlay
      if (!inputGroup.querySelector('.premium-overlay')) {
        const overlay = document.createElement('div');
        overlay.className = 'premium-overlay';
        overlay.innerHTML = `
          <div class="premium-overlay-content">
            <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <rect x="3" y="11" width="18" height="11" rx="2" ry="2"></rect>
              <path d="M7 11V7a5 5 0 0 1 10 0v4"></path>
            </svg>
            <span>Premium Feature</span>
          </div>
        `;
        inputGroup.appendChild(overlay);
      }
    });
  }

  lockPremiumThemes() {
    const brandSelect = document.getElementById('brand');
    const options = brandSelect.querySelectorAll('option');
    
    options.forEach(option => {
      if (this.premiumThemes.includes(option.value)) {
        option.disabled = true;
        option.textContent = option.textContent + ' 🔒 Premium';
      }
    });
  }

  lockLayoutOptions() {
    const horizontalRadio = document.getElementById('layoutHorizontal');
    const horizontalLabel = document.querySelector('label[for="layoutHorizontal"]');
    
    horizontalRadio.disabled = true;
    horizontalLabel.style.opacity = '0.5';
    horizontalLabel.style.cursor = 'not-allowed';
    
    if (!horizontalLabel.querySelector('.premium-badge')) {
      const badge = this.createPremiumBadge();
      horizontalLabel.appendChild(badge);
    }
    
    horizontalLabel.onclick = (e) => {
      e.preventDefault();
      this.showUpgradeModal('Horizontal Layout');
    };
  }

  lockAdvancedControls() {
    // Lock font size control
    const fontSizeInput = document.getElementById('fontSize');
    fontSizeInput.disabled = true;
    fontSizeInput.value = 14; // Reset to default
    
    // Lock icon toggle
    const showIconsCheckbox = document.getElementById('showIcons');
    showIconsCheckbox.disabled = true;
    showIconsCheckbox.checked = true; // Reset to default
  }

  unlockAllFeatures() {
    // Remove all premium overlays and badges
    document.querySelectorAll('.premium-overlay').forEach(el => el.remove());
    document.querySelectorAll('.premium-badge').forEach(el => el.remove());
    
    // Enable all inputs
    this.premiumFields.forEach(field => {
      const element = document.getElementById(field.id);
      element.disabled = false;
      if (field.type !== 'file') {
        element.placeholder = element.getAttribute('data-original-placeholder');
      }
      
      const inputGroup = element.closest('.input-group');
      inputGroup.style.cursor = 'default';
      inputGroup.onclick = null;
    });
    
    // Enable all themes
    const brandSelect = document.getElementById('brand');
    const options = brandSelect.querySelectorAll('option');
    options.forEach(option => {
      option.disabled = false;
      option.textContent = option.textContent.replace(' 🔒 Premium', '');
    });
    
    // Enable layout options
    const horizontalRadio = document.getElementById('layoutHorizontal');
    const horizontalLabel = document.querySelector('label[for="layoutHorizontal"]');
    horizontalRadio.disabled = false;
    horizontalLabel.style.opacity = '1';
    horizontalLabel.style.cursor = 'pointer';
    horizontalLabel.onclick = null;
    
    // Enable advanced controls
    document.getElementById('fontSize').disabled = false;
    document.getElementById('showIcons').disabled = false;
  }

  createPremiumBadge() {
    const badge = document.createElement('span');
    badge.className = 'premium-badge';
    badge.innerHTML = '⭐ Premium';
    return badge;
  }

  showUpgradeModal(featureName) {
    const modal = document.getElementById('upgradeModal');
    if (!modal) {
      this.createUpgradeModal(featureName);
    } else {
      this.updateUpgradeModalContent(featureName);
      modal.style.display = 'flex';
    }
  }

  createUpgradeModal(featureName) {
    const modal = document.createElement('div');
    modal.id = 'upgradeModal';
    modal.className = 'modal';
    modal.innerHTML = `
      <div class="modal-content">
        <button class="modal-close" onclick="document.getElementById('upgradeModal').style.display='none'">&times;</button>
        <div class="modal-header">
          <h2>🌟 Unlock Premium Features</h2>
          <p class="modal-subtitle">Upgrade to access <strong id="featureName">${featureName}</strong> and more!</p>
        </div>
        <div class="pricing-tiers">
          <div class="pricing-tier">
            <h3>Premium</h3>
            <div class="price">$9.99<span>/month</span></div>
            <div class="price-annual">or $99/year (save 17%)</div>
            <ul class="features-list">
              <li>✓ Custom Logo Upload</li>
              <li>✓ All 10 Brand Themes</li>
              <li>✓ Both Layout Options</li>
              <li>✓ Font Size Control</li>
              <li>✓ Advanced Fields (Company, Address, etc.)</li>
              <li>✓ Save up to 5 Signatures</li>
              <li>✓ Priority Support</li>
            </ul>
            <button class="btn-upgrade" onclick="handleUpgrade('premium')">Upgrade to Premium</button>
          </div>
          <div class="pricing-tier featured">
            <div class="popular-badge">Most Popular</div>
            <h3>Premium Annual</h3>
            <div class="price">$99<span>/year</span></div>
            <div class="price-annual">Just $8.25/month</div>
            <ul class="features-list">
              <li>✓ Everything in Premium</li>
              <li>✓ Save 17% compared to monthly</li>
              <li>✓ 2 Months Free</li>
            </ul>
            <button class="btn-upgrade primary" onclick="handleUpgrade('premium-annual')">Best Value - Upgrade Now</button>
          </div>
        </div>
        <div class="modal-footer">
          <p>💳 Secure payment powered by Stripe</p>
          <p>🔄 Cancel anytime, no questions asked</p>
        </div>
      </div>
    `;
    document.body.appendChild(modal);
    modal.style.display = 'flex';
  }

  updateUpgradeModalContent(featureName) {
    const featureNameEl = document.getElementById('featureName');
    if (featureNameEl) {
      featureNameEl.textContent = featureName;
    }
  }
}

const featureGatingController = new FeatureGatingController();
```

### 2. Backend Services (Vercel Functions)

#### API Endpoints Required

**A. /api/verify-subscription**
- Verify user's subscription status
- Called on page load and auth state change
- Returns subscription tier and features

**B. /api/create-checkout-session**
- Create Stripe Checkout session
- Called when user clicks "Upgrade"
- Returns checkout URL

**C. /api/webhook**
- Handle Stripe webhooks
- Update Firestore on subscription events
- Events: subscription.created, subscription.updated, subscription.deleted

**D. /api/create-portal-session**
- Create Stripe Customer Portal session
- Called when user clicks "Manage Subscription"
- Returns portal URL

### 3. Database Schema (Firestore)

#### Collection: users
```
users/{userId}
  - email: string
  - displayName: string
  - photoURL: string
  - subscriptionTier: string ('free', 'premium', 'enterprise')
  - stripeCustomerId: string
  - stripeSubscriptionId: string
  - subscriptionStatus: string ('active', 'canceled', 'past_due')
  - subscriptionStartDate: timestamp
  - subscriptionEndDate: timestamp
  - createdAt: timestamp
  - updatedAt: timestamp
```

#### Collection: signatures
```
signatures/{signatureId}
  - userId: string
  - name: string
  - data: object (signature fields)
  - createdAt: timestamp
  - updatedAt: timestamp
  - isDefault: boolean
```

#### Collection: usage_logs (optional, for analytics)
```
usage_logs/{logId}
  - userId: string
  - action: string ('copy', 'export', 'save')
  - timestamp: timestamp
  - tier: string
```

### 4. Stripe Configuration

#### Products to Create

**Product 1: Email Signature Premium**
- Monthly Price: $9.99 USD
- Annual Price: $99 USD
- Recurring: Yes
- Features: All premium features

**Product 2: Email Signature Enterprise**
- Monthly Price: $49.99 USD
- Annual Price: $499 USD
- Recurring: Yes
- Features: All enterprise features

#### Webhooks to Configure
- `customer.subscription.created`
- `customer.subscription.updated`
- `customer.subscription.deleted`
- `invoice.payment_succeeded`
- `invoice.payment_failed`

### 5. Security Implementation

#### Frontend Validation (UX Only)
```javascript
// Check features on UI interaction
function validateFeatureAccess(featureName) {
  if (!authManager.hasFeature(featureName)) {
    featureGatingController.showUpgradeModal(featureName);
    return false;
  }
  return true;
}

// Example usage
document.getElementById('company').addEventListener('focus', (e) => {
  if (!validateFeatureAccess('company')) {
    e.target.blur();
  }
});
```

#### Backend Verification (Security)
```javascript
// Verify subscription on server before allowing premium operations
// Example: Export HTML with custom logo

// Client-side
async function exportHTML() {
  if (hasCustomLogo) {
    // Verify server-side
    const response = await fetch('/api/verify-subscription', {
      headers: {
        'Authorization': `Bearer ${await authManager.user.getIdToken()}`
      }
    });
    const { tier } = await response.json();
    if (tier === 'free') {
      showToast('Custom logo export requires Premium subscription', 'error');
      return;
    }
  }
  
  // Proceed with export
  // ...
}
```

### 6. CSS Additions for Subscription UI

```css
/* Premium Badge */
.premium-badge {
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
  padding: 0.125rem 0.5rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-radius: 12px;
  font-size: 0.75rem;
  font-weight: 500;
  margin-left: 0.5rem;
  vertical-align: middle;
}

/* Premium Overlay on Locked Fields */
.premium-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.05);
  backdrop-filter: blur(2px);
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 10px;
  pointer-events: none;
  z-index: 1;
}

.premium-overlay-content {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  font-size: 0.875rem;
  font-weight: 500;
  color: #667eea;
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
  z-index: 9999;
  padding: 1rem;
}

.modal-content {
  background: var(--bg-secondary);
  border-radius: 16px;
  max-width: 900px;
  width: 100%;
  max-height: 90vh;
  overflow-y: auto;
  padding: 2rem;
  position: relative;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
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
  font-size: 2rem;
  margin-bottom: 0.5rem;
}

.modal-subtitle {
  color: var(--text-muted);
  font-size: 1.1rem;
}

.pricing-tiers {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.5rem;
  margin-bottom: 2rem;
}

.pricing-tier {
  background: var(--bg-primary);
  border: 2px solid var(--border);
  border-radius: 12px;
  padding: 2rem;
  position: relative;
}

.pricing-tier.featured {
  border-color: #667eea;
  box-shadow: 0 8px 24px rgba(102, 126, 234, 0.2);
}

.popular-badge {
  position: absolute;
  top: -12px;
  left: 50%;
  transform: translateX(-50%);
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 0.25rem 1rem;
  border-radius: 12px;
  font-size: 0.75rem;
  font-weight: 600;
}

.pricing-tier h3 {
  font-size: 1.5rem;
  margin-bottom: 1rem;
}

.price {
  font-size: 2.5rem;
  font-weight: 700;
  color: var(--accent);
  margin-bottom: 0.25rem;
}

.price span {
  font-size: 1rem;
  color: var(--text-muted);
  font-weight: 400;
}

.price-annual {
  color: var(--text-muted);
  font-size: 0.875rem;
  margin-bottom: 1.5rem;
}

.features-list {
  list-style: none;
  margin-bottom: 1.5rem;
}

.features-list li {
  padding: 0.5rem 0;
  color: var(--text-primary);
  font-size: 0.95rem;
}

.btn-upgrade {
  width: 100%;
  padding: 1rem;
  border: 2px solid var(--accent);
  background: transparent;
  color: var(--accent);
  border-radius: 10px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
}

.btn-upgrade:hover {
  background: var(--accent);
  color: white;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(217, 119, 87, 0.3);
}

.btn-upgrade.primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-color: transparent;
  color: white;
}

.btn-upgrade.primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 24px rgba(102, 126, 234, 0.4);
}

.modal-footer {
  text-align: center;
  padding-top: 1.5rem;
  border-top: 1px solid var(--border);
}

.modal-footer p {
  color: var(--text-muted);
  font-size: 0.875rem;
  margin: 0.5rem 0;
}

/* User Menu */
.user-menu {
  position: fixed;
  top: 2rem;
  right: 5rem;
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.user-avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  cursor: pointer;
  border: 2px solid var(--border);
  transition: all 0.2s ease;
}

.user-avatar:hover {
  border-color: var(--accent);
  transform: scale(1.05);
}

.user-dropdown {
  position: absolute;
  top: 100%;
  right: 0;
  margin-top: 0.5rem;
  background: var(--bg-secondary);
  border: 1px solid var(--border);
  border-radius: 12px;
  box-shadow: 0 8px 24px var(--shadow);
  min-width: 200px;
  display: none;
  z-index: 1000;
}

.user-dropdown.show {
  display: block;
}

.user-dropdown-item {
  padding: 0.875rem 1rem;
  cursor: pointer;
  transition: background 0.2s ease;
  border-bottom: 1px solid var(--border);
}

.user-dropdown-item:last-child {
  border-bottom: none;
}

.user-dropdown-item:hover {
  background: var(--bg-primary);
}

.tier-badge {
  display: inline-block;
  padding: 0.125rem 0.5rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-radius: 8px;
  font-size: 0.7rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-left: 0.5rem;
}

.tier-badge.free {
  background: var(--text-muted);
}
```

### 7. Migration Strategy

#### Phase 1: Add UI Components (No Backend)
1. Add premium badges to UI
2. Add upgrade modal (non-functional)
3. Add user menu placeholder
4. Test visual appearance

#### Phase 2: Add Firebase Auth
1. Initialize Firebase
2. Add sign-in/sign-out functionality
3. Test authentication flow
4. Store auth state

#### Phase 3: Add Feature Gating
1. Implement FeatureGatingController
2. Lock fields based on hardcoded tier
3. Test locked/unlocked states
4. Validate UX

#### Phase 4: Add Stripe Integration
1. Set up Stripe account and products
2. Create Vercel functions
3. Implement checkout flow
4. Test end-to-end payment

#### Phase 5: Connect All Parts
1. Wire Firebase to Stripe via webhooks
2. Sync subscription status
3. Enable dynamic feature gating
4. Test full flow

### 8. Testing Checklist

#### Manual Testing
- [ ] Free user can see locked features
- [ ] Free user sees upgrade modal on click
- [ ] Sign in with Google works
- [ ] Sign out works
- [ ] Premium user sees all features unlocked
- [ ] Payment flow completes successfully
- [ ] Subscription status syncs after payment
- [ ] Features unlock immediately after upgrade
- [ ] Customer portal opens correctly
- [ ] Subscription cancellation works
- [ ] Downgrade from premium to free works
- [ ] Webhooks update database correctly

#### Edge Cases
- [ ] Expired subscription locks features
- [ ] Failed payment shows appropriate message
- [ ] User with no subscription defaults to free
- [ ] LocalStorage data persists across auth states
- [ ] Multiple browser tabs sync auth state
- [ ] Offline mode handles gracefully

### 9. Deployment Checklist

#### Pre-deployment
- [ ] Update CSP headers
- [ ] Add environment variables to Vercel
- [ ] Configure Firebase production project
- [ ] Set up Stripe production environment
- [ ] Test webhook endpoint is accessible
- [ ] Add Terms of Service
- [ ] Add Privacy Policy
- [ ] Add Refund Policy

#### Deployment
- [ ] Deploy Vercel functions
- [ ] Deploy updated frontend
- [ ] Verify webhooks are receiving events
- [ ] Test payment flow in production
- [ ] Monitor error logs

#### Post-deployment
- [ ] Announce feature to users
- [ ] Monitor conversion rates
- [ ] Collect user feedback
- [ ] Fix bugs and iterate

## Conclusion

This architecture provides a scalable, secure foundation for adding subscription features to the Email Signature Generator. The phased approach allows for incremental development and testing, reducing risk and allowing for early feedback.

**Key Benefits**:
- Minimal changes to existing code
- Leverages proven third-party services
- Scalable to thousands of users
- Secure subscription verification
- Clean separation of concerns

**Next Steps**:
1. Review and approve architecture
2. Set up Firebase and Stripe accounts
3. Begin Phase 1 implementation
4. Iterate based on user feedback
