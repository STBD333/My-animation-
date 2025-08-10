# My-animation-
IT'S MY THOUGHT 
import React, { useRef, useState, useEffect } from "react";

// Simple single-file React prototype for an in-browser AI-like Keyframe Animator. // Paste into a React + Tailwind project (CodeSandbox / StackBlitz). Default export is the app. // NOTE: This is a client-side prototype. It uses interpolation, simple rule-based lipsync // heuristics and audio amplitude analysis to sync mouth openness. For full ML-quality // interpolation or phoneme-accurate lipsync, a backend with models (or WebAssembly/ONNX) // would be required.

export default function App() { const [keyframes, setKeyframes] = useState([]); const [selectedIdx, setSelectedIdx] = useState(null); const [framesPerSegment, setFramesPerSegment] = useState(12); const [style, setStyle] = useState("linear"); const [textForLipsync, setTextForLipsync] = useState(""); const [isGenerating, setIsGenerating] = useState(false); const canvasRef = useRef(null); const previewRef = useRef(null); const audioRef = useRef(null); const mediaRecorderRef = useRef(null); const recordedChunksRef = useRef([]);

// Utility to load uploaded image files function handleAddKeyframeFile(file) { const url = URL.createObjectURL(file); const kf = { id: Date.now() + Math.random(), image: url, x: 0, // px offset y: 0, rotation: 0, // degrees scale: 1, durationFrames: framesPerSegment, }; setKeyframes((s) => [...s, kf]); setSelectedIdx(keyframes.length); }

function updateKeyframe(idx, patch) { setKeyframes((s) => s.map((k, i) => (i === idx ? { ...k, ...patch } : k))); }

function removeKeyframe(idx) { setKeyframes((s) => s.filter((_, i) => i !== idx)); setSelectedIdx(null); }

// Interpolate helper function interp(a, b, t, easing = "linear") { if (easing === "ease") t = (1 - Math.cos(Math.PI * t)) / 2; // cosine ease return a + (b - a) * t; }

// Builds per-frame scene states from keyframes function buildFramesFromKeyframes(kfs) { const out = []; if (kfs.length === 0) return out; for (let i = 0; i < kfs.length - 1; i++) { const A = kfs[i]; const B = kfs[i + 1]; const steps = A.durationFrames || framesPerSegment; for (let s = 0; s < steps; s++) { const t = s / Math.max(1, steps - 1); out.push({ image: s === 0 ? A.image : null, // we'll reuse images by id when drawing x: interp(A.x, B.x, t, style), y: interp(A.y, B.y, t, style), rotation: interp(A.rotation, B.rotation, t, style), scale: interp(A.scale, B.scale, t, style), // we keep ref to A/B to look up image srcA: A.image, srcB: B.image, tBetween: t, }); } } // add final keyframe as a frame const last = kfs[kfs.length - 1]; out.push({ image: last.image, x: last.x, y: last.y, rotation: last.rotation, scale: last.scale, srcA: last.image, srcB: last.image, tBetween: 0 }); return out; }

// simple grapheme -> mouth openness heuristic for Bengali/English text function textToVisemeSeries(text, totalFrames) { // For prototype: vowels -> open mouth, consonants -> closed, punctuation -> hold const vowels = "aeiouঅআইঈউঊএঐওঔািীুূেৈোৌ"; const base = []; const simplified = text.toLowerCase().replace(/[^\wঅ-হ্া-ৌ]/g, " "); const words = simplified.split(/\s+/).filter(Boolean); if (words.length === 0) return new Array(totalFrames).fill(0); // distribute viseme events across frames const framesPerWord = Math.max(1, Math.floor(totalFrames / words.length)); let idx = 0; for (const w of words) { const vowelCount = [...w].filter((ch) => vowels.includes(ch)).length || 1; for (let f = 0; f < framesPerWord; f++) { const pos = f / Math.max(1, framesPerWord - 1); // mouth openness oscillates based on presence of vowels const openness = Math.min(1, (vowelCount / Math.max(1, w.length)) * (0.3 + 0.7 * Math.abs(Math.sin(pos * Math.PI * 2)))); base[idx++] = openness; if (idx >= totalFrames) break; } if (idx >= totalFrames) break; } // pad while (base.length < totalFrames) base.push(0); return base; }

// Analyze uploaded audio to produce amplitude envelope matching frame count async function analyzeAudioToEnvelope(file, totalFrames) { const arrayBuffer = await file.arrayBuffer(); const ctx = new (window.AudioContext || window.webkitAudioContext)(); const audioBuf = await ctx.decodeAudioData(arrayBuffer); const raw = audioBuf.getChannelData(0); const block = Math.floor(raw.length / totalFrames); const envelope = new Array(totalFrames).fill(0); for (let i = 0; i < totalFrames; i++) { let sum = 0; const start = i * block; for (let j = 0; j < block && start + j < raw.length; j++) sum += Math.abs(raw[start + j]); envelope[i] = Math.min(1, (sum / Math.max(1, block)) * 5); // scale } ctx.close(); return envelope; }

// Render frames to canvas and optionally record as WebM via MediaRecorder async function generateAndPreview({ record = false, audioFile = null }) { if (keyframes.length < 1) { alert("কমপক্ষে একটি keyframe প্রয়োজন।"); return; } setIsGenerating(true); const allFrames = buildFramesFromKeyframes(keyframes); const canvas = canvasRef.current; const ctx = canvas.getContext("2d"); const w = 720; // internal render size (scales well to mobile) const h = 720; canvas.width = w; canvas.height = h;

// preload images
const imgCache = {};
for (const kf of keyframes) {
  if (!imgCache[kf.image]) {
    const img = new Image();
    img.src = kf.image;
    await new Promise((res) => (img.onload = res));
    imgCache[kf.image] = img;
  }
}

const totalFrames = allFrames.length;

// lipsync via text and/or audio
let visemeSeries = new Array(totalFrames).fill(0);
if (textForLipsync.trim()) visemeSeries = textToVisemeSeries(textForLipsync, totalFrames);
if (audioFile) {
  const audioEnvelope = await analyzeAudioToEnvelope(audioFile, totalFrames);
  // combine: take max of text heuristic and audio envelope
  visemeSeries = visemeSeries.map((v, i) => Math.max(v, audioEnvelope[i] || 0));
}

// Setup recording if requested
recordedChunksRef.current = [];
if (record) {
  const stream = canvas.captureStream(30);
  mediaRecorderRef.current = new MediaRecorder(stream, { mimeType: "video/webm" });
  mediaRecorderRef.current.ondataavailable = (e) => { if (e.data.size) recordedChunksRef.current.push(e.data); };
  mediaRecorderRef.current.start();
}

// draw loop (synchronously draw frames)
for (let i = 0; i < totalFrames; i++) {
  ctx.clearRect(0, 0, w, h);
  // background
  ctx.fillStyle = "#f7f7fb";
  ctx.fillRect(0, 0, w, h);

  // choose image: blend between srcA and srcB based on tBetween
  const frame = allFrames[i];
  const imgA = imgCache[frame.srcA];
  const imgB = imgCache[frame.srcB];

  // compute body secondary motion (subtle wavy movement)
  const secondaryX = Math.sin(i / 6) * 2 * (frame.tBetween * 1.2 + 0.5);
  const secondaryY = Math.cos(i / 8) * 1.5;

  // draw character centered
  ctx.save();
  ctx.translate(w / 2 + frame.x + secondaryX, h / 2 + frame.y + secondaryY);
  ctx.rotate((frame.rotation * Math.PI) / 180);
  ctx.scale(frame.scale, frame.scale);

  // draw image (fit into a 360x360 box)
  const size = 360;
  if (imgA) ctx.drawImage(imgA, -size / 2, -size / 2, size, size);
  ctx.restore();

  // draw mouth overlay based on visemeSeries: a simple oval near lower center
  const openness = visemeSeries[i] || 0;
  const mouthW = 120;
  const mouthH = 12 + openness * 50; // bigger when open
  const mouthX = w / 2 - mouthW / 2;
  const mouthY = h / 2 + 80;
  ctx.fillStyle = "#2b2b2b";
  ctx.beginPath();
  ctx.ellipse(mouthX + mouthW / 2, mouthY + mouthH / 2, mouthW / 2, mouthH / 2, 0, 0, Math.PI * 2);
  ctx.fill();

  // small render delay so MediaRecorder captures frames (not realtime heavy)
  await new Promise((r) => setTimeout(r, 16));
}

if (record && mediaRecorderRef.current) {
  mediaRecorderRef.current.stop();
  await new Promise((res) => (mediaRecorderRef.current.onstop = res));
  const blob = new Blob(recordedChunksRef.current, { type: "video/webm" });
  const url = URL.createObjectURL(blob);
  previewRef.current.src = url;
} else {
  // show a static preview (capture a frame)
  previewRef.current.src = canvas.toDataURL("image/png");
}

setIsGenerating(false);

}

