import tkinter as tk
from tkinter import filedialog, messagebox, ttk, simpledialog
import random
from datetime import datetime, timedelta
import webbrowser
import os
import re
import json
import time
import math

# ========== CẤU HÌNH ==========
POMODORO_WORK = 25 * 60
POMODORO_BREAK = 5 * 60
THEME = {
    "bg": "#f5f5f5", "fg": "#333333", "accent": "#4a6fa5",
    "secondary": "#6c757d", "success": "#28a745", "danger": "#dc3545",
    "entry_bg": "#ffffff", "font": ("Arial Unicode MS", 10),
    "title_font": ("Arial Unicode MS", 16, "bold"), "hanzi_font": ("SimSun", 60),
    "hint_bg": "#fff8e1", "hint_fg": "#ff6f00"
}

# ========== BIẾN TOÀN CỤC ==========
words = []
score = 0
total = 0
history = []
current_word = None
pen_size = 5
grid_size = 300
browser_opened = False
input_mode = "hanzi"
LANGUAGE = "vi"
spaced_repetition_data = {}
pomodoro_start_time = None
is_pomodoro_break = False
pomodoro_timer_id = None
pen_colors = ["#000000", "#FF0000", "#00AA00", "#0000FF", "#FFA500", "#800080"]
current_pen_color = "#000000"
last_vocab_file = ""
no_tone_mode = False
first_import = True

# ========== TIỆN ÍCH ==========
def is_android():
    return 'ANDROID_ARGUMENT' in os.environ or 'ANDROID_ROOT' in os.environ

def normalize_vietnamese(text):
    text = text.lower()
    text = re.sub(r'[dđ]', 'd', text)
    text = re.sub(r'[àáảãạăắằẳẵặâấầẩẫậ]', 'a', text)
    text = re.sub(r'[èéẻẽẹêếềểễệ]', 'e', text)
    text = re.sub(r'[ìíỉĩị]', 'i', text)
    text = re.sub(r'[òóỏõọôốồổỗộơớờởỡợ]', 'o', text)
    text = re.sub(r'[ùúủũụưứừửữự]', 'u', text)
    text = re.sub(r'[ỳýỷỹỵ]', 'y', text)
    return text

def remove_tone(text):
    tone_map = {
        'à': 'a', 'á': 'a', 'ả': 'a', 'ã': 'a', 'ạ': 'a',
        'ă': 'a', 'ằ': 'a', 'ắ': 'a', 'ẳ': 'a', 'ẵ': 'a', 'ặ': 'a',
        'â': 'a', 'ầ': 'a', 'ấ': 'a', 'ẩ': 'a', 'ẫ': 'a', 'ậ': 'a',
        'è': 'e', 'é': 'e', 'ẻ': 'e', 'ẽ': 'e', 'ẹ': 'e',
        'ê': 'e', 'ề': 'e', 'ế': 'e', 'ể': 'e', 'ễ': 'e', 'ệ': 'e',
        'ì': 'i', 'í': 'i', 'ỉ': 'i', 'ĩ': 'i', 'ị': 'i',
        'ò': 'o', 'ó': 'o', 'ỏ': 'o', 'õ': 'o', 'ọ': 'o',
        'ô': 'o', 'ồ': 'o', 'ố': 'o', 'ổ': 'o', 'ỗ': 'o', 'ộ': 'o',
        'ơ': 'o', 'ờ': 'o', 'ớ': 'o', 'ở': 'o', 'ỡ': 'o', 'ợ': 'o',
        'ù': 'u', 'ú': 'u', 'ủ': 'u', 'ũ': 'u', 'ụ': 'u',
        'ư': 'u', 'ừ': 'u', 'ứ': 'u', 'ử': 'u', 'ữ': 'u', 'ự': 'u',
        'ỳ': 'y', 'ý': 'y', 'ỷ': 'y', 'ỹ': 'y', 'ỵ': 'y',
        'đ': 'd'
    }
    return ''.join(tone_map.get(c, c) for c in text.lower())

# ========== HƯỚNG DẪN ==========
def show_help():
    help_window = tk.Toplevel(root)
    help_window.title("Hướng dẫn" if LANGUAGE == "vi" else "Help")
    help_window.geometry("500x400")
    help_window.configure(bg=THEME["bg"])
    
    content = """
CÁCH SỬ DỤNG CHƯƠNG TRÌNH:

1. Nhập từ vựng:
   - Chế độ Hán tự: Nhập nghĩa tiếng Việt tương ứng
   - Chế độ nghĩa: Nhập chữ Hán tương ứng
   - Chấp nhận nhiều nghĩa (vd: 厉害 = lợi hại/nặng/dữ dội)

2. Kiểu nhập tiếng Việt:
   - Telex: as -> á, af -> à, aw -> ă, dd -> đ
   - Không dấu: bật/tắt bằng nút "Không dấu"
   - Hỗ trợ cả chữ hoa/thường

3. So sánh kết quả:
   - Bỏ qua từ không quan trọng (tiếng, nước, chữ...)
   - So khớp các từ khóa chính
""" if LANGUAGE == "vi" else """
HOW TO USE:

1. Vocabulary input:
   - Hanzi mode: Enter corresponding Vietnamese meaning
   - Meaning mode: Enter corresponding Chinese character
   - Accepts multiple meanings (eg: 厉害 = lợi hại/nặng/dữ dội)

2. Vietnamese input:
   - Telex: as -> á, af -> à, aw -> ă, dd -> đ
   - No-tone mode: toggle with "No tone" button
   - Supports both uppercase/lowercase

3. Answer comparison:
   - Ignores unimportant words (tiếng, nước, chữ...)
   - Matches key words
"""
    
    text = tk.Text(help_window, wrap=tk.WORD, bg=THEME["bg"], fg=THEME["fg"],
                  font=THEME["font"], padx=10, pady=10)
    text.insert(tk.END, content)
    text.config(state="disabled")
    text.pack(expand=True, fill="both")
    
    tk.Button(help_window, 
             text="Đóng" if LANGUAGE == "vi" else "Close", 
             command=help_window.destroy,
             bg=THEME["accent"], fg="white", 
             font=THEME["font"]).pack(pady=10)

