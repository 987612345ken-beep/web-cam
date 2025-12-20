from flask import Flask, Response, render_template_string, request, url_for, send_from_directory
import cv2, time, os, datetime, subprocess, atexit, shutil
import threading, requests
import pyaudio
import wave,tempfile
import json,re,logging
from apscheduler.schedulers.background import BackgroundScheduler # 新增：排程器
import psutil # 新增：用於系統資訊和硬碟檢查
from moviepy.editor import VideoFileClip, AudioFileClip, ImageSequenceClip
from apscheduler.events import EVENT_JOB_ERROR, EVENT_JOB_MISSED
import datetime
# --- 檔案路徑與配置 ---
CONFIG_FILE = 'config.json'
DEFAULT_CONFIG = {
    "frame_width": 720,
    "frame_height": 480,
    "frame_rate": 20.0,
    "schedule_enabled": False,
    "recording_schedule": "08:00-18:00", # 格式: HH:MM-HH:MM
    "disk_cleanup_enabled": True,
    "disk_threshold_gb": 10, # 剩餘空間低於此值開始清理
    "save_dir": "video_backup"
}

# --- 全域變數與鎖定 ---
app = Flask(__name__)
# 配置變數將在 load_config() 中初始化
config = DEFAULT_CONFIG.copy() 
camera = None 
recording = False
video_writer = None
audio_thread = None
audio_stop_event = threading.Event()
last_record_path_video = None
last_record_path_audio = None
last_record_path_final = None
lock = threading.Lock()
scheduler = BackgroundScheduler() # 新增：排程器實例
frame_paths = [] # 用於儲存每一幀圖片的路徑
frame_counter = 0 # 幀計數
camera = cv2.VideoCapture(0)
# --- 設定：從配置中讀取 ---
fourcc = cv2.VideoWriter_fourcc(*'mp4v')
FFMPEG_TIMEOUT = 120 

# 音訊設定
CHUNK = 1024
FORMAT = pyaudio.paInt16
CHANNELS = 1
RATE = 44100
p = pyaudio.PyAudio()
for i in range(p.get_device_count()):
    info = p.get_device_info_by_index(i)
    if info['maxInputChannels'] > 0:
        print(f"🎤 找到麥克風 -> Index {i}: {info['name']}")
p.terminate()
logging.basicConfig()
logging.getLogger('apscheduler').setLevel(logging.DEBUG)
@app.route('/status')
def get_status():
    global recording
    return {"recording": recording}
def job_listener(event):
    if event.exception:
        print(f"🚨 排程任務發生崩潰: {event.exception}")
    elif event.code == EVENT_JOB_MISSED:
        print("⚠️ 排程任務錯過了執行時間（可能是時區或系統卡頓）")
def heartbeat():
    print(f"💓 排程器運作中... 當前時間: {datetime.datetime.now().strftime('%H:%M:%S')}")
def update_scheduler():
    global scheduler, config
    scheduler.remove_all_jobs()
    
    # 硬碟清理任務保持不變
    scheduler.add_job(check_disk_space_and_cleanup, 'interval', hours=1, id='disk_cleanup')

    if not config.get('schedule_enabled'):
        return

    try:
        times = config['recording_schedule'].replace(" ", "").split('-')
        s_h, s_m = map(int, times[0].split(':'))
        e_h, e_m = map(int, times[1].split(':'))

        # 1. 設定總體的「開啟」與「關閉」
        scheduler.add_job(start_recording, 'cron', hour=s_h, minute=s_m, id='scheduled_start')
        scheduler.add_job(stop_recording, 'cron', hour=e_h, minute=e_m, id='scheduled_stop')

        # 2. ✅ 設定「分段切換」任務 (例如每 30 分鐘切換一次檔案)
        # 注意：我們加上 jitter=10 防止與其他任務完全重疊
        scheduler.add_job(
            rotate_recording, 
            'interval', 
            minutes=30, 
            id='recording_rotate',
            jitter=10 
        )

        print(f"✅ 排程設定成功：{times[0]}-{times[1]} (每 30 分鐘自動分段)")
        
    except Exception as e:
        print(f"🚨 排程設定解析失敗: {e}")
# --------------------------------------------------
# 【配置管理函式】
# --------------------------------------------------
def start_cloudflare_tunnel():
    """啟動 Cloudflare Tunnel 並抓取產生的網址"""
    # 確保 cloudflared.exe 路徑正確 (若在同資料夾則直接寫檔名)
    cmd = ["cloudflared.exe", "tunnel", "--url", "http://127.0.0.1:5000"]
    
    # 啟動子程序
    process = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True, encoding='utf-8')
    
    print("正在建立 Cloudflare 隧道...")
    
    # 從輸出內容中找尋網址 (通常長得像 https://xxx.trycloudflare.com)
    for line in process.stdout:
        # 使用正規表示法抓取網址
        url_match = re.search(r'https://[a-zA-Z0-9-]+\.trycloudflare\.com', line)
        if url_match:
            public_url = url_match.group(0)
            print("\n" + "="*50)
            print(f"🚀 監控系統已成功推送到外網！")
            print(f"🔗 外網存取網址: {public_url}")
            print("="*50 + "\n")
            break