// small helper UI for uploading audio and images return ( <div className="min-h-screen p-4 bg-gradient-to-b from-white to-slate-50"> <div className="max-w-4xl mx-auto"> <h1 className="text-2xl font-bold mb-2">AI Keyframe Animator — Prototype</h1> <p className="mb-4 text-sm text-slate-600">Browser-only prototype: keyframe interpolation, simple lipsync (text + audio amplitude), body secondary motion, and WebM export.</p>

<div className="grid grid-cols-1 md:grid-cols-3 gap-4">
      <div className="col-span-2">
        <div className="mb-2">
          <label className="block text-sm font-medium">Add Keyframe (image)</label>
          <input type="file" accept="image/*" onChange={(e) => e.target.files?.[0] && handleAddKeyframeFile(e.target.files[0])} />
        </div>

        <div className="flex gap-2 items-center mb-2">
          <label className="text-sm">Frames per segment:</label>
          <input type="number" value={framesPerSegment} onChange={(e) => setFramesPerSegment(Number(e.target.value || 1))} className="w-24 p-1 border rounded" />
          <label className="text-sm">Easing:</label>
          <select value={style} onChange={(e) => setStyle(e.target.value)} className="p-1 border rounded">
            <option value="linear">Linear</option>
            <option value="ease">Smooth</option>
          </select>
        </div>

        <div className="mb-2">
          <label className="block text-sm font-medium">Text for Lipsync (optional)</label>
          <input className="w-full p-2 border rounded" value={textForLipsync} onChange={(e) => setTextForLipsync(e.target.value)} placeholder="Type Bengali or English text..." />
        </div>

        <div className="mb-4">
          <label className="block text-sm font-medium">Upload Voice (optional — used to shape mouth openness)</label>
          <input type="file" accept="audio/*" ref={audioRef} />
        </div>

        <div className="flex gap-2">
          <button onClick={() => generateAndPreview({ record: false, audioFile: audioRef.current?.files?.[0] || null })} className="px-4 py-2 bg-blue-600 text-white rounded">Generate Preview</button>
          <button onClick={() => generateAndPreview({ record: true, audioFile: audioRef.current?.files?.[0] || null })} className="px-4 py-2 bg-green-600 text-white rounded">Render & Export WebM</button>
        </div>

        <div className="mt-4">
          <canvas ref={canvasRef} className="w-full border rounded" style={{ width: '100%', height: 'auto' }} />
          <div className="mt-2">
            <label className="block text-sm">Preview / Download</label>
            <video ref={previewRef} controls className="w-full bg-black rounded" />
          </div>
        </div>
      </div>

      <div>
        <div className="mb-2">
          <h2 className="font-semibold">Keyframes</h2>
          <small className="text-slate-500">Tap to edit</small>
        </div>
        <div className="space-y-3">
          {keyframes.map((kf, idx) => (
            <div key={kf.id} className={`p-2 border rounded ${selectedIdx === idx ? 'ring-2 ring-blue-300' : ''}`} onClick={() => setSelectedIdx(idx)}>
              <img src={kf.image} alt="kf" className="w-full h-32 object-cover rounded-md mb-1" />
              <div className="text-xs text-slate-600">x: {kf.x} y: {kf.y} rot: {kf.rotation} scale: {kf.scale.toFixed(2)}</div>
              {selectedIdx === idx && (
                <div className="mt-2 space-y-1">
                  <label className="block text-xs">X</label>
                  <input type="range" min={-200} max={200} value={kf.x} onChange={(e) => updateKeyframe(idx, { x: Number(e.target.value) })} />
                  <label className="block text-xs">Y</label>
                  <input type="range" min={-200} max={200} value={kf.y} onChange={(e) => updateKeyframe(idx, { y: Number(e.target.value) })} />
                  <label className="block text-xs">Rotation</label>
                  <input type="range" min={-45} max={45} value={kf.rotation} onChange={(e) => updateKeyframe(idx, { rotation: Number(e.target.value) })} />
                  <label className="block text-xs">Scale</label>
                  <input type="range" min={0.5} max={2} step={0.01} value={kf.scale} onChange={(e) => updateKeyframe(idx, { scale: Number(e.target.value) })} />

                  <div className="flex gap-2 mt-2">
                    <button onClick={() => removeKeyframe(idx)} className="text-red-600 text-sm">Remove</button>
                  </div>
                </div>
              )}
            </div>
          ))}
          {keyframes.length === 0 && <div className="text-sm text-slate-500">কোনো keyframe নেই — একটি ছবি আপলোড করুন।</div>}
        </div>
      </div>
    </div>

    <div className="mt-6 text-sm text-slate-600">
      <strong>নোট:</strong> এই প্রোটোটাইপ সম্পূর্ণ ব্রাউজারে চলে। ভালো-কোয়ালিটির AI-চালিত ফ্রেম-ইনটেরপোলেশন (যেমন optical flow / frame synthesis) ও শব্দ-ভিত্তিক ফনেম-স্তরের লিপসিনের জন্য সার্ভার-সাইড ML মডেল লাগবে — আমি সেই অংশগুলোও পরে যোগ করে দেব যদি চান।
    </div>
  </div>
