# GIRRewrite Docker Quick Setup Guide

This quick reference guide provides the essential commands and steps to get your GIRRewrite Discord bot up and running with Docker. For more detailed explanations, refer to the [basic setup guide](./README.md) and [advanced setup guide](./ADVANCED_SETUP.md).

## 1. Clone the Repository

```bash
git clone https://github.com/DiscordGIR/GIRRewrite.git
cd GIRRewrite
```

## 2. Configure Environment Variables

```bash
# Copy the example env file
cp .env.example .env

# Edit the .env file with your values
nano .env
```

Essential values to configure:
```
GIR_TOKEN="your_discord_bot_token_here"
MAIN_GUILD_ID=123456789012345678
OWNER_ID=123456789012345678
AARON_ID=123456789012345678
DB_HOST="mongo"  # Important: must be "mongo" for Docker setup
DB_PORT=27017
```

## 3. Create docker-compose.yml

Replace the existing `docker-compose.yml` file with:

```yaml
version: '3.8'

services:
  gir:
    container_name: gir
    build:
      context: .
      dockerfile: Dockerfile.production
    restart: always
    depends_on:
      - mongo
    networks:
      - gir-network
    environment:
      - TZ=UTC

  mongo:
    image: mongo:latest
    container_name: gir-mongo
    restart: unless-stopped
    environment:
      - MONGO_DATA_DIR=/data/db
      - MONGO_LOG_DIR=/dev/null
    volumes:
      - ./mongo_data:/data/db
    networks:
      - gir-network
    ports:
      - "27017:27017"

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

networks:
  gir-network:
    driver: bridge
```

## 4. Build and Start the Containers

```bash
# Build and start containers
docker-compose up -d --build
```

## 5. Initialize the Database with setup.py

```bash
# Run the setup script
docker exec -it gir python3 setup.py
```

## 6. Sync Slash Commands

Send the bot a direct message with:
```
!sync
```

You only need to do this once after setup or after changing command data.

## 7. Configure Bot Permissions

After the bot is running, set up permission levels:

```
/role add level:1 role:@Moderator
/role add level:2 role:@Admin
/role add level:3 role:@Owner
```

## 8. Common Management Commands

### View logs
```bash
docker-compose logs -f gir
```

### Restart the bot
```bash
docker-compose restart gir
```

### Update the bot
```bash
git pull
docker-compose up -d --build --force-recreate gir
```

### Create a database backup
```bash
docker exec -it gir-mongo mongodump --out /data/db/backup
docker cp gir-mongo:/data/db/backup ./backup
```

### Restore from a backup
```bash
docker cp ./backup gir-mongo:/data/db/
docker exec -it gir-mongo mongorestore /data/db/backup
```

### Stop all containers
```bash
docker-compose down
```

## 9. Accessing MongoDB Admin Interface

Open your browser and navigate to:
```
http://localhost:8081
```

## 10. Security Checklist

- [ ] Set strong passwords for MongoDB if exposed
- [ ] Configure a firewall to restrict ports
- [ ] Set up regular database backups
- [ ] Keep the bot code updated
- [ ] Review logs regularly for issues

## Troubleshooting

### Bot not starting
Check logs: `docker-compose logs gir`

### Database connection issues
Check MongoDB is running: `docker-compose ps`

### Commands not working
Verify slash commands are synced by sending `!sync` to the bot in DMs

For more detailed troubleshooting and advanced configuration options, refer to the [Advanced Setup Guide](./ADVANCED_SETUP.md).
