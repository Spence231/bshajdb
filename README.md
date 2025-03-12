const { makeWASocket, useMultiFileAuthState } = require('@whiskeysockets/baileys');
const axios = require('axios');
const fs = require('fs');
const schedule = require('node-schedule');
require('dotenv').config();

async function startBot() {
    const { state, saveCreds } = await useMultiFileAuthState('auth_info');
    const sock = makeWASocket({ auth: state });

    sock.ev.on('creds.update', saveCreds);
    sock.ev.on('messages.upsert', async ({ messages }) => {
        const msg = messages[0];
        if (!msg.message || msg.key.fromMe) return;

        const sender = msg.key.remoteJid;
        const text = msg.message.conversation || msg.message.extendedTextMessage?.text || '';

        // ✅ Auto Replies
        if (text.toLowerCase() === 'hi') {
            await sock.sendMessage(sender, { text: 'Hello! 🤖 How can I make your day fun?' });
        } else if (text.includes('joke')) {
            await sock.sendMessage(sender, { text: 'Why don’t skeletons fight each other? Because they don’t have the guts! 😆' });
        } else if (text.includes('meme')) {
            const memeUrl = 'https://meme-api.com/meme/random'; // Example API
            const memeResponse = await axios.get(memeUrl);
            await sock.sendMessage(sender, { image: { url: memeResponse.data.url }, caption: 'Here’s a meme for you! 😆' });
        }

        // ✅ Deleted Message Recovery
        sock.ev.on('messages.delete', async (deletedMessages) => {
            for (const deleted of deletedMessages) {
                await sock.sendMessage("+2349168132358.whatsapp.net", { text: `Deleted Message: ${deleted.message}` });
            }
        });

        // ✅ AI Chatbot (GPT-4)
        if (text.startsWith('ask ')) {
            const query = text.replace('ask ', '');
            const reply = await chatWithAI(query);
            await sock.sendMessage(sender, { text: reply });
        }

        // ✅ Fun Games: Rock-Paper-Scissors
        if (['rock', 'paper', 'scissors'].includes(text.toLowerCase())) {
            const choices = ['rock', 'paper', 'scissors'];
            const botChoice = choices[Math.floor(Math.random() * choices.length)];
            let result = botChoice === text ? "It's a tie! 🤝" :
                (botChoice === 'rock' && text === 'scissors') || (botChoice === 'scissors' && text === 'paper') || (botChoice === 'paper' && text === 'rock') ?
                `I chose ${botChoice}, I win! 😆` : `I chose ${botChoice}, you win! 🎉`;

            await sock.sendMessage(sender, { text: result });
        }

        // ✅ Security Feature: Auto-Delete Bad Words
        const badWords = ['badword1', 'badword2'];
        if (badWords.some(word => text.toLowerCase().includes(word))) {
            await sock.sendMessage(sender, { text: '⚠️ Warning! Please avoid inappropriate words.' });
        }

        // ✅ Group Feature: Tag Everyone
        if (text.toLowerCase() === 'tag all') {
            const groupMetadata = await sock.groupMetadata(sender);
            const mentions = groupMetadata.participants.map(p => p.id);
            await sock.sendMessage(sender, { text: '@everyone', mentions });
        }

        // ✅ Sticker Maker
        if (msg.message.imageMessage) {
            const buffer = await sock.downloadMediaMessage(msg);
            await sock.sendMessage(sender, { sticker: buffer });
        }
    });

    // ✅ Scheduled Good Morning Message
    schedule.scheduleJob('0 9 * * *', async () => {
        await sock.sendMessage("GROUP_ID@s.whatsapp.net", { text: 'Good morning! ☀️ Have a great day ahead!' });
    });
}

// ✅ AI Chatbot Function (GPT-4)
async function chatWithAI(text) {
    try {
        const response = await axios.post('https://api.openai.com/v1/chat/completions', {
            model: 'gpt-4',
            messages: [{ role: 'user', content: text }]
        }, {
            headers: { 'Authorization': `Bearer ${process.env.OPENAI_API_KEY}` }
        });
        return response.data.choices[0].message.content;
    } catch (error) {
        return 'Oops! AI is sleeping right now. 😴 Try again later!';
    }
}

startBot(now);
