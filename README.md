# 🚀 Tutorial: Deployment Flask z Podman + Caddy

## Czego się nauczysz?
- Jak postawić własny serwer za 2 EUR/miesiąc
- Jak uruchomić swoje aplikacje Flask w kontenerach
- Jak skonfigurować automatyczne SSL
- Jak zarządzać wieloma projektami na jednym serwerze

---

## Krok 1: Kupujemy VPS i łączymy się z nim

### Co kupić?
- **IONOS VPS Basic**: 2 EUR/miesiąc
- **System**: Ubuntu 22.04 LTS
- **Zasoby**: 1 vCPU, 1GB RAM (wystarczy na start)

### Pierwsze połączenie
```bash
# Dostajesz IP i hasło na email
ssh root@123.456.789.123
```

### Tworzymy bezpiecznego użytkownika
```bash
# Tworzymy nowego użytkownika (nie używamy root!)
adduser junior
usermod -aG sudo junior

# Przechodzimy na nowego użytkownika
su - junior
```

---

## Krok 2: Instalujemy wszystko co potrzebne

### Aktualizujemy system
```bash
sudo apt update && sudo apt upgrade -y
```

### Instalujemy Podman (zamiennik Docker-a)
```bash
sudo apt install podman -y

# Sprawdzamy czy działa
podman --version
```

### Instalujemy Caddy (nasz reverse proxy z darmowym SSL)
```bash
# Dodajemy repozytorium Caddy
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list

# Instalujemy
sudo apt update
sudo apt install caddy -y

# Sprawdzamy
caddy version
```

---

## Krok 3: Przygotowujemy strukturę folderów

```bash
# Tworzymy foldery dla naszych projektów
mkdir -p ~/projekty/{sklep,blog,api,portfolio}
cd ~/projekty
```

---

## Krok 4: Przygotowujemy pierwszą aplikację

### Przykład prostej aplikacji Flask

**~/projekty/sklep/app.py**:
```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return '<h1>Witaj w moim sklepie!</h1>'

@app.route('/produkty')
def produkty():
    return '<h1>Lista produktów</h1>'

if __name__ == '__main__':
    app.run(debug=True)
```

**~/projekty/sklep/requirements.txt**:
```
Flask==2.3.3
gunicorn==21.2.0
```

**~/projekty/sklep/Dockerfile**:
```dockerfile
# Używamy lekkiego obrazu Python
FROM python:3.11-slim

# Ustawiamy folder roboczy
WORKDIR /app

# Kopiujemy requirements i instalujemy zależności
COPY requirements.txt .
RUN pip install -r requirements.txt

# Kopiujemy całą aplikację
COPY . .

# Otwieramy port 5000
EXPOSE 5000

# Uruchamiamy aplikację z Gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

---

## Krok 5: Budujemy i uruchamiamy kontener

```bash
cd ~/projekty/sklep

# Budujemy obraz kontenera
podman build -t moj-sklep .

# Uruchamiamy kontener
podman run -d --name sklep -p 5001:5000 moj-sklep

# Sprawdzamy czy działa
podman ps
```

### Testujemy aplikację
```bash
curl localhost:5001
# Powinno zwrócić: <h1>Witaj w moim sklepie!</h1>
```

---

## Krok 6: Konfigurujemy Caddy

### Tworzymy Caddyfile
```bash
sudo nano /etc/caddy/Caddyfile
```

### Zawartość pliku (na razie bez domeny):
```
# Konfiguracja dla IP (na początku)
:80 {
    handle /sklep* {
        reverse_proxy localhost:5001
    }
    
    respond "Serwer działa! Dodaj /sklep na końcu URL"
}
```

### Restart Caddy
```bash
sudo systemctl reload caddy
```

### Test w przeglądarce
Idź na: `http://123.456.789.123/sklep`

---

## Krok 7: Dodajemy kolejne aplikacje

### Blog (port 5002)
```bash
cd ~/projekty/blog
# Kopiujesz tutaj swoje pliki blog-a
# Tworzysz identyczne pliki: app.py, requirements.txt, Dockerfile

podman build -t moj-blog .
podman run -d --name blog -p 5002:5000 moj-blog
```

### API (port 5003)
```bash
cd ~/projekty/api
podman build -t moje-api .
podman run -d --name api -p 5003:5000 moje-api
```

### Portfolio (port 5004)
```bash
cd ~/projekty/portfolio
podman build -t moje-portfolio .
podman run -d --name portfolio -p 5004:5000 moje-portfolio
```

### Aktualizujemy Caddyfile
```bash
sudo nano /etc/caddy/Caddyfile
```

```
:80 {
    handle /sklep* {
        reverse_proxy localhost:5001
    }
    
    handle /blog* {
        reverse_proxy localhost:5002
    }
    
    handle /api* {
        reverse_proxy localhost:5003
    }
    
    handle /portfolio* {
        reverse_proxy localhost:5004
    }
    
    respond "Wybierz: /sklep, /blog, /api, /portfolio"
}
```

