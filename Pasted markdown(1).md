**Программын эх код**

**import cv2**

**import os**

**import csv**

**import time**

**import requests**

**from datetime import datetime**

**from ultralytics import YOLO**

**CONFIG = {**

**&#xA0;   # Camera**

**&#xA0;   # USB camera -> 0**

**&#xA0;   # IP camera -> "rtsp\://username:password\@camera-ip/..."**

**&#xA0;   "camera_source": 0,**

**&#xA0;   # AI models**

**&#xA0;   "sheep_model": "models/sheep_best.pt",**

**&#xA0;   "security_model": "models/wolf_dog_best.pt",**

**&#xA0;   "person_model": "yolo11n.pt",**



**&#xA0;   # Detection parameters**

**&#xA0;   "imgsz": 512,**

**&#xA0;   "sheep_conf": 0.35,**

**&#xA0;   "security_conf": 0.45,**

**&#xA0;   "person_conf": 0.50,**

**&#xA0;   # Counting line**

**&#xA0;   "line_position": 0.60,**



**&#xA0;   # Track settings**

**&#xA0;   "track_timeout": 5.0,**

**&#xA0;   "count_cooldown": 3.0,**



**&#xA0;   # Security**

**&#xA0;   "security_confirm_frames": 3,**

**&#xA0;   "alert_cooldown": 60,**



**&#xA0;   # Telegram**

**&#xA0;   "telegram_enabled": False,**

**&#xA0;   "telegram_bot_token": "",**

**&#xA0;   "telegram_chat_id": "",**



**&#xA0;   # Files**

**&#xA0;   "log_directory": "logs",**

**&#xA0;   "snapshot_directory": "snapshots",**

**&#xA0;   "count_log": "logs/counts.csv",**

**&#xA0;   "security_log": "logs/security.csv"**

**}**





**# ============================================================**

**# 2. CREATE DIRECTORIES**

**# ============================================================**



**os.makedirs(CONFIG["log_directory"], exist_ok=True)**

**os.makedirs(CONFIG["snapshot_directory"], exist_ok=True)**





**# ============================================================**

**# 3. SYSTEM MODE**

**# ============================================================**



**def get_system_mode():**

**&#xA0;   """**

**&#xA0;   06:00 - 08:00 : livestock leaving pen**

**&#xA0;   08:00 - 18:00 : idle**

**&#xA0;   18:00 - 22:00 : livestock entering pen**

**&#xA0;   22:00 - 06:00 : security**

**&#xA0;   """**



**&#xA0;   hour = datetime.now().hour**



**&#xA0;   if 6 <= hour < 8:**

**&#xA0;       return "COUNT_OUT"**



**&#xA0;   if 18 <= hour < 22:**

**&#xA0;       return "COUNT_IN"**



**&#xA0;   if hour >= 22 or hour < 6:**

**&#xA0;       return "SECURITY"**



**&#xA0;   return "IDLE"**





**# ============================================================**

**# 4. CSV LOGGING**

**# ============================================================**



**def initialize_csv(filename, header):**



**&#xA0;   if not os.path.exists(filename):**

**&#xA0;       with open(filename, "w", newline="", encoding="utf-8") as file:**

**&#xA0;           writer = csv.writer(file)**

**&#xA0;           writer.writerow(header)**





**initialize_csv(**

**&#xA0;   CONFIG["count_log"],**

**&#xA0;   ["datetime", "mode", "direction", "track_id",**

**&#xA0;    "count_in", "count_out"]**

**)**



**initialize_csv(**

**&#xA0;   CONFIG["security_log"],**

**&#xA0;   ["datetime", "object", "confidence", "snapshot"]**

**)**





**def log_count(direction, track_id, count_in, count_out):**



**&#xA0;   with open(**

**&#xA0;       CONFIG["count_log"],**

**&#xA0;       "a",**

**&#xA0;       newline="",**

**&#xA0;       encoding="utf-8"**

**&#xA0;   ) as file:**



**&#xA0;       writer = csv.writer(file)**



**&#xA0;       writer.writerow([**

