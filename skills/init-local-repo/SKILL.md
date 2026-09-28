---
name: init-local-repo
description: Initialise de bout en bout un dépôt GitHub sur la machine locale, jusqu'au premier commit d'un README.md poussé sur GitHub (ou jusqu'au clone prêt à l'emploi pour un repo existant). Gère au fil de l'eau les questions à l'utilisateur, l'identité git (nom seul, sans email), la clé SSH dédiée à GitHub avec passphrase, la vérification de l'hôte GitHub, et le premier push. À utiliser quand l'utilisateur veut créer/relier/cloner un repo GitHub en local ou préparer sa machine pour travailler dessus avec Claude Code.
---

# Initialisation d'un repo GitHub en local, de bout en bout

Objectif : partir de rien et arriver à un dépôt local relié à GitHub, avec un
premier commit `README.md` présent sur GitHub. En mode clonage, l'objectif
est un clone prêt à l'emploi.

## Comment poser les questions

- Poser les questions **au fil de l'eau**, au moment où l'étape en a
  besoin, pas toutes au départ. Ne jamais redemander ce qui est déjà connu
  (contexte, réponses précédentes, état de la machine).
- **Choix fermés** (mode, dossier, branche…) : `AskUserQuestion`, avec
  l'option recommandée en premier.
- **Texte libre** (URL, nom git…) : poser la question dans un message
  normal et terminer le tour pour attendre la réponse. Ne pas utiliser une
  option « Je la saisis » dans `AskUserQuestion` : l'utilisateur la
  sélectionne souvent sans taper le texte.
- **Actions que l'utilisateur doit faire lui-même** (créer le repo sur
  github.com, taper une passphrase, ajouter une clé sur GitHub, pousser
  avec une clé à passphrase) : donner les instructions exactes, prêtes à
  copier, puis terminer le tour. Quand l'utilisateur revient, **vérifier**
  que c'est fait avant de continuer, sans le croire sur parole.
- Ne jamais demander de passphrase, de mot de passe ou de token dans la
  conversation.

## Étape 0 — État des lieux (sans question)

```bash
pwd; ls -la                       # dossier courant vide ? déjà un repo git ?
git config user.name              # identité existante
ls -la ~/.ssh; cat ~/.ssh/config  # clés et config SSH existantes
ssh-keygen -F github.com          # GitHub déjà dans known_hosts ?
claude --version                  # Claude Code CLI présent ?
```

## Étape 1 — Dépôt GitHub

1. Demander le **mode** (`AskUserQuestion`) :
   - *Nouveau projet* : dépôt local neuf relié à un dépôt GitHub vide
     (recommandé si le dossier est vide) ;
   - *Cloner* un dépôt GitHub qui a déjà du contenu.
2. Demander l'**URL** en texte libre, de préférence en SSH :
   `git@github.com:<owner>/<repo>.git`. Si l'utilisateur donne une URL
   HTTPS, proposer la forme SSH équivalente.
   - Mode nouveau projet, si le dépôt n'existe pas encore sur GitHub :
     expliquer github.com → **New repository**, nom du repo, et **ne cocher
     ni README, ni .gitignore, ni licence** (le repo doit être vide). Attendre
     l'URL.

## Étape 2 — Dépôt local

1. Dossier : si le dossier courant est vide, le proposer par défaut ;
   sinon proposer un sous-dossier `<repo>`. Vérifier avec `ls -la` que la
   cible n'existe pas ou est vide. Ne jamais écraser un dossier non vide
   sans confirmation explicite.
2. **Nouveau projet** :
   ```bash
   git init -b main
   git remote add origin <url>
   git remote -v
   ```
   Si c'est déjà un repo git avec un autre `origin`, demander avant
   `git remote set-url origin <url>`.
3. **Cloner** : le clone a besoin de l'accès SSH. Faire d'abord l'étape 4,
   puis `git clone <url> <dossier>`.

## Étape 3 — Identité git (nom seul, sans email)

- Montrer l'identité globale trouvée à l'étape 0 et demander
  (`AskUserQuestion`) : la garder, ou utiliser un autre nom pour ce repo.
