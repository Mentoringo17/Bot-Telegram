# Bot-Telegram
const fetch = require('node-fetch');
const TOKEN = '8679998760:AAFeVEtdFJbJP_NNnzIIY6yF2XhCyj7MeY8';
const CHANNEL = -1004448079386;

class RateLimiter {
  constructor() {
    this.queue = [];
    this.processing = false;
  }
  async add(chatId, msg, priority = 0) {
    return new Promise((r, rej) => {
      this.queue.push({ chatId, msg, priority, r, rej });
      this.queue.sort((a, b) => a.priority - b.priority);
      if (!this.processing) this.process();
    });
  }
  async process() {
    if (!this.queue.length) { this.processing = false; return; }
    this.processing = true;
    const item = this.queue.shift();
    try {
      const url = `https://api.telegram.org/bot${TOKEN}/sendMessage`;
      const res = await fetch(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ chat_id: item.chatId, text: item.msg })
      });
      const data = await res.json();
      if (!data.ok) throw new Error(data.description);
      item.r(data);
    } catch (e) { item.rej(e); }
    setTimeout(() => this.process(), 500);
  }
}

class TelegramStorage {
  constructor() {
    this.apiUrl = `https://api.telegram.org/bot${TOKEN}`;
    this.rateLimiter = new RateLimiter();
  }
  async uploadVideo(file, userId) {
    const form = new FormData();
    form.append('chat_id', CHANNEL);
    form.append('video', file);
    form.append('caption', `👤 User: ${userId} | 📅 ${new Date().toLocaleString()}`);
    form.append('supports_streaming', 'true');
    const res = await fetch(`${this.apiUrl}/sendVideo`, { method: 'POST', body: form });
    const data = await res.json();
    if (!data.ok) throw new Error(data.description);
    return { success: true, fileId: data.result.video.file_id, duration: data.result.video.duration, fileSize: data.result.video.file_size };
  }
  async getVideoUrl(fileId) {
    const res = await fetch(`${this.apiUrl}/getFile?file_id=${fileId}`);
    const data = await res.json();
    if (!data.ok) throw new Error(data.description);
    return `https://api.telegram.org/file/bot${TOKEN}/${data.result.file_path}`;
  }
}

class Bot {
  constructor() {
    this.storage = new TelegramStorage();
    this.rateLimiter = new RateLimiter();
    this.offset = 0;
  }
  async start() {
    console.log('🚀 Бот запущен!');
    while (true) {
      try {
        const url = `${this.storage.apiUrl}/getUpdates?offset=${this.offset+1}&timeout=30`;
        const res = await fetch(url);
        const data = await res.json();
        if (data.ok && data.result) {
          for (const update of data.result) {
            this.offset = update.update_id;
            const msg = update.message;
            if (!msg) continue;
            const chatId = msg.chat.id;
            const video = msg.video || msg.document;
            if (video && msg.video) {
              await this.rateLimiter.add(chatId, '✅ Ваше видео сохраняется...', 1);
              try {
                const result = await this.storage.uploadVideo(video.file_id, chatId);
                if (result.success) {
                  const videoUrl = await this.storage.getVideoUrl(result.fileId);
                  await this.rateLimiter.add(chatId,
                    `✅ Видео сохранено!\n📹 Размер: ${(result.fileSize/1024/1024).toFixed(1)} MB\n⏱️ Длительность: ${Math.floor(result.duration/60)}:${String(Math.floor(result.duration%60)).padStart(2,'0')}\n🔗 Ссылка: ${videoUrl}`,
                    0
                  );
                }
              } catch (e) {
                await this.rateLimiter.add(chatId, `❌ Ошибка: ${e.message}`, 0);
              }
            }
            if (msg.text === '/start') {
              await this.rateLimiter.add(chatId, '🤖 Привет! Отправь видео, я сохраню его.', 1);
            }
          }
        }
      } catch (e) { console.error('❌ Ошибка:', e.message); }
      await new Promise(r => setTimeout(r, 1000));
    }
  }
}
new Bot().start();