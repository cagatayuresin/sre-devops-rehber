# GitHub'a Push Etme Adımları

## 1. Git Repository Başlat
git init
git add .
git commit -m "Initial commit: SRE/DevOps Rehber"

## 2. GitHub'da Repo Oluştur
# GitHub'a gidip 'sre-devops-rehber' adında yeni bir repo oluşturun

## 3. Remote Ekle ve Push Et
git branch -M main
git remote add origin https://github.com/cagatayuresin/sre-devops-rehber.git
git push -u origin main

## 4. GitHub Pages'i Aktif Et
# GitHub repo sayfasında:
# Settings → Pages → Source: GitHub Actions seçin

## 5. Site Hazır!
# Birkaç dakika içinde siteniz şu adreste yayında olacak:
# https://cagatayuresin.github.io/sre-devops-rehber/
