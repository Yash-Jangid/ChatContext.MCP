# Complete Assembly Guide: From Zero to Running System

## The Mental Model Before Touching Anything

```
Think of this like building a car:
  Engine     = Python scripts (does the actual work)
  Dashboard  = n8n (shows you what's happening, controls timing)
  Fuel tank  = SQLite (stores everything)
  Horn       = Gotify (alerts you when something matters)
  Driver     = You (only you decide what emails to send)

You build in this order:
  1. Lay the road (directories + system)
  2. Build the engine (Python scripts)
  3. Install the fuel tank (SQLite + DB init)
  4. Connect the horn (Gotify)
  5. Wire the dashboard (n8n)
  6. Test each part alone
  7. Test everything together
  8. Drive carefully
```

---

## PHASE 0: MACHINE REQUIREMENTS

```
Minimum specs for 24/7 operation:
  RAM:  4GB minimum, 8GB recommended
  CPU:  2 cores minimum
  Disk: 20GB free
  OS:   Ubuntu 22.04 LTS (this guide assumes this)
  
Options (all free):
  - Old laptop running Ubuntu
  - Free Oracle Cloud VM (4GB RAM, 2 CPU — permanently free tier)
  - Any spare Linux machine you have
  
Internet: Stable connection required for Tor exit routing
```

---

## PHASE 1: LAYING THE ROAD

### Step 1.1 — Update System

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y \
    curl wget git vim \
    sqlite3 \
    tor \
    python3 python3-pip python3-venv \
    build-essential \
    unzip \
    net-tools

# Verify versions
python3 --version    # Should be 3.10+
sqlite3 --version    # Should be 3.x
tor --version        # Should be 0.4+
```

### Step 1.2 — Install Docker

```bash
# Install Docker properly
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add yourself to docker group (no sudo needed)
sudo usermod -aG docker $USER

# Apply group change without logout
newgrp docker

# Verify
docker --version
docker run hello-world
```

### Step 1.3 — Install Google Chrome (NOT Chromium)

```bash
# UCD works best with real Chrome, not Chromium
wget -q -O - https://dl.google.com/linux/linux_signing_key.pub | sudo apt-key add -
echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" | \
    sudo tee /etc/apt/sources.list.d/google-chrome.list

sudo apt-get update
sudo apt-get install -y google-chrome-stable

# Verify
google-chrome --version
# Should show: Google Chrome 120.x.x.x (or current stable)
```

### Step 1.4 — Create Directory Structure

```bash
# Create ALL directories in one command
sudo mkdir -p \
    /scripts/linkedin_intel \
    /data/linkedin_intel/logs \
    /data/linkedin_intel/state \
    /data/linkedin_intel/backups \
    /data/linkedin_intel/cookies \
    /data/linkedin_intel/sessions \
    /data/linkedin_intel/notifications/pending \
    /data/linkedin_intel/raw_posts/archive \
    /data/linkedin_intel/fingerprints

# Give your user ownership
sudo chown -R $USER:$USER /scripts
sudo chown -R $USER:$USER /data

# Verify structure
tree /data/linkedin_intel
# You should see:
# /data/linkedin_intel/
# ├── backups/
# ├── cookies/
# ├── fingerprints/
# ├── logs/
# ├── notifications/
# │   └── pending/
# ├── raw_posts/
# │   └── archive/
# ├── sessions/
# └── state/
```

### Step 1.5 — Create Initial State Files

```bash
# These files tell the system its current state
echo '{"frozen": false}' > \
    /data/linkedin_intel/state/system_freeze.json

echo '{"index": 0}' > \
    /data/linkedin_intel/state/query_rotation.json

echo '{"last_used": {}, "current_index": 0}' > \
    /data/linkedin_intel/state/profile_rotation.json

# Touch log files
touch /data/linkedin_intel/logs/system.log
touch /data/linkedin_intel/logs/errors.log
touch /data/linkedin_intel/logs/sessions.log

# Verify
cat /data/linkedin_intel/state/system_freeze.json
# Should print: {"frozen": false}
```

---

## PHASE 2: BUILDING THE ENGINE (Python Environment)

### Step 2.1 — Create Virtual Environment

```bash
# Create isolated Python environment
python3 -m venv /opt/linkedin_intel_env

# Activate it
source /opt/linkedin_intel_env/bin/activate

# Verify you're in the venv
which python3
# Should show: /opt/linkedin_intel_env/bin/python3

# Upgrade pip
pip install --upgrade pip setuptools wheel
```

### Step 2.2 — Install All Python Dependencies

```bash
# Install all at once — this takes 3-5 minutes
pip install \
    undetected-chromedriver==3.5.5 \
    selenium==4.16.0 \
    playwright==1.40.0 \
    playwright-stealth==1.0.6 \
    beautifulsoup4==4.12.2 \
    lxml==4.9.3 \
    spacy==3.7.2 \
    requests==2.31.0 \
    stem==1.8.2 \
    numpy==1.26.2 \
    cryptography==41.0.7 \
    streamlit==1.29.0 \
    pandas==2.1.4

# Install Playwright browsers
playwright install chromium

# Download spaCy English model
python3 -m spacy download en_core_web_sm

# Verify critical installs
python3 -c "import undetected_chromedriver; print('UCD OK')"
python3 -c "import spacy; nlp = spacy.load('en_core_web_sm'); print('spaCy OK')"
python3 -c "import stem; print('stem OK')"
python3 -c "import bs4; print('BeautifulSoup OK')"
```

### Step 2.3 — Create Python Activation Script

```bash
# Create a convenience script you'll always run first
cat > /scripts/linkedin_intel/activate_env.sh << 'EOF'
#!/bin/bash
source /opt/linkedin_intel_env/bin/activate
export PYTHONPATH=/scripts/linkedin_intel
echo "Environment activated: $(which python3)"
EOF

chmod +x /scripts/linkedin_intel/activate_env.sh

# Add to your shell profile for convenience
echo 'alias lienv="source /scripts/linkedin_intel/activate_env.sh"' >> ~/.bashrc
source ~/.bashrc
```

---

## PHASE 3: CONFIGURE TOR (IP ROTATION ENGINE)

### Step 3.1 — Generate Tor Password Hash

```bash
# Choose a strong password and hash it
# IMPORTANT: Replace "MyTorPassword123!" with YOUR password
TOR_PASS="MyTorPassword123!"

# Generate the hash
TOR_HASH=$(tor --hash-password "$TOR_PASS" 2>/dev/null | tail -1)
echo "Your Tor hash: $TOR_HASH"
# Copy this hash — you'll need it in the next step
```

### Step 3.2 — Configure Tor

```bash
# Backup original config
sudo cp /etc/tor/torrc /etc/tor/torrc.backup

# Write new config
sudo tee /etc/tor/torrc << EOF
# LinkedIn Intel - Tor Configuration
SocksPort 9050
ControlPort 9051
HashedControlPassword ${TOR_HASH}
ExitNodes {US},{GB},{DE},{CA},{AU}
StrictNodes 1
MaxCircuitDirtiness 600
NewCircuitPeriod 30
EOF

# Restart Tor with new config
sudo systemctl restart tor
sudo systemctl enable tor

# Wait for Tor to connect (takes 15-30 seconds)
sleep 20

# Verify Tor is working
curl --socks5 127.0.0.1:9050 \
     --socks5-hostname 127.0.0.1 \
     --max-time 15 \
     https://api.ipify.org

# You should see an IP address
# Run it twice — should show DIFFERENT IPs after rotation
```

### Step 3.3 — Test Tor IP Rotation

```bash
source /scripts/linkedin_intel/activate_env.sh

python3 << 'EOF'
import requests
import time
from stem import Signal
from stem.control import Controller

TOR_PROXIES = {
    "http": "socks5h://127.0.0.1:9050",
    "https": "socks5h://127.0.0.1:9050"
}

def get_ip():
    try:
        r = requests.get("https://api.ipify.org", 
                        proxies=TOR_PROXIES, timeout=15)
        return r.text.strip()
    except Exception as e:
        return f"ERROR: {e}"

def rotate():
    with Controller.from_port(port=9051) as c:
        c.authenticate(password="MyTorPassword123!")
        c.signal(Signal.NEWNYM)
    time.sleep(5)

print("IP before rotation:", get_ip())
rotate()
print("IP after rotation:", get_ip())
print("Tor rotation: WORKING" if True else "FAILED")
EOF