def load_config():
    """載入配置並強制修正中文路徑問題"""
    global config, camera
    try:
        if os.path.exists(CONFIG_FILE):
            with open(CONFIG_FILE, 'r', encoding='utf-8') as f:
                loaded_config = json.load(f)
                config.update(loaded_config)
    except Exception as e:
        print(f"⚠️ 載入配置失敗: {e}")

    # 🚨 核心修正：強制將中文路徑改為英文，避免 cv2.imwrite 失敗
    if config['save_dir'] == "影像備份" or not config['save_dir']:
        config['save_dir'] = "video_backup"
        save_config()
        print("✅ 已將儲存目錄強制修正為英文: video_backup")

    # 確保目錄存在 (使用絕對路徑)
    abs_save_dir = os.path.abspath(config['save_dir'])
    if not os.path.exists(abs_save_dir):
        os.makedirs(abs_save_dir)
    
    # 初始化攝影機
    if camera is None:
        camera = cv2.VideoCapture(0)
    return config
def save_config():
    """使用暫存檔機制確保寫入安全"""
    global config
    try:
        # 1. 確保儲存目錄存在
        if not os.path.exists(config['save_dir']):
            os.makedirs(config['save_dir'], exist_ok=True)

        # 2. 取得 config.json 的絕對路徑
        file_path = os.path.abspath(CONFIG_FILE)
        
        # 3. 先寫入到暫存檔
        with tempfile.NamedTemporaryFile('w', delete=False, dir=os.getcwd(), encoding='utf-8') as tf:
            json.dump(config, tf, indent=4, ensure_ascii=False)
            temp_name = tf.name
        
        # 4. 將暫存檔替換掉舊的 config.json (原子性操作)
        if os.path.exists(file_path):
            os.remove(file_path)
        os.rename(temp_name, file_path)
        
        print("✅ 配置儲存成功")
        return True
    except Exception as e:
        print(f"❌ 儲存配置失敗，原因: {e}")
        return False
def rotate_recording():
    """每 15 分鐘自動切換新檔案"""
    global recording
    # 如果現在根本沒在錄影（例如手動停止了或不在排程內），就不要動作
    if not recording:
        return

    print("🔄 [自動分段] 達到 15 分鐘，正在存檔並開啟新片段...")
    
    # 1. 停止目前的錄影（這會觸發背景合併）
    stop_recording()
    
    # 2. 短暫延遲，確保變數已釋放
    time.sleep(1.5)
    
    # 3. 重新啟動錄影（會產生新的 Timestamp 檔案路徑）
    start_recording()
# --- 核心收音與合成邏輯 ---        
def audio_recording_task(path):
    """獨立執行緒：持續從麥克風讀取數據並寫入 WAV 檔"""
    p = pyaudio.PyAudio()
    try:
        stream = p.open(format=FORMAT, channels=CHANNELS, rate=RATE, input=True, frames_per_buffer=CHUNK)
        frames = []
        while not audio_stop_event.is_set():
            data = stream.read(CHUNK, exception_on_overflow=False)
            frames.append(data)
        stream.stop_stream()
        stream.close()
        with wave.open(path, 'wb') as wf:
            wf.setnchannels(CHANNELS)
            wf.setsampwidth(p.get_sample_size(FORMAT))
            wf.setframerate(RATE)
            wf.writeframes(b''.join(frames))
    except Exception as e: print(f"收音錯誤: {e}")
    finally: p.terminate()
# --------------------------------------------------
# 【資源管理函式】
# --------------------------------------------------

