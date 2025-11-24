import React, { useState, useRef, useEffect } from 'react';
import { Upload, Music, Play, Pause, Download, FileAudio, RefreshCw, Wand2, Mic, MicOff, AlertCircle, X, Loader2 } from 'lucide-react';

// 음계 데이터 (주파수 -> 노트 매핑)
const NOTE_STRINGS = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"];

const getNoteFromFrequency = (frequency) => {
  if (!frequency || frequency < 27.5) return null; // A0 미만은 무시
  const noteNum = 12 * (Math.log(frequency / 440) / Math.log(2));
  const midi = Math.round(noteNum) + 69;
  const note = NOTE_STRINGS[midi % 12];
  const octave = Math.floor(midi / 12) - 1;
  return { note, octave, midi, frequency };
};

const App = () => {
  // 모드: 'file' | 'mic' | null
  const [sourceMode, setSourceMode] = useState(null);
  const [audioFile, setAudioFile] = useState(null);
  const [isPlaying, setIsPlaying] = useState(false); // UI 상태 (재생 중 여부)
  const [currentNote, setCurrentNote] = useState({ note: '-', octave: '', frequency: 0 });
  const [detectedNotes, setDetectedNotes] = useState([]); // { time, note, midi }
  const [errorMessage, setErrorMessage] = useState(null); // 에러 메시지
  const [isAudioReady, setIsAudioReady] = useState(false); // 오디오 로딩 완료 여부
  
  const audioRef = useRef(null);
  const canvasRef = useRef(null);
  const pianoRollRef = useRef(null);
  
  // Audio Context 관련 Refs
  const audioContextRef = useRef(null);
  const analyserRef = useRef(null);
  const sourceRef = useRef(null);      // 파일 소스
  const micSourceRef = useRef(null);   // 마이크 소스
  const micStreamRef = useRef(null);   // 마이크 스트림
  const requestRef = useRef(null);
  const startTimeRef = useRef(0);      // 마이크 모드일 때 시간 측정용
  const isPlayingRef = useRef(false);  // 애니메이션 루프 제어용 (State와 동기화)

  // 오디오 컨텍스트 초기화 (공통)
  const initAudioContext = () => {
    if (!audioContextRef.current) {
      const AudioContext = window.AudioContext || window.webkitAudioContext;
      audioContextRef.current = new AudioContext();
      
      analyserRef.current = audioContextRef.current.createAnalyser();
      analyserRef.current.fftSize = 2048; 
      analyserRef.current.smoothingTimeConstant = 0.8;
    }
  };

  // 안전한 재생 함수
  const safePlay = async () => {
    if (!audioRef.current) return;
    try {
      // AudioContext가 suspended 상태라면 깨움 (브라우저 정책 대응)
      if (audioContextRef.current && audioContextRef.current.state === 'suspended') {
        await audioContextRef.current.resume();
      }
      
      await audioRef.current.play();
      // 성공 시 상태 업데이트
      setIsPlaying(true);
      isPlayingRef.current = true;
      return true; 
    } catch (err) {
      if (err.name !== 'AbortError') {
        console.error("Playback failed:", err);
        setErrorMessage("오디오 재생 실패: " + err.message);
        setIsPlaying(false);
        isPlayingRef.current = false;
      }
      return false; 
    }
  };

  // 안전한 일시정지 함수
  const safePause = () => {
    if (audioRef.current) {
      audioRef.current.pause();
      setIsPlaying(false);
      isPlayingRef.current = false;
    }
  };

  // 파일 업로드 핸들러
  const handleFileUpload = (e) => {
    const file = e.target.files[0];
    if (file) {
      setErrorMessage(null);
      stopMic();
      safePause();
      cancelAnimationFrame(requestRef.current);
      
      const url = URL.createObjectURL(file);
      setAudioFile({ file, url });
      setDetectedNotes([]);
      setSourceMode('file');
      setIsAudioReady(false); // 로딩 시작
      
      if (audioRef.current) {
        audioRef.current.src = url;
        audioRef.current.load();
      }
    }
  };

  // 오디오 로딩 완료 핸들러
  const handleCanPlay = () => {
    setIsAudioReady(true);
  };

  // 마이크 켜기/끄기 토글
  const toggleMic = async () => {
    if (sourceMode === 'mic' && isPlaying) {
      stopMic();
    } else {
      await startMic();
    }
  };

  const startMic = async () => {
    setErrorMessage(null);
    try {
      if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
        throw new Error("이 브라우저 환경에서는 마이크 API를 지원하지 않습니다.");
      }

      initAudioContext();
      if (audioContextRef.current.state === 'suspended') {
        await audioContextRef.current.resume();
      }

      if (audioRef.current) {
        safePause();
        audioRef.current.currentTime = 0;
      }

      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      micStreamRef.current = stream;
      
      const source = audioContextRef.current.createMediaStreamSource(stream);
      source.connect(analyserRef.current);
      
      micSourceRef.current = source;
      
      setSourceMode('mic');
      setAudioFile(null);
      setDetectedNotes([]);
      startTimeRef.current = Date.now();
      
      setIsPlaying(true);
      isPlayingRef.current = true;
      
      // 즉시 루프 시작
      cancelAnimationFrame(requestRef.current);
      requestRef.current = requestAnimationFrame(animate);

    } catch (err) {
      console.error("마이크 접근 오류:", err);
      let friendlyMsg = "마이크를 사용할 수 없습니다.";
      if (err.name === 'NotFoundError' || err.message.includes('not found')) {
        friendlyMsg = "마이크 장치를 찾을 수 없습니다.";
      } else if (err.name === 'NotAllowedError' || err.name === 'PermissionDeniedError') {
        friendlyMsg = "마이크 권한이 거부되었습니다. 설정에서 허용해주세요.";
      }
      setErrorMessage(friendlyMsg);
      setIsPlaying(false);
      isPlayingRef.current = false;
    }
  };

  const stopMic = () => {
    if (micStreamRef.current) {
      micStreamRef.current.getTracks().forEach(track => track.stop());
      micStreamRef.current = null;
    }
    if (micSourceRef.current) {
      micSourceRef.current.disconnect();
      micSourceRef.current = null;
    }
    setIsPlaying(false);
    isPlayingRef.current = false;
    if (requestRef.current) cancelAnimationFrame(requestRef.current);
    setSourceMode(null);
  };

  // 파일 재생 토글
  const toggleFilePlay = async () => {
    if (!audioFile) return;
    setErrorMessage(null);

    // 이미 재생 중이면 일시정지
    if (isPlaying && sourceMode === 'file') {
      safePause();
      cancelAnimationFrame(requestRef.current);
    } else {
      // 재생 시작
      initAudioContext();
      
      // 소스 연결 (한 번만 수행하되, 연결이 끊겼을 수 있으므로 체크)
      // 주의: createMediaElementSource는 한 요소당 한 번만 생성 가능.
      if (!sourceRef.current) {
        try {
          sourceRef.current = audioContextRef.current.createMediaElementSource(audioRef.current);
          sourceRef.current.connect(analyserRef.current);
          analyserRef.current.connect(audioContextRef.current.destination);
        } catch (e) {
          console.warn("MediaElementSource reconnect error (ignored):", e);
        }
      }

      setSourceMode('file'); 
      
      // UI를 먼저 '재생 중'으로 변경하여 반응성 향상
      setIsPlaying(true);
      isPlayingRef.current = true;

      // 루프 강제 시작 (play가 완료되기 전이라도 시각화 준비)
      cancelAnimationFrame(requestRef.current);
      requestRef.current = requestAnimationFrame(animate);

      // 실제 오디오 재생
      await safePlay();
    }
  };

  // 분석 루프
  const animate = () => {
    if (!analyserRef.current || !canvasRef.current) return;

    const bufferLength = analyserRef.current.frequencyBinCount;
    const dataArray = new Uint8Array(bufferLength);
    analyserRef.current.getByteFrequencyData(dataArray);

    // 1. 캔버스 시각화
    const canvas = canvasRef.current;
    const ctx = canvas.getContext('2d');
    const width = canvas.width;
    const height = canvas.height;

    ctx.fillStyle = '#111827';
    ctx.fillRect(0, 0, width, height);

    const barWidth = (width / bufferLength) * 2.5;
    let barHeight;
    let x = 0;
    
    // 그라데이션
    const gradient = ctx.createLinearGradient(0, height, 0, 0);
    gradient.addColorStop(0, '#8b5cf6'); // Violet
    gradient.addColorStop(1, '#3b82f6'); // Blue

    let maxVal = -Infinity;
    let maxIndex = -1;
    let hasSignal = false; // 신호 감지 여부

    for (let i = 0; i < bufferLength; i++) {
      barHeight = dataArray[i] / 2;
      
      if (dataArray[i] > 10) hasSignal = true; // 아주 작은 신호라도 있는지 체크

      ctx.fillStyle = gradient;
      ctx.fillRect(x, height - barHeight, barWidth, barHeight);
      x += barWidth + 1;

      // Pitch Detection Logic
      if (dataArray[i] > maxVal && dataArray[i] > 130) { 
        maxVal = dataArray[i];
        maxIndex = i;
      }
    }

    // 2. 노트 변환 (신호가 있을 때만)
    if (maxIndex !== -1 && hasSignal) {
      const nyquist = audioContextRef.current.sampleRate / 2;
      const frequency = maxIndex * (nyquist / bufferLength);
      const noteData = getNoteFromFrequency(frequency);

      if (noteData) {
        setCurrentNote(noteData);
        
        // 시간 계산
        let currentTime = 0;
        if (audioRef.current) {
           currentTime = audioRef.current.currentTime;
        } else if (micStreamRef.current) {
           currentTime = (Date.now() - startTimeRef.current) / 1000;
        }

        setDetectedNotes(prev => {
          const last = prev[prev.length - 1];
          // 0.1초 간격 혹은 노트가 바뀔 때 기록
          if (!last || Math.abs(last.time - currentTime) > 0.1) {
            return [...prev, { time: currentTime, ...noteData }];
          }
          return prev;
        });
      }
    } else {
        // 신호가 없으면 노트 표시 초기화하지 않고 유지 (보기에 좋음)
    }

    // 루프 지속 로직 개선:
    // 1. 사용자가 '재생'을 눌렀다고 표시된 경우 (isPlayingRef.current) 무조건 루프
    // 2. 단, 파일 재생이 실제로 끝났으면(ended) 멈춤
    
    let shouldContinue = isPlayingRef.current;
    if (sourceMode === 'file' && audioRef.current && audioRef.current.ended) {
        shouldContinue = false;
        setIsPlaying(false);
        isPlayingRef.current = false;
    }

    if (shouldContinue) {
       requestRef.current = requestAnimationFrame(animate);
    }
  };

  // 피아노 롤 자동 스크롤
  useEffect(() => {
      if (pianoRollRef.current) {
          pianoRollRef.current.scrollLeft = pianoRollRef.current.scrollWidth;
      }
  }, [detectedNotes]);

  // 컴포넌트 언마운트 시 정리
  useEffect(() => {
    return () => {
      cancelAnimationFrame(requestRef.current);
      if (micStreamRef.current) {
        micStreamRef.current.getTracks().forEach(track => track.stop());
      }
      if (audioContextRef.current) {
        audioContextRef.current.close();
      }
    };
  }, []);

  const downloadNotes = () => {
    const text = detectedNotes.map(n => `[${n.time.toFixed(2)}s] Note: ${n.note}${n.octave} (${n.frequency.toFixed(0)}Hz)`).join('\n');
    const blob = new Blob([text], { type: 'text/plain' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'transcribed_notes.txt';
    a.click();
  };

  const resetAll = () => {
      stopMic();
      if(audioRef.current) {
          safePause();
          audioRef.current.currentTime = 0;
      }
      cancelAnimationFrame(requestRef.current);
      setDetectedNotes([]);
      setIsPlaying(false);
      isPlayingRef.current = false;
      setErrorMessage(null);
      setCurrentNote({ note: '-', octave: '', frequency: 0 });
  };

  return (
    <div className="min-h-screen bg-gray-900 text-white font-sans selection:bg-purple-500 selection:text-white">
      {/* Header */}
      <header className="border-b border-gray-800 bg-gray-900/50 backdrop-blur-md sticky top-0 z-10">
        <div className="max-w-6xl mx-auto px-6 h-16 flex items-center justify-between">
          <div className="flex items-center space-x-2">
            <div className="w-8 h-8 bg-gradient-to-br from-purple-500 to-blue-500 rounded-lg flex items-center justify-center">
              <Music className="w-5 h-5 text-white" />
            </div>
            <span className="text-xl font-bold bg-clip-text text-transparent bg-gradient-to-r from-purple-400 to-blue-400">
              AI Melody Studio
            </span>
          </div>
          <div className="hidden md:flex items-center space-x-2 text-xs bg-gray-800 px-3 py-1 rounded-full border border-gray-700">
             <div className="w-2 h-2 rounded-full bg-red-500 animate-pulse"></div>
             <span className="text-gray-300">Beta: Chrome 브라우저 권장</span>
          </div>
        </div>
      </header>

      <main className="max-w-6xl mx-auto px-6 py-8 space-y-8">
        
        {/* Error Banner */}
        {errorMessage && (
          <div className="bg-red-500/10 border border-red-500/50 text-red-200 px-6 py-4 rounded-xl flex items-center justify-between animate-fade-in">
            <div className="flex items-center space-x-3">
              <AlertCircle className="w-5 h-5 text-red-500" />
              <span>{errorMessage}</span>
            </div>
            <button onClick={() => setErrorMessage(null)} className="hover:text-white transition">
              <X className="w-5 h-5" />
            </button>
          </div>
        )}

        {/* Intro Section (파일도 마이크도 없을 때) */}
        {!audioFile && sourceMode !== 'mic' && (
          <div className="text-center py-20 space-y-6 animate-fade-in">
            <h1 className="text-4xl md:text-5xl font-bold leading-tight">
              당신의 <span className="text-purple-400">흥얼거림</span>을 악보로
            </h1>
            <p className="text-gray-400 text-lg max-w-2xl mx-auto">
              파일을 업로드하거나 마이크를 켜고 직접 노래를 불러보세요.<br/>
              AI가 실시간으로 멜로디를 분석하여 코드를 찾아줍니다.
            </p>
            
            <div className="flex flex-col md:flex-row justify-center items-center gap-6 mt-8">
              {/* File Upload Button */}
              <label className="group relative cursor-pointer w-full md:w-auto">
                <div className="absolute -inset-1 bg-gradient-to-r from-purple-600 to-blue-600 rounded-xl blur opacity-25 group-hover:opacity-100 transition duration-1000 group-hover:duration-200"></div>
                <div className="relative flex items-center justify-center space-x-3 bg-gray-800 hover:bg-gray-750 px-8 py-5 rounded-xl border border-gray-700 transition-all transform group-hover:-translate-y-1">
                  <Upload className="w-6 h-6 text-purple-400" />
                  <span className="font-semibold text-lg">오디오 파일 업로드</span>
                </div>
                <input type="file" accept="audio/*" onChange={handleFileUpload} className="hidden" />
              </label>

              <span className="text-gray-500 font-medium">OR</span>

              {/* Mic Button */}
              <button 
                onClick={toggleMic}
                className="group relative w-full md:w-auto"
              >
                <div className="absolute -inset-1 bg-gradient-to-r from-red-500 to-orange-500 rounded-xl blur opacity-25 group-hover:opacity-100 transition duration-1000 group-hover:duration-200"></div>
                <div className="relative flex items-center justify-center space-x-3 bg-gray-800 hover:bg-gray-750 px-8 py-5 rounded-xl border border-gray-700 transition-all transform group-hover:-translate-y-1">
                  <Mic className="w-6 h-6 text-red-400" />
                  <span className="font-semibold text-lg">마이크로 흥얼거리기</span>
                </div>
              </button>
            </div>
          </div>
        )}

        {/* Workstation (파일이 있거나 마이크 모드일 때) */}
        {(audioFile || sourceMode === 'mic') && (
          <div className="grid grid-cols-1 lg:grid-cols-3 gap-6 animate-slide-up">
            
            {/* Left Panel: Visualizer & Controls */}
            <div className="lg:col-span-2 space-y-6">
              
              {/* Visualizer Canvas */}
              <div className="bg-gray-800 rounded-2xl p-6 border border-gray-700 shadow-2xl relative overflow-hidden">
                 <div className="absolute top-4 left-6 flex items-center space-x-2 z-10">
                    <div className={`w-3 h-3 rounded-full ${isPlaying ? 'bg-green-500 animate-pulse' : 'bg-gray-500'}`}></div>
                    <span className="text-xs font-mono text-gray-400">
                        {isPlaying ? (sourceMode === 'mic' ? 'MIC LISTENING' : 'PLAYING') : 'READY'}
                    </span>
                 </div>
                <canvas 
                  ref={canvasRef} 
                  width="800" 
                  height="300" 
                  className="w-full h-[300px] rounded-lg bg-gray-900"
                ></canvas>
              </div>

              {/* Player Controls */}
              <div className="bg-gray-800 rounded-xl p-4 border border-gray-700 flex items-center justify-between flex-wrap gap-4">
                <div className="flex items-center space-x-4">
                    {/* Play/Pause Button (Switch based on mode) */}
                    {sourceMode === 'mic' ? (
                        <button 
                            onClick={toggleMic}
                            className={`w-12 h-12 rounded-full flex items-center justify-center transition shadow-lg ${isPlaying ? 'bg-red-500 text-white animate-pulse' : 'bg-gray-700 text-gray-300'}`}
                        >
                            {isPlaying ? <MicOff className="fill-current" /> : <Mic className="fill-current" />}
                        </button>
                    ) : (
                        <button 
                            onClick={toggleFilePlay}
                            disabled={!isAudioReady}
                            className={`w-12 h-12 rounded-full flex items-center justify-center transition shadow-lg ${isAudioReady ? 'bg-white text-gray-900 hover:bg-gray-200' : 'bg-gray-600 text-gray-400 cursor-not-allowed'}`}
                        >
                            {!isAudioReady ? (
                                <Loader2 className="animate-spin w-5 h-5" />
                            ) : isPlaying ? (
                                <Pause className="fill-current" />
                            ) : (
                                <Play className="fill-current ml-1" />
                            )}
                        </button>
                    )}

                    <div className="flex flex-col">
                        <span className="font-medium text-white truncate max-w-[200px]">
                            {sourceMode === 'mic' ? 'Microphone Input' : audioFile?.file.name}
                        </span>
                        <span className="text-xs text-purple-400 flex items-center">
                            {sourceMode === 'mic' ? (
                                '실시간 입력 감지 중...'
                            ) : !isAudioReady ? (
                                <span className="flex items-center"><Loader2 className="w-3 h-3 animate-spin mr-1"/> 오디오 로딩 중...</span>
                            ) : (
                                '파일 분석 준비 완료'
                            )}
                        </span>
                    </div>
                </div>
                
                <div className="flex items-center space-x-3">
                     <button 
                        onClick={resetAll}
                        className="p-2 hover:bg-gray-700 rounded-lg text-gray-400 hover:text-white transition"
                        title="초기화"
                     >
                        <RefreshCw className="w-5 h-5" />
                     </button>
                     
                     {/* Switch Source Buttons */}
                     {sourceMode === 'mic' ? (
                        <label className="px-4 py-2 bg-gray-700 hover:bg-gray-600 rounded-lg text-sm font-medium transition cursor-pointer">
                            파일 업로드로 전환
                            <input type="file" accept="audio/*" onChange={handleFileUpload} className="hidden" />
                        </label>
                     ) : (
                        <button 
                            onClick={async () => {
                                setAudioFile(null);
                                await startMic();
                            }}
                            className="px-4 py-2 bg-gray-700 hover:bg-gray-600 rounded-lg text-sm font-medium transition flex items-center gap-2"
                        >
                            <Mic className="w-4 h-4" />
                            마이크로 전환
                        </button>
                     )}
                </div>
                {/* 중요: onCanPlay 이벤트 추가 및 crossOrigin 설정 */}
                <audio 
                    ref={audioRef} 
                    onCanPlay={handleCanPlay}
                    onEnded={() => {
                        setIsPlaying(false);
                        isPlayingRef.current = false;
                    }} 
                    crossOrigin="anonymous" 
                    className="hidden" 
                />
              </div>
            </div>

            {/* Right Panel: Analysis Result */}
            <div className="space-y-6">
              
              {/* Current Note Display */}
              <div className="bg-gradient-to-br from-purple-900/50 to-blue-900/50 rounded-2xl p-8 border border-purple-500/30 flex flex-col items-center justify-center text-center h-[200px]">
                <h3 className="text-purple-300 text-sm font-semibold uppercase tracking-wider mb-2">Pitch</h3>
                <div className="flex items-baseline space-x-1">
                    <span className="text-7xl font-black text-white drop-shadow-[0_0_15px_rgba(168,85,247,0.5)]">
                        {currentNote.note}
                    </span>
                    <span className="text-3xl text-gray-400 font-light">{currentNote.octave}</span>
                </div>
                <div className="mt-2 text-sm text-gray-400 font-mono">
                    {currentNote.frequency > 0 ? `${currentNote.frequency.toFixed(1)} Hz` : 'Listening...'}
                </div>
              </div>

              {/* Tools */}
              <div className="bg-gray-800 rounded-xl p-5 border border-gray-700 space-y-4">
                  <div className="flex items-center space-x-3 text-gray-300 pb-4 border-b border-gray-700">
                      <Wand2 className="w-5 h-5 text-purple-400" />
                      <span className="font-semibold">분석 도구</span>
                  </div>
                  <div className="space-y-2">
                      <button 
                        onClick={downloadNotes}
                        disabled={detectedNotes.length === 0}
                        className="w-full flex items-center justify-center space-x-2 py-3 bg-gray-700 hover:bg-gray-600 disabled:opacity-50 disabled:cursor-not-allowed rounded-lg text-sm font-medium transition group"
                      >
                          <Download className="w-4 h-4 group-hover:-translate-y-0.5 transition" />
                          <span>텍스트 악보 다운로드</span>
                      </button>
                      <p className="text-xs text-gray-500 text-center px-2 leading-relaxed">
                          팁: 마이크 사용 시 주변 소음을 줄이고 마이크에 가까이 대고 흥얼거려 보세요.
                      </p>
                  </div>
              </div>
            </div>

            {/* Bottom Panel: Piano Roll Visualization */}
            <div className="lg:col-span-3">
                 <div className="bg-gray-800 rounded-xl border border-gray-700 overflow-hidden">
                     <div className="px-6 py-4 border-b border-gray-700 flex justify-between items-center bg-gray-800/50">
                        <h3 className="font-semibold text-gray-200 flex items-center">
                            <FileAudio className="w-4 h-4 mr-2 text-blue-400" />
                            Melody Piano Roll
                        </h3>
                        <span className="text-xs text-gray-500 font-mono">
                            {detectedNotes.length} NOTES DETECTED
                        </span>
                     </div>
                     
                     <div className="relative h-64 bg-gray-900 overflow-hidden">
                         {/* Background Grid (Octaves) */}
                         <div className="absolute inset-0 flex flex-col justify-between py-2 opacity-20 pointer-events-none">
                             {[...Array(7)].map((_, i) => (
                                 <div key={i} className="w-full border-t border-gray-600 h-8 flex items-center px-2 text-[10px] text-gray-400">
                                     C{7-i}
                                 </div>
                             ))}
                         </div>

                         {/* Notes Visualizer (Scrolling) */}
                         <div 
                            ref={pianoRollRef}
                            className="absolute inset-0 overflow-x-auto overflow-y-hidden flex items-end pl-[50%] scroll-smooth no-scrollbar"
                            style={{ scrollBehavior: 'auto' }}
                         >
                            <div className="flex h-full relative" style={{ minWidth: '100%' }}>
                                {detectedNotes.map((n, idx) => {
                                    const minMidi = 36; // C2
                                    const maxMidi = 96; // C7
                                    let bottomPercent = ((n.midi - minMidi) / (maxMidi - minMidi)) * 100;
                                    bottomPercent = Math.max(0, Math.min(100, bottomPercent));
                                    
                                    return (
                                        <div 
                                            key={idx}
                                            className="w-2 md:w-3 flex-shrink-0 bg-gradient-to-t from-purple-500 to-blue-400 rounded-sm opacity-80 mx-[1px]"
                                            style={{ 
                                                height: '15%',
                                                marginBottom: `${bottomPercent}%` 
                                            }}
                                            title={`${n.note}${n.octave}`}
                                        ></div>
                                    )
                                })}
                                {/* 스크롤 앵커 */}
                                <div className="w-[50%] flex-shrink-0"></div>
                            </div>
                         </div>
                     </div>
                 </div>
            </div>

          </div>
        )}
      </main>

      {/* Footer */}
      <footer className="max-w-6xl mx-auto px-6 py-8 text-center text-gray-500 text-sm border-t border-gray-800 mt-12">
        <p>© 2024 AI Melody Studio. Created for music lovers.</p>
        <p className="mt-2 text-xs text-gray-600">
            Note: 이 데모는 브라우저의 Web Audio API를 사용하여 단일 피치(Monophonic)를 감지합니다. 
            복잡한 화음이나 드럼 소리가 섞인 곡은 정확도가 낮을 수 있습니다.
        </p>
      </footer>
      
      <style>{`
        .no-scrollbar::-webkit-scrollbar {
          display: none;
        }
        .no-scrollbar {
          -ms-overflow-style: none;
          scrollbar-width: none;
        }
        @keyframes fade-in {
            from { opacity: 0; }
            to { opacity: 1; }
        }
        .animate-fade-in {
            animation: fade-in 1s ease-out;
        }
        @keyframes slide-up {
            from { opacity: 0; transform: translateY(20px); }
            to { opacity: 1; transform: translateY(0); }
        }
        .animate-slide-up {
            animation: slide-up 0.8s ease-out;
        }
      `}</style>
    </div>
  );
};

export default App;