# Expected output:
# IP before rotation: 185.x.x.x
# IP after rotation: 45.x.x.x   (different!)
# Tor rotation: WORKING
```

---

## PHASE 4: INSTALL THE VOICE (Gotify)

### Step 4.1 — Start Gotify

```bash
# Create data directory
sudo mkdir -p /opt/gotify/data
sudo chown -R $USER:$USER /opt/gotify

# Start Gotify container
docker run -d \
    --name gotify \
    --restart always \
    -p 8080:80 \
    -v /opt/gotify/data:/app/data \
    gotify/server

# Wait for it to start
sleep 5

# Verify it's running
docker ps | grep gotify
curl -s http://localhost:8080/health
# Should return: {"database":"ok","health":"green"}
```

### Step 4.2 — Create Gotify App Token

```bash
# Step 1: Open browser and go to http://YOUR_SERVER_IP:8080
# Default credentials: admin / admin
# 
# Step 2: Change admin password IMMEDIATELY
#   Go to: Users → Edit admin → change password
#
# Step 3: Create application token
#   Go to: Apps → Create Application
#   Name: "LinkedIn Intel"
#   Click Create
#   Copy the TOKEN shown (looks like: Abc1234xyz...)
#
# Step 4: Install Gotify app on phone
#   Android: https://github.com/gotify/android/releases
#   iOS: Use ntfy.sh instead (Gotify has no iOS app)
#
# Step 5: Test notification from command line
GOTIFY_TOKEN="your_token_here"  # Replace with real token

curl -s \
    -F "title=Test Notification" \
    -F "message=LinkedIn Intel system is online" \
    -F "priority=5" \
    "http://localhost:8080/message?token=${GOTIFY_TOKEN}"

# You should receive a push notification on your phone immediately
```

### Step 4.3 — iOS Alternative (ntfy)

```bash
# If you're on iOS, use ntfy instead of Gotify
docker run -d \
    --name ntfy \
    --restart always \
    -p 8090:80 \
    -v /opt/ntfy/data:/var/cache/ntfy \
    -v /opt/ntfy/config:/etc/ntfy \
    binwiederhier/ntfy serve

# Install ntfy app from App Store
# Subscribe to topic: http://YOUR_IP:8090/linkedin-intel
# Test:
curl -d "System online and ready" http://localhost:8090/linkedin-intel
```

---

## PHASE 5: WRITING ALL THE PYTHON SCRIPTS

### Step 5.1 — Create Each Script File

```bash
# You need to create these files IN THIS ORDER
# Each file depends on the ones before it

FILES_TO_CREATE=(
    "config.py"
    "db_manager.py"
    "ip_rotation.py"
    "fingerprint_manager.py"
    "human_behavior.py"
    "session_limit_check.py"
    "session_health.py"
    "main_scraper.py"
    "post_processor.py"
    "session_recovery.py"
    "detection_handler.py"
    "notifier.py"
    "session_teardown.py"
    "daily_maintenance.py"
    "self_heal.py"
)

for f in "${FILES_TO_CREATE[@]}"; do
    touch /scripts/linkedin_intel/$f
    echo "Created: /scripts/linkedin_intel/$f"
done

ls -la /scripts/linkedin_intel/
```

### Step 5.2 — Fill Each Script (Critical Order)

```bash
# Open your editor
# Use nano for simplicity, or vim/vscode if you prefer

# SCRIPT 1 FIRST — everything else imports this
nano /scripts/linkedin_intel/config.py
```

**Paste the complete `config.py` from the previous response, then edit these specific lines:**

```python
# FIND AND EDIT THESE LINES IN config.py:

# Line ~18: Your LinkedIn credentials
LINKEDIN_EMAIL = os.getenv("LINKEDIN_EMAIL", "YOUR_ACTUAL_EMAIL@gmail.com")
LINKEDIN_PASSWORD = os.getenv("LINKEDIN_PASSWORD", "YOUR_ACTUAL_PASSWORD")

# Line ~23: Your Tor password (same one you used in Phase 3)
TOR_CONTROL_PASSWORD = os.getenv("TOR_PASSWORD", "MyTorPassword123!")

# Line ~50-67: YOUR tech stack (edit completely)
YOUR_TECH_STACK = {
    "python", "django", "fastapi",   # Replace with YOUR skills
    "postgresql", "redis", "docker",
    # Add everything you know
}

# Line ~69-75: YOUR target roles
YOUR_TARGET_ROLES = {
    "senior software engineer",       # Replace with YOUR targets
    "backend engineer",
    # Add your specific targets
}

# Line ~77: YOUR experience level
YOUR_EXPERIENCE_LEVEL = "senior"    # entry | mid | senior | lead

# Line ~79: Minimum score to notify
HIGH_PRIORITY_THRESHOLD = 8        # Lower = more notifications
```

```bash
# After config.py, create all other scripts
# Copy each script from the previous response into its file

nano /scripts/linkedin_intel/db_manager.py
# Paste db_manager.py content

nano /scripts/linkedin_intel/ip_rotation.py
# Paste ip_rotation.py content
# EDIT LINE: TOR_CONTROL_PASSWORD to match your password

nano /scripts/linkedin_intel/fingerprint_manager.py
# Paste fingerprint_manager.py content

nano /scripts/linkedin_intel/human_behavior.py
# Paste human_behavior.py content

nano /scripts/linkedin_intel/session_limit_check.py
# Paste session_limit_check.py content

nano /scripts/linkedin_intel/session_health.py
# Paste session_health.py content

nano /scripts/linkedin_intel/main_scraper.py
# Paste main_scraper.py content

nano /scripts/linkedin_intel/post_processor.py
# Paste post_processor.py content

nano /scripts/linkedin_intel/session_recovery.py
# Paste session_recovery.py content

nano /scripts/linkedin_intel/detection_handler.py
# Paste detection_handler.py content

nano /scripts/linkedin_intel/notifier.py
# Paste notifier.py content

nano /scripts/linkedin_intel/session_teardown.py
# Paste session_teardown.py content

nano /scripts/linkedin_intel/daily_maintenance.py
# Paste daily_maintenance.py content

nano /scripts/linux_intel/self_heal.py
# Paste self_heal.py content
```

### Step 5.3 — Create the n8n Environment Variables File

```bash
# n8n reads this file for all secrets
# Keep this file PRIVATE

cat > ~/.n8n/.env << EOF
LINKEDIN_EMAIL=your_actual_email@gmail.com
LINKEDIN_PASSWORD=your_actual_password
GOTIFY_APP_TOKEN=your_gotify_token_from_step_4
TOR_PASSWORD=MyTorPassword123!
GOTIFY_URL=http://localhost:8080
N8N_ENCRYPTION_KEY=generate_a_random_32char_string_here
EOF

chmod 600 ~/.n8n/.env
echo "Secrets file created and locked"
```

---

## PHASE 6: INITIALIZE THE MEMORY (SQLite)

### Step 6.1 — Initialize Database

```bash
source /scripts/linkedin_intel/activate_env.sh
cd /scripts/linkedin_intel

python3 << 'EOF'
import sys
sys.path.insert(0, '/scripts/linkedin_intel')
from db_manager import initialize_database

initialize_database()
print("Database initialized successfully")

# Verify tables were created
import sqlite3
conn = sqlite3.connect('/data/linkedin_intel/leads.db')
tables = conn.execute(
    "SELECT name FROM sqlite_master WHERE type='table'"
).fetchall()
print(f"Tables created: {[t[0] for t in tables]}")
conn.close()
EOF

# Expected output:
# Database initialized successfully
# Tables created: ['leads', 'sessions', 'email_dedup', 
#                  'detection_events', 'query_rotation']
```

### Step 6.2 — Verify Database Structure

```bash
# Inspect the database directly
sqlite3 /data/linkedin_intel/leads.db << 'EOF'
.tables
.schema leads
.schema sessions
.quit
EOF
```

---

## PHASE 7: TEST EACH PYTHON SCRIPT INDEPENDENTLY

**This is the most important phase. Test each script alone before connecting to n8n.**

### Test 7.1 — IP Rotation Script

```bash
source /scripts/linkedin_intel/activate_env.sh
cd /scripts/linkedin_intel

