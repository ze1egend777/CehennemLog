
    #!/usr/bin/env python3
# -*- coding: utf-8 -*-
import http.server
import socketserver
import threading
import time
import sys
import os
import subprocess
import signal
import re
import json
import urllib.request
from datetime import datetime

PORT = 8080
CLOUDFLARED_PATH = None

# ============================================
# VPN / PROXY TESPİT SERVİSİ
# ============================================
def check_vpn_proxy(ip):
    """
    IP adresinin VPN, proxy, barındırma veya veri merkezi olup olmadığını kontrol eder.
    ip-api.com ücretsiz API'si kullanılır.
    Dönen değer: (is_vpn: bool, detay: str)
    """
    try:
        url = f"http://ip-api.com/json/{ip}?fields=proxy,hosting,isp,org,as"
        req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
        with urllib.request.urlopen(req, timeout=5) as response:
            data = json.loads(response.read().decode('utf-8'))
        
        is_proxy = data.get('proxy', False)
        is_hosting = data.get('hosting', False)
        isp = data.get('isp', 'Bilinmiyor')
        org = data.get('org', 'Bilinmiyor')
        asn = data.get('as', 'Bilinmiyor')
        
        # VPN/Proxy barındırma sağlayıcılarının ASN'lerini kontrol et
        vpn_asn_keywords = [
            'M247', 'Datacamp', 'PacketHub', 'RoyaleHosting', 'IT Hosting',
            'RackNerd', 'ColoCrossing', 'DigitalOcean', 'Vultr', 'Linode',
            'Hetzner', 'OVH', 'Contabo', 'BuyVM', 'HostHatch', 'NetCup',
            'AlexHost', 'PQ HOSTING', 'DediPath', 'Continent 8',
            'NordVPN', 'ExpressVPN', 'ProtonVPN', 'CyberGhost', 'Surfshark',
            'Private Internet', 'Windscribe', 'HideMyAss', 'IPVanish',
            'Tor Exit', 'Mullvad', 'IVPN', 'AirVPN', 'Perfect Privacy',
            'VPN', 'PROXY', 'HOSTING', 'DATACENTER', 'SERVER'
        ]
        
        # ISP ve Org kontrolü
        isp_org = (isp + ' ' + org + ' ' + asn).upper()
        for keyword in vpn_asn_keywords:
            if keyword.upper() in isp_org:
                return True, f"VPN/Proxy tespit edildi: {keyword}"
        
        if is_proxy or is_hosting:
            return True, f"Proxy/Hosting tespit edildi (ISP: {isp})"
        
        return False, "Temiz bağlantı"
    
    except Exception as e:
        # API hatasında güvenlik için bağlantıyı reddet
        return True, f"API hatası - bağlantı reddedildi ({str(e)[:50]})"

# ============================================
# HTML - PENTAGRAM KALDIRILDI, SADECE YAZILAR
# ============================================
HTML = """<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>.</title>
<style>
*{margin:0;padding:0;box-sizing:border-box}
body{background:#000;font-family:'Courier New',monospace;overflow:hidden;height:100vh;cursor:crosshair;display:flex;justify-content:center;align-items:center}
.floating-text{position:absolute;font-size:24px;font-weight:bold;pointer-events:none;z-index:1;color:#ff0000 !important}
@keyframes shake{0%{transform:translate(1px,1px) rotate(0deg)}10%{transform:translate(-1px,-2px) rotate(-1deg)}20%{transform:translate(-3px,0px) rotate(1deg)}30%{transform:translate(3px,2px) rotate(0deg)}40%{transform:translate(1px,-1px) rotate(1deg)}50%{transform:translate(-1px,2px) rotate(-1deg)}60%{transform:translate(-3px,1px) rotate(0deg)}70%{transform:translate(3px,1px) rotate(-1deg)}80%{transform:translate(-1px,-1px) rotate(1deg)}90%{transform:translate(1px,2px) rotate(0deg)}100%{transform:translate(1px,-2px) rotate(-1deg)}}
</style>
</head>
<body>
<script>
for(let i=0;i<50;i++){
    let el=document.createElement('div');
    el.className='floating-text';
    el.textContent='Cehenneme Hoşgeldin';
    el.style.left=Math.random()*(window.innerWidth-200)+'px';
    el.style.top=Math.random()*(window.innerHeight-50)+'px';
    el.style.color='#ff0000';
    el.style.animation='shake 0.5s infinite';
    document.body.appendChild(el);
}
setInterval(()=>{
    document.querySelectorAll('.floating-text').forEach(el=>{
        el.style.left=Math.random()*(window.innerWidth-200)+'px';
        el.style.top=Math.random()*(window.innerHeight-50)+'px';
    });
},100);
</script>
</body>
</html>"""