```bash
sudo systemctl reload caddy
```

---

## Krok 8: Automatyzacja z prostym skryptem

### Tworzymy skrypt do deploy-u
```bash
nano ~/deploy.sh
```

```bash
#!/bin/bash

echo "🚀 Rozpoczynam deployment..."

# Lista projektów i portów
declare -A projekty=(
    ["sklep"]=5001
    ["blog"]=5002
    ["api"]=5003
    ["portfolio"]=5004
)

# Deploy każdego projektu
for projekt in "${!projekty[@]}"; do
    port=${projekty[$projekt]}
    
    echo "📦 Deploying $projekt na porcie $port..."
    
    cd ~/projekty/$projekt
    
    # Zatrzymujemy stary kontener
    podman stop $projekt 2>/dev/null || true
    podman rm $projekt 2>/dev/null || true
    
    # Budujemy nowy
    podman build -t moj-$projekt .
    
    # Uruchamiamy
    podman run -d --name $projekt -p $port:5000 moj-$projekt
    
    echo "✅ $projekt działa na porcie $port"
done

echo "🎉 Wszystkie aplikacje uruchomione!"
echo "🌐 Sprawdź: http://$(curl -s ifconfig.me)/sklep"
```

```bash
chmod +x ~/deploy.sh
```

### Użycie skryptu
```bash
./deploy.sh
```

---

## Krok 9: Automatyczne uruchamianie po restarcie

### Tworzymy systemd service
```bash
sudo nano /etc/systemd/system/moje-aplikacje.service
```

```ini
[Unit]
Description=Moje Flask Apps
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
User=junior
ExecStart=/home/junior/deploy.sh
ExecStop=/usr/bin/podman stop sklep blog api portfolio

[Install]
WantedBy=multi-user.target
```

### Włączamy serwis
```bash
sudo systemctl daemon-reload
sudo systemctl enable moje-aplikacje
sudo systemctl start moje-aplikacje
```

---

## Krok 10: Dodajemy domenę (opcjonalnie)

### Jeśli masz domenę (np. mojadomena.pl)
```bash
sudo nano /etc/caddy/Caddyfile
```

```
mojadomena.pl {
    handle /sklep* {
        reverse_proxy localhost:5001
    }
    
    handle /blog* {
        reverse_proxy localhost:5002
    }
    
    handle /api* {
        reverse_proxy localhost:5003
    }
    
    handle /portfolio* {
        reverse_proxy localhost:5004
    }
    
    respond "Witaj na mojej stronie!"
}
```

**Caddy automatycznie skonfiguruje SSL! 🔒**

---

## 🛠️ Przydatne komendy do debugowania

### Sprawdzanie kontenerów
```bash
podman ps                    # Działające kontenery
podman ps -a                 # Wszystkie kontenery
podman logs sklep            # Logi kontenera
podman restart sklep         # Restart kontenera
```

### Sprawdzanie Caddy
```bash
sudo systemctl status caddy  # Status Caddy
sudo journalctl -u caddy -f   # Logi Caddy na żywo
caddy validate --config /etc/caddy/Caddyfile  # Sprawdzenie konfiguracji
```

### Monitoring serwera
```bash
htop                         # Zużycie CPU/RAM
df -h                        # Miejsce na dysku
```

---

## 🎯 Co dalej?

1. **Naucz się Git**: Zautomatyzuj deployment z GitHub
2. **Dodaj bazę danych**: PostgreSQL w kontenerze
3. **Monitoring**: Dodaj logi i metryki
4. **Backup**: Automatyczne kopie zapasowe
5. **CI/CD**: GitHub Actions do auto-deployment

---

## 💡 Częste problemy i rozwiązania

### Kontener się nie uruchamia
```bash
podman logs nazwa-kontenera  # Zobacz co jest nie tak
```

### Port zajęty
```bash
sudo lsof -i :5001          # Zobacz co używa portu
podman stop stary-kontener  # Zatrzymaj stary kontener
```

### Caddy nie działa
```bash
sudo systemctl status caddy
caddy validate --config /etc/caddy/Caddyfile
```

### Brak miejsca na dysku
```bash
podman system prune -a      # Usuń nieużywane obrazy
```

---

## 🎉 Gratulacje!

Masz teraz:
- ✅ Własny serwer za 2 EUR/miesiąc
- ✅ 4 aplikacje Flask w kontenerach
- ✅ Automatyczne SSL
- ✅ Prostą automatyzację
- ✅ Podstawy DevOps

**Koszt**: ~2 EUR/miesiąc  
**Czas setup**: ~30 minut  
**Poziom skomplikowania**: Początkujący  

**Jesteś teraz junior DevOps engineer! 🚀**