</div>

); }

/* ----------------------------- FULL-STACK: Backend + Library + Deployment (Appended by assistant on user's request) ----------------------------- */

/* README: Full-stack pieces added below. Paste each 'file' into your project repo.

Structure (suggested):

/ (root) /frontend    <-- React + Tailwind (existing file) src/ App.jsx   <-- main app (replace with prototype or merge) components/Library.jsx  <-- NEW library UI /backend server.js   <-- Node/Express API (auth-less for prototype) worker.py   <-- Python ML worker scaffold (ONNX/Wav2Lip/RIFE placeholders) package.json /infra docker-compose.yml   <-- for local testing (frontend, backend, worker) README_DEPLOY.md

Notes:

This scaffold is ready-to-run for prototype testing. ML model files (Wav2Lip, RIFE) must be added to /backend/models and the worker scripts must be adapted to the actual model weights (download links & instructions in README_DEPLOY.md).

For production, replace local storage with Firebase Storage or S3 and secure the API with authentication. */


/* ========================= BACKEND: server.js ========================= */

// backend/server.js // Simple Express API to accept render requests, store metadata (Mongo), and forward heavy jobs to the Python worker. // For prototype use: videos are stored on disk in /backend/storage. Replace with S3/Firebase later.

const express = require('express'); const multer = require('multer'); const { execFile } = require('child_process'); const path = require('path'); const fs = require('fs'); const { MongoClient, ObjectId } = require('mongodb');

