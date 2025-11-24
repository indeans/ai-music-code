import React, { useState, useRef, useEffect } from 'react';
import { Upload, Play, Pause, Music, FileAudio, RefreshCw, Volume2, Sparkles, PenTool, BookOpen, Mic2, AlertCircle, Loader2, History, Trash2 } from 'lucide-react';

const NOTE_STRINGS = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"];

// ----------------------------------------------------------------------
// Audio Analysis Helpers
// ----------------------------------------------------------------------

const getNoteFromFrequency = (frequency) => {
  const noteNum = 12 * (Math.log(frequency / 440) / Math.log(2));
  const midi = Math.round(noteNum) + 69;
  const note = NOTE_STRINGS[midi % 12];
  const octave = Math.floor(midi / 12) - 1;
  const cents = Math.floor((noteNum - Math.round(noteNum)) * 100);
  return { note, octave, cents, midi };
};

const autoCorrelate = (buf, sampleRate) => {
  let SIZE = buf.length;
  let rms = 0;

  for (let i = 0; i < SIZE; i++) {
    const val = buf[i];
    rms += val * val;
  }
  rms = Math.sqrt(rms / SIZE);
  if (rms < 0.01) return -1;

  let r1 = 0, r2 = SIZE - 1, thres = 0.2;
  for (let i = 0; i < SIZE / 2; i++) {
    if (Math.abs(buf[i]) < thres) { r1 = i; break; }
  }
  for (let i = 1; i < SIZE / 2; i++) {
    if (Math.abs(buf[SIZE - i]) < thres) { r2 = SIZE - i; break; }
  }

  buf = buf.slice(r1, r2);
  SIZE = buf.length;

  let c = new Array(SIZE).fill(0);
  for (let i = 0; i < SIZE; i++) {
    for (let j = 0; j < SIZE - i; j++) {
      c[i] = c[i] + buf[j] * buf[j + i];
    }
  }

  let d = 0;
  while (c[d] > c[d + 1]) d++;
  let maxval = -1, maxpos = -1;
  for (let i = d; i < SIZE; i++) {
    if (c[i] > maxval) {
      maxval = c[i];
      maxpos = i;
    }
  }
  let T0 = maxpos;
  let x1 = c[T0 - 1], x2 = c[T0], x3 = c[T0 + 1];
  let a = (x1 + x3 - 2 * x2) / 2;
  let b = (x3 - x1) / 2;
  if (a) T0 = T0 - b / (2 * a);

  return sampleRate / T0;
};

// ----------------------------------------------------------------------
// Main Component
// ----------------------------------------------------------------------

