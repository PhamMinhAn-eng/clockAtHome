<!DOCTYPE html>
<html lang="vi">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Đồng hồ Học - Nghỉ</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #e0f7e9;
            color: #2f4f2f;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            margin: 0;
        }

        h1 {
            color: #2f7f2f;
            font-size: 2rem;
        }

        label,
        h2 {
            color: #2f4f2f;
        }

        input[type="number"] {
            width: 60px;
            padding: 5px;
            border: 1px solid #2f7f2f;
            border-radius: 5px;
            text-align: center;
            font-size: 1rem;
            color: #2f4f2f;
        }

        button {
            padding: 10px 20px;
            margin: 10px;
            border: none;
            border-radius: 5px;
            font-size: 1rem;
            cursor: pointer;
            color: white;
        }

        #startButton {
            background-color: #2f7f2f;
        }

        #stopButton {
            background-color: #ff4d4d;
        }

        #timer {
            font-size: 2.5rem;
            font-weight: bold;
            margin-top: 10px;
        }

        /* Màn hình bật lên */
        #overlay {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background-color: rgba(0, 0, 0, 0.5);
            display: none;
            align-items: center;
            justify-content: center;
            z-index: 1000;
        }

        #popup {
            background-color: #ffffff;
            padding: 20px;
            border-radius: 8px;
            text-align: center;
            color: #2f4f2f;
        }

        #popup h2 {
            margin-top: 0;
            color: #2f7f2f;
        }

        #continueButton {
            padding: 10px 20px;
            background-color: #2f7f2f;
            color: white;
            border: none;
            border-radius: 5px;
            font-size: 1rem;
            cursor: pointer;
        }
    </style>
</head>

<body>
    <h1>Đồng hồ Học - Nghỉ</h1>

    <label for="studyTime">Thời gian học (phút):</label>
    <input type="number" id="studyTime" min="1" value="25"><br><br>

    <label for="breakTime">Thời gian nghỉ (phút):</label>
    <input type="number" id="breakTime" min="1" value="5"><br><br>

    <button id="startButton" onclick="startTimer()">Bắt đầu</button>
    <button id="stopButton" onclick="stopTimer()">Dừng</button>

    <h2 id="status">Đang chờ bắt đầu...</h2>
    <h2 id="timer">00:00</h2>

    <audio id="alarmSound" preload="auto"></audio>

    <!-- Màn hình bật lên -->
    <div id="overlay">
        <div id="popup">
            <h2>Đã hết thời gian!</h2>
            <button id="continueButton" onclick="continueTimer()">Tiếp tục</button>
        </div>
    </div>

    <script>
        let studyTime, breakTime;
        let isStudyTime = true;
        let worker;
        let alarmSound = document.getElementById("alarmSound");

        // Inline Web Worker
        const workerBlob = new Blob([`
            let studyTime;
            let breakTime;
            let currentTime;
            let timerInterval;
            let isPaused = false;
            let isStudyTime = true;

            onmessage = function (e) {
                const data = e.data;
                if (data.studyTime && data.breakTime) {
                    studyTime = data.studyTime;
                    breakTime = data.breakTime;
                    currentTime = studyTime;
                    isPaused = false;
                    isStudyTime = true;
                    startTimer();
                }
                if (data.action === 'continue') {
                    isPaused = false;
                    startTimer();
                }
            };

            function startTimer() {
                if (timerInterval) clearInterval(timerInterval);

                timerInterval = setInterval(() => {
                    if (isPaused) return;
                    currentTime--;

                    if (currentTime <= 0) {
                        clearInterval(timerInterval);
                        isPaused = true;

                        if (isStudyTime) {
                            postMessage({ status: 'time-up', type: 'study' });
                            currentTime = breakTime;
                        } else {
                            postMessage({ status: 'time-up', type: 'break' });
                            currentTime = studyTime;
                        }
                        isStudyTime = !isStudyTime;
                    } else {
                        postMessage({
                            status: 'update',
                            time: formatTime(currentTime)
                        });
                    }
                }, 1000);
            }

            function formatTime(seconds) {
                const minutes = Math.floor(seconds / 60);
                const remainingSeconds = seconds % 60;
                return \`\${String(minutes).padStart(2, '0')}:\${String(remainingSeconds).padStart(2, '0')}\`;
            }
        `], { type: 'application/javascript' });

        function startTimer() {
            if (worker) worker.terminate();

            worker = new Worker(URL.createObjectURL(workerBlob));

            studyTime = parseInt(document.getElementById("studyTime").value) * 60;
            breakTime = parseInt(document.getElementById("breakTime").value) * 60;

            worker.postMessage({ studyTime, breakTime });

            worker.onmessage = function (e) {
                const data = e.data;
                if (data.status === 'time-up') {
                    playSound(data.type);
                    showOverlay();
                } else if (data.status === 'update') {
                    document.getElementById('timer').innerText = data.time;
                }
            };

            document.getElementById("status").innerText = "Đang chạy...";
            document.getElementById("startButton").disabled = true;
            document.getElementById("stopButton").disabled = false;
        }

        function stopTimer() {
            if (worker) {
                worker.terminate();
                document.getElementById("status").innerText = "Đã dừng";
                document.getElementById("timer").innerText = "00:00";
            }
            alarmSound.pause();
            alarmSound.currentTime = 0;
            document.getElementById("startButton").disabled = false;
            document.getElementById("stopButton").disabled = true;
        }

        function playSound(type) {
            const soundNumber = Math.floor(Math.random() * 5) + 1;
            const soundPath = `sounds/${type}/${soundNumber}.mp3`;
            alarmSound.src = soundPath;
            alarmSound.play();
        }

        function showOverlay() {
            document.getElementById("overlay").style.display = "flex";
        }

        function continueTimer() {
            alarmSound.pause();
            alarmSound.currentTime = 0;
            document.getElementById("overlay").style.display = "none";
            worker.postMessage({ action: 'continue' });
            document.getElementById("status").innerText = "Đang chạy...";
        }
    </script>
</body>

</html>
