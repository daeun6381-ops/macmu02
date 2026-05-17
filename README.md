# macmu02
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Finger Synthesizer</title>
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- MediaPipe Hands -->
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>
    <!-- Tone.js & Tonal.js for Audio & Music Theory -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tone/14.8.49/Tone.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/tonal/browser/tonal.min.js" crossorigin="anonymous"></script>
    
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@300;400;600;700&display=swap');
        body {
            font-family: 'Noto Sans KR', sans-serif;
            background-color: #0f172a;
            margin: 0;
            overflow: hidden;
        }
        .glass-panel {
            background: rgba(255, 255, 255, 0.1);
            backdrop-filter: blur(12px);
            border: 1px solid rgba(255, 255, 255, 0.2);
            box-shadow: 0 4px 30px rgba(0, 0, 0, 0.1);
        }
        .glow-text {
            text-shadow: 0 0 10px rgba(167, 139, 250, 0.8), 0 0 20px rgba(167, 139, 250, 0.5);
        }
        #output_canvas {
            transform: scaleX(-1); /* 거울 모드 좌우 반전 */
        }
    </style>
</head>
<body class="w-screen h-screen flex items-center justify-center bg-gradient-to-br from-indigo-950 via-purple-900 to-black text-white">

    <!-- 카메라 비디오 소스 (숨김) -->
    <video id="input_video" class="hidden" autoplay playsinline></video>

    <!-- 메인 캔버스 -->
    <canvas id="output_canvas" class="absolute top-0 left-0 w-full h-full object-cover opacity-60 mix-blend-screen"></canvas>

    <!-- 시작 화면 오버레이 -->
    <div id="start-screen" class="absolute inset-0 z-50 flex flex-col items-center justify-center bg-black/80 backdrop-blur-md transition-opacity duration-500 overflow-y-auto py-10">
        <h1 class="text-4xl md:text-5xl font-bold mb-6 glow-text text-center mt-auto">Finger Synthesizer</h1>
        
        <div class="glass-panel p-6 rounded-2xl max-w-xl mb-8 w-[90%] text-center space-y-4 text-indigo-100">
            <p class="text-lg">Open your fingers to play dreamy melodies and chords.</p>
            
            <!-- 커스텀 코드 입력 섹션 -->
            <div class="w-full mt-4 bg-black/40 p-5 rounded-xl border border-indigo-500/30">
                <label for="chord-input" class="block text-sm text-indigo-300 mb-2 font-bold text-left">연주할 코드 (쉼표로 구분하여 입력)</label>
                <input type="text" id="chord-input" class="w-full bg-indigo-950/70 border border-indigo-500/50 rounded-lg p-3 text-white outline-none focus:border-indigo-400 focus:ring-2 focus:ring-indigo-500 transition-all text-lg font-mono tracking-wider" value="Cmaj7, D7, Em7, Fmaj7, G7">
                
                <div class="mt-3 text-xs md:text-sm text-indigo-200 text-left space-y-1.5 leading-relaxed bg-indigo-900/30 p-3 rounded-lg">
                    <p>✨ <strong>[1~5개 입력 시]</strong> 양손이 동일하게 1~5번 화음을 공유합니다.</p>
                    <p>✨ <strong>[6~10개 입력 시]</strong> 양손이 각각 완전히 다른 화음을 가집니다!</p>
                </div>
            </div>

            <p class="mt-2 text-sm text-purple-300">💡 <strong>팁:</strong> 양손에 같은 번호의 코드를 연주하면 소리가 더 크고 풍성해집니다!</p>
        </div>

        <button id="start-btn" class="px-10 py-4 bg-indigo-600 hover:bg-indigo-500 text-white rounded-full font-bold text-xl transition-all shadow-[0_0_20px_rgba(79,70,229,0.5)] hover:shadow-[0_0_30px_rgba(79,70,229,0.8)] mb-auto">
            연주 시작하기
        </button>
        <p id="loading-text" class="mt-4 text-indigo-300 hidden mb-auto animate-pulse">카메라와 AI 모델을 불러오는 중입니다...</p>
    </div>

    <!-- UI 오버레이 -->
    <div class="absolute top-0 left-0 w-full h-full pointer-events-none p-4 md:p-6 flex flex-col justify-between z-10">
        <!-- 상단: 좌/우 코드 가이드 표 및 무음 존 -->
        <div class="flex justify-between items-start w-full gap-2">
            
            <!-- 왼쪽 화면 그룹 (표 + 무음 존) -->
            <div class="flex flex-col gap-3 w-32 md:w-40">
                <div class="glass-panel p-3 md:p-4 rounded-xl text-xs md:text-sm text-indigo-100 backdrop-blur-md bg-black/40 border border-indigo-500/30 shadow-[0_0_15px_rgba(79,70,229,0.2)] pointer-events-auto">
                    <h3 class="font-bold mb-2 text-indigo-300 text-center border-b border-indigo-500/30 pb-1">화면 왼쪽</h3>
                    <ul id="left-chord-list" class="space-y-1 md:space-y-1.5 text-left">
                        <!-- JS에서 동적 생성됨 -->
                    </ul>
                </div>
                <!-- 왼쪽 무음 존 (길이 확장 상태) -->
                <div id="mute-zone-left" class="h-48 md:h-64 rounded-xl border-dashed border-2 border-indigo-500/50 bg-indigo-900/30 text-center text-indigo-300 font-bold transition-all duration-300 pointer-events-auto flex flex-col items-center justify-center shadow-[0_0_10px_rgba(79,70,229,0.1)]">
                    <span class="text-2xl mb-2">🔇</span>
                    <span class="text-sm md:text-base tracking-wider">무음 존</span>
                </div>
            </div>

            <!-- 중앙 볼륨 인디케이터 -->
            <div id="volume-indicator" class="glass-panel px-4 py-2 md:px-6 rounded-full text-xs md:text-sm font-bold tracking-widest text-indigo-200 transition-all duration-300 h-10 flex items-center justify-center min-w-[140px]">
                NORMAL VOLUME
            </div>
            
            <!-- 오른쪽 화면 그룹 (표 + 무음 존) -->
            <div class="flex flex-col gap-3 w-32 md:w-40">
                <div class="glass-panel p-3 md:p-4 rounded-xl text-xs md:text-sm text-purple-100 backdrop-blur-md bg-black/40 border border-purple-500/30 shadow-[0_0_15px_rgba(168,85,247,0.2)] pointer-events-auto">
                    <h3 class="font-bold mb-2 text-purple-300 text-center border-b border-purple-500/30 pb-1">화면 오른쪽</h3>
                    <ul id="right-chord-list" class="space-y-1 md:space-y-1.5 text-left">
                        <!-- JS에서 동적 생성됨 -->
                    </ul>
                </div>
                <!-- 오른쪽 무음 존 (길이 확장 상태) -->
                <div id="mute-zone-right" class="h-48 md:h-64 rounded-xl border-dashed border-2 border-purple-500/50 bg-purple-900/30 text-center text-purple-300 font-bold transition-all duration-300 pointer-events-auto flex flex-col items-center justify-center shadow-[0_0_10px_rgba(168,85,247,0.1)]">
                    <span class="text-2xl mb-2">🔇</span>
                    <span class="text-sm md:text-base tracking-wider">무음 존</span>
                </div>
            </div>

        </div>

        <!-- 하단: 현재 인식된 손가락 및 코드 표시 -->
        <div class="flex justify-between items-end w-full">
            <!-- 화면 왼쪽 정보 -->
            <div class="glass-panel p-4 md:p-6 rounded-2xl w-32 md:w-40 text-center flex flex-col items-center border border-indigo-400/30">
                <p class="text-xs md:text-sm text-indigo-300 mb-1">현재 상태</p>
                <h2 id="left-chord-display" class="text-2xl md:text-3xl font-bold text-white mb-2 glow-text">-</h2>
                <div class="w-10 h-10 md:w-12 md:h-12 rounded-full bg-indigo-500/20 flex items-center justify-center border border-indigo-400/30">
                    <span id="left-finger-display" class="text-lg md:text-xl font-bold text-indigo-200">0</span>
                </div>
            </div>

            <!-- 화면 오른쪽 정보 -->
            <div class="glass-panel p-4 md:p-6 rounded-2xl w-32 md:w-40 text-center flex flex-col items-center border border-purple-400/30">
                <p class="text-xs md:text-sm text-purple-300 mb-1">현재 상태</p>
                <h2 id="right-chord-display" class="text-2xl md:text-3xl font-bold text-white mb-2 glow-text">-</h2>
                <div class="w-10 h-10 md:w-12 md:h-12 rounded-full bg-purple-500/20 flex items-center justify-center border border-purple-400/30">
                    <span id="right-finger-display" class="text-lg md:text-xl font-bold text-purple-200">0</span>
                </div>
            </div>
        </div>
    </div>

    <script>
        // DOM 요소
        const videoElement = document.getElementById('input_video');
        const canvasElement = document.getElementById('output_canvas');
        const canvasCtx = canvasElement.getContext('2d');
        const startBtn = document.getElementById('start-btn');
        const startScreen = document.getElementById('start-screen');
        const loadingText = document.getElementById('loading-text');
        const chordInput = document.getElementById('chord-input');
        
        // [코드 기억 기능] 3중 백업 메모리
        function saveChords(val) {
            try { localStorage.setItem('customChords', val); } catch(e) {}
            try { sessionStorage.setItem('customChords', val); } catch(e) {}
            try { history.replaceState(null, null, '#' + encodeURIComponent(val)); } catch(e) {}
        }

        function loadChords() {
            try { const local = localStorage.getItem('customChords'); if (local) return local; } catch(e) {}
            try { const session = sessionStorage.getItem('customChords'); if (session) return session; } catch(e) {}
            try { if (window.location.hash.length > 1) return decodeURIComponent(window.location.hash.substring(1)); } catch(e) {}
            return null;
        }

        const savedChords = loadChords();
        if (savedChords) {
            chordInput.value = savedChords;
        }

        chordInput.addEventListener('input', (e) => saveChords(e.target.value));
        chordInput.addEventListener('change', (e) => saveChords(e.target.value));

        // UI 요소
        const uiLeftChord = document.getElementById('left-chord-display');
        const uiRightChord = document.getElementById('right-chord-display');
        const uiLeftFinger = document.getElementById('left-finger-display');
        const uiRightFinger = document.getElementById('right-finger-display');
        const uiVolume = document.getElementById('volume-indicator');
        const leftChordListUI = document.getElementById('left-chord-list');
        const rightChordListUI = document.getElementById('right-chord-list');
        const muteZoneLeftEl = document.getElementById('mute-zone-left');
        const muteZoneRightEl = document.getElementById('mute-zone-right');

        // === 1. 코드 매핑 및 파싱 로직 ===
        let leftChordsMap = {};  
        let rightChordsMap = {};
        let leftChordNames = {}; 
        let rightChordNames = {};
        
        const FINGER_LABELS = ['1️⃣ 검지', '2️⃣ 중지', '3️⃣ 약지', '4️⃣ 새끼', '5️⃣ 엄지'];

        function getNotesWithOctave(chordName, rootOctave = 4) {
            const chordInfo = Tonal.Chord.get(chordName);
            if (chordInfo.empty) {
                const fallback = Tonal.Chord.get("C");
                return fallback.intervals.map(iv => Tonal.Note.transpose(`C${rootOctave}`, iv));
            }
            const root = chordInfo.tonic || chordInfo.root || "C";
            return chordInfo.intervals.map(iv => Tonal.Note.transpose(`${root}${rootOctave}`, iv));
        }

        function parseAndAssignChords() {
            const inputStr = chordInput.value;
            let rawChords = inputStr.split(',').map(s => s.trim()).filter(s => s.length > 0);
            
            if (rawChords.length === 0) {
                rawChords = ['Cmaj7', 'D7', 'Em7', 'Fmaj7', 'G7'];
            }

            let expandedChords = [];
            for (let i = 0; i < 10; i++) {
                expandedChords.push(rawChords[i % rawChords.length]);
            }

            leftChordsMap = {}; rightChordsMap = {};
            leftChordNames = {}; rightChordNames = {};

            if (rawChords.length <= 5) {
                for (let i = 1; i <= 5; i++) {
                    const chordName = rawChords[(i - 1) % rawChords.length];
                    const notes = getNotesWithOctave(chordName, 4);
                    
                    leftChordsMap[i] = notes;
                    rightChordsMap[i] = notes;
                    leftChordNames[i] = chordName;
                    rightChordNames[i] = chordName;
                }
            } else {
                for (let i = 1; i <= 5; i++) {
                    const leftName = expandedChords[i - 1]; 
                    const rightName = expandedChords[i + 4]; 
                    
                    leftChordsMap[i] = getNotesWithOctave(leftName, 4);
                    leftChordNames[i] = leftName;
                    
                    rightChordsMap[i] = getNotesWithOctave(rightName, 4);
                    rightChordNames[i] = rightName;
                }
            }

            renderChordGuidesUI();
        }

        function renderChordGuidesUI() {
            leftChordListUI.innerHTML = '';
            rightChordListUI.innerHTML = '';
            
            for (let i = 1; i <= 5; i++) {
                leftChordListUI.innerHTML += `
                    <li class="flex justify-between items-center w-full">
                        <span class="tracking-widest mr-2 text-xs">${FINGER_LABELS[i-1]}</span> 
                        <strong class="text-white truncate">${leftChordNames[i]}</strong>
                    </li>`;
                rightChordListUI.innerHTML += `
                    <li class="flex justify-between items-center w-full">
                        <span class="tracking-widest mr-2 text-xs">${FINGER_LABELS[i-1]}</span> 
                        <strong class="text-white truncate">${rightChordNames[i]}</strong>
                    </li>`;
            }
        }

        // === 2. 오디오 (Tone.js) 설정 ===
        // [소리 깨짐 방지] 폴리포니 제한 및 내부 볼륨 대폭 축소
        const synthOptions = {
            oscillator: { type: "sine" },
            envelope: { attack: 0.03, decay: 0.1, sustain: 1.0, release: 2.5 },
            volume: -26 
        };

        let leftSynth, rightSynth, masterVolume;

        async function initAudio() {
            Tone.setContext(new Tone.Context({ latencyHint: "interactive", lookAhead: 0 }));
            await Tone.start();
            
            const limiter = new Tone.Limiter(-2).toDestination();
            const reverb = new Tone.Reverb({ decay: 5, wet: 0.5 });
            const delay = new Tone.FeedbackDelay("8n", 0.2); 
            const lowcutFilter = new Tone.Filter({ frequency: 250, type: "highpass", Q: 0 }); 
            
            masterVolume = new Tone.Volume(-5);

            masterVolume.connect(limiter);
            reverb.connect(masterVolume);
            delay.connect(reverb);
            lowcutFilter.connect(delay);

            leftSynth = new Tone.PolySynth(Tone.Synth, synthOptions).connect(lowcutFilter);
            rightSynth = new Tone.PolySynth(Tone.Synth, synthOptions).connect(lowcutFilter);
            
            leftSynth.maxPolyphony = 16;
            rightSynth.maxPolyphony = 16;
        }

        // === 3. 손가락 인식 및 데이터 처리 ===
        function getDistance(p1, p2) {
            return Math.sqrt(Math.pow(p1.x - p2.x, 2) + Math.pow(p1.y - p2.y, 2));
        }

        function identifyFinger(landmarks) {
            // [가장 단순하고 완벽한 주먹/손가락 인식 로직] 
            // 손목(0)을 기준으로 손가락 끝(Tip)이 중간 관절(PIP)보다 멀면 펴진 것
            const isIndex = getDistance(landmarks[8], landmarks[0]) > getDistance(landmarks[6], landmarks[0]);
            const isMiddle = getDistance(landmarks[12], landmarks[0]) > getDistance(landmarks[10], landmarks[0]);
            const isRing = getDistance(landmarks[16], landmarks[0]) > getDistance(landmarks[14], landmarks[0]);
            const isPinky = getDistance(landmarks[20], landmarks[0]) > getDistance(landmarks[18], landmarks[0]);
            
            // 엄지는 손바닥 중심(9)에서 떨어져 있어야 펴진 것 (주먹 쥐면 붙음)
            const isThumb = getDistance(landmarks[4], landmarks[9]) > getDistance(landmarks[3], landmarks[9]) * 1.1;

            const openFingers = [];
            if (isIndex) openFingers.push(1);
            if (isMiddle) openFingers.push(2);
            if (isRing) openFingers.push(3);
            if (isPinky) openFingers.push(4);
            if (isThumb) openFingers.push(5);

            // 펴진 게 0개면 무조건 완벽한 주먹(-1) -> 소리 즉시 끔
            if (openFingers.length === 0) {
                return -1; 
            }

            if (openFingers.length === 1) return openFingers[0];
            if (openFingers.length > 1) return openFingers.length;
            
            return 0; 
        }

        class FingerSmoother {
            constructor(threshold = 1) {
                this.threshold = threshold;
                this.currentValue = 0;
                this.candidateValue = 0;
                this.count = 0;
            }
            add(val) {
                if (val === this.candidateValue) {
                    this.count++;
                } else {
                    this.candidateValue = val;
                    this.count = 1;
                }
                
                if (this.count >= this.threshold) {
                    this.currentValue = val;
                }
                return this.currentValue;
            }
        }

        const leftSmoother = new FingerSmoother();
        const rightSmoother = new FingerSmoother();

        let currentLeftFingers = 0;
        let currentRightFingers = 0;

        function updateAudioAndUI(leftCount, rightCount) {
            const isLeftActive = leftCount > 0;
            const isRightActive = rightCount > 0;

            if (isLeftActive && isRightActive && leftCount === rightCount) {
                masterVolume.volume.rampTo(0, 0.5); 
                uiVolume.innerText = "MAX RESONANCE";
                uiVolume.classList.add("text-fuchsia-300", "shadow-[0_0_15px_rgba(217,70,239,0.8)]");
            } else {
                masterVolume.volume.rampTo(-10, 0.5);
                uiVolume.innerText = "NORMAL VOLUME";
                uiVolume.classList.remove("text-fuchsia-300", "shadow-[0_0_15px_rgba(217,70,239,0.8)]");
            }

            if (leftCount !== currentLeftFingers) {
                const releaseTime = leftCount === -1 ? 0.1 : 2.5;
                leftSynth.set({ envelope: { release: releaseTime } });
                leftSynth.releaseAll(); 
                
                if (leftCount > 0 && leftChordsMap[leftCount]) {
                    leftSynth.triggerAttack(leftChordsMap[leftCount], Tone.now());
                }
                currentLeftFingers = leftCount;
            }

            if (rightCount !== currentRightFingers) {
                const releaseTime = rightCount === -1 ? 0.1 : 2.5;
                rightSynth.set({ envelope: { release: releaseTime } });
                rightSynth.releaseAll();

                if (rightCount > 0 && rightChordsMap[rightCount]) {
                    rightSynth.triggerAttack(rightChordsMap[rightCount], Tone.now());
                }
                currentRightFingers = rightCount;
            }

            uiLeftChord.innerText = leftCount > 0 ? leftChordNames[leftCount] : (leftCount === -1 ? '주먹' : '무음 존');
            uiLeftFinger.innerText = leftCount > 0 ? leftCount : (leftCount === -1 ? '✊' : '🔇');
            
            uiRightChord.innerText = rightCount > 0 ? rightChordNames[rightCount] : (rightCount === -1 ? '주먹' : '무음 존');
            uiRightFinger.innerText = rightCount > 0 ? rightCount : (rightCount === -1 ? '✊' : '🔇');
        }

        // === 4. MediaPipe 렌더링 루프 ===
        function onResults(results) {
            canvasCtx.save();
            canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);
            
            canvasCtx.drawImage(results.image, 0, 0, canvasElement.width, canvasElement.height);
            canvasCtx.fillStyle = "rgba(15, 23, 42, 0.5)";
            canvasCtx.fillRect(0, 0, canvasElement.width, canvasElement.height);

            let rawLeft = null;
            let rawRight = null;
            
            let isLeftZoneActive = false;
            let isRightZoneActive = false;

            const leftZoneRect = muteZoneLeftEl.getBoundingClientRect();
            const rightZoneRect = muteZoneRightEl.getBoundingClientRect();

            if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
                for (let i = 0; i < results.multiHandLandmarks.length; i++) {
                    const landmarks = results.multiHandLandmarks[i];
                    if (!landmarks || landmarks.length < 21) continue;

                    const screenX = window.innerWidth * (1 - landmarks[9].x); 
                    const screenY = window.innerHeight * landmarks[9].y;

                    let fingerCount = identifyFinger(landmarks);
                    const isLeftScreen = landmarks[0].x > 0.5;

                    if (screenX >= leftZoneRect.left && screenX <= leftZoneRect.right &&
                        screenY >= leftZoneRect.top && screenY <= leftZoneRect.bottom) {
                        fingerCount = 0; 
                        isLeftZoneActive = true;
                    }
                    else if (screenX >= rightZoneRect.left && screenX <= rightZoneRect.right &&
                             screenY >= rightZoneRect.top && screenY <= rightZoneRect.bottom) {
                        fingerCount = 0; 
                        isRightZoneActive = true;
                    }

                    const lineColor = isLeftScreen ? 'rgba(167, 139, 250, 0.5)' : 'rgba(192, 132, 252, 0.5)';
                    const pointColor = isLeftScreen ? '#818cf8' : '#e879f9';
                    drawConnectors(canvasCtx, landmarks, HAND_CONNECTIONS, {color: lineColor, lineWidth: 4});
                    drawLandmarks(canvasCtx, landmarks, {color: pointColor, lineWidth: 1, radius: 3});

                    if (isLeftScreen) {
                        if (rawLeft === null) rawLeft = fingerCount;
                    } else {
                        if (rawRight === null) rawRight = fingerCount;
                    }
                }
            }

            if (isLeftZoneActive) {
                muteZoneLeftEl.style.backgroundColor = 'rgba(99, 102, 241, 0.6)';
                muteZoneLeftEl.style.borderColor = 'rgba(165, 180, 252, 1)';
            } else {
                muteZoneLeftEl.style.backgroundColor = 'rgba(49, 46, 129, 0.3)';
                muteZoneLeftEl.style.borderColor = 'rgba(99, 102, 241, 0.5)';
            }

            if (isRightZoneActive) {
                muteZoneRightEl.style.backgroundColor = 'rgba(168, 85, 247, 0.6)';
                muteZoneRightEl.style.borderColor = 'rgba(216, 180, 254, 1)';
            } else {
                muteZoneRightEl.style.backgroundColor = 'rgba(88, 28, 135, 0.3)';
                muteZoneRightEl.style.borderColor = 'rgba(168, 85, 247, 0.5)';
            }

            if (rawLeft === null) rawLeft = 0;
            if (rawRight === null) rawRight = 0;

            const smoothedLeft = leftSmoother.add(rawLeft);
            const smoothedRight = rightSmoother.add(rawRight);

            if (leftSynth && rightSynth) {
                updateAudioAndUI(smoothedLeft, smoothedRight);
            }

            canvasCtx.restore();
        }

        const hands = new Hands({locateFile: (file) => {
            return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
        }});
        
        hands.setOptions({
            maxNumHands: 2,
            modelComplexity: 0, 
            minDetectionConfidence: 0.7,
            minTrackingConfidence: 0.7
        });
        hands.onResults(onResults);

        const camera = new Camera(videoElement, {
            onFrame: async () => {
                await hands.send({image: videoElement});
            },
            width: 640,
            height: 480
        });

        function resizeCanvas() {
            canvasElement.width = window.innerWidth;
            canvasElement.height = window.innerHeight;
        }
        window.addEventListener('resize', resizeCanvas);
        resizeCanvas();

        startBtn.addEventListener('click', async () => {
            saveChords(chordInput.value);
            parseAndAssignChords();

            startBtn.classList.add('hidden');
            chordInput.parentElement.classList.add('hidden'); 
            loadingText.classList.remove('hidden');
            
            try {
                await initAudio();
                await camera.start();
                
                startScreen.style.opacity = '0';
                setTimeout(() => {
                    startScreen.style.display = 'none';
                }, 500);
                
            } catch (error) {
                console.error("초기화 오류:", error);
                loadingText.innerText = "카메라 접근 권한이 필요합니다. 새로고침 후 허용해주세요.";
            }
        });
    </script>
</body>
</html>


```
