package.json
{
  "name": "telepost-to-wp",
  "version": "1.0.0",
  "description": "Telegram to WordPress Auto Poster",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "axios": "^1.6.2",
    "body-parser": "^1.20.2",
    "express": "^4.18.2",
    "form-data": "^4.0.0"
  }
}
.env
TELEGRAM_BOT_TOKEN=7686028873:AAHf0cY3WdUgwh4Ig62-S_ABGBB4icsZTPM
WP_JWT_TOKEN=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwczovL2VuZXJneXBhdGhuaXIiLCJpYXQiOjE3NDYzNDUyNjEsIm5iZiI6MTc0NjM0NTI2MSwiZXhwIjoxNzQ2OTUwMDYxLCJkYXRhIjp7InVzZXIiOnsiaWQiOiIxIn19fQ.D_qjo1_EZ-yW9V23noWDIRbAtAAGQt2N3JGg8RE3_jw
const express = require('express');
const axios = require('axios');
const bodyParser = require('body-parser');
const FormData = require('form-data');

// ==== تنظیمات ====
const TELEGRAM_BOT_TOKEN = process.env.TELEGRAM_BOT_TOKEN;
const WP_SITE_URL = 'https://energypath.ir';
const WP_JWT_TOKEN = process.env.WP_JWT_TOKEN;

const DEFAULT_CATEGORY_ID = 10; // دسته‌بندی مورد نظر
const DEFAULT_TAGS = [15];     // تگ‌های پیش‌فرض

// ==== توابع کمکی ====
async function extractTitle(content) {
  return content.split('\n')[0].trim() || 'خبر جدید از تلگرام';
}

async function extractExcerpt(content) {
  const words = content.split(' ');
  return words.slice(0, 50).join(' ') + '...';
}

async function getFileUrl(file_id) {
  try {
    const res = await axios.get(`https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getFile?file_id=${file_id}`);
    return `https://api.telegram.org/file/bot${TELEGRAM_BOT_TOKEN}/${res.data.result.file_path}`;
  } catch (err) {
    console.error('❌ خطا در دریافت URL فایل:', err.message);
    return null;
  }
}

async function uploadImageToWordPress(imageUrl) {
  try {
    const response = await axios.get(imageUrl, { responseType: 'arraybuffer' });
    const form = new FormData();
    const buffer = Buffer.from(response.data, 'binary');

    form.append('file', buffer, {
      filename: `telepost-${Date.now()}.jpg`,
      contentType: 'image/jpeg'
    });

    const uploadRes = await axios.post(`${WP_SITE_URL}/wp-json/wp/v2/media`, form, {
      headers: {
        ...form.getHeaders(),
        Authorization: `Bearer ${WP_JWT_TOKEN}`
      }
    });

    return uploadRes.data.id;
  } catch (err) {
    console.error('❌ خطا در آپلود عکس:', err.message);
    return null;
  }
}

async function createPost(title, content, excerpt, featured_media, category, tags) {
  const post = {
    title,
    content,
    excerpt,
    status: 'publish',
    categories: category,
    tags: tags,
    ...(featured_media && { featured_media })
  };

  try {
    await axios.post(`${WP_SITE_URL}/wp-json/wp/v2/posts`, post, {
      headers: {
        Authorization: `Bearer ${WP_JWT_TOKEN}`,
        'Content-Type': 'application/json'
      }
    });
    console.log('✅ پست با موفقیت منتشر شد.');
  } catch (error) {
    console.error('❌ خطا در انتشار پست:', error.response?.data || error.message);
  }
}

// ==== تنظیمات اصلی برنامه ====
const app = express();
app.use(bodyParser.json());

// ==== دریافت پیام از تلگرام ====
app.post('/telegram-webhook', async (req, res) => {
  const update = req.body;

  if (update.message && update.message.chat.username === 'energypath_ir') {
    try {
      const content = update.message.text || update.message.caption || '';
      if (!content || content.length < 20) return res.sendStatus(200);

      const title = await extractTitle(content);
      const excerpt = await extractExcerpt(content);

      let featuredMediaId = null;
      if (update.message.photo) {
        const photo = update.message.photo[update.message.photo.length - 1];
        const imageUrl = await getFileUrl(photo.file_id);
        if (imageUrl) {
          featuredMediaId = await uploadImageToWordPress(imageUrl);
        }
      }

      await createPost(title, content, excerpt, featuredMediaId, DEFAULT_CATEGORY_ID, DEFAULT_TAGS);
    } catch (err) {
      console.error('❌ خطا در پردازش پیام:', err.message);
    }
  }

  res.sendStatus(200);
});

// ==== تنظیم وب هوک در تلگرام ====
async function setWebhook() {
  try {
    const projectName = process.env.PROJECT_NAME || 'your-project-name';
    const webhookUrl = `https://${projectName}.up.railway.app/telegram-webhook`;

    // حذف وب هوک قبلی
    await axios.get(`https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/deleteWebhook`);

    // تنظیم وب هوک جدید
    await axios.get(`https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setWebhook?url=${webhookUrl}`);

    console.log(`✅ وب هوک تنظیم شد: ${webhookUrl}`);
  } catch (err) {
    console.error('❌ خطا در تنظیم وب هوک:', err.message);
  }
}

// ==== روت تستی ====
app.get('/', (req, res) => {
  res.send('✅ سرور در حال اجرا است. منتظر پیام‌های تلگرام هست...');
});

// ==== اجرای سرور ====
const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`🚀 سرور در حال اجرا است.`);
});

// جدا از app.listen
setWebhook();