python3 ip_rotation.py rotate
# Expected JSON output:
# {"success": true, "ip": "x.x.x.x", "country": "US", 
#  "timezone": "America/New_York", "attempt": 1}

python3 ip_rotation.py current
# Expected:
# {"success": true, "ip": "x.x.x.x", "country": "US", 
#  "usable": true, "reason": "OK"}
```

### Test 7.2 — Session Limit Check

```bash
python3 session_limit_check.py test_session_001 3
# Expected (first run, no sessions today):
# {"canProceed": "true", "sessionId": "test_session_001", 
#  "todayCount": 0, "maxSessions": 3}
```

### Test 7.3 — Detection Handler

```bash
python3 detection_handler.py test_session_001 SOFT_SIGNAL
# Expected:
# {"action": "SOFT_PAUSE", "freezeHours": 2, 
#  "event": "SOFT_SIGNAL", "sessionId": "test_session_001"}

# Check freeze file was written
cat /data/linkedin_intel/state/system_freeze.json
# Should show frozen: true, resumeAfter: [2 hours from now]

# IMPORTANT: Unfreeze now so testing continues
echo '{"frozen": false}' > /data/linkedin_intel/state/system_freeze.json
```

### Test 7.4 — Notifier

```bash
# Replace with your real Gotify token before testing
python3 notifier.py '[{"role_title": "Senior Engineer", "author_name": "John Smith", "author_company": "TechCorp", "emails": ["john@techcorp.com"], "tech_stack": ["python", "docker"], "remote_signal": "remote", "relevance_score": 9, "post_url": "https://linkedin.com/test"}]' test_session_001

# Expected:
# {"notificationsSent": 1, "totalLeads": 1, "sessionId": "test_session_001"}
# AND you should receive a push notification on your phone
```

### Test 7.5 — Database Write/Read Cycle

```bash
python3 << 'EOF'
import sys
sys.path.insert(0, '/scripts/linkedin_intel')
from db_manager import save_lead, get_leads_by_status, initialize_database

initialize_database()

# Write a test lead
test_lead = {
    "author_name": "Test Recruiter",
    "author_url": "https://linkedin.com/in/test-recruiter-123",
    "author_headline": "Head of Talent @ TestCorp",
    "author_company": "TestCorp",
    "post_url": "https://linkedin.com/feed/update/test123",
    "post_timestamp": "2024-01-15T10:00:00",
    "post_text": "We're hiring a Senior Python Engineer. Email: hire@testcorp.com",
    "emails": ["hire@testcorp.com"],
    "hashtags": ["#hiring", "#python"],
    "tech_stack": ["python", "docker"],
    "role_title": "Senior Python Engineer",
    "experience_level": "senior",
    "remote_signal": "remote",
    "location": "Remote",
    "salary": "$150k-180k",
    "relevance_score": 9,
    "bait_score": 0,
    "session_id": "test_session_001"
}

result = save_lead(test_lead)
print(f"Lead saved (new): {result}")

# Try duplicate
result2 = save_lead(test_lead)
print(f"Duplicate prevented: {not result2}")

# Read back
leads = get_leads_by_status("NEW")
print(f"Leads in DB: {len(leads)}")
print(f"First lead email: {leads[0]['emails']}")
EOF

# Expected:
# Lead saved (new): True
# Duplicate prevented: True
# Leads in DB: 1
# First lead email: ["hire@testcorp.com"]
```

### Test 7.6 — CRITICAL: Manual Browser Test

```bash
# This tests the browser fingerprint BEFORE any LinkedIn scraping
# Run this on a machine with a display, or use VNC

source /scripts/linkedin_intel/activate_env.sh
cd /scripts/linkedin_intel

python3 << 'EOF'
import sys
sys.path.insert(0, '/scripts/linkedin_intel')
import undetected_chromedriver as uc
from fingerprint_manager import get_next_profile, apply_fingerprint_profile
from config import TOR_SOCKS_HOST, TOR_SOCKS_PORT
import time

profile = get_next_profile()
print(f"Using profile: {profile['id']}")

options = uc.ChromeOptions()
options.add_argument(f"--window-size={profile['window_width']},{profile['window_height']}")
options.add_argument(f"--proxy-server=socks5://{TOR_SOCKS_HOST}:{TOR_SOCKS_PORT}")
options.add_argument("--no-sandbox")

driver = uc.Chrome(options=options, headless=False, use_subprocess=True)
apply_fingerprint_profile(driver, profile)

# Navigate to bot detection test site
driver.get("https://bot.sannysoft.com")
time.sleep(5)

# Take screenshot to verify
driver.save_screenshot("/data/linkedin_intel/logs/bot_test.png")
print("Screenshot saved to /data/linkedin_intel/logs/bot_test.png")
print("Check: All tests should show GREEN (passed)")
print(f"Current UA: {driver.execute_script('return navigator.userAgent')}")
print(f"Webdriver: {driver.execute_script('return navigator.webdriver')}")

time.sleep(10)  # View the page
driver.quit()
print("Browser test complete")
EOF

# After running:
# 1. View the screenshot at /data/linkedin_intel/logs/bot_test.png
# 2. All items should show green / not detected
# 3. navigator.webdriver should be: undefined (not true)
```

---

## PHASE 8: FIRST LINKEDIN SESSION (Manual, Supervised)

**Do NOT automate yet. Do the first session manually to establish cookies.**

### Step 8.1 — Manual Login to LinkedIn

```bash
source /scripts/linkedin_intel/activate_env.sh
cd /scripts/linkedin_intel

python3 << 'EOF'
import sys, time, pickle
sys.path.insert(0, '/scripts/linkedin_intel')
import undetected_chromedriver as uc
from fingerprint_manager import get_next_profile, apply_fingerprint_profile
from config import TOR_SOCKS_HOST, TOR_SOCKS_PORT, LINKEDIN_EMAIL, LINKEDIN_PASSWORD
from pathlib import Path
from human_behavior import gaussian_wait, human_type, human_click
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

COOKIE_FILE = Path("/data/linkedin_intel/cookies/linkedin_session.pkl")

profile = get_next_profile()
options = uc.ChromeOptions()
options.add_argument(f"--window-size={profile['window_width']},{profile['window_height']}")
options.add_argument(f"--proxy-server=socks5://{TOR_SOCKS_HOST}:{TOR_SOCKS_PORT}")
options.add_argument("--no-sandbox")
options.add_argument("--disable-dev-shm-usage")
options.add_argument(f"--user-data-dir=/data/linkedin_intel/sessions/{profile['id']}")

driver = uc.Chrome(options=options, headless=False, use_subprocess=True)
apply_fingerprint_profile(driver, profile)

print("Opening LinkedIn login page...")
driver.get("https://www.linkedin.com/login")
gaussian_wait(3.0, 1.0)

print("Typing email...")
email_field = WebDriverWait(driver, 10).until(
    EC.presence_of_element_located((By.ID, "username"))
)
human_type(driver, email_field, LINKEDIN_EMAIL)

print("Typing password...")
password_field = driver.find_element(By.ID, "password")
human_type(driver, password_field, LINKEDIN_PASSWORD)

print("Clicking login...")
submit = driver.find_element(By.CSS_SELECTOR, "[type='submit']")
human_click(driver, submit)

gaussian_wait(5.0, 1.0)

print(f"Current URL: {driver.current_url}")

# Handle 2FA if needed
if any(x in driver.current_url for x in ["checkpoint", "challenge", "two-step"]):
    print("\n=== 2FA REQUIRED ===")
    print("Complete the 2FA verification in the browser window")
    print("You have 3 minutes...")
    for i in range(36):
        time.sleep(5)
        if "feed" in driver.current_url:
            print("2FA completed!")
            break
        print(f"Waiting for 2FA... ({(i+1)*5}s)")

if "feed" in driver.current_url:
    print("Login SUCCESSFUL")
    # Save cookies
    COOKIE_FILE.parent.mkdir(parents=True, exist_ok=True)
    with open(COOKIE_FILE, "wb") as f:
        pickle.dump(driver.get_cookies(), f)
    print(f"Session cookies saved to: {COOKIE_FILE}")
    print(f"Cookie count: {len(driver.get_cookies())}")
else:
    print(f"Login FAILED - URL: {driver.current_url}")

print("\nBrowse normally for 2-3 minutes to warm up the session...")
time.sleep(180)  # 3 minute warmup browsing

