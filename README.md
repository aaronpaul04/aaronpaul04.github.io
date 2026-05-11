# CTF Writeups

HackTheBox and CTF walkthroughs. Powered by Hugo + Blowfish theme, deployed on GitHub Pages.

## Setup

### 1. Install Hugo

**Mac:**
```
brew install hugo
```

**Windows:**
```
choco install hugo-extended
```

**Linux:**
```
sudo apt install hugo
```

Or download from https://gohugo.io/installation/

### 2. Add Blowfish theme

```bash
cd ctf-writeups
git init
git submodule add https://github.com/nunocoracao/blowfish.git themes/blowfish
```

### 3. Update config

Open `hugo.toml` and change `YOUR-USERNAME` to your GitHub username:

```
baseURL = "https://YOUR-USERNAME.github.io/ctf-writeups/"
```

### 4. Preview locally

```bash
hugo server -D
```

Open http://localhost:1313/ctf-writeups/

### 5. Deploy to GitHub Pages

```bash
git add .
git commit -m "initial commit"
git remote add origin https://github.com/YOUR-USERNAME/ctf-writeups.git
git branch -M main
git push -u origin main
```

Then in your GitHub repo:
- Go to **Settings > Pages**
- Set Source to **GitHub Actions**

The included `.github/workflows/hugo.yml` will auto-build and deploy on every push.

## Adding new writeups

1. Create a folder: `content/ctf-writeups/MACHINE-NAME/`
2. Drop your screenshots in that folder
3. Create `index.md` with front matter and `![](screenshot.png)` references
4. Push to main

Or use the archetype:
```bash
hugo new ctf-writeups/machine-name/index.md
```
