
const express = require('express');
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;
const session = require('express-session');
const dotenv = require('dotenv');
const { google } = require('googleapis');
dotenv.config();

const app = express();
app.use(express.json());
app.use(session({ secret: process.env.SESSION_SECRET, resave: false, saveUninitialized: true }));
app.use(passport.initialize());
app.use(passport.session());

passport.use(new GoogleStrategy({
  clientID: process.env.GOOGLE_CLIENT_ID,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET,
  callbackURL: '/auth/google/callback'
}, (accessToken, refreshToken, profile, done) => {
  return done(null, { profile, accessToken });
}));

passport.serializeUser((user, done) => done(null, user));
passport.deserializeUser((obj, done) => done(null, obj));

app.get('/auth/google', passport.authenticate('google', { scope: ['profile', 'email', 'https://www.googleapis.com/auth/drive.file'] }));
app.get('/auth/google/callback', passport.authenticate('google', { failureRedirect: '/' }), (req, res) => {
  res.redirect('/dashboard');
});

const ensureAuthenticated = (req, res, next) => {
  if (req.isAuthenticated()) return next();
  res.status(401).json({ error: 'Unauthorized' });
};

app.post('/save-letter', ensureAuthenticated, async (req, res) => {
  const { accessToken } = req.user;
  const { title, content } = req.body;
  const drive = google.drive({ version: 'v3', auth: accessToken });

  try {
    const file = await drive.files.create({
      resource: { name: `${title}.txt`, mimeType: 'text/plain' },
      media: { mimeType: 'text/plain', body: content },
      fields: 'id'
    });
    res.json({ success: true, fileId: file.data.id });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.get('/get-letters', ensureAuthenticated, async (req, res) => {
  const { accessToken } = req.user;
  const drive = google.drive({ version: 'v3', auth: accessToken });
  
  try {
    const response = await drive.files.list({ q: "mimeType='text/plain'", fields: 'files(id, name)' });
    res.json({ success: true, files: response.data.files });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

import React, { useState, useEffect } from 'react';
import ReactQuill from 'react-quill';
import 'react-quill/dist/quill.snow.css';
import axios from 'axios';

const LetterEditor = () => {
  const [content, setContent] = useState('');
  const [title, setTitle] = useState('');
  const [letters, setLetters] = useState([]);

  const saveLetter = async () => {
    try {
      const response = await axios.post('/save-letter', { title, content });
      alert(`Letter saved! File ID: ${response.data.fileId}`);
      fetchLetters();
    } catch (error) {
      alert('Error saving letter: ' + error.message);
    }
  };

  const fetchLetters = async () => {
    try {
      const response = await axios.get('/get-letters');
      setLetters(response.data.files);
    } catch (error) {
      alert('Error fetching letters: ' + error.message);
    }
  };

  useEffect(() => {
    fetchLetters();
  }, []);

  return (
    <div>
      <input type="text" placeholder="Title" value={title} onChange={(e) => setTitle(e.target.value)} />
      <ReactQuill value={content} onChange={setContent} />
      <button onClick={saveLetter}>Save to Google Drive</button>
      <h2>Saved Letters</h2>
      <ul>
        {letters.map(letter => (<li key={letter.id}>{letter.name}</li>))}
      </ul>
    </div>
  );
};

export default LetterEditor;

app.listen(5000, () => console.log('Server running on port 5000'));