const app = express(); app.use(express.json());

const STORAGE_DIR = path.join(__dirname, 'storage'); if (!fs.existsSync(STORAGE_DIR)) fs.mkdirSync(STORAGE_DIR);

// Multer for uploads const upload = multer({ dest: path.join(__dirname, 'uploads') });

// MongoDB - for prototype you can run local MongoDB and set MONGO_URI env var const MONGO_URI = process.env.MONGO_URI || 'mongodb://127.0.0.1:27017'; const DB_NAME = process.env.DB_NAME || 'ai_animator'; let db; MongoClient.connect(MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true }) .then((client) => { db = client.db(DB_NAME); console.log('Mongo connected'); }) .catch((err) => console.error('Mongo connect error:', err));

// POST /api/render // Accepts: keyframes JSON (small), optional audio file, optional title app.post('/api/render', upload.single('audio'), async (req, res) => { try { const { keyframes, framesPerSegment = 12, style = 'linear', textForLipsync = '', title = '' } = req.body; const audioPath = req.file ? req.file.path : null; const jobId = new ObjectId().toString(); const jobDir = path.join(STORAGE_DIR, jobId); fs.mkdirSync(jobDir);

// save keyframes JSON
fs.writeFileSync(path.join(jobDir, 'keyframes.json'), JSON.stringify({ keyframes: JSON.parse(keyframes), framesPerSegment, style, textForLipsync }, null, 2));
if (audioPath) fs.copyFileSync(audioPath, path.join(jobDir, 'audio.webm'));

