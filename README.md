# seeker

Inspired by Mistborn Seekers, seeker can sense power. This is a collection of tools to help track metrics, logs, and other data in a distributed system.

## Getting Started

Prerequisites:
- Docker
- Telegram account

This setup uses a standalone docker network. 

To create the Docker network, run:
```sh
docker network create gs_network_dev
```
Change the name to reflect your landing zone and environment.

## Tools

### Watchtower

Watchtower automatically updates running Docker containers whenever a new image is available. It also sends notifications about the updates.

**Warning:**  Pinning all of your containers to "latest" should be done with caution. Consider if this is within your risk appetite for your environment.

#### Configuration

Watchtower is configured using the `config.json` file and environment variables in the `docker-compose.yml` file.

- `config.json`: Contains authentication details for Docker registry.
- `docker-compose.yml`: Contains environment variables for Watchtower notifications and scheduling.

Steps: https://containrrr.dev/watchtower/private-registries/


To base64 encode your Docker registry authentication, run:
```sh
echo -n 'username:password' | base64
```
Replace `username` and `password` with your Docker registry credentials.

**Warning:** Base64 encoding is very easily reversed. Do not commit the encoded credentials to GitHub.


#### Environment Variables

- `TZ`: Time zone for the container (e.g., `US/Central`).
- `WATCHTOWER_NOTIFICATION_REPORT`: Enables detailed notifications about updates (set to `"true"`).
- `WATCHTOWER_SCHEDULE`: Cron expression to schedule updates (e.g., `"0 9 15-23 * * *"` runs at minute 9 of hours 15-23).
- `WATCHTOWER_CLEANUP`: Removes old images after updates (set to `true`).
- `WATCHTOWER_NOTIFICATIONS`: Notification service to use (e.g., `shoutrrr`).
- `WATCHTOWER_NOTIFICATION_URL`: URL for the notification service (e.g., `telegram://${BOT_TOKEN}@telegram/?channels=${CHAT_ID}`).
- `WATCHTOWER_NOTIFICATION_TEMPLATE`: Template for the notification message.

#### Setting Up Telegram Notifications

Below are the steps to set up Telegram notifications for Watchtower updates:

1. **Create a Telegram Bot**
   - Open Telegram and search for BotFather. Start a chat with BotFather and send the `/newbot` command. Follow the prompts to choose a name and username for your bot. Once completed, BotFather will provide you with a bot token. This token (BOT_TOKEN) is needed to allow Watchtower to send messages.

2. **Create a Telegram Channel or Group**
   - Create a channel (or a group) where you want to receive the update messages. If you use a channel, make sure that it is set up to receive messages from bots.

3. **Invite the Bot to Your Channel/Group**
   - Add your new bot to the channel or group as an administrator. This is required for the bot to post messages. For channels, simply use the “Add Admin” process; for groups, send the `/add` command or use the group settings to add the bot.

4. **Obtain the Channel or Group Identifier**
   - The WATCHTOWER_NOTIFICATION_URL requires a channel identifier (CHAT_ID). For public channels, you can often use the channel’s username (prefixed with @), but if you need a numeric chat id, you might use:
     - A dedicated Telegram ID bot, such as `userinfobot`, to get your channel or group id.
     - Forward a message from your channel to the bot and check the details. Take note of this identifier (CHAT_ID).

5. **Update Your environment variables**
   - Replace TELEGRAM_BOT_TOKEN and TELEGRAM_CHAT_ID in the Watchtower environment variable with the values you obtained.

For more details on obtaining the channel ID, you can refer to this [gist](https://gist.github.com/mraaroncruz/e76d19f7d61d59419002db54030ebe35).

### Loki

Loki is a log aggregation system designed to store and query logs from various sources.

#### Configuration

Loki is configured using the `loki-config.yml` file.

- `loki-config.yml`: Contains settings for server ports, storage paths, and schema configurations.

### Grafana

Grafana is a data visualization tool that integrates with Loki to display logs and metrics.

#### Configuration

Grafana is configured using the `docker-compose.yml` file.

- `docker-compose.yml`: Contains volume mappings and network settings for Grafana.

## Environment Variables

The environment variables are defined in the `prod.env` file.

- `LZ`: Logical zone (e.g., `gs`).
- `ENV`: Environment (e.g., `prod`).
- `COMPOSE_PROJECT_NAME`: Docker Compose project name.
- `BOT_TOKEN`: Telegram bot token for notifications.
- `CHAT_ID`: Telegram chat ID for notifications.

## Running the Stack

To run the stack, use Docker Compose:

```sh
docker-compose up -d
```

This command will start Loki, Grafana, and Watchtower services as defined in the `docker-compose.yml` file.