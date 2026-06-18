#!/usr/bin/env python3
# Objedinennyy skript: server + terminalnyy interfeys v odnom fayle
# Zapusk: python main.py

import http.server
import socketserver
import socket
import threading
import time
import sys
import os
import json
from datetime import datetime

# ============================================
# GLOBAL DEĞİŞKENLER
# ============================================
PORT = 8080

# ASCII sanatı ile korkutucu logo - base64 yerine direkt HTML canvas kullanıyor
html_content = """<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Site</title>
<style>
*{margin:0;padding:0;box-sizing:border-box}
body{
    background:#000;
    font-family:'Courier New',monospace;
    overflow:hidden;
    height:100vh;
    cursor:crosshair;
    display:flex;
    flex-direction:column;
    justify-content:center;
    align-items:center;
}
canvas#logo{
    display:block;
    z-index:10;
    filter:drop-shadow(0 0 30px #ff0000) drop-shadow(0 0 60px #8b0000);
    animation:pulseGlow 1.5s ease-in-out infinite;
}
@keyframes pulseGlow{
    0%,100%{filter:drop-shadow(0 0 30px #ff0000) drop-shadow(0 0 60px #8b0000)}
    50%{filter:drop-shadow(0 0 50px #ff4444) drop-shadow(0 0 100px #ff0000) drop-shadow(0 0 150px #8b0000)}
}
#zeid-text{
    color:#cc0000;
    font-size:38px;
    font-weight:bold;
    text-align:center;
    margin-top:30px;
    z-index:10;
    text-shadow:0 0 20px #ff0000,0 0 40px #8b0000,0 0 80px #550000;
    animation:textFlicker 3s infinite;
    letter-spacing:4px;
}
@keyframes textFlicker{
    0%,100%{opacity:1}
    5%{opacity:0.8}
    10%{opacity:1}
    15%{opacity:0.6}
    20%{opacity:1}
    60%{opacity:1}
    62%{opacity:0.3}
    64%{opacity:1}
    80%{opacity:1}
    82%{opacity:0.5}
    84%{opacity:1}
}
.floating-text{
    position:absolute;
    font-size:24px;
    font-weight:bold;
    pointer-events:none;
    z-index:1;
}
@keyframes shake{
    0%{transform:translate(1px,1px) rotate(0deg)}
    10%{transform:translate(-1px,-2px) rotate(-1deg)}
    20%{transform:translate(-3px,0px) rotate(1deg)}
    30%{transform:translate(3px,2px) rotate(0deg)}
    40%{transform:translate(1px,-1px) rotate(1deg)}
    50%{transform:translate(-1px,2px) rotate(-1deg)}
    60%{transform:translate(-3px,1px) rotate(0deg)}
    70%{transform:translate(3px,1px) rotate(-1deg)}
    80%{transform:translate(-1px,-1px) rotate(1deg)}
    90%{transform:translate(1px,2px) rotate(0deg)}
    100%{transform:translate(1px,-2px) rotate(-1deg)}
}
</style>
</head>
<body>

<canvas id="logo" width="280" height="280"></canvas>
<div id="zeid-text">He is Ze1dd de Cassa</div>

<script>
// ============ KORKUTUCU PENTAGRAM / ŞEYTANİ LOGO ÇİZİMİ ============
const canvas = document.getElementById('logo');
const ctx = canvas.getContext('2d');
const cx = 140, cy = 140, r = 100;

function drawPentagram(time){
    ctx.clearRect(0,0,280,280);
    
    // Dış halka - dönen
    ctx.save();
    ctx.translate(cx,cy);
    ctx.rotate(time * 0.001);
    ctx.strokeStyle = '#ff0000';
    ctx.lineWidth = 3;
    ctx.shadowColor = '#ff0000';
    ctx.shadowBlur = 20;
    ctx.beginPath();
    ctx.arc(0,0,r+15,0,Math.PI*2);
    ctx.stroke();
    
    // İkinci halka
    ctx.rotate(-time * 0.002);
    ctx.strokeStyle = '#8b0000';
    ctx.lineWidth = 2;
    ctx.shadowBlur = 15;
    ctx.beginPath();
    ctx.arc(0,0,r+8,0,Math.PI*2);
    ctx.stroke();
    ctx.restore();
    
    // Pentagram yıldızı
    const points = [];
    for(let i=0;i<5;i++){
        let angle = (i * 4 * Math.PI / 5) - Math.PI/2;
        points.push({
            x: cx + r * Math.cos(angle),
            y: cy + r * Math.sin(angle)
        });
    }
    
    // Yıldız çizimi
    ctx.strokeStyle = '#ff1111';
    ctx.lineWidth = 4;
    ctx.shadowColor = '#ff0000';
    ctx.shadowBlur = 25;
    ctx.beginPath();
    ctx.moveTo(points[0].x, points[0].y);
    ctx.lineTo(points[2].x, points[2].y);
    ctx.lineTo(points[4].x, points[4].y);
    ctx.lineTo(points[1].x, points[1].y);
    ctx.lineTo(points[3].x, points[3].y);
    ctx.closePath();
    ctx.stroke();
    
    // İç çember
    ctx.beginPath();
    ctx.arc(cx,cy,r*0.38,0,Math.PI*2);
    ctx.strokeStyle = '#cc0000';
    ctx.lineWidth = 3;
    ctx.shadowBlur = 30;
    ctx.stroke();
    
    // Merkez nokta - parlayan göz
    const pulseSize = 6 + Math.sin(time*0.005)*3;
    ctx.beginPath();
    ctx.arc(cx,cy,pulseSize,0,Math.PI*2);
    ctx.fillStyle = '#ff4444';
    ctx.shadowBlur = 40;
    ctx.shadowColor = '#ff0000';
    ctx.fill();
    
    // Göz bebeği
    ctx.beginPath();
    ctx.arc(cx,cy,3,0,Math.PI*2);
    ctx.fillStyle = '#000';
    ctx.shadowBlur = 0;
    ctx.fill();
    
    // Şeytani semboller - yıldız uçlarında
    const symbols = ['?','?','?','?','?'];
    ctx.font = 'bold 24px serif';
    ctx.fillStyle = '#ff2222';
    ctx.shadowBlur = 15;
    ctx.shadowColor = '#ff0000';
    for(let i=0;i<5;i++){
        let angle = (i * 4 * Math.PI / 5) - Math.PI/2;
        let sx = cx + (r+30) * Math.cos(angle);
        let sy = cy + (r+30) * Math.sin(angle);
        ctx.fillText(symbols[i], sx-10, sy+8);
    }
    
    requestAnimationFrame(drawPentagram);
}

requestAnimationFrame(drawPentagram);

// ============ ARKA PLAN YAZILARI ============
for(let i=0;i<50;i++){
    let el = document.createElement('div');
    el.className = 'floating-text';
    el.textContent = 'Cehenneme Ho?geldin';
    el.style.left = Math.random()*(window.innerWidth-200)+'px';
    el.style.top = Math.random()*(window.innerHeight-50)+'px';
    el.style.color = '#'+Math.floor(Math.random()*16777215).toString(16);
    el.style.animation = 'shake 0.5s infinite';
    document.body.appendChild(el);
}

setInterval(()=>{
    document.querySelectorAll('.floating-text').forEach(el=>{
        el.style.left = Math.random()*(window.innerWidth-200)+'px';
        el.style.top = Math.random()*(window.innerHeight-50)+'px';
    });
},100);

setInterval(()=>{
    document.body.style.backgroundColor = '#'+Math.floor(Math.random()*16777215).toString(16);
},200);
</script>
</body>
</html>"""

