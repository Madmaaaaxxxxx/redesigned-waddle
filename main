#!/bin/bash
# Quick Start Setup for hutrade.pro VPN
# This script will guide you through the complete setup

set -e  # Exit on error

echo "======================================"
echo "  hutrade.pro VPN Quick Setup"
echo "======================================"
echo ""

# Color codes
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Function to print colored messages
print_success() {
    echo -e "${GREEN}✓ $1${NC}"
}

print_error() {
    echo -e "${RED}✗ $1${NC}"
}

print_warning() {
    echo -e "${YELLOW}⚠ $1${NC}"
}

# Check if running as root
if [[ $EUID -eq 0 ]]; then
   print_error "This script should NOT be run as root. Please run as a normal user with sudo privileges."
   exit 1
fi

# Check system
echo "Checking system requirements..."
if ! command -v apt &> /dev/null; then
    print_error "This script is designed for Ubuntu/Debian systems"
    exit 1
fi
print_success "System check passed"

# Update system
echo ""
echo "Step 1: Updating system packages..."
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git nano ufw net-tools
print_success "System updated"

# Configure firewall
echo ""
echo "Step 2: Configuring firewall..."
read -p "Configure UFW firewall? (recommended) [Y/n]: " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]] || [[ -z $REPLY ]]; then
    sudo ufw --force enable
    sudo ufw allow 22/tcp comment 'SSH'
    sudo ufw allow 80/tcp comment 'HTTP'
    sudo ufw allow 443/tcp comment 'HTTPS'
    sudo ufw allow 51820/udp comment 'WireGuard VPN'
    sudo ufw allow 51821/tcp comment 'WG-Easy Web UI'
    print_success "Firewall configured"
else
    print_warning "Skipping firewall configuration"
fi

# Get server IP
SERVER_IP=$(curl -s ifconfig.me)
print_success "Server IP detected: $SERVER_IP"

# Choose VPN solution
echo ""
echo "======================================"
echo "Step 3: Choose VPN Solution"
echo "======================================"
echo "1. WireGuard with WG-Easy (Recommended)"
echo "   ✓ Modern, fast, secure"
echo "   ✓ Beautiful web UI with QR codes"
echo "   ✓ Easy client management"
echo ""
echo "2. Pritunl (Enterprise-grade)"
echo "   ✓ Advanced analytics"
echo "   ✓ User management"
echo "   ✓ Multi-server support"
echo ""
echo "3. Outline VPN (Simplest)"
echo "   ✓ Easiest to use"
echo "   ✓ Resistant to blocking"
echo "   ✓ Simple key sharing"
echo ""
read -p "Enter choice (1-3): " vpn_choice

# Install Docker
echo ""
echo "Installing Docker..."
if ! command -v docker &> /dev/null; then
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh
    sudo usermod -aG docker $USER
    rm get-docker.sh
    print_success "Docker installed"
else
    print_success "Docker already installed"
fi

# Install Docker Compose
if ! command -v docker-compose &> /dev/null; then
    echo "Installing Docker Compose..."
    sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
    print_success "Docker Compose installed"
else
    print_success "Docker Compose already installed"
fi

# Install chosen VPN
case $vpn_choice in
    1)
        echo ""
        echo "======================================"
        echo "Installing WireGuard with WG-Easy"
        echo "======================================"
        
        # Set password
        read -s -p "Enter admin password for WG-Easy: " WG_PASSWORD
        echo ""
        read -s -p "Confirm password: " WG_PASSWORD2
        echo ""
        
        if [ "$WG_PASSWORD" != "$WG_PASSWORD2" ]; then
            print_error "Passwords don't match!"
            exit 1
        fi
        
        # Create directory
        mkdir -p ~/wg-easy
        cd ~/wg-easy
        
        # Create docker-compose.yml
        cat > docker-compose.yml <<EOF
version: "3.8"
services:
  wg-easy:
    environment:
      - WG_HOST=$SERVER_IP
      - PASSWORD=$WG_PASSWORD
      - WG_PORT=51820
      - WG_DEFAULT_ADDRESS=10.8.0.x
      - WG_DEFAULT_DNS=1.1.1.1
      - WG_ALLOWED_IPS=0.0.0.0/0
      - WG_PERSISTENT_KEEPALIVE=25
      - WG_MTU=1420
    image: weejewel/wg-easy
    container_name: wg-easy
    volumes:
      - ~/.wg-easy:/etc/wireguard
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
EOF
        
        # Start service
        sudo docker-compose up -d
        
        sleep 5
        
        echo ""
        print_success "WireGuard with WG-Easy installed successfully!"
        echo ""
        echo "======================================"
        echo "Access Information:"
        echo "======================================"
        echo "Web UI: http://$SERVER_IP:51821"
        echo "Or: https://panel.hutrade.pro (after nginx setup)"
        echo "Username: admin"
        echo "Password: [the password you set]"
        echo ""
        echo "WireGuard Port: 51820/udp"
        echo "Web UI Port: 51821/tcp"
        echo "======================================"
        ;;
        
    2)
        echo ""
        echo "======================================"
        echo "Installing Pritunl"
        echo "======================================"
        
        # Add repository
        sudo tee /etc/apt/sources.list.d/pritunl.list << EOF