- Autre nom : le demander en texte libre, **sans demander d'email**, puis :
  ```bash
  git config --local user.name "<nom>"
  git config --local user.email ""      # empêche de reprendre l'email global
  git var GIT_AUTHOR_IDENT              # doit afficher "<nom> <> ..."
  ```
- Ne définir un email que si l'utilisateur en donne un de lui-même. Ne
  jamais toucher à `--global` sans demande explicite.
- Signaler que les commits `<nom> <>` ne sont pas reliés au compte GitHub
  (pas d'avatar). L'adresse noreply GitHub est possible si l'utilisateur
  le souhaite plus tard.

## Étape 4 — Accès SSH à GitHub

### 4.1 Hôte GitHub connu

Si `ssh-keygen -F github.com` ne trouve rien, récupérer la clé d'hôte dans
le scratchpad et vérifier son empreinte **avant** de l'ajouter. Elle doit
être exactement l'empreinte ed25519 officielle de GitHub
`SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU` :
```bash
ssh-keyscan -t ed25519 github.com 2>/dev/null > <scratchpad>/gh_hostkey
ssh-keygen -lf <scratchpad>/gh_hostkey
cat <scratchpad>/gh_hostkey >> ~/.ssh/known_hosts   # seulement si identique
```
Si l'empreinte diffère : s'arrêter et prévenir l'utilisateur.

### 4.2 Test de la clé actuelle

```bash
ssh -v -o BatchMode=yes -o ConnectTimeout=10 -T git@github.com 2>&1 \
  | grep -Ei "offering|server accepts|successfully|denied"
```
- `successfully authenticated` → accès OK, sans passphrase. Passer à 4.5.
- `Server accepts key` puis `Permission denied` → la clé est sur GitHub
  mais protégée par une passphrase. Accès OK, mais **c'est l'utilisateur
  qui poussera** (étape 5). Passer à 4.5.
- `Permission denied` sans `Server accepts key` → aucune clé connue de
  GitHub. Aller en 4.3.

### 4.3 Clé dédiée à GitHub, avec passphrase

Si une clé existe déjà (ex. `~/.ssh/id_ed25519`), ne pas la modifier : elle
peut servir ailleurs (voir `~/.ssh/config`) et sa passphrase peut être
inconnue. Proposer (`AskUserQuestion`) :
- *Nouvelle clé dédiée GitHub avec passphrase* (recommandé) ;
- *Utiliser la clé existante* : l'utilisateur doit l'ajouter sur GitHub et
  connaître sa passphrase ;
- *Nouvelle clé sans passphrase* : Claude peut alors pousser lui-même, mais
  le fichier de clé donne un accès direct au compte. Expliquer ce risque.

**Avec passphrase**, c'est l'utilisateur qui crée la clé, dans **son
propre terminal** (Git Bash ou PowerShell, hors de Claude Code), pour taper
la passphrase lui-même. Vérifier avant que `~/.ssh/id_ed25519_github`
n'existe pas :
```
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github -C "<nom>-github"
```
(en PowerShell : `-f $HOME\.ssh\id_ed25519_github`). Terminer le tour. Au
retour, vérifier :
```bash
ls ~/.ssh/id_ed25519_github ~/.ssh/id_ed25519_github.pub
ssh-keygen -y -P "" -f ~/.ssh/id_ed25519_github >/dev/null 2>&1; echo $?
# 255 attendu = la passphrase est bien en place
```

**Sans passphrase** (seulement si l'utilisateur l'a choisi) :
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github -C "<nom>-github" -N "" -q
```
Si l'utilisateur change d'avis ensuite, il peut ajouter une passphrase
dans son terminal avec `ssh-keygen -p -f ~/.ssh/id_ed25519_github`
(ancienne passphrase : Entrée). La clé publique reste la même.

Puis faire utiliser cette clé pour GitHub uniquement, en ajoutant à
`~/.ssh/config` s'il n'y a pas déjà de bloc `Host github.com` :
```
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_github
  IdentitiesOnly yes
```

### 4.4 Ajout de la clé publique sur GitHub

Afficher le contenu de la clé publique (`cat ~/.ssh/id_ed25519_github.pub`)
dans un bloc de code, et préciser qu'elle ne contient rien de secret.
Instructions : GitHub → **Settings → SSH and GPG keys → New SSH key**,
titre au choix (nom du PC), coller la clé. Terminer le tour.

Au retour, refaire le test 4.2. Attendre `Server accepts key` (clé à
passphrase) ou `successfully authenticated` (sans passphrase). Sinon,
vérifier avec l'utilisateur que la clé collée est bien celle affichée
(empreinte : `ssh-keygen -lf ~/.ssh/id_ed25519_github.pub`, visible aussi
sur la page GitHub).

### 4.5 Accès au dépôt

Seulement si la clé n'a pas de passphrase (sinon ce test échoue forcément
depuis Claude) :
```bash
git ls-remote origin      # vide et sans erreur = repo vide accessible
```
`Repository not found` → URL erronée ou droits manquants.

HTTPS au lieu de SSH : orienter vers le Git Credential Manager ou
`gh auth login`, jamais vers un token écrit dans l'URL ou dans le repo.

## Étape 5 — Premier commit README.md et premier push (nouveau projet)

1. Créer le `README.md` (par défaut `# <repo>`, ou le contenu demandé) et
   faire le commit :
   ```bash
   printf '# <repo>\n' > README.md
   git add README.md
   git commit -m "Premier commit"
   git log --stat            # montrer ce qui partira sur GitHub
   ```
   L'avertissement `LF will be replaced by CRLF` sous Windows est normal.
2. Push :
   - **Clé avec passphrase** : Claude ne peut pas la saisir. Donner à
     l'utilisateur la commande à lancer dans son propre terminal, puis
     terminer le tour :
     ```
     cd <dossier>
     git push -u origin main
     ```
     Mentionner l'option agent SSH pour ne taper la passphrase qu'une fois
     (`ssh-add ~/.ssh/id_ed25519_github`), sans l'imposer.
   - **Clé sans passphrase** : demander confirmation (le push publie sur
     GitHub), puis lancer
     `GIT_SSH_COMMAND="ssh -o BatchMode=yes" git push -u origin main`.
3. Vérifier, y compris après un push fait par l'utilisateur :
   ```bash
   git status -sb                  # "## main...origin/main" sans [ahead]
   git rev-parse --verify origin/main
   ```
   Si `origin/main` n'existe pas, le push n'a pas eu lieu : relire l'erreur
   avec l'utilisateur (repo non vide → il a coché README à la création ;
   `Permission denied` → retour à l'étape 4).

## Étape 5 bis — Mode clonage

Après le clone : si l'utilisateur reprend un travail existant (ex. session
cloud Claude Code), `git fetch origin <branche>` puis
`git checkout <branche>`. Sinon, proposer `git checkout -b <branche>`.
Installer les dépendances détectées : `package.json` → npm/pnpm/yarn selon
le lockfile ; `requirements.txt`/`pyproject.toml` → pip/poetry ;
`Cargo.toml` → `cargo build` ; `go.mod` → `go mod download`. Aucun push
dans ce mode.

## Étape 6 — Résumé final

Donner en quelques lignes : dossier local, `origin`, branche active et son
suivi, identité git (`<nom> <>`), clé SSH utilisée (et si elle a une
passphrase), résultat du premier push (ou état du clone), et comment
lancer Claude Code (`claude` dans le dossier ; sinon
`npm install -g @anthropic-ai/claude-code`). Rappeler le cycle suivant :
`git add` → `git commit -m "..."` → `git push` (dans son terminal si la
clé a une passphrase).

## Règles

- Ne jamais écraser un dossier, un repo, une clé SSH ou un bloc
  `~/.ssh/config` existant sans confirmation explicite.
- Ne jamais modifier la config git globale sans demande explicite.
- Ne jamais ajouter une clé d'hôte sans avoir vérifié son empreinte.
- Seul push autorisé : le premier push d'un repo vide (étape 5). Jamais de
  `--force`, jamais de PR.
- Ne jamais demander ni manipuler de passphrase ou de token dans la
  conversation.