# ========== POMODORO ==========
def start_pomodoro():
    global pomodoro_start_time, is_pomodoro_break, pomodoro_timer_id
    if pomodoro_timer_id:
        root.after_cancel(pomodoro_timer_id)
    pomodoro_start_time = time.time()
    is_pomodoro_break = False
    pomodoro_label.config(text="⏳ Học 25:00" if LANGUAGE == "vi" else "⏳ Study 25:00")
    update_pomodoro_timer()
    messagebox.showinfo("Pomodoro", "Phiên học 25 phút bắt đầu!" if LANGUAGE == "vi" else "25-minute study session starts!")

def update_pomodoro_timer():
    global is_pomodoro_break, pomodoro_timer_id, pomodoro_start_time
    
    if pomodoro_start_time is None:
        return
    
    current_time = time.time()
    elapsed = current_time - pomodoro_start_time
    remaining = (POMODORO_WORK if not is_pomodoro_break else POMODORO_BREAK) - elapsed
    
    if remaining <= 0:
        if not is_pomodoro_break:
            is_pomodoro_break = True
            pomodoro_start_time = current_time
            messagebox.showwarning("Pomodoro", "Dừng lại nghỉ ngơi, 5 phút sau quay lại nhé!" if LANGUAGE == "vi" else "Take a break! Come back in 5 minutes")
            root.withdraw()
        else:
            is_pomodoro_break = False
            pomodoro_start_time = current_time
            root.deiconify()
            messagebox.showinfo("Pomodoro", "Bắt đầu phiên học mới!" if LANGUAGE == "vi" else "Start new study session!")
    
    mins, secs = divmod(int(remaining), 60)
    timer_text = f"{mins:02d}:{secs:02d}"
    status = "⏳ " + ("Nghỉ" if is_pomodoro_break else "Học") if LANGUAGE == "vi" else "⏳ " + ("Break" if is_pomodoro_break else "Study")
    pomodoro_label.config(text=f"{status} {timer_text}")
    
    pomodoro_timer_id = root.after(1000, update_pomodoro_timer)

# ========== NHẬP LIỆU ==========
def create_vietnamese_entry():
    entry = tk.Entry(
        main_frame, 
        font=THEME["font"],
        justify="center", 
        bg=THEME["entry_bg"], 
        fg=THEME["fg"], 
        relief="flat",
        highlightthickness=1, 
        highlightcolor=THEME["accent"]
    )
    entry.pack(fill="x", ipady=12 if is_android() else 8, pady=(0, 15))
    setup_telex_input(entry)
    return entry

def setup_telex_input(entry_widget):
    def handle_key(event):
        if entry_widget.focus_get() == entry_widget:
            if event.char in ('s', 'f', 'r', 'x', 'j', 'w', 'd', 'a', 'e', 'o', 'u'):
                pos = entry_widget.index(tk.INSERT)
                if pos > 0:
                    prev_char = entry_widget.get()[pos-1]
                    new_char = apply_telex_rule(prev_char, event.char)
                    if new_char != prev_char:
                        entry_widget.delete(pos-1, pos)
                        entry_widget.insert(pos-1, new_char)
                        return "break"
        return None
    entry_widget.bind('<Key>', handle_key)

def apply_telex_rule(char, telex_key):
    telex_rules = {
        'a': {'w': 'ă', 's': 'á', 'f': 'à', 'r': 'ả', 'x': 'ã', 'j': 'ạ'},
        'd': {'d': 'đ'},
        'e': {'w': 'ê', 's': 'é', 'f': 'è', 'r': 'ẻ', 'x': 'ẽ', 'j': 'ẹ'},
        'o': {'w': 'ơ', 's': 'ó', 'f': 'ò', 'r': 'ỏ', 'x': 'õ', 'j': 'ọ'},
        'u': {'w': 'ư', 's': 'ú', 'f': 'ù', 'r': 'ủ', 'x': 'ũ', 'j': 'ụ'},
        'A': {'w': 'Ă', 's': 'Á', 'f': 'À', 'r': 'Ả', 'x': 'Ã', 'j': 'Ạ'},
        'D': {'d': 'Đ'},
        'E': {'w': 'Ê', 's': 'É', 'f': 'È', 'r': 'Ẻ', 'x': 'Ẽ', 'j': 'Ẹ'},
        'O': {'w': 'Ơ', 's': 'Ó', 'f': 'Ò', 'r': 'Ỏ', 'x': 'Õ', 'j': 'Ọ'},
        'U': {'w': 'Ư', 's': 'Ú', 'f': 'Ù', 'r': 'Ủ', 'x': 'Ũ', 'j': 'Ụ'}
    }
    lower_char = char.lower()
    if lower_char in telex_rules and telex_key in telex_rules[lower_char]:
        if char.isupper():
            return telex_rules[lower_char][telex_key].upper()
        return telex_rules[lower_char][telex_key]
    return char

