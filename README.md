from flask import Flask, render_template_string, abort
import os, json, requests

app = Flask(__name__)

QURAN_JSON = "quran_data.json"
API_BASE = "https://quran-api-id.vercel.app/surah/{}"

# ==== Fungsi fetch data dari API ====
def fetch_all_surah():
    surahs = []
    print("🚀 Mengambil data 114 surah dari API...")
    for i in range(1, 115):
        try:
            resp = requests.get(API_BASE.format(i), timeout=30)
            resp.raise_for_status()
            wrapper = resp.json()
            data = wrapper.get("data", {})

            name_data = data.get("name", {})
            verses_data = data.get("verses", [])

            surah = {
                "id": data.get("number", i),
                "name": name_data.get("short", ""),
                "transliteration": name_data.get("transliteration", {}).get("id", ""),
                "translation": name_data.get("translation", {}).get("id", ""),
                "verses": []
            }

            for v in verses_data:
                num = v.get("number", {}).get("inSurah") if isinstance(v.get("number"), dict) else v.get("number")
                text = v.get("text", {}).get("arab") if isinstance(v.get("text"), dict) else v.get("text")
                trans = v.get("translation", {}).get("id") if isinstance(v.get("translation"), dict) else v.get("translation")
                surah["verses"].append({
                    "id": num,
                    "text": text,
                    "translation": trans
                })

            print(f"Fetched surah {i}: {surah['transliteration']}")
            surahs.append(surah)
        except Exception as e:
            print(f"❌ Gagal fetch surah {i}: {e}")
            break

    # Simpan cache
    with open(QURAN_JSON, "w", encoding="utf-8") as f:
        json.dump(surahs, f, ensure_ascii=False, indent=2)
    print(f"✅ Data Quran disimpan ke {QURAN_JSON}")
    return surahs

# ==== Load data (cache atau fetch) ====
if os.path.exists(QURAN_JSON):
    with open(QURAN_JSON, "r", encoding="utf-8") as f:
        surahs = json.load(f)
else:
    surahs = fetch_all_surah()

# ==== Ubah link background di sini ====
BG_IMAGE = "https://images.unsplash.com/photo-1507525428034-b723cf961d3e"  # Ganti dengan link jpg kamu

# ==== Template HTML ====
INDEX_HTML = """
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <title>Kelompok 6 Al-Quran Digital</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <style>
    body {
      background-image: url('https://images.unsplash.com/photo-1507525428034-b723cf961d3e');
      background-size: cover;
      background-attachment: fixed;
    }
    .card {
      backdrop-filter: blur(8px);
      background-color: rgba(255, 255, 255, 0.75);
    }
  </style>
</head>
<body class="min-h-screen text-gray-800">
  <div class="container mx-auto p-6">
    <h1 class="text-4xl font-bold mb-6 text-center text-blue-900">
    Kelompok 6 Web Al-Quran Digital 📖 Daftar Surah</h1>
    <ul class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
      {% for surah in surahs %}
      <li>
        <a href="/surah/{{ surah.id }}" class="card block p-4 rounded shadow hover:shadow-lg transition duration-300">
          <h2 class="text-xl font-bold text-blue-800">{{ surah.id }}. {{ surah.name }}</h2>
          <p class="text-gray-700">{{ surah.transliteration }}</p>
          <p class="text-gray-600">{{ surah.translation }}</p>
        </a>
      </li>
      {% endfor %}
    </ul>
  </div>
</body>
</html>
"""

SURAH_HTML = """
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <title>{{ surah.transliteration }}</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <style>
    body {
      background-image: url('https://images.unsplash.com/photo-1507525428034-b723cf961d3e');
      background-size: cover;
      background-attachment: fixed;
    }
    .card {
      backdrop-filter: blur(8px);
      background-color: rgba(255, 255, 255, 0.75);
    }
  </style>
</head>
<body class="min-h-screen text-gray-800">
  <div class="container mx-auto p-6">
    <a href="/" class="text-blue-800 hover:underline mb-4 inline-block">← Kembali ke daftar surah</a>
    <h1 class="text-3xl font-bold mb-2">{{ surah.name }} ({{ surah.transliteration }})</h1>
    <h3 class="text-lg text-gray-700 mb-6">{{ surah.translation }}</h3>

    <div class="space-y-6">
      {% for ayat in surah.verses %}
      <div class="card p-4 rounded shadow">
        <p class="text-gray-800 font-semibold mb-1">Ayat {{ ayat.id }}</p>
        <p class="text-2xl arabic mb-2" dir="rtl" style="font-family: 'Scheherazade', serif;">{{ ayat.text }}</p>
        <p class="text-gray-700">{{ ayat.translation }}</p>
      </div>
      {% endfor %}
    </div>
  </div>
</body>
</html>
"""

# ==== Routes ====
@app.route("/")
def index():
    return render_template_string(INDEX_HTML, surahs=surahs)

@app.route("/surah/<int:surah_id>")
def surah_page(surah_id):
    if surah_id < 1 or surah_id > len(surahs):
        abort(404)
    surah = surahs[surah_id - 1]
    return render_template_string(SURAH_HTML, surah=surah)

# ==== Run server ====
if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=5000) 