**&#xA0;           datetime.now().strftime("%Y-%m-%d %H:%M:%S"),**

**&#xA0;           get_system_mode(),**

**&#xA0;           direction,**

**&#xA0;           track_id,**

**&#xA0;           count_in,**

**&#xA0;           count_out**

**&#xA0;       ])**





**def log_security(object_name, confidence, snapshot):**



**&#xA0;   with open(**

**&#xA0;       CONFIG["security_log"],**

**&#xA0;       "a",**

**&#xA0;       newline="",**

**&#xA0;       encoding="utf-8"**

**&#xA0;   ) as file:**



**&#xA0;       writer = csv.writer(file)**



**&#xA0;       writer.writerow([**

**&#xA0;           datetime.now().strftime("%Y-%m-%d %H:%M:%S"),**

**&#xA0;           object_name,**

**&#xA0;           round(confidence, 3),**

**&#xA0;           snapshot**

**&#xA0;       ])**





**# ============================================================**

**# 5. TELEGRAM ALERT**

**# ============================================================**



**def send_telegram_alert(message, image_path=None):**



**&#xA0;   if not CONFIG["telegram_enabled"]:**

**&#xA0;       return**



**&#xA0;   token = CONFIG["telegram_bot_token"]**

**&#xA0;   chat_id = CONFIG["telegram_chat_id"]**



**&#xA0;   if not token or not chat_id:**

**&#xA0;       return**



**&#xA0;   try:**



**&#xA0;       if image_path and os.path.exists(image_path):**



**&#xA0;           url = (**

**&#xA0;               f"https\://api.telegram.org/"**

**&#xA0;               f"bot{token}/sendPhoto"**

**&#xA0;           )**



**&#xA0;           with open(image_path, "rb") as photo:**



**&#xA0;               requests.post(**

**&#xA0;                   url,**

**&#xA0;                   data={**

**&#xA0;                       "chat_id": chat_id,**

**&#xA0;                       "caption": message**

**&#xA0;                   },**

**&#xA0;                   files={"photo": photo},**

**&#xA0;                   timeout=10**

**&#xA0;               )**



**&#xA0;       else:**



**&#xA0;           url = (**

**&#xA0;               f"https\://api.telegram.org/"**

**&#xA0;               f"bot{token}/sendMessage"**

**&#xA0;           )**



**&#xA0;           requests.post(**

**&#xA0;               url,**

**&#xA0;               data={**

**&#xA0;                   "chat_id": chat_id,**

**&#xA0;                   "text": message**

**&#xA0;               },**

**&#xA0;               timeout=10**

**&#xA0;           )**



**&#xA0;   except requests.RequestException as error:**



**&#xA0;       print("Telegram error:", error)**





**# ============================================================**

**# 6. SAVE SNAPSHOT**

**# ============================================================**



**def save_snapshot(frame, prefix):**



**&#xA0;   timestamp = datetime.now().strftime("%Y%m%d\_%H%M%S")**



**&#xA0;   filename = os.path.join(**

**&#xA0;       CONFIG["snapshot_directory"],**

**&#xA0;       f"{prefix}\_{timestamp}.jpg"**

**&#xA0;   )**



**&#xA0;   cv2.imwrite(filename, frame)**



**&#xA0;   return filename**





**# ============================================================**

**# 7. LIVESTOCK COUNTER**

**# ============================================================**



**class LivestockCounter:**



**&#xA0;   def \_\_init\_\_(self):**



**&#xA0;       print("Loading sheep model...")**



**&#xA0;       self.model = YOLO(CONFIG["sheep_model"])**



**&#xA0;       self.previous_positions = {}**

**&#xA0;       self.last_seen = {}**

**&#xA0;       self.counted_tracks = {}**



**&#xA0;       self.count_in = 0**

**&#xA0;       self.count_out = 0**



**&#xA0;   def reset(self):**



**&#xA0;       self.previous_positions.clear()**

**&#xA0;       self.last_seen.clear()**

**&#xA0;       self.counted_tracks.clear()**



**&#xA0;       self.count_in = 0**

**&#xA0;       self.count_out = 0**