# ========== CHỨC NĂNG CHÍNH ==========
def load_vocab_file(file_path):
    global words, score, total, history, last_vocab_file, first_import
    
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            raw_lines = [line.strip() for line in f if line.strip()]
            
        words = []
        for line in raw_lines:
            parts = re.split(r'\t+| {2,}', line.strip())
            if len(parts) >= 3:
                hanzi = parts[0]
                pinyin = parts[1]
                meanings = [m.strip() for m in re.split(r'[,/]', ' '.join(parts[2:])) if m.strip()]
                words.append([hanzi, pinyin, meanings])
            elif len(parts) == 2:
                words.append([parts[0], parts[1], [""]])
                
        last_vocab_file = file_path
        score = total = 0
        history = []
        first_import = False
        load_new_word()
        update_file_label()
        return True
        
    except Exception as e:
        messagebox.showerror("Lỗi" if LANGUAGE == "vi" else "Error", 
                           f"Không thể đọc file:\n{str(e)}" if LANGUAGE == "vi" 
                           else f"Can't read file:\n{str(e)}")
        return False

def import_file():
    global last_vocab_file, first_import
    
    if first_import:
        password = simpledialog.askstring("Password", "Nhập password để tiếp tục:", show='*')
        if password != "991016":
            messagebox.showerror("Lỗi", "Password không đúng!")
            return
    
    try:
        initial_dir = os.path.dirname(last_vocab_file) if last_vocab_file else (
            os.path.expanduser("~") if not is_android() else "/storage/emulated/0/Documents"
        )
        
        file_path = filedialog.askopenfilename(
            title="Chọn file từ vựng HSK" if LANGUAGE == "vi" else "Select HSK vocabulary file",
            filetypes=[("Text files", "*.txt"), ("All files", "*.*")],
            initialdir=initial_dir
        )
        
        if file_path:
            if load_vocab_file(file_path):
                success_msg = (f"Đã import {len(words)} từ vựng!\n"
                              f"Ví dụ: {words[0][0]} ({words[0][1]}) = {', '.join(words[0][2])}")
                
                messagebox.showinfo("Thành công" if LANGUAGE == "vi" else "Success", 
                                  success_msg if LANGUAGE == "vi" 
                                  else f"Imported {len(words)} words!\nExample: {words[0][0]} ({words[0][1]}) = {', '.join(words[0][2])}")
                start_pomodoro()
    except Exception as e:
        messagebox.showerror("Lỗi", f"Không thể mở file:\n{str(e)}")
        return False

def update_file_label():
    if last_vocab_file:
        file_label.config(text=f"File: {os.path.basename(last_vocab_file)}")
    else:
        file_label.config(text="Chưa chọn file từ vựng" if LANGUAGE == "vi" else "No vocabulary file selected")

def load_new_word():
    global current_word, browser_opened
    
    if is_pomodoro_break:
        return
        
    if not words:
        hanzi_label.config(text="(Chưa có từ vựng)" if LANGUAGE == "vi" else "(No vocabulary)")
        return
        
    current_word = get_next_word()
    browser_opened = False
    update_display()
    entry.delete(0, tk.END)
    result_label.config(text="")
    hanzii_btn.config(state=tk.NORMAL)
    entry.focus()

def update_display():
    if not current_word:
        return
        
    if input_mode == "hanzi":
        hanzi_label.config(text=current_word[0], font=THEME["hanzi_font"], fg=THEME["fg"])
        mode_label.config(text=f"{'Mode' if LANGUAGE == 'en' else 'Chế độ'}: {'Enter Meaning' if LANGUAGE == 'en' else 'Nhập nghĩa'}")
    else:
        hanzi_label.config(text=current_word[2][0], font=THEME["font"], fg=THEME["fg"])
        mode_label.config(text=f"{'Mode' if LANGUAGE == 'en' else 'Chế độ'}: {'Enter Hanzi' if LANGUAGE == 'en' else 'Nhập chữ Hán'}")

def toggle_input_mode():
    global input_mode
    input_mode = "meaning" if input_mode == "hanzi" else "hanzi"
    update_display()
    entry.focus()

def toggle_no_tone_mode():
    global no_tone_mode
    no_tone_mode = not no_tone_mode
    if LANGUAGE == "vi":
        no_tone_btn.config(text="Tắt không dấu" if no_tone_mode else "Bật không dấu")
    else:
        no_tone_btn.config(text="Disable no tone" if no_tone_mode else "Enable no tone")
    entry.focus()

def skip_word():
    global history
    if current_word:
        history.append({
            'time': datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            'word': current_word,
            'input': "(Bỏ qua)" if LANGUAGE == "vi" else "(Skipped)",
            'result': "Bỏ qua" if LANGUAGE == "vi" else "Skipped",
            'mode': input_mode,
            'next_review': spaced_repetition_data.get(current_word[0], {}).get("next_review", datetime.now())
        })
    load_new_word()