def check_disk_space_and_cleanup():
    """檢查硬碟空間，如果低於閾值，則刪除最舊的檔案。"""
    global config
    
    # 確保只在啟用清理且配置目錄存在時執行
    if not config['disk_cleanup_enabled'] or not os.path.isdir(config['save_dir']):
        return
        
    try:
        # 檢查硬碟使用情況
        total, used, free = shutil.disk_usage(config['save_dir'])
        free_gb = free / (1024 ** 3)
        threshold = config['disk_threshold_gb']

        print(f"💾 硬碟檢查：剩餘空間 {free_gb:.2f} GB (閾值 {threshold} GB)。")

        if free_gb < threshold:
            print(f"🔥 空間不足！開始清理最舊的檔案...")
            
            files_to_delete = []
            for filename in os.listdir(config['save_dir']):
                if filename.endswith('.mp4') and not filename.startswith('temp_'):
                    filepath = os.path.join(config['save_dir'], filename)
                    # 獲取檔案的修改時間 (mtime) 作為排序依據
                    files_to_delete.append((os.path.getmtime(filepath), filepath))
            
            # 依照時間戳記升序排列 (最舊的在前面)
            files_to_delete.sort(key=lambda x: x[0])
            
            # 刪除直到空間足夠或沒有檔案可刪
            for _, filepath in files_to_delete:
                if free_gb < threshold:
                    try:
                        os.remove(filepath)
                        print(f"   - 已刪除最舊檔案: {os.path.basename(filepath)}")
                        # 重新檢查空間 (簡化處理，實際應用中可更精確計算)
                        total, used, free = shutil.disk_usage(config['save_dir'])
                        free_gb = free / (1024 ** 3)
                    except OSError as e:
                        print(f"   - 刪除檔案 {os.path.basename(filepath)} 失敗: {e}")
                else:
                    break # 空間足夠，停止刪除
            
            if free_gb >= threshold:
                 print("✅ 清理完成，空間已恢復到安全閾值之上。")
            else:
                 print("⚠️ 已刪除所有可刪除檔案，但空間仍低於閾值。")

    except Exception as e:
        print(f" 硬碟空間檢查/清理時發生錯誤: {e}")

# --------------------------------------------------
# 【排程管理函式】
# --------------------------------------------------

def update_scheduler():
    global scheduler, config
    scheduler.remove_all_jobs()
    
    # 維護任務：每小時檢查硬碟
    scheduler.add_job(check_disk_space_and_cleanup, 'interval', hours=1, id='disk_cleanup')

    if not config.get('schedule_enabled'):
        print("📢 排程錄影功能：關閉")
        return

    try:
        # 解析時段 (如 08:00-18:00)
        times = config['recording_schedule'].replace(" ", "").split('-')
        s_h, s_m = map(int, times[0].split(':'))
        e_h, e_m = map(int, times[1].split(':'))

        # 1. 每天定時【開啟】錄影
        scheduler.add_job(start_recording, 'cron', hour=s_h, minute=s_m, id='scheduled_start')
        
        # 2. 每天定時【停止】錄影
        scheduler.add_job(stop_recording, 'cron', hour=e_h, minute=e_m, id='scheduled_stop')

        # 3. ✅ 每 15 分鐘自動執行一次分段切換
        # jitter=5 表示隨機正負 5 秒，避免跟其他任務在同一秒鐘打架
        scheduler.add_job(
            rotate_recording, 
            'interval', 
            minutes=15, 
            id='recording_rotate',
            jitter=5 
        )

        print(f"✅ 排程設定成功：{times[0]}-{times[1]} (每 15 分鐘自動切換檔案)")
        
        # 顯示清單確保三個任務都在
        jobs = [j.id for j in scheduler.get_jobs()]
        print(f"🔍 當前活躍任務: {jobs}")
        
    except Exception as e:
        print(f"🚨 排程設定失敗: {e}")
# 在 scheduler.start() 之後加入：
print(f"⏰ 系統目前時間: {datetime.datetime.now()}")
job_stop = scheduler.get_job('scheduled_stop')
if job_stop:
    print(f"📅 排程器認定的『停止時間』為: {job_stop.next_run_time}")
else:
    print("❌ 找不到 scheduled_stop 任務！")        
# --------------------------------------------------
# 【錄影與音訊控制函式 (核心邏輯不變，但使用 config 變數)】
# --------------------------------------------------
def audio_recording_task(file_path):
    global recording
    p = pyaudio.PyAudio()
    
    # --- 關鍵修正 1：確保父資料夾 100% 存在 ---
    try:
        folder = os.path.dirname(file_path)
        if not os.path.exists(folder):
            os.makedirs(folder, exist_ok=True)
            print(f"📁 已自動建立遺失的資料夾: {folder}")
    except Exception as e:
        print(f"🚨 無法建立資料夾: {e}")
        return

    print(f"🎙️ 音訊線程啟動，目標：{file_path}")
    
    try:
        # 使用與手動錄影一致的參數
        stream = p.open(format=FORMAT, channels=CHANNELS, rate=RATE, 
                        input=True, frames_per_buffer=CHUNK)
        
        frames = []
        while recording:
            try:
                data = stream.read(CHUNK, exception_on_overflow=False)
                frames.append(data)
            except Exception as e:
                print(f"⚠️ 讀取音訊流跳過: {e}")
                break
            
        stream.stop_stream()
        stream.close()
        
        # --- 關鍵修正 2：使用二進位寫入模式確保檔案產生 ---
        with wave.open(file_path, 'wb') as wf:
            wf.setnchannels(CHANNELS)
            wf.setsampwidth(p.get_sample_size(FORMAT))
            wf.setframerate(RATE)
            wf.writeframes(b''.join(frames))
            
        if os.path.exists(file_path):
            print(f"✅ 音訊錄製成功，大小: {os.path.getsize(file_path)} bytes")
        
    except Exception as e:
        print(f"🎧 [錄音失敗] 詳細原因: {e}")
    finally:
        p.terminate()