**&#xA0;   def cleanup_tracks(self):**



**&#xA0;       current_time = time.time()**



**&#xA0;       expired = []**



**&#xA0;       for track_id, last_time in self.last_seen.items():**



**&#xA0;           if (**

**&#xA0;               current_time - last_time**

**&#xA0;               > CONFIG["track_timeout"]**

**&#xA0;           ):**

**&#xA0;               expired.append(track_id)**



**&#xA0;       for track_id in expired:**



**&#xA0;           self.previous_positions.pop(track_id, None)**

**&#xA0;           self.last_seen.pop(track_id, None)**



**&#xA0;   def process(self, frame, mode):**



**&#xA0;       height, width = frame.shape[:2]**



**&#xA0;       line_y = int(**

**&#xA0;           height \* CONFIG["line_position"]**

**&#xA0;       )**



**&#xA0;       # Custom sheep model:**

**&#xA0;       # class 0 = sheep**



**&#xA0;       results = self.model.track(**

**&#xA0;           frame,**

**&#xA0;           persist=True,**

**&#xA0;           classes=[0],**

**&#xA0;           imgsz=CONFIG["imgsz"],**

**&#xA0;           conf=CONFIG["sheep_conf"],**

**&#xA0;           tracker="bytetrack.yaml",**

**&#xA0;           verbose=False**

**&#xA0;       )**



**&#xA0;       visible_sheep = 0**



**&#xA0;       if results and results[0].boxes is not None:**



**&#xA0;           boxes = results[0].boxes**



**&#xA0;           visible_sheep = len(boxes)**



**&#xA0;           for box in boxes:**



**&#xA0;               if box.id is None:**

**&#xA0;                   continue**



**&#xA0;               track_id = int(box.id.item())**



**&#xA0;               x1, y1, x2, y2 = map(**

**&#xA0;                   int,**

**&#xA0;                   box.xyxy[0].tolist()**

**&#xA0;               )**



**&#xA0;               confidence = float(**

**&#xA0;                   box.conf[0].item()**

**&#xA0;               )**



**&#xA0;               center_y = int(**

**&#xA0;                   (y1 + y2) / 2**

**&#xA0;               )**



**&#xA0;               current_time = time.time()**



**&#xA0;               self.last_seen[track_id] = current_time**



**&#xA0;               # Draw bounding box**



**&#xA0;               cv2.rectangle(**

**&#xA0;                   frame,**

**&#xA0;                   (x1, y1),**

**&#xA0;                   (x2, y2),**

**&#xA0;                   (0, 255, 0),**

**&#xA0;                   2**

**&#xA0;               )**



**&#xA0;               cv2.putText(**

**&#xA0;                   frame,**

**&#xA0;                   f"Sheep ID:{track_id} {confidence:.2f}",**

**&#xA0;                   (x1, max(20, y1 - 10)),**

**&#xA0;                   cv2.FONT_HERSHEY_SIMPLEX,**

**&#xA0;                   0.5,**

**&#xA0;                   (0, 255, 0),**

**&#xA0;                   2**

**&#xA0;               )**



**&#xA0;               # ------------------------------------------------**

**&#xA0;               # LINE CROSSING**

**&#xA0;               # ------------------------------------------------**



**&#xA0;               if track_id in self.previous_positions:**



**&#xA0;                   previous_y = (**

**&#xA0;                       self.previous_positions[track_id]**

**&#xA0;                   )**



**&#xA0;                   last_count_time = (**

**&#xA0;                       self.counted_tracks.get(**

**&#xA0;                           track_id,**

**&#xA0;                           0**

**&#xA0;                       )**

**&#xA0;                   )**



**&#xA0;                   can_count = (**

**&#xA0;                       current_time - last_count_time**

**&#xA0;                       > CONFIG["count_cooldown"]**

**&#xA0;                   )**



**&#xA0;                   # Animal moving downward**

**&#xA0;                   if (**

**&#xA0;                       previous_y < line_y <= center_y**

**&#xA0;                       and can_count**

**&#xA0;                   ):**



