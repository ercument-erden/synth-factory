Elbette! Aşağıdaki **tek komut**, tüm sistemi senin Fabrika CEO projene uygun şekilde kurar:

- ✅ Tamamen **ücretsiz ve self-hosted**  
- ✅ **Ollama** ile yerel LLM (Phi-3, Gemma2, Mistral)  
- ✅ Gerçek sistem izleme (`psutil`)  
- ✅ SQLite veritabanı  
- ✅ Docker ile çalıştırılabilir  
- ✅ Harici API’ye **bağımlılık yok**  

---

### 🔧 Tek Komut (Terminal’e Yapıştır):

```bash
mkdir -p ~/fabrika/{data,logs} && cd ~/fabrika && cat > docker-compose.yml <<'EOF'
version: '3.8'
services:
  ollama:
    image: ollama/ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_/root/.ollama
    deploy:
      resources:
        limits:
          memory: 6G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:11434"]
      interval: 30s
      timeout: 10s
      retries: 3

  fabrika-nexus:
    image: python:3.10-slim
    working_dir: /app
    volumes:
      - .//app/data
      - ./logs:/app/logs
      - .:/app
    environment:
      - OLLAMA_HOST=http://ollama:11434
    depends_on:
      ollama:
        condition: service_healthy
    command: |
      sh -c "
        pip install aiohttp psutil sqlite3 &&
        python main.py
      "

volumes:
  ollama_
EOF

cat > main.py <<'EOF'
import asyncio
import os
import json
import sqlite3
import psutil
from datetime import datetime
import aiohttp

OLLAMA_HOST = os.getenv("OLLAMA_HOST", "http://localhost:11434")
MODELS = ["phi3", "gemma2", "mistral"]

# Basit veritabanı
os.makedirs("data", exist_ok=True)
conn = sqlite3.connect("data/innovation.db")
conn.execute("CREATE TABLE IF NOT EXISTS runs (id INTEGER PRIMARY KEY, result TEXT);")
conn.commit()

class OllamaAgent:
    def __init__(self, model):
        self.model = model
        self.session = None

    async def ensure_session(self):
        if not self.session:
            self.session = aiohttp.ClientSession()

    async def generate(self, prompt):
        await self.ensure_session()
        try:
            async with self.session.post(
                f"{OLLAMA_HOST}/api/generate",
                json={"model": self.model, "prompt": prompt, "stream": False}
            ) as r:
                data = await r.json()
                return data.get("response", "No response")
        except Exception as e:
            return f"Error: {str(e)}"

async def dialectic_debate(prompt: str):
    print("🤖 Diyalektik motor başlatılıyor...")
    agents = {m: OllamaAgent(m) for m in MODELS}
    positions = {}

    for name, agent in agents.items():
        print(f"   🧠 {name} düşünüyor...")
        resp = await agent.generate(f"Kısa ve net çözüm öner: {prompt}")
        positions[name] = resp[:200]

    # Sentez
    synthesis = " | ".join(f"{k}: {v}" for k, v in positions.items())
    result = {
        "prompt": prompt,
        "positions": positions,
        "synthesis": synthesis,
        "system": {
            "cpu": psutil.cpu_percent(),
            "ram": psutil.virtual_memory().percent,
            "disk": psutil.disk_usage('/').percent
        },
        "timestamp": datetime.now().isoformat()
    }

    # Kaydet
    conn.execute("INSERT INTO runs (result) VALUES (?)", (json.dumps(result),))
    conn.commit()
    print("✅ Sonuç kaydedildi. Özet:", synthesis[:100] + "...")
    return result

async def main():
    print("🚀 Fabrika Nexus (Ücretsiz & Self-Hosted) Başlatılıyor...")
    # Modelleri önceden çek (isteğe bağlı)
    async with aiohttp.ClientSession() as s:
        for m in MODELS:
            print(f"📥 Model indiriliyor: {m}")
            await s.post(f"{OLLAMA_HOST}/api/pull", json={"name": m, "stream": False})

    # Test çalıştırması
    await dialectic_debate("500$ ile nasıl e-ticaret MVP'si kurulur?")
    print("\n💡 Sistem çalışıyor! Gerçek çağrılar için HTTP API eklenebilir.")

if __name__ == "__main__":
    asyncio.run(main())
EOF

docker compose up --build
```

---

### 📌 Ne Yapar Bu Komut?

1. `~/fabrika` klasörünü oluşturur  
2. `docker-compose.yml` ve `main.py` dosyalarını yazar  
3. Gerekli modelleri (`phi3`, `gemma2`, `mistral`) otomatik çeker  
4. Gerçek CPU/RAM/Disk izleme yapar  
5. Sonuçları `data/innovation.db`’ye kaydeder  
6. Tümü **ücretsiz, offline, self-hosted**

> 💡 İlk çalıştırma 5-10 dakika sürebilir (modeller indirilir). Sonraki çalıştırmalar anında başlar.

İstersen bu sisteme daha sonra **Uptime Kuma**, **Vaultwarden**, veya **Gitea** gibi self-hosted araçları da ekleyebiliriz.

Hazır mısın? Komutu kopyala ve Ubuntu terminaline yapıştır. 🚀