export default function MusicTranscriber() {
  // Audio State
  const [audioContext, setAudioContext] = useState(null);
  const [sourceNode, setSourceNode] = useState(null);
  const [isPlaying, setIsPlaying] = useState(false);
  const [fileName, setFileName] = useState("");
  const [currentNote, setCurrentNote] = useState({ note: "-", octave: "", cents: 0 });
  const [errorMessage, setErrorMessage] = useState("");
  const [isAudioLoaded, setIsAudioLoaded] = useState(false);
  const [isDecoding, setIsDecoding] = useState(false);
  
  // Note History State
  const [noteHistory, setNoteHistory] = useState([]);
  const lastNoteRef = useRef(null); // To prevent duplicate consecutive notes
  
  // AI Feature State
  const [songMood, setSongMood] = useState("");
  const [aiResponse, setAiResponse] = useState("");
  const [isAiLoading, setIsAiLoading] = useState(false);
  const [aiError, setAiError] = useState("");

  const audioRef = useRef(null);
  const analyserRef = useRef(null);
  const requestRef = useRef(null);
  const canvasRef = useRef(null);
  const bufferRef = useRef(null);

  useEffect(() => {
    return () => {
      if (audioContext) audioContext.close();
      cancelAnimationFrame(requestRef.current);
    };
  }, []);

  const handleFileUpload = async (event) => {
    const file = event.target.files[0];
    if (!file) return;

    setFileName(file.name);
    setErrorMessage("");
    setIsPlaying(false);
    setIsAudioLoaded(false);
    setIsDecoding(true);
    setNoteHistory([]); // Clear history on new file
    lastNoteRef.current = null;
    
    try {
      const ctx = new (window.AudioContext || window.webkitAudioContext)();
      setAudioContext(ctx);

      const arrayBuffer = await file.arrayBuffer();
      const audioBuffer = await ctx.decodeAudioData(arrayBuffer);
      bufferRef.current = audioBuffer;
      setIsAudioLoaded(true);
    } catch (error) {
      console.error("Audio decode error:", error);
      setErrorMessage("파일을 분석할 수 없습니다. 오디오 파일이 손상되었거나 지원하지 않는 형식일 수 있습니다.");
      setFileName("");
      setIsAudioLoaded(false);
    } finally {
      setIsDecoding(false);
    }
  };

  const startAnalysis = async () => {
    if (!audioContext || !bufferRef.current) return;
    
    if (audioContext.state === 'suspended') {
      try {
        await audioContext.resume();
      } catch (err) {
        console.error("Failed to resume audio context", err);
        setErrorMessage("오디오 재생을 시작할 수 없습니다. 브라우저 설정을 확인해주세요.");
        return;
      }
    }

    if (sourceNode) sourceNode.disconnect();
    
    const source = audioContext.createBufferSource();
    source.buffer = bufferRef.current;
    
    const analyser = audioContext.createAnalyser();
    analyser.fftSize = 2048;
    
    source.connect(analyser);
    analyser.connect(audioContext.destination);
    
    analyserRef.current = analyser;
    setSourceNode(source);
    
    source.start(0);
    setIsPlaying(true);
    source.onended = () => setIsPlaying(false);
    
    visualize();
  };

  const stopAnalysis = () => {
    if (sourceNode) {
      try {
        sourceNode.stop();
      } catch (e) {
        // Ignore errors if already stopped
      }
      setIsPlaying(false);
    }
    cancelAnimationFrame(requestRef.current);
  };

  const visualize = () => {
    if (!analyserRef.current || !canvasRef.current) return;
    const canvas = canvasRef.current;
    const canvasCtx = canvas.getContext('2d');
    const buffer = new Float32Array(analyserRef.current.fftSize);

    // Throttle history updates to avoid performance issues
    let frameCount = 0;

    const draw = () => {
      requestRef.current = requestAnimationFrame(draw);
      frameCount++;

      analyserRef.current.getFloatTimeDomainData(buffer);
      const frequency = autoCorrelate(buffer, audioContext.sampleRate);
      
      const width = canvas.width;
      const height = canvas.height;
      const imageData = canvasCtx.getImageData(2, 0, width - 2, height);
      canvasCtx.putImageData(imageData, 0, 0);
      
      canvasCtx.fillStyle = '#111827';
      canvasCtx.fillRect(width - 2, 0, 2, height);

      if (frequency !== -1 && frequency > 60 && frequency < 2000) {
        const noteData = getNoteFromFrequency(frequency);
        setCurrentNote(noteData);

        // History Logic: Add note if it's different from the last one
        // and only update every few frames to reduce jitter
        const noteString = `${noteData.note}${noteData.octave}`;
        if (frameCount % 5 === 0 && lastNoteRef.current !== noteString) {
          lastNoteRef.current = noteString;
          setNoteHistory(prev => {
            const newHistory = [...prev, noteString];
            return newHistory.slice(-20); // Keep last 20 notes
          });
        }

        const minMidi = 36;
        const maxMidi = 84;
        const normalize = (noteData.midi - minMidi) / (maxMidi - minMidi);
        const y = height - (normalize * height);

        canvasCtx.fillStyle = '#60A5FA';
        canvasCtx.fillRect(width - 4, y, 4, 4);
        
        canvasCtx.fillStyle = 'rgba(255, 255, 255, 0.05)';
        canvasCtx.fillRect(width - 2, 0, 2, height);
      } else {
        // Don't reset currentNote immediately to avoid flickering UI
        // setCurrentNote({ note: "-", octave: "", cents: 0 });
      }
    };
    draw();
  };

  const clearHistory = () => {
    setNoteHistory([]);
    lastNoteRef.current = null;
  };

  // ----------------------------------------------------------------------
  // Gemini API Integration
  // ----------------------------------------------------------------------

  const callGemini = async (promptType) => {
    if (!songMood) {
      setAiError("곡의 분위기나 장르를 먼저 입력해주세요!");
      return;
    }

    setIsAiLoading(true);
    setAiError("");
    setAiResponse("");

    const apiKey = ""; 
    const model = "gemini-2.5-flash-preview-09-2025";
    const url = `https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent?key=${apiKey}`;

    let systemInstruction = "";
    if (promptType === 'coach') {
      systemInstruction = `당신은 전문적인 음악 선생님이자 연주 코치입니다. 사용자가 입력한 음악 스타일에 맞춰 다음 내용을 한국어로 제공하세요:
      1. 추천 연습 스케일 (Scale) 및 아르페지오
      2. 이 장르를 연주할 때 중요한 테크닉 팁
      3. 감정 표현을 위한 조언
      간결하고 격려하는 어조로 작성하세요.`;
    } else if (promptType === 'lyrics') {
      systemInstruction = `당신은 감성적인 작사가이자 아트 디렉터입니다. 사용자가 입력한 음악 분위기에 맞춰 다음 내용을 한국어로 제공하세요:
      1. 이 곡에 어울리는 짧은 가사 (Verse 1, Chorus)
      2. 이 곡의 앨범 커버를 위한 시각적 묘사/아이디어
      시적이고 창의적인 어조로 작성하세요.`;
    }

    const payload = {
      contents: [{ parts: [{ text: `이 곡의 분위기/장르: ${songMood}` }] }],
      systemInstruction: { parts: [{ text: systemInstruction }] }
    };

    const fetchWithRetry = async (url, options, retries = 5, delay = 1000) => {
      try {
        const response = await fetch(url, options);
        if (!response.ok) {
           if (retries > 0 && (response.status === 429 || response.status >= 500)) {
              throw new Error("Retryable error");
           }
           throw new Error(`API Error: ${response.status}`);
        }
        return response;
      } catch (error) {
        if (retries === 0) throw error;
        await new Promise(resolve => setTimeout(resolve, delay));
        return fetchWithRetry(url, options, retries - 1, delay * 2);
      }
    };

    try {
      const response = await fetchWithRetry(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });

      const data = await response.json();
      const text = data.candidates?.[0]?.content?.parts?.[0]?.text || "응답을 생성하지 못했습니다.";
      setAiResponse(text);
    } catch (error) {
      console.error(error);
      setAiError("AI 응답을 가져오는 중 오류가 발생했습니다. 잠시 후 다시 시도해주세요.");
    } finally {
      setIsAiLoading(false);
    }
  };

  return (
    <div className="min-h-screen bg-gray-900 text-white p-4 font-sans">
      <div className="max-w-4xl mx-auto space-y-8">
        
        {/* Header */}
        <div className="text-center py-6 border-b border-gray-800">
          <h1 className="text-3xl font-bold bg-gradient-to-r from-blue-400 to-purple-500 bg-clip-text text-transparent flex items-center justify-center gap-2">
            <Music className="w-8 h-8 text-blue-400" />
            AI 음악 스튜디오
          </h1>
          <p className="text-gray-400 mt-2">MP3 분석부터 AI 연주 코칭까지 한번에</p>
        </div>

        {/* 1. Analyzer Section */}
        <div className="bg-gray-800 rounded-xl p-6 shadow-xl border border-gray-700">
          <h2 className="text-xl font-bold mb-4 flex items-center gap-2 text-white">
            <Volume2 className="w-5 h-5 text-green-400" /> 
            오디오 분석기
          </h2>
          
          <div className="flex flex-col md:flex-row items-center justify-between gap-4 mb-6">
            <label className="flex items-center gap-2 cursor-pointer bg-gray-700 hover:bg-gray-600 text-white py-2 px-4 rounded-lg transition-colors border border-gray-600 w-full md:w-auto justify-center">
              <Upload className="w-5 h-5" />
              <span className="truncate max-w-[200px]">{fileName || "음악 파일 선택 (MP3/WAV)"}</span>
              <input type="file" accept="audio/*" onChange={handleFileUpload} className="hidden" />
            </label>

            <div className="flex items-center gap-4 w-full md:w-auto justify-center">
              {!isPlaying ? (
                <button 
                  onClick={startAnalysis} 
                  disabled={!isAudioLoaded || isDecoding}
                  className={`flex items-center gap-2 py-2 px-6 rounded-lg font-bold transition-all w-full md:w-auto justify-center 
                    ${(!isAudioLoaded || isDecoding) 
                      ? 'bg-gray-700 text-gray-500 cursor-not-allowed' 
                      : 'bg-blue-500 hover:bg-blue-600 text-white shadow-lg shadow-blue-500/30'}`}
                >
                  {isDecoding ? (
                    <><Loader2 className="w-5 h-5 animate-spin" /> 로딩 중...</>
                  ) : (
                    <><Play className="w-5 h-5" /> 분석 시작</>
                  )}
                </button>
              ) : (
                <button 
                  onClick={stopAnalysis}
                  className="flex items-center gap-2 py-2 px-6 rounded-lg font-bold bg-red-500 hover:bg-red-600 text-white shadow-lg shadow-red-500/30 transition-all w-full md:w-auto justify-center"
                >
                  <Pause className="w-5 h-5" /> 정지
                </button>
              )}
            </div>
          </div>
          
          {/* General Error Message Display */}
          {errorMessage && (
             <div className="mb-4 bg-red-500/10 border border-red-500/40 text-red-200 p-3 rounded-lg flex items-center gap-2 text-sm">
                <AlertCircle className="w-4 h-4" /> {errorMessage}
             </div>
          )}

          <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
            {/* Note Display */}
            <div className="bg-gray-900 rounded-lg p-6 flex flex-col items-center justify-center border border-gray-700 aspect-video relative overflow-hidden">
               <div className="absolute top-2 left-3 text-gray-500 text-sm">실시간 계이름</div>
               <div className="text-center z-10">
                 <div className="text-7xl font-black text-white tracking-tighter transition-all duration-75" style={{ textShadow: '0 0 20px rgba(59, 130, 246, 0.5)' }}>
                   {currentNote.note}
                   <span className="text-4xl text-blue-400 ml-1">{currentNote.octave}</span>
                 </div>
                 <div className="mt-4 flex items-center justify-center gap-2">
                   <div className="w-32 h-2 bg-gray-700 rounded-full overflow-hidden relative">
                     <div 
                        className={`absolute top-0 bottom-0 w-1 transition-all duration-100 ${Math.abs(currentNote.cents) < 10 ? 'bg-green-500' : 'bg-yellow-500'}`}
                        style={{ left: `${50 + (currentNote.cents / 2)}%` }}
                     />
                   </div>
                   <span className="text-xs text-gray-400 w-8 text-right">{currentNote.cents > 0 ? '+' : ''}{currentNote.cents}</span>
                 </div>
               </div>
            </div>

            {/* Canvas */}
            <div className="bg-gray-900 rounded-lg p-1 border border-gray-700 aspect-video relative">
              <div className="absolute top-2 left-3 text-gray-500 text-sm z-10 pointer-events-none">멜로디 트래커</div>
              <canvas ref={canvasRef} width={400} height={200} className="w-full h-full rounded bg-gray-950" />
            </div>
          </div>

          {/* New: Note History Section */}
          <div className="mt-4 bg-gray-900 rounded-lg p-4 border border-gray-700">
            <div className="flex items-center justify-between mb-2">
              <h3 className="text-sm font-bold text-gray-400 flex items-center gap-2">
                <History className="w-4 h-4" /> 감지된 멜로디 기록 (최근 20개)
              </h3>
              <button onClick={clearHistory} className="text-xs text-red-400 hover:text-red-300 flex items-center gap-1">
                <Trash2 className="w-3 h-3" /> 기록 삭제
              </button>
            </div>
            <div className="flex flex-wrap gap-2 min-h-[40px] items-center">
              {noteHistory.length === 0 ? (
                <span className="text-gray-600 text-sm italic">재생하면 계이름이 여기에 기록됩니다...</span>
              ) : (
                noteHistory.map((note, idx) => (
                  <div key={idx} className="bg-gray-800 text-blue-400 px-3 py-1 rounded-full text-sm font-bold border border-gray-700 animate-fade-in">
                    {note}
                  </div>
                ))
              )}
            </div>
          </div>
        </div>

        {/* 2. Gemini AI Features Section */}
        <div className="bg-gradient-to-br from-indigo-900 to-purple-900 rounded-xl p-6 shadow-xl border border-indigo-700">
          <div className="flex items-center gap-2 mb-4">
            <Sparkles className="w-6 h-6 text-yellow-300" />
            <h2 className="text-xl font-bold text-white">Gemini AI 크리에이티브 어시스턴트</h2>
          </div>
          
          <p className="text-indigo-200 mb-6 text-sm">
            지금 연습 중인 곡의 분위기나 장르를 입력하고 AI의 도움을 받아보세요. 
            연주 팁부터 가사 아이디어까지 제공합니다.
          </p>

          <div className="space-y-4">
            <div>
              <label className="block text-sm font-medium text-indigo-200 mb-1">
                어떤 곡인가요? (예: 슬픈 재즈 발라드, 신나는 펑크 록, 차분한 피아노 연주곡)
              </label>
              <input 
                type="text" 
                value={songMood}
                onChange={(e) => setSongMood(e.target.value)}
                placeholder="곡의 장르나 분위기를 입력하세요..."
                className="w-full bg-indigo-950/50 border border-indigo-600 rounded-lg px-4 py-3 text-white placeholder-indigo-400 focus:outline-none focus:ring-2 focus:ring-yellow-400 transition-all"
              />
            </div>

            <div className="flex flex-col sm:flex-row gap-3">
              <button 
                onClick={() => callGemini('coach')}
                disabled={isAiLoading}
                className="flex-1 flex items-center justify-center gap-2 bg-indigo-600 hover:bg-indigo-500 text-white py-3 px-4 rounded-lg font-bold transition-all shadow-lg shadow-indigo-900/50 disabled:opacity-50"
              >
                {isAiLoading ? <RefreshCw className="w-5 h-5 animate-spin" /> : <BookOpen className="w-5 h-5" />}
                ✨ AI 연주 코치
              </button>
              
              <button 
                onClick={() => callGemini('lyrics')}
                disabled={isAiLoading}
                className="flex-1 flex items-center justify-center gap-2 bg-purple-600 hover:bg-purple-500 text-white py-3 px-4 rounded-lg font-bold transition-all shadow-lg shadow-purple-900/50 disabled:opacity-50"
              >
                {isAiLoading ? <RefreshCw className="w-5 h-5 animate-spin" /> : <PenTool className="w-5 h-5" />}
                ✨ 가사/테마 생성
              </button>
            </div>

            {aiError && (
              <div className="bg-red-500/20 text-red-200 p-3 rounded-lg text-sm border border-red-500/50">
                {aiError}
              </div>
            )}

            {aiResponse && (
              <div className="mt-6 bg-white/10 rounded-lg p-6 border border-white/10 animate-fade-in">
                <h3 className="text-yellow-300 font-bold mb-3 flex items-center gap-2">
                  <Sparkles className="w-4 h-4" /> AI 제안 결과
                </h3>
                <div className="prose prose-invert max-w-none text-sm leading-relaxed whitespace-pre-wrap">
                  {aiResponse}
                </div>
              </div>
            )}
          </div>
        </div>

        {/* Footer Info */}
        <div className="text-center text-gray-500 text-xs pb-4">
          Powered by Gemini API • Web Audio API • React
        </div>

      </div>
    </div>
  );
}
