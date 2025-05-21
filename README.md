# TWITTER-BOT

# 🔧 Step 1: Project Setup
  
  1. Create a project folder:
```bash
mkdir twitter-tracker-bot && cd twitter-tracker-bot
```

2. Initialize Node.js project:

```bash
npm init -y
```
3. Install dependencies:
```bash
npm install dotenv fs path puppeteer node-telegram-bot-api axios
```

# 📁 Step 2: Create Project Files

1. .env (store sensitive data)
Create a .env file:
```bash
touch .env
```

Add the following:

```bash
TELEGRAM_TOKEN=your_telegram_bot_token
CHAT_ID=your_telegram_chat_id
TWITTER_USERNAME=your_twitter_username
TWITTER_PASSWORD=your_twitter_password
```

# 2. trackedHandles.json (auto-generated)
This file will be created automatically to track user states. No need to create it manually.

# 🧠 Step 3: Bot Script
Create a script file:
```bash
touch bot.js
```

Paste the full script you shared above into bot.js.

```bash
require('dotenv').config();
const fs = require('fs');
const path = require('path');
const puppeteer = require('puppeteer');
const TelegramBot = require('node-telegram-bot-api');
const axios = require('axios');

const TELEGRAM_TOKEN = process.env.TELEGRAM_TOKEN;
const CHAT_ID = process.env.CHAT_ID;
const TWITTER_USERNAME = process.env.TWITTER_USERNAME;
const TWITTER_PASSWORD = process.env.TWITTER_PASSWORD;

const TRACK_FILE = path.join(__dirname, 'trackedHandles.json');
let tracked = { follow: {}, tweet: {}, discord: {} };
let twitterAuthHeaders = null;

const bot = new TelegramBot(TELEGRAM_TOKEN, { polling: true });
const LOW_FOLLOWER_THRESHOLD = 100;

if (fs.existsSync(TRACK_FILE)) {
  tracked = JSON.parse(fs.readFileSync(TRACK_FILE));
}

async function saveTracked() {
  fs.writeFileSync(TRACK_FILE, JSON.stringify(tracked, null, 2));
}

async function loginAndGetTwitterAuth() {
  const browser = await puppeteer.launch({ headless: true });
  const page = await browser.newPage();
  await page.goto('https://twitter.com/login', { waitUntil: 'networkidle2' });

  await page.type('input[name="session[username_or_email]"]', TWITTER_USERNAME, { delay: 50 });
  await page.type('input[name="session[password]"]', TWITTER_PASSWORD, { delay: 50 });
  await Promise.all([
    page.click('div[data-testid="LoginForm_Login_Button"]'),
    page.waitForNavigation({ waitUntil: 'networkidle2' }),
  ]);

  await page.goto('https://twitter.com/home', { waitUntil: 'networkidle2' });

  const request = await page.waitForRequest(req =>
    req.url().startsWith('https://api.twitter.com/2/'), { timeout: 10000 });

  const headers = request.headers();

  twitterAuthHeaders = {
    authorization: headers['authorization'],
    'x-csrf-token': headers['x-csrf-token'],
    cookie: headers['cookie'],
    'user-agent': headers['user-agent'],
  };

  await browser.close();
  console.log('Twitter auth headers refreshed.');
}

async function fetchUserInfo(username) {
  try {
    const res = await axios.get(`https://api.twitter.com/2/users/by/username/${username}`, {
      headers: twitterAuthHeaders,
    });
    return res.data.data;
  } catch {
    return null;
  }
}

async function fetchLatestTweets(userId) {
  try {
    const res = await axios.get(`https://api.twitter.com/2/users/${userId}/tweets?tweet.fields=created_at&max_results=5`, {
      headers: twitterAuthHeaders,
    });
    return res.data.data || [];
  } catch {
    return [];
  }
}

async function fetchFollowing(userId) {
  try {
    const res = await axios.get(`https://api.twitter.com/2/users/${userId}/following?user.fields=public_metrics,description,verified,protected`, {
      headers: twitterAuthHeaders,
    });
    return res.data.data || [];
  } catch {
    return [];
  }
}

function formatFollowAlert(watchedUser, followedUser) {
  return `👤 @${watchedUser.username} followed @${followedUser.username}
📔 Bio: ${followedUser.description || 'N/A'}
👥 Followers: ${followedUser.public_metrics.followers_count}
🔒 Protected: ${followedUser.protected ? 'Yes' : 'No'}`;
}

