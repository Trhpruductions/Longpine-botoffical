// LongPine Bot - Single File Version
// Place this file in a folder, run `npm install discord.js mongoose dotenv` before starting.
// Create a .env file with BOT_TOKEN, MONGO_URI, OWNER_ID, CLIENT_ID, CLIENT_SECRET, PUBLIC_KEY, PERMISSIONS, INSTALL_LINK

require('dotenv').config();
const { Client, GatewayIntentBits, Partials, Collection } = require('discord.js');
const mongoose = require('mongoose');
const fs = require('fs');

// --- CONFIGURATION ---
const config = {
    token: process.env.BOT_TOKEN,
    mongoUri: process.env.MONGO_URI,
    prefix: '/',
    ownerId: process.env.OWNER_ID,
};

// --- DATABASE SETUP (Economy) ---
const economySchema = new mongoose.Schema({
    userId: String,
    coins: { type: Number, default: 0 },
    lastEarn: { type: Number, default: 0 }
});
const Economy = mongoose.model('Economy', economySchema);

// --- DISCORD CLIENT SETUP ---
const client = new Client({
    intents: [
        GatewayIntentBits.Guilds,
        GatewayIntentBits.GuildMessages,
        GatewayIntentBits.MessageContent,
        GatewayIntentBits.GuildMembers,
        GatewayIntentBits.GuildPresences,
    ],
    partials: [Partials.Message, Partials.Channel, Partials.Reaction],
});

client.commands = new Collection();

// --- COMMANDS ---
const commands = [
    {
        name: 'ping',
        description: 'Replies with Pong!',
        async execute(interaction) {
            await interaction.reply('Pong!');
        }
    },
    {
        name: 'server',
        description: 'Displays server info.',
        async execute(interaction) {
            await interaction.reply(`Server: ${interaction.guild?.name || 'Unknown'} | ID: ${interaction.guild?.id || 'Unknown'}`);
        }
    },
    {
        name: 'status',
        description: 'Shows if a player is in-city and their role.',
        async execute(interaction) {
            await interaction.reply('Status feature coming soon.');
        }
    },
    {
        name: 'balance',
        description: 'Check your coin balance.',
        async execute(interaction) {
            const userId = interaction.user.id;
            let entry = await Economy.findOne({ userId });
            const bal = entry ? entry.coins : 0;
            await interaction.reply(`You have ${bal} coins.`);
        }
    },
    {
        name: 'earn',
        description: 'Earn random coins (cooldown: 30s).',
        async execute(interaction) {
            const userId = interaction.user.id;
            let entry = await Economy.findOne({ userId });
            const now = Date.now();
            if (entry && now - entry.lastEarn < 30000) {
                await interaction.reply('You must wait before earning again!');
                return;
            }
            const coins = Math.floor(Math.random() * 50) + 10;
            if (!entry) entry = new Economy({ userId, coins: 0 });
            entry.coins += coins;
            entry.lastEarn = now;
            await entry.save();
            await interaction.reply(`You earned ${coins} coins!`);
        }
    },
    {
        name: 'leaderboard',
        description: 'Show the top 5 richest players.',
        async execute(interaction) {
            const top = await Economy.find().sort({ coins: -1 }).limit(5);
            if (!top.length) {
                await interaction.reply('No leaderboard data yet.');
                return;
            }
            let msg = '**Leaderboard:**\n';
            for (let i = 0; i < top.length; i++) {
                msg += `#${i + 1} <@${top[i].userId}>: ${top[i].coins} coins\n`;
            }
            await interaction.reply({ content: msg, allowedMentions: { users: [] } });
        }
    },
    {
        name: 'gamble',
        description: 'Gamble coins for a chance to win big! Usage: /gamble <amount>',
        options: [{ name: 'amount', type: 4, description: 'Amount to gamble', required: true }],
        async execute(interaction) {
            const userId = interaction.user.id;
            const amount = interaction.options.getInteger('amount');
            if (!amount || amount <= 0) {
                await interaction.reply('Please specify a valid amount to gamble.');
                return;
            }
            let entry = await Economy.findOne({ userId });
            if (!entry) entry = new Economy({ userId, coins: 0 });
            if (entry.coins < amount) {
                await interaction.reply('You do not have enough coins to gamble that amount.');
                return;
            }
            const roll = Math.random();
            let result, won = 0;
            if (roll < 0.48) {
                won = amount;
                entry.coins += won;
                result = `You won ${won} coins!`;
            } else if (roll < 0.96) {
                won = -amount;
                entry.coins += won;
                result = `You lost ${amount} coins.`;
            } else {
                won = amount * 3;
                entry.coins += won;
                result = `JACKPOT! You won ${won} coins!`;
            }
            await entry.save();
            await interaction.reply(result + ` (Balance: ${entry.coins})`);
        }
    }
];

// Register commands
client.once('ready', async () => {
    console.log(`LongPine Bot is online as ${client.user.tag}`);
    // Register slash commands (guild only for fast update)
    const guild = client.guilds.cache.first();
    if (guild) {
        await guild.commands.set(commands.map(cmd => ({
            name: cmd.name,
            description: cmd.description,
            options: cmd.options || []
        })));
    }
    console.log('Commands registered.');
});

client.on('interactionCreate', async interaction => {
    if (!interaction.isCommand()) return;
    const cmd = commands.find(c => c.name === interaction.commandName);
    if (cmd) {
        try {
            await cmd.execute(interaction);
        } catch (err) {
            console.error(err);
            await interaction.reply({ content: 'Error executing command.', ephemeral: true });
        }
    }
});

// --- DATABASE CONNECTION & BOT STARTUP ---
(async () => {
    try {
        await mongoose.connect(config.mongoUri, {
            useNewUrlParser: true,
            useUnifiedTopology: true,
        });
        console.log('MongoDB connected.');
        await client.login(config.token);
    } catch (err) {
        console.error('Startup error:', err);
        process.exit(1);
    }
})();