driver.quit()
print("Done. Session saved.")
EOF
```

### Step 8.2 — Verify Saved Session

```bash
python3 << 'EOF'
import pickle
from pathlib import Path

cookie_file = Path("/data/linkedin_intel/cookies/linkedin_session.pkl")
if cookie_file.exists():
    with open(cookie_file, "rb") as f:
        cookies = pickle.load(f)
    print(f"Session file exists: YES")
    print(f"Cookie count: {len(cookies)}")
    # Show cookie names (not values for security)
    names = [c.get('name', 'unnamed') for c in cookies]
    print(f"Cookie names: {names[:10]}")
    # li_at is the critical auth cookie
    has_auth = any(c.get('name') == 'li_at' for c in cookies)
    print(f"Auth cookie (li_at) present: {has_auth}")
else:
    print("ERROR: No session file found")
EOF
```

### Step 8.3 — Test Session Health Check

```bash
source /scripts/linkedin_intel/activate_env.sh
cd /scripts/linkedin_intel

python3 session_health.py check test_health_001
# Expected:
# {"status": "HEALTHY", "needsReauth": false, 
#  "currentUrl": "https://www.linkedin.com/feed/", 
#  "checkedAt": "...", "sessionId": "test_health_001"}
```

---

## PHASE 9: TEST THE FULL SCRAPING PIPELINE (Manual)

### Step 9.1 — First Manual Scrape Test

```bash
source /scripts/linkedin_intel/activate_env.sh
cd /scripts/linkedin_intel

# Generate a session ID
SESSION_ID="sess_$(date +%s)_manual_test"
echo "Session ID: $SESSION_ID"

# Run the scraper
python3 main_scraper.py $SESSION_ID

# Expected output (JSON):
# {
#   "status": "COMPLETED",
#   "sessionId": "sess_..._manual_test",
#   "query": "\"email\" #hiring #wearehiring",
#   "profileId": "profile_001",
#   "posts_scanned": 47,
#   "emails_found": 3,
#   "leads_qualified": 3,
#   "detection_events": []
# }
```

### Step 9.2 — Check Raw Posts File

```bash
# Find the raw posts file for this session
ls -la /data/linkedin_intel/raw_posts/

# View the contents (should have real recruiter posts)
python3 << 'EOF'
import json
from pathlib import Path

# Get most recent raw posts file
raw_dir = Path("/data/linkedin_intel/raw_posts")
files = sorted(raw_dir.glob("*_raw.json"), key=lambda f: f.stat().st_mtime)

if not files:
    print("No raw posts found yet")
else:
    latest = files[-1]
    print(f"Raw posts file: {latest.name}")
    posts = json.loads(latest.read_text())
    print(f"Posts captured: {len(posts)}")
    
    for i, post in enumerate(posts[:3]):  # Show first 3
        print(f"\n--- Post {i+1} ---")
        print(f"Author: {post.get('author_name', 'Unknown')}")
        print(f"Emails: {post.get('emails', [])}")
        print(f"Bait score: {post.get('bait_score', 0)}")
        print(f"Text preview: {post.get('post_text', '')[:200]}...")
EOF
```

### Step 9.3 — Run Post Processor

```bash
# Use the same SESSION_ID from Step 9.1
python3 post_processor.py $SESSION_ID

# Expected output:
# {
#   "leadsStored": 2,
#   "highPriorityLeads": [...],
#   "totalProcessed": 3,
#   "sessionId": "..."
# }
```

### Step 9.4 — Verify Leads in Database

```bash
python3 << 'EOF'
import sys, json
sys.path.insert(0, '/scripts/linkedin_intel')
from db_manager import get_leads_by_status, get_daily_stats

leads = get_leads_by_status("NEW", limit=5)
print(f"Leads in database: {len(leads)}")

for lead in leads:
    print(f"\n{'='*50}")
    print(f"Author: {lead['author_name']}")
    print(f"Role: {lead['role_title']}")
    print(f"Company: {lead['author_company']}")
    print(f"Emails: {lead['emails']}")
    print(f"Stack: {lead['tech_stack']}")
    print(f"Score: {lead['relevance_score']}")
    print(f"Remote: {lead['remote_signal']}")

stats = get_daily_stats()
print(f"\nDaily stats: {json.dumps(stats, indent=2)}")
EOF
```

---

## PHASE 10: LAUNCH n8n AND BUILD WORKFLOWS

### Step 10.1 — Start n8n

```bash
# Create n8n config directory
mkdir -p ~/.n8n

# Create environment file
cat > ~/.n8n/.env << EOF
LINKEDIN_EMAIL=your_email@gmail.com
LINKEDIN_PASSWORD=your_password
GOTIFY_APP_TOKEN=your_gotify_token
TOR_PASSWORD=MyTorPassword123!
GOTIFY_URL=http://localhost:8080
N8N_ENCRYPTION_KEY=$(openssl rand -hex 16)
EXECUTIONS_DATA_SAVE_ON_ERROR=all
EXECUTIONS_DATA_SAVE_ON_SUCCESS=none
EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=false
N8N_LOG_LEVEL=info
EOF

# Start n8n container
docker run -d \
    --name n8n \
    --restart always \
    -p 5678:5678 \
    -v ~/.n8n:/home/node/.n8n \
    -v /scripts:/scripts \
    -v /data:/data \
    -v /opt/linkedin_intel_env:/opt/linkedin_intel_env \
    --env-file ~/.n8n/.env \
    n8nio/n8n

# Wait for startup
sleep 15

# Verify
docker ps | grep n8n
curl -s http://localhost:5678/healthz
# Should return: {"status":"ok"}
```

### Step 10.2 — Access n8n UI

```
Open browser: http://YOUR_SERVER_IP:5678

First time setup:
1. Click "Create account"
2. Enter email and password
3. Skip the "Which integration" question
4. You're in the n8n dashboard
```

### Step 10.3 — Create Global Credentials in n8n

```
In n8n UI:
1. Click top-right menu → "Credentials"
2. Click "Add credential"

Create these (not strictly needed since we use 
env vars, but useful for the HTTP Request nodes):

Credential 1: Gotify
  Type: Header Auth
  Name: Gotify Token
  Header Name: X-Gotify-Key
  Header Value: your_gotify_token

Credential 2: None needed for others 
  (Python scripts handle all auth directly)
```

### Step 10.4 — Import Workflows Into n8n

```
IMPORTANT: Import in this exact order

Step 1: Click "Workflows" in left sidebar

Step 2: Click "Add workflow" → "Import from JSON"

Step 3: Import Workflow 1 (Error Handler):
  - Copy the COMPLETE JSON from "WORKFLOW 2: ERROR HANDLER"
    in the previous response
  - Paste into the import dialog
  - Click Import
  - Click Save
  - DO NOT activate yet

Step 4: Import Workflow 2 (Master Orchestrator):
  - Copy the COMPLETE JSON from "WORKFLOW 1: MASTER ORCHESTRATOR"
  - Paste and import
  - Click Save
  - DO NOT activate yet

Step 5: Import Workflow 3 (Freeze Guard):
  - Copy the COMPLETE JSON from "WORKFLOW 3: FREEZE GUARD"
  - Paste and import
  - Click Save
  - Activate this one NOW (toggle in top right)

Step 6: Import Workflow 4 (Daily Maintenance):
  - Copy the COMPLETE JSON from "WORKFLOW 4: DAILY CLEANUP"
  - Paste and import
  - Click Save
  - DO NOT activate yet
```

### Step 10.5 — Link Error Handler to Master Orchestrator

```
In n8n UI:

1. Open "Master Orchestrator" workflow
2. Click the three-dot menu (top right) → Settings
3. Find "Error Workflow" dropdown
4. Select "LinkedIn Recruiter Intelligence - Error Handler"
5. Click Save

This wires the two workflows together.
Any crash in Master Orchestrator triggers Error Handler.
```

### Step 10.6 — Fix the Execute Command Nodes

```
Every Execute Command node needs the venv path prepended.

In n8n, open Master Orchestrator workflow.
Click each Execute Command node and verify the command:

WRONG:  python3 /scripts/linkedin_intel/ip_rotation.py rotate
RIGHT:  /opt/linkedin_intel_env/bin/python3 /scripts/linkedin_intel/ip_rotation.py rotate

Fix ALL Execute Command nodes to use full path:
  /opt/linkedin_intel_env/bin/python3