# VPN ile girenlere gösterilecek sayfa
HTML_BLOCKED = """<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Erişim Engellendi</title>
<style>
*{margin:0;padding:0;box-sizing:border-box}
body{background:#000;font-family:'Courier New',monospace;display:flex;justify-content:center;align-items:center;height:100vh}
.msg{color:#ff0000;text-align:center;border:3px solid #ff0000;padding:40px;max-width:500px}
.msg h1{font-size:48px;margin-bottom:20px}
.msg p{font-size:20px}
</style>
</head>
<body>
<div class="msg">
<h1>ERISIM ENGELLENDI</h1>
<p>VPN/Proxy baglantisi tespit edildi.</p>
<p>Lutfen VPN'inizi kapatip tekrar deneyin.</p>
</div>
</body>
</html>"""

# ============================================
# HTTP İŞLEYİCİ - VPN/PROXY KONTROLLÜ
# ============================================
class Handler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        # Gerçek IP'yi X-Forwarded-For header'ından al
        real_ip = self.headers.get('X-Forwarded-For')
        if real_ip:
            real_ip = real_ip.split(',')[0].strip()
        else:
            real_ip = self.headers.get('X-Real-IP')
        if not real_ip:
            real_ip = self.client_address[0]
        
        now = datetime.now().strftime("%H:%M:%S")
        
        # VPN/Proxy kontrolü
        is_vpn, vpn_detail = check_vpn_proxy(real_ip)
        
        if is_vpn:
            # VPN'li bağlantı -> engelle
            print(f"\033[91m[{now}] \033[93m[BLOKE] \033[97m{real_ip} \033[90m- {vpn_detail}\033[0m")
            sys.stdout.flush()
            
            self.send_response(403)
            self.send_header('Content-type','text/html; charset=utf-8')
            self.end_headers()
            self.wfile.write(HTML_BLOCKED.encode('utf-8'))
        else:
            # Temiz bağlantı -> siteyi göster
            print(f"\033[91m[{now}] \033[92m>> \033[97m{real_ip}\033[0m")
            sys.stdout.flush()
            
            self.send_response(200)
            self.send_header('Content-type','text/html; charset=utf-8')
            self.end_headers()
            self.wfile.write(HTML.encode('utf-8'))
    
    def log_message(self, *args):
        pass

# ============================================
# SUNUCU THREAD
# ============================================
class ServerThread(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self)
        self.daemon = True
        self.httpd = None
    def run(self):
        try:
            self.httpd = socketserver.TCPServer(("",PORT), Handler)
            self.httpd.serve_forever()
        except OSError:
            print(f"\033[91m[!] Port {PORT} kullaniliyor.\033[0m")
    def stop(self):
        if self.httpd:
            self.httpd.shutdown()
            self.httpd.server_close()

# ============================================
# CLOUDFLARED YÖNETİMİ
# ============================================
def find_cloudflared():
    global CLOUDFLARED_PATH
    for path in os.environ.get('PATH', '').split(':'):
        full = os.path.join(path, 'cloudflared')
        if os.path.isfile(full) and os.access(full, os.X_OK):
            CLOUDFLARED_PATH = full
            return True
    possible = [
        '/data/data/com.termux/files/usr/bin/cloudflared',
        os.path.expanduser('~/../usr/bin/cloudflared'),
        'cloudflared'
    ]
    for p in possible:
        try:
            result = subprocess.run([p, '--version'], capture_output=True, timeout=3)
            if result.returncode == 0:
                CLOUDFLARED_PATH = p
                return True
        except:
            continue
    return False