def check_answer(event=None):
    global score, total, history
    
    if is_pomodoro_break:
        return
        
    user_input = entry.get().strip()
    if not user_input: return
        
    total += 1
    is_correct = False
    
    if input_mode == "hanzi":
        user_text = remove_tone(normalize_vietnamese(user_input)) if no_tone_mode else normalize_vietnamese(user_input)
        
        ignore_words = ['tiếng', 'nước', 'chữ', 'của', 'trong', 'là', 'cái', 'bài', 'việc']
        user_words = [w for w in user_text.split() if w not in ignore_words]
        
        for meaning in current_word[2]:
            correct_text = remove_tone(normalize_vietnamese(meaning)) if no_tone_mode else normalize_vietnamese(meaning)
            correct_words = [w for w in correct_text.split() if w not in ignore_words]
            
            if set(user_words) == set(correct_words):
                is_correct = True
                break
                
        correct_answer = ", ".join(current_word[2])
    else:
        is_correct = (user_input.lower() == current_word[0].lower())
        correct_answer = current_word[0]
    
    performance = 4 if is_correct else 0
    update_spaced_repetition(current_word[0], performance)
    
    if is_correct:
        score += 1
        result_label.config(text="✅ Đúng rồi giỏi cáa!" if LANGUAGE == "vi" else "✅ Correct!", fg=THEME["success"])
        root.after(800, load_new_word)
    else:
        result_label.config(text=f"❌ {'Sai rồi hehe!' if LANGUAGE == 'vi' else 'Wrong'}! {'Đáp án' if LANGUAGE == 'vi' else 'Answer'}: {correct_answer}", 
                          fg=THEME["danger"])
        show_result_window(user_input, correct_answer)
    
    score_text = f"{'Điểm' if LANGUAGE == 'vi' else 'Score'}: {score}/{total}"
    if total > 0:
        score_text += f" ({'Tỉ lệ' if LANGUAGE == 'vi' else 'Rate'}: {score/total*100:.1f}%)"
    score_label.config(text=score_text)
    
    history.append({
        'time': datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        'word': current_word,
        'input': user_input,
        'result': "Đúng" if is_correct else "Sai" if LANGUAGE == "vi" else "Correct" if is_correct else "Wrong",
        'mode': input_mode,
        'next_review': spaced_repetition_data.get(current_word[0], {}).get("next_review", datetime.now())
    })
    
    save_progress()

def show_result_window(user_input, correct_answer):
    result_window = tk.Toplevel(root)
    result_window.title("Kết quả" if LANGUAGE == "vi" else "Result")
    result_window.geometry("500x450")
    result_window.configure(bg=THEME["bg"])
    result_window.resizable(False, False)
    result_window.grab_set()
    
    main_frame = tk.Frame(result_window, bg=THEME["bg"])
    main_frame.pack(fill="both", expand=True, padx=20, pady=20)
    
    vocab_frame = tk.Frame(main_frame, bg=THEME["bg"])
    vocab_frame.pack(fill="x", pady=(0, 15))
    
    tk.Label(vocab_frame, text=current_word[0], font=("SimSun", 36), 
            bg=THEME["bg"], fg=THEME["fg"]).pack()
    tk.Label(vocab_frame, text=current_word[1], font=("Arial Unicode MS", 14), 
            bg=THEME["bg"], fg=THEME["accent"]).pack()
    tk.Label(vocab_frame, text=", ".join(current_word[2]), font=("Arial Unicode MS", 12), 
            bg=THEME["bg"], fg=THEME["fg"]).pack(pady=(5, 0))
    
    input_frame = tk.Frame(main_frame, bg=THEME["bg"])
    input_frame.pack(fill="x", pady=(10, 5))
    
    mode_text = "nghĩa tiếng Việt" if input_mode == "hanzi" else "chữ Hán" if LANGUAGE == "vi" else "Hanzi"
    tk.Label(input_frame, text=f"{'Bạn nhập' if LANGUAGE == 'vi' else 'Your input'} ({mode_text}):", 
            font=THEME["font"], bg=THEME["bg"], fg=THEME["fg"]).pack(anchor="w")
    
    text_widget = tk.Text(input_frame, height=2, 
                         font=("Arial Unicode MS", 14) if input_mode == "hanzi" else ("SimSun", 24), 
                         bg=THEME["entry_bg"], fg=THEME["fg"], wrap=tk.WORD,
                         padx=10, pady=5, borderwidth=0, highlightthickness=1,
                         highlightcolor=THEME["accent"])
    text_widget.pack(fill="x")
    text_widget.insert("end", user_input)
    
    if input_mode == "hanzi":
        user_words = normalize_vietnamese(user_input).split()
        correct_words = normalize_vietnamese(correct_answer).split()
        
        if len(user_words) == len(correct_words):
            for i, (u_word, c_word) in enumerate(zip(user_words, correct_words)):
                if u_word != c_word:
                    start = " ".join(user_words[:i]).count(" ") + i
                    end = start + len(user_words[i])
                    text_widget.tag_add("wrong", f"1.{start}", f"1.{end}")
                    text_widget.tag_config("wrong", foreground=THEME["danger"])
    else:
        if len(user_input) == len(correct_answer):
            for i, (u_char, c_char) in enumerate(zip(user_input, correct_answer)):
                if u_char.lower() != c_char.lower():
                    text_widget.tag_add("wrong", f"1.{i}", f"1.{i+1}")
                    text_widget.tag_config("wrong", foreground=THEME["danger"])
    
    text_widget.config(state="disabled")
    
    hint_frame = tk.Frame(main_frame, bg=THEME["hint_bg"], bd=1, relief="solid")
    hint_frame.pack(fill="x", pady=(10, 0))
    
    hint_text = f"💡 {'Đáp án đúng' if LANGUAGE == 'vi' else 'Correct answer'}: '{correct_answer}'"
    tk.Label(hint_frame, text=hint_text, font=("Arial Unicode MS", 10), 
            bg=THEME["hint_bg"], fg=THEME["hint_fg"]).pack(padx=10, pady=5)
    
    btn_frame = tk.Frame(main_frame, bg=THEME["bg"])
    btn_frame.pack(fill="x", pady=(15, 0))
    
    tk.Button(btn_frame, text="Tra cứu" if LANGUAGE == "vi" else "Look up", 
             command=lambda: [open_hanzii(), result_window.lift()],
             bg=THEME["accent"], fg="white", font=THEME["font"],
             relief="flat", padx=15).pack(side="left", expand=True)
    
    tk.Button(btn_frame, text="Tiếp tục" if LANGUAGE == "vi" else "Continue", 
             command=lambda: [result_window.destroy(), load_new_word()],
             bg=THEME["success"], fg="white", font=THEME["font"],
             relief="flat", padx=15).pack(side="right", expand=True)

