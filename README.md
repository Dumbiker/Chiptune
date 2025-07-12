<!DOCTYPE html>
<html>
<head>
  <title>FamiLite - Chiptune Tracker</title>
  <style>
    body { background: #111; color: #0f0; font-family: monospace; padding: 20px; }
    table, td, th { border: 1px solid #0f0; border-collapse: collapse; padding: 4px; }
    input, select, button { background: #000; color: #0f0; border: 1px solid #0f0; margin: 4px; padding: 4px; }
    .channel { margin-top: 20px; }
  </style>
</head>
<body>
  <h2>🎵 FamiLite Tracker</h2>
  <label>BPM: <input type="number" id="bpm" value="120" /></label>
  <button onclick="play()">▶ Play</button>
  <button onclick="stop()">⏹ Stop</button>
  <button onclick="exportWav()">💾 Export WAV</button>

  <div id="channels"></div>

  <script>
    const notes = {
      "": 0,
      C4: 261.63, D4: 293.66, E4: 329.63,
      F4: 349.23, G4: 392.00, A4: 440.00,
      B4: 493.88, C5: 523.25
    };

    const channelNames = ["square1", "square2", "noise"];
    const steps = 16;
    const ctx = new (window.AudioContext || window.webkitAudioContext)();
    let isPlaying = false;

    // Initialize pattern editor
    const channelData = {};
    const container = document.getElementById("channels");
    channelNames.forEach(name => {
      channelData[name] = new Array(steps).fill("");
      const div = document.createElement("div");
      div.className = "channel";
      div.innerHTML = `<h3>${name.toUpperCase()}</h3><table><tr>${[...Array(steps).keys()].map(i => `<td><select onchange="updateNote('${name}', ${i}, this.value)">${Object.keys(notes).map(n => `<option value="${n}">${n}</option>`).join('')}</select></td>`).join('')}</tr></table>`;
      container.appendChild(div);
    });

    function updateNote(channel, step, note) {
      channelData[channel][step] = note;
    }

    async function play() {
      if (isPlaying) return;
      isPlaying = true;
      const bpm = parseFloat(document.getElementById("bpm").value);
      const stepDuration = 60 / bpm;

      for (let i = 0; i < steps && isPlaying; i++) {
        for (const ch of channelNames) {
          const note = channelData[ch][i];
          if (ch === "noise" && note !== "") {
            playNoise(stepDuration * 0.9);
          } else if (notes[note]) {
            playTone(notes[note], stepDuration * 0.9, ch);
          }
        }
        await new Promise(r => setTimeout(r, stepDuration * 1000));
      }

      isPlaying = false;
    }

    function stop() {
      isPlaying = false;
    }

    function playTone(freq, duration, type) {
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      osc.type = "square";
      osc.frequency.setValueAtTime(freq, ctx.currentTime);
      gain.gain.setValueAtTime(0.2, ctx.currentTime);
      osc.connect(gain).connect(ctx.destination);
      osc.start();
      osc.stop(ctx.currentTime + duration);
    }

    function playNoise(duration) {
      const bufferSize = ctx.sampleRate * duration;
      const buffer = ctx.createBuffer(1, bufferSize, ctx.sampleRate);
      const data = buffer.getChannelData(0);
      for (let i = 0; i < bufferSize; i++) {
        data[i] = Math.random() * 2 - 1;
      }
      const noise = ctx.createBufferSource();
      noise.buffer = buffer;
      const gain = ctx.createGain();
      gain.gain.setValueAtTime(0.2, ctx.currentTime);
      noise.connect(gain).connect(ctx.destination);
      noise.start();
    }

    function exportWav() {
      const sampleRate = 44100;
      const bpm = parseFloat(document.getElementById("bpm").value);
      const stepDuration = 60 / bpm;
      const totalLength = Math.floor(sampleRate * stepDuration * steps);
      const audioBuffer = new AudioBuffer({ length: totalLength, sampleRate });
      const output = audioBuffer.getChannelData(0);

      let writeIndex = 0;

      for (let i = 0; i < steps; i++) {
        const timeOffset = Math.floor(i * stepDuration * sampleRate);

        for (const ch of channelNames) {
          const note = channelData[ch][i];
          if (ch === "noise" && note !== "") {
            for (let j = 0; j < sampleRate * stepDuration * 0.9; j++) {
              const idx = timeOffset + j;
              if (idx < totalLength) output[idx] += (Math.random() * 2 - 1) * 0.1;
            }
          } else if (notes[note]) {
            const freq = notes[note];
            for (let j = 0; j < sampleRate * stepDuration * 0.9; j++) {
              const t = j / sampleRate;
              const idx = timeOffset + j;
              if (idx < totalLength)
                output[idx] += Math.sign(Math.sin(2 * Math.PI * freq * t)) * 0.1;
            }
          }
        }
      }

      // Encode WAV
      const wavData = encodeWAV([output], sampleRate);
      const blob = new Blob([wavData], { type: "audio/wav" });
      const url = URL.createObjectURL(blob);

      const a = document.createElement("a");
      a.href = url;
      a.download = "famitrack.wav";
      a.click();
    }

    // WAV encoder
    function encodeWAV(channels, sampleRate) {
      const numChannels = channels.length;
      const length = channels[0].length * numChannels * 2;
      const buffer = new ArrayBuffer(44 + length);
      const view = new DataView(buffer);

      function writeString(offset, str) {
        for (let i = 0; i < str.length; i++) view.setUint8(offset + i, str.charCodeAt(i));
      }

      writeString(0, 'RIFF');
      view.setUint32(4, 36 + length, true);
      writeString(8, 'WAVE');
      writeString(12, 'fmt ');
      view.setUint32(16, 16, true); // SubChunk1Size
      view.setUint16(20, 1, true);  // AudioFormat (PCM)
      view.setUint16(22, numChannels, true);
      view.setUint32(24, sampleRate, true);
      view.setUint32(28, sampleRate * numChannels * 2, true); // ByteRate
      view.setUint16(32, numChannels * 2, true); // BlockAlign
      view.setUint16(34, 16, true); // BitsPerSample
      writeString(36, 'data');
      view.setUint32(40, length, true);

      let offset = 44;
      for (let i = 0; i < channels[0].length; i++) {
        for (let ch = 0; ch < numChannels; ch++) {
          const sample = Math.max(-1, Math.min(1, channels[ch][i]));
          view.setInt16(offset, sample * 32767, true);
          offset += 2;
        }
      }

      return view;
    }
  </script>
</body>
</html>
