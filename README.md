from flask import Flask, Response, render_template_string, request
import cv2, time, os, datetime, subprocess, atexit
import threading 
# --- 全域變數與鎖定 ---
app = Flask(__name__)
camera = cv2.VideoCapture(0)
recording = False
video_writer = None
last_record_path = None
#  新增鎖定機制，保護 recording 和 video_writer 等共享變數
lock = threading.Lock()
# --- 設定 ---
fourcc = cv2.VideoWriter_fourcc(*'mp4v')
camera.set(cv2.CAP_PROP_FRAME_WIDTH, 720)
camera.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
camera.set(cv2.CAP_PROP_FPS, 30) 

# 儲存資料夾：使用相對路徑，提高可移植性
save_dir = os.path.join(os.getcwd(), "影像備份")
os.makedirs(save_dir, exist_ok=True)

@atexit.register
def cleanup():
    # 確保攝影機和 VideoWriter 在程序結束時釋放
    global video_writer
    if camera.isOpened():
        camera.release()
    
    # 確保釋放 VideoWriter
    if video_writer:
        video_writer.release()
# --------------------------------------------------
# 【Flask 路由與視訊串流】
# --------------------------------------------------
def generate_frames():
    global recording, video_writer
    while True:
        success, frame = camera.read()
        if not success:
            break
        
        timestamp = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        cv2.putText(frame, f"CAMMER1 | {timestamp}", (10, 30),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.8, (255, 255, 255), 2)
        
        #  使用鎖定保護 video_writer 的寫入操作
        with lock:
             if recording and video_writer:
                 video_writer.write(frame)

        # 串流輸出
        ret, buffer = cv2.imencode('.jpg', frame)
        frame = buffer.tobytes()
        yield (b'--frame\r\n'
               b'Content-Type: image/jpeg\r\n\r\n' + frame + b'\r\n')

@app.route('/')
def index():
    #  使用鎖定讀取 recording 狀態
    with lock:
        current_recording_status = recording
        
    return render_template_string('''
    <html>
    <head><title>串流錄影系統</title></head>
    <body style="margin:0; background:#111; display:flex; flex-direction:column; align-items:center; justify-content:center; height:100vh; color:white;">
        <img src="/video" style="max-width:100%; height:auto;" />
        <h2 style="margin-top:20px; color:{{ 'lime' if recording else 'red' }};">
            {{ '🔴 正在錄影' if recording else '⛔ 已停止錄影' }}
        </h2>
        <form method="POST" action="/record" style="margin-top:10px;">
            <button type="submit" name="action" value="start" style="padding:10px 20px;">▶️ 開始錄影</button>
            <button type="submit" name="action" value="stop" style="padding:10px 20px;">⏹ 停止錄影</button>
        </form>
    </body>
    </html>
    ''', recording=current_recording_status) # 傳入被鎖定保護的狀態

@app.route('/video')
def video():
    return Response(generate_frames(), mimetype='multipart/x-mixed-replace; boundary=frame')

@app.route('/record', methods=['POST'])
def record():
    global recording, video_writer, last_record_path
    action = request.form.get('action')

    #  使用鎖定保護所有對全局錄影狀態的修改和存取
    with lock:
        if action == 'start' and not recording:
            timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
            filename = f"record_{timestamp}.mp4"
            full_path = os.path.join(save_dir, filename)
            video_writer = cv2.VideoWriter(full_path, fourcc, 15.0, (720, 480)) # 記得 FPS 也改為 15.0
            recording = True
            last_record_path = filename
            print(f"🎥 錄影開始：{full_path}")

        elif action == 'stop' and recording:
            recording = False
            if video_writer:
                video_writer.release()
                video_writer = None
            print("🛑 錄影停止")
            
    # 呼叫 index 時，也會再次使用鎖定讀取狀態
    return index()

@app.route('/health')
def health():
    return "OK", 200

if __name__ == '__main__':
    # 確保 Flask 伺服器有時間完成埠號綁定
    print(" 等待 3 秒，確保本地 Flask 服務已啟動...")
    time.sleep(3) 
    # 2. 在主執行緒中啟動 Flask 伺服器
    print(f"\n 啟動 Flask 伺服器...")
    app.run(host='0.0.0.0' ,port=5000)