def start_audio_recording(audio_path):
    global audio_thread, audio_stop_event
    audio_stop_event.clear()
    # 這裡必須將 audio_path 作為 args 傳給任務函式
    audio_thread = threading.Thread(target=audio_recording_task, args=(audio_path,), daemon=True)
    audio_thread.start()
    print(f"🎙️ 音訊線程已手動啟動")
    
def stop_recording():
    global recording
    if not recording:
        print("🛑 [排程停止] 系統目前並未在錄影中，跳過。")
        return False
        
    print("🛑 [排程停止] 執行中...")
    
    # 1. 關鍵：同時變更錄影狀態與發送音訊停止訊號
    with lock:
        recording = False
    audio_stop_event.set() # 確保音訊執行緒收到停止訊號

    # 2. 背景執行合併，避免阻塞主程式
    def delayed_combine():
        time.sleep(2) # 緩衝時間，確保檔案寫入磁碟
        combine_to_mp4(last_record_path_video, last_record_path_audio, last_record_path_final)
        
    threading.Thread(target=delayed_combine, daemon=True).start()
    return True

def combine_to_mp4(frames_dir, audio_path, final_output):
    """將圖片序列與音訊合併 (強化音訊處理版)"""
    video_clip = None
    audio_clip = None
    try:
        print(f"🎬 開始合成影片: {final_output}")
        images = sorted([os.path.join(frames_dir, img) for img in os.listdir(frames_dir) if img.endswith(".png")])
        
        if not images:
            print("❌ 錯誤：找不到幀圖片")
            return

        # 1. 建立影像軌
        video_clip = ImageSequenceClip(images, fps=config['frame_rate'])
        
        # 2. 檢查音軌檔案是否存在且大於 44 bytes (WAV Header 大小)
        if os.path.exists(audio_path) and os.path.getsize(audio_path) > 1000:
            print(f"🎵 檢測到音軌，大小: {os.path.getsize(audio_path)} bytes")
            audio_clip = AudioFileClip(audio_path)
            
            # 強制將音訊長度修剪或延伸至影像長度，避免合成失敗
            audio_clip = audio_clip.set_duration(video_clip.duration)
            
            # 設定音訊到影片
            video_clip = video_clip.set_audio(audio_clip)
        else:
            print("⚠️ 警告：找不到音訊檔或檔案過小，將生成無聲影片。")

        # 3. 寫入檔案 (aac 是一般瀏覽器最相容的音訊格式)
        video_clip.write_videofile(
            final_output, 
            codec="libx264", 
            audio_codec="aac", 
            temp_audiofile="temp-audio.m4a", # 避免檔名衝突
            remove_temp=True, 
            logger=None
        )
        print(f"✅ 錄影存檔成功: {final_output}")

    except Exception as e:
        print(f"🚨 合併失敗: {e}")
    finally:
        # 必須手動關閉 Clip 以釋放檔案鎖定，否則之後無法刪除暫存檔
        if video_clip: video_clip.close()
        if audio_clip: audio_clip.close()
        
        # 清理暫存 (延遲 1 秒確保檔案已釋放)
        time.sleep(1)
        shutil.rmtree(frames_dir, ignore_errors=True)
        if os.path.exists(audio_path): 
            try: os.remove(audio_path)
            except: pass
def start_recording():
    global recording, last_record_path_video, last_record_path_audio, last_record_path_final, frame_counter
    
    if recording:
        print("🟡 排程觸發，但系統已在錄影中，跳過。")
        return False

    # 1. 確保基礎目錄 (使用絕對路徑)
    save_dir = os.path.abspath(config.get('save_dir', 'video_backup'))
    os.makedirs(save_dir, exist_ok=True)

    # 2. 生成檔名
    ts = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    last_record_path_video = os.path.join(save_dir, f"temp_frames_{ts}")
    last_record_path_audio = os.path.join(save_dir, f"temp_audio_{ts}.wav")
    last_record_path_final = os.path.join(save_dir, f"record_{ts}.mp4")

    # 3. 建立圖片暫存夾
    os.makedirs(last_record_path_video, exist_ok=True)
    
    # 4. 重置狀態並啟動錄音
    frame_counter = 0
    recording = True # 必須在啟動 Thread 前設為 True
    
    threading.Thread(target=audio_recording_task, args=(last_record_path_audio,), daemon=True).start()
    
    print(f"🟢 [排程/自動] 錄影開始：{last_record_path_final}")
    return True
