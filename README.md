<!--
Single-file demo website: Text/Voice -> Video
Features:
- Type text or speak (SpeechRecognition) -> text
- Create simple slide-based video by rendering text on canvas
- Record canvas to WebM using MediaRecorder and allow download
- 'Ad before download' simulated modal (replace with real ads like AdSense on server)
- Allow background image upload (user-provided) to avoid copyright
- Basic SEO meta tags included

Limitations:
- This is a client-side demo. For production features (mp4 conversion, large/long videos, monetization with AdSense, server-side assets), use a server or services.
- For mp4 conversion in-browser use ffmpeg.wasm (heavy) or convert server-side with ffmpeg.
--><!doctype html>

<html lang="hi">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>मेरा टेक्स्ट/वॉइस टू वीडियो</title>
  <meta name="description" content="टाइप या बोलिए और वीडियो बनाइए — मोबाइल पर काम करता साधारण डेमो" />
  <meta name="robots" content="index, follow" />
  <!-- Open Graph for social previews -->
  <meta property="og:title" content="टेक्स्ट/वॉइस टू वीडियो" />
  <meta property="og:description" content="टाइप या बोलकर अपने फोन पर वीडियो बनाएं" />
  <meta property="og:type" content="website" />
  <style>
    :root{font-family:system-ui,Segoe UI,Roboto,"Noto Sans",sans-serif}
    body{display:flex;flex-direction:column;gap:12px;align-items:center;padding:12px;max-width:980px;margin:0 auto}
    header{width:100%;text-align:center}
    textarea{width:100%;min-height:80px;padding:8px;font-size:16px}
    .controls{display:flex;gap:8px;flex-wrap:wrap;width:100%}
    button{padding:10px 14px;border-radius:8px;border:1px solid #ccc;background:#f6f6f6}
    #canvasWrap{position:relative;width:720px;max-width:100%;border:1px solid #ddd}
    canvas{width:100%;height:auto;background:#000;display:block}
    input[type=file]{display:block}
    .adModal{position:fixed;inset:0;background:rgba(0,0,0,.6);display:flex;align-items:center;justify-content:center;z-index:999;visibility:hidden}
    .adModal .card{background:#fff;padding:20px;border-radius:12px;max-width:90%;text-align:center}
    .adModal.show{visibility:visible}
    footer{font-size:13px;color:#555}
    label{display:block}
  </style>
</head>
<body>
  <header>
    <h1>टेक्स्ट/वॉइस → वीडियो (डेमो)</h1>
    <p>टाइप करें या बोलें, फिर "Generate" दबाकर वीडियो बनाएं।</p>
  </header>  <main style="width:100%">
    <div>
      <label>टेक्स्ट यहाँ लिखें:</label>
      <textarea id="textInput" placeholder="यहाँ लिखें..."></textarea>
    </div><div class="controls">
  <button id="startRec">माइक से बोलें (Voice)</button>
  <button id="stopRec" disabled>रोकें</button>
  <button id="generate">Generate Video</button>
  <button id="download" disabled>Download Video</button>
  <input id="bgImage" type="file" accept="image/*" />
  <label style="align-self:center">Slide length (sec): <input id="slideDuration" type="number" value="2" min="1" style="width:60px" /></label>
</div>

<div id="canvasWrap">
  <canvas id="videoCanvas" width="1280" height="720"></canvas>
</div>

<p>Note: यह डेमो ब्राउज़र-आधारित है — बड़े/लंबे वीडियो के लिए सर्वर या ffmpeg.wasm/सर्वर-साइड ffmpeg की सलाह।</p>

  </main>  <!-- Simulated Ad modal shown before allowing download. Replace with AdSense scripts on production pages. -->  <div id="adModal" class="adModal">
    <div class="card">
      <h3>एक छोटा सा विज्ञापन</h3>
      <p>यहां आप अपनी वास्तविक ad network (जैसे Google AdSense) की स्क्रिप्ट लगा सकते हैं।</p>
      <p id="adTimer">डाउनलोड उपलब्ध होने में: <strong id="count">5</strong> सेकंड</p>
      <button id="closeAd" disabled>Download अब</button>
    </div>
  </div>  <footer>
    <p>Created with HTML5 Canvas + MediaRecorder. Background images should be uploaded by आप (user) ताकि कॉपीराइट इश्यू कम हों।</p>
  </footer>  <script>
    // Basic behavior
    const textInput = document.getElementById('textInput')
    const startRec = document.getElementById('startRec')
    const stopRec = document.getElementById('stopRec')
    const generateBtn = document.getElementById('generate')
    const downloadBtn = document.getElementById('download')
    const canvas = document.getElementById('videoCanvas')
    const ctx = canvas.getContext('2d')
    const bgImageInput = document.getElementById('bgImage')
    const adModal = document.getElementById('adModal')
    const closeAd = document.getElementById('closeAd')
    const countEl = document.getElementById('count')
    let bgImage = null

    // Voice-to-text using Web Speech API
    let recognition
    if (window.webkitSpeechRecognition || window.SpeechRecognition) {
      const SR = window.SpeechRecognition || window.webkitSpeechRecognition
      recognition = new SR()
      recognition.lang = 'hi-IN'
      recognition.interimResults = true
      recognition.onresult = e => {
        let text = ''
        for (let i=0;i<e.results.length;i++) text += e.results[i][0].transcript
        textInput.value = text
      }
      recognition.onend = ()=>{
        startRec.disabled = false
        stopRec.disabled = true
      }
    } else {
      startRec.disabled = true
      startRec.textContent = 'Voice not supported'
    }

    startRec.onclick = ()=>{ recognition.start(); startRec.disabled=true; stopRec.disabled=false }
    stopRec.onclick = ()=>{ recognition.stop(); }

    // Load background image if uploaded
    bgImageInput.addEventListener('change', (ev)=>{
      const f = ev.target.files[0]
      if(!f) return
      const url = URL.createObjectURL(f)
      const img = new Image()
      img.onload = ()=>{ bgImage = img }
      img.src = url
    })

    // Video generation: create frames per slide and record canvas
    function drawSlide(text, tIndex, total) {
      // clear
      ctx.fillStyle = 'black'
      ctx.fillRect(0,0,canvas.width,canvas.height)
      // draw bg image if present (cover)
      if (bgImage) {
        const ar = bgImage.width/bgImage.height
        const cw = canvas.width, ch = canvas.height
        let w = cw, h = cw/ar
        if (h < ch) { h = ch; w = ch*ar }
        ctx.drawImage(bgImage, (cw-w)/2, (ch-h)/2, w, h)
        ctx.fillStyle = 'rgba(0,0,0,0.35)'
        ctx.fillRect(0,0,canvas.width,canvas.height)
      }
      // Text styling
      ctx.fillStyle='white'
      ctx.textAlign='center'
      ctx.font = 'bold 56px sans-serif'
      // wrap text
      const words = text.split(' ')
      const lines = []
      let cur = ''
      for (let w of words){
        const test = cur?cur+' '+w:w
        const width = ctx.measureText(test).width
        if (width>canvas.width*0.8){ lines.push(cur); cur=w }
        else cur=test
      }
      if (cur) lines.push(cur)
      const startY = canvas.height/2 - (lines.length-1)*34
      for (let i=0;i<lines.length;i++){
        ctx.fillText(lines[i], canvas.width/2, startY + i*68)
      }
      // footer
      ctx.font='20px sans-serif'
      ctx.fillText(`Slide ${tIndex+1}/${total}`, canvas.width/2, canvas.height-30)
    }

    async function generateVideo(){
      const text = textInput.value.trim()
      if (!text) { alert('Pehele text daalo'); return }
      const slides = text.split('\n').filter(s=>s.trim())
      // if single paragraph, split into chunks by sentence
      let finalSlides = []
      if (slides.length===1){
        const sentences = text.match(/[^\.\!\?]+[\.\!\?]?/g) || [text]
        finalSlides = sentences.map(s=>s.trim()).filter(Boolean)
      } else finalSlides = slides

      const slideDuration = Math.max(1, parseFloat(document.getElementById('slideDuration').value)||2)

      // Setup MediaRecorder
      const stream = canvas.captureStream(30) // 30fps
      let recordedChunks = []
      const mime = 'video/webm;codecs=vp9'
      const mr = new MediaRecorder(stream, {mimeType: mime})
      mr.ondataavailable = e=>{ if(e.data && e.data.size>0) recordedChunks.push(e.data) }

      mr.start()

      for (let i=0;i<finalSlides.length;i++){
        drawSlide(finalSlides[i], i, finalSlides.length)
        const frames = Math.round(slideDuration * 30)
        for (let f=0; f<frames; f++){
          // small animation tick if desired
          await new Promise(r=>requestAnimationFrame(r))
        }
      }

      // stop recorder after a tiny timeout to flush
      await new Promise(r=>setTimeout(r, 200))
      mr.stop()

      await new Promise(resolve=>{ mr.onstop = resolve })

      const blob = new Blob(recordedChunks, {type: mime})
      const url = URL.createObjectURL(blob)
      downloadBtn.href = url
      downloadBtn.download = 'my-video.webm'
      downloadBtn.dataset.blobUrl = url
      downloadBtn.dataset.blobReady = '1'
      downloadBtn.disabled = false
      alert('Video तैयार है — Download दबाइए (Ad दिखेगा)।')
    }

    generateBtn.addEventListener('click', generateVideo)

    // Simulated ad flow before download
    downloadBtn.addEventListener('click', (e)=>{
      if (downloadBtn.disabled) return
      e.preventDefault()
      // show ad modal for 5 seconds
      adModal.classList.add('show')
      let t = 5
      countEl.textContent = t
      closeAd.disabled = true
      const iv = setInterval(()=>{
        t--
        countEl.textContent = t
        if (t<=0){ clearInterval(iv); closeAd.disabled = false; countEl.textContent = 0 }
      },1000)
    })

    closeAd.addEventListener('click', ()=>{
      adModal.classList.remove('show')
      // actually download
      const url = downloadBtn.dataset.blobUrl
      if (!url) return
      const a = document.createElement('a')
      a.href = url
      a.download = downloadBtn.download || 'video.webm'
      document.body.appendChild(a)
      a.click()
      a.remove()
    })

    // Cleanup object URLs on unload
    window.addEventListener('beforeunload', ()=>{
      const url = downloadBtn.dataset.blobUrl
      if (url) URL.revokeObjectURL(url)
    })
  </script></body>
</html>
