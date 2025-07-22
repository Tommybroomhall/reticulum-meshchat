# GHOST GRID: Technical Implementation Guide
## Integrating Cryptocurrency Wallets with Reticulum MeshChat

---

## 🎯 **IMMEDIATE IMPLEMENTATION PLAN**

### Step 1: Wallet Integration with Existing MeshChat

#### Modify Identity Generation
```python
# File: src/backend/ghost_wallet.py
import hashlib
import hmac
from bitcoinlib import HDWallet
from eth_account import Account
import RNS

class GhostGridWallet:
    def __init__(self, reticulum_identity):
        self.reticulum_identity = reticulum_identity
        self.master_seed = self._derive_master_seed()
        self.wallets = self._generate_all_wallets()
    
    def _derive_master_seed(self):
        """Derive master seed from Reticulum identity"""
        identity_bytes = self.reticulum_identity.get_private_key()
        # Use HKDF to derive wallet seed from identity
        salt = b"GHOST_GRID_WALLET_SALT_V1"
        return hmac.new(salt, identity_bytes, hashlib.sha256).digest()
    
    def _generate_all_wallets(self):
        """Generate all cryptocurrency wallets from master seed"""
        wallets = {}
        
        # Bitcoin wallet (BIP44 m/44'/0'/0')
        btc_wallet = HDWallet.from_seed(self.master_seed, network='bitcoin')
        btc_wallet.generate_key("m/44'/0'/0'/0/0")
        wallets['bitcoin'] = {
            'private_key': btc_wallet.private_key_hex(),
            'address': btc_wallet.address(),
            'public_key': btc_wallet.public_key_hex()
        }
        
        # Ethereum wallet
        eth_account = Account.from_key(self.master_seed[:32])
        wallets['ethereum'] = {
            'private_key': eth_account.key.hex(),
            'address': eth_account.address,
            'public_key': eth_account._key_obj.public_key.to_hex()
        }
        
        # GHOST token wallet (same as Ethereum for ERC-20)
        wallets['ghost'] = wallets['ethereum'].copy()
        
        return wallets
    
    def get_wallet_info(self):
        """Return wallet addresses for sharing"""
        return {
            'ghost_id': self.reticulum_identity.hash.hex(),
            'lxmf_address': self.reticulum_identity.hash.hex(), # Simplified
            'bitcoin_address': self.wallets['bitcoin']['address'],
            'ethereum_address': self.wallets['ethereum']['address'],
            'ghost_address': self.wallets['ghost']['address']
        }
```

#### Modify meshchat.py to Include Wallets
```python
# Add to meshchat.py __init__ method
from src.backend.ghost_wallet import GhostGridWallet

class ReticulumMeshChat:
    def __init__(self, identity: RNS.Identity, storage_dir, reticulum_config_dir):
        # ... existing code ...
        
        # Initialize GHOST GRID wallet
        self.ghost_wallet = GhostGridWallet(identity)
        
        # Store wallet info in config
        wallet_info = self.ghost_wallet.get_wallet_info()
        self.config.wallet_addresses = wallet_info
        
        print(f"GHOST GRID Identity Generated:")
        print(f"  Ghost ID: {wallet_info['ghost_id']}")
        print(f"  LXMF Address: {wallet_info['lxmf_address']}")
        print(f"  Bitcoin Address: {wallet_info['bitcoin_address']}")
        print(f"  Ethereum Address: {wallet_info['ethereum_address']}")
```

### Step 2: Payment Message Protocol

#### Extend LXMF Message Format
```python
# File: src/backend/payment_message.py
import json
from decimal import Decimal

class PaymentMessage:
    def __init__(self, amount, currency, recipient_address, message=""):
        self.amount = Decimal(str(amount))
        self.currency = currency.upper()
        self.recipient_address = recipient_address
        self.message = message
        self.timestamp = time.time()
        self.payment_id = self._generate_payment_id()
    
    def _generate_payment_id(self):
        """Generate unique payment ID"""
        data = f"{self.amount}{self.currency}{self.recipient_address}{self.timestamp}"
        return hashlib.sha256(data.encode()).hexdigest()[:16]
    
    def to_lxmf_content(self):
        """Convert to LXMF message content"""
        payment_data = {
            'type': 'payment',
            'payment_id': self.payment_id,
            'amount': str(self.amount),
            'currency': self.currency,
            'recipient_address': self.recipient_address,
            'message': self.message,
            'timestamp': self.timestamp
        }
        return json.dumps(payment_data)
    
    @classmethod
    def from_lxmf_content(cls, content):
        """Parse payment from LXMF message"""
        try:
            data = json.loads(content)
            if data.get('type') == 'payment':
                return cls(
                    amount=data['amount'],
                    currency=data['currency'],
                    recipient_address=data['recipient_address'],
                    message=data.get('message', '')
                )
        except:
            pass
        return None

# Add to meshchat.py
async def send_payment_message(self, destination_hash, amount, currency, message=""):
    """Send a payment with message"""
    # Get recipient's wallet address
    recipient_wallet = self._get_recipient_wallet(destination_hash, currency)
    
    # Create payment message
    payment_msg = PaymentMessage(amount, currency, recipient_wallet, message)
    
    # Send as LXMF message
    await self.send_lxmf_message(
        destination_hash=destination_hash,
        content=payment_msg.to_lxmf_content(),
        title=f"Payment: {amount} {currency}",
        fields=None
    )
    
    # Execute actual cryptocurrency transaction
    tx_hash = await self._execute_crypto_transaction(payment_msg)
    
    # Store payment record
    self._store_payment_record(payment_msg, tx_hash)
```