def stop_recording():
    global recording, last_record_path_video, last_record_path_audio, last_record_path_final
    
    if not recording:
        print("🛑 [排程停止] 系統目前並未在錄影中，略過停止指令。")
        return False
        
    print("🛑 [排程停止] 觸發定時停止錄影...")
    
    # 1. 變更狀態
    with lock:
        recording = False
    
    # 2. 停止音訊並合併
    try:
       
        
        if last_record_path_video and os.path.exists(last_record_path_video):
            # 啟動背景合併
            threading.Thread(
                target=combine_to_mp4, 
                args=(last_record_path_video, last_record_path_audio, last_record_path_final),
                daemon=True
            ).start()
            print("🚀 合併線程啟動，錄影結束。")
        else:
            print("⚠️ 找不到錄影暫存檔，無法合併。")
    except Exception as e:
        print(f"❌ 停止錄影過程中發生錯誤: {e}")

    return True
# --------------------------------------------------
# 【Flask 路由：新增/修改 設定頁面】
# --------------------------------------------------
# 保持 generate_frames, video, download_file 路由不變
def gen_frames():
    global recording, last_record_path_video, frame_counter, camera 
    
    while True:
        success, frame = camera.read()
        if not success:
            break
        
        # 錄影模式：將當前幀存為圖片
        if recording and last_record_path_video: 
            try:
                frame_counter += 1
                # 這裡的變數必須與 record_action 定義的一致
                img_path = os.path.join(last_record_path_video, f"frame_{frame_counter:06d}.png")
                cv2.imwrite(img_path, frame)
            except Exception as e:
                print(f"❌ 幀寫入失敗: {e}")

        # 加上浮水印
        timestamp = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        cv2.putText(frame, f"CAMMER1 | {timestamp}", (10, 30),
                    cv2.FONT_HERSHEY_SIMPLEX, 0.8, (255, 255, 255), 2)
        
        # 串流輸出
        ret, buffer = cv2.imencode('.jpg', frame)
        yield (b'--frame\r\n' b'Content-Type: image/jpeg\r\n\r\n' + buffer.tobytes() + b'\r\n')
@app.route('/')
def index():
    return render_template_string('''
    <html>
    <head><title>監控儀表板</title>
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <style>
            body { background: #111; color: white !important; font-family: sans-serif; text-align: center; padding: 20px; margin: 0; }
            .status { margin: 15px; font-weight: bold; font-size: 1.2em; }
            .control-panel { 
                display: flex; justify-content: center; align-items: center; 
                gap: 15px; flex-wrap: wrap; margin-top: 30px; max-width: 800px; margin-left: auto; margin-right: auto;
            }
            .btn { 
                /* 統一按鈕大小的核心設定 */
                width: 160px; height: 50px; 
                font-size: 16px; font-weight: bold; cursor: pointer; border: none; 
                border-radius: 8px; color: white !important; text-decoration: none; 
                transition: 0.3s; display: inline-flex; align-items: center; justify-content: center;
                box-sizing: border-box;
            }
            .btn-start { background: #28a745; }
            .btn-stop { background: #dc3545; }
            .btn-list { background: #007bff; }
            .btn-settings { background: #6c757d; }
            .btn:hover { opacity: 0.8; filter: brightness(1.1); transform: translateY(-2px); }
            .btn:active { transform: translateY(0); }
            img { border: 3px solid #333; width: 95%; max-width: 850px; border-radius: 12px; box-shadow: 0 4px 20px rgba(0,0,0,0.6); }
            form { margin: 0; }
        </style>
    </head>
    <body>
       <div id="status-text" class="status">
            系統狀態: {{ "🔴 正在錄影" if recording else "⚪ 待機中" }}
        </div>
        
        <img src="{{ url_for('video') }}">

        <div class="control-panel">
            </div>

        <div class="control-panel">
            <form action="/record" method="post">
                <button name="action" value="start" class="btn btn-start">▶️ 開始錄影</button>
            </form>
            <form action="/record" method="post">
                <button name="action" value="stop" class="btn btn-stop">⏹️ 停止錄影</button>
            </form>
            <a href="{{ url_for('list_videos') }}" class="btn btn-list">📂 檔案管理</a>
            <a href="{{ url_for('settings') }}" class="btn btn-settings">⚙️ 系統設定</a>
        </div>
        <script>
            // 每 3 秒檢查一次後端狀態
            setInterval(function() {
                fetch('/status')
                    .then(response => response.json())
                    .then(data => {
                        const statusDiv = document.getElementById('status-text');
                        if (data.recording) {
                            statusDiv.innerHTML = "系統狀態: <span class='status-red'>🔴 正在錄影</span>";
                        } else {
                            statusDiv.innerHTML = "系統狀態: <span class='status-gray'>⚪ 待機中</span>";
                        }
                    });
            }, 3000); 
        </script>
    </body>
    </html>
    ''', recording=recording, config=config) # 不再傳遞 free_gb