deb https://repo.pritunl.com/stable/apt jammy main
EOF
        
        sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com --recv 7568D9BB55FF9E5287D586017AE645C0CF8E292A
        
        # Install MongoDB
        wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | sudo apt-key add -
        echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
        
        sudo apt update
        sudo apt install -y pritunl mongodb-org
        
        # Start services
        sudo systemctl start mongod pritunl
        sudo systemctl enable mongod pritunl
        
        sleep 5
        
        # Get credentials
        SETUP_KEY=$(sudo pritunl setup-key)
        DEFAULT_PASS=$(sudo pritunl default-password)
        
        echo ""
        print_success "Pritunl installed successfully!"
        echo ""
        echo "======================================"
        echo "Access Information:"
        echo "======================================"
        echo "Web UI: https://$SERVER_IP"
        echo "Or: https://panel.hutrade.pro (after nginx setup)"
        echo ""
        echo "Setup Key: $SETUP_KEY"
        echo ""
        echo "Default Login:"
        echo "Username: pritunl"
        echo "Password: $DEFAULT_PASS"
        echo ""
        echo "⚠️  SAVE THESE CREDENTIALS!"
        echo "======================================"
        
        # Save to file
        cat > ~/pritunl-credentials.txt <<EOF
Pritunl Access Information
==========================
Web UI: https://$SERVER_IP
Setup Key: $SETUP_KEY
Username: pritunl
Password: $DEFAULT_PASS
EOF
        print_success "Credentials saved to ~/pritunl-credentials.txt"
        ;;
        
    3)
        echo ""
        echo "======================================"
        echo "Installing Outline VPN"
        echo "======================================"
        
        # Install Outline
        sudo bash -c "$(wget -qO- https://raw.githubusercontent.com/Jigsaw-Code/outline-server/master/src/server_manager/install_scripts/install_server.sh)" | tee ~/outline-install.log
        
        print_success "Outline installed successfully!"
        echo ""
        echo "⚠️  IMPORTANT: The API URL and Management Key are in ~/outline-install.log"
        echo ""
        echo "Next steps:"
        echo "1. Download Outline Manager: https://getoutline.org/get-started/"
        echo "2. Open the log file: cat ~/outline-install.log"
        echo "3. Copy the API URL"
        echo "4. Paste it in Outline Manager"
        ;;
        
    *)
        print_error "Invalid choice"
        exit 1
        ;;
esac

# Install Nginx
echo ""
echo "======================================"
echo "Step 4: Installing Nginx"
echo "======================================"
read -p "Install and configure Nginx? [Y/n]: " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]] || [[ -z $REPLY ]]; then
    sudo apt install -y nginx
    sudo systemctl enable nginx
    print_success "Nginx installed"
    
    print_warning "You'll need to:"
    echo "1. Configure SSL certificates (Cloudflare or Let's Encrypt)"
    echo "2. Create nginx configuration for hutrade.pro"
    echo "3. Enable the site and restart nginx"
fi

# Final steps
echo ""
echo "======================================"
echo "✓ Installation Complete!"
echo "======================================"
echo ""
echo "Next Steps:"
echo ""
echo "1. Domain Configuration:"
echo "   - Update nameservers at Hostinger to Cloudflare"
echo "   - Add DNS records in Cloudflare for hutrade.pro"
echo "   - Wait for DNS propagation (24-48 hours)"
echo ""
echo "2. SSL Certificate:"
echo "   - Get Cloudflare Origin Certificate OR"
echo "   - Install Let's Encrypt certificate"
echo ""
echo "3. Nginx Configuration:"
echo "   - Create /etc/nginx/sites-available/hutrade.pro"
echo "   - Enable the site"
echo "   - Test and restart nginx"
echo ""
echo "4. Access VPN Panel:"
echo "   - Temporarily: http://$SERVER_IP:51821 (WG-Easy)"
echo "   - After setup: https://panel.hutrade.pro"
echo ""
echo "5. Security:"
echo "   - Change all default passwords"
echo "   - Configure firewall rules"
echo "   - Enable fail2ban"
echo ""

if [ $vpn_choice -eq 1 ]; then
    echo "WireGuard Web UI: http://$SERVER_IP:51821"
elif [ $vpn_choice -eq 2 ]; then
    echo "Pritunl credentials: ~/pritunl-credentials.txt"
elif [ $vpn_choice -eq 3 ]; then
    echo "Outline setup info: ~/outline-install.log"
fi

echo ""
print_success "Setup script completed!"
echo ""