# ============================================
# CUSTOM HTTP İSTEK İŞLEYİCİ
# ============================================
class CustomHandler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        client_ip = self.client_address[0]
        timestamp = datetime.now().strftime("%H:%M:%S")
        
        # IP ADRESİNİ DOĞRUDAN ANA EKRANA YAZDIR
        print(f"\n  ╔══════════════════════════════════════════════════════╗")
        print(f"  ║  [+] BAĞLANTI ALGILANDI                              ║")
        print(f"  ║  [?] Zaman  : {timestamp}                            ║")
        print(f"  ║  [!] IP     : {client_ip}                            ║")
        print(f"  ║  [*] Port   : {self.client_address[1]}               ║")
        print(f"  ╚══════════════════════════════════════════════════════╝")
        sys.stdout.flush()
        
        self.send_response(200)
        self.send_header('Content-type', 'text/html; charset=utf-8')
        self.end_headers()
        self.wfile.write(html_content.encode('utf-8'))
    
    # Sunucu loglarini tamamen sustur
    def log_message(self, format, *args):
        return

# ============================================
# SUNUCU THREAD'I
# ============================================
class ServerThread(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self)
        self.daemon = True
        self.httpd = None
    
    def run(self):
        try:
            self.httpd = socketserver.TCPServer(("", PORT), CustomHandler)
            self.httpd.serve_forever()
        except OSError as e:
            print(f"\n[!] Port {PORT} kullanımda. Server başlatılamadı.")
    
    def stop(self):
        if self.httpd:
            self.httpd.shutdown()
            self.httpd.server_close()