@app.route('/video')
def video():
    return Response(gen_frames(), mimetype='multipart/x-mixed-replace; boundary=frame')
@app.route('/record', methods=['POST'])
def record_action():
    global recording, last_record_path_video, last_record_path_audio, last_record_path_final, frame_counter
    action = request.form.get('action')

    # 1. 處理「開始錄影」
    if action == 'start':
        if not recording:
            ts = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
            save_dir = os.path.abspath(config['save_dir'])
            os.makedirs(save_dir, exist_ok=True)
            
            last_record_path_video = os.path.join(save_dir, f"temp_frames_{ts}")
            last_record_path_audio = os.path.join(save_dir, f"temp_audio_{ts}.wav")
            last_record_path_final = os.path.join(save_dir, f"record_{ts}.mp4")
            
            os.makedirs(last_record_path_video, exist_ok=True)
            frame_counter = 0
            audio_stop_event.clear()
            
            # 啟動錄音 (傳入 path 參數)
            threading.Thread(target=audio_recording_task, args=(last_record_path_audio,), daemon=True).start()
            recording = True
            print(f"🟢 手動啟動錄影成功: {last_record_path_final}")
        else:
            print("🟡 系統已在錄影中，忽略開始指令。")

    # 2. 處理「停止錄影」
    elif action == 'stop':
        if recording:
            recording = False
            audio_stop_event.set()
            
            # 在背景合併
            threading.Thread(target=combine_to_mp4, 
                             args=(last_record_path_video, last_record_path_audio, last_record_path_final),
                             daemon=True).start()
            print("🛑 手動停止錄影，啟動背景合併...")
        else:
            print("⚪ 系統目前不在錄影狀態，忽略停止指令。")

    # 🚀 關鍵修正：確保所有路徑最後都會執行這個 return
    return f"<script>window.location.href='{url_for('index')}';</script>"
