# Running a Gnosis Chain Validator Node - Manual Setup

Complete guide for manually setting up a Gnosis Chain validator using Docker.

## 📋 Requirements

### Hardware
- **CPU:** 4+ cores (recommended: 8 cores)
- **RAM:** Minimum 8GB (recommended: 16GB)
- **Storage:** 500GB SSD minimum (currently ~200GB used, growing)
- **Network:** Stable wired internet connection (~700 MB/hour upload)
- **OS:** Ubuntu 20.04 LTS or similar Linux distribution

### What You Need
- 1 GNO token (for staking deposit)
- 0.1 xDAI (for transaction fees)
- MetaMask or compatible Web3 wallet
- Docker & Docker Compose installed

### Syncing Time
- **Execution Layer:** 1-3 days depending on hardware
- **Consensus Layer:** Several hours
- **Validator Activation:** ~1.5 hours after deposit

## 🚀 Setup Instructions

### Step 1: Install Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo apt install docker-compose -y

# Add user to docker group
sudo usermod -aG docker $USER

# Log out and back in for group changes to take effect
```

### Step 2: Create Directory Structure

```bash
# Create main directory
mkdir -p ~/gnosis/{execution,consensus,keystores,validators,jwtsecret}
cd ~/gnosis
```

### Step 3: Generate JWT Secret

```bash
# Create JWT secret for client authentication
openssl rand -hex 32 | tr -d "\n" > ./jwtsecret/jwt.hex
```

### Step 4: Create Docker Compose Configuration

Create `docker-compose.yml` in `~/gnosis/`:

```yaml
version: '3.8'

services:
  execution:
    image: nethermind/nethermind:latest
    container_name: gnosis-execution
    restart: unless-stopped
    ports:
      - "30303:30303/tcp"
      - "30303:30303/udp"
      - "8545:8545"
      - "8551:8551"
    volumes:
      - ./execution:/data
      - ./jwtsecret:/jwtsecret
    command: |
      --config gnosis
      --datadir /data
      --JsonRpc.JwtSecretFile=/jwtsecret/jwt.hex
      --JsonRpc.Enabled=true
      --JsonRpc.Host=0.0.0.0
      --JsonRpc.Port=8545
    networks:
      - gnosis_net

  consensus:
    image: sigp/lighthouse:latest-modern
    container_name: gnosis-consensus
    restart: unless-stopped
    depends_on:
      - execution
    ports:
      - "9000:9000/tcp"
      - "9000:9000/udp"
      - "4000:4000"
    volumes:
      - ./consensus:/data
      - ./jwtsecret:/jwtsecret
    command: |
      lighthouse bn
      --network=gnosis
      --datadir=/data
      --execution-endpoint=http://execution:8551
      --execution-jwt=/jwtsecret/jwt.hex
      --http
      --http-address=0.0.0.0
      --http-port=4000
      --metrics
      --metrics-address=0.0.0.0
      --checkpoint-sync-url=https://checkpoint.gnosischain.com
    networks:
      - gnosis_net

  validator:
    image: sigp/lighthouse:latest-modern
    container_name: gnosis-validator
    restart: unless-stopped
    depends_on:
      - consensus
    env_file:
      - .env
    volumes:
      - ./validators:/data/validators
    command: |
      lighthouse validator_client
      --network=gnosis
      --validators-dir=/data/validators
      --beacon-nodes=http://consensus:4000
      --graffiti=${GRAFFITI}
      --suggested-fee-recipient=${FEE_RECIPIENT}
      --metrics
      --metrics-address=0.0.0.0
      --metrics-port=5064
    networks:
      - gnosis_net

networks:
  gnosis_net:
    name: gnosis_net
```

### Step 5: Create Environment File

Create `.env` file in `~/gnosis/`:

```bash
# Your Gnosis Chain address to receive transaction fees
FEE_RECIPIENT=0xYourGnosisAddressHere

# Optional: Custom message in blocks you propose
GRAFFITI="My Gnosis Validator"
```

Replace `0xYourGnosisAddressHere` with your actual Gnosis Chain address.

### Step 6: Start Execution and Consensus Clients

```bash
# Start execution and consensus layers
docker-compose up -d execution consensus