**&#xA0;                       if mode == "COUNT_IN":**



**&#xA0;                           self.count_in += 1**



**&#xA0;                           self.counted_tracks[**

**&#xA0;                               track_id**

**&#xA0;                           ] = current_time**



**&#xA0;                           log_count(**

**&#xA0;                               "IN",**

**&#xA0;                               track_id,**

**&#xA0;                               self.count_in,**

**&#xA0;                               self.count_out**

**&#xA0;                           )**



**&#xA0;                   # Animal moving upward**

**&#xA0;                   elif (**

**&#xA0;                       previous_y > line_y >= center_y**

**&#xA0;                       and can_count**

**&#xA0;                   ):**



**&#xA0;                       if mode == "COUNT_OUT":**



**&#xA0;                           self.count_out += 1**



**&#xA0;                           self.counted_tracks[**

**&#xA0;                               track_id**

**&#xA0;                           ] = current_time**



**&#xA0;                           log_count(**

**&#xA0;                               "OUT",**

**&#xA0;                               track_id,**

**&#xA0;                               self.count_in,**

**&#xA0;                               self.count_out**

**&#xA0;                           )**



**&#xA0;               self.previous_positions[**

**&#xA0;                   track_id**

**&#xA0;               ] = center_y**



**&#xA0;       # Counting line**



**&#xA0;       cv2.line(**

**&#xA0;           frame,**

**&#xA0;           (0, line_y),**

**&#xA0;           (width, line_y),**

**&#xA0;           (255, 0, 0),**

**&#xA0;           2**

**&#xA0;       )**



**&#xA0;       self.cleanup_tracks()**



**&#xA0;       return (**

**&#xA0;           frame,**

**&#xA0;           visible_sheep,**

**&#xA0;           self.count_in,**

**&#xA0;           self.count_out**

**&#xA0;       )**





**# ============================================================**

**# 8. SECURITY SYSTEM**

**# ============================================================**



**class SecuritySystem:**



**&#xA0;   def \_\_init\_\_(self):**



**&#xA0;       print("Loading wolf/dog model...")**



**&#xA0;       self.security_model = YOLO(**

**&#xA0;           CONFIG["security_model"]**

**&#xA0;       )**



**&#xA0;       print("Loading person model...")**



**&#xA0;       self.person_model = YOLO(**

**&#xA0;           CONFIG["person_model"]**

**&#xA0;       )**



**&#xA0;       self.confirmation = {**

**&#xA0;           "wolf": 0,**

**&#xA0;           "person": 0**

**&#xA0;       }**



**&#xA0;       self.last_alert = {**

**&#xA0;           "wolf": 0,**

**&#xA0;           "person": 0**

**&#xA0;       }**



**&#xA0;   def alert_allowed(self, object_name):**



**&#xA0;       current_time = time.time()**



**&#xA0;       return (**

**&#xA0;           current_time**

**&#xA0;           - self.last_alert.get(object_name, 0)**

**&#xA0;           > CONFIG["alert_cooldown"]**

**&#xA0;       )**



**&#xA0;   def trigger_alert(**

**&#xA0;       self,**

**&#xA0;       frame,**

**&#xA0;       object_name,**

**&#xA0;       confidence**

**&#xA0;   ):**



**&#xA0;       if not self.alert_allowed(object_name):**

**&#xA0;           return**



**&#xA0;       snapshot = save_snapshot(**

**&#xA0;           frame,**

**&#xA0;           object_name**

**&#xA0;       )**



**&#xA0;       log_security(**

**&#xA0;           object_name,**

**&#xA0;           confidence,**

**&#xA0;           snapshot**

**&#xA0;       )**



**&#xA0;       message = (**

**&#xA0;           f"WARNING!\n"**

**&#xA0;           f"Detected: {object_name.upper()}\n"**

**&#xA0;           f"Confidence: {confidence:.2f}\n"**

**&#xA0;           f"Time: "**

**&#xA0;           f"{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}"**

**&#xA0;       )**



**&#xA0;       send_telegram_alert(**

**&#xA0;           message,**

**&#xA0;           snapshot**

