// ============================================================
// 🤖 TELEGRAM БОТ — БЕСКОНЕЧНОЕ ХРАНИЛИЩЕ С CLOUDINARY
// ============================================================

const fetch = require('node-fetch');
const FormData = require('form-data');

// ============================================================
// НАСТРОЙКИ
// ============================================================

const TELEGRAM_TOKEN = '8679998760:AAFeVEtdFJbJP_NNnzIIY6yF2XhCyj7MeY8';
const CHANNEL_ID = -1004448079386;

const CLOUDINARY_CLOUD = 'c-b37bc64';
const CLOUDINARY_PRESET = 'MENTIGO_file';

// ============================================================
// ХРАНИЛИЩЕ ДАННЫХ
// ============================================================

class VideoStore {
  constructor() {
    this.videos = new Map(); // fileId -> { cloudinaryId, chatId, expiresAt, views }
    this.offset = 0;
  }

  add(fileId, cloudinaryId, chatId) {
    this.videos.set(fileId, {
      cloudinaryId,
      chatId,
      uploadedAt: Date.now(),
      expiresAt: Date.now() + 60 * 60 * 1000, // 1 час
      views: 0
    });
  }

  get(fileId) {
    return this.videos.get(fileId);
  }

  delete(fileId) {
    this.videos.delete(fileId);
  }

  isExpired(fileId) {
    const video = this.videos.get(fileId);
    if (!video) return true;
    return Date.now() > video.expiresAt;
  }

  extendLifetime(fileId) {
    const video = this.videos.get(fileId);
    if (video) {
      video.expiresAt = Date.now() + 60 * 60 * 1000;
    }
  }
}

// ============================================================
// ОЧЕРЕДЬ ЗАПРОСОВ (чтобы не перегружать Cloudinary)
// ============================================================

class VideoQueue {
  constructor() {
    this.queue = [];
    this.processing = false;
    this.maxConcurrent = 3;
  }

  async add(fileId, userId) {
    return new Promise((resolve, reject) => {
      this.queue.push({ fileId, userId, resolve, reject });
      this.process();
    });
  }

  async process() {
    if (this.processing || this.queue.length === 0) return;
    this.processing = true;

    const batch = this.queue.splice(0, this.maxConcurrent);
    const promises = batch.map(item => this.processItem(item));
    await Promise.all(promises);

    this.processing = false;
    this.process();
  }

  async processItem(item) {
    try {
      const result = await getVideoUrlFromCloudinary(item.fileId);
      item.resolve(result);
    } catch (error) {
      item.reject(error);
    }
  }
}

// ============================================================
// ОСНОВНАЯ ЛОГИКА
// ============================================================

const store = new VideoStore();
const queue = new VideoQueue();

// Получение ссылки из Cloudinary
async function getVideoUrlFromCloudinary(fileId) {
  const video = store.get(fileId);
  if (!video) {
    // Видео удалено — перезагружаем
    return await reuploadVideo(fileId);
  }

  // Обновляем время жизни
  video.expiresAt = Date.now() + 60 * 60 * 1000;
  video.views++;
  store.videos.set(fileId, video);

  // Генерируем ссылку
  const url = `https://res.cloudinary.com/${CLOUDINARY_CLOUD}/video/upload/${video.cloudinaryId}`;
  return url;
}

// Перезагрузка видео в Cloudinary
async function reuploadVideo(fileId) {
  const video = store.get(fileId);
  if (!video) throw new Error('Video not found');

  // Получаем файл из Telegram по file_id
  const fileResponse = await fetch(
    `https://api.telegram.org/bot${TELEGRAM_TOKEN}/getFile?file_id=${fileId}`
  );
  const fileData = await fileResponse.json();
  if (!fileData.ok) throw new Error('Failed to get file from Telegram');

  const filePath = fileData.result.file_path;
  const fileUrl = `https://api.telegram.org/file/bot${TELEGRAM_TOKEN}/${filePath}`;

  // Скачиваем файл
  const videoFile = await fetch(fileUrl);
  const buffer = await videoFile.buffer();

  // Загружаем в Cloudinary
  const form = new FormData();
  form.append('file', buffer, { filename: 'video.mp4' });
  form.append('upload_preset', CLOUDINARY_PRESET);

  const uploadResponse = await fetch(
    `https://api.cloudinary.com/v1_1/${CLOUDINARY_CLOUD}/video/upload`,
    { method: 'POST', body: form }
  );
  const uploadData = await uploadResponse.json();
  if (!uploadData.secure_url) throw new Error('Failed to upload to Cloudinary');

  // Обновляем хранилище
  const newCloudinaryId = uploadData.public_id;
  video.cloudinaryId = newCloudinaryId;
  video.expiresAt = Date.now() + 60 * 60 * 1000;
  store.videos.set(fileId, video);

  console.log(`🔄 Видео ${fileId} перезагружено в Cloudinary`);
  return uploadData.secure_url;
}

// Удаление видео из Cloudinary
async function deleteVideoFromCloudinary(fileId) {
  const video = store.get(fileId);
  if (!video) return;

  try {
    const response = await fetch(
      `https://api.cloudinary.com/v1_1/${CLOUDINARY_CLOUD}/resources/video/upload/${video.cloudinaryId}`,
      { method: 'DELETE' }
    );
    const data = await response.json();
    if (data.result === 'ok') {
      console.log(`🗑️ Видео ${fileId} удалено из Cloudinary`);
      store.delete(fileId);
    }
  } catch (error) {
    console.error('❌ Ошибка удаления из Cloudinary:', error);
  }
}

