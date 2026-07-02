import os
from flask import Flask, request, render_template_string, redirect, url_for, send_from_directory, jsonify
from werkzeug.utils import secure_filename

app = Flask(__name__)

# تنظیمات
UPLOAD_FOLDER = 'uploaded_afl_files'
ALLOWED_EXTENSIONS = {'afl'}

app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
os.makedirs(UPLOAD_FOLDER, exist_ok=True)

# ✅ تابع allowed_file
def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

# کل قالب HTML به صورت یک رشته چندخطی (Multi-line String)
HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="fa" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AFL Uploader</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Vazirmatn:wght@100;400;700&display=swap');
        body { font-family: 'Vazirmatn', sans-serif; background-color: #0f172a; color: #f8fafc; }
        .glass { background: rgba(30, 41, 59, 0.7); backdrop-filter: blur(12px); border: 1px solid rgba(255, 255, 255, 0.1); }
        .gradient-text { background: linear-gradient(90deg, #38bdf8, #818cf8); -webkit-background-clip: text; -webkit-text-fill-color: transparent; }
        @keyframes bounce {
            0%, 100% { transform: translateY(0); }
            50% { transform: translateY(-0.5rem); }
        }
        .animate-bounce {
            animation: bounce 1s infinite;
        }
        .animate-bounce-delay-1 { animation-delay: 0.1s; }
        .animate-bounce-delay-2 { animation-delay: 0.2s; }
    </style>
</head>
<body class="min-h-screen flex items-center justify-center p-4">
    <div class="max-w-md w-full glass rounded-3xl p-8 shadow-2xl">
        <div class="text-center mb-8">
            <div class="inline-block p-4 rounded-2xl bg-blue-500/10 mb-4"><i class="fas fa-cloud-upload-alt text-4xl text-blue-400"></i></div>
            <h1 class="text-3xl font-bold gradient-text">AFL Uploader</h1>
            <p class="text-slate-400 mt-2 text-sm">آپلود فایل‌های .afl شما</p>
        </div>
        <form id="uploadForm" enctype="multipart/form-data" action="{{ url_for('upload_file') }}" method="POST" class="space-y-6">
            <div class="relative group">
                <input type="file" id="fileInput" name="file" accept=".afl" class="hidden" required>
                <label for="fileInput" class="flex flex-col items-center justify-center w-full h-32 border-2 border-dashed border-slate-600 rounded-2xl cursor-pointer group-hover:border-blue-500 group-hover:bg-blue-500/5 transition-all">
                    <i class="fas fa-file-code text-3xl text-slate-500 group-hover:text-blue-400 mb-2"></i>
                    <span id="fileNameDisplay" class="text-slate-400 text-sm">انتخاب فایل AFL...</span>
                </label>
            </div>
            <button type="submit" id="submitBtn" class="w-full bg-blue-600 hover:bg-blue-500 text-white font-bold py-4 rounded-xl transition-all transform active:scale-95 flex items-center justify-center">
                <span>آپلود فایل</span><i class="fas fa-paper-plane mr-2"></i>
            </button>
        </form>
        <div id="statusArea" class="mt-6 hidden">
            <div id="loader" class="flex items-center justify-center space-x-2 space-x-reverse">
                <div class="w-3 h-3 bg-blue-400 rounded-full animate-bounce"></div>
                <div class="w-3 h-3 bg-blue-400 rounded-full animate-bounce animate-bounce-delay-1"></div>
                <div class="w-3 h-3 bg-blue-400 rounded-full animate-bounce animate-bounce-delay-2"></div>
                <span class="text-sm text-slate-400 mr-2">در حال ارسال...</span>
            </div>
            <div id="resultBox" class="hidden p-4 rounded-xl text-sm"></div>
        </div>
        {% if message %}
            <div id="messageArea" class="mt-6 p-4 rounded-xl text-sm {% if 'موفقیت' in message %}bg-green-500/10 text-green-400 border border-green-500/20{% else %}bg-red-500/10 text-red-400 border border-red-500/20{% endif %}">
                {{ message|safe }}
            </div>
        {% endif %}
    </div>

    <script>
        const fileInput = document.getElementById('fileInput');
        const fileNameDisplay = document.getElementById('fileNameDisplay');
        const uploadForm = document.getElementById('uploadForm');
        const statusArea = document.getElementById('statusArea');
        const loader = document.getElementById('loader');
        const resultBox = document.getElementById('resultBox');
        const submitBtn = document.getElementById('submitBtn');

        fileInput.onchange = () => {
            if (fileInput.files.length > 0) {
                fileNameDisplay.textContent = fileInput.files[0].name;
                fileNameDisplay.classList.add('text-blue-400');
            }
        };

        uploadForm.onsubmit = async (e) => {
            e.preventDefault();
            statusArea.classList.remove('hidden');
            loader.classList.remove('hidden');
            resultBox.classList.add('hidden');
            submitBtn.disabled = true;
            
            const messageArea = document.getElementById('messageArea');
            if (messageArea) messageArea.classList.add('hidden');

            const formData = new FormData();
            formData.append('file', fileInput.files[0]);

            try {
                const response = await fetch(uploadForm.action, { method: 'POST', body: formData });
                const data = await response.json();
                loader.classList.add('hidden');
                resultBox.classList.remove('hidden');

                if (response.ok) {
                    resultBox.className = 'p-4 rounded-xl text-sm bg-green-500/10 text-green-400 border border-green-500/20';
                    resultBox.innerHTML = `<div class="text-center"><p class="font-bold">موفقیت‌آمیز!</p><a href="${data.url}" target="_blank" class="text-xs underline break-all">${data.url}</a></div>`;
                } else {
                    resultBox.className = 'p-4 rounded-xl text-sm bg-red-500/10 text-red-400 border border-red-500/20';
                    resultBox.innerHTML = `<p class="text-center">خطا: ${data.error || 'نامشخص'}</p>`;
                }
            } catch (err) {
                loader.classList.add('hidden');
                resultBox.classList.remove('hidden');
                resultBox.className = 'p-4 rounded-xl text-sm bg-red-500/10 text-red-400 border border-red-500/20';
                resultBox.innerHTML = `<p class="text-center">خطای ارتباط با سرور</p>`;
                console.error("Upload error:", err);
            } finally {
                submitBtn.disabled = false;
            }
        };
    </script>
</body>
</html>
"""

@app.route('/')
def index():
    return render_template_string(HTML_TEMPLATE, message=None)

@app.route('/upload', methods=['POST'])
def upload_file():
    if 'file' not in request.files:
        return render_template_string(HTML_TEMPLATE, message="خطا: هیچ فایلی ارسال نشده است."), 400
    
    file = request.files['file']
    
    if file.filename == '':
        return render_template_string(HTML_TEMPLATE, message="خطا: نام فایل خالی است."), 400
    
    if file and allowed_file(file.filename):
        filename = secure_filename(file.filename)
        filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
        
        try:
            file.save(filepath)
            download_link = url_for('download_file', name=filename, _external=True)
            
            response_data = {
                "url": download_link,
                "status": "success",
                "message": f"فایل '{filename}' با موفقیت آپلود شد."
            }
            return jsonify(response_data), 200
            
        except Exception as e:
            print(f"Error saving file: {e}")
            return render_template_string(HTML_TEMPLATE, message=f"خطا در ذخیره فایل: {str(e)}"), 500
            
    else:
        return render_template_string(HTML_TEMPLATE, message="خطا: فرمت فایل مجاز نیست. لطفاً فایل با پسوند .afl انتخاب کنید."), 400

@app.route('/uploads/<name>')
def download_file(name):
    try:
        return send_from_directory(app.config['UPLOAD_FOLDER'], name, as_attachment=True)
    except FileNotFoundError:
        return "فایل یافت نشد", 404

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
