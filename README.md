<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ระบบข้อมูลส่วนบุคคล - Smart Login</title>
    <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
    <style>
        body { 
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; 
            display: flex; justify-content: center; align-items: center; 
            min-height: 100vh; background-color: #f4f7f6; margin: 0; padding: 20px; box-sizing: border-box;
        }
        .app-container { 
            width: 100%; max-width: 400px; background: white; padding: 35px 25px; 
            border-radius: 20px; box-shadow: 0 10px 30px rgba(0,0,0,0.06); text-align: center; 
        }
        .logo-placeholder { width: 60px; height: 60px; background: #0066ff; color: white; border-radius: 16px; display: inline-flex; justify-content: center; align-items: center; font-size: 24px; font-weight: bold; margin-bottom: 15px; }
        h2 { margin: 0 0 8px 0; color: #1a1a1a; font-size: 22px; }
        p#prompt-text { color: #5f6368; font-size: 15px; margin-bottom: 25px; line-height: 1.5; }
        input { 
            width: 100%; padding: 14px; margin-bottom: 15px; 
            border: 1.5px solid #e0e0e0; border-radius: 10px; font-size: 16px; box-sizing: border-box; text-align: center; transition: border-color 0.2s;
        }
        input:focus { border-color: #0066ff; outline: none; }
        button { 
            width: 100%; padding: 14px; background: #0066ff; color: white; border: none; 
            border-radius: 10px; font-size: 16px; font-weight: 600; cursor: pointer; transition: background 0.2s; 
        }
        button:hover { background: #0052cc; }
        .error-text { color: #d93025; font-size: 14px; margin-top: 15px; display: none; font-weight: 500; }
        
        /* สัดส่วนวิดีโอ 9:16 สำหรับ Mobile UI */
        .video-wrapper { 
            width: 100%; aspect-ratio: 9 / 16; background: #000; 
            border-radius: 16px; overflow: hidden; display: none; margin-top: 10px; box-shadow: 0 4px 12px rgba(0,0,0,0.1);
        }
        video { width: 100%; height: 100%; object-fit: cover; }
        img.secure-img { width: 100%; height: 100%; object-fit: contain; display: none; }
    </style>
</head>
<body oncontextmenu="return false;"> <div class="app-container">
        <div id="login-section">
            <div class="logo-placeholder">SL</div>
            <h2 id="title-text">กำลังโหลด...</h2>
            <p id="prompt-text">โปรดรอสักครู่</p>
            
            <input type="text" id="userInput" placeholder="กรอกข้อมูลที่นี่..." required>
            <button onclick="verifyAccess()" id="verify-btn">เข้าสู่ระบบ</button>
            <p id="error-msg" class="error-text">ข้อมูลไม่ถูกต้อง คุณไม่มีสิทธิ์เข้าถึงข้อมูลนี้</p>
        </div>

        <div id="file-section" class="video-wrapper">
            <video id="secure-video" controls controlsList="nodownload" disablePictureInPicture></video>
            <img id="secure-img" class="secure-img" alt="Secure Content" ondragstart="return false;"/>
        </div>
    </div>

    <script>
        const SUPABASE_URL = 'https://qspfyaqsnbtyzqcvfzsd.supabase.co';
        const SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InFzcGZ5YXFzbmJ0eXpxY3ZmenNkIiwicm9sZSI6ImFub24iLCJpYXQiOjE3ODIxMTgzOTMsImV4cCI6MjA5NzY5NDM5M30.je7dDqR3XnHP3SwNgUCM4Ae2DnajhLGAUh0wnSKCh2c';
        const supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

        const urlParams = new URLSearchParams(window.location.search);
        const currentFileId = urlParams.get('id'); 

        let fileDataPath = '';
        let allowedUsers = [];

        async function loadSystemConfig() {
            if (!currentFileId) {
                document.getElementById('title-text').innerText = "ข้อผิดพลาดของระบบ";
                document.getElementById('prompt-text').innerText = "ไม่พบ ID ของไฟล์ โปรดตรวจสอบลิงก์อีกครั้ง";
                document.getElementById('userInput').style.display = 'none';
                document.getElementById('verify-btn').style.display = 'none';
                return;
            }

            const { data, error } = await supabase
                .from('system_files')
                .select('file_title, login_prompt, storage_path, allowed_credentials')
                .eq('id', currentFileId)
                .single();

            if (data) {
                document.getElementById('title-text').innerText = data.file_title;
                document.getElementById('prompt-text').innerText = data.login_prompt;
                fileDataPath = data.storage_path;
                allowedUsers = data.allowed_credentials;
            } else {
                document.getElementById('title-text').innerText = "ไม่พบข้อมูล";
                document.getElementById('prompt-text').innerText = "ข้อมูลนี้อาจถูกลบหรือไม่มีอยู่ในระบบ";
                document.getElementById('userInput').style.display = 'none';
                document.getElementById('verify-btn').style.display = 'none';
            }
        }

        async function verifyAccess() {
            const input = document.getElementById('userInput').value.trim();
            const errorMsg = document.getElementById('error-msg');
            const verifyBtn = document.getElementById('verify-btn');
            
            verifyBtn.innerText = 'กำลังตรวจสอบ...';
            verifyBtn.disabled = true;

            if (allowedUsers.includes(input)) {
                errorMsg.style.display = 'none';
                
                // สร้าง Signed URL อายุ 60 วินาที
                const { data, error } = await supabase.storage
                    .from('secure_vault')
                    .createSignedUrl(fileDataPath, 60);

                if (data) {
                    document.getElementById('login-section').style.display = 'none';
                    document.getElementById('file-section').style.display = 'block';
                    
                    // เช็คว่าเป็นรูปภาพหรือวิดีโอ (อย่างง่าย)
                    if (fileDataPath.match(/\.(jpeg|jpg|gif|png)$/i)) {
                        const imgEl = document.getElementById('secure-img');
                        imgEl.src = data.signedUrl;
                        imgEl.style.display = 'block';
                        document.getElementById('secure-video').style.display = 'none';
                    } else {
                        document.getElementById('secure-video').src = data.signedUrl;
                    }
                } else {
                    alert("เกิดข้อผิดพลาดในการดึงไฟล์ความปลอดภัย");
                    verifyBtn.innerText = 'เข้าสู่ระบบ';
                    verifyBtn.disabled = false;
                }
            } else {
                errorMsg.style.display = 'block';
                verifyBtn.innerText = 'เข้าสู่ระบบ';
                verifyBtn.disabled = false;
            }
        }

        window.onload = loadSystemConfig;
    </script>
</body>
</html>