### Step 3: Web Interface Updates

#### Add Wallet Display to Frontend
```javascript
// File: src/frontend/components/wallet/WalletPage.vue
<template>
  <div class="wallet-page">
    <div class="wallet-header">
      <h2>GHOST GRID Wallet</h2>
      <div class="identity-info">
        <p><strong>Ghost ID:</strong> {{ walletInfo.ghost_id }}</p>
        <p><strong>LXMF:</strong> {{ walletInfo.lxmf_address }}</p>
      </div>
    </div>
    
    <div class="wallet-balances">
      <div class="balance-card" v-for="wallet in wallets" :key="wallet.currency">
        <h3>{{ wallet.currency }}</h3>
        <p class="balance">{{ wallet.balance }}</p>
        <p class="address">{{ wallet.address }}</p>
        <div class="actions">
          <button @click="sendPayment(wallet.currency)">Send</button>
          <button @click="receivePayment(wallet.currency)">Receive</button>
        </div>
      </div>
    </div>
    
    <div class="payment-history">
      <h3>Recent Transactions</h3>
      <div v-for="payment in recentPayments" :key="payment.id" class="payment-item">
        <span class="amount">{{ payment.amount }} {{ payment.currency }}</span>
        <span class="recipient">{{ payment.recipient }}</span>
        <span class="message">{{ payment.message }}</span>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'WalletPage',
  data() {
    return {
      walletInfo: {},
      wallets: [],
      recentPayments: []
    }
  },
  async mounted() {
    await this.loadWalletData();
  },
  methods: {
    async loadWalletData() {
      const response = await axios.get('/api/wallet/info');
      this.walletInfo = response.data.wallet_info;
      this.wallets = response.data.wallets;
      this.recentPayments = response.data.recent_payments;
    },
    
    sendPayment(currency) {
      this.$router.push(`/wallet/send/${currency}`);
    },
    
    receivePayment(currency) {
      this.$router.push(`/wallet/receive/${currency}`);
    }
  }
}
</script>
```

#### Add Payment Button to Messages
```javascript
// File: src/frontend/components/messages/PaymentButton.vue
<template>
  <div class="payment-button">
    <button @click="showPaymentModal = true" class="pay-btn">
      💰 Send Payment
    </button>
    
    <div v-if="showPaymentModal" class="payment-modal">
      <div class="modal-content">
        <h3>Send Payment</h3>
        <form @submit.prevent="sendPayment">
          <div class="form-group">
            <label>Amount:</label>
            <input v-model="paymentAmount" type="number" step="0.00000001" required>
          </div>
          <div class="form-group">
            <label>Currency:</label>
            <select v-model="paymentCurrency">
              <option value="BTC">Bitcoin</option>
              <option value="ETH">Ethereum</option>
              <option value="GHOST">GHOST Token</option>
            </select>
          </div>
          <div class="form-group">
            <label>Message:</label>
            <input v-model="paymentMessage" type="text" placeholder="Coffee money! ☕">
          </div>
          <div class="form-actions">
            <button type="submit">Send Payment</button>
            <button type="button" @click="showPaymentModal = false">Cancel</button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'PaymentButton',
  props: ['destinationHash'],
  data() {
    return {
      showPaymentModal: false,
      paymentAmount: '',
      paymentCurrency: 'BTC',
      paymentMessage: ''
    }
  },
  methods: {
    async sendPayment() {
      try {
        await axios.post('/api/payments/send', {
          destination_hash: this.destinationHash,
          amount: this.paymentAmount,
          currency: this.paymentCurrency,
          message: this.paymentMessage
        });
        
        this.showPaymentModal = false;
        this.$emit('payment-sent');
        
        // Show success notification
        this.$toast.success(`Payment of ${this.paymentAmount} ${this.paymentCurrency} sent!`);
        
      } catch (error) {
        this.$toast.error('Payment failed: ' + error.message);
      }
    }
  }
}
</script>
```

