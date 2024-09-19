const { Client, GatewayIntentBits, ActionRowBuilder, ButtonBuilder, ButtonStyle } = require('discord.js');

// สร้างอินสแตนซ์บอท
const client = new Client({
  intents: [GatewayIntentBits.Guilds, GatewayIntentBits.GuildMessages, GatewayIntentBits.MessageContent],
});

// สร้างกองไพ่ใหม่
function createDeck() {
  const deck = [];
  const suits = ['♠', '♥', '♦', '♣'];
  const values = ['2', '3', '4', '5', '6', '7', '8', '9', '10', 'J', 'Q', 'K', 'A'];
  
  for (const suit of suits) {
    for (const value of values) {
      deck.push({ suit, value });
    }
  }
  return deck;
}

let deck = createDeck(); // เก็บไพ่เริ่มต้น

// เก็บข้อมูลการออกไพ่
let cardStats = {};
for (const suit of ['♠', '♥', '♦', '♣']) {
  for (const value of ['2', '3', '4', '5', '6', '7', '8', '9', '10', 'J', 'Q', 'K', 'A']) {
    cardStats[`${value}${suit}`] = 0;
  }
}

// ฟังก์ชันในการสุ่มไพ่จากกองไพ่ (โดยไม่ให้ซ้ำ) และอัปเดตสถิติ
function drawCard() {
  if (deck.length === 0) {
    deck = createDeck(); // รีเซ็ตกองไพ่ใหม่เมื่อไพ่หมด
  }
  
  const randomIndex = Math.floor(Math.random() * deck.length);
  const card = deck.splice(randomIndex, 1)[0];  // ลบไพ่ที่ถูกจั่วออกจากกอง
  
  // อัปเดตการออกไพ่ในสถิติ
  cardStats[`${card.value}${card.suit}`] += 1;
  return card;
}

// ฟังก์ชันในการคำนวณคะแนนจากไพ่
function calculatePoints(cards) {
  let points = 0;
  let aces = 0;
  
  for (const card of cards) {
    if (card.value === 'A') {
      points += 11;
      aces += 1;
    } else if (['K', 'Q', 'J'].includes(card.value)) {
      points += 10;
    } else {
      points += parseInt(card.value, 10);
    }
  }
  
  while (points > 21 && aces > 0) {
    points -= 10;
    aces -= 1;
  }
  
  return points;
}

// ฟังก์ชันในการคำนวณเปอร์เซ็นต์ของการออกไพ่แต่ละใบ
function calculateCardPercentages() {
  const totalDraws = Object.values(cardStats).reduce((sum, count) => sum + count, 0);
  const percentages = {};
  for (const card in cardStats) {
    percentages[card] = ((cardStats[card] / totalDraws) * 100).toFixed(2);  // คำนวณเป็นเปอร์เซ็นต์
  }
  return percentages;
}

// เมื่อบอทพร้อมใช้งาน
client.once('ready', () => {
  console.log(`${client.user.tag} พร้อมใช้งานแล้ว!`);
});

client.on('messageCreate', async (message) => {
  if (message.author.bot) return;
  
  const prefix = '!';
  if (!message.content.startsWith(prefix)) return;
  
  const args = message.content.slice(prefix.length).trim().split(/ +/);
  const command = args.shift().toLowerCase();
  
  if (command === 'blackjack') {
    const playerCards = [drawCard(), drawCard()];
    const dealerCards = [drawCard(), drawCard()];
    
    const playerPoints = calculatePoints(playerCards);
    const dealerPoints = calculatePoints(dealerCards);
    
    const row = new ActionRowBuilder()
      .addComponents(
        new ButtonBuilder()
          .setCustomId('hit')
          .setLabel('Hit')
          .setStyle(ButtonStyle.Primary),
        new ButtonBuilder()
          .setCustomId('stand')
          .setLabel('Stand')
          .setStyle(ButtonStyle.Secondary)
      );
    
    const gameMessage = await message.channel.send({
      content: `ไพ่ของคุณ: ${playerCards.map(card => `${card.value}${card.suit}`).join(', ')} (แต้ม: ${playerPoints})\nไพ่ของดีลเลอร์: ${dealerCards[0].value}${dealerCards[0].suit}, ?`,
      components: [row]
    });

    const checkGameResult = (playerPoints, dealerPoints) => {
      if (playerPoints > 21) return 'คุณแพ้! คุณเกิน 21 แต้ม';
      if (dealerPoints > 21) return 'คุณชนะ! ดีลเลอร์เกิน 21 แต้ม';
      if (dealerPoints === playerPoints) return 'เสมอ!';
      if (playerPoints > dealerPoints) return 'คุณชนะ!';
      return 'คุณแพ้!';
    };

    const filter = (interaction) => interaction.user.id === message.author.id;
    const collector = gameMessage.createMessageComponentCollector({ filter, time: 60000 });

    let playerFinished = false;

    collector.on('collect', async (interaction) => {
      if (interaction.customId === 'hit') {
        playerCards.push(drawCard());
        const playerPoints = calculatePoints(playerCards);
        await interaction.update({
          content: `ไพ่ของคุณ: ${playerCards.map(card => `${card.value}${card.suit}`).join(', ')} (แต้ม: ${playerPoints})\nไพ่ของดีลเลอร์: ${dealerCards[0].value}${dealerCards[0].suit}, ?`,
          components: playerPoints >= 21 ? [] : [row]
        });

        if (playerPoints >= 21) {
          playerFinished = true;
          const result = checkGameResult(playerPoints, dealerPoints);
          await interaction.followUp(result);
          collector.stop();
        }
      } else if (interaction.customId === 'stand') {
        playerFinished = true;

        while (dealerPoints < 17) {
          dealerCards.push(drawCard());
          dealerPoints = calculatePoints(dealerCards);
        }

        const result = checkGameResult(playerPoints, dealerPoints);
        await interaction.update({
          content: `ไพ่ของคุณ: ${playerCards.map(card => `${card.value}${card.suit}`).join(', ')} (แต้ม: ${playerPoints})\nไพ่ของดีลเลอร์: ${dealerCards.map(card => `${card.value}${card.suit}`).join(', ')} (แต้ม: ${dealerPoints})`,
          components: []
        });
        await interaction.followUp(result);
        collector.stop();
      }
    });

    collector.on('end', async () => {
      if (!playerFinished) {
        await gameMessage.edit({ content: 'หมดเวลา! เกมสิ้นสุดลง', components: [] });
      }
      // แสดงเปอร์เซ็นต์การออกไพ่หลังจบเกม
      const percentages = calculateCardPercentages();
      await message.channel.send(`เปอร์เซ็นต์การออกไพ่: ${JSON.stringify(percentages)}`);
    });
  }
});

client.login('YOUR_BOT_TOKEN');


<!---
QuarterAries/QuarterAries is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
