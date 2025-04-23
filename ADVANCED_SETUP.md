# GIRRewrite Advanced Docker Setup Guide

This advanced guide provides detailed instructions for setting up the [GIRRewrite Discord bot](https://github.com/DiscordGIR/GIRRewrite) using Docker with a focus on proper configuration, customization, and advanced operations. This guide assumes you've already completed the [basic setup](./README.md) or have some familiarity with Docker.

## Prerequisites

- Docker and Docker Compose already installed
- A Discord Bot created on the [Discord Developer Portal](https://discord.com/developers/applications)
- Basic knowledge of command line operations
- Git installed on your system

## Advanced Configuration

### Custom Environment Variables Setup

The `.env` file supports more than the basic configuration shown in the basic guide. Here's a more comprehensive setup:

```bash
# Essential Bot Configuration
GIR_TOKEN="your_discord_bot_token_here"
MAIN_GUILD_ID=123456789012345678  # Your main Discord server ID
OWNER_ID=123456789012345678       # Your Discord user ID (bot owner)
AARON_ID=123456789012345678       # Server owner ID (can be same as OWNER_ID)

# Database Configuration
DB_HOST="mongo"  # Use "mongo" for dockerized MongoDB
DB_PORT=27017

# Optional: Development Mode (disables mod pings)
DEV=True

# Optional: Ban Appeals System
BAN_APPEAL_URL="https://forms.your-site.com/ban-appeals"
BAN_APPEAL_GUILD_ID=876543210987654321  # ID of your ban appeals server
BAN_APPEAL_MOD_ROLE=234567890123456789  # Role ID of mods in ban appeals server

# Optional: Webhook Logging
LOGGING_WEBHOOK_URL="https://discord.com/api/webhooks/your-webhook-url"

# Optional: External Services
PASTEE_TOKEN="your_pastee_api_key"  # For auto-uploading tweak lists
RESNEXT_TOKEN="your_token_here"     # For neural_net meme command

# Optional: Feature Flags
ENABLE_MARKOV=True  # Enable /memegen text and /memegen aipfp commands
```

### Advanced Docker Compose Configuration

Create a more robust `docker-compose.yml` file with better networking, volume management, and restart policies:

```yaml
version: '3.8'

services:
  gir:
    container_name: gir
    build:
      context: .
      dockerfile: Dockerfile.production
    restart: unless-stopped
    depends_on:
      - mongo
    networks:
      - gir-network
    volumes:
      - ./logs:/usr/src/app/logs
      - ./data:/usr/src/app/data
    environment:
      - TZ=UTC  # Set your preferred timezone
    deploy:
      resources:
        limits:
          memory: 1G  # Limit memory usage
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "3"

  mongo:
    image: mongo:latest
    container_name: gir-mongo
    restart: unless-stopped
    environment:
      - MONGO_DATA_DIR=/data/db
      - MONGO_LOG_DIR=/dev/null
      # Optional: Uncomment for authentication
      # - MONGO_INITDB_ROOT_USERNAME=gir_admin
      # - MONGO_INITDB_ROOT_PASSWORD=secure_password_here
    volumes:
      - ./mongo_data:/data/db
      # Optional: Configure MongoDB with custom config file
      # - ./mongo-config:/etc/mongo
    networks:
      - gir-network
    ports:
      - "27017:27017"  # Optional: Only needed if you want to connect from host
    command: mongod --wiredTigerCacheSizeGB 0.5  # Limit cache size for smaller servers

  # MongoDB admin interface (optional but recommended)
  mongo-express:
    image: mongo-express:latest
    container_name: gir-mongo-express
    restart: unless-stopped
    depends_on:
      - mongo
    networks:
      - gir-network
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_URL: mongodb://mongo:27017/
      # If using authentication:
      # ME_CONFIG_MONGODB_URL: mongodb://gir_admin:secure_password_here@mongo:27017/
      ME_CONFIG_BASICAUTH_USERNAME: admin
      ME_CONFIG_BASICAUTH_PASSWORD: secure_admin_password_here
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "1"

networks:
  gir-network:
    driver: bridge
```

## Advanced Setup Procedures

### Custom Dockerfile

If you need to extend the base Dockerfile for your specific deployment, create a custom `Dockerfile.custom` with additional dependencies or configurations:

```dockerfile
# Start from the production Dockerfile
FROM python:3.10.2-slim-bullseye 
WORKDIR /usr/src/app

ENV VIRTUAL_ENV=/opt/venv
RUN python3 -m venv $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

# System dependencies
RUN apt-get update && apt-get upgrade -y
RUN apt install -y git gcc python3-dev fonts-noto-color-emoji ffmpeg

# Python dependencies
COPY ./requirements.txt .
RUN pip install --upgrade pip setuptools wheel
RUN pip install -r requirements.txt

# Add custom packages if needed
RUN pip install psutil pillow

# Copy the application code
COPY . .

# Pre-compile Python files for faster startup
RUN python -m compileall .

# Set up proper timezone (adjust as needed)
ENV TZ=UTC
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# Clean up to reduce image size
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Start the bot
CMD ["python", "-u", "main.py"]
```

### MongoDB Authentication Setup

For better security in production environments, set up MongoDB with authentication:

1. Update your docker-compose.yml file to include authentication variables (as shown above)
2. Update your .env file to include authentication credentials:

```
DB_CONNECTION_STRING="mongodb://gir_admin:secure_password_here@mongo:27017/botty"
```

3. Create a MongoDB init script to set up authentication:

```bash
# Create a directory for the init script
mkdir -p ./mongo-init

# Create the initialization script
cat > ./mongo-init/init-mongo.js << 'EOF'
db = db.getSiblingDB('admin');
db.createUser({
  user: 'gir_admin',
  pwd: 'secure_password_here', // Replace with your secure password
  roles: [
    { role: 'userAdminAnyDatabase', db: 'admin' },
    { role: 'readWriteAnyDatabase', db: 'admin' }
  ]
});

db = db.getSiblingDB('botty');
db.createUser({
  user: 'gir_user',
  pwd: 'secure_bot_password_here', // Replace with a secure password
  roles: [
    { role: 'readWrite', db: 'botty' }
  ]
});
EOF

# Update docker-compose.yml to use the init script
# Add this under the mongo service:
#    volumes:
#      - ./mongo_data:/data/db
#      - ./mongo-init:/docker-entrypoint-initdb.d/
```

## Production Deployment Considerations

### Using Docker Swarm for High Availability

For mission-critical deployments, you can use Docker Swarm for increased reliability:

```bash
# Initialize swarm mode
docker swarm init

# Deploy the stack
docker stack deploy -c docker-compose.yml gir
```

### Setting Up Automated Backups

Create a backup script (`backup.sh`) to automate MongoDB backups:

```bash
#!/bin/bash
# MongoDB backup script for GIR bot

# Configuration
BACKUP_DIR="/path/to/backups"
DATETIME=$(date +"%Y%m%d_%H%M%S")
CONTAINER_NAME="gir-mongo"
BACKUP_NAME="gir_mongo_$DATETIME"

# Create backup directory if it doesn't exist
mkdir -p $BACKUP_DIR

# Create backup
echo "Creating backup $BACKUP_NAME..."
docker exec $CONTAINER_NAME mongodump --out="/data/db/backup_$DATETIME"
docker cp $CONTAINER_NAME:/data/db/backup_$DATETIME $BACKUP_DIR/$BACKUP_NAME

# Compress backup
echo "Compressing backup..."
tar -czf $BACKUP_DIR/$BACKUP_NAME.tar.gz -C $BACKUP_DIR $BACKUP_NAME
rm -rf $BACKUP_DIR/$BACKUP_NAME

# Keep only the last 7 backups
echo "Cleaning old backups..."
ls -t $BACKUP_DIR/*.tar.gz | tail -n +8 | xargs -r rm

echo "Backup completed successfully!"
```

Make the script executable and set up a cron job:

```bash
chmod +x backup.sh

# Add to crontab (run every day at 3 AM)
(crontab -l 2>/dev/null; echo "0 3 * * * /path/to/backup.sh >> /path/to/backup.log 2>&1") | crontab -
```

### Setting Up Monitoring

Install and configure Prometheus and Grafana for monitoring:

1. Create a `monitoring-compose.yml` file:

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: gir-prometheus
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
    ports:
      - "9090:9090"
    networks:
      - gir-network

  grafana:
    image: grafana/grafana:latest
    container_name: gir-grafana
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - gir-network

volumes:
  prometheus_data:
  grafana_data:

networks:
  gir-network:
    external: true
```

2. Configure Prometheus to scrape node metrics:

```bash
mkdir -p ./prometheus

cat > ./prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  
  - job_name: 'docker'
    static_configs:
      - targets: ['host.docker.internal:9323']
EOF
```

3. Start the monitoring stack:

```bash
docker-compose -f monitoring-compose.yml up -d
```

4. Access Grafana at http://your-server:3000 (default login: admin/admin)

## Advanced Bot Configuration

### Customizing the Setup Script

You can customize the `setup.py` script for your server's specific needs:

```python
import asyncio
import os

import mongoengine
from dotenv import find_dotenv, load_dotenv

from data.model.guild import Guild
from data.model.cases import Case
from data.model.tag import Tag

load_dotenv(find_dotenv())

async def setup():
    print("STARTING ADVANCED SETUP...")
    guild = Guild()

    # Basic configuration
    guild._id = int(os.environ.get("MAIN_GUILD_ID"))
    guild.case_id = 1
    
    # Custom excludes
    guild.logging_excluded_channels = [
        123456789012345678,  # Replace with actual channel IDs
        234567890123456789
    ]
    guild.filter_excluded_channels = [
        345678901234567890,  # Replace with actual channel IDs
        456789012345678901
    ]
    guild.filter_excluded_guilds = [
        567890123456789012,  # Replace with actual server IDs for allowed invites
        678901234567890123
    ]

    # NSA configuration (message mirroring)
    guild.nsa_guild_id = 789012345678901234  # Set to actual guild ID or leave as is

    # Save guild configuration
    guild.save()
    
    # Create some initial tags
    await create_default_tags(guild._id)
    
    print("SETUP COMPLETE!")

async def create_default_tags(guild_id):
    # Create some helpful default tags
    tags = [
        {"name": "rules", "content": "Please read our server rules at <#RULES_CHANNEL_ID>", "guild_id": guild_id},
        {"name": "support", "content": "For support, please provide the following information:\n- Device model:\n- iOS/iPadOS version:\n- Issue description:", "guild_id": guild_id},
        {"name": "help", "content": "Need help? Just ask your question directly instead of asking to ask. Include details about your issue so people can help you better!", "guild_id": guild_id}
    ]
    
    for tag_data in tags:
        tag = Tag(**tag_data)
        tag.save()
        print(f"Created tag: {tag_data['name']}")

if __name__ == "__main__":
    if os.environ.get("DB_CONNECTION_STRING") is None:
        mongoengine.register_connection(
            host=os.environ.get("DB_HOST"), 
            port=int(os.environ.get("DB_PORT")), 
            alias="default", 
            name="botty")
    else:
        mongoengine.register_connection(
            host=os.environ.get("DB_CONNECTION_STRING"), 
            alias="default", 
            name="botty")
    
    res = asyncio.get_event_loop().run_until_complete(setup())
```

### Configuring Role-Based Permissions

GIR uses a permission system based on Discord roles. You can configure these after the bot is running using the following commands:

1. Add roles to permission levels:
```
/role add level:1 role:@Moderator
/role add level:2 role:@Admin
/role add level:3 role:@Owner
```

Permission levels:
- Level 1: Basic moderation (warn, mute, etc.)
- Level 2: Advanced moderation (ban, unmute others' mutes)
- Level 3: Administrative (configure bot settings)

2. Configure anti-raid protection:
```
/antiraid setup min_account_age:7d cooldown:10s
```

## Troubleshooting Advanced Issues

### Database Connection Problems

If you're experiencing database connection issues:

1. Check if MongoDB is running:
```bash
docker exec -it gir-mongo mongo --eval "db.adminCommand('ping')"
```

2. Verify network connectivity:
```bash
docker exec -it gir ping mongo
```

3. Check MongoDB logs:
```bash
docker logs gir-mongo
```

4. Repair database if corrupted:
```bash
docker exec -it gir-mongo mongod --repair
```

### Bot Not Responding to Commands

1. Check if the bot is connected to Discord:
```bash
docker logs gir | grep "Logged in as"
```

2. Verify slash commands are synced:
```bash
# Connect to bot container
docker exec -it gir bash

# Run sync script manually
python3 sync_commands.py
```

3. Check for rate limiting:
```bash
docker logs gir | grep "rate limit"
```

### Memory or Performance Issues

1. Check resource usage:
```bash
docker stats gir gir-mongo
```

2. Optimize MongoDB for low-memory environments:
```bash
# Add to mongo service in docker-compose.yml
command: mongod --wiredTigerCacheSizeGB 0.25
```

3. Enable MongoDB oplog for better performance:
```bash
# Add to mongo service in docker-compose.yml
command: mongod --replSet rs0 --oplogSize 128
```

## Advanced Features and Customization

### Adding Custom Commands

GIR supports custom commands through the tags system. You can add these directly via MongoDB or through the bot:

```
/tag create name:welcome content:Welcome to our server, {user}! Please check out the rules in <#CHANNEL_ID>.
```

### Implementing Custom Cogs (Extensions)

You can extend GIR's functionality by creating custom cogs:

1. Create a new Python file in the `cogs` directory, for example `custom_features.py`:

```python
import discord
from discord.ext import commands
from discord.commands import slash_command, Option
from utils.context import Context

class CustomFeatures(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
    
    @slash_command(name="greet", description="Send a custom greeting")
    async def greet(self, ctx: Context, user: Option(discord.Member, "User to greet", required=False)):
        target = user or ctx.author
        await ctx.respond(f"Hello, {target.mention}! Welcome to our server!")

def setup(bot):
    bot.add_cog(CustomFeatures(bot))
```

2. Register your cog by adding it to `main.py`:

```python
# Add this line in the cog loading section
bot.load_extension("cogs.custom_features")
```

### Customizing Embed Colors and Appearance

To customize the bot's appearance, you can edit `data/config.json` after the container is running:

```bash
# Access the container
docker exec -it gir bash

# Edit the config file
nano data/config.json
```

Update the embed colors and other styling options according to your preferences, then restart the bot:

```bash
docker-compose restart gir
```

## Maintenance and Updates

### Updating the Bot Safely

To update GIR without data loss:

```bash
# Pull the latest changes
git pull origin main

# Backup the database first
./backup.sh

# Rebuild and restart containers
docker-compose up -d --build --force-recreate gir
```

### Database Maintenance

Regular database maintenance helps keep GIR running smoothly:

```bash
# Connect to MongoDB
docker exec -it gir-mongo mongosh

# Switch to bot database
use botty

# Run database stats
db.stats()

# Repair and compact collections
db.repairDatabase()

# Find and clean up orphaned documents
db.tags.deleteMany({ guild_id: { $exists: false } })

# Exit MongoDB shell
exit
```

## Security Hardening

### Securing MongoDB

For production environments, always enable authentication and consider these additional security measures:

1. Configure MongoDB with TLS/SSL:
```bash
# Generate certificates
mkdir -p ./mongo-certs
openssl req -newkey rsa:2048 -new -x509 -days 365 -nodes -out ./mongo-certs/mongodb-cert.crt -keyout ./mongo-certs/mongodb-cert.key

# Create MongoDB configuration file
cat > ./mongo-config/mongod.conf << 'EOF'
net:
  port: 27017
  bindIp: 0.0.0.0
  ssl:
    mode: requireSSL
    PEMKeyFile: /etc/mongo/ssl/mongodb-cert.key
    CAFile: /etc/mongo/ssl/mongodb-cert.crt
EOF

# Update your docker-compose.yml to use this config
```

2. Set up a firewall to restrict access:
```bash
# Allow only necessary ports
ufw allow ssh
ufw allow 443/tcp
ufw deny 27017/tcp
ufw deny 8081/tcp
ufw enable
```

### Securing the Bot Application

1. Run the bot as a non-root user:
```dockerfile
# Add to your Dockerfile.custom
RUN groupadd -r giruser && useradd -r -g giruser giruser
USER giruser
```

2. Use environment secrets for sensitive data:
```bash
# Create secrets file
echo "GIR_TOKEN=your_token_here" > .env.secrets

# Update docker-compose.yml to use secrets
secrets:
  girtoken:
    file: .env.secrets
```

## Conclusion

This advanced guide covers extensive configuration options, security enhancements, and troubleshooting procedures for running GIRRewrite in Docker. With these techniques, you can deploy a robust, secure, and customized Discord bot that meets your server's specific requirements.

Remember to regularly update your bot, perform database maintenance, and keep your security configurations up to date to ensure the best experience for your server members.

If you encounter issues not covered in this guide, check the [official GIRRewrite repository](https://github.com/DiscordGIR/GIRRewrite) or open an issue for assistance.