def install_cloudflared():
    print("\033[93m[*] cloudflared kuruluyor...\033[0m")
    result = subprocess.run('pkg install cloudflared -y', shell=True, capture_output=True, text=True)
    if result.returncode != 0:
        print("\033[91m[!] cloudflared kurulamadi. Elle kurun: pkg install cloudflared -y\033[0m")
        return False
    return find_cloudflared()

class CloudflaredTunnel:
    def __init__(self):
        self.process = None
        self.url = None
    def start(self):
        if not CLOUDFLARED_PATH:
            return False
        try:
            self.process = subprocess.Popen(
                [CLOUDFLARED_PATH, 'tunnel', '--url', f'http://localhost:{PORT}'],
                stdout=subprocess.PIPE,
                stderr=subprocess.STDOUT,
                text=True,
                bufsize=1
            )
            start_time = time.time()
            while time.time() - start_time < 30:
                line = self.process.stdout.readline()
                if not line:
                    time.sleep(0.1)
                    continue
                if 'trycloudflare.com' in line:
                    match = re.search(r'https://[a-z0-9-]+\.trycloudflare\.com', line)
                    if match:
                        self.url = match.group(0)
                        return True
                if 'error' in line.lower() or 'failed' in line.lower():
                    print(f"\033[91m[!] Cloudflared: {line.strip()}\033[0m")
                    return False
            return False
        except Exception as e:
            print(f"\033[91m[!] Cloudflared başlatılamadı: {e}\033[0m")
            return False
    def stop(self):
        if self.process:
            self.process.terminate()
            try:
                self.process.wait(timeout=5)
            except:
                self.process.kill()
            self.process = None
            self.url = None

