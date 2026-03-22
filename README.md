# Stock Watchlist App

A real-time stock tracking application built with React and Node.js. Search for stocks, add them to your watchlist, and see live price updates via WebSocket.

## Features:

- **Real-time Price Updates**: WebSocket integration with Finnhub API for live stock prices
- **Stock Search**: Search stocks by symbol or company name
- **Persistent Watchlist**: Session-based watchlist stored in MongoDB
- **Live Data**: See price changes in real-time with visual indicators
- **Responsive Design**: Beautiful gradient UI that works on all devices

## Tech Stack

**Frontend:** React 18, Vite, WebSocket client, CSS3

**Backend:** Node.js, Express, WebSocket (ws), MongoDB/Mongoose, Finnhub API, Axios

---

# 1. Local Development

## Prerequisites

- Node.js (v18 or higher)
- MongoDB Atlas account (or local MongoDB)
- Finnhub API key

## Installation

```bash
# Install all dependencies from root
npm install
npm --prefix client install
npm --prefix server install
```

## Environment Setup

Create `server/.env`:

```env
MONGODB_URI=your_mongodb_connection_string
FINNHUB_API_KEY=your_finnhub_api_key
PORT=3001
WS_PORT=3002
```

## Run Locally

```bash
# Start both client and server
npm run dev
```

This starts:
- Client: `http://localhost:5173`
- API: `http://localhost:3001`
- WebSocket: `ws://localhost:3002`

## Usage

1. **Search for stocks** - Type a company name or symbol (e.g., "AAPL" or "Apple")
2. **Add to watchlist** - Click the "Add" button on any stock
3. **Watch live updates** - Prices update in real-time when market is open
4. **Remove stocks** - Click "Remove" to delete from watchlist

---

# 2. Deployment to AWS EC2

## Prerequisites

- AWS account with EC2 access
- GitHub repository with this code
- MongoDB Atlas with IP whitelist configured

## Step 1: Launch EC2 Instance

1. Go to **AWS Console** → **EC2** → **Launch Instance**
2. Configure:

   | Setting | Value |
   |---------|-------|
   | **Name** | `stock-watchlist` |
   | **AMI** | Ubuntu Server 22.04 LTS |
   | **Instance type** | `t2.micro` (free tier) or `t2.small` |
   | **Key pair** | Create new → `stock-watchlist-key` → Download `.pem` |

3. **Security Group** inbound rules:

   | Type | Port | Source |
   |------|------|--------|
   | SSH | 22 | Your IP |
   | HTTP | 80 | 0.0.0.0/0 |
   | HTTPS | 443 | 0.0.0.0/0 |

4. Launch and wait for instance to start

## Step 2: Connect to EC2

```bash
# Copy key to safe location (WSL users)
cp /mnt/c/Users/<WINDOWS_USER>/Downloads/stock-watchlist-key.pem ~/
chmod 400 ~/stock-watchlist-key.pem

# Connect
ssh -i ~/stock-watchlist-key.pem ubuntu@<EC2_PUBLIC_IP>
```

## Step 3: Install Software on EC2

Run on EC2:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install PM2 (process manager)
sudo npm install -g pm2

# Install Nginx
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx

# Verify
node -v && pm2 -v && nginx -v
```

## Step 4: Configure Nginx

Run on EC2:

```bash
# Create app directory
sudo mkdir -p /var/www/stock-watchlist
sudo chown ubuntu:ubuntu /var/www/stock-watchlist

# Create Nginx config
sudo nano /etc/nginx/sites-available/stock-watchlist
```

Paste this config:

```nginx
server {
    listen 80;
    server_name _;

    # Serve React client (static files)
    location / {
        root /var/www/stock-watchlist/client;
        try_files $uri $uri/ /index.html;
    }

    # Proxy API requests to Node.js server
    location /api {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Proxy WebSocket connections
    location /ws {
        proxy_pass http://localhost:3002;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

```bash
# Enable config
sudo ln -s /etc/nginx/sites-available/stock-watchlist /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

## Step 5: GitHub Secrets Setup

Go to **GitHub repo** → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| Secret Name | Value |
|-------------|-------|
| `EC2_HOST` | Your EC2 public IP (or Elastic IP) |
| `EC2_USER` | `ubuntu` |
| `EC2_SSH_KEY` | Contents of your `.pem` file |
| `MONGODB_URI` | Your MongoDB connection string |
| `FINNHUB_API_KEY` | Your Finnhub API key |

## Step 6: MongoDB Atlas IP Whitelist

1. Go to **MongoDB Atlas** → **Network Access**
2. Click **Add IP Address**
3. Add your EC2 public IP or `0.0.0.0/0` (allow all)

## Step 7: Deploy

Push to `main` branch to trigger automatic deployment:

```bash
git add .
git commit -m "Deploy changes"
git push origin main
```

Monitor progress in **GitHub repo** → **Actions** tab.

## Step 8: Verify Deployment

1. Visit `http://<EC2_PUBLIC_IP>` in your browser
2. Check PM2 status on EC2: `pm2 status`
3. View server logs: `pm2 logs stock-server`

## Troubleshooting

**Check Nginx:**
```bash
sudo nginx -t
sudo systemctl status nginx
sudo tail -f /var/log/nginx/error.log
```

**Check PM2:**
```bash
pm2 status
pm2 logs stock-server
```

**Restart services:**
```bash
pm2 restart stock-server
sudo systemctl restart nginx
```

---

## Architecture

```
Finnhub WS → Node.js Backend → WebSocket Server → React Client
                ↓
            MongoDB
```

## API Endpoints

- `GET /api/search?q=AAPL` - Search for stocks
- `GET /api/watchlist/:sessionId` - Get user's watchlist
- `POST /api/watchlist` - Add stock to watchlist
- `DELETE /api/watchlist/:sessionId/:symbol` - Remove stock from watchlist

## WebSocket Messages

**Client → Server:**
- `{ type: 'init', sessionId: '...' }` - Initialize connection
- `{ type: 'subscribe', symbol: 'AAPL' }` - Subscribe to stock
- `{ type: 'unsubscribe', symbol: 'AAPL' }` - Unsubscribe from stock

**Server → Client:**
- `{ type: 'price-update', data: { symbol, price, timestamp } }` - Real-time price update