def show_history():
    if not history:
        messagebox.showinfo("Lịch sử" if LANGUAGE == "vi" else "History", 
                          "Chưa có dữ liệu lịch sử!" if LANGUAGE == "vi" else "No history data!")
        return
        
    history_window = tk.Toplevel(root)
    history_window.title("Lịch sử luyện tập" if LANGUAGE == "vi" else "Practice History")
    history_window.geometry("700x500")
    history_window.configure(bg=THEME["bg"])
    
    style = ttk.Style()
    style.theme_use("clam")
    style.configure("Treeview", 
                  background="#ffffff",
                  fieldbackground="#ffffff",
                  foreground=THEME["fg"],
                  rowheight=25,
                  font=THEME["font"])
    style.configure("Treeview.Heading", 
                  font=("Arial Unicode MS", 10, "bold"),
                  background=THEME["accent"],
                  foreground="white")
    
    tree = ttk.Treeview(history_window, columns=("time", "hanzi", "input", "result", "next_review", "details"), show="headings")
    tree.heading("time", text="Thời gian" if LANGUAGE == "vi" else "Time")
    tree.heading("hanzi", text="Chữ Hán" if LANGUAGE == "vi" else "Hanzi")
    tree.heading("input", text="Bạn nhập" if LANGUAGE == "vi" else "Your input")
    tree.heading("result", text="Kết quả" if LANGUAGE == "vi" else "Result")
    tree.heading("next_review", text="Ôn tiếp sau" if LANGUAGE == "vi" else "Next review")
    tree.heading("details", text="Chi tiết" if LANGUAGE == "vi" else "Details")
    
    tree.column("time", width=120, anchor="center")
    tree.column("hanzi", width=80, anchor="center")
    tree.column("input", width=120, anchor="center")
    tree.column("result", width=70, anchor="center")
    tree.column("next_review", width=100, anchor="center")
    tree.column("details", width=150, anchor="center")
    
    for item in reversed(history):
        color = THEME["success"] if item['result'] in ("Đúng", "Correct") else THEME["danger"]
        mode_text = "Hán tự" if item['mode'] == "hanzi" else "Nghĩa" if LANGUAGE == "vi" else "Meaning"
        next_review = item.get('next_review', datetime.now())
        if isinstance(next_review, str):
            next_review = datetime.strptime(next_review.split('.')[0], "%Y-%m-%d %H:%M:%S") if '.' in next_review else datetime.strptime(next_review, "%Y-%m-%d %H:%M:%S")
        
        days = (next_review - datetime.now()).days
        hours = (next_review - datetime.now()).seconds // 3600
        minutes = ((next_review - datetime.now()).seconds % 3600) // 60
        
        if days > 0:
            review_text = f"{days} {'ngày' if LANGUAGE == 'vi' else 'days'}"
        elif hours > 0:
            review_text = f"{hours} {'giờ' if LANGUAGE == 'vi' else 'hours'}"
        else:
            review_text = f"{minutes} {'phút' if LANGUAGE == 'vi' else 'mins'}"
        
        details = f"{mode_text} | {item['word'][1]}"
        
        tree.insert("", "end", 
                   values=(item['time'], item['word'][0], item['input'], item['result'], review_text, details),
                   tags=(color,))
    
    tree.tag_configure(THEME["success"], foreground=THEME["success"])
    tree.tag_configure(THEME["danger"], foreground=THEME["danger"])
    
    scroll = ttk.Scrollbar(history_window, orient="vertical", command=tree.yview)
    tree.configure(yscrollcommand=scroll.set)
    
    scroll.pack(side="right", fill="y")
    tree.pack(expand=True, fill="both", padx=10, pady=10)
    
    close_btn = tk.Button(history_window, 
                         text="Đóng" if LANGUAGE == "vi" else "Close", 
                         command=history_window.destroy,
                         bg=THEME["secondary"], fg="white", font=THEME["font"],
                         relief="flat", padx=15)
    close_btn.pack(pady=10)

def open_hanzii():
    global browser_opened
    if current_word and not browser_opened:
        try:
            url = f"https://hanzii.net/search/word/{current_word[0]}?hl=vi"
            webbrowser.open_new(url)
            browser_opened = True
        except Exception as e:
            messagebox.showerror("Lỗi" if LANGUAGE == "vi" else "Error", 
                               f"Không thể mở trình duyệt:\n{str(e)}" if LANGUAGE == "vi" else f"Can't open browser:\n{str(e)}")
            try:
                os.startfile(url)
            except:
                pass