Nodes to fix:
  - Session Limit Check
  - IP Rotation Agent  
  - Session Health Check
  - Session Recovery Agent
  - Main Scraper Agent
  - Post Processor + Enricher
  - Detection Handler
  - Send Priority Notifications
  - Session Teardown
  - Emergency IP Rotation
  - Self-Heal Components (in Freeze Guard workflow)
  - Run Daily Maintenance Script (in Daily Maintenance)
```

### Step 10.7 — Set Environment Variables in n8n Nodes

```
For all HTTP Request nodes that call Gotify:
The URL uses: http://localhost:8080/message
The token uses: ={{$env.GOTIFY_APP_TOKEN}}

Verify n8n can read env vars:
1. In n8n, create a new test workflow
2. Add a "Code" node
3. Paste: return [{json: {token: $env.GOTIFY_APP_TOKEN}}]
4. Run it
5. Should show your actual token value
6. Delete this test workflow after verifying
```

---

## PHASE 11: TEST WORKFLOWS INDIVIDUALLY IN n8n

### Step 11.1 — Test Error Handler

```
In n8n UI:
1. Open "Error Handler" workflow
2. Click the "Error Trigger" node
3. Click "Execute Node" (just this node)
4. It won't trigger (needs an actual error)
5. Instead, manually trigger the whole workflow:
   - Click "Execute Workflow" button (top right)
   - It should run through the classifier with empty data
   - Verify no crash — just empty output
6. Check: Gotify notification NOT sent (because no real error)
```

### Step 11.2 — Test Freeze Guard

```
In n8n UI:
1. Open "Freeze Guard" workflow — it's already active
2. Click "Execute Workflow" manually
3. Watch each node turn green

Expected flow:
  5-Minute Guard Trigger → Check Freeze State
  → Just Unfrozen? (NO, system not frozen)
  → Component Health Check
  → Components Down? (NO if everything is running)
  → Stops here (no alert sent)

If you see Component Health Check FAIL:
  Check that Tor is running: sudo systemctl status tor
  Check that Gotify is running: docker ps | grep gotify
  Check that DB exists: ls /data/linkedin_intel/leads.db
```

### Step 11.3 — Test Master Orchestrator (Single Manual Run)

```
In n8n UI:
1. Open "Master Orchestrator" workflow
2. DO NOT activate it yet
3. Click "Execute Workflow" to run ONCE manually
4. Watch each node in real-time

Node-by-node expected behavior:

  ✓ Master Cron Scheduler → fires immediately (manual run)
  ✓ Jitter + Week Variation → outputs session config JSON
  ✓ Blackout Day Check → goes to FALSE path (not blackout)
  ✓ Session Limit Check → outputs {"canProceed": "true", ...}
  ✓ Parse Session Limit Result → parses the JSON
  ✓ Can Proceed? → goes to TRUE path
  ✓ Apply Jitter Delay → waits 0-8 minutes (you see a delay)
  ✓ IP Rotation Agent → outputs new Tor IP
  ✓ Parse IP Rotation Result → parses result
  ✓ IP Rotation Successful? → goes to TRUE path
  ✓ Session Health Check → runs, outputs HEALTHY
  ✓ Parse Health Result → parses result
  ✓ Session Healthy? → goes to TRUE path
  ✓ Main Scraper Agent → RUNS FOR ~20 MINUTES
  ✓ Parse Scraper Result → outputs post counts
  ✓ Scraper Completed? → goes to TRUE path
  ✓ Post Processor + Enricher → enriches and saves
  ✓ Parse Processor Result → shows leads stored count
  ✓ High Priority Leads Found? → branches correctly
  ✓ Session Teardown → cleans up
  ✓ Log Session Complete → writes to log

If Main Scraper Agent shows ERROR:
  Click the node → click "Details" → see the error message
  Most common fix: wrong Python path in command
  Run this to debug:
    /opt/linkedin_intel_env/bin/python3 \
      /scripts/linkedin_intel/main_scraper.py \
      test_debug_001
```

### Step 11.4 — Fix Common n8n Issues

```bash
# ISSUE 1: "python3 not found"
# FIX: Use full path in all Execute Command nodes
# Change: python3 /scripts/...
# To:     /opt/linkedin_intel_env/bin/python3 /scripts/...

# ISSUE 2: "No module named 'xxx'"
# FIX: Script runs as different user inside n8n Docker
# Check what user n8n runs as:
docker exec n8n whoami
# If it's "node", the venv needs to be accessible to node user

# Grant access:
sudo chmod -R 755 /opt/linkedin_intel_env
sudo chmod -R 755 /scripts/linkedin_intel
sudo chmod -R 777 /data/linkedin_intel

# ISSUE 3: "Permission denied" on /data/
# FIX:
sudo chmod -R 777 /data/linkedin_intel
# (liberal for dev, tighten in production)

# ISSUE 4: Tor not accessible from inside n8n Docker
# FIX: n8n container needs host network access
# Stop current n8n:
docker stop n8n && docker rm n8n

# Restart with host network mode:
docker run -d \
    --name n8n \
    --restart always \
    --network host \
    -v ~/.n8n:/home/node/.n8n \
    -v /scripts:/scripts \
    -v /data:/data \
    -v /opt/linkedin_intel_env:/opt/linkedin_intel_env \
    --env-file ~/.n8n/.env \
    n8nio/n8n

# With --network host, n8n can reach localhost:9050 (Tor)
# and localhost:8080 (Gotify)
# n8n will now be at http://localhost:5678
```

---

## PHASE 12: BUILD THE STREAMLIT DASHBOARD

### Step 12.1 — Create Dashboard Script

```bash
cat > /scripts/linkedin_intel/dashboard.py << 'DASHBOARD_EOF'
"""
LinkedIn Recruiter Intelligence - Review Dashboard
Run with: streamlit run /scripts/linkedin_intel/dashboard.py
Access at: http://localhost:8501
"""

import sys
sys.path.insert(0, '/scripts/linkedin_intel')

import json
import sqlite3
import pandas as pd
import streamlit as st
from datetime import datetime, timedelta
from pathlib import Path
from db_manager import get_connection, get_daily_stats, LEADS_DB

st.set_page_config(
    page_title="LinkedIn Intel Dashboard",
    page_icon="🎯",
    layout="wide"
)

# ── Header ──────────────────────────────────────────────────────────────────
st.title("🎯 LinkedIn Recruiter Intelligence")
st.caption(f"Last refreshed: {datetime.now().strftime('%H:%M:%S')}")

# ── Auto-refresh every 5 minutes ────────────────────────────────────────────
st.markdown("""
<script>
setTimeout(function() { window.location.reload(); }, 300000);
</script>
""", unsafe_allow_html=True)

# ── Metrics Row ─────────────────────────────────────────────────────────────
stats = get_daily_stats()
col1, col2, col3, col4, col5 = st.columns(5)

with col1:
    st.metric("🆕 New Today", stats["newLeads"])
with col2:
    st.metric("⭐ High Priority", stats["highPriorityLeads"])
with col3:
    st.metric("🤖 Sessions Today", stats["sessionsRun"])
with col4:
    st.metric("📊 Total in DB", stats["totalLeadsInDB"])
with col5:
    # System status
    freeze_file = Path("/data/linkedin_intel/state/system_freeze.json")
    try:
        freeze = json.loads(freeze_file.read_text())
        if freeze.get("frozen"):
            st.metric("⚙️ System", "🔴 FROZEN")
        else:
            st.metric("⚙️ System", "🟢 ACTIVE")
    except Exception:
        st.metric("⚙️ System", "⚪ UNKNOWN")

st.divider()

