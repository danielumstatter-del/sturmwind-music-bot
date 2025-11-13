# sturmwind-music-bot// index.js
const { Client, GatewayIntentBits, SlashCommandBuilder, Routes, REST } = require('discord.js');
const { joinVoiceChannel, createAudioPlayer, createAudioResource, AudioPlayerStatus } = require('@discordjs/voice');
const play = require('play-dl');
const express = require('express');

const TOKEN = process.env.TOKEN;
const CLIENT_ID = process.env.CLIENT_ID;
const GUILD_ID = process.env.GUILD_ID;

const client = new Client({
  intents: [GatewayIntentBits.Guilds, GatewayIntentBits.GuildVoiceStates],
});

const commands = [
  new SlashCommandBuilder()
    .setName('play')
    .setDescription('Spielt einen Song von YouTube oder Spotify ab')
    .addStringOption(option => option.setName('link').setDescription('YouTube/Spotify Link').setRequired(true)),
  new SlashCommandBuilder().setName('stop').setDescription('Stoppt die Wiedergabe'),
  new SlashCommandBuilder().setName('skip').setDescription('Überspringt den Song'),
];

const rest = new REST({ version: '10' }).setToken(TOKEN);

(async () => {
  try {
    console.log('🔁 Lade Slash-Commands hoch...');
    await rest.put(Routes.applicationGuildCommands(CLIENT_ID, GUILD_ID), { body: commands });
    console.log('✅ Slash-Commands registriert.');
  } catch (error) {
    console.error(error);
  }
})();

const player = createAudioPlayer();
let connection;

client.on('ready', () => console.log(`🤖 Eingeloggt als ${client.user.tag}!`));

client.on('interactionCreate', async interaction => {
  if (!interaction.isChatInputCommand()) return;

  const { commandName } = interaction;

  if (commandName === 'play') {
    const link = interaction.options.getString('link');
    const channel = interaction.member.voice.channel;
    if (!channel) return interaction.reply('❗ Du musst in einem Voice-Channel sein!');
    
    await interaction.reply(`🎶 Lade: ${link} ...`);
    
    connection = joinVoiceChannel({
      channelId: channel.id,
      guildId: channel.guild.id,
      adapterCreator: channel.guild.voiceAdapterCreator,
    });

    try {
      const stream = await play.stream(link);
      const resource = createAudioResource(stream.stream, { inputType: stream.type });
      player.play(resource);
      connection.subscribe(player);
      player.on(AudioPlayerStatus.Idle, () => console.log('Song beendet.'));
      await interaction.followUp(`✅ Jetzt läuft: ${link}`);
    } catch {
      await interaction.followUp('❌ Fehler beim Abspielen!');
    }
  }

  if (commandName === 'stop') {
    player.stop();
    if (connection) connection.destroy();
    interaction.reply('🛑 Wiedergabe gestoppt.');
  }

  if (commandName === 'skip') {
    player.stop();
    interaction.reply('⏭️ Song übersprungen.');
  }
});

// Keep-Alive
const app = express();
app.get('/', (req, res) => res.send('Sturmwind Musik Bot online!'));
app.listen(3000, () => console.log('✅ Keep-Alive läuft auf Port 3000'));

client.login(TOKEN);

