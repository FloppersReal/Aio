const { SlashCommandBuilder } = require('@discordjs/builders');
const { EmbedBuilder } = require('discord.js');
const axios = require('axios');
const lang = require('../../events/loadLanguage');

// Store active sessions based on the sender (who initiated the tracking)
const activeSessions = new Map();

// --- Configuration for the dedicated Gagstock channel ---
const GAGSTOCK_CHANNEL_ID = 'YOUR_GAGSTOCK_CHANNEL_ID_HERE'; // <--- !!! REPLACE THIS WITH YOUR ACTUAL CHANNEL ID !!!


// Helper function for countdown
const countdown = (updatedAt, intervalSec) => {
    const now = Date.now();
    const diff = Math.max(intervalSec - Math.floor((now - updatedAt) / 1000), 0);
    const m = Math.floor((diff % 3600) / 60);
    const s = diff % 60;
    return `${m}m ${s}s`;
};

// Helper function for honey countdown (specific to Manila time)
const honeyCountdown = () => {
    const now = new Date(new Date().toLocaleString("en-US", { timeZone: "Asia/Manila" }));
    const m = 59 - now.getMinutes();
    const s = 60 - now.getSeconds();
    return `${m}m ${s < 10 ? `0${s}` : s}s`;
};

// Function to fetch and notify about stock changes
// This function now needs access to the Discord client to fetch the channel
const fetchAndNotify = async (client, senderId) => {
    const sessionData = activeSessions.get(senderId);
    if (!sessionData) return;

    try {
        const [
            gearRes, eggRes, weatherRes, honeyRes, cosmeticsRes, emojiRes
        ] = await Promise.all([
            axios.get('https://growagardenstock.com/api/stock?type=gear-seeds'),
            axios.get('https://growagardenstock.com/api/stock?type=egg'),
            axios.get('https://growagardenstock.com/api/stock/weather'),
            axios.get('http://65.108.103.151:22377/api/stocks?type=honeyStock'),
            axios.get('https://growagardenstock.com/api/special-stock?type=cosmetics'),
            axios.get('http://65.108.103.151:22377/api/stocks?type=seedsStock')
        ]);

        const gear = gearRes.data;
        const egg = eggRes.data;
        const weather = weatherRes.data;
        const honey = honeyRes.data;
        const cosmetics = cosmeticsRes.data;
        const emojis = emojiRes.data?.seedsStock || [];

        const key = JSON.stringify({
            gear: gear.gear,
            seeds: gear.seeds,
            egg: egg.egg,
            weather: weather.updatedAt,
            honey: honey.honeyStock,
            cosmetics: cosmetics.cosmetics
        });

        if (key === sessionData.lastKey) return;
        sessionData.lastKey = key;

        const cosmeticsTimer = countdown(cosmetics.updatedAt, 14400);
        const gearTimer = countdown(gear.updatedAt, 300);
        const eggTimer = countdown(egg.updatedAt, 600);
        const honeyTimer = honeyCountdown();

        const gearText = gear.gear?.map(g => `- ${g}`).join("\n") || "None";
        const seedText = gear.seeds?.map(s => {
            const name = s.split(" **")[0];
            const emoji = emojis.find(e => e.name.toLowerCase() === name.toLowerCase())?.emoji || "";
            return `- ${emoji} ${s}`;
        }).join("\n") || "None";

        const eggText = egg.egg?.map(e => `- ${e}`).join("\n") || "None";
        const honeyText = honey.honeyStock?.map(h => `- ${h.name}: ${h.value}`).join("\n") || "None";
        const cosmeticsText = cosmetics.cosmetics?.map(c => `- ${c}`).join("\n") || "None";
        const weatherText = `${weather.icon || "☁️"} ${weather.currentWeather || "Unknown"}`;
        const cropBonus = weather.cropBonuses || "None";

        const msgContent = `🌿 𝗚𝗿𝗼𝘄 𝗔 𝗚𝗮𝗿𝗱𝗲𝗻 — 𝗦𝘁𝗼𝗰𝗸 𝗨𝗽𝗱𝗮𝘁𝗲\n\n` +
            `🛠️ 𝗚𝗲𝗮𝗿:\n${gearText}\n\n🌱 𝗦𝗲𝗲𝗱𝘀:\n${seedText}\n\n🥚 𝗘𝗴𝗴𝘀:\n${eggText}\n\n` +
            `🎨 𝗖𝗼𝘀𝗺𝗲𝘁𝗶𝗰𝘀:\n${cosmeticsText}\n⏳ 𝗖𝗼𝘀𝗺𝗲𝘁𝗶𝗰 𝗿𝗲𝘀𝘁𝗼𝗰𝗸: ${cosmeticsTimer}\n\n` +
            `🍯 𝗛𝗼𝗻𝗲𝘆:\n${honeyText}\n⏳ 𝗛𝗼𝗻𝗲𝘆 𝗿𝗲𝘀𝘁𝗼𝗰k: ${honeyTimer}\n\n` +
            `🌤️ 𝗪𝗲𝗮𝘁𝗵𝗲𝗿: ${weatherText}\n🪴 𝗖𝗿𝗼𝗽 𝗕𝗼𝗻𝘂𝘀: ${cropBonus}\n\n` +
            `📅 𝗚𝗲𝗮𝗿/𝗦𝗲𝗲𝗱 𝗿𝗲𝘀𝘁𝗼𝗰𝗸: ${gearTimer}\n📅 𝗘𝗴𝗴 𝗿𝗲𝘀𝘁𝗼𝗰𝗸: ${eggTimer}`;

        if (msgContent !== sessionData.lastMsg) {
            sessionData.lastMsg = msgContent;
            const embed = new EmbedBuilder()
                .setColor(0x00FF00)
                .setTitle('Grow A Garden Stock Update')
                .setDescription(msgContent)
                .setTimestamp();

            const messagePayload = {
                content: '@everyone', // The @everyone mention
                embeds: [embed]
            };

            // Fetch the specific channel
            const targetChannel = await client.channels.fetch(GAGSTOCK_CHANNEL_ID);
            if (targetChannel && targetChannel.isTextBased()) {
                await targetChannel.send(messagePayload);
            } else {
                console.error(`Could not find text channel with ID: ${GAGSTOCK_CHANNEL_ID}`);
                // Optionally, inform the user who started the tracking if the channel isn't found
                const user = await client.users.fetch(senderId);
                if (user) {
                    await user.send(`⚠️ I couldn't send the Gagstock update to the designated channel (ID: ${GAGSTOCK_CHANNEL_ID}). Please ensure the channel exists and I have permissions.`);
                }
            }
        }

    } catch (err) {
        console.error('gagstock error:', err.message);
        // Optionally, send an error message to the user who started the tracking
        const user = await client.users.fetch(senderId);
        if (user) {
            const errorEmbed = new EmbedBuilder()
                .setColor(0xFF0000)
                .setTitle('Error Fetching Gagstock Data')
                .setDescription('An error occurred while fetching Grow A Garden stock data. Please try again later.')
                .setTimestamp();
            await user.send({ embeds: [errorEmbed] });
        }
    }
};