### Step 4: API Endpoints

#### Add Wallet API Routes
```python
# Add to meshchat.py routes
routes = [
    # ... existing routes ...
    
    # Wallet endpoints
    web.get('/api/wallet/info', self.api_wallet_info),
    web.post('/api/wallet/send', self.api_wallet_send),
    web.get('/api/wallet/balance/{currency}', self.api_wallet_balance),
    web.get('/api/wallet/transactions', self.api_wallet_transactions),
    
    # Payment endpoints
    web.post('/api/payments/send', self.api_send_payment),
    web.get('/api/payments/history', self.api_payment_history),
]

async def api_wallet_info(self, request):
    """Get wallet addresses and basic info"""
    wallet_info = self.ghost_wallet.get_wallet_info()
    
    # Get current balances (would need blockchain API integration)
    wallets = []
    for currency in ['bitcoin', 'ethereum', 'ghost']:
        balance = await self._get_wallet_balance(currency)
        wallets.append({
            'currency': currency.upper(),
            'address': wallet_info[f'{currency}_address'],
            'balance': str(balance)
        })
    
    return web.json_response({
        'wallet_info': wallet_info,
        'wallets': wallets,
        'recent_payments': self._get_recent_payments()
    })

async def api_send_payment(self, request):
    """Send payment with message"""
    data = await request.json()
    
    try:
        await self.send_payment_message(
            destination_hash=data['destination_hash'],
            amount=data['amount'],
            currency=data['currency'],
            message=data.get('message', '')
        )
        
        return web.json_response({'success': True})
        
    except Exception as e:
        return web.json_response({
            'success': False,
            'error': str(e)
        }, status=400)
```

### Step 5: Database Schema Updates

#### Add Payment Tables
```python
# Add to database.py
class Payment(BaseModel):
    id = BigAutoField()
    payment_id = CharField(unique=True)  # Unique payment identifier
    from_ghost_id = CharField(index=True)  # Sender's Ghost ID
    to_ghost_id = CharField(index=True)  # Recipient's Ghost ID
    amount = DecimalField(max_digits=20, decimal_places=8)  # Payment amount
    currency = CharField()  # BTC, ETH, GHOST, etc.
    message = TextField(null=True)  # Optional message
    transaction_hash = CharField(null=True)  # Blockchain transaction hash
    status = CharField()  # pending, confirmed, failed
    created_at = DateTimeField(default=lambda: datetime.now(timezone.utc))
    confirmed_at = DateTimeField(null=True)

class WalletBalance(BaseModel):
    id = BigAutoField()
    ghost_id = CharField(index=True)  # Owner's Ghost ID
    currency = CharField()  # Currency type
    balance = DecimalField(max_digits=20, decimal_places=8)  # Current balance
    last_updated = DateTimeField(default=lambda: datetime.now(timezone.utc))
    
    class Meta:
        indexes = (
            (('ghost_id', 'currency'), True),  # Unique constraint
        )
```

---

## 🚀 **NEXT DEVELOPMENT PHASES**

### Phase 2: Lightning Network Integration
- Integrate LND (Lightning Network Daemon)
- Create Lightning invoice generation
- Implement payment channel management
- Add Lightning payments over LoRa

### Phase 3: GHOST Token Smart Contract
- Deploy ERC-20 token contract
- Implement staking mechanisms
- Create governance voting system
- Add token distribution logic

### Phase 4: Mobile App Development
- React Native app with wallet features
- Biometric authentication
- QR code scanning for payments
- Push notifications for payments

### Phase 5: Hardware Integration
- ESP32 firmware with crypto wallets
- Fingerprint scanner integration
- Secure element for key storage
- Solar charging optimization

---

## 📋 **DEVELOPMENT CHECKLIST**

### Immediate (Week 1-2)
- [ ] Create GhostGridWallet class
- [ ] Integrate wallet generation with Reticulum identity
- [ ] Add wallet info display to web interface
- [ ] Test wallet address generation

### Short Term (Week 3-4)
- [ ] Implement PaymentMessage protocol
- [ ] Add payment sending API endpoints
- [ ] Create payment UI components
- [ ] Test payment messaging over mesh

### Medium Term (Month 2-3)
- [ ] Integrate with blockchain APIs for balance checking
- [ ] Implement actual cryptocurrency transactions
- [ ] Add payment history and tracking
- [ ] Create mobile app prototype

### Long Term (Month 4-6)
- [ ] Lightning Network integration
- [ ] GHOST token smart contract deployment
- [ ] Hardware prototype development
- [ ] Beta testing with real users

---

*This implementation guide provides the technical roadmap for integrating cryptocurrency wallets with the existing Reticulum MeshChat codebase, creating the foundation for the GHOST GRID ecosystem.*