# ========== LUYỆN VIẾT ==========
def open_drawing_area():
    global current_pen_color
    show_example_flag = False
    
    draw_window = tk.Toplevel(root)
    draw_window.title("Luyện viết chữ Hán" if LANGUAGE == "vi" else "Chinese Writing Practice")
    
    canvas_size = min(root.winfo_screenwidth() * 0.5, 400)
    grid_size = 8

    container = tk.Frame(draw_window, bg=THEME["bg"])
    container.pack(fill="both", expand=True, padx=10, pady=10)

    control_frame = tk.Frame(container, bg=THEME["bg"])
    control_frame.pack(fill="x", pady=(0, 10))

    tk.Label(control_frame, 
            text="Pen size:" if LANGUAGE == "en" else "Độ dày bút:", 
            bg=THEME["bg"], fg=THEME["fg"], 
            font=THEME["font"]).pack(side="left")
    
    pen_size_slider = tk.Scale(control_frame, from_=1, to=20, orient="horizontal",
                             bg=THEME["bg"], fg=THEME["fg"], highlightthickness=0,
                             command=lambda s: set_pen_size(int(s)))
    pen_size_slider.set(pen_size)
    pen_size_slider.pack(side="left", fill="x", expand=True)

    color_frame = tk.Frame(control_frame, bg=THEME["bg"])
    color_frame.pack(side="left", padx=10)
    
    tk.Label(color_frame, 
            text="Color:" if LANGUAGE == "en" else "Màu:", 
            bg=THEME["bg"], fg=THEME["fg"], 
            font=THEME["font"]).pack(side="left")
    
    def update_color_buttons():
        for child in color_frame.winfo_children():
            if isinstance(child, tk.Button) and hasattr(child, 'cget'):
                child.config(relief="sunken" if child.cget("bg") == current_pen_color else "raised")
    
    for color in pen_colors:
        btn = tk.Button(color_frame, bg=color, width=3, height=1, relief="sunken" if color == current_pen_color else "raised",
                      command=lambda c=color: [globals().update(current_pen_color=c), update_color_buttons()])
        btn.pack(side="left", padx=2)

    canvas = tk.Canvas(container, width=canvas_size, height=canvas_size, 
                      bg="white", highlightthickness=1, highlightbackground="#333333")
    canvas.pack(pady=10)

    def draw_grid():
        canvas.delete("grid")
        canvas.create_line(0, canvas_size/2, canvas_size, canvas_size/2, width=1.5, fill="#e0e0e0", tags="grid")
        canvas.create_line(canvas_size/2, 0, canvas_size/2, canvas_size, width=1.5, fill="#e0e0e0", tags="grid")
        canvas.create_line(0, 0, canvas_size, canvas_size, width=1.5, fill="#e0e0e0", tags="grid")
        canvas.create_line(canvas_size, 0, 0, canvas_size, width=1.5, fill="#e0e0e0", tags="grid")
        mid_x, mid_y = canvas_size/2, canvas_size/2
        canvas.create_line(0, mid_y, canvas_size, mid_y, width=1.5, fill="#e0e0e0", tags="grid")
        canvas.create_line(mid_x, 0, mid_x, canvas_size, width=1.5, fill="#e0e0e0", tags="grid")

    def clear_canvas():
        canvas.delete("all")
        draw_grid()
        nonlocal show_example_flag
        show_example_flag = False

    def show_example():
        nonlocal show_example_flag
        canvas.delete("example")
        if current_word:
            font_size = int(min(canvas_size/grid_size*1.5, 80))
            canvas.create_text(canvas_size/2, canvas_size/2, 
                             text=current_word[0], 
                             font=(THEME["hanzi_font"][0], font_size), 
                             fill="#776E65", tags="example")
            show_example_flag = True

    btn_frame = tk.Frame(container, bg=THEME["bg"])
    btn_frame.pack(fill="x")

    buttons = [
        ("Clear" if LANGUAGE == "en" else "Xóa", clear_canvas, THEME["secondary"]),
        ("Example" if LANGUAGE == "en" else "Mẫu", show_example, THEME["accent"]),
        (f"Look up {current_word[0]}" if current_word and LANGUAGE == "en" 
         else f"Tra {current_word[0]}" if current_word else "Tra cứu", 
         open_hanzii, THEME["accent"]),
        ("Close" if LANGUAGE == "en" else "Đóng", draw_window.destroy, THEME["danger"])
    ]

    for text, cmd, color in buttons:
        tk.Button(btn_frame, text=text, command=cmd,
                bg=color, fg="white", font=THEME["font"],
                relief="flat", padx=10).pack(side="left", padx=5, fill="x", expand=True)

    last_x, last_y = None, None

    def start_draw(event):
        nonlocal last_x, last_y
        last_x, last_y = event.x, event.y

    def draw(event):
        nonlocal last_x, last_y
        if last_x and last_y:
            canvas.create_line(last_x, last_y, event.x, event.y, 
                             width=pen_size, fill=current_pen_color,
                             capstyle="round", smooth=True)
        last_x, last_y = event.x, event.y

    canvas.bind('<Button-1>', start_draw)
    canvas.bind('<B1-Motion>', draw)
    canvas.bind('<ButtonRelease-1>', lambda e: (start_draw(e), draw(e)))

    draw_grid()

def set_pen_size(size):
    global pen_size
    pen_size = size

# ========== SPACED REPETITION ==========
def update_spaced_repetition(word, performance_rating):
    if word not in spaced_repetition_data:
        spaced_repetition_data[word] = {
            "interval": 1,
            "repetition": 1,
            "ease_factor": 2.5,
            "next_review": datetime.now()
        }
    
    card = spaced_repetition_data[word]
    
    if performance_rating >= 3:
        if card["repetition"] == 1:
            card["interval"] = 1
        elif card["repetition"] == 2:
            card["interval"] = 6
        else:
            card["interval"] = math.ceil(card["interval"] * card["ease_factor"])
        
        card["ease_factor"] = max(1.3, card["ease_factor"] + 0.1 - (5 - performance_rating) * 0.08)
        card["repetition"] += 1
    else:
        card["interval"] = 1
        card["repetition"] = 1
        card["ease_factor"] = max(1.3, card["ease_factor"] - 0.2)
    
    card["next_review"] = datetime.now() + timedelta(days=card["interval"])
    return card["interval"]