async function checkUpdates() {
  if (!twitterAuthHeaders) await loginAndGetTwitterAuth();

  const combinedUsernames = new Set([
    ...Object.keys(tracked.follow),
    ...Object.keys(tracked.tweet),
    ...Object.keys(tracked.discord)
  ]);

  for (const username of combinedUsernames) {
    const userData = tracked.follow[username] || tracked.tweet[username] || tracked.discord[username];
    const userInfo = await fetchUserInfo(username);
    if (!userInfo) continue;

    // Tweets
    const tweets = await fetchLatestTweets(userInfo.id);
    if (tweets.length > 0) {
      const latest = tweets[0];
      const tweetUrl = `https://twitter.com/${username}/status/${latest.id}`;

      // /tweet alerts
      if (tracked.tweet[username] && tracked.tweet[username].lastTweetId !== latest.id) {
        await bot.sendMessage(CHAT_ID, `@${username} tweeted:\n${latest.text}\n${tweetUrl}`);
        tracked.tweet[username].lastTweetId = latest.id;
      }

      // /discord alerts
      const containsDiscord = /discord\.gg|discord\.com\/invite/i.test(latest.text);
      if (tracked.discord[username] && tracked.discord[username].lastTweetId !== latest.id && containsDiscord) {
        await bot.sendMessage(CHAT_ID, `🚨 Discord link alert from @${username}:\n${latest.text}\n${tweetUrl}`);
        tracked.discord[username].lastTweetId = latest.id;
      }
    }

    // Follow alerts
    if (tracked.follow[username]) {
      const currFollowing = await fetchFollowing(userInfo.id);
      const previous = tracked.follow[username].following || [];
      const newFollows = currFollowing.filter(f => !previous.some(p => p.id === f.id));
      for (const followedUser of newFollows) {
        await bot.sendMessage(CHAT_ID, formatFollowAlert(userInfo, followedUser));
      }
      tracked.follow[username].following = currFollowing;
    }

    await saveTracked();
  }
}

// COMMANDS

bot.onText(/\/follow (@?\w+)/, async (msg, match) => {
  const username = match[1].replace('@', '').toLowerCase();
  if (tracked.follow[username]) return bot.sendMessage(msg.chat.id, `Already tracking follows for @${username}`);
  const info = await fetchUserInfo(username);
  if (!info) return bot.sendMessage(msg.chat.id, `User @${username} not found`);
  const following = await fetchFollowing(info.id);
  tracked.follow[username] = { following };
  await saveTracked();
  bot.sendMessage(msg.chat.id, `Started tracking follows for @${username}`);
});

bot.onText(/\/tweet (@?\w+)/, async (msg, match) => {
  const username = match[1].replace('@', '').toLowerCase();
  if (tracked.tweet[username]) return bot.sendMessage(msg.chat.id, `Already tracking tweets for @${username}`);
  const info = await fetchUserInfo(username);
  if (!info) return bot.sendMessage(msg.chat.id, `User @${username} not found`);
  const tweets = await fetchLatestTweets(info.id);
  tracked.tweet[username] = { lastTweetId: tweets[0]?.id || null };
  await saveTracked();
  bot.sendMessage(msg.chat.id, `Started tracking tweets for @${username}`);
});

bot.onText(/\/discord (@?\w+)/, async (msg, match) => {
  const username = match[1].replace('@', '').toLowerCase();
  if (tracked.discord[username]) return bot.sendMessage(msg.chat.id, `Already tracking Discord links from @${username}`);
  const info = await fetchUserInfo(username);
  if (!info) return bot.sendMessage(msg.chat.id, `User @${username} not found`);
  const tweets = await fetchLatestTweets(info.id);
  tracked.discord[username] = { lastTweetId: tweets[0]?.id || null };
  await saveTracked();
  bot.sendMessage(msg.chat.id, `Started tracking Discord links from @${username}`);
});

bot.onText(/\/help/, msg => {
  const commands = [
    '/follow @username - Track accounts they follow',
    '/tweet @username - Get notified when they tweet',
    '/discord @username - Alert on tweets with Discord links'
  ];
  bot.sendMessage(msg.chat.id, `Available commands:\n\n${commands.join('\n')}`);
});

bot.setMyCommands([
  { command: 'follow', description: 'Track follows of a Twitter user' },
  { command: 'tweet', description: 'Track new tweets from a user' },
  { command: 'discord', description: 'Track only Discord links from a user' },
  { command: 'help', description: 'Show available commands' }
]);

// Start monitoring
setInterval(() => checkUpdates().catch(console.error), 2 * 60 * 1000);

(async () => {
  await loginAndGetTwitterAuth();
  console.log('Bot started.');
  checkUpdates();
})();
```

# ✅ Step 4: Run the Bot
```bash
node bot.js
```
If everything is correct, you'll see:

```bash
Twitter auth headers refreshed.
Bot started.
```

# 🧪 Step 5: Test Telegram Commands

Open your Telegram app and talk to your bot:

/follow @elonmusk → alerts when Elon follows someone

/tweet @elonmusk → alerts when Elon tweets

/discord @elonmusk → alerts when Elon tweets a Discord link

/help → shows commands

# 💡 Tips and Notes
Headless Puppeteer:
If login fails due to bot detection, try:

Set bot polling on a server:

Use pm2 or screen to keep the bot running persistently.

```bash
npm install -g pm2
pm2 start bot.js
pm2 save
```
Cron Alternative:
The bot uses setInterval() to check updates every 2 minutes.

Twitter API Caution:
You're scraping Twitter API headers using Puppeteer (due to Twitter API restrictions). This may break if Twitter changes structure. Keep the project maintained accordingly.




