# ============================================
# TERMINAL ARAYÜZÜ
# ============================================
def clear_screen():
    os.system('clear' if os.name != 'nt' else 'cls')

def print_banner():
    banner = """
  ╔══════════════════════════════════════════════════════════╗
  ║                                                          ║
  ║    ██████╗███████╗██╗  ██╗███████╗███╗   ██╗███╗   ██╗ ███████╗███╗   ███╗
  ║   ██╔════╝██╔════╝██║  ██║██╔════╝████╗  ██║████╗  ██║ ██╔════╝████╗ ████║
  ║   ██║     █████╗  ███████║█████╗  ██╔██╗ ██║██╔██╗ ██║ █████╗  ██╔████╔██║
  ║   ██║     ██╔══╝  ██╔══██║██╔══╝  ██║╚██╗██║██║╚██╗██║ ██╔══╝  ██║╚██╔╝██║
  ║   ╚██████╗███████╗██║  ██║███████╗██║ ╚████║██║ ╚████║ ███████╗██║ ╚═╝ ██║
  ║    ╚═════╝╚══════╝╚═╝  ╚═╝╚══════╝╚═╝  ╚═══╝╚═╝  ╚═══╝ ╚══════╝╚═╝     ╚═╝
  ║                                                          ║
  ╚══════════════════════════════════════════════════════════╝
  """
    print(banner)

def print_status(server_running):
    status = "AKTIF" if server_running else "BEKLIYOR"
    color = "\033[92m" if server_running else "\033[91m"
    print(f"  [?] Sunucu Durumu : {color}{status}\033[0m")
    print(f"  [?] Port          : {PORT}")
    print(f"  [?] Serveo Komutu : ssh -R 80:localhost:{PORT} serveo.net")
    print(f"  ══════════════════════════════════════════════════════")
    print(f"  [?] Bağlantılar aşağıda gerçek zamanlı görünecek:")
    print(f"  ──────────────────────────────────────────────────────")

def main():
    server_thread = None
    server_running = False
    
    clear_screen()
    print_banner()
    print_status(server_running)
    
    try:
        while True:
            cmd = input("\n  [?] Komut (1:başlat, 2:durdur, 3:çıkış) > ").strip()
            
            if cmd == '1':
                if not server_running:
                    print("  [+] Sunucu başlatılıyor...")
                    server_thread = ServerThread()
                    server_thread.start()
                    time.sleep(0.5)
                    server_running = True
                    print(f"  [+] Sunucu başarıyla başlatıldı. Port: {PORT}")
                    print(f"  [+] Serveo komutu: ssh -R 80:localhost:{PORT} serveo.net")
                    print(f"  [*] BAĞLANTILAR İZLENİYOR...")
                    print(f"  ──────────────────────────────────────────────────────")
                else:
                    print("  [!] Sunucu zaten çalışıyor.")
            
            elif cmd == '2':
                if server_running and server_thread:
                    print("  [!] Sunucu durduruluyor...")
                    server_thread.stop()
                    server_running = False
                    print("  [+] Sunucu durduruldu.")
                else:
                    print("  [!] Sunucu zaten kapalı.")
            
            elif cmd == '3':
                if server_running and server_thread:
                    server_thread.stop()
                print("  [+] Çıkış yapılıyor...")
                sys.exit(0)
            
            else:
                print("  [!] Geçersiz komut. 1:başlat, 2:durdur, 3:çıkış")
    
    except KeyboardInterrupt:
        if server_running and server_thread:
            server_thread.stop()
        print("\n  [+] Program sonlandırıldı.")
        sys.exit(0)

if __name__ == "__main__":
    main()