// Отправка сообщения пользователю
async function sendMessage(chatId, text, parseMode = 'HTML') {
  await fetch(`https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ chat_id: chatId, text, parse_mode: parseMode })
  });
}

// ============================================================
// ЗАПУСК БОТА
// ============================================================

async function main() {
  console.log('🚀 Бот запущен в режиме "Кэширующее хранилище"');
  console.log('📊 Очередь: максимум 3 одновременных запроса');

  while (true) {
    try {
      const url = `https://api.telegram.org/bot${TELEGRAM_TOKEN}/getUpdates?offset=${store.offset+1}&timeout=30`;
      const response = await fetch(url);
      const data = await response.json();

      if (!data.ok) {
        console.error('❌ Ошибка получения обновлений:', data.description);
        await new Promise(r => setTimeout(r, 5000));
        continue;
      }

      for (const update of data.result || []) {
        store.offset = update.update_id;
        const msg = update.message;
        if (!msg) continue;

        const chatId = msg.chat.id;

        // ============================================================
        // 1. ПОЛЬЗОВАТЕЛЬ ОТПРАВИЛ ВИДЕО
        // ============================================================
        if (msg.video) {
          const fileId = msg.video.file_id;
          const cloudinaryId = `video_${Date.now()}_${fileId.slice(0, 8)}`;

          // Загружаем в Cloudinary
          try {
            const fileResponse = await fetch(
              `https://api.telegram.org/bot${TELEGRAM_TOKEN}/getFile?file_id=${fileId}`
            );
            const fileData = await fileResponse.json();
            if (!fileData.ok) throw new Error('Failed to get file');

            const filePath = fileData.result.file_path;
            const fileUrl = `https://api.telegram.org/file/bot${TELEGRAM_TOKEN}/${filePath}`;

            const videoFile = await fetch(fileUrl);
            const buffer = await videoFile.buffer();

            const form = new FormData();
            form.append('file', buffer, { filename: 'video.mp4' });
            form.append('upload_preset', CLOUDINARY_PRESET);

            const uploadResponse = await fetch(
              `https://api.cloudinary.com/v1_1/${CLOUDINARY_CLOUD}/video/upload`,
              { method: 'POST', body: form }
            );
            const uploadData = await uploadResponse.json();

            if (!uploadData.secure_url) {
              throw new Error('Cloudinary upload failed');
            }

            // Сохраняем в хранилище
            store.add(fileId, uploadData.public_id, chatId);

            console.log(`📥 Видео загружено: ${fileId} → ${uploadData.public_id}`);

            // Отправляем подтверждение
            await sendMessage(
              chatId,
              `✅ <b>Видео загружено!</b>\n\n` +
              `📹 Нажми на видео, чтобы посмотреть.\n` +
              `⏱️ Ссылка будет активна 1 час.\n` +
              `🔄 После просмотра видео удалится через 5 минут.`
            );

          } catch (error) {
            console.error('❌ Ошибка загрузки видео:', error);
            await sendMessage(chatId, `❌ Не удалось загрузить видео: ${error.message}`);
          }
        }

        // ============================================================
        // 2. ПОЛЬЗОВАТЕЛЬ ЗАПРОСИЛ ВИДЕО (через мессенджер)
        // ============================================================
        if (msg.text && msg.text.startsWith('/getvideo_')) {
          const fileId = msg.text.replace('/getvideo_', '');

          try {
            // Добавляем запрос в очередь
            const videoUrl = await queue.add(fileId, chatId);

            // Отправляем ссылку (мессенджер подставит её в плеер)
            await sendMessage(
              chatId,
              `🎬 <b>Видео готово!</b>\n\n` +
              `🔗 Ссылка активна 5 минут.\n` +
              `⏱️ Ссылка: ${videoUrl}\n` +
              `⚠️ Видео будет удалено через 5 минут.`
            );

            // Устанавливаем таймер на удаление через 5 минут
            setTimeout(async () => {
              await deleteVideoFromCloudinary(fileId);
              console.log(`🗑️ Видео ${fileId} удалено через 5 минут`);
            }, 5 * 60 * 1000);

          } catch (error) {
            console.error('❌ Ошибка получения видео:', error);
            await sendMessage(chatId, `❌ Видео недоступно: ${error.message}`);
          }
        }

        // ============================================================
        // 3. КОМАНДА /START
        // ============================================================
        if (msg.text === '/start') {
          await sendMessage(
            chatId,
            `🤖 <b>Привет! Я бот для хранения видео.</b>\n\n` +
            `📤 Отправь мне видео, и оно появится в мессенджере.\n` +
            `🔄 Видео будет удалено через 5 минут после просмотра.\n` +
            `♾️ Ты можешь пересматривать его сколько угодно.`
          );
        }
      }
    } catch (error) {
      console.error('❌ Ошибка в основном цикле:', error);
      await new Promise(r => setTimeout(r, 5000));
    }
  }
}

// ============================================================
// ЗАПУСК
// ============================================================

main().catch(console.error);
console.log('✅ Бот готов к работе!');