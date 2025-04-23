# GIRRewrite Docker Setup Tutorial

This tutorial provides step-by-step instructions for setting up the [GIRRewrite Discord bot](https://github.com/DiscordGIR/GIRRewrite) with Docker, including MongoDB.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed on your system
- [Docker Compose](https://docs.docker.com/compose/install/) installed on your system
- A Discord Bot application created on the [Discord Developer Portal](https://discord.com/developers/applications)
- Git installed on your system

## Step 1: Clone the Repository

```bash
# Clone the GIRRewrite repository
git clone https://github.com/DiscordGIR/GIRRewrite.git

# Change to the repository directory
cd GIRRewrite
```

## Step 2: Set Up the Environment Variables

First, create a `.env` file based on the provided example:

```bash
# Copy the example env file
cp .env.example .env
```

Now edit the `.env` file with your preferred text editor and fill in the required values:

```
GIR_TOKEN="your_discord_bot_token_here"

MAIN_GUILD_ID=123456789012345678  # Your Discord server ID
OWNER_ID=123456789012345678       # Your Discord user ID
AARON_ID=123456789012345678       # ID of the server owner (can be same as OWNER_ID)

# We'll use "mongo" as hostname since we're using Docker
DB_HOST="mongo"
DB_PORT=27017

# Optional: Set to True if you're testing (disables mod pings)
DEV=True

# You can delete or comment out the optional sections you don't need:
# BAN_APPEAL_URL, LOGGING_WEBHOOK_URL, AARON_ROLE, PASTEE_TOKEN, etc.
```

## Step 3: Modify the docker-compose.yml File

We need to edit the `docker-compose.yml` file to enable MongoDB and fix the networking configuration:

```bash
# Create a backup of the original file
cp docker-compose.yml docker-compose.yml.bak
```

Now edit the `docker-compose.yml` file to look like this:

```yaml
version: '3.4'

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
      - TZ=UTC  # Set your preferred timezone

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
      - "27017:27017"  # Expose MongoDB port (optional, only needed if you want to connect from host)

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

networks:
  gir-network:
    driver: bridge
```

## Step 4: Create a setup.py Configuration File

Edit the `setup.py` file to configure your bot's initial settings:

```python
# Make sure the guild ID matches what you set in the .env file
# You can leave most values as they are unless you have specific requirements
```

## Step 5: Build and Start the Docker Containers

```bash
# Build and start the containers in detached mode
docker-compose up -d --build
```

## Step 6: Run the Setup Script

After the containers are running, you need to execute the setup script to initialize the database:

```bash
# Get the container name
docker container ls

# Run the setup script
docker exec -it gir python3 setup.py
```

## Step 7: Sync Slash Commands

Once the bot is running, you need to sync the slash commands with Discord. DM the bot with the following message:

```
!sync
```

This only needs to be done once when setting up the bot, or when you change any command data.

## Step 8: Accessing MongoDB Admin Interface

You can access the MongoDB admin interface at:

```
http://localhost:8081
```

This will help you manage and inspect your database without command-line tools.

## Additional Configuration and Commands

1. **Restarting the Bot**:
```bash
docker-compose restart gir
```

2. **Viewing Bot Logs**:
```bash
docker-compose logs gir
```

3. **Updating the Bot**:
```bash
git pull && docker-compose up -d --build --force-recreate
```

4. **Stopping the Bot**:
```bash
docker-compose down
```

## Troubleshooting

### Container Keeps Restarting
- Check the logs: `docker-compose logs gir`
- Verify that your `.env` file is properly configured
- Make sure MongoDB is running and accessible

### Slash Commands Not Working
- Ensure you've sent `!sync` to the bot in DMs
- Check that the bot has the correct permissions in your server
- Verify that the `MAIN_GUILD_ID` in your `.env` matches your server ID

### Database Connection Issues
- Check that MongoDB is running with `docker-compose ps`
- Verify network configuration in `docker-compose.yml`
- Ensure `DB_HOST` is set to `mongo` in the `.env` file

## Security Considerations

- The MongoDB instance is configured without authentication. Do not expose port 27017 in production environments.
- The mongo-express web interface should be properly secured with firewall rules in production or disabled completely.
- Use environment variables for all sensitive information and never commit your `.env` file to version control.

## Backups

To backup your MongoDB data:

```bash
# Create a backup of the MongoDB data
docker exec -it gir-mongo mongodump --out /data/db/backup

# Copy the backup to your host system
docker cp gir-mongo:/data/db/backup ./backup
```

To restore from a backup:

```bash
# Copy the backup to the container
docker cp ./backup gir-mongo:/data/db/

# Restore the backup
docker exec -it gir-mongo mongorestore /data/db/backup
```