// create db record
const meta = {
  title: title || `Render ${new Date().toISOString()}`,
  createdAt: new Date(),
  status: 'queued',
  jobId,
  storagePath: jobDir,
};
const result = await db.collection('renders').insertOne(meta);

// call python worker synchronously (for prototype). In production, push to job queue (Redis/RabbitMQ)
const workerScript = path.join(__dirname, 'worker.py');
execFile('python3', [workerScript, jobDir], { maxBuffer: 1024 * 1024 * 50 }, (err, stdout, stderr) => {
  if (err) {
    console.error('Worker error', err, stderr);
    db.collection('renders').updateOne({ jobId }, { $set: { status: 'failed', error: String(err) } });
  } else {
    console.log('Worker done', stdout);
    db.collection('renders').updateOne({ jobId }, { $set: { status: 'done', output: path.join(jobDir, 'output.webm') } });
  }
});

return res.json({ ok: true, jobId, dbId: result.insertedId });

} catch (e) { console.error(e); return res.status(500).json({ ok: false, error: String(e) }); } });

// GET /api/renders -> list all renders app.get('/api/renders', async (req, res) => { const rows = await db.collection('renders').find({}).sort({ createdAt: -1 }).toArray(); res.json(rows); });

// GET /api/renders/:jobId -> metadata app.get('/api/renders/:jobId', async (req, res) => { const job = await db.collection('renders').findOne({ jobId: req.params.jobId }); if (!job) return res.status(404).json({ error: 'not found' }); res.json(job); });

// GET /api/download/:jobId -> serve video file if exists app.get('/api/download/:jobId', (req, res) => { const jobDir = path.join(STORAGE_DIR, req.params.jobId); const out = path.join(jobDir, 'output.webm'); if (!fs.existsSync(out)) return res.status(404).send('Not ready'); res.sendFile(out); });

const PORT = process.env.PORT || 4000; app.listen(PORT, () => console.log('API running on', PORT));

/* ========================= BACKEND: worker.py (ML worker scaffold) ========================= */

backend/worker.py

Prototype worker that reads keyframes.json, uses the frontend interpolation approach to render frames

and invokes placeholder ML model commands (RIFE/Wav2Lip) if model files are present.

import sys import os import json import subprocess from pathlib import Path

JOBDIR = sys.argv[1] if len(sys.argv) > 1 else '.' print('Worker started for', JOBDIR)

load keyframes

with open(os.path.join(JOBDIR, 'keyframes.json'), 'r', encoding='utf-8') as f: data = json.load(f)

keyframes = data.get('keyframes', []) frames_per_segment = int(data.get('framesPerSegment', 12)) style = data.get('style', 'linear') text_for_lipsync = data.get('textForLipsync', '')

For prototype: call a Node.js headless renderer or use ffmpeg + simple transformations.

Here we'll create a placeholder (a short black video) to represent output.

output_path = os.path.join(JOBDIR, 'output.webm') print('Creating placeholder output at', output_path)

Use ffmpeg to create a 3-second black video as placeholder

cmd = [ 'ffmpeg', '-y', '-f', 'lavfi', '-i', 'color=size=720x720:rate=30:color=white', '-t', '3', '-c:v', 'libvpx-vp9', output_path ]

try: subprocess.run(cmd, check=True) print('Placeholder created') except Exception as e: print('ffmpeg failed:', e) sys.exit(1)

TODO: If you have ONNX/Wav2Lip/RIFE model files, call them here to produce real frames:

Example (conceptual):

subprocess.run(['python', 'rife_infer.py', '--input', frames_dir, '--output', rife_out_dir])

subprocess.run(['python', 'wav2lip_infer.py', '--audio', audio_path, '--video', rife_out_video, '--output', output_path])

print('Worker finished')

/* ========================= FRONTEND: Library Component ========================= */

// src/components/Library.jsx import React, { useEffect, useState } from 'react';

export default function Library({ apiBase = 'http://localhost:4000' }) { const [items, setItems] = useState([]); const [loading, setLoading] = useState(true);

async function fetchList() { setLoading(true); const res = await fetch 