**&#xA0;       )**



**&#xA0;       self.last_alert[**

**&#xA0;           object_name**

**&#xA0;       ] = time.time()**



**&#xA0;   def process_wolf_dog(self, frame):**



**&#xA0;       # Custom model:**

**&#xA0;       # class 0 = dog**

**&#xA0;       # class 1 = wolf**



**&#xA0;       results = self.security_model(**

**&#xA0;           frame,**

**&#xA0;           imgsz=CONFIG["imgsz"],**

**&#xA0;           conf=CONFIG["security_conf"],**

**&#xA0;           verbose=False**

**&#xA0;       )**



**&#xA0;       wolf_detected = False**

**&#xA0;       max_wolf_conf = 0.0**



**&#xA0;       if results:**



**&#xA0;           for box in results[0].boxes:**



**&#xA0;               class_id = int(**

**&#xA0;                   box.cls[0].item()**

**&#xA0;               )**



**&#xA0;               confidence = float(**

**&#xA0;                   box.conf[0].item()**

**&#xA0;               )**



**&#xA0;               x1, y1, x2, y2 = map(**

**&#xA0;                   int,**

**&#xA0;                   box.xyxy[0].tolist()**

**&#xA0;               )**



**&#xA0;               if class_id == 0:**



**&#xA0;                   name = "Dog"**



**&#xA0;               elif class_id == 1:**



**&#xA0;                   name = "Wolf"**



**&#xA0;                   wolf_detected = True**



**&#xA0;                   max_wolf_conf = max(**

**&#xA0;                       max_wolf_conf,**

**&#xA0;                       confidence**

**&#xA0;                   )**



**&#xA0;               else:**

**&#xA0;                   continue**



**&#xA0;               cv2.rectangle(**

**&#xA0;                   frame,**

**&#xA0;                   (x1, y1),**

**&#xA0;                   (x2, y2),**

**&#xA0;                   (0, 255, 255),**

**&#xA0;                   2**

**&#xA0;               )**



**&#xA0;               cv2.putText(**

**&#xA0;                   frame,**

**&#xA0;                   f"{name} {confidence:.2f}",**

**&#xA0;                   (x1, max(20, y1 - 10)),**

**&#xA0;                   cv2.FONT_HERSHEY_SIMPLEX,**

**&#xA0;                   0.6,**

**&#xA0;                   (0, 255, 255),**

**&#xA0;                   2**

**&#xA0;               )**



**&#xA0;       # Multi-frame confirmation**



**&#xA0;       if wolf_detected:**



**&#xA0;           self.confirmation["wolf"] += 1**



**&#xA0;       else:**



**&#xA0;           self.confirmation["wolf"] = 0**



**&#xA0;       if (**

**&#xA0;           self.confirmation["wolf"]**

**&#xA0;           >= CONFIG["security_confirm_frames"]**

**&#xA0;       ):**



**&#xA0;           self.trigger_alert(**

**&#xA0;               frame,**

**&#xA0;               "wolf",**

**&#xA0;               max_wolf_conf**

**&#xA0;           )**



**&#xA0;           self.confirmation["wolf"] = 0**



**&#xA0;       return frame**



**&#xA0;   def process_person(self, frame):**



**&#xA0;       # COCO class 0 = person**



**&#xA0;       results = self.person_model(**

**&#xA0;           frame,**

**&#xA0;           imgsz=CONFIG["imgsz"],**

**&#xA0;           conf=CONFIG["person_conf"],**

**&#xA0;           classes=[0],**

**&#xA0;           verbose=False**

**&#xA0;       )**



**&#xA0;       person_detected = False**

**&#xA0;       max_person_conf = 0.0**



**&#xA0;       if results:**



**&#xA0;           for box in results[0].boxes:**



**&#xA0;               confidence = float(**

**&#xA0;                   box.conf[0].item()**

**&#xA0;               )**



**&#xA0;               x1, y1, x2, y2 = map(**

**&#xA0;                   int,**

**&#xA0;                   box.xyxy[0].tolist()**

**&#xA0;               )**



**&#xA0;               person_detected = True**