@app.route('/settings', methods=['GET', 'POST'])
def settings():
    global config
    message = ""
    if request.method == 'POST':
        try:
            # 建立暫存配置，避免轉型失敗時毀損全域變數
            new_config = config.copy()
            new_config['frame_width'] = int(request.form.get('frame_width'))
            new_config['frame_height'] = int(request.form.get('frame_height'))
            new_config['frame_rate'] = float(request.form.get('frame_rate'))
            new_config['schedule_enabled'] = 'schedule_enabled' in request.form
            new_config['recording_schedule'] = request.form.get('recording_schedule')
            new_config['disk_cleanup_enabled'] = 'disk_cleanup_enabled' in request.form
            new_config['disk_threshold_gb'] = int(request.form.get('disk_threshold_gb'))

            # 更新全域變數
            config.update(new_config)
            
            if save_config():
                update_scheduler()
                message = "✅ 設定已成功更新並生效！"
            else:
                message = "❌ 檔案寫入失敗，請檢查硬碟權限。"
        except ValueError:
            message = "❌ 格式錯誤：請確保寬度、高度與閾值填入的是數字。"
        except Exception as e:
            message = f"❌ 系統錯誤: {e}"

    # ----------------------------------------
    # 2. 獲取即時狀態資訊 (保持不變)
    # ----------------------------------------
    
    # 獲取硬碟資訊
    try:
        disk_info = shutil.disk_usage(os.path.join(os.getcwd(), config['save_dir']).split(os.path.sep)[0])
        free_gb = disk_info.free / (1024 ** 3)
        total_gb = disk_info.total / (1024 ** 3)
        disk_status = f"剩餘 {free_gb:.2f} GB / 總共 {total_gb:.2f} GB"
    except Exception:
        free_gb = 0.0
        disk_status = "無法讀取硬碟資訊"

    # 獲取 CPU 和記憶體使用率
    try:
        cpu_usage = psutil.cpu_percent(interval=None)
        mem_info = psutil.virtual_memory()
        mem_usage = mem_info.percent
        mem_total = mem_info.total / (1024 ** 3)
    except Exception:
        cpu_usage = "N/A"
        mem_usage = "N/A"
        mem_total = 0.0

    # 獲取錄影狀態
    with lock:
        current_recording_status = recording
        
    return render_template_string('''
    <html>
    <head><title>系統設定與狀態</title>
        <style>
            body { background:#111; color:white !important; font-family: sans-serif; padding: 20px; }
            h1 { color: white; }
            .back-link a { color: white; text-decoration: none; }
            
            /* --- 新增佈局樣式 --- */
            .content-container {
                display: flex;       /* 啟用 Flexbox */
                justify-content: center; /* 讓內容居中 */
                gap: 30px;           /* 區塊之間的間距 */
                max-width: 1200px;
                margin: 20px auto;
                flex-wrap: wrap;     /* 當螢幕縮小時，允許換行 */
            }

            .status-box, form {
                background: #222; 
                padding: 20px; 
                border-radius: 8px;
                flex: 1; /* 讓兩個區塊平均分配剩餘空間 */
                min-width: 300px; /* 最小寬度，防止太窄 */
            }
            
            .status-box { background: #333; margin-bottom: 0; } /* 調整背景色和邊距 */
            /* --- 其他樣式保持不變 --- */
            .form-group { margin-bottom: 15px; }
            .form-group label { display: block; margin-bottom: 5px; font-weight: bold; }
            .form-group input[type="text"], .form-group input[type="number"] { width: 100%; padding: 8px; box-sizing: border-box; background: #333; border: 1px solid #555; color: white; border-radius: 4px; }
            .form-group input[type="checkbox"] { margin-right: 10px; }
            button { padding: 10px 15px; background-color: #ADD8E6; color: #111; border: none; border-radius: 4px; cursor: pointer; font-size: 1em; margin-top: 15px; }
            .message { padding: 10px; margin-bottom: 15px; border-radius: 4px; background-color: #f9e300; color: #111; text-align: center; }
            .status-box p { margin: 5px 0; }
        </style>
    </head>
    <body>
        <div class="back-link"><a href="{{ url_for('index') }}">⬅️ 返回監視器畫面</a></div>
        <h1>系統設定與狀態</h1>
        {% if message %}<div class="message">{{ message }}</div>{% endif %}

        <div class="content-container">
            <div class="status-box">
                <h2>📊 即時系統狀態</h2>
                <p><strong>錄影狀態:</strong> <span style="color:{{ 'lime' if recording else 'red' }};">{{ '🔴 正在錄影' if recording else '⛔ 已停止錄影' }}</span></p>
                <p><strong>CPU 使用率:</strong> {{ cpu_usage }}%</p>
                <p><strong>記憶體使用率:</strong> {{ mem_usage }}% (總共 {{ "%.2f"|format(mem_total) }} GB)</p>
                <p><strong>硬碟空間:</strong> {{ disk_status }}</p>
                <p><strong>清理閾值:</strong> {{ config.get('disk_threshold_gb') }} GB</p>
                <p><strong>排程狀態:</strong> {{ '✅ 啟用' if config.get('schedule_enabled') else '❌ 停用' }} (時段: {{ config.get('recording_schedule', 'N/A') }})</p>
                <p><strong>錄影參數:</strong> {{ config.get('frame_width') }}x{{ config.get('frame_height') }} @ {{ config.get('frame_rate') }} FPS</p>
            </div>
            
            <form method="POST">
                <h2>⚙️ 配置修改</h2>
                
                <h2>錄影參數</h2>
                <div class="form-group">
                    <label for="frame_width">寬度 (px):</label>
                    <input type="number" name="frame_width" value="{{ config.get('frame_width') }}">
                </div>
                <div class="form-group">
                    <label for="frame_height">高度 (px):</label>
                    <input type="number" name="frame_height" value="{{ config.get('frame_height') }}">
                </div>
                <div class="form-group">
                    <label for="frame_rate">幀率 (FPS):</label>
                    <input type="number" step="0.1" name="frame_rate" value="{{ config.get('frame_rate') }}">
                </div>

                <h2>排程錄影</h2>
                <div class="form-group">
                    <input type="checkbox" name="schedule_enabled" id="schedule_enabled" {% if config.get('schedule_enabled') %}checked{% endif %}>
                    <label for="schedule_enabled" style="display:inline;">啟用排程</label>
                </div>
                <div class="form-group">
                    <label for="recording_schedule">錄影時段 (HH:MM-HH:MM):</label>
                    <input type="text" name="recording_schedule" value="{{ config.get('recording_schedule') }}">
                </div>

                <h2>資源管理 (硬碟清理)</h2>
                <div class="form-group">
                    <input type="checkbox" name="disk_cleanup_enabled" id="disk_cleanup_enabled" {% if config.get('disk_cleanup_enabled') %}checked{% endif %}>
                    <label for="disk_cleanup_enabled" style="display:inline;">啟用自動清理</label>
                </div>
                <div class="form-group">
                    <label for="disk_threshold_gb">最低剩餘空間閾值 (GB):</label>
                    <input type="number" name="disk_threshold_gb" value="{{ config.get('disk_threshold_gb') }}">
                    <span>（低於此值將刪除最舊檔案）</span>
                </div>

                <button type="submit">儲存設定並套用</button>
            </form>
        </div>
    </body>
    </html>
    ''', config=config, message=message, recording=current_recording_status, 
        cpu_usage=cpu_usage, mem_usage=mem_usage, mem_total=mem_total, disk_status=disk_status)
