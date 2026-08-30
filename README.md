<div align="center">

# Deploy to Virtual Machine

<p>Perintah-perintah terminal step by step untuk melakukan deploy <a href="https://huggingface.co/virtual-miku/miku-voice/tree/main"><b>miku-voice.zip</b></a> dari komputer lokal ke Azure VM via SCP (Secure Copy Protocol)</p>

[![Azure](https://img.shields.io/badge/Azure-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)](https://azure.microsoft.com)
[![PM2](https://img.shields.io/badge/PM2-2B037A?style=for-the-badge&logo=pm2&logoColor=white)](https://pm2.io)
[![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)](https://nginx.org)
[![npm](https://img.shields.io/badge/npm-CB3837?style=for-the-badge&logo=npm&logoColor=white)](https://www.npmjs.com)

</div>

### Memulai
```
winget install Microsoft.AzureCLI
az --version
az login
```

## Backend

```ps1
az group create `
   --name MikuVoiceRG `
   --location indonesiacentral

az vm create `
   --resource-group MikuVoiceRG `
   --name MikuVoiceVM `
   --image Ubuntu2204 `
   --size Standard_D2as_v5 `
   --admin-username azureuser `
   --generate-ssh-keys `
   --location indonesiacentral

az vm open-port `
   --resource-group MikuVoiceRG `
   --name MikuVoiceVM `
   --port 3001 `
   --priority 1001

scp miku-voice.zip azureuser@48.193.42.161:~/

ssh azureuser@48.193.42.161
```

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

sudo apt install -y nodejs ffmpeg unzip python3-pip python3-venv python-is-python3

unzip miku-voice.zip -d miku-voice

cd ~/miku-voice/miku-voice/backend

sudo npm install -g pm2

npm install

pm2 start index.js --name "miku-voice"

pm2 startup

sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u azureuser --hp /home/azureuser

pm2 save

pm2 logs miku-voice --lines 20

curl http://localhost:3001
```

atau buka di browser: `http://48.193.42.161:3001`

--

## Worker

```bash
cd ~/miku-voice/miku-voice/worker

sudo apt update && sudo apt install -y python3-pip python3-venv

pip3 install torch==2.3.1 torchaudio==2.3.1 --extra-index-url https://download.pytorch.org/whl/cpu

pip3 install -r requirements.txt

RVC_DEVICE=cpu pm2 start main.py --name "miku-worker" --interpreter python3

pm2 save

pm2 logs miku-worker --lines 20
```

## Frontend

```bash
cd ~/miku-voice/miku-voice/frontend

npm install

chmod -R +x node_modules/.bin

sed -i 's/localhost:3001/48.193.42.161:3001/g' $(grep -rl "localhost:3001" src/)

npm run build

sudo npm install -g serve

pm2 start serve --name "miku-frontend" -- -s dist -l 8000

pm2 save
```

```ps1
az vm open-port `
   --resource-group MikuVoiceRG `
   --name MikuVoiceVM `
   --port 8000 `
   --priority 1003
```

Akses di: `http://48.193.42.161:8000`

## Ubah IP VM ke DNS

Tambahkan DNS record baru:
- Type: A
- Name/Host: voice
- Value / Target (IP VM): 48.193.42.161
- TTL: Auto

Atau subdomain gratis Azure:
- Cek nama IP Publik VM:
```ps1
az network public-ip list --resource-group MikuVoiceRG --query "[].{Name:name}" -o table
```

- Setel DNS name (Sesuaikan `MikuVoiceVMPublicIP`):
```ps1
az network public-ip update `
   --resource-group MikuVoiceRG `
   --name MikuVoiceVMPublicIP `
   --dns-name miku
```

```bash
cd ~/miku-voice/miku-voice/frontend
```

- ubah `voice.miku.my.id` menjadi `miku.indonesiacentral.cloudapp.azure.com` jika pakai subdomain gratis Azure:
```bash
sed -i 's/48.193.42.161:3001/voice.miku.my.id:3001/g' $(grep -rl "48.193.42.161:3001" src/)

npm run build

pm2 restart miku-frontend
```

## Reverse proxy Nginx
```ps1
az vm open-port --resource-group MikuVoiceRG --name MikuVoiceVM --port 80 --priority 1004
```

```bash
sudo apt update && sudo apt install -y nginx

sudo bash -c 'cat << "EOF" > /etc/nginx/sites-available/default
server {
    listen 80;
    server_name voice.miku.my.id;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF'

sudo nginx -t

sudo systemctl restart nginx
```

Akses di: `http://voice.miku.my.id`

## Konfigurasi HTTPS

```ps1
az vm open-port --resource-group MikuVoiceRG --name MikuVoiceVM --port 443 --priority 1005
```

```bash
sudo apt update && sudo apt install -y certbot python3-certbot-nginx

sudo certbot --nginx -d voice.miku.my.id

sudo bash -c 'cat << "EOF" > /etc/nginx/sites-available/default
server {
    listen 80;
    server_name voice.miku.my.id;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name voice.miku.my.id;

    # Certbot SSL Configuration
    ssl_certificate /etc/letsencrypt/live/voice.miku.my.id/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/voice.miku.my.id/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Route Frontend (Port 8000)
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Route Backend API & File Media (Port 3001)
    location ~ ^/(api|audio|uploads|creations|storage)/ {
        proxy_pass http://127.0.0.1:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        client_max_body_size 100M;
    }
}
EOF'

sudo nginx -t

sudo systemctl reload nginx

cd ~/miku-voice/miku-voice/frontend

sed -i 's|http://voice.miku.my.id:3001|https://voice.miku.my.id|g' $(grep -rl "voice.miku.my.id" src/)
sed -i 's|48.193.42.161:3001|https://voice.miku.my.id|g' $(grep -rl "48.193.42.161" src/)
```
Apabila muncul `sed: no input files`, abaikan saja.

```bash
npm run build

pm2 restart miku-frontend
```

Akses: `https://voice.miku.my.id/`

## Menghentikan VM
```ps1
az vm deallocate --resource-group MikuVoiceRG --name MikuVoiceVM
```

## Menjalankan VM
```ps1
az vm start --resource-group MikuVoiceRG --name MikuVoiceVM
```

## File Manager
```bash
sudo apt install -y mc

mc
```