**&#xA0;               max_person_conf = max(**

**&#xA0;                   max_person_conf,**

**&#xA0;                   confidence**

**&#xA0;               )**



**&#xA0;               cv2.rectangle(**

**&#xA0;                   frame,**

**&#xA0;                   (x1, y1),**

**&#xA0;                   (x2, y2),**

**&#xA0;                   (0, 0, 255),**

**&#xA0;                   2**

**&#xA0;               )**



**&#xA0;               cv2.putText(**

**&#xA0;                   frame,**

**&#xA0;                   f"Person {confidence:.2f}",**

**&#xA0;                   (x1, max(20, y1 - 10)),**

**&#xA0;                   cv2.FONT_HERSHEY_SIMPLEX,**

**&#xA0;                   0.6,**

**&#xA0;                   (0, 0, 255),**

**&#xA0;                   2**

**&#xA0;               )**



**&#xA0;       if person_detected:**



**&#xA0;           self.confirmation["person"] += 1**



**&#xA0;       else:**



**&#xA0;           self.confirmation["person"] = 0**



**&#xA0;       if (**

**&#xA0;           self.confirmation["person"]**

**&#xA0;           >= CONFIG["security_confirm_frames"]**

**&#xA0;       ):**



**&#xA0;           self.trigger_alert(**

**&#xA0;               frame,**

**&#xA0;               "person",**

**&#xA0;               max_person_conf**

**&#xA0;           )**



**&#xA0;           self.confirmation["person"] = 0**



**&#xA0;       return frame**



**&#xA0;   def process(self, frame):**



**&#xA0;       frame = self.process_wolf_dog(frame)**

**&#xA0;       frame = self.process_person(frame)**



**&#xA0;       return frame**





**# ============================================================**

**# 9. USER INTERFACE**

**# ============================================================**



**def draw_dashboard(**

**&#xA0;   frame,**

**&#xA0;   mode,**

**&#xA0;   visible_sheep,**

**&#xA0;   count_in,**

**&#xA0;   count_out,**

**&#xA0;   fps**

**):**



**&#xA0;   difference = count_in - count_out**



**&#xA0;   cv2.rectangle(**

**&#xA0;       frame,**

**&#xA0;       (10, 10),**

**&#xA0;       (350, 180),**

**&#xA0;       (0, 0, 0),**

**&#xA0;       -1**

**&#xA0;   )**



**&#xA0;   information = [**

**&#xA0;       f"MODE: {mode}",**

**&#xA0;       f"LIVE DETECTED: {visible_sheep}",**

**&#xA0;       f"EVENING IN: {count_in}",**

**&#xA0;       f"MORNING OUT: {count_out}",**

**&#xA0;       f"DIFFERENCE: {difference}",**

**&#xA0;       f"FPS: {fps:.1f}"**

**&#xA0;   ]**



**&#xA0;   y = 35**



**&#xA0;   for text in information:**



**&#xA0;       cv2.putText(**

**&#xA0;           frame,**

**&#xA0;           text,**

**&#xA0;           (20, y),**

**&#xA0;           cv2.FONT_HERSHEY_SIMPLEX,**

**&#xA0;           0.6,**

**&#xA0;           (255, 255, 255),**

**&#xA0;           2**

**&#xA0;       )**



**&#xA0;       y += 25**



**&#xA0;   return frame**





**# ============================================================**

**# 10. MAIN PROGRAM**

**# ============================================================**



**def main():**



**&#xA0;   print("=" \* 50)**

**&#xA0;   print("AI LIVESTOCK MONITORING SYSTEM")**

**&#xA0;   print("=" \* 50)**



**&#xA0;   counter = LivestockCounter()**

**&#xA0;   security = SecuritySystem()**



**&#xA0;   camera = cv2.VideoCapture(**

**&#xA0;       CONFIG["camera_source"]**

**&#xA0;   )**



**&#xA0;   if not camera.isOpened():**



**&#xA0;       print("Camera could not be opened.")**

**&#xA0;       return**



**&#xA0;   previous_mode = None**



**&#xA0;   previous_time = time.time()**



