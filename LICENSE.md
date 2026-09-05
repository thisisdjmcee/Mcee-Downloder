// ============================================================
//  MCEE TOPSTAR - MEDIA DOWNLOADER SERVER
//  Inatumia yt-dlp kupakua video/audio kutoka YouTube, TikTok,
//  Instagram, Facebook na X (Twitter).
// ============================================================

const express = require('express');
const cors = require('cors');
const { spawn } = require('child_process');
const path = require('path');
const fs = require('fs');
const crypto = require('crypto');

const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors());
app.use(express.json());

// Folder ya muda ya kutunza faili kabla ya kuzipeleka kwa mtumiaji
const TMP_DIR = path.join(__dirname, 'tmp');
if (!fs.existsSync(TMP_DIR)) fs.mkdirSync(TMP_DIR);

// Safisha faili za zamani (zaidi ya dakika 20) kila baada ya dakika 10
setInterval(() => {
  fs.readdir(TMP_DIR, (err, files) => {
    if (err) return;
    const now = Date.now();
    files.forEach(file => {
      const filePath = path.join(TMP_DIR, file);
      fs.stat(filePath, (err, stats) => {
        if (err) return;
        if (now - stats.mtimeMs > 20 * 60 * 1000) {
          fs.unlink(filePath, () => {});
        }
      });
    });
  });
}, 10 * 60 * 1000);

// ------------------------------------------------------------
// Endpoint: /api/info  -> Pata taarifa za video (jina, cover, muda)
// ------------------------------------------------------------
app.post('/api/info', (req, res) => {
  const { url } = req.body;
  if (!url) return res.status(400).json({ error: 'Weka URL sahihi.' });

  const ytdlp = spawn('yt-dlp', [
    '--dump-json',
    '--no-playlist',
    url
  ]);

  let data = '';
  let errorData = '';

  ytdlp.stdout.on('data', chunk => { data += chunk; });
  ytdlp.stderr.on('data', chunk => { errorData += chunk; });

  ytdlp.on('close', code => {
    if (code !== 0) {
      console.error(errorData);
      return res.status(500).json({ error: 'Imeshindwa kupata taarifa za video. Hakikisha link ni sahihi.' });
    }
    try {
      const info = JSON.parse(data);
      res.json({
        title: info.title,
        thumbnail: info.thumbnail,
        duration: info.duration,
        uploader: info.uploader,
        platform: info.extractor_key
      });
    } catch (e) {
      res.status(500).json({ error: 'Imeshindwa kusoma taarifa za video.' });
    }
  });
});

// ------------------------------------------------------------
// Endpoint: /api/download -> Pakua faili halisi na uipeleke
// format: "mp3" (audio pekee) au "mp4" (video)
// ------------------------------------------------------------
app.post('/api/download', (req, res) => {
  const { url, format } = req.body;
  if (!url) return res.status(400).json({ error: 'Weka URL sahihi.' });

  const id = crypto.randomBytes(8).toString('hex');
  const outputTemplate = path.join(TMP_DIR, `${id}.%(ext)s`);

  let args;
  if (format === 'mp3') {
    args = [
      '-x', '--audio-format', 'mp3',
      '--no-playlist',
      '-o', outputTemplate,
      url
    ];
  } else {
    args = [
      '-f', 'best[ext=mp4]/best',
      '--no-playlist',
      '-o', outputTemplate,
      url
    ];
  }

  const ytdlp = spawn('yt-dlp', args);
  let errorData = '';

  ytdlp.stderr.on('data', chunk => { errorData += chunk; });

  ytdlp.on('close', code => {
    if (code !== 0) {
      console.error(errorData);
      return res.status(500).json({ error: 'Imeshindwa kupakua. Jaribu tena au angalia link.' });
    }

    // Tafuta faili iliyotengenezwa (jina lina ID yetu)
    fs.readdir(TMP_DIR, (err, files) => {
      if (err) return res.status(500).json({ error: 'Hitilafu ya server.' });
      const match = files.find(f => f.startsWith(id));
      if (!match) return res.status(500).json({ error: 'Faili haikupatikana baada ya kupakua.' });

      const filePath = path.join(TMP_DIR, match);
      res.download(filePath, match, err => {
        // Futa faili baada ya kuipeleka kwa mtumiaji
        if (!err) fs.unlink(filePath, () => {});
      });
    });
  });
});

app.get('/', (req, res) => {
  res.send('Mcee Topstar Downloader Server iko hai ✅');
});

app.listen(PORT, () => {
  console.log(`Server inaendesha kwenye port ${PORT}`);
});