def get_next_word():
    if not words:
        return None
    
    now = datetime.now()
    
    due_words = [w for w in words 
                if w[0] in spaced_repetition_data 
                and spaced_repetition_data[w[0]]["next_review"] <= now]
    
    if due_words:
        return random.choice(due_words)
    
    new_words = [w for w in words if w[0] not in spaced_repetition_data]
    if new_words:
        return random.choice(new_words)
    
    return min(words, key=lambda w: spaced_repetition_data[w[0]]["next_review"])

# ========== LƯU/LOAD TIẾN TRÌNH ==========
def save_progress():
    data = {
        'spaced_repetition': spaced_repetition_data,
        'history': history,
        'score': score,
        'total': total,
        'language': LANGUAGE,
        'last_vocab_file': last_vocab_file,
        'no_tone_mode': no_tone_mode,
        'first_import': first_import
    }
    try:
        with open("hsk_progress.json", "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2, default=str)
    except Exception as e:
        print("Lỗi khi lưu tiến trình:", e)

def load_progress():
    global spaced_repetition_data, history, score, total, LANGUAGE, last_vocab_file, no_tone_mode, first_import
    
    def parse_datetime(dt_str):
        try:
            if '.' in dt_str:
                return datetime.strptime(dt_str.split('.')[0], "%Y-%m-%d %H:%M:%S")
            return datetime.strptime(dt_str, "%Y-%m-%d %H:%M:%S")
        except ValueError:
            return datetime.now()
    
    try:
        with open("hsk_progress.json", "r", encoding="utf-8") as f:
            data = json.load(f)
            
            spaced_repetition_data = {k: {
                **v, 
                'next_review': parse_datetime(v['next_review']) if isinstance(v['next_review'], str) else datetime.now()
            } for k, v in data.get('spaced_repetition', {}).items()}
            
            history = data.get('history', [])
            score = data.get('score', 0)
            total = data.get('total', 0)
            LANGUAGE = data.get('language', "vi")
            last_vocab_file = data.get('last_vocab_file', "")
            no_tone_mode = data.get('no_tone_mode', False)
            first_import = data.get('first_import', True)
            
            if last_vocab_file and os.path.exists(last_vocab_file):
                load_vocab_file(last_vocab_file)
            elif last_vocab_file:
                choice = messagebox.askyesno(
                    "Cảnh báo" if LANGUAGE == "vi" else "Warning",
                    f"Không tìm thấy file từ vựng trước đó:\n{last_vocab_file}\n\n{'Bạn có muốn chọn file khác?' if LANGUAGE == 'vi' else 'Do you want to select another file?'}")
                if choice:
                    import_file()
                last_vocab_file = ""
                
    except (FileNotFoundError, json.JSONDecodeError) as e:
        print(f"Error loading progress: {e}")
        spaced_repetition_data = {}
        history = []
        score = total = 0
        LANGUAGE = "vi"
        last_vocab_file = ""
        no_tone_mode = False
        first_import = True

def toggle_language():
    global LANGUAGE
    LANGUAGE = "en" if LANGUAGE == "vi" else "vi"
    update_ui_language()
    update_display()

def update_ui_language():
    root.title("HSK Vocabulary Trainer Pro_version-Quanxike9x")
    
    pomodoro_btn.config(text="Start Pomodoro" if LANGUAGE == "en" else "Bắt đầu Pomodoro")
    hanzii_btn.config(text="Look up" if LANGUAGE == "en" else "Tra cứu")
    action_btn1.config(text="Check (Enter)" if LANGUAGE == "en" else "Kiểm tra (Enter)")
    action_btn2.config(text="Skip" if LANGUAGE == "en" else "Bỏ qua")
    lang_btn.config(text="VI" if LANGUAGE == "en" else "EN")
    
    for i, (text_key, cmd, color) in enumerate(buttons):
        btn_text = {
            "import": "Import" if LANGUAGE == "en" else "Nhập từ",
            "history": "History" if LANGUAGE == "en" else "Lịch sử",
            "practice": "Practice" if LANGUAGE == "en" else "Luyện viết",
            "lookup": "Look up" if LANGUAGE == "en" else "Tra cứu",
            "toggle_mode": "Toggle Mode" if LANGUAGE == "en" else "Đổi chế độ"
        }.get(text_key, text_key)
        btn_frame.grid_slaves(row=0, column=i)[0].config(text=btn_text)
    
    if 'no_tone_btn' in globals():
        no_tone_btn.config(text="Disable no tone" if no_tone_mode else "Enable no tone" if LANGUAGE == "en" else "Tắt không dấu" if no_tone_mode else "Bật không dấu")
    
    mode_label.config(text=f"{'Mode' if LANGUAGE == 'en' else 'Chế độ'}: {'Enter Meaning' if input_mode == 'hanzi' else 'Enter Hanzi'}")
    update_file_label()
    
    score_text = f"{'Score' if LANGUAGE == 'en' else 'Điểm'}: {score}/{total}"
    if total > 0:
        score_text += f" ({'Rate' if LANGUAGE == 'en' else 'Tỉ lệ'}: {score/total*100:.1f}%)"
    score_label.config(text=score_text)
    
    if pomodoro_start_time is None:
        pomodoro_label.config(text="Not started" if LANGUAGE == "en" else "Chưa bắt đầu")

# ========== GIAO DIỆN CHÍNH ==========
root = tk.Tk()
root.title("HSK Vocabulary Trainer Pro - Quanxike9x")
root.geometry("700x650")
root.configure(bg=THEME["bg"])
root.resizable(False, False)

# Header
header = tk.Frame(root, bg=THEME["accent"], height=80)
header.pack(fill="x")

tk.Label(header, text="汉语水平_HSK_TRAINER", 
        font=THEME["title_font"], fg="white", bg=THEME["accent"]).pack(pady=20)

# Pomodoro UI
pomodoro_frame = tk.Frame(root, bg=THEME["bg"])
pomodoro_frame.pack(fill="x", padx=10, pady=5)

pomodoro_btn = tk.Button(pomodoro_frame, text="Bắt đầu Pomodoro", command=start_pomodoro,
                       bg="#e74c3c", fg="white", font=THEME["font"])
pomodoro_btn.pack(side="left")

pomodoro_label = tk.Label(pomodoro_frame, text="Chưa bắt đầu", 
                         bg=THEME["bg"], fg=THEME["fg"], font=THEME["font"])
pomodoro_label.pack(side="right")

# Main content
main_frame = tk.Frame(root, bg=THEME["bg"])
main_frame.pack(fill="both", expand=True, padx=30, pady=20)

mode_label = tk.Label(main_frame, text="Chế độ: Nhập chữ Hán", 
                     font=THEME["font"], bg=THEME["bg"], fg=THEME["accent"])
mode_label.pack(anchor="w", pady=(0, 5))

hanzi_label = tk.Label(main_frame, text="(Nhập chữ Hán tương ứng)", 
                      font=THEME["font"], fg=THEME["fg"], bg=THEME["bg"])
hanzi_label.pack(pady=(0, 20))

entry = create_vietnamese_entry()
entry.focus()

result_label = tk.Label(main_frame, text="", font=THEME["font"], bg=THEME["bg"])
result_label.pack()

score_label = tk.Label(main_frame, text="Điểm: 0/0 (Tỉ lệ: 0%)", 
                     font=("Arial Unicode MS", 12, "bold"), fg=THEME["fg"], bg=THEME["bg"])
score_label.pack(pady=(0, 20))

btn_frame = tk.Frame(main_frame, bg=THEME["bg"])
btn_frame.pack(fill="x")

buttons = [
    ("import", import_file, THEME["accent"]),
    ("history", show_history, THEME["accent"]),
    ("practice", open_drawing_area, THEME["accent"]),
    ("lookup", open_hanzii, THEME["accent"]),
    ("toggle_mode", toggle_input_mode, THEME["secondary"])
]

for i, (text_key, cmd, color) in enumerate(buttons):
    btn = tk.Button(btn_frame, text=text_key, command=cmd,
                   bg=color, fg="white", 
                   font=THEME["font"],
                   relief="flat", padx=5)
    btn.grid(row=0, column=i, padx=2, pady=2, sticky="ew")
    if text_key == "lookup":
        hanzii_btn = btn
        hanzii_btn.config(state=tk.DISABLED)

no_tone_btn = tk.Button(btn_frame, text="Bật không dấu", 
                       command=toggle_no_tone_mode,
                       bg=THEME["secondary"], fg="white", 
                       font=THEME["font"],
                       relief="flat", padx=5)
no_tone_btn.grid(row=0, column=len(buttons), padx=2, pady=2, sticky="ew")

action_frame = tk.Frame(main_frame, bg=THEME["bg"])
action_frame.pack(fill="x", pady=(10, 0))

action_btn1 = tk.Button(action_frame, text="Kiểm tra (Enter)", 
                      bg=THEME["success"], fg="white",
                      command=check_answer, 
                      font=THEME["font"],
                      relief="flat", padx=5)
action_btn1.pack(side="left", expand=True, fill="x")

action_btn2 = tk.Button(action_frame, text="Bỏ qua", 
                      bg=THEME["danger"], fg="white",
                      command=skip_word,
                      font=THEME["font"],
                      relief="flat", padx=5)
action_btn2.pack(side="left", expand=True, fill="x")

# Help và ngôn ngữ
help_btn = tk.Button(root, text="?", command=show_help,
                    bg=THEME["accent"], fg="white",
                    font=("Arial", 14, "bold"),
                    relief="flat", width=3, height=1)
help_btn.place(relx=0.95, rely=0.02, anchor="ne")

lang_btn = tk.Button(root, text="EN/VI", command=toggle_language,
                    bg=THEME["secondary"], fg="white",
                    font=("Arial", 10, "bold"),
                    relief="flat", width=4, height=1)
lang_btn.place(relx=0.88, rely=0.02, anchor="ne")

# File label
file_label = tk.Label(root, text="", font=("Arial", 8), fg="gray")
file_label.pack(side="bottom", anchor="w", padx=10)

footer = tk.Frame(root, bg=THEME["bg"], height=30)
footer.pack(fill="x", side="bottom")

tk.Label(footer, text="© 2023 Quanxike9x", cursor="hand2",
        font=("Arial Unicode MS", 8), fg="#777777", bg=THEME["bg"]).pack(side="right", padx=10)

# Xử lý sự kiện
root.bind('<Return>', check_answer)
root.protocol("WM_DELETE_WINDOW", lambda: [save_progress(), root.destroy()])

# Khởi tạo
load_progress()
update_ui_language()
update_file_label()

root.mainloop()