**&#xA0;   visible_sheep = 0**

**&#xA0;   count_in = 0**

**&#xA0;   count_out = 0**



**&#xA0;   while True:**



**&#xA0;       success, frame = camera.read()**



**&#xA0;       if not success:**



**&#xA0;           print("Failed to read camera frame.")**

**&#xA0;           time.sleep(1)**

**&#xA0;           continue**



**&#xA0;       current_mode = get_system_mode()**



**&#xA0;       # ----------------------------------------------------**

**&#xA0;       # Detect mode change**

**&#xA0;       # ----------------------------------------------------**



**&#xA0;       if current_mode != previous_mode:**



**&#xA0;           print(**

**&#xA0;               f"System mode changed: "**

**&#xA0;               f"{previous_mode} -> {current_mode}"**

**&#xA0;           )**



**&#xA0;           previous_mode = current_mode**



**&#xA0;       # ----------------------------------------------------**

**&#xA0;       # COUNTING MODE**

**&#xA0;       # ----------------------------------------------------**



**&#xA0;       if current_mode in [**

**&#xA0;           "COUNT_IN",**

**&#xA0;           "COUNT_OUT"**

**&#xA0;       ]:**



**&#xA0;           (**

**&#xA0;               frame,**

**&#xA0;               visible_sheep,**

**&#xA0;               count_in,**

**&#xA0;               count_out**

**&#xA0;           ) = counter.process(**

**&#xA0;               frame,**

**&#xA0;               current_mode**

**&#xA0;           )**



**&#xA0;       # ----------------------------------------------------**

**&#xA0;       # SECURITY MODE**

**&#xA0;       # ----------------------------------------------------**



**&#xA0;       elif current_mode == "SECURITY":**



**&#xA0;           visible_sheep = 0**



**&#xA0;           frame = security.process(frame)**



**&#xA0;       # ----------------------------------------------------**

**&#xA0;       # IDLE MODE**

**&#xA0;       # ----------------------------------------------------**



**&#xA0;       else:**



**&#xA0;           visible_sheep = 0**



**&#xA0;           cv2.putText(**

**&#xA0;               frame,**

**&#xA0;               "SYSTEM IDLE",**

**&#xA0;               (20, 220),**

**&#xA0;               cv2.FONT_HERSHEY_SIMPLEX,**

**&#xA0;               0.8,**

**&#xA0;               (0, 255, 255),**

**&#xA0;               2**

**&#xA0;           )**



**&#xA0;       # ----------------------------------------------------**

**&#xA0;       # FPS**

**&#xA0;       # ----------------------------------------------------**



**&#xA0;       current_time = time.time()**



**&#xA0;       elapsed = current_time - previous_time**



**&#xA0;       if elapsed > 0:**

**&#xA0;           fps = 1.0 / elapsed**

**&#xA0;       else:**

**&#xA0;           fps = 0.0**



**&#xA0;       previous_time = current_time**





**&#xA0;       frame = draw_dashboard(**

**&#xA0;           frame,**

**&#xA0;           current_mode,**

**&#xA0;           visible_sheep,**

**&#xA0;           count_in,**

**&#xA0;           count_out,**

**&#xA0;           fps**

**&#xA0;       )**



**&#xA0;       cv2.imshow(**

**&#xA0;           "AI Livestock Monitoring System",**

**&#xA0;           frame**

**&#xA0;       )**



**&#xA0;       key = cv2.waitKey(1) & 0xFF**



**&#xA0;       # Q = Quit**

**&#xA0;       if key == ord("q"):**

**&#xA0;           break**



**&#xA0;       # R = Reset counts**

**&#xA0;       if key == ord("r"):**



**&#xA0;           counter.reset()**



**&#xA0;           count_in = 0**

**&#xA0;           count_out = 0**



**&#xA0;           print("Counter reset.")**



**&#xA0;   camera.release()**



**&#xA0;   cv2.destroyAllWindows()**



**&#xA0;   print("System stopped.")**



**if \_\_name\_\_ == "\_\_main\_\_":**

**&#xA0;   main()**