# ── Sidebar Controls ─────────────────────────────────────────────────────────
with st.sidebar:
    st.header("Filters")
    status_filter = st.selectbox(
        "Status", ["NEW", "REVIEWED", "CONTACTED", "REJECTED", "ALL"]
    )
    min_score = st.slider("Minimum Score", 0, 15, 0)
    limit = st.selectbox("Show", [25, 50, 100, 200])

    st.divider()
    st.header("System State")

    # Freeze control
    freeze_file = Path("/data/linkedin_intel/state/system_freeze.json")
    try:
        freeze = json.loads(freeze_file.read_text())
        if freeze.get("frozen"):
            st.error(f"🔴 FROZEN\nReason: {freeze.get('reason', 'unknown')}")
            resume = freeze.get("resumeAfter", "unknown")
            st.caption(f"Resumes: {resume}")
            if st.button("⚡ Force Unfreeze"):
                freeze_file.write_text('{"frozen": false}')
                st.success("System unfrozen")
                st.rerun()
        else:
            st.success("🟢 System Active")
            if st.button("⏸️ Manual Freeze"):
                freeze_data = {
                    "frozen": True,
                    "reason": "MANUAL_FREEZE",
                    "frozenAt": datetime.utcnow().isoformat(),
                    "resumeAfter": (datetime.utcnow() + timedelta(hours=24)).isoformat()
                }
                freeze_file.write_text(json.dumps(freeze_data))
                st.warning("System frozen for 24h")
                st.rerun()
    except Exception as e:
        st.error(f"Cannot read state: {e}")

# ── Load Leads ───────────────────────────────────────────────────────────────
conn = get_connection()
try:
    if status_filter == "ALL":
        where_clause = ""
        params = []
    else:
        where_clause = "WHERE status = ?"
        params = [status_filter]

    if min_score > 0:
        if where_clause:
            where_clause += " AND relevance_score >= ?"
        else:
            where_clause = "WHERE relevance_score >= ?"
        params.append(min_score)

    query = f"""
        SELECT * FROM leads 
        {where_clause}
        ORDER BY relevance_score DESC, created_at DESC 
        LIMIT {limit}
    """
    df = pd.read_sql_query(query, conn, params=params)
finally:
    conn.close()

# ── Leads Table ──────────────────────────────────────────────────────────────
st.subheader(f"📋 Leads ({len(df)} shown)")

if df.empty:
    st.info("No leads match your filters. The system may still be running its first sessions.")
else:
    # Parse JSON columns for display
    df["emails_display"] = df["emails"].apply(
        lambda x: ", ".join(json.loads(x)) if x else ""
    )
    df["stack_display"] = df["tech_stack"].apply(
        lambda x: ", ".join(json.loads(x)[:4]) if x else ""
    )

    # Display columns
    display_df = df[[
        "relevance_score", "status", "role_title", "author_name",
        "author_company", "emails_display", "stack_display",
        "remote_signal", "location", "created_at"
    ]].copy()

    display_df.columns = [
        "Score", "Status", "Role", "Author",
        "Company", "Email(s)", "Stack",
        "Work", "Location", "Found At"
    ]

    st.dataframe(
        display_df,
        use_container_width=True,
        hide_index=True,
        column_config={
            "Score": st.column_config.NumberColumn(format="%d ⭐"),
            "Status": st.column_config.TextColumn(),
        }
    )

    # Export
    csv = display_df.to_csv(index=False)
    st.download_button(
        "📥 Export CSV",
        data=csv,
        file_name=f"leads_{datetime.now().strftime('%Y%m%d')}.csv",
        mime="text/csv"
    )

# ── Lead Detail View ─────────────────────────────────────────────────────────
st.divider()
st.subheader("🔍 Lead Detail + Actions")

if not df.empty:
    lead_options = {
        f"{row['role_title']} @ {row['author_company']} (Score: {row['relevance_score']})": row['id']
        for _, row in df.iterrows()
    }

    selected_label = st.selectbox("Select a lead to review:", list(lead_options.keys()))

    if selected_label:
        lead_id = lead_options[selected_label]
        lead = df[df["id"] == lead_id].iloc[0]

        col_a, col_b = st.columns([1, 1])

        with col_a:
            st.markdown("**Lead Details**")
            st.write(f"**Author:** {lead['author_name']}")
            st.write(f"**Headline:** {lead['author_headline']}")
            st.write(f"**Company:** {lead['author_company']}")
            emails = json.loads(lead['emails']) if lead['emails'] else []
            for email in emails:
                st.code(email)
            st.write(f"**Role:** {lead['role_title']}")
            st.write(f"**Level:** {lead['experience_level']}")
            st.write(f"**Work:** {lead['remote_signal']}")
            st.write(f"**Location:** {lead['location']}")
            st.write(f"**Salary:** {lead['salary'] or 'Not mentioned'}")
            stack = json.loads(lead['tech_stack']) if lead['tech_stack'] else []
            st.write(f"**Stack:** {', '.join(stack)}")

            if lead['post_url']:
                st.link_button("🔗 View Original Post", lead['post_url'])

        with col_b:
            st.markdown("**Post Text**")
            st.text_area(
                "Full post content:",
                value=lead['post_text'],
                height=200,
                disabled=True
            )

            st.markdown("**📧 Email Draft Template** (copy and personalize)")
            email_draft = f"""Subject: {lead['role_title']} - Experienced {', '.join(YOUR_STACK[:2]) if (YOUR_STACK := stack[:2]) else 'Engineer'}

Hi {lead['author_name'].split()[0] if lead['author_name'] else 'there'},

I came across your post about the {lead['role_title']} role{' at ' + lead['author_company'] if lead['author_company'] else ''}.

[2-3 sentences about your most relevant experience for THIS specific role]

Key relevant experience:
• [Most relevant project/achievement]
• [Second most relevant skill match: {', '.join(stack[:2])}]

[1 sentence about why this role/company specifically interests you]

Happy to share my portfolio/resume if this looks like a fit.

Best,
[Your name]"""

            st.text_area(
                "Draft email (NOT auto-sent — copy this manually):",
                value=email_draft,
                height=250
            )

        # Status Update Buttons
        st.markdown("**Update Status**")
        btn_col1, btn_col2, btn_col3, btn_col4 = st.columns(4)

        def update_status(new_status):
            conn = get_connection()
            try:
                conn.execute(
                    "UPDATE leads SET status = ? WHERE id = ?",
                    (new_status, lead_id)
                )
                conn.commit()
            finally:
                conn.close()
            st.rerun()

        with btn_col1:
            if st.button("👁️ Mark Reviewed"):
                update_status("REVIEWED")
        with btn_col2:
            if st.button("✉️ Mark Contacted"):
                update_status("CONTACTED")
        with btn_col3:
            if st.button("❌ Reject"):
                update_status("REJECTED")
        with btn_col4:
            if st.button("🔄 Reset to NEW"):
                update_status("NEW")

        # Notes
        st.text_area("Notes (save manually):", key=f"notes_{lead_id}")

# ── Session History ──────────────────────────────────────────────────────────
st.divider()
st.subheader("📈 Recent Sessions")

conn = get_connection()
try:
    sessions_df = pd.read_sql_query("""
        SELECT session_id, started_at, status,
               posts_scanned, emails_found, leads_stored,
               profile_used, ip_used
        FROM sessions
        ORDER BY started_at DESC
        LIMIT 10
    """, conn)
finally:
    conn.close()

if not sessions_df.empty:
    st.dataframe(sessions_df, use_container_width=True, hide_index=True)
else:
    st.info("No sessions recorded yet")

DASHBOARD_EOF

echo "Dashboard created at /scripts/linkedin_intel/dashboard.py"
```

### Step 12.2 — Launch Dashboard

```bash
source /scripts/linkedin_intel/activate_env.sh

# Run dashboard (accessible at http://localhost:8501)
streamlit run /scripts/linkedin_intel/dashboard.py \
    --server.port 8501 \
    --server.address 0.0.0.0 \
    --server.headless true &

echo "Dashboard running at http://$(hostname -I | awk '{print $1}'):8501"
```

### Step 12.3 — Make Dashboard Start Automatically

```bash
# Create systemd service for dashboard
sudo tee /etc/systemd/system/linkedin-intel-dashboard.service << 'EOF'
[Unit]
Description=LinkedIn Intel Streamlit Dashboard
After=network.target

[Service]
Type=simple
User=YOUR_USERNAME
WorkingDirectory=/scripts/linkedin_intel
Environment=PYTHONPATH=/scripts/linkedin_intel
ExecStart=/opt/linkedin_intel_env/bin/streamlit run \
    /scripts/linkedin_intel/dashboard.py \
    --server.port 8501 \
    --server.address 0.0.0.0 \
    --server.headless true
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Replace YOUR_USERNAME with actual username
sudo sed -i "s/YOUR_USERNAME/$USER/g" \
    /etc/systemd/system/linkedin-intel-dashboard.service

