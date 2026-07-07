from flask import Flask, request, render_template_string

app = Flask(__name__)

# --- استایل مدرن و تیره ---
STYLE = """
<style>
    body { background-color: #0f0f0f; color: #e0e0e0; font-family: sans-serif; padding: 30px; }
    .container { max-width: 900px; margin: auto; background: #1a1a1a; padding: 30px; border-radius: 15px; box-shadow: 0 10px 30px rgba(0,0,0,0.7); }
    h2 { color: #007acc; border-bottom: 1px solid #333; padding-bottom: 10px; }
    textarea { width: 100%; height: 200px; background: #252526; color: #dcdcdc; border: 1px solid #3c3c3c; padding: 15px; font-size: 15px; font-family: monospace; box-sizing: border-box; border-radius: 8px; resize: vertical; }
    .btn { width: 100%; padding: 15px; background: #007acc; color: white; border: none; border-radius: 8px; font-size: 18px; font-weight: bold; cursor: pointer; margin-top: 15px; }
    .console { margin-top: 25px; background: #000; padding: 15px; border-radius: 8px; border-left: 5px solid #007acc; font-family: monospace; font-size: 14px; color: #00ff00; }
    .links-section { margin-top: 25px; padding: 15px; background: #252526; border-radius: 8px; border: 1px solid #007acc; }
    .link-item { display: block; padding: 10px; color: #4fc1ff; text-decoration: none; font-weight: bold; margin-top: 5px; }
    .guide { margin-top: 20px; font-size: 13px; color: #777; }
    .site-content { background: white; color: #333; padding: 50px; border-radius: 10px; min-height: 300px; text-align: center; font-size: 24px; box-shadow: 0 5px 15px rgba(0,0,0,0.3); }
</style>
"""

class AFLEngine:
    def __init__(self):
        self.routes = {}

    def execute(self, code):
        lines = code.split('\n')
        output_log = []
        for line in lines:
            line = line.strip()
            if not line: continue

            if line.startswith("RT "):
                path_part = line[3:].strip()
                path = f"/{path_part}" if not path_part.startswith("/") else path_part
                content = f"<div class='site-content'><h1>🚀 Welcome to {path}</h1><p>Built by AFL!</p></div>"
                self.routes[path] = content
                output_log.append(f"✨ SUCCESS: Created -> {path}")
            elif line.startswith("R "):
                output_log.append(f"> {line[2:].strip()}")
            else:
                output_log.append(f"⚠️ Unknown: {line}")
        return "<br>".join(output_log)

engine = AFLEngine()

# --- قالب اصلی (بدون استفاده از Jinja2 برای جلوگیری از خطا) ---
def get_html(log_content, code, links_html):
    return f"""
    <!DOCTYPE html>
    <html>
    <head><title>AFL Builder</title>{STYLE}</head>
    <body>
        <div class="container">
            <h2>🛠️ AFL Web Engine</h2>
            <form method="POST" action="/run">
                <textarea name="code" placeholder="Type AFL code...">{code}</textarea>
                <button type="submit" class="btn">🚀 BUILD & DEPLOY</button>
            </form>
            <div class="console"><strong>Logs:</strong><br>{log_content}</div>
            {links_html}
            <div class="guide"><b>Commands:</b> RT /name | R message</div>
        </div>
    </body>
    </html>
    """

@app.route('/')
def index():
    return get_html("System Ready...", "", "")

@app.route('/run', methods=['POST'])
def run():
    code = request.form.get('code', '')
    log_display = engine.execute(code)
    
    # ساخت بخش لینک‌ها به صورت دستی
    links_html = ""
    if engine.routes:
        links_html = '<div class="links-section"><strong>🔗 Live Websites:</strong>'
        for path in engine.routes.keys():
            links_html += f'<a class="link-item" href="{path}">🌐 Open: {path}</a>'
        links_html += '</div>'
    
    return get_html(log_display, code, links_html)

@app.before_request
def handle_dynamic_routes():
    path = request.path
    if path in engine.routes:
        return f"""
        <html><head>{STYLE}</head>
        <body style="display:flex; justify-content:center; align-items:center; min-height:100vh;">
            <div style="max-width:800px; width:100%;">{engine.routes[path]}
            <br><center><a href="/" style="color:#007acc;">⬅️ Back</a></center></div>
        </body></html>
        """

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