# ============================================
# TERMINAL ARAYÜZÜ
# ============================================
class TerminalUI:
    def __init__(self):
        self.server = None
        self.tunnel = CloudflaredTunnel()
        self.server_running = False
        self.tunnel_running = False
        self.connection_count = 0
        self.blocked_count = 0
        self.public_url = None
    def clear(self):
        os.system('clear 2>/dev/null || printf "\\033c"')
    def banner(self):
        print("""
\033[91m
    ╔══════════════════════════════════════════════════════╗
    ║                                                      ║
    ║         ██████╗███████╗██╗  ██╗███████╗███╗   ██╗   ║
    ║        ██╔════╝██╔════╝██║  ██║██╔════╝████╗  ██║   ║
    ║        ██║     █████╗  ███████║█████╗  ██╔██╗ ██║   ║
    ║        ██║     ██╔══╝  ██╔══██║██╔══╝  ██║╚██╗██║   ║
    ║        ╚██████╗███████╗██║  ██║███████╗██║ ╚████║   ║
    ║         ╚═════╝╚══════╝╚═╝  ╚═╝╚══════╝╚═╝  ╚═══╝   ║
    ║                                                      ║
    ║              CLOUDFLARED + VPN KORUMALI              ║
    ╚══════════════════════════════════════════════════════╝
\033[0m""")
    def status_bar(self):
        s_color = "\033[92m" if self.server_running else "\033[91m"
        t_color = "\033[92m" if self.tunnel_running else "\033[91m"
        print(f"""
\033[90m┌─────────────────────────────────────────────────────────┐\033[0m
\033[90m│\033[0m  \033[97mSunucu:\033[0m {s_color}{'AKTIF' if self.server_running else 'KAPALI'}\033[0m     \033[90m│\033[0m  \033[97mPort:\033[0m \033[93m{PORT}\033[0m           \033[90m│\033[0m
\033[90m│\033[0m  \033[97mTünel :\033[0m {t_color}{'AKTIF' if self.tunnel_running else 'KAPALI'}\033[0m     \033[90m│\033[0m  \033[97mBağlantı:\033[0m \033[92m{self.connection_count}\033[0m     \033[90m│\033[0m
\033[90m│\033[0m  \033[97mVPN Bl:\033[0m \033[93m{self.blocked_count}\033[0m                                          \033[90m│\033[0m
\033[90m├─────────────────────────────────────────────────────────┤\033[0m""")
        if self.public_url:
            print(f"\033[90m│\033[0m  \033[97mURL:\033[0m \033[96m{self.public_url}\033[0m")
            print(f"\033[90m├─────────────────────────────────────────────────────────┤\033[0m")
        print(f"\033[90m│\033[0m  \033[97mKomutlar: \033[93m1\033[0m-Başlat \033[93m2\033[0m-Durdur \033[93m3\033[0m-Tünel \033[93m4\033[0m-Çıkış   \033[90m│\033[0m")
        print(f"\033[90m└─────────────────────────────────────────────────────────┘\033[0m")
    def add_connection(self):
        self.connection_count += 1
    def add_blocked(self):
        self.blocked_count += 1
    def start_server(self):
        if self.server_running:
            print("\033[93m[!] Sunucu zaten calisiyor.\033[0m")
            return
        print("\033[93m[*] Sunucu baslatiliyor...\033[0m")
        self.server = ServerThread()
        self.server.start()
        time.sleep(0.5)
        self.server_running = True
        print("\033[92m[+] Sunucu baslatildi.\033[0m")
    def stop_server(self):
        if not self.server_running:
            print("\033[93m[!] Sunucu zaten kapali.\033[0m")
            return
        if self.tunnel_running:
            self.stop_tunnel()
        print("\033[93m[*] Sunucu durduruluyor...\033[0m")
        self.server.stop()
        self.server_running = False
        print("\033[92m[+] Sunucu durduruldu.\033[0m")
    def start_tunnel(self):
        if not self.server_running:
            print("\033[91m[!] Once sunucuyu baslatin (Komut: 1)\033[0m")
            return
        if self.tunnel_running:
            print("\033[93m[!] Tunel zaten calisiyor.\033[0m")
            return
        if not CLOUDFLARED_PATH:
            if not find_cloudflared():
                if not install_cloudflared():
                    return
        print("\033[93m[*] Cloudflare tuneli baslatiliyor...\033[0m")
        print("\033[90m[*] URL alinmasi bekleniyor (max 30 saniye)...\033[0m")
        if self.tunnel.start():
            self.tunnel_running = True
            self.public_url = self.tunnel.url
            print(f"\033[92m[+] Tunel aktif!\033[0m")
            print(f"\033[96m[+] URL: {self.public_url}\033[0m")
        else:
            print("\033[91m[!] Tunel baslatilamadi.\033[0m")
    def stop_tunnel(self):
        if not self.tunnel_running:
            return
        self.tunnel.stop()
        self.tunnel_running = False
        self.public_url = None
        print("\033[92m[+] Tunel durduruldu.\033[0m")

# ============================================
# ANA FONKSİYON
# ============================================
def main():
    ui = TerminalUI()
    if not find_cloudflared():
        install_cloudflared()
    def signal_handler(sig, frame):
        print("\n\033[93m[*] Kapatiliyor...\033[0m")
        if ui.tunnel_running:
            ui.tunnel.stop()
        if ui.server_running:
            ui.server.stop()
        sys.exit(0)
    signal.signal(signal.SIGINT, signal_handler)
    while True:
        ui.clear()
        ui.banner()
        ui.status_bar()
        try:
            cmd = input("\033[97m> \033[0m").strip()
        except EOFError:
            signal_handler(None, None)
        if cmd == '1':
            ui.start_server()
            input("\033[90m[Enter] devam...\033[0m")
        elif cmd == '2':
            ui.stop_server()
            input("\033[90m[Enter] devam...\033[0m")
        elif cmd == '3':
            ui.start_tunnel()
            input("\033[90m[Enter] devam...\033[0m")
        elif cmd == '4':
            if ui.server_running:
                ui.stop_server()
            print("\033[92m[+] Cikis yapildi.\033[0m")
            sys.exit(0)
        else:
            print("\033[91m[!] Gecersiz komut.\033[0m")
            time.sleep(1)

if __name__ == "__main__":
    main()