sudo systemctl daemon-reload
sudo systemctl enable linkedin-intel-dashboard
sudo systemctl start linkedin-intel-dashboard

# Verify
sudo systemctl status linkedin-intel-dashboard
```

---

## PHASE 13: FULL INTEGRATION TEST

### Step 13.1 — End-to-End Integration Checklist

```bash
# Run this complete verification script
cat > /scripts/linkedin_intel/verify_system.sh << 'VERIFY_EOF'
#!/bin/bash

echo "=========================================="
echo "  LinkedIn Intel System Verification"
echo "=========================================="
echo ""

PASS=0
FAIL=0

check() {
    local name="$1"
    local result="$2"
    if [ "$result" = "OK" ]; then
        echo "  ✅ $name"
        ((PASS++))
    else
        echo "  ❌ $name: $result"
        ((FAIL++))
    fi
}

echo "[ INFRASTRUCTURE ]"

# Docker running
docker ps > /dev/null 2>&1 && \
    check "Docker daemon" "OK" || \
    check "Docker daemon" "NOT RUNNING"

# n8n running
docker ps | grep -q n8n && \
    check "n8n container" "OK" || \
    check "n8n container" "NOT RUNNING"

# Gotify running
docker ps | grep -q gotify && \
    check "Gotify container" "OK" || \
    check "Gotify container" "NOT RUNNING"

# Tor running
systemctl is-active tor > /dev/null 2>&1 && \
    check "Tor service" "OK" || \
    check "Tor service" "NOT RUNNING"

echo ""
echo "[ NETWORK ]"

