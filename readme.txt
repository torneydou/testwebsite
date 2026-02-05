YouTube Music Backup Setup Guide
1. Install Python

    Download Python 3.10 or higher from python.org.

    Run the installer.

    Important: Check the box labeled "Add Python to PATH" before clicking Install.

2. Install Required Libraries

Open Command Prompt and run the following command:

	pip install ytmusicapi mutagen Pillow thefuzz yt-dlp

3. Download External Tools


Open Command Prompt and run the following command:

	winget install ffmpeg && winget install yt-dlp

4. Authentication (Setup browser.json)

    Open Firefox and go to music.youtube.com.

    Log in to your account.

    Press F12 to open Developer Tools and select the Network tab.

    Click on your Liked Songs playlist.

    In the Network tab "Filter" box, type: browse

    Right-click the result, select Copy Value, then Copy Request Headers.

    Open Command Prompt in your script folder and run:

	ytmusicapi browser

    Paste the headers when prompted and press Ctrl+Z then Enter (or follow the on-screen instructions) to save the file.

5. Create/Download and run the Script

If you downloaded the existing sript 

Double-click your Python script or open a command prompt in the folder with the script and run:

	python Backup_Script.py 

If you want to paste it in yourself, create a txt file, paste in the following script, save the file as Backup_Script.py and then double-click your Python script or open a command prompt in the folder with the script and run:

	python Backup_Script.py 

----------
the script
----------
import json
import os
import subprocess
import time
from ytmusicapi import YTMusic
from datetime import datetime
from mutagen.id3 import ID3, APIC, ID3NoHeaderError
from PIL import Image
from io import BytesIO
from thefuzz import fuzz

# ---------------------------------------------------------
# 1. IMAGE PROCESSING FUNCTIONS
# ---------------------------------------------------------
def center_crop(img):
    w, h = img.size
    side = min(w, h)
    left = (w - side) // 2
    top = (h - side) // 2
    return img.crop((left, top, left + side, top + side))

def process_mp3_cover(path):
    """Bakes a square-cropped JPEG into the MP3 metadata."""
    try:
        try:
            audio = ID3(path)
        except ID3NoHeaderError:
            audio = ID3()
            audio.save(path)
            audio = ID3(path)

        pics = audio.getall('APIC')
        if not pics:
            audio.update_to_v23()
            pics = audio.getall('APIC')

        if not pics:
            return

        tag = pics[0]
        img = Image.open(BytesIO(tag.data))
        img = center_crop(img)

        if img.mode in ("RGBA", "LA"):
            img = img.convert("RGB")

        buffer = BytesIO()
        img.save(buffer, format='JPEG', quality=95)

        audio.delall('APIC')
        audio.add(APIC(
            encoding=3,
            mime='image/jpeg',
            type=3,
            desc='Cover',
            data=buffer.getvalue()
        ))
        audio.save(v2_version=3)
        print(f"  [Art] Square crop applied: {os.path.basename(path)}")
    except Exception as e:
        print(f"  [Art Error] {os.path.basename(path)}: {e}")

# ---------------------------------------------------------
# 2. FILE COMPARISON FUNCTIONS
# ---------------------------------------------------------
def is_already_downloaded(target_title, folder_path, threshold=90):
    """Checks if a similar filename exists in the folder."""
    if not os.path.exists(folder_path):
        return False, None
        
    existing_files = [f for f in os.listdir(folder_path) if f.lower().endswith(".mp3")]
    
    for file_name in existing_files:
        clean_name = file_name.rsplit('.', 1)[0]
        if "[" in clean_name:
            clean_name = clean_name.split("[")[0].strip()
            
        similarity = fuzz.token_sort_ratio(target_title.lower(), clean_name.lower())
        if similarity >= threshold:
            return True, file_name
    return False, None