@app.route('/videos')
def list_videos():
    """檔案管理清單"""
    save_dir = config.get('save_dir', 'video_backup')
    abs_path = os.path.abspath(save_dir)
    
    if not os.path.exists(abs_path):
        os.makedirs(abs_path)
        
    files = [f for f in os.listdir(abs_path) if f.endswith('.mp4')]
    files.sort(reverse=True)
    
    return render_template_string('''
    <html>
    <head><title>檔案管理</title>
       <style>
           <style>
           body { background:#111; color:white !important; font-family: sans-serif; text-align: center; padding: 20px; }
        /* 加大的返回按鈕樣式 */
        .btn-back { 
            display: inline-flex; align-items: center; justify-content: center;
            width: 180px; height: 50px; background: #444; color: white !important; 
            text-decoration: none; border-radius: 8px; font-weight: bold; margin-bottom: 20px; transition: 0.3s;
        }
        .btn-back:hover { background: #555; transform: scale(1.05); }
        .box { background: #222; padding: 25px; border-radius: 12px; display: inline-block; text-align: left; border: 1px solid #444; }
        input { width: 100%; padding: 10px; margin: 10px 0; background: #333; color: white; border: 1px solid #555; border-radius: 5px; }
        .btn-save { background: #28a745; color: white !important; padding: 12px; border: none; cursor: pointer; width: 100%; border-radius: 8px; font-weight: bold; }
    /* 檔案列表行容器 */
        li { 
            background: #222; margin: 12px 0; padding: 15px; border-radius: 10px; 
            display: flex; justify-content: space-between; align-items: center; 
            border: 1px solid #333; gap: 15px;
        }
        
        /* 檔案名稱樣式 - 確保與按鈕同一行且不被擠壓 */
        .file-info { 
            flex-grow: 1; text-align: left; overflow: hidden; 
            text-overflow: ellipsis; white-space: nowrap; color: white;
        }
        
        /* 按鈕群組 */
        .btn-group { display: flex; gap: 10px; flex-shrink: 0; }
        
        .btn-s { 
            padding: 8px 18px; color: white !important; text-decoration: none; 
            border-radius: 6px; font-size: 14px; font-weight: bold; border: none; cursor: pointer;
            transition: 0.2s; min-width: 70px; text-align: center;
        }
        .bg-g { background: #28a745; } .bg-r { background: #dc3545; }
        .btn-s:hover { opacity: 0.8; filter: brightness(1.2); }
    </style></head>
    <body>
        <a href="/" class="btn-back">⬅️ 返回監視器畫面</a>
        <h1>📂 錄影檔案管理</h1>
        <ul>
        {% for f in files %}
            <li>
                <div class="file-info">{{ f }}</div>
                <div class="btn-group">
                    <a href="{{ url_for('download_file', filename=f) }}" class="btn-s bg-g">下載</a>
                    <form action="{{ url_for('delete_file', filename=f) }}" method="POST" style="margin:0;">
                        <button type="submit" class="btn-s bg-r" onclick="return confirm('確定刪除？')">刪除</button>
                    </form>
                </div>
            </li>
        {% endfor %}
        </ul>
    </body>
    </html>
    ''', files=files)
@app.route('/download/<filename>')
def download_file(filename):
    """修正後的下載功能"""
    abs_save_dir = os.path.abspath(config.get('save_dir', 'video_backup'))
    try:
        return send_from_directory(directory=abs_save_dir, path=filename, as_attachment=True)
    except Exception as e:
        return f"下載失敗: {e}", 404
@app.route('/delete/<filename>', methods=['POST'])
def delete_file(filename):
    """新增的刪除功能"""
    abs_save_dir = os.path.abspath(config.get('save_dir', 'video_backup'))
    file_path = os.path.join(abs_save_dir, filename)
    if os.path.exists(file_path):
        os.remove(file_path)
    return "<script>window.location.href='/videos';</script>"
# --------------------------------------------------
# 【主程式入口】
# --------------------------------------------------

if __name__ == '__main__':
    load_config()
    
    # 1. 啟動排程器
    scheduler.start() 
    
    # 2. 更新任務
    update_scheduler()
    
    # 3. 強制檢查
    print("\n" + "="*30)
    all_jobs = scheduler.get_jobs()
    print(f"📊 目前共有 {len(all_jobs)} 個排程任務執行中")
    for j in all_jobs:
        print(f"📌 任務 ID: {j.id} | 下次執行: {j.next_run_time}")
    print("="*30 + "\n")

    app.run(host='0.0.0.0', port=5000, debug=False, threaded=True)