# Tor connectivity
TOR_IP=$(curl -s --socks5 127.0.0.1:9050 \
    --socks5-hostname 127.0.0.1 \
    --max-time 10 https://api.ipify.org 2>/dev/null)
[ -n "$TOR_IP" ] && \
    check "Tor exit IP" "OK ($TOR_IP)" || \
    check "Tor exit IP" "UNREACHABLE"

# Gotify accessible
curl -s --max-time 5 http://localhost:8080/health > /dev/null 2>&1 && \
    check "Gotify HTTP" "OK" || \
    check "Gotify HTTP" "UNREACHABLE"

# n8n accessible
curl -s --max-time 5 http://localhost:5678/healthz > /dev/null 2>&1 && \
    check "n8n HTTP" "OK" || \
    check "n8n HTTP" "UNREACHABLE"

echo ""
echo "[ FILES ]"

# All scripts exist
for script in config.py db_manager.py ip_rotation.py \
    fingerprint_manager.py human_behavior.py \
    session_limit_check.py session_health.py \
    main_scraper.py post_processor.py \
    session_recovery.py detection_handler.py \
    notifier.py session_teardown.py \
    daily_maintenance.py self_heal.py dashboard.py; do
    [ -f "/scripts/linkedin_intel/$script" ] && \
        check "Script: $script" "OK" || \
        check "Script: $script" "MISSING"
done

# Database exists
[ -f "/data/linkedin_intel/leads.db" ] && \
    check "SQLite database" "OK" || \
    check "SQLite database" "MISSING"

# Cookie file exists
[ -f "/data/linkedin_intel/cookies/linkedin_session.pkl" ] && \
    check "LinkedIn session cookies" "OK" || \
    check "LinkedIn session cookies" "MISSING (login required)"

# State files exist
for statefile in system_freeze.json query_rotation.json profile_rotation.json; do
    [ -f "/data/linkedin_intel/state/$statefile" ] && \
        check "State: $statefile" "OK" || \
        check "State: $statefile" "MISSING"
done

echo ""
echo "[ PYTHON ENVIRONMENT ]"

PYTHON=/opt/linkedin_intel_env/bin/python3

# Python imports
$PYTHON -c "import undetected_chromedriver" 2>/dev/null && \
    check "undetected-chromedriver" "OK" || \
    check "undetected-chromedriver" "IMPORT FAILED"

$PYTHON -c "import spacy; spacy.load('en_core_web_sm')" 2>/dev/null && \
    check "spaCy en_core_web_sm" "OK" || \
    check "spaCy en_core_web_sm" "IMPORT FAILED"

$PYTHON -c "import stem" 2>/dev/null && \
    check "stem (Tor)" "OK" || \
    check "stem (Tor)" "IMPORT FAILED"

$PYTHON -c "import bs4" 2>/dev/null && \
    check "BeautifulSoup4" "OK" || \
    check "BeautifulSoup4" "IMPORT FAILED"

$PYTHON -c "import streamlit" 2>/dev/null && \
    check "Streamlit" "OK" || \
    check "Streamlit" "IMPORT FAILED"

echo ""
echo "[ DATABASE ]"

# Tables exist
TABLES=$(sqlite3 /data/linkedin_intel/leads.db \
    "SELECT COUNT(*) FROM sqlite_master WHERE type='table'" 2>/dev/null)
[ "$TABLES" -ge "5" ] 2>/dev/null && \
    check "DB tables ($TABLES tables)" "OK" || \
    check "DB tables" "MISSING (run db init)"

echo ""
echo "=========================================="
echo "  Results: $PASS passed, $FAIL failed"
echo "=========================================="

if [ $FAIL -eq 0 ]; then
    echo "  🎉 System ready for activation!"
else
    echo "  ⚠️  Fix $FAIL issues before activating"
fi
VERIFY_EOF

chmod +x /scripts/linkedin_intel/verify_system.sh
bash /scripts/linkedin_intel/verify_system.sh
```

---

## PHASE 14: ACTIVATE THE SYSTEM (Staged Approach)

### Activation Schedule

```
DAY 1 (TODAY):
  ✅ Freeze Guard workflow — ACTIVATE NOW
     Monitors system 24/7 immediately
     Low risk: only watches, doesn't scrape

DAY 2-3: Manual testing phase
  Run main_scraper.py manually 2-3 times
  Verify leads appear in dashboard
  Verify Gotify notifications arrive
  Check SQLite has real data

DAY 4-5: Supervised automation
  ✅ Daily Maintenance workflow — ACTIVATE
  ✅ Error Handler workflow — ACTIVATE
  Leave Master Orchestrator OFF
  Watch system logs daily

DAY 7+: Full automation (only if days 1-6 clean)
  ✅ Master Orchestrator — ACTIVATE
  Start with REDUCED schedule (edit cron to 1x/day only)
  Watch for 1 week
  Increase to full schedule if no detection events
```

### Step 14.1 — Activate Freeze Guard (Now)

```
In n8n UI:
1. Open "Freeze Guard" workflow
2. Click the toggle in top right: OFF → ON
3. Verify: workflow shows "Active" badge
4. Wait 5 minutes
5. Check logs: cat /data/linkedin_intel/logs/system.log
   Should see health check entries appearing
```

### Step 14.2 — Activate Remaining Workflows (Day 7)

```
In n8n UI, activate in this order:
1. Error Handler (activate, keep as reference)
2. Daily Maintenance (activate - runs at 2AM)
3. Master Orchestrator (activate LAST - this starts scraping)

Before activating Master Orchestrator:
  - Verify you have valid LinkedIn session cookies
  - Verify Tor is responding
  - Verify at least one manual scrape succeeded
  - Verify you're getting Gotify notifications
  - Verify leads appear in Streamlit dashboard
```

---

## PHASE 15: DAILY OPERATING PROCEDURES

### Your Daily 5-Minute Routine

```
MORNING (2 minutes):
  1. Open dashboard: http://YOUR_IP:8501
  2. Check "New Today" metric
  3. Review any HIGH PRIORITY notifications in Gotify
  4. Scan new leads sorted by score

LEAD REVIEW (3 minutes per lead):
  1. Click lead in dashboard
  2. Read full post text (was it real?)
  3. Check their LinkedIn profile manually
  4. If pursuing: copy email draft, personalize it
  5. Send from YOUR email client (NOT the system)
  6. Click "Mark Contacted" in dashboard

WEEKLY CHECK (10 minutes):
  1. Review session logs in dashboard
  2. Check for any detection events
  3. Verify backup files exist: ls /data/linkedin_intel/backups/
  4. Review false positive rate (rejected leads)
  5. Tune MIN_RELEVANCE_SCORE in config.py if needed
```

### Quick Reference Commands

```bash
# Check system status
bash /scripts/linkedin_intel/verify_system.sh

# View live logs
tail -f /data/linkedin_intel/logs/system.log

# View error logs
tail -f /data/linkedin_intel/logs/errors.log

# View session logs
tail -f /data/linkedin_intel/logs/sessions.log

# Manual unfreeze (if frozen accidentally)
echo '{"frozen": false}' > /data/linkedin_intel/state/system_freeze.json

# Manual freeze (e.g., before travel)
python3 /scripts/linkedin_intel/detection_handler.py \
    manual_freeze MANUAL_FREEZE_24H

# Check Tor IP
curl --socks5 127.0.0.1:9050 \
     --socks5-hostname 127.0.0.1 https://api.ipify.org

# Rotate Tor IP manually
source /scripts/linkedin_intel/activate_env.sh
python3 /scripts/linkedin_intel/ip_rotation.py rotate

# Run scraper manually (supervised)
source /scripts/linkedin_intel/activate_env.sh
SESSION_ID="sess_$(date +%s)_manual"
python3 /scripts/linkedin_intel/main_scraper.py $SESSION_ID
python3 /scripts/linkedin_intel/post_processor.py $SESSION_ID

# Check database stats
sqlite3 /data/linkedin_intel/leads.db \
    "SELECT status, COUNT(*) as count FROM leads GROUP BY status;"

# Restart everything after system reboot
sudo systemctl restart tor
docker restart gotify n8n
sudo systemctl restart linkedin-intel-dashboard

# View n8n logs
docker logs n8n --tail 50 -f

# View Gotify logs
docker logs gotify --tail 20
```

---

## FINAL WIRING DIAGRAM (Complete System)

```
YOUR MACHINE
═══════════════════════════════════════════════════════════════════
│
│  ┌─────────────────────────────────────────────────────────┐
│  │  n8n (Docker :5678) — THE NERVOUS SYSTEM               │
│  │                                                         │
│  │  Workflow 3: Freeze Guard (every 5 min, ALWAYS ON)     │
│  │    └─► Watches state/system_freeze.json                │
│  │    └─► Checks: Tor UP? Gotify UP? DB exists?           │
│  │    └─► Calls: self_heal.py if component down           │
│  │    └─► Sends: Gotify alert if problem                  │
│  │                                                         │
│  │  Workflow 4: Daily Maintenance (2AM daily)             │
│  │    └─► Calls: daily_maintenance.py                     │
│  │    └─► Creates: SQLite backup in /backups/             │
│  │    └─► Sends: Daily digest via Gotify                  │
│  │                                                         │
│  │  Workflow 2: Error Handler (triggered by errors)       │
│  │    └─► Classifies: severity level                      │
│  │    └─► Writes: system_freeze.json if CRITICAL          │
│  │    └─► Sends: Gotify alert with severity               │
│  │                                                         │
│  │  Workflow 1: Master Orchestrator (6x/week schedule)   │
│  │    ├─► Jitter + blackout check                         │
│  │    ├─► Calls: session_limit_check.py                   │
│  │    ├─► Calls: ip_rotation.py (Tor circuit)             │
│  │    ├─► Calls: session_health.py                        │
│  │    ├─► Calls: main_scraper.py ──────────────────────┐  │
│  │    ├─► Calls: post_processor.py ─────────────────┐  │  │
│  │    ├─► Calls: notifier.py (high priority only)   │  │  │
│  │    ├─► Calls: session_teardown.py                │  │  │
│  │    └─► Calls: detection_handler.py (if needed)   │  │  │
│  └────────────────────────────────────────────────────┼──┼──┘
│                                                        │  │
│  ┌─────────────────────────────────────────────────┐  │  │
│  │  Python Scripts — THE MUSCLES                   │  │  │
│  │  /scripts/linkedin_intel/                       │  │  │
│  │                                                 │  │  │
│  │  main_scraper.py ◄──────────────────────────────┼──┘  │
│  │    ├─► Imports: fingerprint_manager.py          │     │
│  │    ├─► Imports: human_behavior.py               │     │
│  │    ├─► Controls: Chrome via UCD                 │     │
│  │    ├─► Routes traffic through: Tor              │     │
│  │    ├─► Extracts: post HTML via BeautifulSoup    │     │
│  │    └─► Writes: raw_posts/{session}_raw.json     │     │
│  │                                                 │     │
│  │  post_processor.py ◄────────────────────────────┼─────┘
│  │    ├─► Reads: raw_posts/{session}_raw.json      │
│  │    ├─► Enriches with: spaCy NLP                 │
│  │    ├─► Scores with: config.YOUR_TECH_STACK      │
│  │    └─► Writes: leads.db via db_manager.py       │
│  └─────────────────────────────────────────────────┘
│                                │
│  ┌─────────────────────────────▼───────────────────┐
│  │  SQLite (/data/linkedin_intel/leads.db)          │
│  │  THE MEMORY                                      │
│  │                                                  │
│  │  Tables: leads, sessions, email_dedup,           │
│  │          detection_events, query_rotation        │
│  └─────────────────────────────┬────────────────────┘
│                                │
│  ┌─────────────────────────────▼───────────────────┐
│  │  Streamlit Dashboard (:8501) — YOUR INTERFACE   │
│  │                                                  │
│  │  Reads: leads.db (read-only display)             │
│  │  Shows: leads by score, status controls          │
│  │  Provides: email draft templates                 │
│  │  Never: auto-sends anything                     │
│  └─────────────────────────────────────────────────┘
│
│  ┌────────────────────────────────────────────────┐
│  │  Gotify (Docker :8080) — THE VOICE             │
│  │                                                │
│  │  Receives alerts from: n8n HTTP Request nodes  │
│  │  Receives alerts from: notifier.py direct      │
│  │  Pushes to: your phone app                     │
│  │  Alert types:                                  │
│  │    ⭐ High-priority lead found                  │
│  │    ⚠️  Detection event                          │
│  │    🚨 Hard signal / freeze                     │
│  │    📊 Daily digest                             │
│  │    ✅ System unfrozen                          │
│  └────────────────────────────────────────────────┘
│
│  ┌────────────────────────────────────────────────┐
│  │  Tor (:9050/:9051) — THE ANONYMIZER           │
│  │                                                │
│  │  All browser traffic routes through Tor       │
│  │  Circuit rotation before each session         │
│  │  Exit nodes: US, GB, DE, CA, AU only         │
│  └────────────────────────────────────────────────┘
│
│  ┌────────────────────────────────────────────────┐
│  │  Chrome (Real browser, not headless)          │
│  │  Controlled by: undetected-chromedriver       │
│  │  Fingerprint: injected per session            │
│  │  Session: cookies persisted across runs       │
│  └────────────────────────────────────────────────┘
═══════════════════════════════════════════════════════════════════
                             ↓
                      LINKEDIN.COM
                  (sees a human browsing)
                             ↓
                    REAL RECRUITER POSTS
                    WITH EMAIL ADDRESSES
                             ↓
                   YOUR STREAMLIT DASHBOARD
                             ↓
                      YOU READ THE LEAD
                             ↓
                   YOU WRITE THE EMAIL
                             ↓
                   YOU SEND IT YOURSELF
```

---

## THE ONE-PAGE CHEAT SHEET

```
BRING SYSTEM UP (after reboot):
  sudo systemctl start tor
  docker start gotify n8n
  sudo systemctl start linkedin-intel-dashboard

CHECK EVERYTHING:
  bash /scripts/linkedin_intel/verify_system.sh

MANUAL TEST RUN:
  source /scripts/linkedin_intel/activate_env.sh
  SESSION=sess_$(date +%s)_test
  python3 /scripts/linkedin_intel/main_scraper.py $SESSION
  python3 /scripts/linkedin_intel/post_processor.py $SESSION

VIEW YOUR LEADS:
  http://localhost:8501

VIEW n8n WORKFLOWS:
  http://localhost:5678

UNFREEZE SYSTEM:
  echo '{"frozen": false}' > /data/linkedin_intel/state/system_freeze.json

VIEW LIVE ERRORS:
  tail -f /data/linkedin_intel/logs/errors.log

WHEN YOU FIND A LEAD YOU LIKE:
  1. Read the original post (click link in dashboard)
  2. Read their LinkedIn profile manually
  3. Personalize the email template shown in dashboard
  4. Open YOUR email client
  5. Send the email yourself
  6. Mark as CONTACTED in dashboard
  7. Wait — don't follow up for minimum 5 business days
```