# Check logs to monitor syncing
docker-compose logs -f execution
```

**Important:** Wait for both clients to fully sync before proceeding. This can take 1-3 days.

Check sync status:
```bash
# Execution client sync status
docker-compose logs execution | grep -i "sync"

# Consensus client sync status
docker-compose logs consensus | grep -i "sync"
```

### Step 7: Generate Validator Keys

**While clients are syncing**, generate your validator keys:

```bash
cd ~/gnosis

docker run -it --rm \
  -v $(pwd)/keystores:/app/validator_keys \
  ghcr.io/gnosischain/validator-data-generator:latest new-mnemonic \
  --num_validators=1 \
  --mnemonic_language=english \
  --chain=gnosis \
  --folder=/app/validator_keys \
  --eth1_withdrawal_address=0xYourGnosisAddressHere
```

Replace `0xYourGnosisAddressHere` with your Gnosis address where you want withdrawals sent.

**⚠️ CRITICAL SECURITY:**
- You'll be shown a 24-word mnemonic seed phrase
- **Write this down on paper and store it securely offline**
- This phrase is the ONLY way to recover your validator
- Never share it with anyone
- Never store it digitally

You'll also need to create a password for your validator keystore. Remember this password!

### Step 8: Import Validator Keys

```bash
# Import the generated keys into Lighthouse
docker run --rm \
  -v $(pwd)/keystores:/keystores \
  -v $(pwd)/validators:/data \
  sigp/lighthouse:latest-modern \
  lighthouse account validator import \
  --network gnosis \
  --password-file /keystores/password.txt \
  --reuse-password \
  --directory /keystores \
  --datadir /data
```

### Step 9: Start the Validator

**Only start validator after both execution and consensus clients are fully synced!**

```bash
# Start validator
docker-compose up -d validator

# Check validator logs
docker-compose logs -f validator
```

### Step 10: Bridge GNO to Gnosis Chain

Before making your deposit, you need GNO on Gnosis Chain:

1. **Option A: Use Transferto.xyz**
   - Go to [https://transferto.xyz](https://transferto.xyz)
   - Connect wallet
   - Bridge GNO from Ethereum to Gnosis Chain
   - Also bridge 0.1 xDAI for gas fees

2. **Option B: Use OmniBridge**
   - Go to [https://omnibridge.gnosischain.com/](https://omnibridge.gnosischain.com/)
   - Connect MetaMask to Ethereum Mainnet
   - Bridge at least 1 GNO to Gnosis Chain

### Step 11: Make Validator Deposit

1. **Navigate to deposit interface:**
   - Go to [https://deposit.gnosischain.com/](https://deposit.gnosischain.com/)

2. **Connect wallet:**
   - Connect MetaMask
   - Switch to Gnosis Chain network

3. **Upload deposit data:**
   - Click "Deposit" tab
   - Upload the `deposit_data-*.json` file from `~/gnosis/keystores/`
   - Review the deposit details

4. **Confirm transaction:**
   - Approve the transaction in MetaMask
   - Wait for transaction confirmation

5. **Wait for activation:**
   - Takes approximately 1.5 hours
   - Gnosis Chain waits for 1024 blocks + up to 64 epochs

### Step 12: Verify Validator Status

After ~1.5 hours, check your validator:

1. Find your validator public key in `deposit_data-*.json`
2. Go to [https://gnosischa.in/](https://gnosischa.in/)
3. Paste your public key in the search box
4. Verify status shows as "Active"

## 📊 Monitoring

### View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f execution
docker-compose logs -f consensus
docker-compose logs -f validator
```

### Check Container Status

```bash
# List running containers
docker ps

# Check container health
docker-compose ps
```

### Monitor Validator Performance