# ---------------------------------------------------------
# 3. MAIN BACKUP LOGIC
# ---------------------------------------------------------
def run_backup():
    # --- CONSTANTS & PATHS ---
    base_path = os.path.join(os.path.expanduser("~"), "Documents", "ytmusicbackup", "liked_songs")
    music_folder = os.path.join(os.path.expanduser("~"), "Music")
    download_path = os.path.join(music_folder, "YTMusic Songs Backup")
    ffmpeg_path = os.path.expandvars(r'%userprofile%\AppData\Local\Microsoft\WinGet\Links\ffmpeg.exe')

    # --- AUTHENTICATION CHECK ---
    if not os.path.exists('browser.json'):
        if not os.path.exists(base_path): 
            os.makedirs(base_path)

        print("="*60)
        print("ERROR: browser.json not found!")
        print("="*60)
        print("\n[STEP 1: Getting response headers]")
        print("1. Go to music.youtube.com and login to the desired account.")
        print("2. (In Firefox) Press F12.")
        print("3. Click on your 'Liked Songs' playlist.")
        print("4. In the Developer Tools at the bottom, select the 'Network' tab.")
        print("5. Click on 'Filter URLs' and type: browse")
        print("6. Right click on the result > Copy Value > Copy Request Headers")
        
        print("\n[STEP 2: Getting browser.json]")
        print("1. Create a folder (anywhere you want with any name you want)")
        print("2. Put this script in that folder if you haven't already")
        print("3. Open a command prompt in the folder with your script (right click > open in terminal).")
        print(f"4. Run: cd /d %CD% && ytmusicapi browser")
        print("5. Follow instructions on the command prompt (paste headers when asked).")
        print("\n" + "="*60)
        input("\nPress Enter to close and run again once browser.json is ready...")
        return

    try:
        # --- DIRECTORY SETUP ---
        for path in [base_path, download_path]:
            if not os.path.exists(path): os.makedirs(path)

        # --- FETCH LATEST LIKES ---
        print("Connecting to YouTube Music...")
        yt = YTMusic('browser.json')
        liked_songs = yt.get_liked_songs(limit=None)
        
        date_str = datetime.now().strftime("%Y-%m-%d_%H%M")
        current_filename = os.path.join(base_path, f'my_likes_{date_str}.json')
        with open(current_filename, 'w', encoding='utf-8') as f:
            json.dump(liked_songs, f, indent=4)

        # --- COMPARE JSON BACKUPS ---
        files = sorted([f for f in os.listdir(base_path) if f.startswith('my_likes_') and f.endswith('.json')])
        if len(files) < 2:
            print("First backup created. Add a song to your likes and run again to download.")
            return

        with open(os.path.join(base_path, files[-1]), encoding='utf-8') as f_new, \
             open(os.path.join(base_path, files[-2]), encoding='utf-8') as f_old:
            new_tracks = {t['videoId']: t['title'] for t in json.load(f_new).get('tracks', [])}
            old_ids = {t['videoId'] for t in json.load(f_old).get('tracks', [])}

        added_ids = [vid for vid in new_tracks if vid not in old_ids]

        # --- PROCESS DOWNLOADS ---
        if not added_ids:
            print("No new songs found in your Likes list.")
        else:
            print(f"Found {len(added_ids)} new songs. Checking library...")
            for vid in added_ids:
                title = new_tracks[vid]
                
                exists, existing_name = is_already_downloaded(title, download_path)
                if exists:
                    print(f"  [Skip] '{title}' matches existing file: {existing_name}")
                    continue

                print(f"  [New] Downloading: {title}")
                file_template = os.path.join(download_path, f"%(title)s [{vid}].%(ext)s")
                
                subprocess.run([
                    'yt-dlp', '-x', '--audio-format', 'mp3',
                    '--embed-metadata', '--embed-thumbnail',
                    '--no-abort-on-error', '--continue',
                    '--sleep-interval', '2.25', '--sleep-requests', '1.25',
                    '--ffmpeg-location', ffmpeg_path,
                    '-o', file_template,
                    f"https://www.youtube.com/watch?v={vid}"
                ], check=True)

                # Fix the thumbnail immediately after download
                for f in os.listdir(download_path):
                    if vid in f and f.lower().endswith(".mp3"):
                        process_mp3_cover(os.path.join(download_path, f))

        print("\nProcess complete!")
        os.startfile(download_path)

    except Exception as e:
        print(f"\n--- SCRIPT FAILED ---\n{e}")
        input("\nPress Enter to close...")

if __name__ == "__main__":
    run_backup()