module.exports = {
    data: new SlashCommandBuilder()
        .setName('gagstock')
        .setDescription('Track Grow A Garden stocks, weather, and cosmetics in real time.')
        .addStringOption(option =>
            option.setName('action')
                .setDescription('Start or stop tracking.')
                .setRequired(true)
                .addChoices(
                    { name: 'on', value: 'on' },
                    { name: 'off', value: 'off' }
                )),

    async execute(interaction) {
        const action = interaction.options.getString('action');
        const senderId = interaction.user.id;
        const client = interaction.client; // Get the Discord client instance

        if (action === 'off') {
            const session = activeSessions.get(senderId);
            if (session) {
                clearInterval(session.interval);
                activeSessions.delete(senderId);
                return interaction.reply({ content: '🛑 Gagstock tracking stopped.', ephemeral: false });
            } else {
                return interaction.reply({ content: '⚠️ You have no active gagstock tracking.', ephemeral: false });
            }
        }

        if (activeSessions.has(senderId)) {
            return interaction.reply({ content: '📡 You\'re already tracking Gagstock. Use `/gagstock off` to stop.', ephemeral: false });
        }

        await interaction.deferReply({ ephemeral: false });

        const sessionData = {
            interval: null,
            lastKey: null,
            lastMsg: ''
        };

        // Pass the client instance to fetchAndNotify
        sessionData.interval = setInterval(() => fetchAndNotify(client, senderId), 300000); // Changed to 300000 milliseconds (5 minutes)
        activeSessions.set(senderId, sessionData);

        await interaction.followUp({ content: `✅ Gagstock tracking started! Updates will be sent to the designated channel.` });
        await fetchAndNotify(client, senderId); // Initial fetch
    },

    async executePrefix(message, args) {
        const action = args[0]?.toLowerCase();
        const senderId = message.author.id;
        const client = message.client; // Get the Discord client instance

        if (!action || !['on', 'off'].includes(action)) {
            return message.channel.send(
                '📌 𝗨𝘀𝗮𝗴𝗲:\n• `gagstock on` — Start tracking\n• `gagstock off` — Stop tracking'
            );
        }

        if (action === 'off') {
            const session = activeSessions.get(senderId);
            if (session) {
                clearInterval(session.interval);
                activeSessions.delete(senderId);
                return message.channel.send('🛑 Gagstock tracking stopped.');
            } else {
                return message.channel.send('⚠️ You have no active gagstock tracking.');
            }
        }

        if (activeSessions.has(senderId)) {
            return message.channel.send('📡 You\'re already tracking Gagstock. Use `gagstock off` to stop.');
        }

        await message.channel.send('✅ Gagstock tracking started! Updates will be sent to the designated channel.');

        const sessionData = {
            interval: null,
            lastKey: null,
            lastMsg: ''
        };

        // Pass the client instance to fetchAndNotify
        sessionData.interval = setInterval(() => fetchAndNotify(client, senderId), 300000); // Changed to 300000 milliseconds (5 minutes)
        activeSessions.set(senderId, sessionData);
        await fetchAndNotify(client, senderId); // Initial fetch
    },
};