- **Beacon Chain Explorer:** [https://gnosischa.in/](https://gnosischa.in/)
- **Dune Analytics Dashboard:** [Gnosis Beacon Chain Stats](https://dune.com/gnosischain)

### Key Metrics to Watch

- ✅ **Attestation effectiveness:** Should be >95%
- ✅ **Proposal success rate:** Should be 100% when selected
- ✅ **Balance:** Should steadily increase
- ✅ **Peer connections:** Should have 50+ peers
- ✅ **Sync status:** Both EL and CL should show "synced"

## 🔧 Maintenance

### Update Clients

```bash
# Stop services
docker-compose down

# Pull latest images
docker-compose pull

# Restart services
docker-compose up -d
```

### Restart Services

```bash
# Restart all
docker-compose restart

# Restart specific service
docker-compose restart validator
```

### Check Disk Space

```bash
# Check disk usage
df -h

# Check Docker volume sizes
docker system df -v
```

## ⚠️ Important Notes

### Slashing Risks

You can be slashed (lose staked GNO) for:
- **Double signing:** Running same validator keys on multiple machines simultaneously
- **Surround voting:** Conflicting attestations
- **Extended downtime:** Being offline for long periods

**Never run the same validator keys on multiple machines!**

### Withdrawals

- **Partial withdrawals:** Any balance over 1 GNO automatically withdrawn to your withdrawal address
- **Full withdrawals:** Exit your validator to withdraw all staked GNO
- Withdrawals are sent to the address you specified during key generation

### Client Diversity

For network security, consider running minority clients:
- **Execution clients:** Nethermind, Erigon, Geth
- **Consensus clients:** Lighthouse, Lodestar, Teku, Prysm

Check current [client diversity stats](https://clientdiversity.org/) before choosing.

## 🛠️ Troubleshooting

### Execution client won't sync

```bash
# Check logs for errors
docker-compose logs execution

# Restart execution client
docker-compose restart execution

# If persistent, try fresh sync
docker-compose down
rm -rf ./execution/*
docker-compose up -d execution
```

### Validator missing attestations

**Common causes:**
- Execution or consensus client not fully synced
- Poor internet connection
- Not enough peers
- Insufficient system resources

**Solutions:**
```bash
# Check sync status
docker-compose logs consensus | tail -n 50

# Check peer count
docker-compose logs consensus | grep -i "peers"

# Restart consensus and validator
docker-compose restart consensus validator
```

### Out of disk space

```bash
# Check disk usage
df -h

# Clean up Docker
docker system prune -a

# Consider using Erigon (more space-efficient)
# Edit docker-compose.yml and replace Nethermind with Erigon
```

### Container keeps restarting

```bash
# Check specific container logs
docker-compose logs validator

# Check if ports are already in use
sudo netstat -tulpn | grep -E '9000|4000|8545|30303'

# Verify JWT secret exists and has correct permissions
ls -la ./jwtsecret/jwt.hex
```

## 💰 Economics

- **Minimum stake:** 1 GNO (~$100)
- **Current APY:** ~13% in GNO
- **Additional income:** Transaction fees in xDAI
- **Validators per node:** Up to 128 recommended (256 maximum)
- **Withdrawal address:** Set during key generation, cannot be changed

## 🔐 Security Best Practices

1. **Protect your mnemonic:**
   - Write on paper, store in safe place
   - Never take photos or store digitally
   - Consider metal backup for fire/water resistance

2. **Secure your server:**
   - Enable UFW firewall
   - Only open necessary ports
   - Keep system updated
   - Use SSH keys instead of passwords

3. **Backup strategy:**
   - Backup validator keystores
   - Document your setup
   - Keep withdrawal address secure

4. **Never share:**
   - Validator keys
   - Mnemonic seed phrase
   - Keystore passwords

## 📚 Useful Resources

- **Official Docs:** [https://docs.gnosischain.com/node/](https://docs.gnosischain.com/node/)
- **Validator Guide:** [https://www.validategnosis.com/](https://www.validategnosis.com/)
- **Discord:** [https://discord.gg/gnosischain](https://discord.gg/gnosischain)
- **Forum:** [https://forum.gnosis.io/](https://forum.gnosis.io/)
- **Client Docs:**
  - [Nethermind](https://docs.nethermind.io/)
  - [Lighthouse](https://lighthouse-book.sigmaprime.io/)

## 🎯 Quick Command Reference

```bash
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# View logs (all services)
docker-compose logs -f

# View logs (specific service)
docker-compose logs -f validator

# Restart specific service
docker-compose restart validator

# Check running containers
docker ps

# Update all clients
docker-compose pull && docker-compose up -d

# Check disk space
df -h

# Monitor resource usage
docker stats
```

---

**Good luck with your validator! 🎉**

*For support, join the [Gnosis Discord](https://discord.gg/gnosischain)*
